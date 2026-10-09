# Clipto (Android) — User-Facing Feature Review

Clipto (`com.wb.clipboard`, app module `app`, package `clipto`) is an offline-first
**notepad + clipboard manager** for Android. There is no README in the repo; this review
was derived from `AndroidManifest.xml`, the navigation graph (`res/navigation/nav_main.xml`),
the string resources (`res/values/strings.xml`), and the `clipto.presentation` /
`clipto.domain` code.

## Core data model
- **Notes (Clips)** — text notes with title, body, tags, folder, starred flag, color,
  attachments, dynamic fields, public links, and sync state.
- **Files** — uploaded/attached files (image, video, audio, document, archive) browsable
  in a dedicated "Files" category with metadata-only sync.
- **Tags**, **Filters**, **Folders**, **Snippets / Snippet Kits** — organisation and reuse.

## Main user-facing functions

### 1. Notes management (main list)
- Create, edit, delete (soft delete → recycle bin / restore), star, copy, share, print,
  merge multiple notes, and batch-edit attributes.
- Multiple categories in the left drawer: All, Untagged, Starred, Filtered, Notes from
  Clipboard, Snippets, Filters, Folders, Files, Tags, Deleted.
- Search, per-category sorting (usage/edit/create/delete date, usage count, title, text,
  tags, size, color, manual) and list styles (default, comfortable, condensed, preview, grid).
- Multi-select with swipe actions (copy / star / delete) and double-click actions.

### 2. Creating notes & adding files
- **New Note**, **Scan Barcode** (save links/text from most barcode formats),
  **Copy from Clipboard**, **Import from File** (size-limited).
- **Add files**: choose file, take photo, record video; attach to notes or the Files area.

### 3. Clipboard integration
- **Notes from Clipboard**: background monitoring of the clipboard that auto-creates notes
  (requires the Clipboard rune; shows a persistent notification).
- Notification companion with clipboard history / actions and pause/resume monitoring.
- Android 10 (Q) restrictions handled via the "Clipto…" `PROCESS_TEXT` action, paste from
  the notification, or ADB-granted permissions (guided in-app).

### 4. Organisation
- **Tags** — colour-coded, many per note, per-tag display settings (synced).
- **Folders** — hierarchical structure, one folder per item, deep or shallow delete.
- **Filters** — saved search templates plus an **Advanced Search** screen (text, text type,
  category, tags, created/updated dates, public links, attachments, unsynced).

### 5. Snippets & Snippet Kits
- Reusable static or dynamic text templates that update everywhere they are used.
- Organised into kits; kits can be shared, installed from a public link or the public
  library, published, updated, and uninstalled.

### 6. Dynamic values / fields
- Insert runtime values into notes/snippets: date-time, random number, clipboard,
  device info, snippet references, and interactive fields (text, number, select, toggle)
  resolved at action time.

### 7. Actions on a note ("Quick Actions")
- Compose email/SMS/Tweet, send message, web/Wikipedia/YouTube/DuckDuckGo/StackOverflow/
  Reddit/Google Maps/Translate search, playback link, dial number, add calendar event,
  insert contact, copy/clear clipboard, text-to-speech, share, send as TXT/PDF/Markdown/QR
  code/file, export to file, shorten link.

### 8. Public links
- Per-note public links with optional access password (+hint), access delay, expiry time,
  and one-time opening; copy/share/remove link.

### 9. Skill Runes (feature toggles)
Modular add-ons the user activates: Clipboard, Iron Shield (passcode lock + fingerprint),
Chameleon Skin (themes), Astral Sync (instant cloud sync), Keyboard Companion (Texpand-style
floating panel to insert notes into other apps via accessibility service), The Oculus (link
preview), Point of No Return (auto-save), Revive (remember last filter), Swipe Power,
Shifted Focus, Blowing Copy (hide on copy), Tentative Steps (double-click actions),
Inspiring Support, Time Machine (backup/restore), and Hotkey Observer (in-app/global hotkeys).

### 10. Security & privacy
- Passcode lock with inactivity timeout and fingerprint unlock.
- Themes (light/dark/AMOLED and colour variants), configurable language, run-at-startup.

### 11. Backup, restore & sync
- **Backup to File / Restore from File** (multiple formats).
- **Account** sign-in for cloud sync; tiered **sync plans** (Free + paid note-sync tiers),
  contributor plan, automatic sync, rebuild data index, delete account. App remains fully
  functional offline when the sync limit is reached.

### 12. Cross-platform / integrations
- Desktop companion (Windows/Linux/macOS) with tray, shortcut keys, and download menu.
- Zapier integration ("Text Copied to the Clipboard" trigger) and a public snippet library.

## Notable entry points in code
- `clipto.AppContainer` — main `singleTask` activity hosting the nav graph.
- `clipto.presentation.contextactions.ContextActionsActivity` — the `PROCESS_TEXT`/`SEND`
  "Clipto…" handler for sharing text/files into the app.
- `clipto.presentation.main` — main list, actions, navigation.
- `clipto.presentation.clip`, `.../file`, `.../snippets`, `.../folder`, `.../filter`,
  `.../runes`, `.../settings`, `.../account`, `.../plan`, `.../lockscreen`.
