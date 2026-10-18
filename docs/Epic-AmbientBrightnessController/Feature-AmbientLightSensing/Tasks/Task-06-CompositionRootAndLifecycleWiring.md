# T5 — Composition root and lifecycle wiring

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-AmbientLightSensing.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T5 — Composition root and lifecycle wiring**: bind clients, use-case and service in
  `:app`, wire `start()`/`stop()` so `stop()` unregisters the listener idempotently and the
  screen-off gate is honoured. *Files: `:app/src/main/kotlin/com/ambientbrightness/app/**`.*
  *Verify: instrumented test asserting no sensor events after `stop()` (AC1.3) and idle while
  screen off (AC1.4).*
  *Done: implementation commit `d3ce1e7`; progress documentation commit `aada8a5`. Added
  `AmbientBrightnessApplication` and the
  app-scoped `AmbientBrightnessComponent`, which constructs the Android sensor/screen clients,
  `AmbientLightUseCase`, range-configured `OutlierClampingStabiliser`, `DefaultAmbientLightService`
  and `AmbientLightCollector`. A missing/invalid range uses the safe positive 1,000-lux fallback.
  The activity collects while started and cancels/stops on stop and destroy; the collector cancels
  sensor collection whenever screen state turns off, forwarding no screen-off samples and restarting
  when it turns on. Added app instrumentation coverage using controllable clients for listener
  unregistration/no post-stop samples and screen-off suppression/restart; no device sensor behavior
  is assumed. Verification: `:app:testDebugUnitTest :core:test :infrastructure:test
  :app:assembleDebug --no-daemon` → BUILD SUCCESSFUL; `:app:connectedDebugAndroidTest --no-daemon`
  → BUILD SUCCESSFUL on `medium_phone` API 36 (2 tests, 0 failures). Files changed:
  `app/build.gradle.kts`, `app/src/main/AndroidManifest.xml`,
  `app/src/main/kotlin/com/ambientbrightness/app/Composition/AmbientBrightnessApplication.kt` (moved by T9),
  `app/src/main/kotlin/com/ambientbrightness/app/Composition/AmbientBrightnessComponent.kt` (moved by T9),
  `app/src/main/kotlin/com/ambientbrightness/app/AmbientLightSensing/Collector/AmbientLightCollector.kt`
  (moved by T9),
  `app/src/main/kotlin/com/ambientbrightness/app/MainActivity.kt`,
  `app/src/androidTest/kotlin/com/ambientbrightness/app/AmbientBrightnessComponentTest.kt`.
  The separate progress-document commit is the commit containing this note.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.
