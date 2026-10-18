# T5 — Controller orchestration, conflict state, snapshot and composition root

**Completion:** Not completed — reported bug follow-up required

## Dependencies

See the [Suggested Implementation Order](../Feature-AutomaticBrightnessControl.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T5 — Controller orchestration, conflict state, snapshot and composition root**: implement
  `AutomaticBrightnessController` + `BrightnessControllerUseCase` wiring the F1 stream through map →
  transform-hooks → clamp → dead-band → write, publish `status`,
  `adaptiveBrightnessConflict` and `BrightnessSnapshot`, and bind everything in `:app` with
  `BrightnessStatusCollector`.
  *Files: `:core/.../AutomaticBrightnessControl/BrightnessControllerUseCase/**`,
  `:app/src/main/kotlin/com/ambientbrightness/app/**`.*
  *Verify: end-to-end fake-driven test (AC1.1, AC1.3, AC1.4, AC4.1–4.3); `stop()` halts writes.*
  Note: the use case cannot live in a folder/package named `AutomaticBrightnessController`, because
  that package would collide with the interface classifier of the same name. The DDD rule "one
  directory per use case" therefore names the folder `BrightnessControllerUseCase`, matching its
  class.

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `fb6b0b239c4b9b9ef0c191d36b67ead738f6a14d`.*

- Added `BrightnessControllerUseCase` (`:core`, public) implementing `AutomaticBrightnessController`.
  `start()` is idempotent, captures the restoration snapshot from `SystemBrightnessClient.readCurrent()`,
  resolves the device range through the existing `BrightnessRangeResolver`, resets the dead band and
  seeds it with the current system value, clears the failure latch, publishes conflict state and
  launches the control loop in the injected scope. `stop()` is idempotent: it cancels the loop,
  releases the F1 stream through `AmbientLightService.stop()` and marks the status stopped.
- The control loop collects `ScreenStateClient.isScreenOn` with `collectLatest`, so the single F1
  ambient-light collection runs only while the display is on and is released again on screen-off,
  and it observes `WriteSettingsPermissionChecker.hasWriteSettingsPermission` in a sibling coroutine
  that clears the failure latch when permission is (re-)granted (AC1.4, AC2.4).
- Write path: samples update `status.lastLux`; the accepted reading is computed once and remembered;
  the write is skipped while the failure latch is set or the permission is false; a request inside
  the minimum step is suppressed through `MinimumStepDeadBand` and only updates
  `status.writeState`; `Applied` records the applied value in the dead band and clears `writeError`;
  `Failed` stores `status.writeState`/`status.writeError` and latches the failure so no further write
  happens until the next `start()` or a re-granted permission (AC1.1, AC1.2, AC1.3, AC2.4).
- Added the internal `BrightnessPipeline` task collaborator (hysteresis gate → mapping curve →
  registered `BrightnessTransform` hooks → `BrightnessRangeScaler`) and `BrightnessComputation`.
  A NaN result from a faulty hook falls back to the plain curve value, so a broken hook can neither
  crash the loop nor produce an invalid value. `registerTransform()` plus `reapply()` recompute the
  last accepted reading without consulting the gate, which is the hook F3 (offset) and F4 (ramp) are
  designed against; `reapply()` is a no-op before the first accepted reading or while stopped.
- Added the internal `AdaptiveBrightnessConflictDetector`, which reports the built-in mode and
  degrades a throwing platform client to `UNKNOWN` instead of failing the controller (AC4.1–4.3).
- Added the read-only core boundary `BuiltInAdaptiveBrightnessClient` (F2 never changes the mode) with
  `AndroidBuiltInAdaptiveBrightnessClient`, which reads only `SCREEN_BRIGHTNESS_MODE` and maps
  `AUTOMATIC`→`ENABLED`, `MANUAL`→`DISABLED`, anything else (including a missing setting) → `UNKNOWN`,
  and a settable `DummyBuiltInAdaptiveBrightnessClient`.
- Added `status` and `adaptiveBrightnessConflict` accessors, plus the `snapshot` accessor required by
  the F5 restore-on-stop design. This is the small F2 follow-up F5's doc asks for; the controller
  itself never restores brightness.
- `:app` composition root: `AmbientBrightnessComponent` now takes the app scope and the four
  brightness clients (defaulting to the deterministic `Dummy*` adapters), exposes
  `controller: AutomaticBrightnessController` and `statusCollector: BrightnessStatusCollector`, and
  documents that the activity never starts the controller because F5 owns the loop lifetime and F1's
  `AmbientLightCollector` already consumes the single ambient-light stream.
  `AmbientBrightnessApplication` wires one app scope into the screen-state client, the controller and
  the permission checker, and passes the real Android brightness, range and built-in-mode adapters.
- Files added:
  `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BrightnessControllerUseCase/BrightnessControllerUseCase.kt`,
  `.../BrightnessControllerUseCase/Task/BrightnessPipeline.kt`,
  `.../BrightnessControllerUseCase/Task/AdaptiveBrightnessConflictDetector.kt`,
  `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BuiltInAdaptiveBrightnessClient.kt`,
  `infrastructure/src/main/kotlin/com/ambientbrightness/infrastructure/AutomaticBrightnessControl/BuiltInAdaptiveBrightnessClient/Android/AndroidBuiltInAdaptiveBrightnessClient.kt`,
  `.../BuiltInAdaptiveBrightnessClient/Dummy/DummyBuiltInAdaptiveBrightnessClient.kt`,
  `app/src/main/kotlin/com/ambientbrightness/app/AutomaticBrightnessControl/Collector/BrightnessStatusCollector.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BrightnessControllerUseCase/BrightnessControllerUseCaseTest.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/AdaptiveBrightnessConflictDetector/AdaptiveBrightnessConflictDetectorTest.kt`.
  Files changed: `AutomaticBrightnessController.kt` (snapshot accessor),
  `MinimumStepDeadBand/MinimumStepDeadBand.kt` (reset), `BrightnessContractFakes.kt`
  (recording/failure/dead-band fakes, `FakeAmbientLightService`, `FakeScreenStateClient`,
  `FakeBuiltInAdaptiveBrightnessClient`), `MinimumStepDeadBandTest.kt`,
  `DummyBrightnessAdaptersTest.kt`, both app androidTest classes.
- Deviation from the original task text: the use case folder is `BrightnessControllerUseCase/` and
  not `AutomaticBrightnessController/`, because a package with the latter name cannot coexist with
  the interface classifier `com.ambientbrightness.core.AutomaticBrightnessControl.AutomaticBrightnessController`
  (Kotlin reports "Package conflicts with classifier"). `ScreenStateClient` is a constructor
  dependency of the use case rather than of the published interface, so F5 can swap the loop owner
  without changing the `AutomaticBrightnessController` contract.
- Verification passed: `:core:test --no-daemon` (102 tests, 0 failures, including the folder/package
  guard), `:infrastructure:test --no-daemon`, `:app:assembleDebug --no-daemon` and
  `:app:assembleDebugAndroidTest --no-daemon`. The F1→F2 end-to-end test drives the real
  `DefaultAmbientLightService`/`AmbientLightUseCase` chain with the fake sensor client and asserts the
  resulting system write.
- Not verified on device: two app instrumented tests were added
  (`builtInAdaptiveBrightnessModeFollowsTheSystemSetting`, `controllerStatusAndAdaptiveBrightnessConflictAreCollectable`)
  but no emulator or device is attached, so they were only compiled. They need the usual
  `adb shell appops set com.ambientbrightness.app WRITE_SETTINGS allow` grant before they run.

## Follow-up — single ambient-light collection owner

- [ ] Replace the competing cold-flow subscriptions across the F1 Activity collector, F2 controller,
  and F7 dashboard with one explicit sensor collection owner and sample fan-out. Keep the controller
  subscribed for F5's active lifetime and let dashboard/activity subscriptions observe that shared
  stream; cancellation of an observer must not call a global stop that cancels F5's stream.
- [ ] Ensure the chosen owner still unregisters the physical sensor exactly once when the final
  consumer/lifecycle ends, and preserve screen-off collection policy without requiring a lock/unlock
  cycle to restart brightness control.
- **Dependencies:** coordinate with F7 T2, which must consume the shared sample stream; preserve F5
  ownership when changing Activity lifecycle cleanup.
- **Size:** M.
- **Verification:** fake-driven tests start controller, Activity and dashboard consumers in multiple
  orders; one sensor registration emits to both the F2 writer and F7 lux readout; first valid sample
  after Start produces a write without locking/unlocking; cancelling Activity/dashboard observation
  does not stop the active F5 controller; final stop unregisters the sensor.
