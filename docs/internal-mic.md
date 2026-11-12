# Terra internal mic: DMIC1 vs DMIC2

## Symptom

The built-in mic records pure digital silence: mean about −91 dBFS and tail
RMS 0–1, including after the recorder's start-up pop decays. PulseAudio shows
the source `alsa_input.platform-cht-bsw-rt5645.HiFi__Mic1__source` unmuted at
100%, so nothing looks wrong at that level.

## Hardware

- Card 0: `chtrt5650` (`Google-Terra-1.0`), Cherry Trail SST (`intel_sst_acpi`)
  with an RT5650 codec
- UCM profile: `/usr/share/alsa/ucm2/Intel/chtrt5650/HiFi.conf`. Its `Mic1`
  device turns on `Int Mic` and the `Sto1 ADC MIX* ADC2` (DMIC) path.
- The shared verb include `/usr/share/alsa/ucm2/codecs/rt5645/EnableSeq.conf`
  sets `cset "name='Stereo1 DMIC Mux' 0"`, which selects **DMIC1**.

## Diagnosis

Each path was tested by recording 4 s and measuring RMS after the first 2 s,
which skips the ADC start-up pop:

| Path | Tail level |
|---|---|
| `Stereo1 DMIC Mux` = DMIC1 (UCM default) | −95.7 dBFS (dead) |
| `Stereo1 DMIC Mux` = DMIC2 | **−64.0 dBFS (room noise, live)** |
| Analog IN2 via ADC1 (`Int Analog Mic`, `RECMIX BST2`, `IN2 Boost` 8) | −71.1 dBFS |

DMIC2 is the internal mic. Speech transcribes correctly through it.

A misleading result along the way: right after `alsa-utils` was installed, the
whole-file mean jumped to about −22 dBFS. That came entirely from a sub-60 Hz
start-up pop in the first ~0.3 s. Always measure the tail of the clip, not the
whole file.

The kernel's `snd_soc_rt5645` `quirk` parameter is unset (`4294967295`).

## Current fix (user-level, PTT only)

`ptt start` runs this before each recording:

```sh
amixer -c 0 -q sset "Stereo1 DMIC Mux" DMIC2
```

This is needed because the UCM EnableSeq resets the mux to DMIC1 whenever
PulseAudio initializes the card, which happens at login and possibly after
resume. For the same reason, `alsactl store` plus `alsa-restore.service` won't
keep the setting.

## System-wide fix (not done yet)

Override the UCM so every app gets the working mic. One option is adding
`cset "name='Stereo1 DMIC Mux' DMIC2"` to the `Mic1` EnableSequence in a local
copy of the chtrt5650 profile. Any edit under `/usr/share/alsa/ucm2` is
overwritten by `alsa-ucm-conf` updates, so it needs either a
dpkg-divert/override or a re-apply hook.

## Saved mixer state

`docs/asound-baseline.state` is a snapshot of the original (UCM default) card 0
mixer state, saved during debugging. Restore it with:

```sh
/usr/sbin/alsactl -f docs/asound-baseline.state restore 0
```
