# Local folder vault + Windows desktop companion

Two questions: (1) how much code must change to swap the external Firebase sync for a
**local folder vault** synced by Dropbox/OneDrive/etc.; (2) the **lowest-cost way to build a
Windows PC app** given the Android codebase.

## 0. Key architecture facts (what makes this feasible)

- **The app is already local-first.** All UI/data reads go through **ObjectBox**
  (`dao/objectbox/*`: `ClipBox`, `FileRefBox`, `FilterBox`, `SettingsBox`, `UserBox`,
  `LinkPreviewBox`). Firebase is *only* the sync/transport layer.
- **A file-based interchange format already exists.** `backup/` exports/imports a Gson JSON
  document (`CliptoBackup` = notes + filters/tags/snippet-kits + settings) and can also import
  Clipper, SimpleNote, Google Keep, ClipStack and ClipboardManager formats.
- **SAF (Storage Access Framework) is already used** for that import/export
  (`ACTION_OPEN_DOCUMENT` / `ACTION_CREATE_DOCUMENT`, `takePersistableUriPermission`) — so
  pointing at a Dropbox/OneDrive folder that exposes a `DocumentsProvider` is a solved pattern.
- **Firebase coupling is concentrated**: only **26 files** import `com.google.firebase`, and the
  sync itself lives in **`dao/firebase/*`** (6–7 classes) called from **4 repositories**.
- **The domain model is ~80% portable**: of ~41 `clipto/domain/*` files only 8 import `android.*`.
- **`converter` is a pure-JVM module** (Jackson, `java.io.File`, `@JvmStatic main`) — proof the
  project already builds a non-Android artifact.
- **UI is classic Android Views** — 186 XML layouts + viewBinding, **zero Compose**.

## 1. Local folder vault instead of Firebase — how much code changes?

**Verdict: a focused, medium-sized refactor — not a rewrite.** The UI, domain, ObjectBox store,
`store/*` state holders and the backup format are reused. The work is concentrated in the
**sync layer, the 4 repositories, auth, and files/attachments.**

### Must change
| Area | Files | What to do |
|---|---|---|
| **Sync transport** | `dao/firebase/*` (`ClipFirebaseDao`, `FileFirebaseDao`, `FilterFirebaseDao`, `FirebaseDaoHelper`, `FirebaseOptimizedSnapshotListener`, `FunctionsFunctionsHelper`) | Replace with a folder-vault engine (watch a directory, read/write per-item JSON, merge). Delete/replace Firebase classes. |
| **Repositories** | `repository/ClipRepository.kt`, `FileRepository.kt`, `FilterRepository.kt`, `UserRepository.kt` | They import Firebase directly (`ClipFirebaseDao`, `FirebaseDaoHelper`, `ClipMapper`, `DocumentChange`, `FieldValue`, `UserCollection`, `FirebaseAuth`). Introduce an interface (e.g. `IVaultSync`/`ISyncDao`) and depend on that instead. `startSync`/`stopSync` call sites are here (ClipRepository:60/65, FileRepository:68/124, FilterRepository:40/45). |
| **Files / attachments** | `repository/FileRepository.kt` (~800 lines), `domain/FileRef.kt`, `dao/firebase/model/UserCollection.kt` | Currently content-URIs + Firebase Storage (`putFile`, `downloadUrl`, `{uid}/{folder}/{name}`). For a vault, store binaries under `vault/files/...` and reference by relative path. **This is the hardest part.** |
| **Server API** | `api/IApi.kt`, `api/Api.kt`, `dao/firebase/FunctionsFunctionsHelper.kt` | Session/billing, public links, snippet-kit public library, link previews are server features. Stub or drop them in local mode. |
| **Auth** | `presentation/auth/Auth.kt`, `repository/UserRepository.kt` (line 265 `FirebaseAuth` listener) | Replace FirebaseUI/FirebaseAuth with a local single-user identity (no login). |
| **Config / misc** | `config/AppConfig.kt` (Remote Config), `analytics/*`, `store/analytics/AnalyticsState.kt`, `AppLinkPreviewManager.kt`, `utils/GlideUtils.kt`, `AppUtils.kt`, `AppContainer*.kt`, DI modules | Remote Config → keep bundled defaults (already works offline). Strip/replace Analytics/Crashlytics/FCM. Swap DI providers. |
| **Manifest/build** | `AndroidManifest.xml`, `app/build.gradle`, `google-services.json` | Remove Firebase init, plugin, config. |

### Likely reusable as-is
`domain/*` (mostly), `dao/objectbox/*`, `store/*` (except analytics), `backup/*`,
all `presentation/*` screens (186 layouts), `common-presentation`, `converter`.

