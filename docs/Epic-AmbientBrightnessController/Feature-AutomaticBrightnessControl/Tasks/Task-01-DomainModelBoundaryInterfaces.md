# T1 — Domain model & boundary interfaces

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-AutomaticBrightnessControl.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T1 — Domain model & boundary interfaces**: add `BrightnessValue`, `BrightnessRange`,
  `BrightnessWriteResult`, `BrightnessStatus`, `BuiltInAdaptiveBrightnessState`, `BrightnessContext`
  and `BrightnessSnapshot`, plus the `AutomaticBrightnessController`, `BrightnessMappingCurve`,
  `SystemBrightnessClient`, `DeviceBrightnessRangeClient`, `WriteSettingsPermissionChecker`,
  `HysteresisGate` and `BrightnessTransform` interfaces in `:core`, Android-free.
  *Files: `:core/src/main/kotlin/com/ambientbrightness/core/**`.*
  *Verify: compiles with no Android dependency; VOs enforce `>= 1`; contract tests on fakes.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `8d137e13aab57afb3e28464d22953d81b7abe9f7`. Added these files:*

- `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/AutomaticBrightnessController.kt`
- `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BrightnessContext.kt`
- `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BrightnessMappingCurve/BrightnessMappingCurve.kt`
- `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BrightnessRange.kt`
- `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BrightnessSnapshot.kt`
- `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BrightnessStatus.kt`
- `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BrightnessTransform.kt`
- `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BrightnessValue.kt`
- `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BrightnessWriteResult.kt`
- `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BuiltInAdaptiveBrightnessState.kt`
- `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/DeviceBrightnessRangeClient.kt`
- `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/HysteresisGate/HysteresisGate.kt`
- `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/SystemBrightnessClient.kt`
- `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/WriteSettingsPermissionChecker.kt`
- `core/src/test/kotlin/com/ambientbrightness/core/BrightnessContractsTest.kt`
- `core/src/test/kotlin/com/ambientbrightness/core/testdoubles/BrightnessContractFakes.kt`

  `BrightnessValue` rejects values below 1; `BrightnessRange` requires a positive minimum and
  `max >= min`. Status exposes the last applied value, last lux, write outcome and error. Range
  discovery is a separate client; permission grant is observable as a `StateFlow`. No deviations.
  Verification passed: `:core:test`; `:core:dependencies --configuration compileClasspath`
  (no Android artifacts); `:core:test :infrastructure:test :app:assembleDebug`.
