# T2 — Infrastructure adapters (real + Dummy)

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-AmbientLightSensing.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T2 — Infrastructure adapters (real + Dummy)**: implement `AndroidAmbientLightSensorClient`
  (registration, events, `SensorProfile` from `getMaximumRange`/`getMinDelay`) and
  `AndroidScreenStateClient`; add `DummyAmbientLightSensorClient` for tests/development.
  *Files: `:infrastructure/src/main/kotlin/com/ambientbrightness/infrastructure/**`.*
  *Verify: instrumented smoke test that a light event is received; Dummy path runs on emulator.*
  *Done: commit `d650dc4`. Added `SensorProfileFactory` (pure Kotlin, `null` for a non-positive
  `maxRangeLux`, negative `minDelayMicros` clamped to `0`), `AndroidAmbientLightSensorClient`
  (eager availability via the factory, idempotent registration with callback replacement,
  deterministic idempotent `unregister`), `AndroidScreenStateClient` (event-driven `callbackFlow`
  over `PowerManager.isInteractive` + `DisplayManager.DisplayListener`, unregistered in `awaitClose`,
  `stateIn(WhileSubscribed(5_000))`) and the two Dummy clients (`DummyAmbientLightSensorClient` with
  a settable `available` flag and `emit`; `DummyScreenStateClient` with `setScreenOn`).*
  *Verify results: `.\gradlew.bat :infrastructure:test :infrastructure:assembleDebug
  :infrastructure:assembleDebugAndroidTest --no-daemon` → BUILD SUCCESSFUL (unit tests green;
  instrumented test compiled). `.\gradlew.bat :core:test :infrastructure:test :app:assembleDebug
  --no-daemon` → BUILD SUCCESSFUL. `.\gradlew.bat :infrastructure:connectedDebugAndroidTest
  --no-daemon --console=plain` → BUILD SUCCESSFUL on `medium_phone` (Android 16 / API 36);
  `AndroidAmbientLightSensorClientTest.lightEventReachesCallback` passed (1 test, 0 failures).*
  *Files added: `:infrastructure/src/main/kotlin/com/ambientbrightness/infrastructure/{AmbientLightSensing/AmbientLightSensorClient/Android/AndroidAmbientLightSensorClient,AmbientLightSensing/AmbientLightSensorClient/Dummy/DummyAmbientLightSensorClient,AmbientLightSensing/ScreenStateClient/Android/AndroidScreenStateClient,AmbientLightSensing/ScreenStateClient/Dummy/DummyScreenStateClient,AmbientLightSensing/SensorProfileFactory/SensorProfileFactory}.kt` (regrouped by T8),
  `:infrastructure/src/test/kotlin/com/ambientbrightness/infrastructure/{SensorProfileFactoryTest,DummyAmbientLightSensorClientTest,DummyScreenStateClientTest}.kt`,
  `:infrastructure/src/androidTest/kotlin/com/ambientbrightness/infrastructure/AndroidAmbientLightSensorClientTest.kt`;
  changed `:infrastructure/build.gradle.kts` (catalog deps + `testInstrumentationRunner`) and
  `gradle/libs.versions.toml` (`junit:junit:4.13.2`, `androidx.test:runner:1.7.0`,
  `androidx.test.ext:junit:1.3.0`).*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.