### Two implementation strategies
- **Strategy A — real vault sync (recommended).** Store one JSON document per item
  (`vault/notes/<id>.json`, `filters/<id>.json`, `settings.json`) plus `vault/files/…` blobs,
  reusing the existing mappers for (de)serialization. A `FileObserver`/periodic scan replaces
  the Firestore real-time listeners. Gives incremental sync, tombstones and per-item merge.
  More work, best result.
- **Strategy B — reuse the backup JSON (lowest effort).** Point the existing
  Backup/Restore at the Dropbox folder and add a watcher that auto-exports on change and
  auto-imports on external change. Minimal new code, but coarse-grained (whole-vault rewrite,
  last-writer-wins) and **attachments are not included** in the backup JSON, so files need
  separate handling.

### Things that need care
- **Conflicts:** Dropbox/OneDrive create "conflicted copy" files; the vault writer must detect
  and merge them (per-item JSON + change timestamps makes this tractable; a single monolithic
  file does not).
- **Deletions:** keep tombstones (the app already has `deleteDate`) so deletes propagate.
- **Attachments:** the current JSON backup carries `fileIds` (references) but **not the
  binaries** — copy file blobs into the vault explicitly.
- **Android background:** a folder watcher + periodic worker (the app already uses WorkManager)
  is more reliable than a live watcher on modern Android.

### Effort (rough)
- Sync engine + repository abstraction: the bulk of the work.
- Auth/API/config de-Firebase-ing: mechanical.
- Files/attachments on a folder: non-trivial (Android `Uri`/SAF + `FileRepository`).
- UI/domain/ObjectBox: essentially untouched.


## 2. Lowest-cost way to build the Windows PC app

**Context that drives the cost:** the Android UI is **classic Views (186 XML layouts, no
Compose)**, so the presentation layer **cannot be reused** on desktop. However the `domain`
model is mostly pure Kotlin, **ObjectBox has a desktop/JVM edition**, the JSON vault format is
plain JSON, and `converter` proves the repo already produces JVM artifacts.

