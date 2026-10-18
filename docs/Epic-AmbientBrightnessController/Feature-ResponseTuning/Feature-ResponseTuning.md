# F4 — Response Tuning

| Field | Value |
|---|---|
| **Epic** | [EPIC: Ambient Brightness Controller](../Epic-AmbientBrightnessController.md) |
| **Depends on** | F1 (Ambient Light Sensing), F2 (Automatic Brightness Control) |
| **Depended on by** | F7 (Status Dashboard) |

## Overview

This feature controls the *timing* of brightness changes: how fast the screen follows a change in
ambient light, how large the individual steps are, and how brief light changes are filtered out. It is
what separates a brightness controller that feels snappy and calm from one that feels sluggish or
twitchy.

## User Story 1 — Choose how fast the screen reacts

As a user, I want to control how quickly the brightness follows changes in ambient light, so that the
screen feels neither sluggish nor jittery.

#### Acceptance Criteria

- Given the response speed is set towards fast, when the ambient light changes significantly, then the
  brightness reaches the new target within the short reaction window used for the fast setting.
- Given the response speed is set towards slow, when the ambient light changes significantly, then the
  brightness approaches the new target gradually over the long reaction window used for the slow
  setting.
- Given any response speed setting, when the ambient light is steady, then the brightness settles and
  does not oscillate around the target.
- Given the response speed has been changed, when the controller is already running, then the new
  setting takes effect for the next adjustment without requiring a restart.

## User Story 2 — No visible stepping

As a user, I want large brightness changes to be smooth, so that I do not see the screen jump or flash
when I walk from a dark room into daylight.

#### Acceptance Criteria

- Given a large difference between the current and the newly computed brightness, when the change is
  applied, then it is applied as a ramp through intermediate values rather than as a single jump.
- Given the ramp is in progress, when no individual step exceeds the configured maximum step size,
  then the transition is not perceived as a flash.
- Given the target value changes again while a ramp is in progress, when the new target is received,
  then the ramp retargets from the current value and does not overshoot the new target.

## User Story 3 — Filtering out brief light changes

As a user, I want brief changes in ambient light to be ignored, so that passing shadows, a hand
movement, or a quick cloud do not make my brightness wobble.

#### Acceptance Criteria

- Given a change in ambient light lasts less than the debounce window, when the light returns to its
  previous level, then no brightness change is applied at all.
- Given a change in ambient light is sustained for longer than the debounce window, when the window
  elapses, then the adjustment begins.
- Given a sequence of small, alternating readings around a stable average, when they are processed,
  then they are treated as no change rather than as repeated small adjustments.

## User Story 4 — The setting survives restarts

As a user, I want my response speed to persist, so that I do not have to reconfigure the feel of the
app every time it restarts.

#### Acceptance Criteria

- Given I have changed the response speed, when the app process is restarted or the device is
  rebooted, then the setting is still in effect when the controller resumes.
- Given I have never changed the response speed, when the controller runs for the first time, then a
  sensible middle-of-the-range default is used.

## Out of Scope

- The manual offset on the brightness value (that is F3).
- The shape of the lux-to-brightness curve (that is F2).
- Per-app or per-scenario response profiles.
- Adaptive/learned response behaviour based on user behaviour history.

## Resolved Decisions

- A **single** "response speed" control is used, with a sensible middle-of-the-range default; there are
  no separate controls for reaction window and smoothing.
- The control lives on the app's dedicated settings screen, together with the offset.
- The concrete default value and the fast/slow window durations are fixed in the technical design.

## Technical Design

### Architecture & Components

