# T5 — Persistence, battery-optimization client and boot restart

**Completion:** Not completed — reported bug follow-up required

## Dependencies

See the [Suggested Implementation Order](../Feature-BackgroundService.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T5 — Persistence, battery-optimization client and boot restart**: implement
  `AndroidControllerStateStore`, `AndroidBatteryOptimizationClient` (+ `Dummy*`), `BootCompletedReceiver`,
  `RECEIVE_BOOT_COMPLETED` and the `ACTION_BOOT_COMPLETED` filter; receiver starts the service only
  when `desiredRunning` is set. *Files: `:infrastructure/.../ControllerStateStore/**`,
  `.../BatteryOptimizationClient/**`, `.../BootCompletedReceiver/**`, `:app/src/main/AndroidManifest.xml`.*
  *Verify: receiver test with `desiredRunning` true → start, false → no-op (AC2.3, AC2.4); exemption
  state reads correctly and is exposed (AC2.2).*
  *Depends on: T1, T4.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `fb10bc2ba0150c5b3094f142c37a0274ee11c02b`.*

- Added `AndroidControllerStateStore` (`:infrastructure/.../BackgroundService/ControllerStateStore/Android/`)
  as the persistent `ControllerStateStore`.
  - **Storage choice: `SharedPreferences`, not DataStore.** The T1 contract declares `desiredRunning`
    and `lastStartReason` as *synchronous* `var`s, and the boot receiver (AC2.3/AC2.4) must read
    `desiredRunning` inside `onReceive` without suspending — DataStore only exposes a
    suspending/flow API, so a DataStore store would have forced a `runBlocking` in the receiver. One
    `SharedPreferences` file (`background_service_state`) serves reads from the in-memory snapshot the
    framework loads on first access; writes use `Editor.apply()`.
  - `lastStartReason` is persisted by `StartReason.name`; an unknown or absent name reads back as
    `null` via the `internal fun startReasonOrNull(name)` helper (unit-testable, no Android), so a
    renamed/removed enum value can never crash boot.
- Added `DummyControllerStateStore` (`ControllerStateStore/Dummy/`, in-memory) so `:app` (T6) can be
  composed without persistence.
- Added `AndroidBatteryOptimizationClient` (`.../BatteryOptimizationClient/Android/`).
  - **Documented choice:** Android has no change callback for the exemption, so `isExempt` is a
    `MutableStateFlow` (exposed as `StateFlow<Boolean>`, a `Flow` subtype) that is re-read from
    `PowerManager.isIgnoringBatteryOptimizations(packageName)` in `refresh()` — at construction, before
    every `requestExemption()`/`openSettings()` launch, and on demand for F6/F7 when the user returns
    from the system screen. No coroutine scope is needed because the flow is never `stateIn`-backed.
  - Both entry points open the advisory `Settings.ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS` list
    (with `FLAG_ACTIVITY_NEW_TASK`); the app-specific `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`
    dialog is deliberately **not** used and no `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` permission was
    added (Play policy implications, per *Android Platform Specifics*).
  - Construction is seam-based (`exemptionReader: () -> Boolean`, `settingsScreenOpener: (String) ->
    Unit`) with a `companion fun create(context)` for real use, so the client is unit-testable without a
    device or a `Context`.
- Added `DummyBatteryOptimizationClient` (`BatteryOptimizationClient/Dummy/`): in-memory exemption plus
  call counters for the two launch entry points.
- Added `BootCompletedReceiver` (`.../BootCompletedReceiver/`), registered for
  `Intent.ACTION_BOOT_COMPLETED` **only** (never `ACTION_LOCKED_BOOT_COMPLETED`, since the store lives
  on credential-protected storage that is unavailable before the first unlock).
  - **How it decides to start:** `onReceive` checks the action, then
    `handleBootCompleted(action, storeFactory(context), serviceStarter(context))` — when the action is
    `BOOT_COMPLETED` it reads the synchronous `store.desiredRunning`; only when it is `true` does it call
    `ContextCompat.startForegroundService(ForegroundBrightnessService)`. When `false` it does nothing at
    all: no service start and therefore no brightness write (AC2.4). It returns immediately (no
    suspension, no I/O beyond the in-memory SharedPreferences snapshot).
  - Kept independent of `:app`: it constructs the infrastructure store directly via an overridable
    `internal var storeFactory: (Context) -> ControllerStateStore` and an overridable
    `internal var serviceStarter: (Context) -> Unit`, and its tested logic lives in the `internal fun
    handleBootCompleted(action, store, startService)` seam (same pattern as T4/T5's lambda seams), so a
    stubbed store drives it without a `Context`.
- Manifest (`app/src/main/AndroidManifest.xml`): added `RECEIVE_BOOT_COMPLETED` (no duplicate of the
  T4 permissions) and
  `<receiver android:name="...BootCompletedReceiver.BootCompletedReceiver" android:enabled="true"
  android:exported="false">` with an `<intent-filter>` for `android.intent.action.BOOT_COMPLETED`.
  Verified in the merged manifest of `:app:assembleDebug`.
- Tests (JUnit 4, `:infrastructure/src/test/...`): `BootCompletedReceiverTest` (desiredRunning `true` →
  one start; `false` → none; a non-boot action → ignored and the store not even read — AC2.3, AC2.4);
  `AndroidBatteryOptimizationClientTest` (exemption read at construction and exposed, `refresh()`
  re-reads, both entry points open `ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS` and not
  `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` — AC2.2); `DummyBackgroundServiceAdaptersTest` and
  `AndroidControllerStateStoreTest` (enum-name round-trip incl. unknown-name → null).
- Verification (from the repo root):
  `$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot'` then
  `.\gradlew.bat :infrastructure:test :app:assembleDebug --no-daemon --console=plain`
  → **BUILD SUCCESSFUL** (61 actionable tasks); `:infrastructure` unit tests 12 classes / 45 tests,
  0 failures, 0 errors, 0 skipped (the four T5 test classes add 13 tests); `:app:assembleDebug`
  compiles/links. No `:core` change.

## Follow-up — prove boot start resumes the controller

- [ ] Extend boot-restart verification beyond asserting that the receiver calls
  `ContextCompat.startForegroundService`. That proves only that Android is asked to create the
  foreground service; `ForegroundBrightnessService.onStartCommand()` currently promotes a
  notification without invoking the F5 lifecycle or F2 controller.
- [ ] Reconcile persisted `desiredRunning` through the idempotent F5 recovery path: true must resume
  the controller with persisted settings and publish live state, while false must not start a
  controller or write brightness. Keep the synchronous receiver quick and preserve credential-storage
  availability constraints.
- **Dependencies:** F5 T4 sticky-restart handoff and T6 composition/lifecycle recovery (the existing
  persistence store and receiver implementation in this task are already present).
- **Size:** S.
- **Verification:** unit/integration test follows the boot receiver's start request through the
  recovery contract for desired-running true and false; device/emulator boot test confirms foreground
  status, controller sampling/writes and dashboard state independently. Test repeated starts for
  idempotency.
