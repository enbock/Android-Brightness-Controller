# T4 — App navigation, lifecycle and wiring

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-PermissionsSetupFlow.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T4 — App navigation, lifecycle and wiring**: implement `SetupFlowNavigator` (`perform(SetupAction)` → `ACTION_MANAGE_WRITE_SETTINGS` with `package:` URI, battery-optimization settings intent, notification permission launcher; `ActivityNotFoundException` fallback chain), `SetupFlowCollector` (thin pass-through) and `SetupLifecycleObserver` (ON_RESUME → `refresh()`); wire DI in the composition root and add `POST_NOTIFICATIONS` to the manifest. *Files: `:app/src/main/kotlin/com/ambientbrightness/app/SetupFlow*/**`, `:app/src/main/AndroidManifest.xml`.* *Verify: intent/fallback tests; ON_RESUME triggers `refresh()`.* (depends on: T2, T3)

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `46325a2692e4956a18c17291a2119b83fcecd2fe`.*

- Added `app/src/main/kotlin/com/ambientbrightness/app/SetupFlow/` with three thin `:app` pieces: `Collector/SetupFlowCollector.kt` (logic-free forwarder of `status` and `isReadyToStart`, mirroring `BrightnessServiceCollector`), `Navigator/SetupFlowNavigator.kt` and `SetupFlow/Lifecycle/SetupLifecycleObserver.kt`.
- **Navigator design (deviation in shape, not in behaviour).** The navigation *decision* is pure data: a new `Navigator/SetupScreen.kt` (`action` + optional `package:` `dataUri`) describes a system screen, and opening one is a single injected seam `(SetupScreen) -> Boolean` that reports whether the device handled it. The three `WRITE_SETTINGS` candidates form the documented fallback order (per-app `ACTION_MANAGE_WRITE_SETTINGS` with `package:`, the same action without the URI, then `ACTION_APPLICATION_DETAILS_SETTINGS`), and the battery row uses `ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS` — never the restricted `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`. `RequestNotificationPermission` delegates to an injected launcher (F7 supplies `rememberLauncherForActivityResult(RequestPermission())`), and `None` does nothing.
  *Why the seam returns a boolean instead of catching `ActivityNotFoundException` in the navigator:* the exception can only be raised by the real `startActivity`, and `Intent`/`Uri`/`ActivityNotFoundException` are Android types that a JVM unit test cannot instantiate (verified: constructing them throws `RuntimeException("Stub!")` under the plain `android.jar`). Moving the `try/catch (ActivityNotFoundException)` into `SetupFlowNavigator.create(context, ...)` keeps the platform surface to one lambda, keeps `:app`'s behaviour identical (the chain is still walked on a missing Activity and never crashes), and makes the whole flow unit-testable off-device.
- `SetupLifecycleObserver` is a `DefaultLifecycleObserver` whose `onResume` calls `AppSetupService.refresh()`, so a grant made in a system screen is re-derived without a restart and without polling (US2 AC2.4).
- **Wiring.** `Composition.AmbientBrightnessComponent` gained Dummy-defaulted `notificationPermissionChecker` and `setupStateStore` parameters, builds one app-scoped `SetupRequirementAggregator` over the existing F1 `service`, F2 `writeSettingsPermissionChecker`, F5 `batteryOptimizationClient` and the injected `Clock`, and exposes `setupService: AppSetupService` and `setupFlowCollector`. `Composition.AmbientBrightnessApplication` passes `AndroidNotificationPermissionChecker.create(this)` and `AndroidSetupStateStore(this)`. `POST_NOTIFICATIONS` was already declared by F5 T4, so the manifest is unchanged. The composition-root isolation rule still holds: `git grep "com\.ambientbrightness\.infrastructure\." -- app/src/main/kotlin` matches only `Composition/**`.
- Files added: `:app .../SetupFlow/Collector/SetupFlowCollector.kt`, `.../SetupFlow/Navigator/SetupFlowNavigator.kt`, `.../SetupFlow/Navigator/SetupScreen.kt`, `.../SetupFlow/Lifecycle/SetupLifecycleObserver.kt`, `:app test SetupFlowNavigatorTest.kt` (7 tests) and `SetupLifecycleObserverTest.kt` (1 test).
- Tests assert the fallback chain end to end: the per-app screen is preferred; a missing per-app screen falls back to the plain action and then to the app details screen; a device that handles none is tolerated without crashing; the battery row uses the advisory action; the notification row requests the runtime permission and opens no screen; `None` starts nothing; and `onResume` calls `refresh()` exactly once.
- Verification (from the repo root): `.\gradlew.bat :core:test :infrastructure:test :app:testDebugUnitTest :app:assembleDebug :app:assembleDebugAndroidTest --no-daemon --console=plain` → BUILD SUCCESSFUL, 0 failures; `:core` 33 classes / 226 tests, `:infrastructure` 13 / 48, `:app` 5 / 16 (T4 adds `SetupFlowNavigatorTest` and `SetupLifecycleObserverTest`).
- Cleanup: an untracked scratch probe (`app/src/test/kotlin/com/ambientbrightness/app/ScratchAndroidBehaviourTest.kt`) left in the tree while establishing the Android-stub behaviour above was deleted and never committed.
