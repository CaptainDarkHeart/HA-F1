# F1 MIDI Mode Reference (macOS)

## What is MIDI Mode?

The Traktor Kontrol F1 has two operational modes:

1. **Standard (Traktor) Mode** — Integrates deeply with Traktor DJ software
2. **MIDI Mode** — Operates as a generic MIDI controller (ideal for Home Assistant)

### Why Use MIDI Mode?

On **macOS**, NI's `NIHardwareAgent` daemon intercepts all F1 MIDI traffic at the driver level when the F1 is in standard Traktor mode. This prevents third-party tools like midi2mqtt from receiving any events, even through IAC routing.

**MIDI Mode bypasses this limitation entirely**, allowing the F1 to work seamlessly with midi2mqtt on macOS.

On **Linux**, MIDI Mode is optional — the standard mode works fine and sends note events (see [f1-note-map.md](f1-note-map.md)).

---

## Enabling MIDI Mode

1. On the **Traktor Kontrol F1**, hold the **Shift** button
2. Press the **Browse** knob/button
3. The F1 enters MIDI mode (LED indicators will change)
4. To exit MIDI mode, repeat the process or unplug the F1

> **Note:** When in MIDI mode, the F1 no longer functions with Traktor software — it operates as a generic MIDI controller only.

---

## Port Name on macOS

When in MIDI Mode:
- **Port name:** `"Traktor Kontrol F1 - 1 Input"`

This differs from Linux standard mode (`"Traktor Kontrol F1 MIDI 1"`).

Verify the port name on your system:
```bash
midi2mqtt -list-ports
```

---

## MIDI Mode Event Types

MIDI Mode sends **Control Change (CC) events only**. It does NOT send note_on or note_off events.

- **Event type:** Control Change (CC)
- **Channel:** 12 (zero-indexed MIDI convention; = MIDI channel 13 in some tools)
- **Value range:** 0–127

---

## MIDI Mode CC Mapping

The following CC numbers are mapped to F1 controls. Some are confirmed; others marked "TBC" (To Be Confirmed) require verification on your system using `midi2mqtt -test -all-events`.

### Faders

| Fader | CC Number | Value Range | Status |
|-------|-----------|-------------|--------|
| Fader 1 | 6 | 0–127 | ✓ |
| Fader 2 | 7 | 0–127 | ✓ |
| Fader 3 | 8 | 0–127 | ✓ |
| Fader 4 | TBC | 0–127 | To be confirmed |

### Buttons & Controls

| Control | CC Number | Press | Release | Status |
|---------|-----------|-------|---------|--------|
| Button (TBC) | 15 | 127 | 0 | To be confirmed |
| Button (TBC) | 16 | 127 | 0 | To be confirmed |

### Pads (Bottom-Left Area)

| Physical Position | CC Number | Value | Status |
|-------------------|-----------|-------|--------|
| Bottom-left pad | 18 | 127 (press), 0 (release) | ✓ |
| Pad (row 1, col 2) | 19 | 127, 0 | ✓ |
| Pad (row 1, col 3) | 20 | 127, 0 | ✓ |
| Pad (row 1, col 4) | 21 | 127, 0 | ✓ |

**Remaining pads (rows 2–4):** Run discovery command below.

---

## Discovering Unknown Mappings

To find the exact CC numbers for controls you haven't identified yet:

```bash
# List all MIDI ports on your system
midi2mqtt -list-ports

# Start MIDI test mode and capture all events
midi2mqtt -test -all-events -port "Traktor Kontrol F1 - 1 Input"

# Now press each pad, button, and move each fader on the F1.
# Watch the terminal output for the CC numbers being sent.
# Press Ctrl+C to exit.
```

Example output:
```
[2026-03-05T10:15:23Z] MIDI Input on Traktor Kontrol F1 - 1 Input
[2026-03-05T10:15:24Z] Control Change: CC 18, channel 12, value 127  ← bottom-left pad pressed
[2026-03-05T10:15:24Z] Control Change: CC 18, channel 12, value 0    ← bottom-left pad released
```

---

## Configuration

Use the macOS MIDI Mode config template:

```bash
cp ~/HA-F1/config/midi2mqtt-macos-midi-mode.yaml ~/.config/midi2mqtt/midi2mqtt.yaml
nano ~/.config/midi2mqtt/midi2mqtt.yaml
```

Update the MQTT broker host and credentials, then restart midi2mqtt:

```bash
# macOS (if using LaunchAgent)
launchctl stop com.local.midi2mqtt
launchctl start com.local.midi2mqtt

# Or manually run in test mode first:
~/midi2mqtt/midi2mqtt -test
```

---

## Building Automations in MIDI Mode

Since MIDI Mode sends CC events instead of note events, automation conditions differ from standard mode.

### Example 1: Trigger on a Pad Press

In standard mode, pads send note_on events. In MIDI Mode, pads send CC events (press = 127, release = 0):

```yaml
- alias: "F1 Pad (MIDI Mode) — Toggle light"
  trigger:
    - platform: state
      entity_id: sensor.traktor_f1_midi_event
      to: "control_change"
  condition:
    - condition: template
      value_template: >
        {{ state_attr('sensor.traktor_f1_midi_event', 'controller') | int == 18
           and state_attr('sensor.traktor_f1_midi_event', 'value') | int == 127 }}
  action:
    - service: light.toggle
      target:
        entity_id: light.desk_lamp
```

### Example 2: Use a Fader for Brightness

Faders work similarly in both modes (CC events with 0–127 value range):

```yaml
- alias: "F1 Fader 1 (MIDI Mode) — Set brightness"
  trigger:
    - platform: state
      entity_id: sensor.traktor_f1_midi_event
      to: "control_change"
  condition:
    - condition: template
      value_template: >
        {{ state_attr('sensor.traktor_f1_midi_event', 'controller') | int == 6 }}
  action:
    - service: light.turn_on
      target:
        entity_id: light.desk_lamp
      data:
        brightness: >
          {{ ((state_attr('sensor.traktor_f1_midi_event', 'value') | int) / 127 * 255) | int }}
```

---

## Comparison: Standard Mode vs MIDI Mode

| Aspect | Standard Mode (Linux) | MIDI Mode (macOS) |
|--------|----------------------|-------------------|
| **Event type** | note_on, note_off, CC | CC only |
| **Pads** | MIDI notes 36–51 | CC events (18–21, TBC) |
| **Faders** | CC 0–3 | CC 6–8, 15–16 |
| **Channel** | Channel 1 (zero-indexed: 0) | Channel 12 (zero-indexed) |
| **Platform** | Linux (any systemd host) | macOS, or Linux alt mode |
| **Port name** | "Traktor Kontrol F1 MIDI 1" | "Traktor Kontrol F1 - 1 Input" |

---

## Troubleshooting

### "Port not found" or "no MIDI events appear"

1. Verify the F1 is in MIDI Mode: Hold Shift + press Browse on the F1
2. Check the port name with `midi2mqtt -list-ports`
3. Ensure the port name in your config matches exactly
4. Verify midi2mqtt can access the port: `midi2mqtt -test`

### Events are missing or incomplete

The CC mapping may be incomplete for your F1 firmware version. Run the discovery command:

```bash
midi2mqtt -test -all-events -port "Traktor Kontrol F1 - 1 Input"
```

Then press all pads and controls to identify any unmapped CCs. Update `event_types` in your config and this documentation as needed.

### LaunchAgent not starting on macOS

```bash
# Check the status
launchctl list | grep midi2mqtt

# View logs
tail -f /tmp/midi2mqtt.log
tail -f /tmp/midi2mqtt.err
```

If the service fails to start, verify:
- The ProgramArguments path in `midi2mqtt.plist` is correct
- The midi2mqtt binary is built and executable: `~/midi2mqtt/midi2mqtt -version`
- The MQTT broker is reachable from your macOS machine

---

## See Also

- [Standard F1 Note/CC Map (Linux Mode)](f1-note-map.md)
- [README: F1 MIDI Mode Setup](../README.md#f1-midi-mode-macos)
- [Upstream midi2mqtt Project](https://github.com/bzeiss/midi2mqtt)
