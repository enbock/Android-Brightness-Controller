# T5 — Composition, wiring, end-to-end verification

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-ResponseTuning.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T5 — Composition, wiring, end-to-end verification**: wire `SettingsStore`,
  `ResponseTuningService`, `UserConfigurableStabiliser` (into F1) and `ResponsiveBrightnessRamp`
  (into F2) in `:app`; expose `ResponseTuningService` for F7; add the virtual-time integration test.
  *Files: `:app/src/main/kotlin/com/ambientbrightness/app/**`,
  `:app/src/test/kotlin/com/ambientbrightness/app/**`.* *Verify: integration test asserting write
  count, monotone intermediate values and correct use of a persisted speed after a simulated restart
  (AC1.1, AC2.2, AC3.2, AC4.1).*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `b26bf4d0c858dad4c50972fb6c6c250a37da91f1`.*

- The composition root is now the single place that knows F4: `AmbientBrightnessComponent` builds one
  `DefaultResponseTuningService(settingsStore, controllerScope)` and publishes it as `val
  responseTuningService: ResponseTuningService` (F7's boundary), constructs
  `UserConfigurableStabiliser(delegate = OutlierClampingStabiliser(maxRangeLux = sensorRangeLux),
  responseSpeed = responseTuningService.responseSpeed, clock = clock)` for F1's `AmbientLightUseCase`,
  and registers `ResponsiveBrightnessRamp(responseSpeed = …, clock = clock)` as F2's transform hook.
  One `Clock` instance (new constructor parameter, default `SystemClock`) serves stabiliser,
  use case and ramp. All four parts share the one app-scoped store, so a write through the service is
  the value the running pipeline reads on its next invocation — no restart, no second source of truth.
- Files changed: `app/src/main/kotlin/com/ambientbrightness/app/Composition/AmbientBrightnessComponent.kt`
  (T3/T4 wiring, plus the `clock` parameter), `app/build.gradle.kts` (`testImplementation` JUnit 4 +
  `kotlinx-coroutines-test`, so `:app` gains a JVM unit-test source set; it had none before).
  Files added: `app/src/test/kotlin/com/ambientbrightness/app/AmbientBrightnessResponseTuningTest.kt`.
  F2's `MinimumStepDeadBand` needed no change: its default of `DEFAULT_MINIMUM_STEP = 1` device unit
  is already the "≤ 1 device unit" the design requires, and the ramp's step floor sits strictly
  above it.
- The integration test drives the **real** graph — dummy sensor → F1 `AmbientLightUseCase` →
  `UserConfigurableStabiliser` → `DefaultAmbientLightService` → F2 `BrightnessControllerUseCase`
  pipeline and curve → `ResponsiveBrightnessRamp` → a recording `SystemBrightnessClient` — with
  virtual time and the injected `Clock` advanced together by exactly one 500 ms sampling interval per
  tick, so debounce windows and ramp steps are deterministic without a device. Three cases:
  1. *Transient vs sustained (AC3.1, AC3.2, AC2.1, AC2.2):* one tick of bright light followed by the
     previous level writes **nothing** to the system; a reading seen for the first time writes
     nothing until its debounce window has elapsed; a sustained change is then written in ≥ 3 distinct
     monotone steps, none larger than the anti-flash cap (0.15 of the span ≈ 38 of 254 units) and
     converging exactly on the value F2 computes for the settled reading, which the test derives from
     `DefaultBrightnessMappingCurve` + `BrightnessRangeScaler` rather than hardcoding.
  2. *Persisted speed after a simulated restart (AC1.1, AC3.2, AC4.1):* a fresh component over a store
     that already holds `0` reacts earlier than the same graph over `100` — the slow end is still
     holding the change on the tick where the fast end has already written.
  3. *Boundary exposure (F7):* a store holding `80` surfaces `ResponseSpeed.of(80)` through
     `responseTuningService.responseSpeed`.
- Deviation 1: two findings from the integration test were **not** changed in production code but are
  worth recording. (a) The test's first draft used `10 000 lux` as the bright level and stalled one
  device unit short of the target: `10 000` is exactly a hysteresis-gate boundary, and the gate only
  promotes to that region at `11 000`, so a reading of exactly `10 000` is rejected and the pipeline
  is never invoked again. That is F2's documented gate behaviour, not an F4 defect; the test uses
  `5 000 lux`, which sits comfortably inside a region. (b) The first draft asserted an absolute
  reaction tick; the measured index includes the tick on which the new reading first reaches the loop,
  so the test asserts the relation (slow reacts later than fast) plus a bounded budget instead.
- Deviation 2: the design's Test Strategy mentions one `:app` integration test "through fake sample →
  stabiliser → F2 pipeline → ramp → fake `SystemBrightnessClient`". It is implemented against the real
  composition root and the real F1/F2 use cases rather than hand-assembled stages, because that is what
  T5's wiring claim is about; the stages themselves are unit-tested in T3/T4.
- Verification (from the repo root, all with `--no-daemon --console=plain`): `.\gradlew.bat :core:test`
  → BUILD SUCCESSFUL, 24 classes / 136 tests, 0 failures; `.\gradlew.bat :infrastructure:test`
  → BUILD SUCCESSFUL, 8 classes / 32 tests, 0 failures; `.\gradlew.bat :app:testDebugUnitTest` →
  BUILD SUCCESSFUL, 1 class / 3 tests, 0 failures; `.\gradlew.bat :app:assembleDebug`,
  `.\gradlew.bat :app:assembleDebugAndroidTest` and
  `.\gradlew.bat :infrastructure:assembleDebugAndroidTest` → BUILD SUCCESSFUL (the existing
  instrumented tests still compile against the changed component). Not run on a device: the
  `androidTest` suites (including T1's DataStore round-trip) are compiled only — no emulator or device
  is attached in this environment.
- Feature status: T1–T5 are complete, so every user story of F4 has a verified implementation.
  AC1.1/1.3/1.4 are covered by T3's and T5's tests, AC2.1–2.3 by T4's and T5's, AC3.1–3.3 by T3's and
  T5's, and AC4.1/4.2 by T2's and T5's.

### Review corrections (post-implementation)


*Fix commit `585035738bcd324fb0d077c99aa8795a70cf3798` (after T5; addresses T3 and T4).*

- **T3 — the debounce window is armed once, not re-seeded.** The first implementation restarted the
  window whenever two consecutive accepted readings differed by more than the noise band, and that band
  belongs to the level *in effect*. For a large step the band is therefore a tiny fraction of the new
  reading, so ordinary jitter (10 lx on a 20 000 lx reading, or a monotone drift while the light keeps
  moving) re-armed the window on every sample and the adjustment never started — AC3.2 was
  unreachable in exactly the situations F4 exists for. Now the window starts when a reading first
  leaves the band of the level in effect, keeps running while readings stay outside it (the target
  follows the latest accepted reading), and is cleared only when a reading returns inside that band.
  The transient case of AC3.1 is unchanged: a change that goes back to the previous level is still
  dropped entirely. New tests: a jittered sustained change is applied, a monotone moving change is
  applied and followed, and a level that returns inside the band is still dropped.
- **T4 — the ramp re-syncs its baseline with the brightness actually applied.** The first
  implementation consulted `BrightnessContext.currentBrightness` only on its very first invocation
  ever, so after a controller restart (F2 resets its dead-band, the app keeps one ramp) or while
  writes were suspended (`writesSuspended`, which keeps the pipeline and therefore the ramp advancing)
  `currentFraction` was stale and the accumulated difference could be emitted as a single jump. The
  applied value now wins whenever it disagrees with the remembered fraction by more than the step
  about to be taken; normal progress stays far below that bound because F2 only suppresses
  sub-device-unit writes, so the guard never fires mid-ramp. New test: a remembered fraction that
  disagrees with the applied brightness steps down instead of jumping to the target.
- Verification after the corrections (from the repo root, `--no-daemon --console=plain`):
  `.\gradlew.bat :core:test` → BUILD SUCCESSFUL, 24 classes / 140 tests, 0 failures (136 before the
  corrections: +4 tests); `.\gradlew.bat :infrastructure:test` → 8 classes / 32 tests, 0 failures;
  `.\gradlew.bat :app:testDebugUnitTest` → 1 class / 3 tests, 0 failures; `.\gradlew.bat
  :app:assembleDebug`, `.\gradlew.bat :app:assembleDebugAndroidTest` → BUILD SUCCESSFUL.
