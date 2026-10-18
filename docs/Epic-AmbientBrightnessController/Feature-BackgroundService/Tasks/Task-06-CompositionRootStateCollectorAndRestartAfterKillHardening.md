# T6 — Composition root, state collector and restart-after-kill hardening

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-BackgroundService.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T6 — Composition root, state collector and restart-after-kill hardening**: wire everything in
  `:app` (manual DI, app-scoped singletons, `BrightnessServiceCollector` forwarding
  `BackgroundServiceState` and `batteryOptimizationExempt` to F6/F7), bind
  `AutomaticBrightnessController` start/stop, and verify `START_STICKY` restart resumes with stored
  settings. *Files: `:app/src/main/kotlin/com/ambientbrightness/app/**`.*
  *Verify: end-to-end fake/instrumented run — start, kill the process, restart resumes `Running` with
  persisted settings (AC2.1); swipe-from-recents keeps it running (AC1.2).*
  *Depends on: T3, T4, T5.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `bc2008bdc268ddb36a57e1a1220da3f424414d14`.*

- **F5 is now complete (T1–T6).** Wired the whole feature into the `:app` composition root; no
  business logic moved into `:app`, and neither `:core` nor `:infrastructure` changed for T6.
- Added `BrightnessServiceCollector` (`:app/.../BackgroundService/Collector/`, folder == package like
  the other collectors): a thin, logic-free forwarder of `BrightnessBackgroundService.state` and
  `batteryOptimizationExempt`, mirroring `AmbientLightCollector`/`BrightnessStatusCollector`.
- `AmbientBrightnessComponent` builds one app-scoped `BackgroundServiceUseCase` from the existing F2
  `brightnessController`, a `ForegroundServiceHost`, a `ControllerStateStore`, a
  `BatteryOptimizationClient`, `StopBrightnessRestorer(brightnessClient)`,
  `BackgroundNotificationBuilder()` and `controllerScope`, and exposes it as
  `val backgroundService: BrightnessBackgroundService` plus `backgroundServiceCollector`. The use case
  drives the same controller the activity's `statusCollector` observes, so there is exactly one
  control loop, and the restore path writes through the same `brightnessClient` (no parallel
  brightness mechanism, AC3.2). New constructor parameters default to `DummyForegroundServiceHost`,
  `DummyControllerStateStore` and `DummyBatteryOptimizationClient`, so JVM tests inject deterministic
  adapters exactly like the existing client parameters.
- **Platform host bridge:** `ForegroundServiceHostBridge` (`:app/Composition/`, the only place allowed
  to import `:infrastructure`). The Android `Service` is created by the platform, so the bridge (1)
  brings it up with `ContextCompat.startForegroundService` (respecting the foreground window before
  any binding), (2) binds it with `bindService`/`BIND_AUTO_CREATE` and obtains the instance through
  `ForegroundBrightnessService.LocalBinder`, and (3) delegates `startForeground`/`updateNotification`/
  `stopForeground` to that instance, queueing the latest `BackgroundNotificationContent` until
  `onServiceConnected` fires (binding is asynchronous, the lifecycle calls are synchronous). It
  forwards the service's `stopRequestListener` to a `stopRequestHandler`, which
  `AmbientBrightnessApplication` wires to `component.backgroundService::stop`, so the notification's
  Stop action travels the same path as a stop from F7 (F2 stop → snapshot restore → foreground stop).
  `AmbientBrightnessApplication` also passes `AndroidControllerStateStore` and
  `AndroidBatteryOptimizationClient.create`. `MainActivity` and the existing collectors are unchanged
  and still compile; starting/stopping from the UI stays F6/F7's job.
- Import rule preserved: `git grep -n "com\.ambientbrightness\.infrastructure\." -- app/src/main/kotlin`
  matches only `Composition/AmbientBrightnessComponent.kt`, `Composition/AmbientBrightnessApplication.kt`
  and `Composition/ForegroundServiceHostBridge.kt`.
- Tests (`:app`, JUnit 4 + `kotlinx-coroutines-test`, `runTest`/`backgroundScope`/`TestClock`, the same
  style as `AmbientBrightnessResponseTuningTest`), driving the real component with dummy adapters and a
  recording `SystemBrightnessClient`:
  - **AC1.1** — `start(UserRequest)` promotes the host, persists `desiredRunning`/`lastStartReason`,
    reaches `BackgroundServiceState.Running`, and the state carries the brightness F2 applied last.
  - **AC2.1** — a **simulated restart** (a fresh component over the same persisted settings store and
    controller state, `start(SystemRestart)`) resumes `Running` with the stored response speed and the
    same applied brightness as before the simulated kill.
  - **AC3.1 / AC3.2 / AC3.3** — `stop()` returns to `Stopped`, clears the foreground service and
    `desiredRunning`, and restores the pre-controller brightness through F2's client.
- **Compile-verified only (needs a device):** the real process-kill/`START_STICKY` restart and
  swipe-from-recents (AC1.2) cannot be reproduced on a plain JVM — they require the Android platform to
  reclaim and restart the process, and the recents swipe is an ActivityManager behaviour. The JVM test
  therefore simulates the restart (above) and the bridge's real delegation stays compile-verified; the
  on-device parts are covered by the `:app` instrumented `ForegroundBrightnessServiceTest`
  and need a device/emulator run.
- Verification (from the repo root):
  `$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot'` then
  `.\gradlew.bat :core:test :infrastructure:test :app:testDebugUnitTest :app:assembleDebug :app:assembleDebugAndroidTest --no-daemon --console=plain`
  → **BUILD SUCCESSFUL** (98 actionable tasks); `:core` 31 classes / 201 tests, `:infrastructure`
  12 classes / 45 tests, `:app` 3 classes / 8 tests — 0 failures, 0 errors, 0 skipped
  (`AmbientBrightnessBackgroundServiceTest` adds 3 tests); `:app:assembleDebug` and
  `:app:assembleDebugAndroidTest` compile/link.
- **AC coverage map:** AC1.1 → `AmbientBrightnessBackgroundServiceTest: AC1_1…`; AC2.1 →
  `…AC2_1…` (AC2.2/AC2.3/AC2.4 by the T5 `:infrastructure` tests); AC3.1/AC3.2/AC3.3 →
  `…AC3_1 AC3_2 AC3_3…`; AC3.4 → T3 `BackgroundServiceUseCaseTest`; AC4.1–AC4.3 → T2
  `BackgroundNotificationBuilderTest` + T4 instrumented test; AC1.2 (swipe-from-recents) and a real
  process kill (AC2.1 on device) are compile-verified only (device required, see above).
