# Pitch Study — implementation plan

Oct 4, 2026 · @David Olivari

## How to use this plan

Build Pitch Study one task at a time, in the order of the step-by-step section, and do not move on until the task's acceptance check passes. The plan is written for an implementing model that follows instructions literally, so every rule below is binding.

1. **One task at a time.** Each task names its files, what to write and an acceptance check. Run the check, report the result, then start the next task.
2. **Never invent APIs.** If you are unsure how a crate or package works, read its documentation (docs.rs, the npm README) before writing code. Use the pinned versions in the guidelines section.
3. **Do not change locked decisions, constants or data contracts** (sections 2, 5 and 6) without asking David first. If a contract seems wrong, stop and explain why.
4. **The sample page is the visual and behavioural reference:** [Pitch Study UI/UX sample](https://claude.ai/artifact/MGXRU92T9DBG4JCC6d3pZu). When a task says "port from the sample", copy its behaviour and constants exactly, then fit them into the module structure given here.
5. **Keep the audio callback real-time safe.** Inside the cpal callback: no allocation, no locks, no logging, no I/O, no panics. Only copy samples into the ring buffer.
6. **English only** for code, comments, identifiers, commit messages and UI copy.
7. **Stop after two failed attempts** at the same check. Report what you tried, the exact error and your hypothesis, then wait.
8. **Commit after each passing task** with the message `M<milestone>.<task>: <task title>`, for example `M2.3: YIN detector`.
9. **Never store audio recordings and never commit secrets.** The MongoDB connection string lives only in the OS keychain or an environment variable.
10. **Ask before adding any dependency** that is not listed in the guidelines section.

## Product summary and locked decisions

Pitch Study is a single-user desktop app that trains singing in tune: it plays a target note, listens to the voice, shows in a 2.5D diorama whether the voice is higher or lower, and adapts precision as the singer improves. It is used at a desktop with an external audio interface and headphones.

The core loop for each note: play the reference tone, the singer sings, the mouth rises or falls with the pitch error, and the note passes when the voice stays inside the gate for 0.7 s. Two clean passes in a row narrow every gate; a miss widens them.

| Area | Decision |
| --- | --- |
| Platform | Desktop only (macOS, Windows, Linux), Tauri 2 shell |
| Audio core | Rust crate: capture with `cpal`, reference tones synthesised in Rust, MIDI with `midir` |
| Pitch detection | YIN in Rust, validated offline against WAV files before any UI |
| UI | Svelte 5 + TypeScript + Vite inside the Tauri webview |
| 3D | Three.js in the webview, ported from the sample page |
| Storage | MongoDB (Atlas or local), written through a local JSONL outbox so practice never depends on the network |
| Data kept | Attempt summaries and downsampled pitch contours; never audio |
| Monitoring | The singer hears their own voice through the interface's hardware direct monitoring, never through software |
| Visual style | Low-poly flat toon diorama with a Swiss typographic HUD, jade water `#93c4ac`, red reserved for gates |
| Feedback modes | Guided (mouth, numbers, doubled tone), Trace (mouth only), Unaided (scored afterwards) |
| Progress metric | Median settled error in cents on Unaided notes only |
| Exercises | Major, natural minor, major and minor pentatonic, major, minor and dominant seventh arpeggios, from C4, octave-folded |

## Architecture overview

Four Rust threads own everything time-critical: capture, analysis, the session state machine and output. The webview receives events and sends commands; storage happens off the critical path through an outbox.

&#91;embedded content: architecture · 5 Rust parts, 2 webview parts, MongoDB\]

Samples flow left to right through a lock-free ring buffer into YIN; the runner turns pitch frames into phases, results and tone commands, and hands finished attempts to the store task, which writes the outbox before MongoDB.

## Repository layout

One Cargo workspace with a pure Rust core crate, a Tauri app crate, and a Svelte UI folder. The core crate has no Tauri, no threads and no device access, so everything in it is unit-testable from WAV files.

```text
pitch-study/
  Cargo.toml                  workspace members: crates/pitch-core, src-tauri
  crates/pitch-core/          pure logic, no I/O except WAV reading in the CLI
    src/lib.rs
    src/exercise.rs           exercise definitions, note names, frequencies
    src/cents.rs              Hz <-> cents, octave folding
    src/yin.rs                YIN pitch detector
    src/smoothing.rs          confidence gate, 5-frame median filter
    src/scoring.rs            attempt evaluation: hold, timeout, metrics
    src/staircase.rs          adaptive tolerance
    src/session.rs            phase state machine (listen, sing, result, travel, summary)
    src/bin/analyze.rs        CLI: WAV in, CSV of pitch frames out
    tests/                    integration tests on synthetic WAVs
  src-tauri/
    tauri.conf.json
    Info.plist                macOS microphone usage description
    src/main.rs               app setup, state, command registration
    src/audio/devices.rs      list and select input/output devices
    src/audio/capture.rs      cpal input stream -> lock-free ring buffer
    src/audio/output.rs       cpal output stream, reference tone synth
    src/engine.rs             analysis thread: ring buffer -> PitchFrame
    src/runner.rs             drives pitch-core session, emits events
    src/store/outbox.rs       JSONL append and background flush
    src/store/mongo.rs        MongoDB writes and progress queries
    src/midi.rs               optional MIDI export of exercises
    src/commands.rs           #[tauri::command] functions
  ui/
    src/main.ts, App.svelte
    src/lib/contracts.ts      TypeScript mirrors of the Rust contracts
    src/lib/ipc.ts            invoke and listen wrappers, nothing else
    src/lib/state.svelte.ts   app state with Svelte 5 runes
    src/scene/                Three.js diorama, ported from the sample
      palette.ts toonMaterial.ts shadows.ts island.ts garden.ts
      staircase.ts gates.ts mouth.ts trail.ts camera.ts Scene.ts
    src/hud/                  Svelte components for the Swiss HUD
      TopBar.svelte TargetPanel.svelte StepStrip.svelte
      ToleranceStat.svelte ResultLog.svelte
      IntroPoster.svelte SummaryPoster.svelte
      research/ResearchPage.svelte research/figures/*.svelte
    src/styles/tokens.css grid.css
  tools/validate_detector.py  Python ground-truth comparison (librosa pYIN)
  fixtures/audio/             synthetic and recorded WAVs (large files gitignored)
```

Rust is the single source of truth for scoring and phases. The UI only displays what the runner emits and sends user commands back; it never computes a score.

## Data contracts

All contracts are defined once in Rust in `pitch-core/src/contracts.rs`, serialised with serde in camelCase, and exported to TypeScript with `ts-rs` into `ui/src/lib/generated/`. Never hand-edit the generated files; change the Rust type and re-export.

### Core types

```rust
// pitch-core/src/contracts.rs
use serde::{Deserialize, Serialize};
use ts_rs::TS;

#[derive(Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize, TS)]
#[serde(rename_all = "camelCase")]
pub enum ExerciseId { Major, Minor, PentaMaj, PentaMin, ArpMaj, ArpMin, ArpDom7 }

#[derive(Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize, TS)]
#[serde(rename_all = "camelCase")]
pub enum FeedbackMode { Guided, Trace, Unaided }

#[derive(Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize, TS)]
#[serde(rename_all = "camelCase")]
pub enum Phase { Idle, Listen, Sing, Result, Travel, Summary }

#[derive(Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize, TS)]
#[serde(rename_all = "camelCase")]
pub enum InputSource { Mic, Demo }

#[derive(Clone, Debug, Serialize, Deserialize, TS)]
#[serde(rename_all = "camelCase")]
pub struct Target { pub index: u32, pub semitone: i32, pub name: String, pub freq_hz: f32 }

#[derive(Clone, Debug, Serialize, Deserialize, TS)]
#[serde(rename_all = "camelCase")]
pub struct PitchFrame {
    pub t_ms: u32,             // time since the Sing phase started
    pub cents: Option<f32>,    // octave-folded error vs target, None when unvoiced
    pub confidence: f32,       // 1 - YIN aperiodicity, 0..1
    pub rms: f32,
}

#[derive(Clone, Debug, Serialize, Deserialize, TS)]
#[serde(rename_all = "camelCase")]
pub struct Contour { pub hop_ms: u16, pub cents: Vec<Option<i16>> } // 20 ms hops

#[derive(Clone, Debug, Serialize, Deserialize, TS)]
#[serde(rename_all = "camelCase")]
pub struct AttemptResult {
    pub target: Target,
    pub passed: bool,
    pub settled_cents: Option<f32>,
    pub onset_cents: Option<f32>,
    pub settle_time_ms: Option<u32>,
    pub stability_sd_cents: Option<f32>,
    pub voiced_ratio: f32,
    pub tolerance_cents: u8,
    pub feedback_mode: FeedbackMode,
    pub contour: Contour,
}

#[derive(Clone, Debug, Serialize, Deserialize, TS)]
#[serde(rename_all = "camelCase")]
pub struct ToleranceState { pub cents: u8, pub streak: u8 }

#[derive(Clone, Debug, Serialize, Deserialize, TS)]
#[serde(rename_all = "camelCase")]
pub struct PhaseEvent { pub phase: Phase, pub step: u32, pub target: Option<Target>, pub duration_ms: u32 }

#[derive(Clone, Debug, Serialize, Deserialize, TS)]
#[serde(rename_all = "camelCase")]
pub struct SessionSummary {
    pub passed: u32, pub total: u32,
    pub median_abs_settled_cents: Option<f32>,
    pub tolerance_start: u8, pub tolerance_end: u8,
}
```

### Events (Rust to UI)

| Event name | Payload | When |
| --- | --- | --- |
| `pitch://frame` | `PitchFrame` | Every analysis hop (about 100 per second), only during Sing |
| `session://phase` | `PhaseEvent` | On every phase change |
| `session://result` | `AttemptResult` | When a note passes or times out |
| `session://tolerance` | `ToleranceState` | After every result |
| `session://summary` | `SessionSummary` | When the last note's result is shown |
| `session://targets` | `Target[]` | When a session starts or the exercise changes |
| `audio://status` | `{ input, output, sampleRate, running, error }` | On start, device change or stream error |

### Commands (UI to Rust)

| Command | Arguments | Returns |
| --- | --- | --- |
| `list_devices` | none | `{ inputs: string[], outputs: string[] }` |
| `select_devices` | `input, output` | audio status |
| `set_input_source` | `InputSource` | nothing |
| `start_session` | `exercise, mode` | `{ sessionId, targets }` |
| `set_exercise` | `exercise` | `{ targets }` (restarts the run) |
| `set_feedback_mode` | `mode` | nothing |
| `replay_tone` | none | nothing |
| `stop_session` | none | nothing |
| `get_progress` | `days` | `{ day, medianAbsCents, n }[]` from Unaided attempts |

### MongoDB documents

Database `pitch_study`. Every `_id` is a UUID v7 string generated in Rust, so retried writes are idempotent upserts.

```json
// sessions
{ "_id": "0192...", "startedAt": "2026-10-04T18:02:00Z", "endedAt": null,
  "exercise": "major", "toleranceStart": 40, "toleranceEnd": null,
  "device": { "input": "...", "output": "...", "sampleRate": 48000, "bufferFrames": 256 },
  "appVersion": "0.1.0", "schemaVersion": 1 }

// attempts: one AttemptResult plus these fields
{ "_id": "0192...", "sessionId": "0192...", "ts": "2026-10-04T18:05:12Z",
  "exercise": "major", "schemaVersion": 1, "...AttemptResult fields": "..." }
```

Indexes on `attempts`: `{ sessionId: 1, ts: 1 }`, `{ feedbackMode: 1, ts: -1 }`, `{ "target.semitone": 1, ts: -1 }`.

### Outbox line format

File `outbox.jsonl` in the Tauri app data directory, one JSON object per line: `{ "op": "upsertSession" | "upsertAttempt", "id": "<_id>", "doc": { ... } }`. The flusher upserts each line by `_id`, then removes flushed lines by rewriting the file through a temporary file and an atomic rename.

## Constants and algorithms

Every number below already runs in the sample page; reuse them exactly. Put them in `pitch-core/src/constants.rs` as `pub const` items with these names.

### Constants

| Name | Value | Meaning |
| --- | --- | --- |
| `C4_HZ` | 261.6256 | Root of every exercise |
| `YIN_WINDOW` | 1024 samples | Difference window |
| `ANALYSIS_BUFFER` | 2048 samples | Samples per analysis (window + max lag) |
| `HOP_MS` | 10 ms | Time between analyses (480 samples at 48 kHz) |
| `YIN_THRESHOLD` | 0.15 | First dip below this is the period |
| `F_MIN_HZ` / `F_MAX_HZ` | 65 / 1100 | Search range for the period |
| `RMS_GATE` | 0.008 | Below this RMS the frame is unvoiced |
| `MEDIAN_FRAMES` | 5 | Median filter length on cents |
| `LISTEN_MS` | 1100 | Reference tone phase |
| `HOLD_MS` | 700 | Time inside tolerance needed to pass |
| `NOTE_MAX_MS` | 5500 | Sing phase timeout (counts as a miss) |
| `RESULT_MS` | 800 | Result display before travel |
| `TRAVEL_MS` / `RESTART_TRAVEL_MS` | 1100 / 1600 | Mouth travel to the next note / back to note 1 |
| `ONSET_WINDOW_MS` | 150 | Onset error window after first voiced frame |
| `MISS_SETTLE_WINDOW_MS` | 1000 | Settled window used when a note is missed |
| `CONTOUR_HOP_MS` | 20 | Stored contour resolution |
| `TOL_START` / `TOL_MIN` / `TOL_MAX` | 40 / 10 / 60 cents | Adaptive tolerance bounds |
| `TOL_TIGHTEN` / `TOL_LOOSEN` | 0.85 / 1.15 | Staircase factors, result rounded |
| `TONE_GAIN` / `TONE_MS` | 0.16 / 950 ms | Reference tone: triangle, 30 ms attack, 150 ms release |
| `DOUBLE_GAIN` | 0.045 | Guided doubling tone: triangle, 200 ms fade in, 150 ms fade out |

### Exercises

Target frequency is `C4_HZ * 2^(semitone / 12)`. Note names use flats: C, C#, D, Eb, E, F, F#, G, Ab, A, Bb, B, with octave `4 + semitone / 12`.

| `ExerciseId` | Display name | Semitones from C4 |
| --- | --- | --- |
| `major` | Major scale | 0 2 4 5 7 9 11 12 |
| `minor` | Natural minor scale | 0 2 3 5 7 8 10 12 |
| `pentaMaj` | Major pentatonic | 0 2 4 7 9 12 |
| `pentaMin` | Minor pentatonic | 0 3 5 7 10 12 |
| `arpMaj` | Major arpeggio, up and down | 0 4 7 12 7 4 0 |
| `arpMin` | Minor arpeggio, up and down | 0 3 7 12 7 3 0 |
| `arpDom7` | Dominant seventh arpeggio, up and down | 0 4 7 10 12 10 7 4 |

No exercise may exceed 8 notes: the diorama corridor is sized for 8 blocks.

### YIN pitch detection (`yin.rs`)

Input: the latest `ANALYSIS_BUFFER` samples as `f32`, and the sample rate `sr`. Output: `Option<(f0_hz, confidence)>`.

1. Compute RMS of the buffer. If RMS < `RMS_GATE`, return `None`.
2. `max_tau = min(1023, floor(sr / F_MIN_HZ))`, `min_tau = floor(sr / F_MAX_HZ)`.
3. For `tau` in `1..=max_tau`: `d[tau] = sum over j in 0..YIN_WINDOW of (x[j] - x[j + tau])^2`.
4. Cumulative mean normalised difference: `d'[0] = 1`; keep a running sum `s += d[tau]`; `d'[tau] = d[tau] * tau / s` (or 1 if `s == 0`).
5. Find the first `tau >= min_tau` with `d'[tau] < YIN_THRESHOLD`, then walk forward while `d'[tau + 1] < d'[tau]`. If none, return `None`.
6. Parabolic interpolation on `d'[tau - 1], d'[tau], d'[tau + 1]`: `shift = 0.5 * (a - c) / (a - 2b + c)` when the denominator is not 0.
7. Return `f0 = sr / (tau + shift)` and `confidence = 1 - d'[tau]`.

Preallocate `d` and `d'` once; the detector must not allocate per call.

### Cents and smoothing (`cents.rs`, `smoothing.rs`)

- `cents = 1200 * log2(f0 / target_hz)`, then fold into one octave: `((cents + 600).rem_euclid(1200)) - 600`. Folding lets any voice range sing any exercise.
- Keep the last `MEDIAN_FRAMES` voiced cents values; the frame's cents is their median. Clear the history on an unvoiced frame.

### Scoring one note (`scoring.rs`)

State per note: `samples: Vec<(t_ms, cents)>`, `first_voiced: Option<u32>`, `hold_start: Option<u32>`, `frames_total`, `frames_voiced`.

1. On each frame: if voiced, push the sample, set `first_voiced` if empty, then check `|cents| <= tolerance`. Inside: set `hold_start` if empty; if `t - hold_start >= HOLD_MS` the note passes. Outside or unvoiced: clear `hold_start`.
2. If `t >= NOTE_MAX_MS` without a pass, the note is missed.
3. Settled cents: median of samples with `t >= hold_start` on a pass, or with `t >= t_end - MISS_SETTLE_WINDOW_MS` on a miss. None if no samples.
4. Onset cents: median of samples with `t <= first_voiced + ONSET_WINDOW_MS`.
5. Settle time: `hold_start - first_voiced` on a pass, else None.
6. Stability: standard deviation of the settled window samples.
7. Voiced ratio: `frames_voiced / frames_total`.
8. Contour: resample samples into 20 ms bins (median per bin, `None` for empty bins), round to `i16`.

### Adaptive tolerance (`staircase.rs`)

On a pass: `streak += 1`; if `streak >= 2`, `tol = max(TOL_MIN, round(tol * TOL_TIGHTEN))` and `streak = 0`. On a miss: `streak = 0`, `tol = min(TOL_MAX, round(tol * TOL_LOOSEN))`. Tolerance persists across exercises and sessions (store the latest value in the session document).

### Session state machine (`session.rs`)

1. `start_session` -> Listen (step 0, play the reference tone for `TONE_MS`).
2. Listen ends after `LISTEN_MS` -> Sing. In Guided mode start the doubling tone.
3. Sing ends on pass or timeout -> Result (`RESULT_MS`); stop the doubling tone, emit the result and the tolerance.
4. Result ends -> Travel (`TRAVEL_MS`) to the next step, or Summary after the last step.
5. Travel ends -> Listen on the next step.
6. `set_exercise` at any time: rebuild the targets, clear results, -> Travel (`RESTART_TRAVEL_MS`) to step 0. From Idle it only rebuilds the targets.
7. `set_feedback_mode` never changes the phase; during Sing it only starts or stops the doubling tone.

The machine is a pure struct with `fn tick(&mut self, now_ms) -> Vec<Effect>` and `fn on_frame(&mut self, frame) -> Vec<Effect>`. Effects are values such as `PlayTone(freq)`, `StartDouble(freq)`, `StopDouble`, `Emit(Event)`, `Persist(AttemptResult)`. The runner executes effects; the machine never touches audio, time or storage directly.

### Demo singer (`InputSource::Demo`)

For development without a voice, port the sample's demo singer into `pitch-core`: random start error of 45 to 135 cents with random sign, exponential convergence to a bias of up to ±(34 - 3 × passes, minimum 8) cents at a rate of 1.1 to 2.7 per second, 5.4 Hz vibrato of 9 cents fading in over 1.2 s, ±2 cents noise, and 120 to 420 ms of silence before onset. It produces cents directly, skipping detection.

## Visual specification

The production UI must look and behave like the sample page. Port its code module by module into `ui/src/scene/` and `ui/src/hud/`; do not redesign anything. Open the sample's source (view source in a browser, or ask David for the HTML file) and keep it beside you while porting.

### Palette and type

| Token | Hex | Use |
| --- | --- | --- |
| `--ground` | `#efead7` | Island, HUD text background areas, research page background |
| `--water` | `#93c4ac` | Clear colour and page background (jade) |
| `--shade` | `#d2d0c4` | Shaded faces |
| `--roof` | `#fcfcf2` | Upward faces, stepping stones, text on red |
| `--accent` | `#ef2e2d` | Gates, red dot, primary button outline, current step dot; nothing else |
| `--ink` | `#302d26` | Text, mouth, voice trail, fern stems |

Derived: accent lit `#ef2e2d`, base `#d62625`, shaded `#9b1a1a`; mouth ink top `#45413a`, shade `#1f1d18`; projected shadows `#6a6658` at 22% opacity. Typeface: Archivo variable from Google Fonts (width 62 to 125, weight 300 to 900); condensed 62% at weight 800 to 850 for note names and titles, width 100% for body text. Fallback stack: Helvetica Neue, Helvetica, Nimbus Sans, sans-serif.

### Scene (Three.js r128 API, as in the sample)

- **Renderer:** antialias on, stencil on, `outputEncoding = LinearEncoding`, no tone mapping, pixel ratio capped at 2.
- **Camera:** perspective, FOV 32°, pitch 66° above the horizon, distance 46 during exercises and 78 on intro and summary (×1.55 in portrait), lazy follow with a frame-rate-independent lerp of 0.06, yaw sway of ±0.18 rad at 0.11 rad/s, disabled when reduced motion is on. Use `setViewOffset` to push the diorama right of the type column (−15% of width) on wide screens, and down 12% on narrow screens.
- **Light:** one direction, `normalize(-0.55, 0.62, 0.35)`.
- **Toon material:** the sample's `toon(base, top, shade, ao)` ShaderMaterial: two light bands, roof mask, wall foot darkening, distance fade toward ground (35% max, 60 to 120 units). Port the GLSL verbatim.
- **Shadows:** planar projection onto y = 0.02 along the light direction, `MeshBasicMaterial` at 22%, stencil-clipped to the island so shadows never fall on water and never darken where they overlap. The island writes stencil 1; shadows test equal to 1 and increment. Shadow meshes use `matrixAutoUpdate = false` and set `matrix`, not `matrixWorld`.
- **Island:** seeded (`mulberry32(20261004)`) organic outline from an ellipse (centre x 21, z −1.5, radii 39 × 15.5) with harmonics 3, 5 and 11, extruded 2.4 deep, a smaller 0.84 tier underneath, a pond hole at (28, −9.5), three islets.
- **Garden:** about 64 generated ferns placed by rejection sampling with a grove field, kept out of the staircase corridor (x −5 to 47, z −4.6 to 5.4) and thinned behind it; five tree ferns; rocks around the pond and shore; stepping stones at z ≈ 4.6. Ferns are merged into one mesh with ink stem lines; wind sway runs in the vertex shader and the fern shadows sway with them.
- **Staircase:** one 4.6-unit block per note at x = 6 × index, height `0.9 + 0.34 × semitone`. Rebuild blocks, gates and their shadows when the exercise changes.
- **Gates:** two posts and two crossbars per block in accent colour; crossbars at hover height ± tolerance × 0.06 units per cent; tolerance animates with a lerp of 0.06; passed gates sink into their block.
- **Mouth:** extruded low-poly lips in mouth ink, teeth bar in roof colour, dark cavity scaled by openness; hover 5 units above the block at 0 cents, offset 0.06 units per cent clamped to \[−4.3, +7.5\]; roll leads the vertical speed; a plumb line drops to the block top during Sing.
- **Voice trail:** three parallel ink lines (opacity 0.85) recording the mouth position while voiced; one set per note; the previous run stays at 0.22 opacity; all trails are cleared when the exercise changes.
- **Vignette:** CSS radial gradient overlay, transparent to 60%, then up to `rgba(40,35,20,.16)`.

### HUD

A 12-column Swiss grid (24 px gutter, 40 px side margins; 4 columns below 860 px wide). Everything is flush left.

- **Top bar** under a 2 px ink rule: brand with red dot and the "What research says" link (columns 1 to 3), the exercise dropdown (4 to 6), voice input text toggles (7 to 9), feedback mode text toggles (10 to 12). Toggles are text buttons with a 2 px underline when pressed, at least 44 px tall.
- **Target panel** (columns 1 to 4): giant condensed note name, "Note x of n, f Hz", cents readout with sign and ¢, direction line ("↑ Sing higher", "↓ Sing lower", "Hold it"). Trace mode hides the numbers; Unaided hides the numbers and the mouth during Sing.
- **Result log** (columns 10 to 12): the last six notes with settled error and time to settle.
- **Bottom strip** under a 1 px rule: one cell per note (number, name, result), the gate tolerance with its bar, the median settled error.
- **Posters:** intro ("Sing the scale", lede, Begin button with red outline, keys and setup notes) and summary (gates passed, median error, tolerance change, "Sing it again").
- **Keys:** Space begins, replays the tone or sings again; 1, 2, 3 switch feedback; G toggles the grid overlay; R opens the research page; Escape closes it. Keys are ignored while focus is on a select.

### Research page

Port the sample's research overlay as `ResearchPage.svelte` with four findings, each a text column (columns 1 to 5) and an animated SVG figure (columns 7 to 12) that plays when scrolled into view and resolves to its final state with reduced motion:

1. Tone deafness is rare: 200 dots, 3 highlighted; Peretz and Vuvan 2017.
2. Seeing your pitch helps: three training groups, pretest to posttest, only visual feedback significant; Berglin et al. 2022; labelled as schematic, showing significance and not effect size.
3. Practise across a wide range: octave and fifth ladders with hopping targets; Musicae Scientiae vol. 29.
4. The evidence is short: single-session studies versus a six-month plan; fading feedback is labelled as a design choice.

Keep the source list with links exactly as in the sample. Never add a claim, number or citation that is not already there; ask David if a new finding is needed. The exercise pauses while the page is open.

## Implementation guidelines

Use the latest stable release of each dependency below at project start, lock it (`Cargo.lock`, `package-lock.json`), and record the exact versions in `DEPENDENCIES.md`. The one exception is Three.js, pinned to `0.128.0` because the sample's code uses r128 APIs such as `outputEncoding`.

### Dependencies allowed

| Where | Packages |
| --- | --- |
| `pitch-core` | `serde`, `serde_json`, `ts-rs`, `thiserror`, `hound` (WAV, CLI and tests only), `rand` (demo singer, seeded) |
| `src-tauri` | `tauri` 2, `cpal`, `rtrb` (lock-free ring buffer), `midir`, `mongodb`, `tokio` (through Tauri's runtime), `uuid` with `v7`, `chrono` with `serde`, `keyring`, `anyhow`, `log` |
| `ui` | `svelte` 5, `vite`, `@sveltejs/vite-plugin-svelte`, `typescript`, `@tauri-apps/api` 2, `@tauri-apps/cli` 2, `three@0.128.0`, `vitest` |
| `tools` | Python 3 with `numpy`, `soundfile`, `librosa` |

### Threads and data flow

1. **Audio input callback** (cpal thread): convert to mono `f32`, push into an `rtrb` producer. Nothing else.
2. **Analysis thread** (`engine.rs`): every 10 ms of new samples, copy the last 2048 samples into a preallocated buffer, run YIN, fold to cents against the current target, apply the median filter, send a `PitchFrame` over a `std::sync::mpsc` channel to the runner. The current target frequency is shared through an `AtomicU32` holding `f32` bits.
3. **Runner thread** (`runner.rs`): owns the `SessionMachine`, calls `tick` every 10 ms and `on_frame` for each frame, and executes effects: emit Tauri events, send tone commands to the output, send `Persist` to the store task.
4. **Audio output callback**: reads tone commands from a second `rtrb` queue and synthesises triangle tones with linear envelopes. No allocation.
5. **Store task** (Tokio): appends each record to the outbox first, then flushes to MongoDB in the background every 5 s and on session end. Network failure only delays the flush.

### Coding standards

- **Rust:** edition 2021, `cargo fmt`, `cargo clippy -- -D warnings` clean. No `unwrap()` or `expect()` outside tests and `main` setup. `thiserror` errors in `pitch-core`, `anyhow` in the app crate.
- **TypeScript:** `strict: true`, no `any`, no non-null assertions. Svelte 5 runes only (`$state`, `$derived`, `$effect`, `$props`); no legacy stores. Components display state and send commands; no scoring logic in the UI.
- **Scene code:** plain TypeScript classes, created once and owned by `Scene.ts`. Dispose geometries and remove shadow casters when rebuilding the staircase. One `requestAnimationFrame` loop.
- **Accessibility:** real `<button>`, `<select>`, `<label>`; visible focus outlines; touch targets of at least 44 px; `aria-live="polite"` on the result log; every animation respects `prefers-reduced-motion`.
- **Copy:** sentence case, plain verbs, no exclamation marks. Errors say what happened and what to do, for example "Microphone not found. Choose an input device in the top bar."
- **Logging:** never in audio callbacks. Phase changes at `info` level in debug builds only.

### Lessons already learned in the sample (do not repeat these bugs)

- The first `requestAnimationFrame` timestamp can be earlier than `performance.now()`. Clamp the frame delta to `[0, 0.05]` seconds, and wrap any array index derived from time with a positive modulo.
- An exception inside the render loop stops the loop and leaves a black canvas. Keep the loop body free of code that can throw; validate data before it reaches the loop.
- Projected shadow meshes must set `matrix` with `matrixAutoUpdate = false`; setting `matrixWorld` is overwritten every frame.
- The island must render before shadows (`renderOrder = -1`) so its stencil value exists when shadows test against it.
- Global key handlers must ignore events whose target is a `<select>`, or Space opens the session instead of the dropdown.

### Performance budgets

| Measure | Budget |
| --- | --- |
| YIN analysis per hop | under 2 ms on the target machine |
| Input buffer | 128 to 256 frames at 48 kHz |
| Pitch event payload | under 120 bytes of JSON |
| UI frame rate | 60 fps on an integrated GPU at 1440 px wide |
| Fern geometry | under 120,000 vertices in total |

## Step-by-step coding work

Twelve milestones, done strictly in order. The riskiest part, pitch detection on a real voice, is proven offline in M1 and M2 before any audio device or UI exists. Each task ends with a check; run it and paste the output in your report.

### M0. Scaffold

1. **M0.1 Workspace.** Create the folders from the repository layout, a root `Cargo.toml` workspace with `crates/pitch-core` and `src-tauri`, and `.gitignore` (target, node\_modules, `fixtures/audio/*.wav` larger than 1 MB, `.env`). Check: `cargo build` succeeds.
2. **M0.2 Tauri + Svelte shell.** Create the Tauri 2 app in `src-tauri` with the frontend in `ui/` (Svelte 5, TypeScript, Vite). The window shows "Pitch Study" on a `#93c4ac` background. Check: `npm run tauri dev` opens the window.
3. **M0.3 Contracts.** Write `contracts.rs` exactly as in the data contracts section, with `#[ts(export, export_to = "../../ui/src/lib/generated/")]` on each type. Check: `cargo test -p pitch-core` generates the `.ts` files and `npx tsc --noEmit` passes in `ui/`.
4. **M0.4 Quality scripts.** Add `scripts/check.sh` (and `check.ps1`) running `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test`, `npx tsc --noEmit`, `npx vitest run`. Check: the script exits 0.

### M1. Core logic in `pitch-core` (no audio, no UI)

1. **M1.1 Constants and exercises.** `constants.rs` and `exercise.rs` with `targets(ExerciseId) -> Vec<Target>`. Check: unit tests assert that major gives 8 targets named C4 to C5 and that `arpDom7` has 8 targets ending on E4; `freq_hz` of A4 in the major scale is 440.0 ± 0.01.
2. **M1.2 Cents.** `cents(f0, target)` and `fold(c)`. Check: tests for 440 vs 440 = 0, 880 vs 440 folds to 0, 466.16 vs 440 ≈ +100, 415.30 vs 440 ≈ −100 (±0.5 cents).
3. **M1.3 YIN.** `Yin::new(sample_rate)` preallocates; `detect(&self, &[f32]) -> Option<(f32, f32)>`. Check: synthetic sine and sawtooth tests at 82.41, 130.81, 261.63, 440, 659.26 and 987.77 Hz, at 44.1 and 48 kHz, each within ±3 cents; white noise at RMS 0.005 returns `None`.
4. **M1.4 Smoothing.** 5-frame median with reset on unvoiced. Check: a sequence with one outlier of +300 cents among values near +10 yields no output above +15.
5. **M1.5 Scoring.** `NoteScorer` per the scoring algorithm. Check: tests for a pass after 700 ms in tolerance, a reset when the voice leaves tolerance at 650 ms, a miss at 5500 ms, the settled median on pass and miss, and the contour length (`ceil(duration / 20 ms)`).
6. **M1.6 Staircase.** Check: from 40, pass-pass gives 34; then a miss gives 39; tolerance never goes below 10 or above 60 over 1,000 random results.
7. **M1.7 Session machine.** `SessionMachine` with `tick` and `on_frame` returning effects. Check: a test drives a full major-scale session with a scripted perfect singer and asserts the exact sequence of phases and that 8 results and 1 summary are emitted; a second test calls `set_exercise(PentaMin)` mid-Sing and asserts Travel to step 0 with 6 targets.
8. **M1.8 Demo singer.** Seeded, as specified. Check: with seed 1, a full session completes and at least half the notes pass at tolerance 40.

### M2. Offline validation of the detector

1. **M2.1 Fixture generator.** `cargo run -p pitch-core --bin analyze -- gen fixtures/audio/synthetic/` writes WAVs: steady tones, slow glides of one octave, 5.5 Hz vibrato of ±30 cents, and tones with added noise at 20 dB SNR. Check: files exist and play.
2. **M2.2 Analyze CLI.** `analyze run <wav> --target-hz 261.63` writes a CSV `t_ms,f0_hz,confidence,cents` using the same YIN, fold and median code as the app. Check: on the steady 261.63 Hz file, median absolute cents is under 1.
3. **M2.3 Python comparison.** `tools/validate_detector.py <wav> <csv>` runs librosa pYIN on the same file, aligns frames by time, and prints median absolute difference, gross error rate (frames differing by more than 50 cents) and voicing agreement. Check: runs on every synthetic file.
4. **M2.4 Real voice.** Ask David for 5 to 10 short recordings of his voice through the interface (sustained notes and slow scales). Run the comparison. Acceptance targets (proposed, to confirm with David): median absolute difference under 10 cents, gross error rate under 3%, voicing agreement over 90%. If a target is missed, report the numbers and stop; do not tune constants without approval.

### M3. Audio input and output

1. **M3.1 Devices.** `list_devices` returns input and output device names from cpal's default host. On Windows, note in `DEPENDENCIES.md` whether ASIO is needed for David's interface. Check: the command returns the interface's name.
2. **M3.2 Capture.** Open the chosen input at 48 kHz (or the device default) with a 256-frame buffer, downmix to mono, push into an `rtrb` ring of 1 second. Check: a debug command reports a live RMS that rises when David speaks.
3. **M3.3 Output synth.** Output stream reading `ToneCommand { Play { freq, gain, ms }, StartDouble { freq }, StopDouble }` from a second ring; triangle oscillator with the envelopes in the constants table. Check: `replay_tone` plays a clean C4 with no clicks at start or end.
4. **M3.4 Error handling.** Stream errors and device removal emit `audio://status` with a plain-language `error`. Check: unplugging the interface shows the error and does not crash.

### M4. Engine, runner and IPC

1. **M4.1 Analysis thread.** Pulls from the ring every 10 ms of new samples and produces frames as specified. Check: a debug page in the UI prints frames while David holds a note, cents stable within ±10.
2. **M4.2 Runner.** Owns the `SessionMachine`, ticks every 10 ms, executes effects. Check: the debug page shows phase events in the right order for a full session using `InputSource::Demo`.
3. **M4.3 Commands.** Implement every command in the contracts table. Check: each command is called from the debug page and produces the expected events.
4. **M4.4 TypeScript IPC layer.** `ipc.ts` wraps `invoke` and `listen` with the generated types and nothing else. Check: `tsc` passes with no `any`.

### M5. Scene port (driven by fake data first)

Port from the sample into one file per concern. Until M7, drive the scene from a fake event source in `ui/src/scene/fakeSession.ts` that replays the sample's demo singer.

1. **M5.1 Palette, toon material, renderer, camera.** Check: an empty island-coloured slab renders with correct colours (sample a screenshot: ground `#efead7`, water `#93c4ac`).
2. **M5.2 Shadows.** Projection, stencil clipping, no double darkening. Check: two overlapping block shadows have one uniform tone; no shadow appears over water.
3. **M5.3 Island and garden.** Seeded outline, pond, islets, ferns with sway and swaying shadows, tree ferns, rocks, stepping stones. Check: side by side with the sample at the same window size, the layout matches (same seed).
4. **M5.4 Staircase and gates.** Built from a `Target[]`, rebuildable, gates following an animated tolerance and sinking on pass. Check: switching the fake exercise from major to `arpMaj` rebuilds a pyramid with no leftover blocks, gates or shadows.
5. **M5.5 Mouth, plumb line, trail.** Check: the fake singer's contour appears as three thin lines; Unaided hides the mouth during Sing and reveals the trail at Result.
6. **M5.6 Loop safety.** Delta clamp, positive modulo, try-free loop body. Check: the scene runs for 10 minutes with no console errors and steady memory in the browser's performance panel.

### M6. HUD port

1. **M6.1 Tokens and grid.** `tokens.css`, `grid.css`, Archivo font, vignette, G grid overlay. Check: the 12-column grid matches the sample at 1440 px and switches to 4 columns below 860 px.
2. **M6.2 Top bar.** Brand, research link, exercise select, input toggles, feedback toggles. Check: keyboard-only navigation reaches every control with a visible focus ring.
3. **M6.3 Target panel, result log, bottom strip, tolerance, median.** Check: with the fake session, every value updates as in the sample, including "no voice" and missed notes.
4. **M6.4 Posters and keys.** Intro and summary posters with their transitions; Space, 1 to 3, G, R, Escape. Check: keys do nothing while the select is focused.

### M7. Wire the real session

1. **M7.1 State.** `state.svelte.ts` holds targets, phase, current frame, results, tolerance and summary from Rust events. Replace the fake source. Check: a full session with the Demo input looks identical to the fake one.
2. **M7.2 Microphone.** Switch to Mic. Check: David sings a major scale and the mouth tracks his voice; passing and missing notes behave as specified.
3. **M7.3 Exercise and mode changes.** Check: changing the exercise mid-session restarts on step 0; changing the feedback mode during Sing starts or stops the doubling tone without changing the phase.

### M8. Persistence

1. **M8.1 Outbox.** Append-only JSONL with fsync after each line; atomic rewrite after a flush. Check: kill the app mid-session, restart, and all completed attempts are still in the outbox.
2. **M8.2 MongoDB.** Connection string from the `PITCH_STUDY_MONGODB_URI` variable or the OS keychain entry `pitch-study/mongodb-uri`; idempotent upserts by `_id`; create the indexes at startup. Check: with the network off, practice works and the outbox grows; with the network back, documents appear in MongoDB and the outbox empties.
3. **M8.3 Tolerance persistence.** The latest tolerance is read from the newest session at startup. Check: tolerance after a restart equals the last value shown.
4. **M8.4 Progress query.** `get_progress(days)` aggregates Unaided attempts by day: median absolute settled cents and count. Check: an integration test against a local MongoDB with seeded documents returns the expected medians.

### M9. Progress view

1. **M9.1 Progress page.** A Swiss-style page reached from the summary poster: a line of daily median Unaided error with the count per day, drawn as inline SVG in the palette, plus the current tolerance. No streaks or badges. Check: with seeded data the chart matches the query output.

### M10. Research page

1. **M10.1 Port.** Port the research overlay and its four animated figures as specified. Check: figures play on scroll, show their final state with reduced motion on, and the sources list matches the sample word for word.

### M11. Packaging and hardening

1. **M11.1 Permissions.** macOS `NSMicrophoneUsageDescription` in `Info.plist`; Windows and Linux microphone access verified. Check: a fresh install prompts for the microphone once.
2. **M11.2 Error states.** No device, device removed, MongoDB unreachable, permission denied: each shows a plain message and a next step. Check: each case reproduced manually.
3. **M11.3 Release build.** `npm run tauri build` for David's OS. Check: the installed app runs a full session with the microphone and writes to MongoDB.

## Testing and validation

Most correctness lives in `pitch-core`, so most tests are fast Rust unit tests that need no device, network or window. Device, UI and MongoDB behaviour is checked by short manual scripts that David runs once per milestone.

| Layer | How it is tested | Runs |
| --- | --- | --- |
| `pitch-core` logic | Unit tests with synthetic signals and scripted singers | Every commit, `scripts/check.sh` |
| Detector accuracy | `analyze` CLI + `tools/validate_detector.py` against pYIN | M2, and again after any detector change |
| Session flow | Rust tests driving `SessionMachine` through full sessions | Every commit |
| TypeScript | `tsc --noEmit` on generated contracts, `vitest` for state reducers | Every commit |
| Scene and HUD | Side-by-side comparison with the sample at 1440 × 900 and 390 × 844 | End of M5 and M6 |
| Audio devices | Manual: list, capture RMS, tone, unplug | End of M3 |
| Persistence | Manual offline/online test plus a MongoDB integration test | End of M8 |

### Manual session script (David, end of M7 and M11)

1. Start the app with the interface connected and direct monitoring on.
2. Sing the major scale in Guided mode; confirm the mouth tracks your voice and gates sink on passes.
3. Switch to Minor pentatonic mid-note; confirm the diorama rebuilds and the run restarts.
4. Sing one run in Unaided mode; confirm the mouth hides while singing and results appear afterwards.
5. Open the research page with R; confirm the exercise pauses and resumes.
6. Quit, restart, and confirm tolerance and history are kept.

### Measuring whether the app works for David

The product's own success test is David's Unaided median settled error over weeks. Before the first real session, record a baseline of 20 Unaided notes per exercise; the progress page then compares against it. This uses data the app already stores, so it needs no extra code.

## Out of scope, guardrails and open questions

Version 1 is a desktop practice tool for one singer; anything below stays out unless David asks for it.

### Out of scope for version 1

- Mobile apps, web deployment, accounts or multi-user data.
- Storing or uploading audio recordings.
- Keys other than C4 as root, exercises longer than 8 notes, melodies or chord progressions with backing.
- Spaced-repetition scheduling of weak notes (the data model already supports it later through `target.semitone` per attempt).
- A VST3 or CLAP plugin build of the core.
- Gamification: points, streaks, badges, confetti.
- Any change to the visual style, palette or research content.

### Guardrails

- Never mix the reference tone into the input path; the singer's own voice is heard only through hardware direct monitoring.
- Never block the audio callbacks or the render loop on storage or network.
- Never show a score computed in the UI; the UI only displays what Rust emits.
- Never fabricate research claims, numbers or citations on the research page.

### Open questions for David

- [ ] Which OS and audio interface will the app run on? This decides whether ASIO setup is needed on Windows.
- [ ] MongoDB Atlas or a local MongoDB? If Atlas, which cluster and database user should the app use?
- [ ] Confirm the detector acceptance targets in M2.4 (median under 10 cents, gross errors under 3%, voicing agreement over 90%).
- [ ] Should exercises follow a fixed weekly plan, or stay free choice from the dropdown?
