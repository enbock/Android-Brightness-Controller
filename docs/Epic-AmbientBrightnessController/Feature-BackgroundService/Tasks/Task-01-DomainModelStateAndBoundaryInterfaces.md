# T1 — Domain model, state and boundary interfaces

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-BackgroundService.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T1 — Domain model, state and boundary interfaces**: add `BackgroundServiceState`, `StartReason`,
  `StartOutcome`, `BackgroundNotificationContent`, and the `BrightnessBackgroundService`,
  `ForegroundServiceHost`, `ControllerStateStore`, `BatteryOptimizationClient` interfaces in `:core`,
  Android-free. *Files: `:core/src/main/kotlin/com/ambientbrightness/core/**`.*
  *Verify: compiles without Android; `BackgroundServiceState` covers all four stories' states.*
  *Depends on: none.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `43f53bcad972ae91ad206ad3fd06feaa3981cfcc`.*

- Built the frozen F5 contract in `:core`, split across two responsibility folders (folder ==
  package, like `AmbientLightSensing`):
  - `BackgroundService/` — `BackgroundServiceState` (sealed interface: `Stopped`, `Starting`,
    `Running(appliedBrightness: BrightnessValue? = null)`, `StoppedBySystem`, `Failed(reason)`; each
    variant carries the story it serves, US1–US4), `StartReason` (`UserRequest`, `SystemRestart`,
    `Boot`), `StartOutcome` (sealed: `Started`, `Blocked(reason)`), `BackgroundNotificationContent`
    (pure VO: `title`, `text`, `ongoing`, `showStopAction`), `ForegroundServiceHost`
    (`startForeground`/`updateNotification`/`stopForeground`), `ControllerStateStore`
    (`var desiredRunning: Boolean`, `var lastStartReason: StartReason?`) and
    `BatteryOptimizationClient` (`isExempt: Flow<Boolean>`, `requestExemption()`, `openSettings()`).
  - `BrightnessBackgroundService/` — `BrightnessBackgroundService` (the only boundary F6/F7 use:
    `start(reason): StartOutcome`, `stop()`, `state: StateFlow<BackgroundServiceState>`,
    `batteryOptimizationExempt: Flow<Boolean>`).
- `:core` gained **no** `android.`/`androidx.datastore` import (the `ArchitectureStructureTest`
  guard passes); the platform `Service`, the notification factory, the DataStore-backed store, the
  boot receiver and the battery-optimization adapter are T4/T5 `:infrastructure` work.
- `BackgroundServiceState.Running` reuses F2's `BrightnessValue` (`@JvmInline value class`,
  `MINIMUM = 1`); no F2 change was needed (`AutomaticBrightnessController.snapshot` already exists).
- Files added:
  `core/src/main/kotlin/com/ambientbrightness/core/BackgroundService/BackgroundServiceState.kt`,
  `.../core/BackgroundService/StartReason.kt`, `.../core/BackgroundService/StartOutcome.kt`,
  `.../core/BackgroundService/BackgroundNotificationContent.kt`,
  `.../core/BackgroundService/ForegroundServiceHost.kt`,
  `.../core/BackgroundService/ControllerStateStore.kt`,
  `.../core/BackgroundService/BatteryOptimizationClient.kt`,
  `core/src/main/kotlin/com/ambientbrightness/core/BrightnessBackgroundService/BrightnessBackgroundService.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/BackgroundService/BackgroundServiceStateTest.kt`.
- Verification (from the repo root):
  `$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot'` then
  `.\gradlew.bat :core:test --no-daemon --console=plain`
  → **BUILD SUCCESSFUL**, 28 test classes / 181 tests, 0 failures, 0 errors, 0 skipped
  (`BackgroundServiceStateTest` adds 10 tests).
