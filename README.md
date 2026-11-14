# whisper-ptt

Push-to-talk dictation for Sway. Hold the **show-all-windows key** (top row,
sends F5) and speak. When you release it, the audio is transcribed by a
Whisper server on the LAN and the text is typed into the focused window.

The current setup is the Chromebook **chromenstein** (Google Terra / ASUS
C202SA, Debian 13, Sway) using the GPU Whisper server on the homelab desktop.
A GNOME (Wayland) port for a Dell XPS 13 9360 lives in `gnome/`. It is also
hold-to-talk, on F7. See [GNOME port](#gnome-port).

## How it works

```
F5 down ──► ptt start ──► parecord (16 kHz mono WAV)
F5 up   ──► ptt stop  ──► SIGINT recorder
                              ──► POST /v1/audio/transcriptions (desktop)
                              ──► wtype "<text>" into focused window
```

- **Server:** `whisper-server` 0.1.0 on a LAN desktop, port 8100. It has an
  OpenAI-compatible API and runs model `large-v3-turbo` on CUDA (int8_float16).
  A short clip takes about 1 s round trip.
- **Hotkey:** the show-all-windows (overview) key, which sends plain `F5`
  because keyd only remaps F6–F10. It was originally Search+T, which needed
  stop bindings on the release of both T and Super_L, since a `$mod+t` release
  binding doesn't fire if Search is let go first. A single key avoids that.
  `ptt stop` still does nothing if no recording is in progress.
- **Safety cap:** recording stops after 120 s if a release event is ever missed.
- **Short taps:** clips under about 0.3 s are discarded without being sent.
- **Waybar indicator:** a mic pill just left of the clock. It's green while
  recording, orange while transcribing, flashes green for 0.4 s once the text
  is typed, then hides. `ptt` writes the state as Waybar JSON and sends
  `SIGRTMIN+8`, so Waybar redraws only on changes. Nothing polls, and it costs
  nothing when idle. A short tap or a failed request hides it right away.
- **Runtime files:** `$XDG_RUNTIME_DIR/ptt/` holds `rec.pid`, `rec.wav` (the
  last clip), `state.json` (indicator state) and `ptt.log` (errors).

## Files

| Repo path | Deployed to |
|---|---|
| `bin/ptt` | `~/.local/bin/ptt` |
| `config.example` | `~/.config/ptt/config` (local settings, e.g. server URL) |
| `sway/ptt.conf` | pasted into `~/.config/sway/config` after the `$mod+n` draft binding |
| `waybar/ptt.jsonc` | module definition in `~/.config/waybar/config.jsonc`; `"custom/ptt"` goes first in `modules-center`, before `"clock"` |
| `waybar/ptt.css` | appended to `~/.config/waybar/style.css`; `#custom-ptt` also joins the shared pill selector |
| `docs/internal-mic.md` | notes on the Terra internal-mic fix (DMIC2) |

## Install

```sh
sudo apt install wtype alsa-utils        # pulseaudio-utils, curl already present
install -m755 bin/ptt ~/.local/bin/ptt
install -Dm644 config.example ~/.config/ptt/config   # then set your server URL
# add sway/ptt.conf contents to ~/.config/sway/config, then:
swaymsg reload
```

Settings come from `~/.config/ptt/config`, which `ptt` sources on every run
and which is kept out of the repo. Environment variables work too. Defaults:

| Variable | Default |
|---|---|
| `PTT_WHISPER_URL` | `http://whisper-host:8100/v1/audio/transcriptions` |
| `PTT_MODEL` | `large-v3-turbo` |

The config file is the easy way, because Sway's `exec` doesn't pick up shell
rc files. `whisper-host` is a placeholder: point `PTT_WHISPER_URL` at your own
OpenAI-compatible Whisper server.

## Gotchas found while building it

1. **`parecord` loses the end of the clip.** By default it buffers about 1 s
   and throws that buffer away on SIGINT, so a short press produced a
   header-only, 44-byte WAV. `--latency-msec=20` fixes it: a 3 s hold now
   captures 2.94 s instead of 1.98 s.
2. **The internal mic records pure digital silence (about −91 dBFS).** The
   Debian UCM profile routes the Terra's internal mic to DMIC1, but the mic is
   wired to DMIC2. `ptt start` sets `Stereo1 DMIC Mux` to DMIC2 before every
   recording. See [docs/internal-mic.md](docs/internal-mic.md).
3. **Nothing on screen showed recording was active.** Fixed with the Waybar
   indicator. It uses a signal rather than polling, so it costs nothing when
   idle.

## Status

Working end to end as of 2026-11-12: dictated text was typed into a foot
terminal.

## GNOME port

`gnome/` runs the same pipeline on GNOME Wayland (tested on Fedora Silverblue 45,
GNOME 51, Dell XPS 13 9360). Hold the unlabeled **F7** key, speak, and release.
Two things in the Sway version don't carry over to GNOME:

- **No release events.** GNOME custom shortcuts fire only on key press, so they
  can't do hold-to-talk. `ptt-daemon` reads the laptop keyboard's evdev node
  directly and runs `ptt-gnome start` on key down and `ptt-gnome stop` on key up.
- **No `wtype`.** Mutter doesn't implement the virtual-keyboard protocol.
  `ptt-daemon` creates its own uinput virtual keyboard and types the transcript
  with it, assuming a US layout. Text it can't type (for example accented
  letters) goes to the clipboard with a "Ctrl+V to paste" notification. Smart
  quotes and dashes are converted to plain ASCII first.

