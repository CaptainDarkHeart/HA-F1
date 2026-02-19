# Napkin — HA-F1

## Corrections
| Date | Source | What Went Wrong | What To Do Instead |
|------|--------|----------------|-------------------|

## User Preferences
- Private GitHub repo
- No forking of upstream (midi2mqtt is a submodule/reference only)
- Config files live at `~/.config/midi2mqtt/` on the target Linux host
- systemd user service (not system-wide) by default

## Patterns That Work
- `midi2mqtt -test` flag is the recommended way to verify MIDI input before enabling the service
- HA sensor uses `state_topic: "midi/events"` with `value_template` pulling `event_type` as state, and `json_attributes_topic` for the full payload

## Patterns That Don't Work

## Domain Notes
- F1 pad notes: 36–51 (bottom-left to top-right, row-major order)
- F1 faders: CC 0–3 on channel 1
- Button assignments vary by firmware; use `-test` to discover
- midi2mqtt binary default location assumed: `~/midi2mqtt/midi2mqtt`
- Config default location: `~/.config/midi2mqtt/midi2mqtt.yaml`
- HA brightness scale is 0–255; fader values are 0–127 (map accordingly)
