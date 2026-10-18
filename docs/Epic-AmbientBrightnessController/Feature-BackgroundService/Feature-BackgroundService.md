# F5 — Background Service

| Field | Value |
|---|---|
| **Epic** | [EPIC: Ambient Brightness Controller](../Epic-AmbientBrightnessController.md) |
| **Depends on** | F1 (Ambient Light Sensing), F2 (Automatic Brightness Control) |
| **Depended on by** | F6 (Permissions & Setup Flow), F7 (Status Dashboard) |

## Overview

This feature is what makes the app actually useful: it keeps the sensing and brightness-control loop
alive while the user is doing something else, and defends it against Android's and OEMs' process
reclamation. It owns the foreground service, the persistent notification, the battery-optimization
exemption, and the start/stop/restart lifecycle of the controller.

## User Story 1 — Keep running while I use my phone normally

As a user, I want the controller to keep running while I use other apps or the screen is on, so that
brightness keeps adapting without me having to open the app.

#### Acceptance Criteria

- Given I have started the controller, when it is running, then an Android foreground service is
  active and shows a persistent notification for the whole time.
- Given the controller is running, when I swipe the app away from the recents list, then the service
  keeps running and brightness control continues uninterrupted.
- Given the controller is running, when the system is under memory pressure, then the foreground
  service is not selected for termination ahead of ordinary cached processes.
- Given the persistent notification is tapped, when it is tapped, then the app opens on the status
  dashboard.

## User Story 2 — Survive system and manufacturer termination

As a user, I want the controller to come back if the system kills it, so that brightness control does
not silently stop working without me noticing.

#### Acceptance Criteria

- Given the service has been terminated by the system, when the service is restarted, then the
  controller resumes with the settings (offset, response speed) that were in effect before.
- Given the device's battery optimization is active for this app, when I open the app, then it detects
  this and guides me to grant the battery-optimization exemption.
- Given the controller was running when the device was shut down or restarted, when the device has
  finished booting, then the controller starts again automatically and resumes with the stored
  settings.
- Given the controller was stopped when the device was shut down or restarted, when the device has
  finished booting, then the controller stays stopped and does not change the screen brightness.

## User Story 3 — A clean, reversible lifecycle

As a user, I want stopping the app to really stop it, so that I stay in control of my screen.

#### Acceptance Criteria

- Given the controller is running, when I stop it, then the foreground service stops, the notification
  is removed, and the ambient light sensor listener is unregistered.
- Given the controller is stopped, when the stop completes, then the screen brightness is restored to
  the value that was in effect before the controller was started.
- Given the controller is stopped, when I use my phone afterwards, then the app does not change
  brightness in any way.
- Given the `WRITE_SETTINGS` permission is revoked at runtime, when the next write fails, then the
  service stops attempting writes, reports the failure visibly, and does not crash.

## User Story 4 — A notification that tells me what is going on

As a user, I want the ongoing notification to show the current state and let me stop the controller,
so that I do not have to open the app just to check or turn it off.

#### Acceptance Criteria

- Given the controller is running, when the notification is shown, then it displays the current
  controller state and at least the currently applied brightness.
- Given the notification is shown, when I use its action, then I can stop the controller directly from
  the notification.
- Given the foreground service is active, when the user tries to dismiss the notification, then it
  cannot be swiped away while the service runs.

## Out of Scope

- Requesting or explaining the permissions in the UI (that is F6).
- The dashboard UI itself (that is F7).
- Running without any notification (not possible for a foreground service on the target Android
  version).
- Being started by a system broadcast other than boot, e.g. by a third-party automation app.
- Guaranteed survival: no app can be fully immune to a user-initiated force-stop.

## Resolved Decisions

- The controller restarts automatically after a device reboot **if it was running before the reboot**;
  if it was stopped, it stays stopped. There is no separate opt-in switch.
- Stopping the controller restores the brightness value that was in effect before it was started; there
  is no "keep the last value" option.
- The ongoing notification shows the state and offers a stop action only; the offset control lives on
  the app's settings screen.

## Technical Design