Same three modules as F1/F2 (`:core` pure Kotlin, `:infrastructure` Android adapters, `:app`
composition root; package root `com.ambientbrightness`). F4 consumes F1 only via the injected
`ReadingStabiliser` (F1's designated extension point) and F2 only via `BrightnessTransform` — no new
pipeline, no F1/F2 signature change. Components to add (all `:core` unless noted):

- `SettingsStore` (interface) + `SettingsKey<T>`/`SettingsKeys` (value objects) — typed, reactive,
  persistent key/value store; the **shared** store F3 (offset) and F7 adopt. `SettingsKeys` holds
  `RESPONSE_SPEED_PERCENT` and is where F3 adds its offset key. `:core` holds the contract only; both
  implementations live in `:infrastructure` — `InMemorySettingsStore` (Dummy) and
  `AndroidDataStoreSettingsStore` (persistent).
- `ResponseSpeed` (value object) — `percent: Int` in `0..100`; `0` = fastest, `100` = slowest,
  `DEFAULT = 50`, out-of-range input clamped.
- `ResponseProfile` (value object) + `ResponseProfileResolver` — derived timings
  (`reactionWindowMillis`, `debounceWindowMillis`, `maxStepFraction`) and the pure
  `ResponseSpeed → ResponseProfile` mapping.
- `ResponseTuningService` (boundary) — the reactive surface F7 observes and writes;
  `responseSpeed: StateFlow<ResponseSpeed>` + `suspend fun setResponseSpeed(speed)`.
- `UserConfigurableStabiliser` (implements F1 `ReadingStabiliser`) — **replaces** F1's
  `OutlierClampingStabiliser` as the injected instance by *decorating* it (composition, not a copy).
- `ResponsiveBrightnessRamp` (implements F2 `BrightnessTransform`) — the ramp of applied brightness;
  sole owner of step size and retargeting.

Fixed constants (owned here): `ResponseSpeed.DEFAULT = 50` (middle of the range);
`reactionWindowMillis` linear from `300 ms` (fast) to `3000 ms` (slow), default `1650 ms`;
`debounceWindowMillis` from `150 ms` to `600 ms` off the same control, default `375 ms`;
`maxStepFraction = 0.15` of the brightness span (anti-flash cap per invocation).

Split of responsibility (explicit): F1's `OutlierClampingStabiliser` keeps its fixed, per-sample, stateless duties (range
clamp, single-outlier rejection, `0 lux` validity) and is **not** made configurable; F4 adds time-domain behaviour on top
(debounce, slew smoothing). Outlier rejection is instantaneous, response tuning is time-based — they never overlap.

### Data Flow

1. `AmbientLightUseCase` (F1) throttles a raw event, then calls the injected
   `UserConfigurableStabiliser.stabilise(sample)`, which delegates to F1's `OutlierClampingStabiliser`
   first (unchanged), then applies the debounce gate (hold a change until sustained for
   `debounceWindowMillis`) plus slew/EMA smoothing from the *current* `ResponseProfile`.
2. F2 `BrightnessControllerUseCase`: `DefaultBrightnessMappingCurve` → `BrightnessTransform` hooks.
3. `ResponsiveBrightnessRamp.apply(fraction, context)` steps from the currently *applied* fraction
   toward the new target: `allowedStep = clamp(rate × elapsedMillis, 1 span-unit, maxStepFraction ×
   span)` with `rate = span / reactionWindowMillis`, `elapsedMillis` measured on the injected `Clock`
   since the previous invocation. One intermediate value per pipeline invocation — no extra ticker.
4. F2 unchanged afterwards: scale by `BrightnessRange`, clamp to `[max(1, min), max]`,
   `HysteresisGate`, `MinimumStepDeadBand`, `SystemBrightnessClient.write`.

Ramp position relative to F2's dead-band and clamping (concrete): the ramp runs **before** F2's clamp and dead-band, so
every intermediate value is independently clamped and still passes the dead-band. The dead-band minimum step must be
**strictly smaller** than one ramp step (configure ≤ 1 device unit) so it cannot swallow ramp progress.

### Key Interfaces & Classes

```kotlin
interface SettingsStore {                              // :core — shared (F3/F4/F7)
    fun <T> observe(key: SettingsKey<T>): Flow<T>
    suspend fun <T> set(key: SettingsKey<T>, value: T)
}
interface ResponseTuningService {                      // :core — F4 boundary observed by F7
    val responseSpeed: StateFlow<ResponseSpeed>
    suspend fun setResponseSpeed(speed: ResponseSpeed)
}
// Reused as-is (not redefined): ReadingStabiliser.stabilise(sample: LuxSample): LuxSample?  (F1)
// and BrightnessTransform.apply(fraction: Float, context: BrightnessContext): Float         (F2)
```

- `UserConfigurableStabiliser` re-reads the `ResponseProfile` from a `StateFlow<ResponseSpeed>` on
  every call, so a changed setting applies to the next adjustment without re-injection or restart
  (AC1.4). `ResponsiveBrightnessRamp` holds state (`currentFraction`, `targetFraction`,
  `lastCallMillis`) and is the **only** component that knows about ramping; F2's dead-band stays
  size-independent of F4. One injected `Clock` (`nowMillis()`) serves both — never
  `System.currentTimeMillis()` directly.

### State & Persistence

Mechanism: **AndroidX DataStore (Preferences)** over a single app-private file. Chosen over `SharedPreferences` (no
reactive `Flow`, main-thread I/O, no transactional writes) and over database/Room (overkill for scalar settings). Being
app-private storage it survives process death and device reboot by construction, satisfying AC4.1 without any receiver.