`ptt-daemon` uses only the Python standard library, with no `python-evdev` and
no `ydotool`, so there's nothing to layer on rpm-ostree and no third-party code
to vet. A GNOME notification takes the place of the Waybar indicator.

```
F7 down ──► ptt-daemon (evdev) ──► ptt-gnome start ──► parecord
F7 up   ──► ptt-daemon ──► ptt-gnome stop ──► whisper-server
                       ◄── transcript ◄──┘
        ──► uinput virtual keyboard types it into the focused window
```

| Repo path | Deployed to |
|---|---|
| `gnome/ptt-gnome` | `~/.local/bin/ptt-gnome` (record + transcribe; `toggle` works standalone, output to clipboard) |
| `gnome/ptt-daemon` | `~/.local/bin/ptt-daemon` (key listener + typer) |
| `gnome/systemd/ptt-daemon.service` | `~/.config/systemd/user/` |
| `gnome/udev/70-ptt.rules` | `/etc/udev/rules.d/`: `uaccess` on the built-in keyboard and `/dev/uinput` |
| `gnome/hwdb/70-ptt.hwdb` | `/etc/udev/hwdb.d/`: remaps F7 (scancode `0x41`) to `KEY_MACRO1` |

Install:

```sh
install -m755 gnome/ptt-gnome gnome/ptt-daemon ~/.local/bin/
install -Dm644 config.example ~/.config/ptt/config      # set PTT_WHISPER_URL
sudo install -m644 gnome/udev/70-ptt.rules /etc/udev/rules.d/
sudo install -m644 gnome/hwdb/70-ptt.hwdb /etc/udev/hwdb.d/
sudo udevadm control --reload && sudo systemd-hwdb update
sudo udevadm trigger -c change -s input -s misc
install -Dm644 gnome/systemd/ptt-daemon.service ~/.config/systemd/user/
systemctl --user enable --now ptt-daemon.service
```

Notes:

- **Why remap F7?** The daemon only watches the key; it doesn't grab it, because
  grabbing would capture the whole keyboard. Unmapped, F7 would also reach the
  focused app, and in Firefox it toggles caret browsing. `KEY_MACRO1` (656) is
  above the xkb keycode range, so no app sees it. The daemon listens for both
  656 and 65 (plain F7), so it still works if the remap isn't installed.
- **Choosing a different key:** run `ptt-daemon --debug`, press the key to see
  its code and scancode, then change the hwdb entry or set `PTT_KEYCODES`.
  `PTT_KBD_NAME` selects the input device (default: `AT Translated Set 2 keyboard`).
- **Security trade-off:** the `uaccess` rule lets any process in your session
  read the built-in keyboard and inject keys through uinput. That's the same
  exposure as being in the `input` group.
- **Debugging:** `journalctl --user -u ptt-daemon` and `$XDG_RUNTIME_DIR/ptt/ptt.log`.

Status: working end to end on 2026-11-14 (a message was dictated into a Claude Code
session).

## Possible follow-ups

- Make the DMIC2 fix system-wide with a UCM override, so other apps such as
  video calls get the internal mic too.
- Add a toggle mode (press once to start, press again to stop) as an
  alternative to holding the keys. `ptt-gnome toggle` already does this on GNOME.
- GNOME: support layouts other than US by reading the xkb keymap instead of using
  a fixed table.
