# Checking a kiosk release

Use this checklist when changing a dashboard or its Android kiosk. It separates
what a build proves from what someone has observed on the frame. Choose the
checks affected by the change; a text edit does not require reinstalling Android.

This is a maintenance workflow, not a device setup or rooting guide. Follow the
[repository's approval and backup rules](../README.md#rules-we-learned-the-hard-way)
before device writes. It does not add an updater, a backend, printing or sound to
the example dashboard.

## Record the starting point

Identify the checkout and exact source revision, the hosted version if there is
one, and the installed Android package/version. Record the actual Android and
WebView versions and screen dimensions privately. A build's JavaScript or CSS
target is not evidence of which engine the frame runs.

Check the affected behavior before changing it: the dashboard URL, saved sign-in,
navigation, and whether the frame can load the site through its ordinary network
connection. Record how to return to the previous working state. Keep credentials,
private URLs, device identifiers, real household content and screenshots out of
public reports. Reference protected backups; do not copy their contents into a
checklist.

Inspect current upstream changes and open discussions. In particular,
[PR #40](https://github.com/wesbos/photo-frame-dashboard/pull/40) proposes a guide
to building for a frame, and [PR #43](https://github.com/wesbos/photo-frame-dashboard/pull/43)
proposes saved-URL and Home-app behavior. Their unmerged code is not an installed
feature. This checklist concerns acceptance of a particular release and can be
used alongside those proposals.

## Keep evidence at the level actually checked

| Check | What it establishes | What it does not establish |
| --- | --- | --- |
| Source tests and production build | Covered logic works with the tested fixtures and build settings | The hosted site or frame has those files |
| Browser with synthetic data | Layout, navigation and interactions in that browser | Rendering in an older Android WebView |
| Hosted version and authenticated requests | The intended release is served and the checked access rules work | An already-open frame loaded the new JavaScript |
| Physical screen observation | The tested screen and interaction work on that unit | Another model, firmware or untested workflow works |
| Native installer confirmation plus installed version | The intended application update completed | Saved sign-in, restart behavior or automatic kiosk return |
| Print preview and printer discovery | The page composes and a printer is selectable | A job was submitted or paper emerged |
| Submitted print job | The checked system accepted a job | Paper output or print quality |
| Sound preference or audio API success | The control or API path worked | Audible output through the frame's speaker |

Record failures and untested steps as such. A later successful stage does not
rewrite an earlier failure; note the change that resolved it.

## Website changes

1. Run the project's relevant tests and build. Use invented people, events and
   tasks for browser tests. Preview the affected view at the frame's dimensions
   and at supported phone widths; check overflow, long labels, focus and Exit.
2. After an authorized deployment, check the actual served version. If no version
   endpoint exists, compare a built asset with the served asset. Do not add a
   public endpoint that exposes configuration or credentials just for this check.
3. Load the new page on the frame. In the current upstream kiosk, hold two fingers
   for 1.5 seconds and choose **Reload dashboard**. Customized kiosks may have
   different controls; inspect their instructions. Avoid reloading during an
   unfinished form, save or print operation.
4. Observe the affected interaction on the physical screen. If the frame is not
   available, leave physical acceptance pending rather than requiring another
   installation or describing a desktop screenshot as device evidence.

Data polling, full-page navigation and an APK installation are separate actions.
A successful deployment does not force an already-open page to load its new
JavaScript. Conversely, checking for an Android update does not necessarily
reload the website.

## Native changes

Check the package, compatible Android version, signing continuity and replacement
version before the owner-approved installation. Preserve application data and
the existing signing key. Do not uninstall a working kiosk to get past an update
failure. A same-version check cannot demonstrate installation of a newer build.

Test the parts the release changes, including cancellation and recovery where
applicable. If testing a network updater, distinguish downloading and installing
through that updater from installing the same APK with ADB. A USB network tunnel
also changes what a successful connectivity test proves.

After installation, independently check the installed version, dashboard return,
saved URL/sign-in, navigation and the intended Home behavior. If automatic return
is under test, do not launch the activity with ADB before observing it. Keep the
recovery route available. A rollback may require a forward version signed with
the same key and compatible saved data; do not assume Android permits installing
an older version.

## Household features and external effects

Use disposable state to test changes to chores, shopping and calendar settings.
Do not mark a real child's task complete merely to prove a button works. If a
network response is lost after a save, read the current state before repeating
the mutation; a timeout alone does not prove it failed.

For printing, first inspect the preview and Cancel/return path. Submit a physical
job only within the owner's authorized test. If it queues or reports paper out,
inspect that existing job first. Resume it only within the owner’s authorized print
test, rather than repeatedly submitting copies. Keep printer
identifiers and network details private.

For sound or animation, check the preference controls and the actual output
separately. If physical sound cannot be heard or confirmed, report it as untested.
Record whether any new data, credential or device setting was changed for QA.

## Handoff example

The following is a blank record format, not a successful test receipt:

```text
Change and intended behavior:
Source revision:
Hosted revision (or not applicable):
Installed package/version (or not checked):
Private environment record location:
Checks performed, method and date:
Observed failures and recovery:
Existing data/settings preserved:
Remaining untested steps:
Rollback/recovery reference:
```

Store the filled record in the owner's durable private location. A temporary
worktree and an ignored directory are not backups; ignored files do not follow
new checkouts automatically. Keep secrets in their existing protected store,
not in the record. [Issue #47](https://github.com/wesbos/photo-frame-dashboard/issues/47)
discusses a broader per-frame record convention; this checklist does not prescribe
that storage layout.

Share only sanitized, reusable findings upstream, using the
[contribution checklist](../.github/pull_request_template.md). Keep the complete
private evidence with the owner. The optional
[verify-frame-release skill](../skills/verify-frame-release/SKILL.md) applies this
workflow for an agent; the checklist is usable without it.
