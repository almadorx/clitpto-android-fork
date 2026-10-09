# Clipto (Android) — repository review notes

Review/analysis documents for the **Clipto** Android app (`com.wb.clipboard`, module `app`,
package `clipto`) — an offline-first **notepad + clipboard manager**. This repo (the
`cliptpto-android-fork` checkout) shipped **no original README**; the notes below were derived
from the source, resources and a read-only probe of the original backend.

## Documents

| File | Contents |
|---|---|
| [`REVIEW.md`](REVIEW.md) | User-facing feature review, architecture overview, code entry points, external-domain pairing. |
| [`BACKEND_REBUILD.md`](BACKEND_REBUILD.md) | Full client↔server contract (Firestore/Storage/Functions/Auth), rebuild & hosting options, and the **live reachability probe** of the `wb-clipto` project. |
| [`LOCAL_VAULT_AND_DESKTOP.md`](LOCAL_VAULT_AND_DESKTOP.md) | Replacing Firebase with a **local folder vault** (Dropbox/OneDrive) and the **lowest-cost Windows desktop** options. |

## TL;DR status

- **App:** abandoned/delisted — `com.wb.clipboard.pro` is **no longer on Google Play**.
- **Backend:** the original **`wb-clipto` Firebase project is still alive and reachable**
  (Hosting/Auth/Firestore/Storage/Remote Config active), but Cloud Functions are **partially
  degraded** and `startSession`/`checkUserSession` are gone → running but effectively
  unmaintained. Data is intact behind auth and cannot be read without project credentials.
- **Domains:** `clipto.pro` (app site) is up; `clipto.page.link` (Dynamic Links) is **down**
  (service retired 2025-08-25).
- **Architecture:** local-first on **ObjectBox**; **Firebase is only the sync/transport layer**
  (~26 files); a JSON import/export "vault" format already exists; UI is classic Android Views.
- **Desktop app:** exists (`clipto-pro/Desktop`, latest **v7.2.17, Nov 2021**) but the repo is
  **binary-only (no source)**; it's an **Electron** app, **cloud-sync-only** (no local
  import/export), stores data as Chromium **LevelDB** under `%APPDATA%\Roaming\Clipto`, and the
  maintainer is **MIA**. See `LOCAL_VAULT_AND_DESKTOP.md` §3.


## Repository layout (high level)

```
app/                  Android app (Kotlin, Hilt, ObjectBox, Firebase)
  src/main/java/clipto/
    dao/objectbox/    local DB (ObjectBox) — the source of truth
    dao/firebase/     Firestore/Storage/Functions sync layer
    repository/       Clip/File/Filter/User repositories (call the sync layer)
    api/              server API (Firebase Functions)
    presentation/     screens (Views + viewBinding)
    domain/           domain model (~80% JVM-portable)
    backup/           JSON vault import/export
common-presentation/  shared Android UI helpers
converter/            pure-JVM tool (builds a non-Android artifact)
```
