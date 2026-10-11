# Analysis: Accessibility usage & Android 10+ clipboard handling

This is the **Clipto** clipboard-manager app (`com.wb.clipboard`). Answers below are
grounded in the actual source with file:line references.

---

## 1. Does the app use Accessibility? — Yes

The app ships a full **`AccessibilityService`** used by the "Keyboard Companion" rune.

| What | Where |
|------|-------|
| Service class | `app/src/main/java/clipto/presentation/runes/keyboard_companion/CompanionService.kt` (`class CompanionService : AccessibilityService()`) |
| Base helper | `common-presentation/src/main/java/clipto/common/presentation/mvvm/base/LifecycleAccessibilityService.kt` |
| Manifest declaration | `app/src/main/AndroidManifest.xml` lines ~234-248: `BIND_ACCESSIBILITY_SERVICE` + `<meta-data android:name="android.accessibilityservice" ...>` |
| Config XML | `app/src/main/res/xml/texpander_service_config.xml` |
| Quick-settings tile | `CompanionTileService` (same manifest block, `BIND_QUICK_SETTINGS_TILE`) |
| Permission gate | `PermissionsManager.isAccessibilityEnabled(...)` / `requestAccessibility(...)` (`app/src/main/java/clipto/PermissionsManager.kt:26,38-41`) |

The service config (`texpander_service_config.xml`) requests:

```xml
android:accessibilityEventTypes="typeViewClicked|typeViewFocused|typeViewTextSelectionChanged|
                               typeViewSelected|typeViewTextChanged|typeWindowStateChanged|typeContextClicked"
android:canRetrieveWindowContent="true"
android:notificationTimeout="100"
```

So the app can **read the window content of other apps** and observe clicks, focus,
text-selection and text changes.

**What it's used for** (`CompanionService.onAccessibilityEvent`, `CompanionPanel`):
detecting the focused/selected input field in *any* app and showing a draggable
floating panel (added via `WindowManager` with `TYPE_APPLICATION_OVERLAY`) that lets
you insert/replace/undo/redo text from your Clipto notes. The actual field edits are
done through accessibility actions in
`app/src/main/java/clipto/extensions/androidx.core.view.accessibility.AccessibilityNodeInfoCompat.kt`:
`ACTION_SET_TEXT`, `ACTION_SET_SELECTION`, `ACTION_FOCUS`, `ACTION_COPY`, `ACTION_PASTE`.

Panel states that drive this: `EditTextPanelState`, `EditTextContextPanelState`,
`DetachedPanelState` (`.../keyboard_companion/panel/`).


---

## 2. How the clipboard problem on Android > 10 (Q+) is solved

**The problem:** since Android Q (API 29) only the app that currently has *focus* may
read clipboard data, so a background clipboard listener no longer receives content.

The app has three layers of handling:

### a. Native tracking only on pre-Q
`ClipboardState.isClipboardSupportedNatively() = AndroidUtils.isPreQ()` (`ClipboardState.kt:138`,
`AndroidUtils.kt:32` → `SDK_INT < Q`). `ClipboardStateManager` picks its strategy based
on this (`ClipboardStateManager.kt:41-47`):

```kotlin
private val adapter by lazy {
    if (clipboardState.isClipboardSupportedNatively()) ClipboardStateManagerDefaultAdapter(this)
    else                                             ClipboardStateManagerAdbAdapter(this)
}
```

* `ClipboardStateManagerDefaultAdapter` → classic `addPrimaryClipChangedListener`
  (`ClipboardStateManagerDefaultAdapter.kt`).
* `ClipboardStateManagerAdbAdapter` → the Q+ workaround (below).

### b. The Q+ workaround = "Advanced clipboard tracking" (ADB setup)
`ClipboardStateManagerAdbAdapter.kt` combines **two** tricks:

1. **Read the clipboard from logcat.** `getClipFromLogcat()` runs
   `logcat -c` then `logcat -s ClipboardService:E` and filters lines that contain the
   app's package name (`ClipboardStateManagerAdbAdapter.kt:100-120`). Needs the
   `READ_LOGS` permission.
2. **Overlay window to gain focus.** `AdvancedClipboardView` is a zero-size
   `FrameLayout` added with `WindowManager` using
   `TYPE_APPLICATION_OVERLAY` + `FLAG_NOT_TOUCHABLE`. The moment it is attached and
   first laid out, the app is considered "foreground", so it may read
   (`AdvancedClipboardViewMode.READ`) or write (`PUT`/`COPY`) the primary clip
   (`ClipboardStateManagerAdbAdapter.kt:122-211`). Needs `SYSTEM_ALERT_WINDOW`.

Availability is gated by `ClipboardState.isClipboardSupportedByAdb()` =
`hasPermission(READ_LOGS) && canDrawOverlayViews()` (`ClipboardState.kt:139-146`).

