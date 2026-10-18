# T2 — Aggregation service and per-requirement tasks

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-PermissionsSetupFlow.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T2 — Aggregation service and per-requirement tasks**: implement `SetupRequirementAggregator` (implements `AppSetupService`) subscribing to F1 `AmbientLightService.availability`, F2 `WriteSettingsPermissionChecker`, F5 `BatteryOptimizationClient.isExempt` and `NotificationPermissionChecker.isGranted`; add the `Task/` classes that map each source to a `SetupRequirement`, compute `isReadyToStart` and `blockers` (only `Blocking` requirements gate), and make `Notifications` and `BatteryOptimization` `Advisory`. *Files: `:core/.../SetupRequirementAggregator/**`.* *Verify: unit tests against the state matrix and the Start gate with fakes.* (depends on: T1)

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `7548d74`.*

- Added `SetupRequirementAggregator` (`core/src/main/kotlin/com/ambientbrightness/core/SetupRequirementAggregator/SetupRequirementAggregator.kt`, package `com.ambientbrightness.core.SetupRequirementAggregator`), implementing `AppSetupService`. It composes F1 `AmbientLightService.availability`, F2 `WriteSettingsPermissionChecker.hasWriteSettingsPermission`, F5 `BatteryOptimizationClient.isExempt` and F6 `NotificationPermissionChecker.isGranted`, plus the `SetupStateStore`, a `CoroutineScope` and an injected `Shared.Clock`.
- Added the `Task/` mappers (all `internal`, one source value → one `SetupRequirement`): `SensorRequirementTask`, `WriteSettingsRequirementTask`, `NotificationRequirementTask`, `BatteryRequirementTask`.
- **Mapping rules.** Sensor (`Blocking`): `Available` → `Satisfied`/`None`, `Unavailable` (both `NoSensor` and `NotUsable`) → `Unsupported(NoAmbientLightSensor)`/`None`; write settings (`Blocking`): `true` → `Satisfied`/`None`, `false` → `Missing(...)`/`OpenWriteSettings`; notifications (`Advisory`): `true` → `Satisfied`/`None`, `false` → `Missing(NotGranted)`/`RequestNotificationPermission`; battery (`Advisory`): `true` → `Satisfied`/`None`, `false` → `Missing(NotGranted)`/`OpenBatteryOptimizationSettings`. A `null` input (the source has not reported yet) maps to `Unknown`/`None` for every requirement.
- **`Missing(Revoked)` decision.** F2's checker exposes only a boolean, so `WriteSettingsRequirementTask` tracks whether it ever observed `true`: a `false` **after** a satisfied observation is reported as `Missing(Revoked)`, a `false` without one stays `Missing(NotGranted)` (US4 AC2). The memory is monotonic, so repeated derivations keep the same reason. Notifications and battery intentionally stay `NotGranted` (the task contract fixes them, and F5/F6 expose no revocation signal).
- **Derivation.** Each source is collected eagerly in the injected scope into a private "latest value" `StateFlow`; every emission re-derives `SetupStatus` (requirement order `AmbientLightSensor`, `WriteSettings`, `Notifications`, `BatteryOptimization`), so `status`/`isReadyToStart` are never cached. `isReadyToStart` is `blockers.isEmpty()` where `blockers` = the non-`Satisfied` `Blocking` ids only — `Notifications`/`BatteryOptimization` can never appear (US3, US2 AC4). `refresh()` re-derives and re-publishes from the latest observed values; because the sources are observed continuously it is normally idempotent, and it exists as the seam the ON_RESUME hook calls. `acknowledgeSetup()` sets `setupAcknowledged = true` and stores `clock.nowMillis()` in `lastShownAtMillis` (US1 AC3).
- Test doubles: new `core/src/test/kotlin/com/ambientbrightness/core/testdoubles/SetupFlowFakes.kt` with `FakeNotificationPermissionChecker` (null initial value withholds the first emission) and `FakeSetupStateStore`. Reused `FakeAmbientLightService`, `FakeWriteSettingsPermissionChecker`, `FakeBatteryOptimizationClient` and `FakeClock`; added `FakeAmbientLightService.setAvailability(...)` and made `FakeBatteryOptimizationClient` accept a nullable `exempt` (default unchanged) so it can withhold its first emission.
- Test `core/src/test/kotlin/com/ambientbrightness/core/SetupRequirementAggregator/SetupRequirementAggregatorTest.kt` (JUnit 5 Jupiter, 14 tests, `runTest`/`runCurrent`): the full state matrix per requirement (satisfied / missing / unsupported / unknown), the all-`Unknown` seed before any source reports, the `isReadyToStart` gate, `blockers` contents (never notifications/battery), a live update without reconstruction (sensor, write settings and notifications republished in turn), `refresh()` re-derivation/idempotence, and `acknowledgeSetup()` persistence.
- Verification: `.\gradlew.bat :core:test --no-daemon --console=plain` → BUILD SUCCESSFUL, 226 tests / 0 failures / 0 errors / 0 skipped (33 suites; 14 of them in `SetupRequirementAggregatorTest`). No `android.*` or `androidx.*` import in any new or changed file.
