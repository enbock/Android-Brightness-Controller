# T6 — Resume reconciliation, error/clamp integration and DI

**Completion:** Not completed — reported bug follow-up required

## Dependencies

See the [Suggested Implementation Order](../Feature-StatusDashboard.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T6 — Resume reconciliation, error/clamp integration and DI**: add an `ON_RESUME`
  (`LifecycleResumeEffect`) hook that re-reads `BrightnessBackgroundService.state` and calls
  `AppSetupService.refresh()`; ensure `StoppedBySystem`, `Failed` and clamped states render through the
  same `DashboardUiState` path; wire manual DI in `:app` (app-scoped `StatusDashboard` singletons, real
  vs `Dummy*` adapters). *Files: `:app/.../StatusDashboard/**`, `:app/.../di/*`,
  `:app/.../MainActivity.kt`.* *Verify: fake/instrumented end-to-end — externally stopped service shows
  `StoppedBySystem` on return (US3 AC3), a simulated write failure flips to the error state (US3 AC2),
  and the app renders with `Dummy*` adapters on an emulator.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `d079363f7026bee125ac1522569389bb120aa247`.*

- **Resume reconciliation (US2 AC2.4, US3 AC3.3).** Added `View/Task/ResumeReconciler.kt` — a named,
  unit-testable unit whose `onResume()` calls `AppSetupService.refresh()`. It exists because only F6 is
  *derived*: F5's lifecycle state and the F3/F4 settings are `StateFlow`s the dashboard observes
  directly, so their current value is already re-read by construction and nothing is cached or polled,
  while F6 has to re-check grants the user may have changed in a system screen. `View/DashboardResumeEffect.kt`
  wraps it in `LifecycleResumeEffect`, `DashboardNavHost` installs it for the whole graph, and
  `MainActivity` supplies the reconciler — so a resume never shows the state from before.
- **`StoppedBySystem` became reachable (US3 AC3.3).** F5 already modelled the state and T1's mapper
  already rendered it, but nothing produced it, so the criterion could not be met end to end. Added an
  additive, documented `BackgroundServiceUseCase.recordStoppedBySystem()` that shares one private
  teardown with `stop()` — the F5 contract treats a Task-Manager kill as a stop, so the controller is
  stopped, the snapshot restored, the foreground service torn down and the persisted flag cleared — but
  ends in `StoppedBySystem`, keeping a system stop distinguishable from a deliberate one. It is a no-op
  when nothing was running (then the honest state already is `Stopped`), and a later user `stop()` cannot
  overwrite the reason. `BrightnessBackgroundService` is unchanged (no interface widening).
- **The platform signal.** `ForegroundServiceHostBridge` gained a `systemStopHandler`, invoked from
  `onServiceDisconnected` — the one callback that fires when the platform loses the service without the
  app stopping it. `AmbientBrightnessApplication` wires it to `component.recordSystemStop()`, and
  `AmbientBrightnessComponent` now keeps the concrete use case in a field (no cast) and exposes that
  internal forwarding method.
- **DI.** The app-scoped graph is unchanged and now complete: `Composition` owns the collector, both
  ViewModels and the resume reconciler, so no screen or activity touches an adapter; Dummy adapters keep
  every JVM test device-free.
- **Tests.** `:core` `BackgroundServiceUseCaseTest` gained 3 cases (a platform stop reports
  `StoppedBySystem` and still tears the lifecycle down, restores the pre-controller brightness and clears
  the flag; a platform stop without a start changes nothing; a later user stop neither tears down again
  nor overwrites the reason). `:app` JVM `ResumeReconcilerTest` (2) proves a resume re-derives the setup
  state and re-derives it on **every** resume. `:app` JVM `DashboardEndToEndTest` (3) runs the real
  composition root with Dummy adapters and a `TestClock`: US3 AC2 — a failing brightness write surfaces
  as `BackgroundServiceState.Failed` and the dashboard shows `ControllerUiState.Failed` with no live
  reading; US3 AC3 — after `recordSystemStop()` the dashboard shows `ControllerUiState.StoppedBySystem`
  instead of the last known running state (asserted to have been `Running` first); and a deliberate
  `stop()` still reads as `Stopped`. `:app` androidTest `AmbientBrightnessAppSmokeTest` (1,
  `createAndroidComposeRule<MainActivity>`) asserts the app boots straight into the dashboard with all
  three panels — **compile-verified only** here (no device attached).
- Verification (from the repo root):
  `$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot'` then
  `.\gradlew.bat :core:test :infrastructure:test :app:testDebugUnitTest :app:assembleDebug
  :app:assembleDebugAndroidTest :infrastructure:assembleDebugAndroidTest --no-daemon --console=plain`
  → **BUILD SUCCESSFUL**, 0 failures; `:core` 33 classes / 240 tests, `:infrastructure` 13 / 48,
  `:app` 16 / 67 (355 tests total).
- **Original implementation status:** at the time these verification notes were recorded, T1–T6
  had been marked complete. The current reported-bug follow-up below marks T2 and T6 incomplete.
  AC1.1–AC1.4 are covered by T1/T2/T4's tests, AC2.1–AC2.3 by T2/T4, AC3.1–AC3.3 by T1/T2/T4/T6, and
  AC4.1–AC4.4 by T3/T5/T6. Not verifiable without a device (stated, not worked around): the real
  foreground-service lifecycle, the real system-initiated kill and the physical sensor — the
  instrumented tests for those compile here and are the addition, while the JVM tests carry the proof.
- **Cross-feature note:** `recordStoppedBySystem()` is an additive F5 API. F5's own design anticipated
  exactly this hook ("expose a documented way to record it … only if it does not widen the
  `BrightnessBackgroundService` interface"); without it F7's AC3.3 could not be satisfied end to end.

## Follow-up — separate service reconciliation from UI resume

- [ ] Keep the ON_RESUME refresh for F6, and additionally ensure F7 observes authoritative F5
  lifecycle state after F5 T6 performs process-recreation recovery. Do not treat a newly constructed
  StateFlow's default `Stopped` as proof that a persisted-running service was stopped.
- [ ] Add coverage that distinguishes (a) Android service/controller recovery, (b) dashboard state
  reconciliation, and (c) general Activity/UI responsiveness; report the actual stopped/error state
  if recovery fails rather than a cached running value.
- **Dependencies:** F5 T6 process-recreation recovery; F7 T2 shared live-lux stream.
- **Size:** S.
- **Verification:** after process recreation, open/resume the dashboard and assert `desiredRunning`
  true produces live Running/applied-brightness/lux state only after the controller is active; with
  the flag false show Stopped with no live lux. Independently assert Activity renders/responds. Test
  state collection with fakes and run the process-kill scenario on a device/emulator.
