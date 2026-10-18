# T4 — Availability, startup timeout and status surfacing

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-AmbientLightSensing.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T4 — Availability, startup timeout and status surfacing**: resolve availability at start,
  apply the startup timeout → `Unavailable(NotUsable)`, and expose both states through
  `AmbientLightService.availability`; add the collector that carries the non-technical message
  to the caller (no UI). *Files: `:core/.../AmbientLightSensing/AmbientLightService/**`,
  `:app/src/main/kotlin/com/ambientbrightness/app/AmbientLightSensing/Collector/**` (moved by T9).*
  *Verify: unit tests for AC2.1–AC2.3.*
  *Done: implementation commit `ad7f8a1`; documentation commit `9fdc255`. Added
  `DefaultAmbientLightService` with injectable `startupTimeoutMillis` (default 5,000 ms): it checks
  sensor availability before registration, publishes `Available(profile)` only after the first
  emitted reading, times out silent sensors as `Unavailable(NotUsable)`, and serializes cold
  collections; `stop()` cancels the session and invokes the client's idempotent unregister.
  Added the app `AmbientLightCollector` to forward samples and availability to caller callbacks,
  plus the `:app` → `:core` project dependency. Files changed:
  `core/src/main/kotlin/com/ambientbrightness/core/AmbientLightSensing/AmbientLightService/DefaultAmbientLightService.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/AmbientLightService/DefaultAmbientLightServiceTest.kt`,
  `app/src/main/kotlin/com/ambientbrightness/app/AmbientLightSensing/Collector/AmbientLightCollector.kt`
  (moved by T9),
  `app/build.gradle.kts`. Tests: focused `:core:test` → BUILD SUCCESSFUL (54 tests, 0 failures;
  five service tests cover missing sensor, first-reading/profile, timeout, valid zero lux and
  cancellation/stop); `:core:test :infrastructure:test :app:assembleDebug` → BUILD SUCCESSFUL.
  No acceptance-criteria or behavior deviations.*
  *Appended note (architecture task T7): the two internal responsibilities of the service were
  split into `:core/src/main/kotlin/com/ambientbrightness/core/AmbientLightSensing/AmbientLightService/Task/{CollectionSessionRegistry,StartupAvailabilityGate}.kt`
  (`internal`, package `...AmbientLightService.Task`); `DefaultAmbientLightService` keeps its
  constructor, `init` validation contract, `availability`, `start()` and `stop()` and only delegates
  to them, so the `AmbientLightService/**` entry and the `DefaultAmbientLightServiceTest` file listed
  above stay valid and the test stayed unmodified.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.
