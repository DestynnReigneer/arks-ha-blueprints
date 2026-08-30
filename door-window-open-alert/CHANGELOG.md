# Changelog

All notable changes to the Door/Window Open Alert blueprint are documented here. Dates are when the change merged.

## [Unreleased]

Pending QA.

### Changed
- **Button layout redesigned.** Buttons now have fixed positions regardless of label: left is always Ignore (new — stops the alert entirely), center is always Close-the-device-or-a-fallback-Snooze, right is always Snooze.
- **Urgent behavior reworked.** Urgent now specifically means "this alert got zero response" and resets back to normal the moment anyone responds, instead of sticking permanently after one acknowledgment. It keeps repeating on the Urgent Recheck Delay until resolved.
- The Acknowledge option is gone — with no closable device, the center button is now just a second, independently-timed Snooze.
- Every notification (including Urgent resends) now shows the sensor's name in the title and a timestamp in the message.
- The auto-close confirmation and failure messages now name who requested the close.

## 2026-08-30

### Fixed
- "Missing input closable_device" error when building an automation, even with nothing entered.
- "Malformed message" error when Option 2's button text fields were left blank.
- "Message malformed" error on save when Closable Device was left blank.
- Automations silently failed to fire (no notification, no error visible to the user) due to an internal template error.
- The auto-close confirmation could fire before the door was actually confirmed closed.

### Changed
- The Close button now auto-labels itself from the selected device's name ("Close Garage") instead of requiring you to type it.
- Snooze buttons now auto-append their delay ("Snooze for 30 min") instead of needing it typed in.
- Every choice (snooze or close) now tells every notification device who picked what.
- Confirmation and Closing Failure messages are now optional, with sensible defaults if left blank.
- All status messages now include a timestamp automatically.
- Every field's description now includes a concrete example.

## 2026-08-28

### Fixed
- Notifications weren't reaching any device — the blueprint called a notify service that doesn't exist.
- The "close automatically" option never worked, even with a device selected, due to an incorrect device-matching check.
- Buttons on repeated reminders didn't do anything — only the very first alert's buttons worked.
- A malformed dismiss-notification call.
- Blueprint import failed with an "unknown error" caused by a broken link (repo rename) and a Home Assistant version-compatibility issue.
- Blueprint import failed with a schema error on the "Notification Devices" field.
- The "Open your Home Assistant instance" import badge linked to the wrong URL.

### Changed
- A flapping sensor no longer stacks up multiple overlapping alert cycles.

## 2026-08-27

### Added
- Initial public release: door/window open alert with snooze, acknowledge, and optional auto-close.
