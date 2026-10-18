# T1 — Offset domain model & boundary interfaces

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-ManualBrightnessOffset.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T1 — Offset domain model & boundary interfaces**: add `BrightnessOffset`,
  `BrightnessOffsetRange`, `BrightnessOffsetApplication`, and the `BrightnessOffsetSource`,
  `BrightnessOffsetService` and `OffsetApplicationSink` interfaces to `:core`, Android-free (no
  persistence port — the shared `SettingsStore` supplies persistence). *Files:
  `:core/src/main/kotlin/com/ambientbrightness/core/BrightnessOffset/**`,
  `.../BrightnessOffsetService/**`.* *Verify: compiles with no Android dependency; `BrightnessOffset`
  boundary tests pass.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `e8b523c8f7d0dbf8315ce4e93fad03c3a6f5be50`.*

- Built the `:core` domain model, Android-free. `BrightnessOffsetRange` is an `object` holding the
  only definition of the limits — `MIN = -50`, `MAX = +50`, `STEP = 1`, `DEFAULT = 0` — so no other
  type duplicates the numbers. `BrightnessOffset` follows the `ResponseSpeed` pattern
  (`@ConsistentCopyVisibility data class` with a private constructor and a validating `of(percent)`
  factory), but rejects instead of clamping: `of` throws `IllegalArgumentException` outside
  `MIN..MAX` or off the `STEP` grid, as the design requires. `BrightnessOffset.DEFAULT` is the
  neutral `0` value and is reused by `reset()` (AC2.4, AC3.3).
- Built `BrightnessOffsetApplication` as a `sealed interface` with `data object` variants
  `NotEvaluated`, `FullyApplied`, `ClampedToMinimum`, `ClampedToMaximum`, the observable clamp
  indicator (AC2.1–AC2.3), matching the existing `SensorAvailability`/`BrightnessWriteResult` style.
- Built the three boundary interfaces in the `BrightnessOffsetService` package.
  `BrightnessOffsetSource` exposes `offset: StateFlow<BrightnessOffset>` and
  `application: StateFlow<BrightnessOffsetApplication>` (F2 and F7 read these);
  `BrightnessOffsetService` extends it with `setOffset`/`adjustBy`/`reset`; `OffsetApplicationSink`
  is the write-side port the offset transform will publish the clamp indicator through, so
  `BrightnessTransform.apply(...): Float` is not widened.
- No persistence port was added: F3 reuses F4's shared `SettingsStore`, so `:core` gained no
  `android.`/`androidx.datastore` import (the `ArchitectureStructureTest` guard, which scans
  `src/main/kotlin` for Android imports and folder/package mismatches, passes unchanged).
- Files added:
  `core/src/main/kotlin/com/ambientbrightness/core/BrightnessOffset/BrightnessOffset.kt`,
  `.../core/BrightnessOffset/BrightnessOffsetRange.kt`,
  `.../core/BrightnessOffset/BrightnessOffsetApplication.kt`,
  `core/src/main/kotlin/com/ambientbrightness/core/BrightnessOffsetService/BrightnessOffsetSource.kt`,
  `.../core/BrightnessOffsetService/BrightnessOffsetService.kt`,
  `.../core/BrightnessOffsetService/OffsetApplicationSink.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/BrightnessOffset/BrightnessOffsetTest.kt`.
- Deviation 1: the task text asked for JUnit 4 (`org.junit.Test`/`org.junit.Assert`), but `:core`
  runs on the JUnit 5 platform (`junit-jupiter` + `junit-platform-launcher` in
  `core/build.gradle.kts`, `useJUnitPlatform()`), and every existing `:core` test imports
  `org.junit.jupiter.api.*`. The test therefore uses JUnit Jupiter to match the module instead of
  adding a second test framework.
- Deviation 2: `STEP = 1` means every integer percent is already on the step grid, so the step
  `require` is defensive (unreachable for an `Int` at the current granularity) and the tests assert
  acceptance for every whole percent rather than a rejected non-step value; the guard is kept so a
  future coarser `STEP` rejects misaligned values.
- Verification (from the repo root):
  `$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot'` then
  `.\gradlew.bat :core:test --no-daemon --console=plain` → BUILD SUCCESSFUL, 25 test classes /
  147 tests, 0 failures (7 of them the new `BrightnessOffsetTest`).
