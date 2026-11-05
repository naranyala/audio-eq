# rust-egui-audio-eq

A native **8-band graphic equalizer for Linux desktops**, written in Rust:
[eframe/egui](https://docs.rs/eframe) for the window and UI,
[cpal](https://docs.rs/cpal) for real-time audio output, and
[egui_plot](https://docs.rs/egui_plot) for the response curve.

The app generates test signals and processes them through its own live DSP
chain. It is an **in-app** equalizer, not a system-wide audio effect. File
playback is planned. The project vision and non-goals are in
[`PYRAMID-OF-INTENTS.md`](PYRAMID-OF-INTENTS.md); planned work and status are
tracked in [`TODOS.md`](TODOS.md).

> **Current status:** The Linux test-signal EQ is implemented. Preamp gain has
> been added to the parameter model and renderer, and the response-curve code
> accepts preamp gain. However, the toolbar and app call sites have not been
> fully integrated with those changes. Treat preamp as **in progress**, not as
> a usable UI feature. Re-run the build and quality checks before relying on
> this checkout; the verification results below describe the earlier baseline.

## Features

**Implemented in the Linux test-signal app**

- **8-band graphic EQ** — 60 / 150 / 400 / 1k / 2.4k / 6k / 12k / 16k Hz,
  ±12 dB per band (low shelf on band 1, high shelf on band 8, peaking Q = 1.0)
- **Master gain** (−24 … +6 dB) and bit-exact bypass for A/B comparison
- **Test signals** — 440 Hz tone, 200 Hz → 8 kHz sweep, white noise,
  pink noise, and silence
- **Response curve** for the graphic EQ on a 20 Hz–20 kHz log-frequency axis
- **Live meters** — callback RMS/peak levels and device, rate, format, and
  stream-error status in the status bar
- **Smoothed band and master controls**
- **Headless startup hooks** — `EQ_INITIAL_GAINS` and `EQ_INITIAL_BYPASS`

**In progress** — preamp gain is represented in shared parameters and the
renderer; finishing its UI/app integration and verifying the current checkout
are still required.

**Not implemented** — file playback, editable parametric filters, Equalizer
APO import/export, AutoEQ profiles, saved presets, device selection/recovery,
offline rendering, FFT overlay, Windows/macOS runtime support, and automated
CI. See [`TODOS.md`](TODOS.md) for priorities and acceptance criteria.

## Verification

The project previously recorded 36 passing tests, clean clippy/fmt checks, a
Linux window/audio smoke run, and a PipeWire sink-monitor capture. These are
**historical baseline results**, not verification of the current in-progress
preamp integration. Re-run the checks for the current checkout:

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
cargo run
```

Historical sink-monitor results: bypass was within 0.12 dB of flat, and a
+6 dB EQ boost measured +6.1 dB relative to flat. The capture method and full
results are documented in [`docs/DEVELOPMENT.md`](docs/DEVELOPMENT.md) and
[`docs/DSP.md`](docs/DSP.md).

## Getting started

### Prerequisites

- Rust stable (the documented development environment used Rust 1.98.1,
  edition 2024)
- Linux audio stack with ALSA and PulseAudio or PipeWire
- Build dependencies (Fedora/AlmaLinux package names): `pkg-config`,
  `alsa-lib`, `libxkbcommon`, and xcb/X11 development packages

### Build and run

```bash
cargo run
cargo test
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

Scripted runs without changing controls in the UI:

```bash
EQ_INITIAL_GAINS=0,6,6,0,0,0,0,0 cargo run
EQ_INITIAL_BYPASS=1 cargo run
```

`EQ_INITIAL_GAINS` takes exactly eight comma-separated band gains in dB;
`EQ_INITIAL_BYPASS` accepts `1`, `true`, or `yes`.

## Using the app

| Region | Contents |
|---|---|
| Top bar | source picker (tone / sweep / white / pink / silence), bypass checkbox, master gain, reset |
| Left panel | eight vertical band sliders (±12 dB), labeled by frequency and filter type (LS / PK / HS) |
| Center | cached EQ response curve on a log-frequency axis |
| Bottom bar | device name · sample rate · sample format · channels, stream errors, RMS/peak meters, BYPASS badge |

The preamp is not yet exposed as a toolbar control. If audio cannot start,
the window reports the startup error in the status bar rather than crashing.

## Dependencies

The project has three direct crates:

| Crate | Version | Role |
|---|---|---|
| `eframe` | 0.36 | App shell, window, and egui integration (glow renderer) |
| `cpal` | 0.18 | Real-time output stream |
| `egui_plot` | 0.37 | Log-frequency response plot |

**Version pairing:** egui 0.36 and egui_plot 0.37 belong together. cpal 0.18
uses `Device::description()` and a plain `u32` sample rate. Dependency feature
choices and audit details are documented in
[`docs/DEPENDENCIES.md`](docs/DEPENDENCIES.md).

There is no persistence or file decoder dependency yet. DSP is implemented in
[`src/dsp/`](src/dsp/) with RBJ biquads and no external DSP crate.

## Project layout

```
src/
├── main.rs          eframe entry point
├── app/             toolbar, band controls, status, response curve
├── engine/          EqEngine contract, live cpal engine, pure render core
├── state.rs         Params and UI/audio shared state
└── dsp/             biquad · EQ chain · smoothing · test-signal sources
docs/              architecture · DSP · development · dependencies
TODOS.md           phased roadmap and acceptance checks
PYRAMID-OF-INTENTS.md   vision, principles, goals, and epics
```

## Documentation

- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — threads, data flow, and
  real-time invariants
- [`docs/DSP.md`](docs/DSP.md) — filter math, bypass, meters, and historical
  audible verification
- [`docs/DEVELOPMENT.md`](docs/DEVELOPMENT.md) — environment, commands, tests,
  and audio capture recipe
- [`docs/DEPENDENCIES.md`](docs/DEPENDENCIES.md) — crate rationale and feature
  selection
- [`TODOS.md`](TODOS.md) — current work and roadmap
- [`PYRAMID-OF-INTENTS.md`](PYRAMID-OF-INTENTS.md) — project intent and scope

## Roadmap

- **P1 — trustworthy and interoperable:** finish preamp integration; harden
  numeric boundaries; add parametric filters, APO import/export, profiles and
  saved presets; add file playback, device selection, offline rendering, and
  engine conformance tests
- **P2 — cross-platform and shippable:** CI, Windows/macOS builds and runtime
  tests, platform matrix, license decision, packaging, and optional FFT overlay

## License

No `LICENSE` file is present. GPL-3.0 has been proposed, but the project license
must be decided before importing license-sensitive code or accepting outside
contributions.
