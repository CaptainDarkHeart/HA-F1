# HA-F1 — Traktor Kontrol F1 → Home Assistant via MIDI/MQTT

Use your **Native Instruments Traktor Kontrol F1** as a hardware controller for
Home Assistant. This project wires the F1's 16 pads, 4 faders, and buttons
into HA automations via MQTT — no custom firmware, no hacking, just
class-compliant USB MIDI bridged over your local network.

```
Traktor F1  →  USB  →  Linux host  →  midi2mqtt  →  MQTT broker (HA)  →  automations
```

> **macOS note:** The F1 works on macOS via **MIDI Mode** (Shift+Browse on the
> F1). Normal Traktor mode is not supported (NI's `NIHardwareAgent` intercepts
> the MIDI at driver level). When in MIDI Mode, the F1 sends CC events instead
> of note events, and works fine with midi2mqtt on macOS. See the
> [macOS Setup](#macos-setup) section and [F1 MIDI Mode](#f1-midi-mode-macos)
> for details.

---

## What's in this repo

| Path | Purpose |
|------|---------|
| `config/midi2mqtt.yaml` | Ready-to-use midi2mqtt config for Linux (standard mode) |
| `config/midi2mqtt-macos-midi-mode.yaml` | Ready-to-use config for macOS (MIDI mode) |
| `systemd/midi2mqtt.service` | systemd user-service for auto-start on Linux |
| `launchagents/midi2mqtt.plist` | LaunchAgent plist for auto-start on macOS |
| `home-assistant/configuration.yaml` | MQTT sensor snippet for HA |
| `home-assistant/automations-example.yaml` | 4 worked automation examples |
| `docs/f1-note-map.md` | Full pad/fader MIDI note & CC reference (Linux standard mode) |
| `docs/f1-midi-mode.md` | MIDI mode reference, CC mappings, and macOS guide |

---

## Requirements

### Hardware
- Native Instruments Traktor Kontrol F1 (USB)
- A **Linux** host with a USB port — Raspberry Pi 4/5, NUC, any Debian/Ubuntu box
  - **macOS is supported via MIDI Mode** (see note at top; requires Shift+Browse on the F1)

### Software
- **[midi2mqtt](https://github.com/bzeiss/midi2mqtt)** — the upstream bridge tool
- **Go 1.21+** (to build midi2mqtt from source), or use a pre-built binary
- **MQTT broker** — Mosquitto running inside Home Assistant (the
  [Mosquitto add-on](https://github.com/home-assistant/addons/tree/master/mosquitto)
  is the easiest path)
- Home Assistant with the **MQTT integration** enabled

### Assumptions
- Linux host (systemd-based, e.g. Raspberry Pi OS, Ubuntu, Debian)
- The F1 is recognised as a class-compliant USB MIDI device (no driver needed on Linux)
- Your MQTT broker does not require TLS (add TLS settings to the config if it does)

---

## Step-by-step setup

### 1. Clone midi2mqtt

```bash
git clone https://github.com/bzeiss/midi2mqtt.git ~/midi2mqtt
```

### 2. Build the binary

The main package is in `cmd/`:

```bash
cd ~/midi2mqtt
go build -o midi2mqtt ./cmd/
```

Or download a pre-built binary from the
[midi2mqtt releases page](https://github.com/bzeiss/midi2mqtt/releases) and
place it at `~/midi2mqtt/midi2mqtt`.

### 3. Clone this repo

```bash
git clone https://github.com/CaptainDarkHeart/HA-F1.git ~/HA-F1
```

### 4. Configure midi2mqtt

Copy the config into place and edit it:

```bash
mkdir -p ~/.config/midi2mqtt
cp ~/HA-F1/config/midi2mqtt.yaml ~/.config/midi2mqtt/midi2mqtt.yaml
nano ~/.config/midi2mqtt/midi2mqtt.yaml
```

**You must change:**
- `mqtt_server.broker.host` — set this to your Home Assistant / MQTT broker IP address

**Verify the MIDI port name:**

```bash
~/midi2mqtt/midi2mqtt -list-ports
```

The F1 usually appears as `Traktor Kontrol F1 MIDI 1` on Linux. If it differs,
update `midi.port` in the config.

> **Security note:** Never commit passwords or credentials.
> Your `~/.config/midi2mqtt/midi2mqtt.yaml` (with real credentials) must never
> end up tracked in git. See `.gitignore` for the patterns that protect you.

### 5. Test before going live

```bash
~/midi2mqtt/midi2mqtt -test
```

midi2mqtt finds its config automatically from `~/.config/midi2mqtt/midi2mqtt.yaml`
(there is no `-config` flag). Tap pads and move faders — you should see MIDI
events printed to the terminal. Press Ctrl+C when satisfied.

### 6. Auto-start the bridge service

Choose the appropriate section for your platform:

#### 6a. macOS: Install LaunchAgent

```bash
# Update the ProgramArguments path in the plist to your actual username
nano ~/HA-F1/launchagents/midi2mqtt.plist

# Copy to LaunchAgents directory
mkdir -p ~/Library/LaunchAgents
cp ~/HA-F1/launchagents/midi2mqtt.plist ~/Library/LaunchAgents/

# Load and start the service
launchctl load ~/Library/LaunchAgents/midi2mqtt.plist

# Check it loaded successfully
launchctl list | grep midi2mqtt

# View logs if needed
tail -f /tmp/midi2mqtt.log
```

The service will now start automatically at login and restart if it crashes.

#### 6b. Linux: Install systemd user service

```bash
mkdir -p ~/.config/systemd/user
cp ~/HA-F1/systemd/midi2mqtt.service ~/.config/systemd/user/midi2mqtt.service

# Verify the ExecStart path matches where you built/placed the binary
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
> add your user to the `audio` group for MIDI access:
> `sudo usermod -aG audio $USER`

### 7. Configure Home Assistant (Both macOS and Linux)

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

## F1 MIDI Mode (macOS)

If you're using the F1 on macOS, you'll be using **MIDI Mode** instead of the default Traktor mode. Here's what you need to know:

### Enabling MIDI Mode

1. On the F1, hold **Shift** and press the **Browse** knob/button
2. The F1 will enter MIDI mode (note: this disables its Traktor functionality)
3. The MIDI port name changes to `"Traktor Kontrol F1 - 1 Input"`

### Key Differences

- **Event types:** MIDI Mode sends **CC (Control Change) events only** — no note_on/note_off events
- **Channel:** All events are sent on **channel 12** (zero-indexed in MIDI; channel 13 in some tools)
- **Faders & Buttons:** Mapped to different CC numbers than Linux mode

For a complete CC mapping reference, see [F1 MIDI Mode Reference](docs/f1-midi-mode.md).

### Configuration for MIDI Mode

Use the macOS-specific config template:

```bash
cp ~/HA-F1/config/midi2mqtt-macos-midi-mode.yaml ~/.config/midi2mqtt/midi2mqtt.yaml
nano ~/.config/midi2mqtt/midi2mqtt.yaml
```

This config is pre-configured for MIDI Mode's port name and CC events. Then restart the midi2mqtt service:

```bash
launchctl stop com.local.midi2mqtt
launchctl start com.local.midi2mqtt
```

### Building Automations in MIDI Mode

In MIDI Mode, pads and buttons send **CC events** instead of note events. Adjust your automation conditions accordingly:

```yaml
trigger:
  - platform: state
    entity_id: sensor.traktor_f1_midi_event
    to: "control_change"
condition:
  - condition: template
    value_template: >
      {{ state_attr('sensor.traktor_f1_midi_event', 'controller') | int == 18 }}
```

Use `midi2mqtt -test -all-events` to discover the exact CC numbers for each pad and control on your system.

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
