# T3 — User-configurable stabiliser on F1's extension point

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-ResponseTuning.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T3 — User-configurable stabiliser on F1's extension point**: implement
  `UserConfigurableStabiliser : ReadingStabiliser`: delegate first to F1's `OutlierClampingStabiliser`
  (unchanged), then apply the debounce gate and time-based smoothing from the live `ResponseProfile`
  via the injected `Clock`; inject it as F1's `AmbientLightUseCase` stabiliser in `:app`. Do not
  modify F1's outlier logic or interfaces. *Files:
  `:core/.../core/UserConfigurableStabiliser/**`, `:app/.../app/**`.* *Verify: fake-clock unit tests
  for suppression below the debounce window (AC3.1), start above it (AC3.2), no reaction to small
  alternating readings (AC3.3), steady-state settling (AC1.3), live profile change without restart
  (AC1.4).*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `587a727573ef1b55a9421e8433ece7c05ad704ab`.*

- Built `UserConfigurableStabiliser` as a pure decorator over the injected `ReadingStabiliser`
  (`OutlierClampingStabiliser` in production, taken as a constructor dependency): it calls the
  delegate first and returns `null` unchanged whenever the delegate rejects a sample, so F1's range
  clamp, single-outlier rejection and `0 lux` validity are untouched, non-configurable and never
  bypassed. F1's outlier logic, its interface and its tests are unmodified.
- Added exactly the two time-domain rules on top. **Debounce gate (AC3.1, AC3.2):** a reading more
  than the noise band away from the level in effect becomes the *candidate* and starts its window at
  the injected `Clock`'s time; only once `now - candidateSinceMillis >= debounceWindowMillis` (the
  window of the live profile) does the value move, and any reading back within the noise band of the
  level in effect clears the candidate — so a transient lights up nothing. **Smoothing (AC1.1,
  AC1.3):** the move is an EMA step of `(target - current) × elapsedMillis / reactionWindowMillis`
  (fraction clamped to `0..1`), so a fast setting reaches the target in one step once the window has
  elapsed (elapsed ≥ window ⇒ fraction 1) while a slow setting approaches it gradually; the value
  snaps exactly onto the target once the remaining distance is below the noise band, which is what
  makes a steady level settle without oscillating.
- Both the profile and the speed are re-read on **every** call (`resolver.resolve(responseSpeed.value)`
  on the injected `StateFlow<ResponseSpeed>`), which is what delivers AC1.4 with no re-injection and no
  restart; the class holds no coroutine scope and starts no collection of its own.
- Noise band: `max(MINIMUM_NOISE_BAND_LUX = 2 lx, NOISE_BAND_FRACTION = 0.05f × level in effect)`,
  owned by this class and the one place "small" is defined. Readings inside it are treated as no
  change at all (AC3.3). Robustness: a non-finite or negative lux from the delegate is dropped
  (`null`) instead of emitted, and the class never throws.
- `:app` wiring: `AmbientBrightnessComponent` now builds `DefaultResponseTuningService(store =
  settingsStore, scope = controllerScope)` and exposes it as `val responseTuningService:
  ResponseTuningService` (F7's boundary, unchanged surface), and passes
  `UserConfigurableStabiliser(delegate = OutlierClampingStabiliser(maxRangeLux = sensorRangeLux),
  responseSpeed = responseTuningService.responseSpeed, clock = clock)` into F1's
  `AmbientLightUseCase`.
- Files added: `core/src/main/kotlin/com/ambientbrightness/core/UserConfigurableStabiliser/UserConfigurableStabiliser.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/UserConfigurableStabiliser/UserConfigurableStabiliserTest.kt`.
  Files changed: `app/src/main/kotlin/com/ambientbrightness/app/Composition/AmbientBrightnessComponent.kt`.
- Deviation 1: the `:app` component gained a `clock: Clock = SystemClock` constructor parameter (it
  previously hardcoded `SystemClock`) and hands that same instance to `AmbientLightUseCase`, so the
  spec's "one injected `Clock` serves both — never `System.currentTimeMillis()` directly" now holds
  for the whole chain. Without it T5's virtual-time integration test cannot be deterministic. The
  default keeps every existing caller (Application, `AmbientBrightnessComponentTest`) source-
  compatible.
- Deviation 2: the design fixed the profile constants but not the smoothing *formula* (it says
  "slew/EMA smoothing"). EMA was chosen over a lux-domain slew because the lux span is not known to
  the stabiliser (`maxRangeLux` belongs to the delegate/profile factory), and because an EMA makes
  "reaches the target within the reaction window" (AC1.1) literally true: once `elapsedMillis ≥
  reactionWindowMillis` the fraction saturates at 1 and the value snaps to the target. The two noise
  band constants above are new and owned here.
- Usage note for T5/F7: because the folder `ResponseTuning/ResponseTuningService/` shares its name
  with the `ResponseTuningService` classifier, the boundary is imported in its `…ResponseTuningService.ResponseTuningService`
  form — the same shape F1's `…AmbientLightSensing.AmbientLightService.AmbientLightService` already
  uses (`import …ResponseTuning.ResponseTuningService.DefaultResponseTuningService` for the
  implementation is unaffected).
- Verification (from the repo root): `.\gradlew.bat :core:test --no-daemon --console=plain` →
  BUILD SUCCESSFUL, 23 test classes / 128 tests, 0 failures, 0 errors (22 / 118 before T3: +1 class,
  +10 tests, all in the new `UserConfigurableStabiliserTest`); `.\gradlew.bat :app:assembleDebug
  --no-daemon --console=plain` → BUILD SUCCESSFUL. `ArchitectureStructureTest` (folder/package +
  Android-import guard) green.
- Test mapping: AC1.1 → fast reaches in one step / slow only gradually; AC3.1 → change shorter than
  the debounce window is not applied at all; AC3.2 → sustained change starts when the window
  elapses (asserted at 160.6 lx, i.e. 500 ms of the 1650 ms window); AC3.3 → small alternating
  readings (100/102/98/101/99/103/97) emit the stable 100 throughout; AC1.3 → steady level settles on
  its target and does not oscillate; AC1.4 → changing the `StateFlow` mid-run applies on the next
  call; plus delegate-rejection, invalid-value and monotone-approach cases.
