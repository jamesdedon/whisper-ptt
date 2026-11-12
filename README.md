# whisper-ptt

Push-to-talk dictation for Sway. Hold the **show-all-windows key** (top row,
sends F5) and speak. When you release it, the audio is transcribed by a
Whisper server on the LAN and the text is typed into the focused window.

The current setup is the Chromebook **chromenstein** (Google Terra / ASUS
C202SA, Debian 13, Sway) using the GPU Whisper server on the homelab desktop.

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

## Possible follow-ups

- Make the DMIC2 fix system-wide with a UCM override, so other apps such as
  video calls get the internal mic too.
- Add a toggle mode (press once to start, press again to stop) as an
  alternative to holding the keys.
