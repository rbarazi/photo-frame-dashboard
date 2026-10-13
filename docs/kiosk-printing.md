# Printing from an Android WebView kiosk

This is an implementation and testing guide, not a printing feature in the
current Home HQ kiosk. Its [`MainActivity`](../dashboard/kiosk/src/com/wesbos/homehq/MainActivity.java)
does not yet create print adapters or open Android's print UI.

The observations below come from one privately customized Frameo deployment on
Android 8.1 with WebView 61, managed from macOS. They do not establish support on
the stock upstream kiosk or every Frameo device. The hardware model, firmware
string and SoC were not recorded for this report; do not use it as a device
identification or flashing guide.

## What was observed

| Check | Result on that customized deployment |
| --- | --- |
| Native print preview | Opened with the printable artwork |
| Existing Android print service | Discovered and selected a printer over the local network |
| Cancel and return | Returned to the printable page and then the dashboard |
| Submission | One single-page test job was submitted |
| Paper output | Not confirmed: the printer reported paper needed |

No additional print service was installed for that test. An empty
`enabled_print_services` setting initially led to the wrong conclusion that no
service was available. Inspect the installed services and Android's UI before
deciding a driver is missing. A queued job is not proof that paper came out;
do not keep submitting copies to diagnose it.

## First inspect what the frame already has

Read the [contribution and device-safety rules](../README.md#for-agents-start-here)
before changing the device. Record its Android and WebView versions, and keep a
working way back to the dashboard before opening another app's settings. The
[kiosk navigation guide](../dashboard/kiosk/README.md) describes the upstream
Back/Dashboard controls.

1. Open Android Settings and look for **Printing**. Menu names and available
   services vary by firmware. Record which services are installed and enabled.
2. If the UI is inconclusive, inspect `adb shell dumpsys print` and the installed
   packages with `adb shell pm list packages`. These are read-only diagnostics;
   keep their raw output private because it can include printer and job details.
3. Try the existing service first. Confirm the frame and printer can reach each
   other on the local network; internet access alone does not prove this.
4. If a service is truly absent or incompatible, stop and identify a compatible
   official option before installing anything. Check its Android minimum version,
   follow the repo's download/hash rules, and get the owner's approval. This guide
   requires no extra APK, account, cloud print service or vendor agreement.

Android distinguishes the app producing a document from the print service that
discovers printers and transports jobs. Enabling a service is a user action in
system settings. See the official [PrintService documentation](https://developer.android.com/reference/android/printservice/PrintService).

## Proposed native integration

For an implementation PR, use a dedicated HTML print preview and a native
**Print this page** control. Show **Back to dashboard** on the preview. The page
should contain only the intended artwork or document, with comfortable margins.
Opening it must not submit a job or open the system dialog automatically.

Android's HTML printing API uses `WebView.createPrintDocumentAdapter(...)` and
`PrintManager.print(...)`. A web page's `window.print()` is not a substitute:
Android documents that HTML JavaScript cannot trigger WebView printing. Wait for
the document to finish loading; a WebView supports one print job at a time. See
[Printing HTML documents](https://developer.android.com/training/printing/html-docs).

The proposed sequence is:

1. Load the trusted print-preview page. Disable the native print control until
   the page and its artwork are ready; a load failure should show a retry action.
2. After the user taps the native control, create the adapter from the loaded
   WebView and call `PrintManager.print(...)` from the Activity.
3. Keep the Activity, WebView and adapter valid while Android needs them. Prevent
   a second request on that WebView, and prevent page navigation from replacing
   its content during document preparation. A separate print WebView is another
   option if the dashboard must stay interactive.
4. Let the user choose printer, page size, colour and copies in Android's dialog.
   Do not automate its final Print action.
5. Handle Cancel and failure, release the busy state, and allow an explicit retry.
   Returning to the Activity is not itself evidence of success.

`PrintManager.print()` returns before the user confirms the job, and finishing
the Activity invalidates its adapter. Once queued, the document belongs to the
system spooler. See [PrintManager](https://developer.android.com/reference/android/print/PrintManager).
Adapter cleanup must follow its lifecycle; `onFinish()` means the adapter is no
longer needed, not that a physical sheet was printed. See
[PrintDocumentAdapter](https://developer.android.com/reference/android/print/PrintDocumentAdapter).

### Restrict which page can be printed

This design needs no JavaScript-to-native bridge. The web link opens the preview;
the native control checks the current page and initiates printing. Treat these
checks as requirements for a future implementation, not existing kiosk behavior:

- Parse the URL and require the exact configured HTTPS origin: scheme, hostname
  and effective port. Reject user-info, malformed URLs and other schemes.
- Allow only a documented print route, for example `/print/coloring/<asset-id>`
  with a bounded asset identifier. Do not match URLs with a hostname substring or
  a broad path prefix. Reject unexpected query parameters, encoded separators
  and traversal segments rather than interpreting them differently at each layer.
- Validate the main-frame destination on navigation and after redirects, then
  revalidate immediately before creating the adapter. A frame inside the page
  must not trigger native actions.
- Use a fixed same-origin dashboard return link. Do not accept an arbitrary
  `returnUrl`, open an external browser, forward credentials to another origin
  or include credentials in URLs or diagnostic logs.

Android's [unsafe URI loading guidance](https://developer.android.com/privacy-and-security/risks/unsafe-uri-loading)
explains why parsed scheme and complete-host checks matter. Add tests for the
application's stricter origin and route policy before shipping it.

## Acceptance checks on the actual frame

Use a non-personal sample page, with no calendar events or family photographs.
Start with preview and Cancel checks; submit one physical job only when the owner
is ready and the printer has paper.

| Test | Evidence to record |
| --- | --- |
| Page load | Artwork appears; failed/slow loads cannot print a blank page |
| Native preview | Page count, margins, paper size and orientation are correct |
| Cancel | No submitted job; preview and dashboard remain usable |
| Repeated taps | One dialog/job flow, without duplicate submissions |
| Navigation | Back and dashboard return work, including on frames without system navigation |
| Printer discovery | Intended printer appears and can be selected |
| Submission | Exactly one intended job enters the queue |
| Paper output | Owner sees one correct sheet; record this separately from queue status |
| Failure/retry | Paper-out or offline status is visible; retry does not silently add another job |
| URL checks | Wrong origin, unexpected port/path and redirect cannot reach the native print action |

Test again after changing the kiosk or WebView. A desktop browser's print preview
does not verify the frame's adapter, service, printer discovery or navigation.
When reporting an unfinished test, name the last observed stage. Android exposes
separate queued, blocked, failed, cancelled and completed job states through
[PrintJob](https://developer.android.com/reference/android/print/PrintJob).

## Undo and recovery

- Before confirmation, cancel Android's print dialog and use the preview's return
  link. If a job was already submitted, inspect and cancel that specific job in
  Android's print queue; closing the preview does not remove it. Paper already
  printed cannot be undone.
- If you enabled an existing service for the trial, record its previous state and
  restore it in Settings when finished. Do not remove a preinstalled service.
- For an implementation trial, keep the previous source and signing key private,
  and document the known-working app update/recovery route before installation.
  Revert the printing changes in source and build a compatible signed update if
  rollback is needed; uninstalling the kiosk may erase its saved settings.
- Remove or hide a web print entry if native support is unavailable. Do not leave
  a button that appears to work but never opens a preview.

This documentation change installs nothing and requires no device write.