- `clipto.action.*` — domain actions (save/copy/delete clips, kits, filters, sessions).

## External domain pairing

**Yes — the app is hard-wired to its own external domain `clipto.pro` and to the Firebase
project `wb-clipto`. It is not self-hostable/generic out of the box.**

### 1. `clipto.pro` — the app's own site/domain
Declared as Android App Links (`android:autoVerify="true"`) in `AndroidManifest.xml`, and
used everywhere via `buildConfigField`s in `app/build.gradle`:

| BuildConfig field | Value |
|---|---|
| `kitLink` | `https://clipto.pro/#/kit` (snippet-kit install deep links) |
| `referralLink` | `https://clipto.pro/#` |
| `appSiteLink` | `https://clipto.pro/#/editor` |
| `appDownloadLink` | `https://clipto.pro/#/download` |
| `supportProductLink` | `https://clipto.pro/#/pricing` |
| `appAuthLink` | `https://clipto.pro/#/__auth_m` (web sign-in) |
| `privacyPolicyUrl` | `https://clipto.pro/#/policy` |
| `tosUrl` | `https://clipto.pro/#/terms` |

Manifest deep links: `clipto.pro` host with `pathPrefix="/kit"` (kit install), plus a custom
OAuth callback scheme `oauth-callback://clipto`. `AppUtils.fromSnippetKitUri` /
`fromReferralUri` parse incoming `clipto.pro` links.

### 2. `clipto.page.link` — Firebase Dynamic Links
`android:host="clipto.page.link"` (http/https, autoVerify) and `BuildConfig.dynamicLink`
are used for shareable/expiring note links and the `generateAppLink` cloud function.

### 3. Firebase project `wb-clipto` (the actual backend)
From `app/google-services.json`:
- `project_id`: **wb-clipto**
- `firebase_url`: `https://wb-clipto.firebaseio.com`
- `storage_bucket`: `wb-clipto.appspot.com`
- Apps: `com.wb.clipboard.debug`, `com.wb.clipboard.pro`
- API key: `AIzaSyCQVM4P1xcIJtrWkOTmAoMVcTlf4rc_Z_Y` (also reused as `youtube_data_api_key` in remote config)

All server communication goes through this project: **Firestore** (notes/files/filters,
per-user collections), **Cloud Storage** (attachments), **Firebase Auth**, **Remote Config**,
**Analytics/Crashlytics**, and **Cloud Functions** (`FunctionsFunctionsHelper` →
`FirebaseFunctions.getInstance().getHttpsCallable(...)`). Cloud-function names called by
`Api.kt`: `startSession`, `checkUserSession`, `deleteAccount`, `generateAppLink`,
`getLinkShortenUrl`, `notePublicLinkGenerate`, `notePublicLinkRemove`, `filePublicLinkGet`.

### 4. Runtime-overridable links (Firebase Remote Config)
`res/xml/remote_config_defaults.xml` points most instructional/support links at `clipto.pro`
(ADB instruction, notification-paste instruction, global-copy instruction, FAQ, invite/reward,
changelog, snippet-kit bonus, etc.), plus social/community domains: `github.com`,
`reddit.com/r/cliptopro`, `facebook.com/cliptopro`, `discord.gg`, `crowdin.com`, `play.google.com`,
`dontkillmyapp.com`.

### 5. Third-party service domains (feature-specific, not app backend)
`google.com` (search/maps/translate), `youtube.com` + `youtube.googleapis.com` + `googlevideo`,
`twitter.com`, `reddit.com`, `stackoverflow.com`, `duckduckgo.com`, `wikipedia.org`,
`player.vimeo.com`, `coub.com`, `aparat.com`, `zendesk.jfrog.io` (Gradle repo), and Zapier.

### Implication for this fork
The GitHub remote is `almadorx/clitpto-android-fork`. Because the backend and deep links are
pinned to `wb-clipto` / `clipto.pro`, a self-hosted fork must (a) supply its own
`google-services.json` + Firebase project (Firestore rules, Storage, the Cloud Functions
above, Auth, Remote Config), and (b) replace all `clipto.pro` / `clipto.page.link` links and
the manifest App Link hosts. Without that, the app will talk to the original Clipto servers.

## Backend / maintenance status (verified live)

A read-only probe of the original infrastructure (see `BACKEND_REBUILD.md` for the full table)
shows:

- **`wb-clipto` Firebase project is still alive** — Firebase Hosting (`wb-clipto.web.app`),
  Auth, Firestore, Storage, RTDB and Remote Config (149 live keys) all respond; `assetlinks.json`
  is served with the real signing certs.
- **The app is delisted from Google Play** (`com.wb.clipboard.pro` → 404; a control app returns
  200), and **Cloud Functions are partially degraded** (most return 500/503; `startSession` /
  `checkUserSession` return 404).
- **`clipto.pro` website is up; `clipto.page.link` is down** (Firebase Dynamic Links shut down
  2025-08-25).

Net: the app is abandoned/delisted but its backend is **not** out of reach — it is running but
effectively unmaintained. Data is intact behind auth and cannot be read without project
credentials. See `BACKEND_REBUILD.md` for rebuild/hosting guidance.


