# Developer Changelog

Detailed technical log for maintainers. For the user-facing summary, see [CHANGELOG.md](CHANGELOG.md).

## Unreleased — [PR #6](https://github.com/DestynnReigneer/arks-ha-blueprints/pull/6): UX rework from real-world testing

Prompted by the first real attempt to build an automation from the blueprint after PR #3/#4/#5 landed.

- **"Missing input closable_device" on import, despite the field being marked optional in its description.** Root cause: Home Assistant's blueprint editor treats any input without a `default:` key as required, regardless of the description text. Fix: added `default: ""` to `closable_device` and to every other optional field (`option_2_title_closable`, `option_2_title_non_closable` got `default: "Acknowledge"`, `confirmation_message_text`, `failure_message_text`, `acknowledgment_message_text` got a real default, `option_1_title`/`option_3_title` got `default: "Snooze"`, `initial_notification_message` got a generic default). This is also what was silently causing the "malformed message" error — an empty string was reaching the notify service as a title/message.
- **Close button label auto-generation.** Added `closable_device_name` variable: `device_attr(device_id(input('closable_device')), 'name')`. `device_id()` resolves an entity_id to its device registry ID; chained into `device_attr(..., 'name')` gets the friendly device name. `option_2_title` falls back to `'Close ' ~ closable_device_name` when `option_2_title_closable` is blank.
- **Snooze button auto-labeling.** New `option_1_button_title` / `option_3_button_title` variables: `"{{ input('option_1_title') or 'Snooze' }} for {{ input('option_1_delay_minutes') }} min"`. Used in the notification `actions` list instead of the raw `!input` title.
- **Broadcast-who-picked-what for snooze, not just close.** Hoisted `responder_name` computation (`device_attr(wait.trigger.event.data.device_id, 'name')`) to run once right after `wait_for_trigger`, covering all three choice branches uniformly, instead of only computing it inside the close branch. Added a broadcast loop in the wait_1/wait_2 branches: `"{{ responder_name }} snoozed this alert for N minutes (at HH:MM)."`
- **Non-optimistic confirmation.** Previously: `cover.close_cover` → immediately send confirmation → delay 1 min → loop failure message if still open. This could tell everyone "closed" before it actually had. Restructured to `cover.close_cover` → delay 1 min → `if` sensor is off: send confirmation; `else`: `repeat: until: sensor off` (send failure message, delay 5 min) then send confirmation once the loop exits. Confirmation is now only ever reachable after the sensor itself reports closed.
- **Timestamps.** Every broadcast message (snooze, acknowledgment, confirmation, failure) appends `(at {{ now().strftime('%H:%M') }})`. Confirmation/failure defaults are plain strings (`closable_device_name ~ ' closed automatically'`) with the timestamp appended outside the `or` so it applies whether the user supplied custom text or not.
- **Status vs. alert notification tags.** Status broadcasts (snooze/ack/confirm/fail) use `{{ notification_id }}_status` as their tag instead of reusing `{{ notification_id }}`, so they don't collide with or overwrite the still-live actionable alert notification on other people's devices.

## [PR #5](https://github.com/DestynnReigneer/arks-ha-blueprints/pull/5): Badge link still broken after the rename fix