### Options, cheapest first
| Option | Dev cost | License cost | Reuse | Notes |
|---|---|---|---|---|
| **1. Run the Android APK in a Windows Android emulator** (LDPlayer, BlueStacks, Google Play Games on PC, or an Android Studio AVD) | **~0** | $0 | 100% of the app | Not a native Windows app, but an emulator **shared folder** can point at the Dropbox vault. Lowest cost by far. Note: *Windows Subsystem for Android was discontinued (Mar 2025)* — use a third-party emulator or AVD instead. |
| **2. Kotlin/JVM desktop app** reusing `domain` + **ObjectBox JVM** + the JSON vault format, new UI in **Compose for Desktop** (or JavaFX/Swing) | Low–medium (weeks) | $0 (ObjectBox Community, Compose Desktop free) | domain, vault format, ObjectBox models | Best "real" Windows app per unit of effort. UI must be written fresh (Views aren't portable). |
| **3. Kotlin Multiplatform / Compose Multiplatform** | Medium–high | $0 | domain + a *future* shared UI | Best long-term (share code with Android later), but current UI is Views so the UI is a rewrite anyway. |
| **4. Electron / Tauri / web app** | High | $0 | only the JSON schema | Full re-implementation in JS/TS; cheap hosting. Only sensible if you want a web-first product. |
| **5. Wrap the existing Clipto web app** | Low | $0 | n/a | **Doesn't fit** the local-vault plan — the web app depends on the Firebase backend. |

### Recommendation
- **Just need it working now:** Option 1 (emulator + shared Dropbox folder). Effectively free.
- **Want a genuine Windows app on a budget:** Option 2 — a thin Kotlin/JVM desktop app that
  reads/writes the **same JSON vault** and reuses the `domain` classes and ObjectBox models.
  Because the vault is the contract, the Windows app and Android app only need to agree on the
  folder format, not on code.
- **Long-term:** Option 3 (Compose Multiplatform), which lets Android and Windows share the
  domain *and* a new Compose UI.

### Suggested shared "vault" contract (keeps the Windows app cheap)
```
<vault>/
  notes/<id>.json          # clip fields (see ClipMapper / CliptoItem)
  filters/<id>.json        # tags, filters, snippet kits
  settings.json
  files/<folder>/<name>    # attachment binaries
  files/<folder>/thumbs/<name>_200x200
  meta.json                # schema version, device ids, last-sync
```
Keep it per-item JSON (not one big file) so Dropbox/OneDrive conflicts are mergeable and

## 3. The official Clipto Desktop app — reality check

There **is** an official Windows/macOS/Linux desktop app, but it is a dead end for a
local-folder-vault workflow.

### What exists
- Releases: `github.com/clipto-pro/Desktop/releases` — latest **v7.2.17 (2021-11-15)**; assets
  `clipto-7.2.17.exe` (Windows, ~69 MB), `.dmg` (macOS), `.AppImage` (Linux) + electron-updater
  `latest*.yml`.
- The `clipto-pro` org has **only two repos**: `Desktop` and `Android`. Both are
  **release-binary-only** — the `Desktop` repo contains **just a `README.md` + PNG screenshots**
  (`git tree` confirms no source). All 21 forks are the same size (no source either).
- **Technology: Electron** (electron-builder/electron-updater artifacts; Chromium **LevelDB**
  data files). So the only "source" obtainable is the bundled/minified `app.asar` inside the
  installed app.

### Why there's no JSON import (maintainer's own words)
From issues [#143](https://github.com/clipto-pro/Desktop/issues/143) and
[#64](https://github.com/clipto-pro/Desktop/issues/64), maintainer **atrashler**:
- *"On Windows there is no such option to import/export notes to local files (only on Android
  now). Now you can use cloud sync…"*
- *"Desktop app is the same, but does not support import/export yet."*
- *"Now it is only possible to do by using cloud account."*
- *"The app uses Firebase Cloud to store and sync data across all platforms."*

So the desktop app is **cloud-sync-only by design**; import/export was requested but never added.

### Where the desktop data lives (useful for recovery/migration)
- Path: **`C:\Users\<USER>\AppData\Roaming\Clipto`** (Windows).
- Contents are a Chromium **LevelDB/IndexedDB** store — `databases.db`, `*.ldb`, `*.log` — not
  JSON. Users report recognizable note text inside the `.ldb` files.
- Maintainer's backup advice: *"Probably it is better to copy the whole directory."*

### Maintenance status
- Issue [#156 "State of the project?"](https://github.com/clipto-pro/Desktop/issues/156):
  the maintainer (`atrashler`) has been **MIA for years** and reportedly has no intention to
  continue. The app is **delisted from Google Play**, sync subscription options are broken, and
  users report being billed without working sync. → Treat the desktop app as **abandoned**.

### Practical options for a Windows + Android local-vault setup
1. **Migrate once, then go vault:** read notes out of the desktop **LevelDB** store
   (`%APPDATA%\Roaming\Clipto`) with a LevelDB/IndexedDB reader, convert to the Android JSON
   backup format, then stop using the desktop app.
2. **Run the Android app on Windows** (emulator + shared folder) — it *does* have JSON
   import/export, so it fits the folder vault (see §2, option 1).
3. **Build a small custom Windows app** against the vault format (see §2, options 2–3).
4. **Reverse-engineer the Electron app:** unpack `resources/app.asar` from the installed app to
   read/patch the bundled JS (the only available "source"). Fragile, unsupported, and likely
   contrary to the app's terms — not recommended.
5. **Keep using cloud sync** while the `wb-clipto` backend is alive — but it is unmaintained and
   the sync-plan/billing path is broken, so this is not a long-term option.


### 3.1 Install folder vs data folder — and how to get the "source"

The listed files (`Clipto.exe`, `resources\`, `locales\`, `*.dll`, `snapshot_blob.bin`,
`LICENSE.electron.txt`, `LICENSES.chromium.html`, `*.pak`, `icudtl.dat`) are the **installed
program directory**. They **confirm Electron** (the Electron/Chromium runtime + license files).

- `resources\app.asar` — **the entire application code, packed.** This *is* the "source" that
  isn't on GitHub (bundled/minified JS). Extract it:
  ```bat
  :: needs Node.js
  npx @electron/asar extract "resources\app.asar" app_src
  ```
  Then read `app_src\...` — look for the `main`/`renderer` bundles, `firebase`/`firestore`
  strings, the collection names (should mirror Android: `u/{uid}/c`, `u/{uid}/f`, `u/{uid}/fs`…),
  and any `app.getPath('userData')` usage. (7-Zip with the Asar7z plugin also works.)
- `resources\app.asar.unpacked\` — native modules, if any.
- `locales\`, `*.pak`, `icudtl.dat`, `ffmpeg.dll`, `libEGL.dll`, `vk_swiftshader*` — Chromium runtime.

**Your notes are NOT in these program files.** They live in a Chromium profile as **LevelDB /
IndexedDB** (`Local Storage\leveldb\*.ldb`, `IndexedDB\`, `*.db`). Find them with:
```powershell
Get-ChildItem -Recurse "$env:APPDATA\Clipto" -Include *.ldb,*.log,*.db,databases.db |
  Select-Object FullName, Length, LastWriteTime
```
If the app's `userData` equals its install dir, the profile is mixed into this same tree;
otherwise check `%APPDATA%\Clipto` and any `Local Storage`/`IndexedDB` subfolders. Read the
`.ldb`/`.log` with a LevelDB/IndexedDB viewer (Node `level`, `chrome-leveldb`, or an IndexedDB
browser) — text notes appear as recognizable strings inside.

### 3.2 Getting your notes out of LevelDB (recovery/migration recipe)

1. Close the Clipto desktop app (so LevelDB locks are released).
2. Copy the whole profile folder (maintainer's advice) before touching it.
3. Open the LevelDB/IndexedDB store and locate the `firestore` database and the `Local Storage`
   store; export the raw key/value entries.
4. Map the recovered documents to the **Android JSON backup schema** (`CliptoBackup`: `notes`,
   `filters`, `settings`) — the field names mirror the Firestore attributes in
   `FirebaseDaoHelper` (`ATTR_*`) used by the Android app.
5. Import the resulting JSON via the **Android** app's *Restore from File*.


### 3.3 Data profile map (confirmed contents of `%APPDATA%\Roaming\Clipto`)

The confirmed profile contains the standard Chromium/Electron `userData` layout. Where the
actual note data sits:

| Item | What it is | Relevance |
|---|---|---|
| `IndexedDB\` | Chromium **IndexedDB** (LevelDB) | **Primary store** — the Firebase JS SDK's Firestore offline persistence (`enableIndexedDbPersistence`) writes documents here as `firestore/[projectId]/[databaseId]`. Note text appears as readable strings. |
| `Local Storage\leveldb\` | Web **localStorage** (LevelDB) | Settings/session/flags; may also cache data. |
| `databases\` (`databases.db`) | legacy **WebSQL** catalog | The 28 KB `databases.db` from issue #162 lives here — a catalog, not the notes. |
| `config.json` | app config (plain JSON) | **Read this first** — likely app/user config and maybe the data path. |
| `Cookies`, `Cookies-journal`, `Network Persistent State` | session/auth | Firebase auth session; keep for reference, not needed for notes. |
| `Cache\`, `GPUCache\`, `Code Cache\`, `blob_storage\` | caches/blobs | Attachments may pass through `blob_storage\`; caches are disposable. |
| `QuotaManager*`, `Local State`, `Preferences`, `lockfile`, `.updaterId` | Chromium bookkeeping / electron-updater | Not note data. |

**So the notes are in `IndexedDB\` (Firestore cache) and possibly `Local Storage\leveldb\`.**

### 3.4 Reading the LevelDB / IndexedDB store

Always **close the app first** and **copy the profile** before opening it.

**Option A — no code (DevTools).** Launch Chrome/Chromium with a *copy* of the profile as its
user-data-dir, open DevTools → **Application → IndexedDB / Local Storage**, and browse the
`firestore/...` database. (Use a copy; never point Chrome at the live profile.)

**Option B — dump it with Node** (fastest for text extraction):
```js
// npm i classic-level   then:  node dump.js "C:\path\to\IndexedDB\<db>.leveldb"
const { ClassicLevel } = require('classic-level');
(async () => {
  const db = new ClassicLevel(process.argv[2], { keyEncoding: 'buffer', valueEncoding: 'buffer' });
  await db.open();
  for await (const [k, v] of db.iterator()) {
    const s = v.toString('utf8');
    if (/[ -~]{6,}/.test(s)) console.log('---\n' + s.replace(/[^\x20-\x7e\n]/g, '.'));
  }
  await db.close();
})();
```

**Option C — GUI viewers:** any LevelDB viewer (VS Code "LevelDB" extension, `leveldb-viewer`,
etc.) can open the `.leveldb` folder inside `IndexedDB\`.

**Then:** map the recovered documents to the Android **`CliptoBackup`** JSON schema
(`notes` / `filters` / `settings`), reusing the field names from the Android
`FirebaseDaoHelper.ATTR_*` constants, and import via the Android app's *Restore from File*.


## Related documents
- `REVIEW.md` — user-facing feature review and architecture overview.
- `BACKEND_REBUILD.md` — external-domain/Firebase backend map, rebuild options, live
  reachability probe.

Android's `FileObserver` + Windows' `FileSystemWatcher` can both diff cheaply.
