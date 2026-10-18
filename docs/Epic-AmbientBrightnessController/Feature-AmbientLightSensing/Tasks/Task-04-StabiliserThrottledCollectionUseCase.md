# T3 — Stabiliser + throttled collection use-case

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-AmbientLightSensing.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T3 — Stabiliser + throttled collection use-case**: implement `ReadingStabiliser`,
  `OutlierClampingStabiliser` (clamp, outlier rejection, 0-lux validity) and `AmbientLightUseCase`
  with injectable `Clock` and injected stabiliser, throttling to one value per interval.
  *Files: `:core/src/main/kotlin/com/ambientbrightness/core/AmbientLightSensing/AmbientLightUseCase/**`.*
  *Verify: unit tests for AC1.2, AC3.1–AC3.4 with fake clients and a fixed clock.*
  *Done: commit `071f29d`. Added `Clock`/`SystemClock` (`:core`, pure Kotlin),
  `OutlierClampingStabiliser` (clamps to `0f..maxRangeLux` keeping the timestamp, always accepts the
  first value, rejects a single sample whose delta exceeds `max(minOutlierDeltaLux,
  outlierFactor * lastAcceptedValue)`, accepts after `consecutiveRejectionsBeforeAccept`
  consecutive rejections and then resets the counters) and `AmbientLightUseCase` (cold
  `channelFlow` with no Android dependency; registers on collection, keeps only the newest raw
  `LuxSample` in a `@Volatile` field with `clock.nowMillis()`, ticks `while (true) {
  delay(samplingIntervalMillis) }`, forwards at most one sample per tick to `stabiliser.stabilise`
  and emits only non-null results; `unregister()` runs once in a `finally` block).
  *Defaults actually used: `DEFAULT_SAMPLING_INTERVAL_MILLIS = 500L`; `OutlierClampingStabiliser`
  `outlierFactor = 3f`, `minOutlierDeltaLux = 20f`, `consecutiveRejectionsBeforeAccept = 3`.*
  *Files added: `:core/src/main/kotlin/com/ambientbrightness/core/Shared/Clock.kt`,
  `:core/src/main/kotlin/com/ambientbrightness/core/AmbientLightSensing/{ReadingStabiliser/OutlierClampingStabiliser,AmbientLightUseCase/AmbientLightUseCase}.kt`,
  `:core/src/test/kotlin/com/ambientbrightness/core/{OutlierClampingStabiliser/OutlierClampingStabiliserTest,AmbientLightUseCase/AmbientLightUseCaseTest}.kt`,
  `:core/src/test/kotlin/com/ambientbrightness/core/testdoubles/{FakeClock,FakeAmbientLightSensorClient,FakeReadingStabiliser}.kt`;
  changed `:core/build.gradle.kts` (`testImplementation(libs.kotlinx.coroutines.test)`).*
  *Verify results: `.\gradlew.bat :core:test --no-daemon` → BUILD SUCCESSFUL (49 tests, 0 failed);
  `.\gradlew.bat :core:test :infrastructure:test :app:assembleDebug --no-daemon` → BUILD SUCCESSFUL;
  `.\gradlew.bat :core:dependencies --configuration compileClasspath --no-daemon` → only
  `kotlin-stdlib` and `kotlinx-coroutines-core` (no `androidx`/`com.android` artifact), so
  `:core` stays Android-free.*
  *Deviations: `FakeClock` exposes the mutable field as `nowMillis` plus `advanceBy`/`advanceTo`
  helpers; the throttling loop tracks the consumed reading with a version counter instead of
  storing a second field, because a local `@Volatile` variable is not allowed in Kotlin.*
  *Appended note (architecture task T6): the raw-reading holder was extracted from
  `AmbientLightUseCase.kt` into
  `:core/src/main/kotlin/com/ambientbrightness/core/AmbientLightSensing/AmbientLightUseCase/Task/LatestReading.kt`
  (`internal`, package `...AmbientLightUseCase.Task`); the `AmbientLightUseCase/**` entry above still
  covers the file, and `AmbientLightUseCaseTest` stayed unmodified.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.