The earlier rename fix ([PR #1](https://github.com/DestynnReigneer/arks-ha-blueprints/pull/1)) used a plain-text find-and-replace on the literal string `DestynnReigneer/ha-blueprints`. The badge's `blueprint_url` query parameter is percent-encoded (`%2F` instead of `/`), so the literal-slash pattern never matched it and the badge kept 404ing while the plain-text install URL right below it was already correct. Lesson: when fixing a URL that appears in multiple encodings, grep for the encoded form too.

## [PR #4](https://github.com/DestynnReigneer/arks-ha-blueprints/pull/4): Device selector schema error

```
Invalid blueprint: extra keys not allowed @ data['blueprint']['input']['notification_devices']['selector']['domain']. Got None
```

Root cause: HA's **device** selector filters by `integration:` (the component that registered the device — e.g. `mobile_app`), not `domain:`. `domain:` is only a valid key on the **entity** selector (which filters by entity domain, e.g. `binary_sensor`). This was a pre-existing bug in the original blueprint (predates this repo); it simply hadn't been reached yet because the import was failing earlier for unrelated reasons (see PR #3).

## [PR #3](https://github.com/DestynnReigneer/arks-ha-blueprints/pull/3): `action:` vs `service:`

The initial logic rewrite in PR #1 used Home Assistant's newer `action:` key for every service call. `action:` was only added as a valid schema key around HA core 2024.8; on any older install it's rejected, surfacing as a generic "Unknown error" in the Import Blueprint dialog with no useful detail (confirmed via the actual HA log: a `ClientResponseError` 404 was the real first-order bug — see below — but the `action:`/`service:` mismatch was a second latent bug that would have blocked older HA versions regardless). `service:` is accepted on every currently-supported HA version, so it's the safe default absent a specific reason to require `action:`. The `action:` keys inside notification button definitions (`data.actions: [{action: "wait_1", ...}]`) are a different, unrelated schema (the Companion App's own actionable-notification format) and were left untouched.

Separately diagnosed from the actual HA error log in this same session: the reported "Unknown error" was actually a 404 — the import URL pointed at `ha-blueprints` (a repo name that was never actually available; the real repo is `arks-ha-blueprints`). Two independent bugs were stacked here: wrong URL (fixed by using the correct URL) and the `action:`/`service:` compatibility issue (fixed in this PR, discovered while investigating before the URL issue was confirmed as root cause for that specific report).

## [PR #1](https://github.com/DestynnReigneer/arks-ha-blueprints/pull/1): Initial repo cleanup + core logic rewrite

Original blueprint (imported from a personal gist, no repo history prior to this) had six functional bugs, found by static review before any live HA testing was possible:

1. **`notification_devices` input collected but never used.** Every notify call was the literal `service: notify.mobile_app`, which is not a real HA service (there's no single service by that name — each device gets its own `notify.mobile_app_<slug>` service). Fixed with a `repeat: for_each: !input notification_devices` loop, calling `notify.mobile_app_{{ device_attr(repeat.item, 'name') | slugify }}` per device.
2. **`!input notification_id` used on a template variable, not a blueprint input.** `notification_id` was defined under the automation's `variables:` block, not `blueprint.input`. `!input` only resolves declared blueprint inputs; this would fail blueprint validation. Fixed by referencing it as `{{ notification_id }}` (Jinja) everywhere instead.
3. **`is_device_id(input('closable_device'))` always false.** `closable_device`'s selector is an **entity** selector, so `input('closable_device')` holds an entity_id (`cover.garage_door`), never a device_id. `is_device_id()` only matches device registry IDs, so the closable-device branch never ran even when correctly configured. Fixed by checking `input('closable_device') not in [none, '']` instead.
4. **Malformed dismiss-notification call.** `service: mobile_app` / `data: {action: dismiss_notification}` isn't how the Companion App clears a notification. Fixed to use the documented pattern: `message: "clear_notification"` with the matching `tag` under `notify.mobile_app_<device>`.
5. **Buttons on repeated reminders were dead.** The original repeat/delay loops just resent the alert and delayed again, with no `wait_for_trigger` inside the loop — so a button tap on the 2nd+ notification did nothing. Restructured into a single `repeat: while: sensor is on` loop that both sends the alert and re-listens for a response on every iteration.
6. **`mode: queued`** could stack up multiple concurrent nag loops for a flapping sensor. Changed to `mode: restart`.

Also in this PR: repo restructured from a single flat file into a per-blueprint-folder layout (`door-window-open-alert/`) to hold future blueprints; added `LICENSE` (MIT) and `.gitignore`; `blueprint.source_url` repointed from the original gist to this repo.
