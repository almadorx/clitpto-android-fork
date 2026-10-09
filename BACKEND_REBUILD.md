# Rebuilding the Clipto backend on your own domain/hosting

## Verdict

**Yes, it is technically feasible — but with important caveats.**

- The repo contains the **complete client↔server contract**: Firestore collection layout,
  every document field name/type, Cloud Storage paths, auth providers + custom-token scheme,
  and the exact names of every callable Cloud Function. That is enough to **recreate the
  structure** of the backend exactly.
- The repo contains **no server-side code**: no Cloud Functions source, no `firestore.rules`,
  no `storage.rules`, no `firebase.json`, no `.firebaserc`, no indexes. Server logic must be
  **re-implemented** from the client's expectations.
- The repo contains **no user data**. If the original Firebase project is gone, existing
  cloud data is not recoverable from this repo (users can only re-import their own
  in-app "Backup to File" exports).
- Several dependencies are **Google-only and/or dead**: Firebase Dynamic Links
  (`clipto.page.link`) was **shut down on 2025-08-25**, and Remote Config / Analytics /
  Crashlytics / FCM have no drop-in non-Google equivalent without client changes.

## What the client expects (the contract we DO have)

### Firestore collections (from `UserCollection.kt` + `FirebaseDaoHelper`)
| Path | Purpose |
|---|---|
| `u/{uid}` | user document |
| `u/{uid}/c` | active clips (notes) |
| `u/{uid}/cd` | deleted clips |
| `u/{uid}/f` | filters |
| `u/{uid}/fs` | files (metadata) |
| `u/{uid}/fsd` | deleted files |
| `n/{uid}/n` | public notes |

Sync is timestamp-based: every doc carries `iid` (device id), `av` (api version) and
`cts` (server change timestamp); listeners do `whereGreaterThan("cts", lastSeen).orderBy("cts")`.

### Document fields
All field names are declared in `FirebaseDaoHelper` (`ATTR_*`) and read/written by the
mappers in `clipto/dao/firebase/mapper/*`. Examples: clip `n`(usage count), `c`/`m`/`u`/`d`
(dates), `ot`(object type), `v`(text type), `h`(title), `t`(text), `li`(tag ids), `fi`(file
ids), `f`(fav), `s`(snippet id), `fid`(folder id), `pl`(public link), `l`(tags). Filters and
files have their own sets. This is a **complete, replicable schema**.

### Cloud Storage layout
- Files: `{uid}/{folder}/{name}`
- Thumbnails: `{uid}/{folder}/thumbs/{uid}_200x200`
- Uploaded with `putFile(...)` and custom metadata (`name`, `type`, `av`); download via
  `downloadUrl`.

### Auth
- Firebase Auth via FirebaseUI with **Google, Facebook, Email, Phone** providers.
- **Custom token** path: `FirebaseAuth.signInWithCustomToken(token)`; the web flow returns a
  token in the URL (`...?c=<AES-encrypted>`) decrypted with a **hardcoded AES key in
  `AppUtils.kt`** — so the web-auth token scheme is reproducible.


### Callable Cloud Functions (full inventory)
`startSession`, `checkUserSession`, `deleteAccount`, `generateAppLink`, `getLinkShortenUrl`,
`getLinkPreview`, `getLinkPlaybackUrl`, `notePublicLinkGenerate`, `notePublicLinkRemove`,
`filePublicLinkGet`, `snippet_kit_category_list`, `snippet_kit_list`, `snippet_kit_get`,
`snippet_kit_publish`, `snippet_kit_discard`, `snippet_kit_link_create`,
`snippet_kit_link_remove`, `snippet_kit_update`, `snippet_kit_install`, `snippet_get`,
`user_data_upgrade`.

Server-enforced behaviour to reproduce: session/plan checks (`startSession`,
`checkUserSession` return `syncLimit`, `plan`, subscription ids), public-link generation with
password/expiry/one-time rules, snippet-kit public library, referral link generation, URL
shortening, and link previews.

