# HA-F1 — Traktor Kontrol F1 → Home Assistant via MIDI/MQTT

Use your **Native Instruments Traktor Kontrol F1** as a hardware controller for
Home Assistant. This project wires the F1's 16 pads, 4 faders, and buttons
into HA automations via MQTT — no custom firmware, no hacking, just
class-compliant USB MIDI bridged over your local network.

```
Traktor F1  →  USB  →  Linux host  →  midi2mqtt  →  MQTT broker (HA)  →  automations
```

---

## What's in this repo

| Path | Purpose |
|------|---------|
| `config/midi2mqtt.yaml` | Ready-to-use midi2mqtt config for the F1 |
| `systemd/midi2mqtt.service` | systemd user-service for auto-start |
| `home-assistant/configuration.yaml` | MQTT sensor snippet for HA |
| `home-assistant/automations-example.yaml` | 4 worked automation examples |
| `docs/f1-note-map.md` | Full pad/fader MIDI note & CC reference |

---

## Requirements

### Hardware
- Native Instruments Traktor Kontrol F1 (USB)
- A Linux host with a USB port (Raspberry Pi, NUC, any desktop)

### Software
- **[midi2mqtt](https://github.com/bzeiss/midi2mqtt)** — the upstream bridge tool
- **Go 1.21+** (to build midi2mqtt from source), or use a pre-built binary
- **MQTT broker** — Mosquitto running inside Home Assistant (the
  [Mosquitto add-on](https://github.com/home-assistant/addons/tree/master/mosquitto)
  is the easiest path)
- Home Assistant with the **MQTT integration** enabled

### Assumptions
- Linux host (systemd-based, e.g. Raspberry Pi OS, Ubuntu, Debian)
- The F1 is recognised as a class-compliant USB MIDI device (no driver needed)
- Your MQTT broker does not require TLS (add TLS settings to the config if it does)

---

## Step-by-step setup

### 1. Clone midi2mqtt

```bash
git clone https://github.com/bzeiss/midi2mqtt.git ~/midi2mqtt
```

### 2. Build the binary

```bash
cd ~/midi2mqtt
go build -o midi2mqtt .
```

Or download a pre-built binary from the
[midi2mqtt releases page](https://github.com/bzeiss/midi2mqtt/releases) and
place it at `~/midi2mqtt/midi2mqtt`.

### 3. Clone this repo

```bash
git clone https://github.com/<your-username>/HA-F1.git ~/HA-F1
```

### 4. Configure midi2mqtt

Copy the config into place and edit it:

```bash
mkdir -p ~/.config/midi2mqtt
cp ~/HA-F1/config/midi2mqtt.yaml ~/.config/midi2mqtt/midi2mqtt.yaml
nano ~/.config/midi2mqtt/midi2mqtt.yaml
```

**You must change:**
- `mqtt.broker` — set this to your Home Assistant / MQTT broker IP address

**Verify the MIDI port name:**

```bash
~/midi2mqtt/midi2mqtt -list
```

The F1 usually appears as `Traktor Kontrol F1 MIDI 1`. If it differs, update
`midi.port` in the config.

> **Security note:** Never commit passwords or credentials.
> The config file is listed in `.gitignore`, but if you fork this repo
> be sure your `~/.config/midi2mqtt/midi2mqtt.yaml` (with real credentials)
> never ends up tracked in git. See `.gitignore` for patterns.

### 5. Test before going live

```bash
~/midi2mqtt/midi2mqtt -test -config ~/.config/midi2mqtt/midi2mqtt.yaml
```

Tap pads and move faders — you should see MIDI events printed to the terminal.
Press Ctrl+C when satisfied.

### 6. Install and start the systemd user service

```bash
mkdir -p ~/.config/systemd/user
cp ~/HA-F1/systemd/midi2mqtt.service ~/.config/systemd/user/midi2mqtt.service

# Open the service file and verify the ExecStart path is correct for your system
nano ~/.config/systemd/user/midi2mqtt.service

systemctl --user daemon-reload
systemctl --user enable --now midi2mqtt

# Check it started cleanly
systemctl --user status midi2mqtt
journalctl --user -u midi2mqtt -f
```

The service will now start automatically on login and restart if it crashes.

> To run as a system service (so it starts without a logged-in user), copy to
> `/etc/systemd/system/` instead, set `User=` to your username, and use
> `systemctl enable --now midi2mqtt` (without `--user`). You may also need to
> add your user to the `audio` group for MIDI access.

### 7. Configure Home Assistant

**Add the MQTT sensor** — open your `configuration.yaml` (or a packages file)
and paste in the contents of `home-assistant/configuration.yaml` from this
repo. Restart HA afterwards.

**Add automations** — paste or import the examples from
`home-assistant/automations-example.yaml`. Replace placeholder entities:

| Placeholder | Replace with |
|-------------|-------------|
| `light.desk_lamp` | Any light entity in your HA |
| `scene.cinema_mode` | Any scene |
| `script.good_morning` | Any script |

### 8. Verify in Home Assistant

1. Open **Developer Tools → MQTT** in HA
2. Subscribe to `midi/events`
3. Press a pad on the F1 — you should see a JSON payload appear
4. Check **Developer Tools → States** for `sensor.traktor_f1_midi_event`

---

## Building automations

Every time you press a pad, the sensor state changes to `note_on` and its
attributes update with `note`, `velocity`, `channel`, etc. Use a state
trigger on `sensor.traktor_f1_midi_event` and a template condition to match
the specific note:

```yaml
trigger:
  - platform: state
    entity_id: sensor.traktor_f1_midi_event
    to: "note_on"
condition:
  - condition: template
    value_template: >
      {{ state_attr('sensor.traktor_f1_midi_event', 'note') | int == 36 }}
```

See `home-assistant/automations-example.yaml` for complete worked examples
and `docs/f1-note-map.md` for the full note number reference.

---

## LED feedback

The F1's pads have RGB LEDs that can be controlled over MIDI (SysEx / Note On
with colour values). This is **not currently implemented** — midi2mqtt is an
input bridge and does not (yet) support sending MIDI output from HA back to
the device. The F1 will still light up using its default behaviour when pads
are pressed.

Future work: a companion `mqtt2midi` bridge or a midi2mqtt output feature
could close this loop and let HA set pad colours to reflect automation state.

---

## Upstream project

This repo depends on **midi2mqtt** by bzeiss:
[https://github.com/bzeiss/midi2mqtt](https://github.com/bzeiss/midi2mqtt)

Please star the upstream repo and report midi2mqtt bugs there, not here.
This repo only contains configuration and documentation for the F1 + HA
use case.

---

## Licence

Configuration files and documentation in this repo are released under
[MIT](https://opensource.org/licenses/MIT). midi2mqtt is a separate project
with its own licence.