- One store instance, file `settings_store.preferences_pb`, app-scoped in `:app`; key
  `response_speed_percent: Int`. F3's offset uses the **same** `SettingsStore`/file with its own key —
  F4 defines the component so F3 and F7 can adopt it unchanged.
- `ResponseTuningService` seeds `responseSpeed` from `SettingsStore.observe(RESPONSE_SPEED_PERCENT)`
  (default `50` when absent → AC4.2), persists on `setResponseSpeed`, and exposes
  `responseSpeed: StateFlow<ResponseSpeed>` as F7's reactive state; F3 should mirror this for offset.

### Android Platform Specifics

- DataStore needs a `Context`, so only `:infrastructure` (`AndroidDataStoreSettingsStore`) and `:app`
  see it — `:core` stays Android-free. Writes are `suspend` on `Dispatchers.IO`; the UI-facing
  `StateFlow` is fed from the store's flow, so F7 never touches I/O.
- No new permission and no `BOOT_COMPLETED` receiver (persistence is storage-based);
  `Settings.System.SCREEN_BRIGHTNESS` writes remain entirely F2's — F4 writes no system setting.

### Edge Cases & Failure Handling

- Corrupt/absent/out-of-range stored value → clamped to `0..100`, falling back to `DEFAULT`; never
  crashes (AC4.2).
- Setting changed mid-ramp → retarget from the current value, new `reactionWindowMillis` effective
  immediately (AC2.3, AC1.4). Target reversal recomputes the step bound and cannot overshoot, because
  the ramp never moves past the target in one step.
- Long gap between invocations (screen off, throttling) → `elapsedMillis` grows but the per-invocation
  step is capped at `maxStepFraction`, so no visible jump on resume. A change shorter than
  `debounceWindowMillis` is suppressed entirely (AC3.1); small alternating readings around a stable
  average count as no change (AC3.3).
- Steady target → smoothing removes residual jitter while F2's `HysteresisGate` still guards curve
  breakpoints; both are idempotent at steady state, so no double damping (AC1.3). DataStore I/O
  failure → keep the in-memory `StateFlow` value, log, surface nothing fatal.

### Test Strategy

All `:core`, pure Kotlin, fake `Clock` + a `:core` test-double settings store (F4's
`InMemorySettingsStore` is a `:infrastructure` implementation, which `:core` cannot depend on), no
Android:
`ResponseProfileResolver` (default, both extremes, monotonicity); `ResponseTuningService` (default
when absent, persistence round-trip, live emission to F7); `UserConfigurableStabiliser` (suppression
below / start above the debounce window, steady-state no-oscillation, live profile change);
`ResponsiveBrightnessRamp` (no single jump, no overshoot on retarget, monotone approach, cap on long
gap). `:infrastructure` gets one instrumented test proving a value written by one
`AndroidDataStoreSettingsStore` is read back by a **fresh** instance (simulated restart, AC4.1);
`:app` gets one virtual-time integration test through fake sample → stabiliser → F2 pipeline → ramp →
fake `SystemBrightnessClient`, asserting intermediate write count and monotonicity.

## Task Breakdown

- [x] [T1 — Shared persistent settings store](./Tasks/Task-01-SharedPersistentSettingsStore.md)
- [x] [T2 — Response speed model, profile mapping, boundary service](./Tasks/Task-02-ResponseSpeedModelProfileMappingBoundaryService.md)
- [x] [T3 — User-configurable stabiliser on F1's extension point](./Tasks/Task-03-UserConfigurableStabiliserOnF1SExtensionPoint.md)
- [x] [T4 — Responsive brightness ramp on F2's transform hook](./Tasks/Task-04-ResponsiveBrightnessRampOnF2STransformHook.md)
- [x] [T5 — Composition, wiring, end-to-end verification](./Tasks/Task-05-CompositionWiringEndToEndVerification.md)

## Suggested Implementation Order

`T1 → T2 → T3 → T4 → T5`. T1 first: the persistence mechanism is shared with F3/F7, so the key
contract must be frozen before anything depends on it. T2 needs only T1's interfaces (its `:core`
tests use a `:core` test-double store, because `:core` cannot depend on `:infrastructure`, where both
store implementations live). T3 and T4 are mutually independent — T3 plugs into F1, T4 into F2 — and can
run in parallel once T2's `ResponseProfile` exists. T5 is last: the only task touching both pipelines.
