# T3 — Device range resolution, scaling, clamping and dead-band

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-AutomaticBrightnessControl.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T3 — Device range resolution, scaling, clamping and dead-band**: implement
  `BrightnessRangeResolver`, `BrightnessRangeScaler` and `MinimumStepDeadBand` as reusable pure-core
  helpers. The controller orchestration consumes these helpers in T5.
  *Files: `:core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/{BrightnessRangeResolver,BrightnessRangeScaler,MinimumStepDeadBand}/**`.*
  *Verify: unit tests for AC2.2/AC2.3, never-0 (AC2.1) and suppression below min step (AC1.2).*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `5f5d0d1a6785e6917ce1be4d5a13dee9141b14e6`; progress documentation
commit `6b3e978`.*

- Added `BrightnessRangeResolver`, which passes through a valid reported range and otherwise uses
  the core fallback `BrightnessRange(1, 255)`; hardware discovery remains an infrastructure concern.
- Added `BrightnessRangeScaler.scale(fraction, range)` for linear inclusive-range scaling, nearest
  device-unit rounding, saturation/clamping, singleton ranges, and explicit rejection of NaN.
- Added `MinimumStepDeadBand` with a default minimum step of 1 device unit. It returns the existing
  `BrightnessWriteResult.Suppressed` status below the threshold and updates its remembered value
  only when the caller records a successfully applied write.
- Added 10 focused JVM tests for AC1.2 and AC2.1–2.3, covering range resolution, endpoints,
  midpoint, saturation, singleton and NaN scaling, never-zero behavior, exact/below-threshold
  decisions, first request, and unchanged prior value after suppression.
- Files:
  `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BrightnessRangeResolver/BrightnessRangeResolver.kt`,
  `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BrightnessRangeScaler/BrightnessRangeScaler.kt`,
  `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/MinimumStepDeadBand/MinimumStepDeadBand.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/BrightnessRangeResolver/BrightnessRangeResolverTest.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/BrightnessRangeScaler/BrightnessRangeScalerTest.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/MinimumStepDeadBand/MinimumStepDeadBandTest.kt`.
- Verification passed: `:core:test --no-daemon` (80 tests, 0 failures); `:core:dependencies
  --configuration compileClasspath --no-daemon` (no Android artifacts); and `:core:test
  :infrastructure:test :app:assembleDebug --no-daemon`. No deviations; orchestration and writes
  remain deferred to T5/T4.
