# T1 — Domain model, boundary interface and classifier

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-AmbientLightSensing.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T1 — Domain model, boundary interface and classifier**: add `LuxSample`, `SensorProfile`,
  `SensorAvailability`, the `LightLevel` enum + `LightLevelClassifier`, the `AmbientLightSensorClient`
  and `ScreenStateClient` interfaces, and the `AmbientLightService` boundary interface in `:core`.
  Keep them Android-free. *Files: `:core/src/main/kotlin/com/ambientbrightness/core/**`.*
  *Verify: compiles with no Android dependency; unit tests for classifier boundaries.*
  *Done: commit `6699490`. All types live in `:core`, package `com.ambientbrightness.core`, one
  directory per type *(superseded: `:core` now files every type by responsibility — the F1 types
  under `AmbientLightSensing/`, the F2 types under `AutomaticBrightnessControl/` — with the data and
  port types directly beside the type they serve and the cross-cutting `Clock` in `Shared/`; future
  `:core` tasks must use those responsibility folders instead of one directory per type. Corrected by
  T10)*, no `android.*` import and no Android plugin. `:core` gained
  `implementation(libs.kotlinx.coroutines.core)` (1.10.2) through the version catalog for
  `Flow`/`StateFlow`; the catalog also gained `kotlinx-coroutines-test` (1.10.2) for T3.*
  *Design gaps the architect resolved before implementation (classifier thresholds, availability
  shape, `ReadingStabiliser` signature, client interfaces) are recorded in `### Architecture &
  Components` / `### Data Flow` and landed with this commit; no change to acceptance criteria.*
  *Files added: `:core/src/main/kotlin/com/ambientbrightness/core/AmbientLightSensing/{LuxSample,SensorProfile,SensorUnavailableReason,SensorAvailability,AmbientLightSensorClient,ScreenStateClient}.kt`,
  `:core/src/main/kotlin/com/ambientbrightness/core/AmbientLightSensing/{LightLevel/LightLevel,LightLevel/LightLevelClassifier,ReadingStabiliser/ReadingStabiliser,AmbientLightService/AmbientLightService}.kt`,
  `:core/src/test/kotlin/com/ambientbrightness/core/{LuxSample,SensorProfile,SensorAvailability,LightLevel,LightLevelClassifier,AmbientLightService}/<Type>Test.kt`
  and `:core/build.gradle.kts` (catalog dependency).*
  *Public boundary signatures:*
  ```kotlin
  interface AmbientLightService {
      val availability: StateFlow<SensorAvailability>
      fun start(): Flow<LuxSample>       // cold; collection registers the listener
      fun stop()                         // idempotent; unregisters deterministically
  }

  interface AmbientLightSensorClient {
      val availability: SensorAvailability
      fun register(onLux: (Float) -> Unit)
      fun unregister()                   // idempotent; no further callbacks
  }

  interface ScreenStateClient {
      val isScreenOn: StateFlow<Boolean> // hot, conflated
  }

  interface ReadingStabiliser {
      fun stabilise(sample: LuxSample): LuxSample? // null = rejected, nothing emitted
  }

  sealed interface SensorAvailability {
      data class Available(val profile: SensorProfile) : SensorAvailability
      data class Unavailable(
          val reason: SensorUnavailableReason,
          val userMessage: String = reason.defaultUserMessage,
      ) : SensorAvailability
  }
  enum class SensorUnavailableReason(val defaultUserMessage: String) { NoSensor, NotUsable }
  object LightLevelClassifier { fun classify(lux: Float): LightLevel }
  ```
  *Verify results: `.\gradlew.bat :core:test --no-daemon` → BUILD SUCCESSFUL, 37 tests / 0 failures
  (`LightLevelClassifierTest` 22 — every level boundary from both sides plus `0`, `Float.MAX_VALUE`,
  NaN, ±Infinity and negative lux; `SensorAvailabilityTest` 5 — AC2.1, AC2.2, AC2.3; `LuxSampleTest`
  3 — incl. AC3.3 `0 lux`; `SensorProfileTest` 2 — AC2.2; `LightLevelTest` 1; contract test 4 for the
  boundary shapes). `.\gradlew.bat :core:dependencies --configuration compileClasspath --no-daemon`
  → only `kotlinx-coroutines-core(-jvm):1.10.2`, no `androidx`/`com.android` artifact.
  `.\gradlew.bat :core:test :app:assembleDebug --no-daemon` → BUILD SUCCESSFUL.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.
