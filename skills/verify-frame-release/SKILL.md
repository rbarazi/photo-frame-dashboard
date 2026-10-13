---
name: verify-frame-release
description: Verify a dashboard or Android kiosk release and prepare an evidence-based handoff. Use for release checks, update acceptance or resuming incomplete verification; not for rooting or firmware flashing.
---

# Verify a frame release

Use the [release checklist](../../docs/kiosk-release-checks.md) in this repository.
Keep this skill with its linked documentation if copying it to another location.
It requires an accessible checkout and the tools needed for the chosen checks;
device access is needed only for physical verification. It adds no remote access
or deployment capability.

## Establish the requested scope

Read the project's setup notes and current state before selecting checks. Identify
which layer changed: content, website, backend or native Android application.
Use the exact source, hosted and installed versions where available. Recheck
relevant facts from older handoffs; do not replay their writes or treat their
approval as authorization for a new device change.

Follow the repository's existing contribution, approval and backup instructions.
Default to inspection when only a review was requested. Do not create credentials,
enable wireless debugging, deploy, install software, print or change household
state solely to complete a checklist without authorization for that action.

## Choose checks that establish the outcome

- Use synthetic fixtures for state-changing browser tests. Match the actual
  frame's viewport, but label desktop browser evidence as desktop evidence.
- Confirm a deployment by inspecting the served revision or asset, then confirm
  separately that the frame loaded it. Keep content refresh, website reload and
  Android installation distinct.
- For a native update, preserve signing identity and data. Observe the real
  installer, cancellation and return behavior within the authorized test. An ADB
  install is not evidence that a network updater worked.
- If an acknowledged result is uncertain, inspect before retrying. A lost save
  response may follow a committed write; a queued print job may still print later.
- Stop the affected operation on identity, integrity or baseline mismatch. Keep
  the last working state and report the discrepancy. Do not fix it by uninstalling,
  clearing storage or relaxing authentication.

## Leave a useful result

Report each relevant layer as passed, failed or not checked, with the method and
version. Record any state changed for QA and the known recovery route. Do not
invent paper, speaker or screen acceptance when only software checks passed.
If device access is unavailable, complete independent checks and leave physical
steps pending; do not claim completion or require reinstalling a working frame.

Keep detailed evidence in the owner's durable private location. Public findings
use fictional data and omit credentials, private URLs, identifiers and household
screenshots. Check existing upstream discussions before recommending a new
contribution; publication is a separate owner-authorized action.