Both permissions must be granted over ADB; the exact commands are shipped as a
user-facing string (`app/src/main/res/values/strings.xml`, `settings.track_clipboard.q.solution.3.command`):

```
adb -d shell pm grant <pkg> android.permission.READ_LOGS;
adb -d shell appops set <pkg> SYSTEM_ALERT_WINDOW allow;
adb -d shell am force-stop <pkg>;
```

### c. User-facing fallbacks (documented in Settings)
`strings.xml` (`settings.track_clipboard.q.*`) explicitly describe the Q+ situation and
three solutions:

1. **"Clipto…" action instead of "Copy"** — via `Intent.ACTION_PROCESS_TEXT`
   (`ActionProcessTextProvider.kt`), i.e. the app appears in the text-selection context
   menu and captures the selected text directly.
2. **Standard "Copy" + tap "Paste" in Clipto's notification** (`ShowClipProvider.kt`,
   notification action) — reads the clipboard while the app is in the foreground.
3. **The ADB setup above** to restore pre-Q behaviour automatically.

Support summary in `ClipboardState.kt:148-149`:

```kotlin
fun isClipboardSupportedBySomehow(...) =
    isClipboardSupportedNatively() || isClipboardSupportedViaForce() || isClipboardSupportedByAdb(...)
```
where `isClipboardSupportedViaForce()` is just the user ticking "ignore the Q warning"
(`doNotDisplayAndroidQWarning`).

Additional Q-hardening:
* `getPrimaryClip()` is wrapped in try/catch and guarded by `hasPrimaryClip()`, because
  access can throw on Q+ (`ClipboardState.kt:151-162`).
* `canEmulateCopyAction()` (`ClipboardState.kt:130`) — when the clipboard is *not*
  supported "somehow", tapping "Clipto…"/"Paste" immediately creates a note from the
  clipboard text with **no** confirmation dialog (`ShowClipProvider.kt:51`,
  `ActionProcessTextProvider.kt:33`).

---

## 3. Can app data be copied to the clipboard automatically (e.g. FBReader title + author)?

**Writing to the clipboard: yes, fully supported.**
* `CopyClipsAction` calls `clipboardState.clipboardManager.setPrimaryClip(clipData)`
  (`app/src/main/java/clipto/action/CopyClipsAction.kt:92`).
* `ClipboardState.refreshClipboard(internal = true)` re-writes the primary clip, and
  `clearClipboard()` empties it (`ClipboardState.kt:97-122`).
* **Universal Clipboard** (`isUniversalClipboardActivated()`, `ClipboardState.kt:135`)
  automatically places synced/copied notes into the local clipboard where enabled.

**Reading another app's data automatically (FBReader book name + author):**
* The app *does* have the capability in principle, because `CompanionService` runs with
  `canRetrieveWindowContent="true"` and reads node text via
  `AccessibilityNodeInfoCompat.getInputText()` (`CompanionEventInfo.getText()`).
* **However, the current implementation only reads the focused *editable* node**
  (`EditTextPanelState.doTest` requires `isEditable && isFocused && isEditOrAutoComplete`,
  and `getInputText()` only returns real input text). `DetachedPanelState` merely copies
  Clipto's *own* notes to the clipboard ("Copy to Clipboard"). There is **no generic
  "scrape arbitrary text from the current window"** feature.

**Conclusion for the FBReader use case:** auto-copying a book's title/author is *not*
something the app does today out of the box. It would be feasible to implement only by
extending the existing accessibility service to walk the node tree of the FBReader
window (`getRootInActiveWindow()` / `findAccessibilityNodeInfosByViewId`) and match the
title/author `TextView`s, then call `setPrimaryClip`. Alternatively, FBReader itself can
share/select the text and Clipto can capture it through the `ACTION_PROCESS_TEXT`
("Clipto...") flow or the notification "Paste" action - but that still requires a user
gesture, not fully automatic capture.

---

## Quick answers

1. **Accessibility?** Yes - `CompanionService` (a real `AccessibilityService`, config
   `texpander_service_config.xml`), powering the "Keyboard Companion" floating panel and
   text expander in any app.
2. **Android 10+ clipboard fix?** Native listener only pre-Q; on Q+ it (i) reads the
   clipboard out of `logcat` (READ_LOGS) and (ii) uses a `TYPE_APPLICATION_OVERLAY`
   window to momentarily gain foreground focus so the OS allows clipboard read/write
   (SYSTEM_ALERT_WINDOW), plus the "Clipto..." `ACTION_PROCESS_TEXT` action and a
   notification "Paste" button as user-level fallbacks.
3. **Auto-copy app data to clipboard?** Writing to the clipboard is fully supported
   (`setPrimaryClip`, Universal Clipboard). Automatically *scraping* FBReader's
   title/author is **not** implemented - the accessibility service only reads focused
   editable fields, so it would need to be extended to crawl the FBReader window tree.
