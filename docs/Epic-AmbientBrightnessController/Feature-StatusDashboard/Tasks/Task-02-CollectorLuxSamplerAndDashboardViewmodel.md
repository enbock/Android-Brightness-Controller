# T2 — Collector, lux sampler and dashboard ViewModel

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-StatusDashboard.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T2 — Collector, lux sampler and dashboard ViewModel**: implement `DashboardStateCollector`
  (implements `DashboardStateProvider`; combines the five service flows), `LuxReadoutSampler`
  (`conflate` + `distinctUntilChanged` + `sample(500 ms)`) and `DashboardViewModel` exposing
  `uiState: StateFlow<DashboardUiState>` with `start()`/`stop()` intents via `ControllerCommandHandler`;
  collect lux only while running. *Files: `.../StatusDashboard/Adapter/DashboardStateCollector.kt`,
  `.../View/Task/LuxReadoutSampler.kt`, `.../View/DashboardViewModel.kt`,
  `.../Controller/ControllerCommandHandler.kt`.* *Verify: fake-driven tests — no live value while stopped
  (US1 AC4); `start()` → `Starting` immediately, then `Running`/`Failed` from the service (US2 AC1,
  US3 AC2); sampler cadence ≤ 2 Hz.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `11f70a7750688ceea9ce78619da575d31552d265`.*

- Added `View/DashboardStateProvider.kt` — `fun snapshots(): Flow<DashboardSnapshot>`, the
  presentation seam that keeps the collector's service knowledge out of the ViewModel.
- Added `Adapter/DashboardStateCollector.kt` — reads the five boundaries and combines them into one
  `DashboardSnapshot`: F5 `state`, F6 `status` + `isReadyToStart`, F3 `offset` + `application`, F4
  `responseSpeed` and the sampled F1 reading. `combine` is nested (5-flow overload + response speed)
  because it supports at most five flows per call. It holds no rule; every presentation decision stays
  in T1's mapper.
- **Lux gating (US1 AC1.4).** `backgroundService.state.flatMapLatest { … }` collects
  `AmbientLightService.start()` only while F5 is `Starting`/`Running`; leaving those states cancels the
  inner collection (which releases F1's sensor listener) and substitutes `null`, so a stopped controller
  blanks the readout rather than showing a stale value.
- Added `View/Task/LuxReadoutSampler.kt` — `conflate()` + `distinctUntilChanged()` +
  `sample(DISPLAY_SAMPLE_INTERVAL_MS = 500L)`, capping the readout at 2 Hz (`@OptIn(FlowPreview::class)`
  for `sample`). A small `LuxSampler` fun interface is the seam the collector depends on, so collector
  tests run on a pass-through sampler with no wall-clock waiting while production always uses
  `LuxReadoutSampler`.
- Added `Controller/ControllerCommandHandler.kt` — Start/Stop intents → `BrightnessBackgroundService`.
  It deliberately holds **no** state of its own: F5's `start()` already publishes `Starting`
  synchronously (verified in F5 T3), so the "optimistic" transition is F5's real state and the ≤ 2 s
  budget of US2 AC1 is met without a second source of truth or a timeout-based lie (US3).
- Added `View/DashboardViewModel.kt` — a thin shell: `provider.snapshots()` → `mapper.map` →
  `distinctUntilChanged()` → `stateIn(viewModelScope, WhileSubscribed(5 s), initialState())`, plus
  `start()`/`stop()` delegating to the command handler. `initialState()` claims nothing about the service
  (`Stopped`, `Unavailable`, `canStart = false`, offset 0, `ResponseSpeed.DEFAULT`).
- Build: added `androidx.lifecycle:lifecycle-viewmodel-ktx` to the version catalog (referencing the
  existing `androidxLifecycle = "2.10.0"`) and to `:app` for `viewModelScope` — catalog only, no
  hard-coded version in a module build file.
- Tests (JUnit 4, `:app` JVM). `DashboardStateCollectorTest` (3): no light is collected and the readout
  is blank while stopped; light is collected only while running and released on stop (asserted through
  the fake's live-collection counter); the snapshot carries every service value. `DashboardViewModelTest`
  (7): US2 AC1 start → `Starting` immediately → `Running(77)` from the service; US2 AC2 stop reaches the
  service; US3 AC2 `Failed` flips the dashboard to the error state; US1 AC4 a reading appears only while
  running; the published state equals the mapped snapshot incl. the clamp hint; nothing is collected
  until a subscriber appears and collection stops again after the `WhileSubscribed` timeout; the initial
  state is neutral. `LuxReadoutSamplerTest` (3): a 50 ms sensor burst yields ≤ 1 emission per 500 ms in a
  2 s window, an unchanged value is emitted exactly once, and the interval constant is 500 ms.
- Verification (from the repo root):
  `$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot'` then
  `.\gradlew.bat :app:testDebugUnitTest --no-daemon --console=plain` → **BUILD SUCCESSFUL**, 11 classes /
  49 tests, 0 failures (T2 adds 3 classes / 13 tests). `:core` was not touched.