### Architecture & Components

Same three modules as F1/F2 (`:core` pure Kotlin, `:infrastructure` Android adapters, `:app` composition root; package root `com.ambientbrightness`). F5 owns process lifetime and consumes F1/F2 **only** through `AmbientLightService` and `AutomaticBrightnessController`.

Components to add (all `:core` unless noted):

- `BrightnessBackgroundService` (boundary) — single public entry point for F5: `start()`, `stop()`, `state: StateFlow<BackgroundServiceState>`. F6/F7 call only this; no UI here.
- `BackgroundServiceUseCase` — orchestration: foreground start → F2 `start()`; on stop: F2 `stop()` → restore snapshot → foreground stop → clear the persisted flag.
- `BackgroundServiceState` (sealed: `Stopped`, `Starting`, `Running(appliedBrightness)`, `StoppedBySystem`, `Failed(reason)`) — the surface F6/F7 read and the notification renders.
- `StartReason` (`UserRequest`, `SystemRestart`, `Boot`) and `StartOutcome` (`Started`/`Blocked(reason)`) — explicit reasons instead of a bare boolean.
- `ForegroundServiceHost` (interface) — `startForeground(content)`, `updateNotification(content)`,
  `stopForeground()`; keeps `:core` Android-free and lets tests assert lifecycle calls.
- `BackgroundNotificationContent` (VO) + `BackgroundNotificationBuilder` — pure, unit-testable rendering of `BackgroundServiceState`.
- `ControllerStateStore` (interface) — persists `desiredRunning`/start reason across process death and reboot; read by the boot receiver.
- `BatteryOptimizationClient` (interface) — `isExempt: Flow<Boolean>`, `requestExemption()`, `openSettings()`; exposed for F6/F7, no UI here.
- `StopBrightnessRestorer` — Task class: writes F2's `BrightnessSnapshot` back via F2's `SystemBrightnessClient` (no parallel mechanism).

`:infrastructure`: `ForegroundBrightnessService` (the only Android `Service`, implements
`ForegroundServiceHost`), `BootCompletedReceiver`, `AndroidControllerStateStore`,
`AndroidBatteryOptimizationClient`, `AndroidBrightnessNotificationFactory`, plus `Dummy*` variants.
`:app`: `BrightnessServiceCollector` (thin forwarder; no business logic) + manual DI wiring.

### Data Flow

