# TODOS — rust-egui-audio-eq

Parent intent for every item: [`PYRAMID-OF-INTENTS.md`](PYRAMID-OF-INTENTS.md).
Tags `[Ex/Gx]` point at the epic and goal each task serves. Order within a
phase is intentional; finish the prerequisites before starting dependent work.

Legend: `[ ]` open · `[x]` done · `[~]` in progress (note current state).

## Codebase audit (2026-11-05)

Compared this tracker with the Rust source, `Cargo.toml`, README, and existing
docs. The roadmap direction is still relevant, but several statuses and
prerequisites needed correction:

- Live RMS/peak meters already run in the callback and are displayed in the
  status bar; the old open item was stale.
- The preamp field has been added to `Params`, but is not yet connected to the
  renderer, UI, or response curve; treat it as in progress, not complete.
- The README on this branch links to `docs/PLATFORMS.md` and `docs/PRESETS.md`,
  but those files are not present in the docs directory. It also describes CI
  that is not present yet. Fix these links/claims or add the referenced assets.
- The current `EqEngine` is a UI/status contract and only `LiveEngine`
  implements it. Keep device-free DSP conformance tests separate from tests
  that need a real audio device.
- Equalizer APO and AutoEQ profiles can contain arbitrary frequencies, Q
  values, and filter types; implement a parameter/filter model capable of
  representing them before claiming general import/export compatibility.
- Existing callback state snapshots use an `RwLock`; review the real-time
  non-blocking requirement before expanding callback-shared state.

## P0 — Make it real (audible EQ on Linux) — complete

- [x] Environment probe (2026-10-15): Rust 1.98.1 / edition 2024, X11 +
      Wayland, ALSA and PulseAudio-compatible sink at 48 kHz, build
      dependencies present, crates.io reachable.
- [x] `[E1/G8]` Starter scaffold: eframe 0.36 / cpal 0.18 / egui_plot 0.37;
      window opens and exits cleanly. Feature selection reduced the dependency
      graph from 347 to 182 crates.
- [x] `[E2/G2]` RBJ biquads: peaking, shelves, LPF, and HPF with unit tests.
- [x] `[E2/G2,G5]` Eight-band `EqChain`, analytic magnitude response, and
      tests for transparency, gain, clamping, and time-domain agreement.
- [x] `[E2/G1]` Shared parameter snapshot and one-pole smoothing.
- [x] `[E6/G1]` Live cpal output on the default device; supported sample
      formats and startup/runtime errors surfaced without crashing.
- [x] `[E3/G1]` Tone, sweep, white noise, pink noise, and silence sources.
- [x] `[E1/G1,G5]` Main UI: band sliders, master gain, bypass, source picker,
      device/error status, and startup test hooks.
- [x] `[E5/G5]` Cached response curve and repaint cadence.
- [x] `[E5/G5]` RMS/peak meters computed by the callback and shown in the
      status bar. (Previously listed as open; verified in `src/engine/live.rs`,
      `src/engine/render.rs`, and `src/app/status.rs`.)
- [x] `[E7/G1]` Audible proof: sink-monitor capture verifies near-transparent
      bypass and measurable EQ gain; methodology and results are in
      `docs/DEVELOPMENT.md` and `docs/DSP.md`.
- [x] `[E7/G8]` Local quality gates were run and a Linux window/audio smoke
      test succeeded on the documented development machine. Keep re-running
      the gates for new changes; this historical check is not a CI workflow.

## P1 — Correct, interoperable, observable

### P1a — Complete the DSP/control contract first

- [~] `[E2/G2]` Preamp gain, −24…+12 dB: `Params.preamp_db` and boundary
      constants exist, but the renderer, smoothing, toolbar control, reset,
      response curve, and DSP tests still need integration. Preserve bypass
      semantics (bypass excludes preamp), clamp the final output, and test
      sanitization, audible gain, clipping, smoothing, and curve agreement.
- [ ] `[E2/G2]` Harden parameter/sample-rate boundaries: define finite-value
      fallback behavior for NaN/±Inf (including gains and preamp), guard invalid
      renderer sample rates before computing `dt`, and test that bad external
      values cannot produce NaN/Inf audio or coefficients.
- [ ] `[E2/G2]` Add a serializable parametric filter model for each band:
      frequency, supported filter kind (PK/LS/HS/notch), Q where applicable,
      enable state, and gain. Sanitize bounds, preserve the current 8-band
      graphic-EQ defaults, update the UI and curve from the same model, and add
      coefficient/response tests. This is a prerequisite for general APO/AutoEQ
      support, not merely a persistence task.
- [ ] `[E2/G1,G5]` Remove or bound potentially blocking locks from the audio
      callback parameter path. Select and document a snapshot handoff with
      bounded callback work; add stress/concurrency tests and keep UI updates
      from waiting on audio processing.

