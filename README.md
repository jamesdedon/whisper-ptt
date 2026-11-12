# whisper-ptt

Push-to-talk dictation for Sway. Hold **Search+T** and speak. When you release
the keys, the audio is transcribed by a Whisper server on the LAN and the text
is typed into the focused window.

The current setup is the Chromebook **chromenstein** (Google Terra / ASUS
C202SA, Debian 13, Sway) using the GPU Whisper server on the homelab desktop.

## How it works

```
Search+T down ──► ptt start ──► parecord (16 kHz mono WAV)
Search+T up   ──► ptt stop  ──► SIGINT recorder
                              ──► POST /v1/audio/transcriptions (desktop)
                              ──► wtype "<text>" into focused window
```

- **Server:** `whisper-server` 0.1.0 at `http://whisper-host:8100`. It has an
  OpenAI-compatible API and runs model `large-v3-turbo` on CUDA (int8_float16).
  A short clip takes about 1 s round trip.
- **Release handling:** the stop binding runs when either T or Search is
  released first. `ptt stop` does nothing if no recording is in progress, so
  extra releases are harmless.
- **Safety cap:** recording stops after 120 s if a release event is ever missed.
- **Short taps:** clips under about 0.3 s are discarded without being sent.
- **Runtime files:** `$XDG_RUNTIME_DIR/ptt/` holds `rec.pid`, `rec.wav` (the
  last clip) and `ptt.log` (errors).

## Files

| Repo path | Deployed to |
|---|---|
| `bin/ptt` | `~/.local/bin/ptt` |
| `sway/ptt.conf` | pasted into `~/.config/sway/config` after the `$mod+n` draft binding |
| `docs/internal-mic.md` | notes on the Terra internal-mic fix (DMIC2) |

## Install

```sh
sudo apt install wtype alsa-utils        # pulseaudio-utils, curl already present
install -m755 bin/ptt ~/.local/bin/ptt
# add sway/ptt.conf contents to ~/.config/sway/config, then:
swaymsg reload
```

Configuration is read from environment variables, with defaults set in the script:

| Variable | Default |
|---|---|
| `PTT_WHISPER_URL` | `http://whisper-host:8100/v1/audio/transcriptions` |
| `PTT_MODEL` | `large-v3-turbo` |

Sway's `exec` doesn't pick up shell rc files, so to change these, either edit
the defaults in the script or put the variables in the binding, e.g.
`exec env PTT_MODEL=... ptt start`.

## Gotchas found while building it

1. **`parecord` loses the end of the clip.** By default it buffers about 1 s
   and throws that buffer away on SIGINT, so a short press produced a
   header-only, 44-byte WAV. `--latency-msec=20` fixes it: a 3 s hold now
   captures 2.94 s instead of 1.98 s.
2. **The internal mic records pure digital silence (about −91 dBFS).** The
   Debian UCM profile routes the Terra's internal mic to DMIC1, but the mic is
   wired to DMIC2. `ptt start` sets `Stereo1 DMIC Mux` to DMIC2 before every
   recording. See [docs/internal-mic.md](docs/internal-mic.md).
3. **Nothing on screen shows recording is active.** That's by design for now.
   A waybar indicator is a possible follow-up.

## Status

Working end to end as of 2026-11-12: dictated text was typed into a foot
terminal.

## Possible follow-ups

- Make the DMIC2 fix system-wide with a UCM override, so other apps such as
  video calls get the internal mic too.
- Add a waybar indicator that shows while recording or transcribing.
- Add a toggle mode (press once to start, press again to stop) as an
  alternative to holding the keys.
