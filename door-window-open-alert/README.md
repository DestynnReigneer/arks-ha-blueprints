# Door/Window Open Alert

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FDestynnReigneer%2Farks-ha-blueprints%2Fmain%2Fdoor-window-open-alert%2Fopen_close_notify_customizable.yaml)

A Home Assistant blueprint that sends configurable, actionable notifications when a door, window, garage, safe, pool gate, or any other `binary_sensor` is left open for too long — and keeps nagging every recipient until the issue is resolved.

## Purpose

The goal of this blueprint is to establish mission-critical notifications to every device that needs one, so that open doors, windows, garages, safes, pool gates, and similar entry points don't get forgotten and left unsecured — think gun safes, vaults, and anything else where "left open" is a real problem, not just an inconvenience.

It's a one-stop notification system: it presents the user with choices, nudges them to act, and — if configured to — takes the closing action itself when nobody responds. Notifications go out to multiple people, and everyone is kept informed of what action was (or wasn't) taken. If nobody responds, everyone gets a repeated nag until the sensor closes or the automatic close succeeds. Repetitive nagging is intentional: it's what actually gets sensitive entry points secured.

## Features

- **Universal sensor support** — works with any `binary_sensor` (door, window, garage, motion, etc.).
- **Actionable notifications with fixed button positions** — left is always Ignore, center is always Close-or-fallback-Snooze, right is always Snooze, regardless of how you label them. Positions stay predictable even when button text is hard to read on some notification UIs.
- **Fully customizable text** — initial message, button titles, confirmation message, and failure message are all inputs.
- **Configurable delays** — set the initial trigger time and an independent re-notification delay per snooze option.
- **Optional auto-close** — point it at a `cover` entity and it will attempt to close it automatically and confirm the result, instead of just asking someone to close it by hand.
- **Real urgent escalation** — an unanswered alert, whether nobody ever responded or a snooze simply ran out with the door still open, is what triggers escalation. Urgent (a distinct relabeling on your own recheck interval) is optional — leave it off and the alert still keeps nagging forever on a plain default interval instead.
- **Custom display name** — override the sensor's Home Assistant name in notifications if it's too technical to read at a glance.
- **Resolves the moment the sensor closes** — however it closes, by hand or automatically, the alert clears, everyone gets a closing confirmation, and the automation ends. It doesn't sit waiting out a snooze or a nag timer first.
- **Names who responded** — snooze, ignore, and close broadcasts use the Home Assistant person linked to the account that tapped the button.

## How It Works

1. The automation triggers once the monitored sensor has been `on` (open) for the configured **Initial Alert Time**.
2. An actionable notification goes out with three buttons, always in this order regardless of label:
   - **Left — Ignore.** Stops the automation entirely for this occurrence. Broadcasts who ignored it to every device first.
   - **Center — Close, or a fallback Snooze.** If a Closable Device is configured, this closes it automatically. If not, this is just a second Snooze with its own independent delay.
   - **Right — Snooze.** The original snooze option, with its own delay.
   - Both snooze buttons automatically show their delay on the label itself, e.g. "Snooze (30)" — no need to type the number.
3. Depending on which button is pressed (or if nobody responds):
   - **Snooze (either button)** — every device gets told who snoozed it and for how long, the alert is dismissed, and nothing happens for that chosen delay. A snooze is a one-time quiet period, not its own repeating cycle — once it ends, the sensor is checked once.
   - **After that check (or after the very first alert gets no response at all)** — if the sensor's closed, the automation is done. If it's still open, the alert re-sends and escalates: **marked Urgent** on your configured Urgent Recheck Delay if you set one, or just re-sent normally on the Standard Nag Delay if you left Urgent at 0 — either way, it keeps repeating on that interval until the sensor closes. Nagging never silently stops; only the Urgent relabeling is optional.
   - **Close, with no closable device configured** — behaves exactly like a snooze (see above), using its own fallback delay.
   - **Close, with a closable device configured** — calls `cover.close_cover`, waits a minute, then checks the sensor. If it's actually closed, sends the confirmation message (naming who requested it); if not, it retries every 5 minutes with the failure message until the sensor agrees it's shut. The confirmation is never sent optimistically — only once the sensor itself confirms it.
   - **Ignore** — the loop stops. If the sensor later closes and reopens, a fresh cycle starts from scratch.
4. **At any point, if the sensor closes, the alert is over.** The blueprint watches the sensor the whole time, not just between nags, so a door closed by hand ends the cycle right then: the open alert is dismissed on every device and a closing confirmation goes out. That applies during the first wait, during a snooze, and during the auto-close retry loop.
5. **A button press is only acted on if the sensor is still open.** The state is re-checked at the moment of the tap. If someone taps Close a few seconds after the door was already shut, that press is discarded rather than sent to the device — so a toggle-style closer can't be flipped back open — and a late Snooze can't silence an alert that's already resolved.

