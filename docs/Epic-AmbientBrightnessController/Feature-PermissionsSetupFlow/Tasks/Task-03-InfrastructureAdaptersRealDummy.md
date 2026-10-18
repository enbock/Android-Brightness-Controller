# T3 — Infrastructure adapters (real + Dummy)

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-PermissionsSetupFlow.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T3 — Infrastructure adapters (real + Dummy)**: implement `AndroidNotificationPermissionChecker` (`NotificationManagerCompat.areNotificationsEnabled`/`checkSelfPermission`) and `AndroidSetupStateStore`, plus `DummyNotificationPermissionChecker` and `DummySetupStateStore`. *Files: `:infrastructure/src/main/kotlin/com/ambientbrightness/infrastructure/**`.* *Verify: instrumented persistence test; `:app` runs on the Dummies.* (depends on: T1)

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `c0035e1`.*

- Added the `:infrastructure` adapters under `com.ambientbrightness.infrastructure.AppSetupService`, one responsibility folder per contract with the `<Client>/<Android|Dummy>/<Impl>.kt` convention: `NotificationPermissionChecker/Android/AndroidNotificationPermissionChecker.kt`, `NotificationPermissionChecker/Dummy/DummyNotificationPermissionChecker.kt`, `SetupStateStore/Android/AndroidSetupStateStore.kt`, `SetupStateStore/Dummy/DummySetupStateStore.kt`.
- `AndroidNotificationPermissionChecker` combines `NotificationManagerCompat.areNotificationsEnabled()` with `ContextCompat.checkSelfPermission(...POST_NOTIFICATIONS)` (a granted permission can still be switched off in the system settings, so only the combination matches what the user sees). Android offers no change callback for either read, so the state is a re-readable `StateFlow` refreshed in a documented `refresh()`, mirroring F5's `AndroidBatteryOptimizationClient`; the two platform reads are constructor seams with a `companion create(context)`.
- `AndroidSetupStateStore` is a `SharedPreferences`-backed store (`setup_state` file), the same mechanism and reasoning F5 T5 used for `AndroidControllerStateStore`: the contract is a synchronous `var`, which DataStore cannot serve. `setupAcknowledged` defaults to `false`, and `lastShownAtMillis` is written/removed explicitly so an absent value reads back as `null` instead of a misleading `0L`.
- The Dummy variants are in-memory only (settable `MutableStateFlow` grant, constructor-injected store facts), so `:app` keeps running without a device. No F1/F2/F5 checker was re-implemented and neither `:core` nor `:app` was touched.
- Tests: `:infrastructure` JUnit 4 unit test `infrastructure/src/test/kotlin/com/ambientbrightness/infrastructure/DummyAppSetupAdaptersTest.kt` (3 tests: default/false and toggled grant, initially-granted grant, absent store defaults plus both facts kept in memory). Instrumented test `infrastructure/src/androidTest/kotlin/com/ambientbrightness/infrastructure/AndroidAppSetupAdaptersTest.kt` (`@RunWith(AndroidJUnit4::class)`, platform APIs only, 4 tests: absent values, persistence round-trip across instances, clearing the shown time, and the checker reading the live state twice) — compiled here, no device attached.
- Verification: `.\gradlew.bat :infrastructure:test :infrastructure:assembleDebugAndroidTest --no-daemon --console=plain` → BUILD SUCCESSFUL; `:infrastructure:test` 48 tests / 0 failures / 0 errors / 0 skipped (13 suites; 3 of them in `DummyAppSetupAdaptersTest`), and `:infrastructure:assembleDebugAndroidTest` compiled the instrumented sources.