Start (`UserRequest`): F7 → `BrightnessBackgroundService.start()` → `desiredRunning = true` → `ForegroundServiceHost.startForeground()` (inside the platform's foreground window) → `AutomaticBrightnessController.start()` (F2, driving F1) → `Starting → Running`; each F2 `BrightnessStatus` update refreshes the notification.

Stop: `stop()` → `AutomaticBrightnessController.stop()` (F2 stops writing; F1 unregisters the sensor listener) → `StopBrightnessRestorer` writes F2's `BrightnessSnapshot` via `SystemBrightnessClient` → `stopForeground()` → `desiredRunning = false` → `Stopped`.

Restart after system kill (`START_STICKY`) or boot: re-read `ControllerStateStore`, call `AutomaticBrightnessController.start()`, resume with persisted offset/response settings → `Running`; boot starts only when `desiredRunning` was `true`.

### Key Interfaces & Classes

```kotlin
interface BrightnessBackgroundService {
    fun start(reason: StartReason): StartOutcome
    fun stop()
    val state: StateFlow<BackgroundServiceState>
    val batteryOptimizationExempt: Flow<Boolean>   // surface for F6/F7
}
interface ForegroundServiceHost {
    fun startForeground(content: BackgroundNotificationContent)
    fun updateNotification(content: BackgroundNotificationContent)
    fun stopForeground()
}
interface ControllerStateStore { var desiredRunning: Boolean; var lastStartReason: StartReason? }
interface BatteryOptimizationClient { val isExempt: Flow<Boolean>; fun requestExemption(); fun openSettings() }
```

- **Notification content:** `BackgroundNotificationBuilder` maps `Running(brightness)` to "Running — brightness N" plus a single **Stop** action; `Stopped/Failed` get their own text. No offset control (Resolved Decisions); pure Kotlin, unit-testable.
- **State reuse:** `appliedBrightness` comes from F2's `BrightnessStatus`; the notification never re-reads the system setting, so it cannot disagree with the controller.
- **Restoration source:** F2's `BrightnessSnapshot` is the pre-controller value. If `AutomaticBrightnessController` lacks a read accessor, add one property there (small F2 follow-up) — do **not** build a second snapshot. *Verify accessor name with F2.*
- **Idempotency:** `start()` while `Running` is a no-op; `stop()` while `Stopped` is a no-op. All transitions run through `BackgroundServiceUseCase` on one serialising scope, so UI/notification/boot calls cannot interleave.

### State & Persistence

`ControllerStateStore` holds two facts: `desiredRunning` and `lastStartReason` (SharedPreferences/DataStore in `:infrastructure`). This makes "was running before the reboot" survivable without persisting controller state; offset/response settings stay owned by F3/F4. Transient state (`BackgroundServiceState`, applied brightness) is in-memory and rebuilt on restart. Force-stop clears the process and the platform blocks restarts until the app is opened again — `desiredRunning` stays `true` but nothing runs; F6/F7 may reconcile that on the next app open.

### Android Platform Specifics

- **Foreground service type (API 36):** declare
  `android:foregroundServiceType="specialUse"` on the service, plus the subtype property
  `<property android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE" android:value="ambient light based screen brightness control"/>`.
  `specialUse` is chosen because the loop is a continuous sensor-driven system-settings writer that
  matches no other type (`dataSync` is for transfers, `connectedDevice` needs an external device,
  `health`/`location` are unrelated and carry stricter runtime checks/preconditions). **Rejected:**
  `dataSync` (semantically wrong; on API 35+ it is capped at 6 hours per 24 hours and a boot receiver
  may not start it at all — see below), and a plain background `Service` (killed by the system).
  **Confirmed against the Android 16 docs:** the subtype property name
  `android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE` is correct and its value is free-form, the matching
  constant for `startForeground()` is `ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE`, and
  `specialUse` has **no runtime prerequisites**. Documented external step: the declared use cases are
  reviewed in the Google Play Console when the app is submitted, so the property value must be
  specific enough to show why no other type fits.
- **Manifest permissions:** `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_SPECIAL_USE` (confirmed: this is the documented type-specific permission for `specialUse`), `POST_NOTIFICATIONS`, `RECEIVE_BOOT_COMPLETED`. Do **not** add `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` by default (see risks).
- **Service declaration:** `android:exported="false"`, started via `ContextCompat.startForegroundService(...)`, with `ServiceCompat.startForeground(..., type)` called promptly in `onStartCommand` (short platform window). `START_STICKY` restart contract; handle the `null` intent after a system restart by falling back to stored state.
- **`POST_NOTIFICATIONS` denied (confirmed for API 33+):** the app does **not** need `POST_NOTIFICATIONS`
  to launch the FGS, but it must still post a notification when it does. With the permission denied the
  FGS notice is **not** visible in the notification drawer — so the Stop action is unreachable there —
  while the system Task Manager still lists the running foreground service. F5 must not crash, must
  treat a Task Manager kill as a stop, and must expose state via `BackgroundServiceState` for F6/F7.
- **Non-dismissible notification:** `setOngoing(true)` + no `setAutoCancel`, which prevents
  swipe-away for an active foreground service. The user can still terminate the service from the
  system UI (foreground-service Task Manager) — that must be treated as a stop, not as a bug.
- **Boot restart (confirmed):** `BootCompletedReceiver` with an `ACTION_BOOT_COMPLETED` intent filter
  and `RECEIVE_BOOT_COMPLETED`; it must return quickly and call `startForegroundService(...)` directly.
  Boot is an allowed exemption from background-start restrictions, but for apps targeting API 35+ a
  boot receiver may **not** launch `dataSync`, `camera`, `mediaPlayback`, `phoneCall`,
  `mediaProjection` or `microphone` (they throw `ForegroundServiceStartNotAllowedException`);
  `specialUse` is not on that list, so this design is permitted. Use `ACTION_BOOT_COMPLETED` and **not**
  `ACTION_LOCKED_BOOT_COMPLETED`, because the receiver reads the DataStore-backed
  `ControllerStateStore`, which is unavailable before the first user unlock. Test with
  `adb shell am compat enable FGS_BOOT_COMPLETED_RESTRICTIONS <pkg>` and
  `adb shell am broadcast -a android.intent.action.BOOT_COMPLETED <pkg>`.
- **Battery optimization:** detect with `PowerManager.isIgnoringBatteryOptimizations(packageName)`.
  Prefer `ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS` (guides the user to the system screen); the
  direct `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` dialog requires the
  `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` permission and has Play Store policy implications (risk).
- **Notification channel:** one low-importance channel (no sound/vibration); updates reuse the same notification id. No wakelock is held (Epic constraint).

### Edge Cases & Failure Handling

- Stop while a write is in flight → serialise through the use-case scope; restore runs after F2 has
  fully stopped, so the snapshot is not overwritten.
- `WRITE_SETTINGS` revoked at runtime → F2 already returns `Failed`; F5 maps it to `Failed(reason)` and keeps the service alive so the user can stop it explicitly; no crash (AC3.4).
- Sensor unavailable at restart → the service still runs foreground and surfaces F2/F1 availability
  through state; it does not crash on restart.
- Double start/stop, stop-from-notification racing stop-from-F7 → idempotent transitions; a second `stop()` after `Stopped` performs no restore (guard flag), so a fresh user-set brightness is never overwritten.
- Boot with `desiredRunning = false` → receiver does nothing; no service, no brightness write (AC2.4).
- User force-stop → not survivable by design; documented, not worked around.

### Test Strategy

`:core` (JVM): `BackgroundServiceUseCase` is driven by fakes of `ForegroundServiceHost`, `ControllerStateStore`, `BatteryOptimizationClient` and F2's `AutomaticBrightnessController`, asserting start order (foreground → controller), stop order (controller → restore → foreground), idempotency, and that nothing happens on a redundant stop. `BackgroundNotificationBuilder` is unit-tested for every `BackgroundServiceState` value and the stop action. `:infrastructure`: instrumented test starting `ForegroundBrightnessService` (service is foreground, notification ongoing, stop action cancels it) and one `BootCompletedReceiver` test with a stubbed `ControllerStateStore`. `Dummy*` adapters keep `:app` runnable without a device. No Compose UI tests (F6/F7 own UI).

## Task Breakdown

- [x] [T1 — Domain model, state and boundary interfaces](./Tasks/Task-01-DomainModelStateAndBoundaryInterfaces.md)
- [x] [T2 — Notification content and state mapping](./Tasks/Task-02-NotificationContentAndStateMapping.md)
- [x] [T3 — Lifecycle orchestration and brightness restoration](./Tasks/Task-03-LifecycleOrchestrationAndBrightnessRestoration.md)
- [x] [T4 — Android foreground service, notification and manifest](./Tasks/Task-04-AndroidForegroundServiceNotificationAndManifest.md)
- [x] [T5 — Persistence, battery-optimization client and boot restart](./Tasks/Task-05-PersistenceBatteryOptimizationClientAndBootRestart.md)
- [x] [T6 — Composition root, state collector and restart-after-kill hardening](./Tasks/Task-06-CompositionRootStateCollectorAndRestartAfterKillHardening.md)

## Suggested Implementation Order

`T1 → T2 → T3` are pure `:core` and can largely proceed in parallel after T1 fixes the contracts.
`T4` (Android service + manifest) and `T5` (persistence/boot) are `:infrastructure` and can run in
parallel with each other once T1/T2 exist. `T6` is last because it is the only place that connects
the F2 controller, the Android service, the receiver and the F6/F7 state surface. T3's snapshot
restore has one external dependency (F2's `BrightnessSnapshot` accessor) — unblock it first.