### P1b — Interoperability and persistence

- [ ] `[E4/G3]` Equalizer APO `.txt` import/export after the parametric model
      lands: parse preamp, `Filter: ON`, PK/LS/HS and `Filter: OFF`; ignore
      disabled filters, report malformed/unsupported enabled filters, preserve
      representable settings, and round-trip supported files. Compare a shared
      profile's response with `dart-flutter-audio-eq`.
- [ ] `[E4/G4]` Built-in Flat/Bass/Vocal/Treble presets and named user presets
      (CRUD), stored locally with schema/version handling; restore the last
      selected preset. Test persistence, malformed files, and migration/default
      behavior.
- [ ] `[E4/G3,G4]` AutoEQ starter bundle: at least 20 headphone profiles,
      searchable/selectable with measurement-source and profile attribution.
      Confirm redistribution terms and record the source/version for every
      bundled profile.

### P1c — Playback, engine portability, and evidence

- [ ] `[E3/G6]` File playback for WAV/FLAC/MP3/OGG using symphonia; play/pause,
      seek, loop, and track label. Decode outside the real-time callback and
      feed it through a bounded, preallocated handoff; test EOF, seek, decode
      errors, and format/rate conversion.
- [ ] `[E6/G1]` Device picker and explicit device/config negotiation; surface
      device removal/recovery in the status bar and keep the window alive.
      Test enumeration/error behavior on each supported OS and manually smoke
      test available devices.
- [ ] `[E1/G1,G5]` Offline renderer: render deterministic input and current
      supported parameters to WAV without a device; test file headers, sample
      count, clipping, and reference output.
- [ ] `[E1/G1,G2]` Shared device-free conformance tests for the DSP/render
      contract (gain, bypass, sanitization, clipping, response). Run against
      mock and offline render paths; document which LiveEngine checks require a
      physical/virtual device rather than implying ordinary unit tests cover
      device behavior.
- [ ] `[E2/G2]` Golden vectors generated from an independent RBJ reference,
      consumed by Rust tests for representative filter types, gains, Q values,
      and edge frequencies. Record reference provenance and numeric tolerance.
- [ ] `[E7/G8]` Update architecture, DSP, and development docs when the
      parametric model, preamp, presets, or playback behavior changes. Keep
      README links valid; add `docs/PRESETS.md` / `docs/PLATFORMS.md` only when
      those documents exist and are maintained.

## P2 — Cross-platform and shippable

- [ ] `[E7/G8]` Add GitHub Actions CI: fmt, clippy, and tests on pull requests
      and pushes; monthly dependency-update report/check; tag-triggered build
      matrix. Install and document required Linux build dependencies. CI build
      success is not a hardware smoke test.
- [ ] `[E1/G7]` Windows build and runtime smoke test (device enumeration and
      audible test signal); fix platform issues without platform-specific DSP.
- [ ] `[E1/G7]` macOS build and CoreAudio runtime smoke test (device
      enumeration and audible test signal).
- [ ] `[E7/G8]` Maintain a tested-platform matrix with OS/version, Rust
      version, audio backend/device, sample format/rate, and evidence. Separate
      compile-only targets from verified runtime targets.
- [ ] `[E8/G8]` Decide and add the project license before importing
      license-sensitive profile data, accepting outside contributions, or
      publishing release artifacts.
- [ ] `[E8/G8]` Define release versioning and app identity; choose one Linux
      package format; automate reproducible Linux and Windows artifacts and
      document installation/upgrade/uninstall behavior.
- [ ] `[E5/G5]` Optional live FFT spectrum overlay, with windowing/averaging
      and callback-safe data transfer; profile CPU cost and keep it optional.

## Backlog (do not promote without a goal)

- System-wide EQ via PipeWire/Pulse hooks — sibling-app lane and current
  non-goal.
- DAW plugin formats (VST/AU/LV2/CLAP) — non-goal unless scope changes.
- Convolution/FIR engine — only if IIR proves insufficient.
- Live-stream recording/capture-and-save, per-app profiles, loudness
  compensation, crossfeed, MIDI/controller mapping, URL/radio playback,
  onboarding, i18n, and cloud sync — require explicit goal/scope review first.
- WASM — cpal does not support the intended audio path; conflicts with the
  desktop-audio vision unless separately re-scoped.

## Working agreement

- Keep one task in flight at a time; mark `[~]` with date and blocker when
  paused. Do not mark work complete until its acceptance checks pass.
- Every task must support a `[Ex/Gx]`; pyramid conflicts are re-scoped, not
  merged.
- Keep this tracker aligned with the actual code and docs; remove stale tasks
  instead of preserving them as roadmap noise.
- Review the backlog monthly and promote at most two items at a time.
