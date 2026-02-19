# Traktor Kontrol F1 — MIDI Note & CC Map

## Pad Grid (Note On / Note Off, Channel 1)

The 16 pads are laid out in a 4×4 grid. Notes ascend left-to-right, then
bottom-to-top — matching the physical controller face-on.

```
┌──────┬──────┬──────┬──────┐  ← Top row (Row 4)
│  48  │  49  │  50  │  51  │
├──────┼──────┼──────┼──────┤
│  44  │  45  │  46  │  47  │  Row 3
├──────┼──────┼──────┼──────┤
│  40  │  41  │  42  │  43  │  Row 2
├──────┼──────┼──────┼──────┤
│  36  │  37  │  38  │  39  │  ← Bottom row (Row 1)
└──────┴──────┴──────┴──────┘
 Col 1  Col 2  Col 3  Col 4
 (left)               (right)
```

### Pad Reference Table

| Physical Position | Row | Col | MIDI Note | Hex  |
|-------------------|-----|-----|-----------|------|
| Bottom-left       |  1  |  1  |    36     | 0x24 |
| Bottom            |  1  |  2  |    37     | 0x25 |
| Bottom            |  1  |  3  |    38     | 0x26 |
| Bottom-right      |  1  |  4  |    39     | 0x27 |
| Row 2, left       |  2  |  1  |    40     | 0x28 |
| Row 2             |  2  |  2  |    41     | 0x29 |
| Row 2             |  2  |  3  |    42     | 0x2A |
| Row 2, right      |  2  |  4  |    43     | 0x2B |
| Row 3, left       |  3  |  1  |    44     | 0x2C |
| Row 3             |  3  |  2  |    45     | 0x2D |
| Row 3             |  3  |  3  |    46     | 0x2E |
| Row 3, right      |  3  |  4  |    47     | 0x2F |
| Top-left          |  4  |  1  |    48     | 0x30 |
| Top               |  4  |  2  |    49     | 0x31 |
| Top               |  4  |  3  |    50     | 0x32 |
| Top-right         |  4  |  4  |    51     | 0x33 |

> **Note:** MIDI notes are sent on Channel 1 (MIDI channel index 0).
> Velocity 0 on a `note_on` message is treated as `note_off` by many
> devices — midi2mqtt reports this as a `note_on` with `velocity: 0`.

---

## Faders (Control Change, Channel 1)

The F1 has 4 faders. These send CC messages on Channel 1.

| Fader     | CC Number | Value Range | Notes                     |
|-----------|-----------|-------------|---------------------------|
| Fader 1   |     0     |   0–127     | Leftmost fader            |
| Fader 2   |     1     |   0–127     |                           |
| Fader 3   |     2     |   0–127     |                           |
| Fader 4   |     3     |   0–127     | Rightmost fader           |

> **Tip:** Use these to drive continuous parameters in HA — brightness,
> volume, cover position, etc. See `home-assistant/automations-example.yaml`
> for a worked example.

---

## Knob / Encoder (if present)

The F1 does not have rotary encoders. All continuous controls are the 4
faders above.

---

## Buttons

The F1 has several dedicated buttons above the pads. Exact CC or note
assignments may vary by firmware version. Use `midi2mqtt -test` and press
each button to discover its note/CC and channel live.

| Button        | Type          | Typical Note/CC | Channel |
|---------------|---------------|-----------------|---------|
| STOP (×4)     | Note On/Off   | TBC — use -test | 1       |
| CAPTURE       | Note On/Off   | TBC             | 1       |
| QUANT         | Note On/Off   | TBC             | 1       |
| SIZE          | Note On/Off   | TBC             | 1       |
| TYPE          | Note On/Off   | TBC             | 1       |
| REVERSE       | Note On/Off   | TBC             | 1       |
| SHIFT         | Note On/Off   | TBC             | 1       |
| Browse knob   | Control Change| TBC             | 1       |

> Run `midi2mqtt -test -port "Traktor Kontrol F1 MIDI 1"` while pressing
> buttons to capture the exact values and fill in this table for your
> firmware version.

---

## Discovering Unknown Mappings

```bash
# List all MIDI ports visible on your system
midi2mqtt -list

# Print all incoming MIDI events in real time (no MQTT publishing)
midi2mqtt -test -port "Traktor Kontrol F1 MIDI 1"
```

Press each pad, fader, and button while watching the output to build your
own complete map.
