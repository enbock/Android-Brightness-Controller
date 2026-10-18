# T1 — Core requirement model & boundary interfaces

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-PermissionsSetupFlow.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T1 — Core requirement model & boundary interfaces**: add `SetupStatus`, `SetupRequirement`, `SetupRequirementId`, `RequirementSeverity`, `RequirementState` (+ `MissingReason`/`UnsupportedReason`), `SetupAction`, the `AppSetupService` boundary interface, and the `NotificationPermissionChecker` and `SetupStateStore` interfaces. Keep them Android-free. *Files: `:core/src/main/kotlin/com/ambientbrightness/core/AppSetupService/**`.* *Verify: compiles without Android; the model expresses every US1–US4 state.* (depends on: none)

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `474d557`.*

- Added the Android-free setup model in `:core` under `com.ambientbrightness.core.AppSetupService`, one type per file: `SetupStatus`, `SetupRequirement`, `SetupRequirementId`, `RequirementSeverity`, `RequirementState`, `MissingReason`, `UnsupportedReason`, `SetupAction`.
- Added the boundaries F7 and `:infrastructure` consume: `AppSetupService` (`status`/`isReadyToStart` as `StateFlow`, `refresh()`, `acknowledgeSetup()`), `NotificationPermissionChecker` (`isGranted: Flow<Boolean>`) and `SetupStateStore` (`setupAcknowledged`, `lastShownAtMillis`).
- Documented the fixed severity mapping in KDoc (`AmbientLightSensor`/`WriteSettings` = `Blocking`, `Notifications`/`BatteryOptimization` = `Advisory`) and that only `Blocking` requirements gate Start; each KDoc names the user story/AC it serves.
- No aggregator, no persistence and no navigation: behaviour lands in T2–T5. The aggregator will own the id→severity mapping at runtime.
- Test `core/src/test/kotlin/com/ambientbrightness/core/AppSetupService/AppSetupModelTest.kt` (JUnit 5 Jupiter, backticked names) constructs every `RequirementState` variant and `SetupAction`, branches over both hierarchies with an exhaustive `when` and no `else`, asserts the severity rule for all four ids, and covers `SetupStatus` equality/`copy` plus `blockers` (only blocking, not-`Satisfied` ids).
- Verification: `.\gradlew.bat :core:test --no-daemon --console=plain` → BUILD SUCCESSFUL, 212 tests / 0 failures / 0 errors / 0 skipped (32 suites; 11 of them in `AppSetupModelTest`). No `android.*` or `androidx.*` import in the new files.