Every notification — including the initial alert and every escalated resend — shows the sensor's name (or your custom Friendly Name) in the title and a timestamp in the message (`@ 14:30`), so it's always clear what's open and when the message was sent, without having to open the app.

If the sensor flaps (closes and reopens) mid-alert, the automation restarts cleanly for the new open event instead of running two nag loops at once.

**Choosing "Closable Device" vs. leaving it blank:** fill it in only when there's an actual device that can close the sensor for you — a smart garage door opener, a smart lock's cover/switch entity. Leave it blank for anything a person has to physically check — an interior door, a safe, a vault, a hinged door with no auto-closer; the center button becomes a second Snooze instead. There's no "open the app" launcher built in yet (see Known Issues below for the idea being considered).

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
| Friendly Name | Optional — falls back to the sensor's HA name | Overrides the name shown in notifications. | "Master Bedroom Door" |
| Initial Alert Time (minutes) | Required | How long the sensor must be `on` before the first alert. | `15` for a garage door, `5` for a mailbox |
| Notification Devices | Required | Mobile app devices to notify. | — |
| Closable Device | Optional — leave blank if nothing can auto-close it | A `cover` entity the blueprint can command closed. Leave blank for doors a person has to check (interior doors, safes, vaults) — the center button becomes a second Snooze instead. | `cover.garage_door` |
| Initial Notification Message | Optional — defaults to a generic message | The primary alert message. | "The garage door has been open for too long." |
| Snooze Button Title (Right) | Optional — defaults to "Snooze" | Base text for the right-hand button. Its delay is appended automatically ("Snooze (30)"). | "Snooze" |
| Snooze Delay - Right Button (minutes) | Required | How long this snooze lasts before re-checking. | `30` |
| Center Button Title (Closable Device Set) | Optional — auto-generates "Close \<device name\>" if left blank | Only fill in for different wording. | "Shut Garage" |
| Center Button Title (Fallback Snooze, No Device) | Optional — defaults to "Snooze" | Text for the center button when it's acting as a fallback snooze. | "Snooze" |
| Snooze Delay - Center Button Fallback (minutes) | Required | How long the fallback snooze lasts, independent of the right button's delay. | `90` |
| Urgent Recheck Delay (minutes) | Optional — 0 turns Urgent off | Whenever an alert gets zero response, how long before it re-sends marked Urgent, and how often it repeats after that until resolved. Leave at `0` to keep nagging on the Standard Nag Delay instead, without the Urgent relabeling. | `10` |
| Standard Nag Delay (minutes) | Optional — defaults to `15` | Used instead of Urgent Recheck Delay whenever that's left at 0, and for the very first alert's own wait. | `15` |
| Urgent Prefix Text | Optional — defaults to "Urgent" | Title used on a re-sent alert that got no response at all, when Urgent Recheck Delay is set above 0. | "Urgent - still open!" |
| Confirmation Message | Optional — has a generated default | Sent on any confirmed close, whether a person closed it by hand or the Closable Device closed it. Never sent until the sensor itself reports closed. Use `{person}` for whoever pressed Close — it renders as "someone" on a manual close, since nobody pressed anything. | "The garage door closed automatically, requested by {person}." |
| Closing Failure Message | Optional — has a generated default | Sent every 5 minutes while the sensor still shows open after a close command. Use `{person}` for whoever pressed Close. | "The garage door is still open — please check it." |

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for what's changed release to release.

## Troubleshooting

- **Notification not sent?** Check the automation's trace log (Settings → Automations & Scenes → open the automation → the three-dot menu → Traces) to see exactly which steps ran and where it failed.
- **Buttons not working?** Make sure the Companion App on each device is fully set up for actionable notifications — this often requires extra configuration inside the app itself.

## Known Issues

- **Who tapped a button can still fall back to "Someone" in rare cases.** Status broadcasts first resolve the responder to the Home Assistant **person** whose user account tapped the button, which is the reliable path. If that lookup comes up empty — no `person` entity is linked to that user account, or the event carries no user context — the blueprint falls back to reading `device_id` off the `mobile_app_notification_action` event. Home Assistant has a [known bug](https://github.com/home-assistant/core/issues/88742) where that field is sometimes the phone's own OS-level ID rather than its device registry ID, in which case the name lands on "Someone". Linking each Home Assistant user to a person entity (Settings → People) avoids this entirely.

## Planned: "Open App" button

For doors controlled by a third-party app the blueprint can't operate directly (MyQ, August, Yale, etc.), the plan is a free-text field where you paste that app's URL scheme (from the app's own docs, e.g. `myq://`), rather than a hardcoded dropdown of app names — a dropdown would need constant upkeep and would still miss whatever app someone actually has. The Companion App's notification actions already support opening an arbitrary link via a `uri:` field, so this is a matter of wiring that up. Not implemented yet.
