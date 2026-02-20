# Napkin — HA-F1

## Corrections
| Date | Source | What Went Wrong | What To Do Instead |
|------|--------|----------------|-------------------|
| 2026-02-20 | self | Original config used wrong schema (`mqtt.broker`, `discovery`, `topic` keys) | Correct schema is `mqtt_server.broker.host`, `mqtt_publications`, `midi.event_types` — always check upstream template at `midi2mqtt.yaml.template` |
| 2026-02-20 | self | Documented `-list` flag | Correct flag is `-list-ports` |
| 2026-02-20 | self | Documented `-config` flag | No `-config` flag exists — config is found automatically via search path |
| 2026-02-20 | self | Assumed IAC routing would work around NIHardwareAgent | NIHardwareAgent intercepts F1 MIDI at driver level — no data ever reaches CoreMIDI; IAC is useless here |
| 2026-02-20 | self | Assumed macOS was a dead end | F1 MIDI mode (Shift+Browse) bypasses NIHardwareAgent entirely — works fine on macOS |
| 2026-02-20 | self | Assumed F1 MIDI mode sends note_on/note_off | F1 MIDI mode sends everything as CC on channel 12 (zero-indexed). No note events at all. |
| 2026-02-20 | self | Wrote README saying macOS not supported | macOS works fine when F1 is in MIDI mode — update README accordingly |

## User Preferences
- Private GitHub repo
- No forking of upstream (midi2mqtt is a submodule/reference only)
- Config files live at `~/.config/midi2mqtt/` on the target host
- systemd user service (not system-wide) by default on Linux
- LaunchAgent on macOS
- Pi 5 on order for permanent deployment
- IAC Bus 1 is in use by Anima-in-Machina project — never route F1 traffic there
- Mosquitto: dedicated MQTT user `midi2mqtt` (not HA login account)

## Patterns That Work
- `midi2mqtt -test -all-events` — use this first on any new device to discover CC/note mappings regardless of event_types filter
- `midi2mqtt -list-ports` lists available ports (not `-list`)
- Config auto-discovered from `~/.config/midi2mqtt/midi2mqtt.yaml` — no flag needed
- Build command: `go build -o midi2mqtt ./cmd/` (not `go build -o midi2mqtt .` — no Go files in root)
- `mosquitto_sub -h <broker> -u <user> -P <pass> -t "midi/events"` from Mac = quickest way to verify events landing on HA Green
- HA MQTT Developer Tools tab only appears when integration configured via UI; if configured via YAML it's hidden — use mosquitto_sub instead

## Patterns That Don't Work
- **macOS + F1 in normal (Traktor) mode**: NIHardwareAgent exposes the port name but swallows all events. IAC routing useless. Confirmed via python-rtmidi.
- Killing NIHardwareAgent: removes the F1 port from CoreMIDI entirely
- `-config` flag: does not exist in midi2mqtt
- `-list` flag: does not exist; use `-list-ports`
- `event_types: [note_on, note_off]` in MIDI mode: F1 sends CC only, this filter silently drops everything

## Domain Notes — F1 MIDI Mode CC Map (channel 12, zero-indexed = MIDI ch 13)
| Controller | What it is | Press/max | Release/min |
|-----------|-----------|-----------|-------------|
| CC 6  | Fader (TBC) | 0–127 | — |
| CC 7  | Fader (TBC) | 0–127 | — |
| CC 8  | Fader (TBC) | 0–127 | — |
| CC 15 | Button (TBC) | 127 | 0/absent |
| CC 16 | Button (TBC) | 127 | 0/absent |
| CC 18 | Pad (bottom-left area) | 127 | 0/absent |
| CC 19 | Pad | 127 | 0/absent |
| CC 20 | Pad | 127 | 0/absent |
| CC 21 | Pad | 127 | 0/absent |

Full pad map: run `midi2mqtt -test -all-events` and tap every pad to discover remaining CCs.
On Linux in normal mode, pads send note_on/note_off 36–51 on channel 1.

## Infrastructure
- macOS port name (MIDI mode): `"Traktor Kontrol F1 - 1 Input"`
- Linux port name (normal mode): `"Traktor Kontrol F1 MIDI 1"`
- Config schema top-level keys: `mqtt_server`, `mqtt_publications`, `midi`, `log_level`
- HA Green IP: 192.168.1.11, Mosquitto on port 1883, SSH on port 22
- midi2mqtt binary: `~/midi2mqtt/midi2mqtt`
- Live config: `~/.config/midi2mqtt/midi2mqtt.yaml`
- HA brightness scale 0–255; fader values 0–127 (map: value/127*255)
