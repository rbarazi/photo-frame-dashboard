# Updating a kiosk without reconnecting USB

This is a design and acceptance guide for adding Wi-Fi updates to your own kiosk.
**The upstream kiosk does not currently download or install updates.** It loads a
hosted website and provides a **Reload dashboard** control. This guide adds no
updater, server, APK download, or installation command.

## Three different kinds of update

| Change | What needs to happen on the frame |
| --- | --- |
| Data, such as a calendar or daily activity | The website fetches it, if its application implements that refresh. |
| Website HTML, JavaScript or CSS | Deploy the website, then reload or navigate the main page. A page already open may still run the old JavaScript. |
| Native Android behavior | Install a newer, compatible APK signed for the same app, then verify the app actually returns. |

In the [current kiosk](../dashboard/kiosk/src/com/wesbos/homehq/MainActivity.java),
hold two fingers still for 1.5 seconds and choose **Reload dashboard**. The network
error retry is not periodic release detection. Publishing new source or seeing a
successful web deployment does not prove that a frame has loaded that release.

## Observed on one customized frame

A customized kiosk on a Frameo-based frame running Android 8.1 and WebView 61
was tested with a macOS host available for diagnostics. Its own update flow
downloaded and installed a newer APK over Wi-Fi. The target APK was not installed
through ADB, and delivery did not use a
USB network tunnel. The dashboard returned with the saved URL and login intact.
Cancellation, retry and installation-permission handling were also exercised
during the preceding update rehearsal.

Those observations concern a downstream implementation, not the code in this
repository. They do not establish compatibility with every Frameo device or
Android version. The design requirements below are a checklist for an
implementation; they are not claims that every failure case was tested on that
unit. Private hosting, credentials, signing material and diagnostic records are
intentionally excluded.

## Keep the identity of the installed app

Before implementing delivery, record the installed package name, version code,
minimum supported Android version and signing certificate in your private release
record. Keep a recoverable copy of your own signing key outside the repository.
For these older frames, retain the same signer for updates to the same package;
do not assume newer key-rotation mechanisms work. Android checks signing identity
when replacing an installed app. See [Android app signing](https://developer.android.com/studio/publish/app-signing).

The repository's [shell build](../dashboard/kiosk/build.sh) and
[PowerShell build](../dashboard/kiosk/build.ps1) currently use the host's Android
debug keystore and hard-code version code `2`, version name `1.1`, and minimum and
target API `23`. A fresh machine can generate a different debug key. Another
person's build is therefore not necessarily an update to your installed copy.
Plan release signing before distributing a maintained build; changing the signer
of an existing installation is not a routine update.

Give each release a higher integer `versionCode`; a different displayed version
name alone is insufficient. Check the APK's minimum API against the device.
Android normally rejects downgrades. See [Android versioning](https://developer.android.com/studio/publish/versioning).

## Design a bounded download and installation flow

These are requirements for a future native updater, not settings present in the
current kiosk. No JavaScript bridge is required; a native control can start it.

1. **Publish an explicit release.** The owner-controlled release record should
   identify the package, version code, minimum API, APK byte length, SHA-256 and
   expected signer. Publish the APK and record through a trusted HTTPS channel.
   A matching hash proves byte integrity, not who authorized a release.
2. **Keep access narrow.** If the download is private, use credentials scoped to
   reading that app's releases. Keep them out of the page, URL, logs and public
   repository. Restrict downloads and redirects to approved origins; never
   forward credentials to an arbitrary redirect destination.
3. **Download before offering installation.** Use app-private temporary storage,
   a maximum byte count, connection/read timeouts and cancellation. Check available
   space. Reject an oversized, truncated or non-APK response, including an HTML
   login page. Remove incomplete files and keep the running app usable.
4. **Validate the finished APK.** Verify its signature and compare its package,
   signer, version, minimum API, size and hash to the approved release and installed
   app. Stop on any mismatch. Parsing metadata is not signature verification; use
   a verifier compatible with the Android versions supported. Android's installer
   remains the final installation authority.
5. **Let Android request consent.** Hand the validated file to the system installer
   using a version-appropriate API. If using an intent, provide a restricted
   `content://` URI with temporary read permission, not a broadly shared file or
   your app's private directory. See [secure file sharing](https://developer.android.com/training/secure-file-sharing/share-file).
6. **Handle all exits.** Show a recoverable state for denied permission, Cancel,
   failed installation and a retry. Do not call a download or an opened installer
   “installed.” Read the installed version after replacement and verify navigation
   back into the app.

Android 8.0 introduced source-specific installation trust. Its
[`canRequestPackageInstalls()` API](https://developer.android.com/reference/android/content/pm/PackageManager#canRequestPackageInstalls())
requires a target of API 26 or later and `REQUEST_INSTALL_PACKAGES`; do not paste
that flow into the current target-23 build without reviewing the target change.
Older versions and vendor settings differ. A `PackageInstaller` implementation
must handle [pending user action and failure results](https://developer.android.com/reference/android/content/pm/PackageInstaller#STATUS_PENDING_USER_ACTION).
Do not bypass the installer's consent screen or enable a permanent network-debug
service just to deliver routine updates.

## Preserve the way home

Treat installation, returning to the dashboard, and retaining its state as separate
acceptance checks. A successful replacement can kill the old process. Save only
the state needed to resume, and establish a return path that works with the unit's
installer and launcher. Do not assume every Android version will automatically
bring the replacement app to the foreground.

The current upstream kiosk accepts an intent URL but does not persist it for a
fresh launch, and is not a default Home app. Saved URLs and Home behavior are being
discussed separately in [PR #40](https://github.com/wesbos/photo-frame-dashboard/pull/40)
and [PR #43](https://github.com/wesbos/photo-frame-dashboard/pull/43). Review their
current status before building on them; this guide does not implement either
proposal. The existing [navigation service](../dashboard/kiosk/README.md#frames-without-home-or-back-buttons)
is another return aid to verify on the actual unit.

Do not uninstall the app, clear its data, replace its package name or clear WebView
storage to work around a failed update. These can discard the saved URL, login or
other state. Preserve both local settings and any independent server-side data.

## Prove the next update before unplugging

Keep a working recovery connection available for the first rehearsal. The new
APK itself must travel over Wi-Fi, with no USB install or tunnel, to validate that
delivery path. Record each result independently:

- The previous app remains usable after a canceled download and canceled installer.
- A refused installation permission has a clear, usable return and retry path.
- Wrong package, signer, hash, incompatible API and older version are rejected.
- Interrupted downloads and expired download access recover without clearing data.
- The intended higher version is installed, the dashboard returns, and a fresh
  launch opens the intended URL with the expected login and settings.
- On frames without system navigation, the installer and settings have a tested
  way back. Confirm the existing Home choice remains correct if one was configured.
- A second check reports the installed version accurately and offers no duplicate
  update. Website refresh is tested independently from native installation.

Keep screenshots and raw diagnostic output private. A shareable report can name
the Android/WebView versions, host OS, tested paths and limitations without
including account details, network identifiers or family content.

## Recovery limits

Before installation, Cancel should leave the current app in place. After a bad
release, a usual repair is the previous working source rebuilt with the same
package and signer and a **higher** version code, provided its storage format is
still compatible. Do not promise that an older APK can be installed over it or
that uninstalling is a safe rollback.

Wi-Fi updates depend on a working app, network, download service and installer.
If the app cannot start, the signing key is lost, or the network is unreachable,
this path may not recover the device. Keep the device-specific recovery notes and
signing backup accessible. A tested updater reduces routine USB visits; it cannot
guarantee that physical access will never be needed.
