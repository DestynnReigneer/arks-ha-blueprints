# Changelog

All notable changes to the Door/Window Open Alert blueprint are documented here. Dates are when the change merged.

## [Unreleased]

Pending QA.

### Added
- **The center button can now run any entity, not just a `cover`.** Point it at a cover, lock, switch, light, fan, input boolean, script, scene, or button. This is what makes the button usable at all for most setups — previously it required a `cover` entity, so anyone whose door closer is a relay, a smart plug, or a script had no way to use it.
- **New "Action to Perform" field.** Defaults to Automatic, which picks the right action from the entity type: covers close, locks lock, buttons press, scripts and scenes run, everything else turns off. Override it for hardware where that's backwards — most commonly a momentary relay that *closes* a door by being switched **on**.
- The center button's label now writes itself from the device and the action ("Close Garage Door", "Turn Off Porch Light") instead of always saying "Close".

### Changed
- "Closable Device" is now called **Action Device**, and "Closing Failure Message" is now **Action Failure Message**, since neither is limited to closing any more. Existing automations keep their settings — only the labels changed.
- The auto-fix confirmation and failure messages now name both the sensor and the device that acted, and no longer assume the action was "closing".

### Added
- **Closing the sensor by hand now ends the alert and tells everyone.** Previously a physical close only stopped the nagging quietly, with no closing notification, and only at the next timeout. Now the automation watches the sensor throughout, so a manual close clears the alert and sends the confirmation message immediately, from any point in the cycle: the first wait, a snooze, or between nags.
- **The sensor state is re-checked before any button press is acted on.** If it already closed in the seconds before someone taps, the tap is ignored instead of acted on, so a late Close tap can't re-open a toggle-style device and a late Snooze can't silence an alert that's already resolved.

### Fixed
- **Confirmation Message now actually fires.** It was only reachable on the auto-close path, so with no Closable Device configured it never appeared no matter what was typed in it. It now sends on any confirmed close.
- **Status messages name the person, not the phone.** Snooze/Ignore/Close broadcasts now resolve to the Home Assistant person linked to the account that tapped the button, falling back to the device name and then to a generic word.
- Snoozes and the auto-close retry loop no longer sit through their full delay after the sensor has closed. They end as soon as it does.

### Changed
- **Status broadcasts now say which sensor they're about.** "Someone snoozed this alert for 5 minutes" gave no way to tell which door it meant when more than one alert was running. Every snooze, ignore, close, and still-open message now names the sensor in both the notification title and the message body.
- Timestamps in every notification changed from `(at 14:30)` to `@ 14:30`.
- **Urgent is now optional.** Leave Urgent Recheck Delay at `0` and nagging still continues forever on a new Standard Nag Delay instead — only the Urgent relabeling turns off.
- **Snoozing now escalates properly.** A snooze is a one-time quiet period; once it runs out with the door still open, the alert escalates straight into the Urgent-or-standard-nag cadence, instead of resending a plain alert first. Fixes Urgent not firing after a snooze.
- New optional Friendly Name input, to override the sensor's Home Assistant name in notifications.
- Snooze button labels changed from "Snooze for 30 min" to "Snooze (30)"; the center button's fallback-snooze label now shows its delay too.

## 2026-08-30

### Changed
- **Button layout redesigned.** Buttons now have fixed positions regardless of label: left is always Ignore (new — stops the alert entirely), center is always Close-the-device-or-a-fallback-Snooze, right is always Snooze.
- **Urgent behavior reworked.** Urgent now specifically means "this alert got zero response" and resets back to normal the moment anyone responds, instead of sticking permanently after one acknowledgment. It keeps repeating on the Urgent Recheck Delay until resolved.
- The Acknowledge option is gone — with no closable device, the center button is now just a second, independently-timed Snooze.
- Every notification (including Urgent resends) now shows the sensor's name in the title and a timestamp in the message.
- The auto-close confirmation and failure messages now name who requested the close.

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