### Remote Config
129 keys, all with **bundled defaults** in `res/xml/remote_config_defaults.xml`
(e.g. `sync_plan_notes_free_limit=300`, `sync_plans`, `api_read_timeout`, `url_*`). Because
defaults ship in the APK, the app still works if Remote Config is absent; you only lose
server-side tuning.

## What you must supply / re-implement

1. **Cloud Functions** (all 21 above) — logic inferred from client request/response shapes.
2. **Firestore security rules** — none in repo; write your own (per-user isolation under `u/{uid}`).
3. **Storage rules** — none in repo.
4. **Composite indexes** for the `cts` queries.
5. **Dynamic Links replacement** — `clipto.page.link` is dead. Use Android App Links /
   Universal Links (with `.well-known/assetlinks.json` / `apple-app-site-association`) or a
   third party (Branch, etc.). This is a **client-side change** too.
6. **Web endpoints** on your domain: auth callback, kit landing page, download page,
   policy/terms — currently all `https://clipto.pro/#/...`.
7. **OAuth/SMS credentials** for Google/Facebook/email/phone sign-in.
8. **Object storage** for attachments and **a server runtime** for functions.
9. Optional: Remote Config, Analytics, Crashlytics, FCM equivalents.

## Hosting options

| Option | Effort | Notes |
|---|---|---|
| **A. New Firebase project** (recommended) | Lowest | Same SDKs; swap `google-services.json`, enable Auth/Firestore/Storage/Functions/Remote Config, port the 21 functions (Blaze plan needed for Functions). |
| **B. Firebase + self-hosted** | Medium | Firebase Auth/Firestore on Google, but your own functions/hosting/domain. Still Google-dependent for data. |
| **C. Full re-platform** (Supabase / Appwrite / Nhost / PocketBase / custom) | High | Requires rewriting the entire `clipto.dao.firebase.*` layer + `Api.kt`; not a config swap. Gives true independence. |
| **D. Firestore emulator / compatible server** | n/a | Emulator is dev-only, not production. No maintained Firestore-compatible OSS server exists for the mobile SDKs. |

### Hosting requirements (minimum)
- A **domain** (replace `clipto.pro`) and a link subdomain; DNS control.
- **HTTPS** hosting for static pages + serverless functions (Firebase Hosting, Netlify,
  Vercel, Cloudflare, or nginx).
- `/.well-known/assetlinks.json` served for App Links verification (the manifest uses
  `autoVerify="true"`).
- Valid **TLS certificates**.
- **Auth** provider setup (Google OAuth client, Facebook app, email, SMS).
- **Database** (Firestore or an alternative) and **object storage**.
- **Function runtime** (Node/Go/Python or serverless).

## Client-side changes required for a fork

1. Replace `app/google-services.json` with your new project (project_id, api_key, app id,
   storage bucket, messaging sender). Note: the current `AIzaSy...` key is a **public client
   key**, not a secret — rotate regardless.
2. Update `app/build.gradle` `buildConfigField`s (`kitLink`, `referralLink`, `appSiteLink`,
   `appDownloadLink`, `supportProductLink`, `appAuthLink`, `privacyPolicyUrl`, `tosUrl`,
   `dynamicLink`).
3. Update `AndroidManifest.xml` App Link hosts (`clipto.pro`, `clipto.page.link`) and the
   `oauth-callback` scheme.
4. Update `res/xml/remote_config_defaults.xml` URLs.
5. Optionally change the AES key in `AppUtils.kt` (must match your web-auth server).
6. Re-point analytics/crash reporting or strip them.

## Do we have enough data to rebuild the Firebase database?

- **Structure/schema: yes — completely.** Collections, field names, types, storage paths,
  queries and function signatures are all derivable from the client code.
- **Existing user content: no.** That lives on Google's infrastructure. If the maintainer
  abandoned the project, data may still exist in the old `wb-clipto` project (Google does not
  delete projects immediately), but billing lapse would have disabled Functions/Firestore
  writes. Users can still recover their own data via the in-app **Backup to File** export and
  re-import it into a rebuilt backend.
