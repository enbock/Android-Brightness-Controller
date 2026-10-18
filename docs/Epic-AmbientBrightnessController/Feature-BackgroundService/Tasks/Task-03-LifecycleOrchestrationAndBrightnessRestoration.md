# T3 — Lifecycle orchestration and brightness restoration

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-BackgroundService.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T3 — Lifecycle orchestration and brightness restoration**: implement
  `BackgroundServiceUseCase` (start/stop ordering, idempotency, serialised transitions) and the
  `StopBrightnessRestorer` Task that writes F2's `BrightnessSnapshot` through `SystemBrightnessClient`.
  *Files: `:core/.../BackgroundServiceUseCase/**`, `:core/.../StopBrightnessRestorer/**`.*
  *Verify: fake-driven tests for AC3.1–3.3, double start/stop, restore ordering; AC3.4 maps F2
  `Failed` to `Failed(reason)` without throwing. Depends on F2 exposing the snapshot accessor.*
  *Depends on: T1.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `d8577879dfda48acfe0bd6952dd5fa5286374f72`.*

- Added `StopBrightnessRestorer` (pure `:core` Task taking F2's `SystemBrightnessClient`):
  `restore(snapshot: BrightnessSnapshot?)` returns `null` and writes nothing when the snapshot is
  `null`, otherwise writes `snapshot.brightness` and returns the `BrightnessWriteResult` without
  throwing on `Failed`. The `null` case is what makes a redundant stop safe (a second `stop()` must
  never clobber a brightness the user has set since), and the write goes through F2's own client, so
  there is no parallel brightness mechanism (AC3.2).
- Added `BackgroundServiceUseCase : BrightnessBackgroundService` with a `MutableStateFlow` state and
  `batteryOptimizationExempt` forwarded from `BatteryOptimizationClient.isExempt`. All transitions
  run under one `lifecycleLock` on the injected `CoroutineScope`, so calls from F7, the notification
  action and the boot receiver cannot interleave.
  - **Start (AC1.1):** `Starting` → persist `desiredRunning = true` and `lastStartReason` →
    `host.startForeground(Starting content)` → `controller.start()` → observe `controller.status` on
    the scope, mapping each update to `Running(status.appliedBrightness)` and calling
    `host.updateNotification(...)`. Start order is therefore foreground before controller, and the
    notification can never disagree with the controller because it never re-reads the system setting.
  - **Stop (AC3.1, AC3.2):** capture `controller.snapshot` → `controller.stop()` (F2 stops writing,
    F1 releases the sensor) → `restorer.restore(snapshot)` → `host.stopForeground()` →
    `desiredRunning = false` → `Stopped`; the status observation is cancelled, so nothing continues to
    write or update the notification afterwards (AC3.3).
  - **Idempotency:** a second `start()` while active and a second `stop()` while stopped are no-ops,
    and the redundant stop performs no restore (guard flag). A `stop()` that was never preceded by a
    `start()` also does nothing.
  - **AC3.4:** a failed F2 write (`BrightnessWriteResult.Failed`) maps to
    `BackgroundServiceState.Failed(reason)` on the status stream without throwing, leaving the service
    alive so the user can stop it explicitly.
- No F2 change was needed: `AutomaticBrightnessController.snapshot` already existed, and no T1/T2 or
  `BrightnessBackgroundService` signature was altered.
- Files added:
  `core/src/main/kotlin/com/ambientbrightness/core/StopBrightnessRestorer/StopBrightnessRestorer.kt`,
  `core/src/main/kotlin/com/ambientbrightness/core/BackgroundServiceUseCase/BackgroundServiceUseCase.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/StopBrightnessRestorer/StopBrightnessRestorerTest.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/BackgroundServiceUseCase/BackgroundServiceUseCaseTest.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/testdoubles/BackgroundServiceFakes.kt`.
- Tests (JUnit 5, `runTest`, `backgroundScope`): an ordered `CallLog` across the fakes proves
  `host.startForeground` precedes `controller.start`, and that on stop `controller.stop` precedes the
  snapshot restore which precedes `host.stopForeground`; AC3.2 asserts the restored value equals the
  snapshot; AC3.3 asserts no status is mirrored after stop; AC3.4 asserts the `Failed(reason)` mapping;
  double start/stop and stop-without-start idempotency; and the exemption flow forwarding.
- Verification (from the repo root):
  `$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot'` then
  `.\gradlew.bat :core:test --no-daemon --console=plain`
  → **BUILD SUCCESSFUL**, 31 test classes / 201 tests, 0 failures, 0 errors, 0 skipped
  (`StopBrightnessRestorerTest` adds 3 tests and `BackgroundServiceUseCaseTest` adds 8). `:core` gained
  **no** `android.`/`androidx.datastore` import.
