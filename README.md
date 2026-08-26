# HA Open/Close Notify Blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FDestynnReigneer%2FHA-Open-Close-Notify-Blueprint%2Fmain%2Fopen_close_notify_customizable.yaml)

A Home Assistant blueprint that sends configurable, actionable notifications when a door, window, garage, safe, pool gate, or any other `binary_sensor` is left open for too long — and keeps nagging every recipient until the issue is resolved.

## Purpose

The goal of this blueprint is to establish mission-critical notifications to every device that needs one, so that open doors, windows, garages, safes, pool gates, and similar entry points don't get forgotten and left unsecured.

It's a one-stop notification system: it presents the user with choices, nudges them to act, and — if configured to — takes the closing action itself when nobody responds. Notifications go out to multiple people, and everyone is kept informed of what action was (or wasn't) taken. If nobody responds, everyone gets a repeated nag until the sensor closes or the automatic close succeeds. Repetitive nagging is intentional: it's what actually gets sensitive entry points secured.

## Features

- **Universal sensor support** — works with any `binary_sensor` (door, window, garage, motion, etc.).
- **Actionable notifications** — each alert includes three configurable buttons.
- **Fully customizable text** — initial message, button titles, acknowledgment message, confirmation message, and failure message are all inputs.
- **Configurable delays** — set the initial trigger time and a separate re-notification delay per button.
- **Optional auto-close** — point it at a `cover` entity and it will attempt to close it automatically and confirm the result, instead of just asking someone to close it by hand.

## How It Works

1. The automation triggers once the monitored sensor has been `on` (open) for the configured **Initial Alert Time**.
2. An actionable notification goes out with three buttons: two "wait" options with independent delays, and a "close" option.
3. Depending on which button is pressed (or if nobody responds):
   - **Wait** — the alert is dismissed and re-sent after the chosen delay, repeating until the sensor closes.
   - **Close**, with no closable device configured — sends an acknowledgment (naming who tapped it), then keeps nagging until the sensor closes.
   - **Close**, with a closable device configured — calls `cover.close_cover`, sends a confirmation, and keeps nagging if the sensor is still open afterward.

## Requirements

- A recent Home Assistant instance.
- The [Home Assistant Companion App](https://companion.home-assistant.io/) installed and set up for actionable notifications on every device you want to notify.
- The sensor you want to monitor exposed as a `binary_sensor`.
- (Optional) A `cover` entity if you want the blueprint to attempt an automatic close.

## Installation

1. Click the **"Open your Home Assistant instance"** badge above, or go to **Settings → Automations & Scenes → Blueprints → Import Blueprint** and paste in:
   ```
   https://raw.githubusercontent.com/DestynnReigneer/HA-Open-Close-Notify-Blueprint/main/open_close_notify_customizable.yaml
   ```
2. Click **Preview Blueprint**, then **Import Blueprint**.
3. Go to the **Automations** tab and click **Create Automation**, then choose **Customizable Door/Window Open Alert**.

## Configuration Reference

| Field | Description | Example |
|---|---|---|
| Sensor to Monitor | The `binary_sensor` that triggers the alert. | `binary_sensor.garage_door_sensor` |
| Initial Alert Time (minutes) | How long the sensor must be `on` before the first alert. | `15` for a garage door, `5` for a mailbox |
| Notification Devices | Mobile app devices to notify. | — |
| Closable Device (optional) | A `cover` entity the blueprint can command closed. Leave blank to show an "Acknowledge" option instead. | `cover.garage_door` |
| Initial Notification Message | The primary alert message. | "The garage door has been open for too long." |
| Option 1 / Option 3 Button Title + Delay | Text and re-nag delay for the two "wait" buttons. | "Wait 30 Minutes" / `30` |
| Option 2 Title (Closable Device Set) | Button text shown when a closable device is configured. | "Close Garage" |
| Option 2 Title (No Device) | Button text shown when no closable device is configured. | "Acknowledge & Close App" |
| Acknowledgment Message | Sent when someone acknowledges without an auto-close device. Use `{person}` for the device name. | "{person} has chosen to close the garage. Please remember to close it manually." |
| Acknowledgment Check Delay (minutes) | How long to wait after acknowledgment before re-checking the sensor. | `10` |
| Urgent Prefix Text | Title used on the re-sent notification after an urgent check fails. | "Urgent!!! Still open" |
| Confirmation Message | Sent after the auto-close command is issued. | "The garage door has been closed." |
| Closing Failure Message | Sent if the close command was issued but the sensor is still `on`. | "The command to close was sent, but the door is still open. Please check it." |

## Troubleshooting

- **Notification not sent?** Check the automation's trace log (Settings → Automations & Scenes → open the automation → the three-dot menu → Traces) to see exactly which steps ran and where it failed.
- **Buttons not working?** Make sure the Companion App on each device is fully set up for actionable notifications — this often requires extra configuration inside the app itself.

## Known Issues

This blueprint has a few outstanding bugs that are being tracked before it can be considered fully working out of the box:

- **Notifications aren't actually targeted.** The `notification_devices` input is collected but never used — every notification call is `notify.mobile_app`, which is not a real Home Assistant service. Delivery needs to loop over the selected devices (e.g. `notify.mobile_app_<device_slug>` per device) instead.
- **`notification_id` is used with `!input` but is a template variable, not a blueprint input.** `!input` only resolves declared `blueprint.input` entries; referencing `!input notification_id` for a value defined under `variables:` is invalid and will fail blueprint validation. It should be referenced as `{{ notification_id }}`.
- **The closable-device check is wrong.** `is_device_id(input('closable_device'))` tests whether the value looks like a device registry ID, but the `closable_device` selector returns an *entity* ID (e.g. `cover.garage_door`), so this condition is effectively always false — the auto-close path never triggers even when a cover is configured.
- **The dismiss-notification call is malformed.** `service: mobile_app` with `data: {action: dismiss_notification}` isn't a valid way to clear a Companion App notification; that's normally done via `notify.mobile_app_<device>` with `message: "clear_notification"` and the matching `tag`.
- **Button presses after the first are ignored.** Once a "wait" option is chosen, the repeat loop just delays and re-sends the same notification — it never listens for `mobile_app_notification_action` again, so pressing a button on a later reminder does nothing.
- **`source_url` points at the original gist**, not this repository, so Home Assistant's "check for blueprint updates" feature won't see changes made here until it's updated to point at this repo's raw file.

## Contributing

Issues and pull requests are welcome. If you spot additional bugs or have use cases this blueprint should support, please open an issue describing the scenario.

## License

No license has been specified for this repository yet — until one is added, all rights are reserved by the author. If you intend to reuse or modify this blueprint, please check with the repository owner or watch for a `LICENSE` file to be added.
