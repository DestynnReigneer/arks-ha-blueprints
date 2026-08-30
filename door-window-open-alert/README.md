# Door/Window Open Alert

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FDestynnReigneer%2Farks-ha-blueprints%2Fmain%2Fdoor-window-open-alert%2Fopen_close_notify_customizable.yaml)

A Home Assistant blueprint that sends configurable, actionable notifications when a door, window, garage, safe, pool gate, or any other `binary_sensor` is left open for too long — and keeps nagging every recipient until the issue is resolved.

## Purpose

The goal of this blueprint is to establish mission-critical notifications to every device that needs one, so that open doors, windows, garages, safes, pool gates, and similar entry points don't get forgotten and left unsecured — think gun safes, vaults, and anything else where "left open" is a real problem, not just an inconvenience.

It's a one-stop notification system: it presents the user with choices, nudges them to act, and — if configured to — takes the closing action itself when nobody responds. Notifications go out to multiple people, and everyone is kept informed of what action was (or wasn't) taken. If nobody responds, everyone gets a repeated nag until the sensor closes or the automatic close succeeds. Repetitive nagging is intentional: it's what actually gets sensitive entry points secured.

## Features

- **Universal sensor support** — works with any `binary_sensor` (door, window, garage, motion, etc.).
- **Actionable notifications** — each alert includes three configurable buttons.
- **Fully customizable text** — initial message, button titles, acknowledgment message, confirmation message, and failure message are all inputs.
- **Configurable delays** — set the initial trigger time and a separate re-notification delay per button.
- **Optional auto-close** — point it at a `cover` entity and it will attempt to close it automatically and confirm the result, instead of just asking someone to close it by hand.

## How It Works

1. The automation triggers once the monitored sensor has been `on` (open) for the configured **Initial Alert Time**.
2. An actionable notification goes out with three buttons: two snooze options with independent delays (their button labels automatically show the delay, e.g. "Snooze for 30 min"), and a close option.
3. Depending on which button is pressed (or if nobody responds):
   - **No response** — the same alert is re-sent after 24 hours as a safety net, still with live buttons. This is a fixed fallback, not something you configure — the two snooze delays below are what actually control the normal re-nag cadence.
   - **Snooze** — every other device gets told who snoozed it and for how long, the alert is dismissed, and it's re-sent (with working buttons again) after the chosen delay, repeating until the sensor closes.
   - **Close**, with no closable device configured — broadcasts your acknowledgment message (naming who tapped it) to every device, then keeps nagging — buttons included, with the Urgent prefix as the title — until the sensor closes.
   - **Close**, with a closable device configured — calls `cover.close_cover`, waits a minute, then checks the sensor. If it's actually closed, sends the confirmation message; if not, it retries every 5 minutes with the failure message until the sensor agrees it's shut. The confirmation is never sent optimistically — only once the sensor itself confirms it.

Every status broadcast (snooze, acknowledgment, confirmation, failure) has a timestamp appended automatically, so nobody has to guess when something happened.

If the sensor flaps (closes and reopens) mid-alert, the automation restarts cleanly for the new open event instead of running two nag loops at once.

**Choosing "Closable Device" vs. leaving it blank:** fill it in only when there's an actual device that can close the sensor for you — a smart garage door opener, a smart lock's cover/switch entity. Leave it blank for anything a person has to physically check — an interior door, a safe, a vault, a hinged door with no auto-closer. There's no "open the app" launcher built in yet (see Known Issues below for the idea being considered).

## Requirements

- A recent Home Assistant instance.
- The [Home Assistant Companion App](https://companion.home-assistant.io/) installed and set up for actionable notifications on every device you want to notify.
- The sensor you want to monitor exposed as a `binary_sensor`.
- (Optional) A `cover` entity if you want the blueprint to attempt an automatic close.

## Installation

1. Click the **"Open your Home Assistant instance"** badge above, or go to **Settings → Automations & Scenes → Blueprints → Import Blueprint** and paste in:
   ```
   https://raw.githubusercontent.com/DestynnReigneer/arks-ha-blueprints/main/door-window-open-alert/open_close_notify_customizable.yaml
   ```
2. Click **Preview Blueprint**, then **Import Blueprint**.
3. Go to the **Automations** tab and click **Create Automation**, then choose **Customizable Door/Window Open Alert**.

## Configuration Reference

| Field | Required? | Description | Example |
|---|---|---|---|
| Sensor to Monitor | Required | The `binary_sensor` that triggers the alert. | `binary_sensor.garage_door_sensor` |
| Initial Alert Time (minutes) | Required | How long the sensor must be `on` before the first alert. | `15` for a garage door, `5` for a mailbox |
| Notification Devices | Required | Mobile app devices to notify. | — |
| Closable Device | Optional — leave blank if nothing can auto-close it | A `cover` entity the blueprint can command closed. Leave blank for doors a person has to check (interior doors, safes, vaults). | `cover.garage_door` |
| Initial Notification Message | Optional — defaults to a generic message | The primary alert message. | "The garage door has been open for too long." |
| Option 1 / Option 3 Button Title | Optional — defaults to "Snooze" | Base text for the two snooze buttons. The delay is appended automatically ("Snooze for 30 min") — no need to type the number. | "Snooze" |
| Option 1 / Option 3 Delay (minutes) | Required | How long each snooze lasts before re-nagging. | `30` and `120` |
| Option 2 Title (Closable Device Set) | Optional — auto-generates "Close \<device name\>" if left blank | Only fill in for different wording. | "Shut Garage" |
| Option 2 Title (No Device) | Optional — defaults to "Acknowledge" | Button text when no closable device is configured. | "I'll check it" |
| Acknowledgment Message | Optional — has a default | Broadcast to every device when someone acknowledges. Use `{person}` for their device name. | "{person} is checking on it — this will nag again later if it's still open." |
| Acknowledgment Check Delay (minutes) | Required | How long to wait after acknowledgment before re-checking and re-nagging with the Urgent prefix. | `10` |
| Urgent Prefix Text | Optional — defaults to "Urgent" | Title used on the re-sent notification after an acknowledgment check still finds it open. | "Urgent - still open!" |
| Confirmation Message | Optional — has a generated default | Only sent once the sensor confirms it's actually closed. | "The garage door closed automatically." |
| Closing Failure Message | Optional — has a generated default | Sent every 5 minutes while the sensor still shows open after a close command. | "The garage door is still open — please check it." |

## Troubleshooting

- **Notification not sent?** Check the automation's trace log (Settings → Automations & Scenes → open the automation → the three-dot menu → Traces) to see exactly which steps ran and where it failed.
- **Buttons not working?** Make sure the Companion App on each device is fully set up for actionable notifications — this often requires extra configuration inside the app itself.

## Known Issues

- **Who tapped "Close" may be reported wrong on some devices.** The blueprint reads `device_id` off the `mobile_app_notification_action` event to look up a name for the acknowledgment message. Home Assistant has a [known bug](https://github.com/home-assistant/core/issues/88742) where this field is sometimes the phone's own OS-level ID rather than its Home Assistant device registry ID, which can make `{person}` resolve to nothing on affected devices. This is a core Home Assistant issue, not something this blueprint can work around.

## Planned: "Open App" button

For doors controlled by a third-party app the blueprint can't operate directly (MyQ, August, Yale, etc.), the plan is a free-text field where you paste that app's URL scheme (from the app's own docs, e.g. `myq://`), rather than a hardcoded dropdown of app names — a dropdown would need constant upkeep and would still miss whatever app someone actually has. The Companion App's notification actions already support opening an arbitrary link via a `uri:` field, so this is a matter of wiring that up. Not implemented yet.
