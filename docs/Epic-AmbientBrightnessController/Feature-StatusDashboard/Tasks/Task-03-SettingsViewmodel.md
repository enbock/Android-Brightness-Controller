# T3 — Settings ViewModel

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-StatusDashboard.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T3 — Settings ViewModel**: implement `SettingsViewModel` + `SettingsUiState` over
  `BrightnessOffsetSource.offset`/`application` and `ResponseTuningService.responseSpeed`, with intents
  `setOffsetPercent(Int)` (snap to `BrightnessOffsetRange`, −50…+50 step 1),
  `resetOffset()` → `BrightnessOffsetService.reset()`, `setResponseSpeedPercent(Int)` (0…100, default 50)
  and `resetResponseSpeed()` → `setResponseSpeed(ResponseSpeed.DEFAULT)`; writes are `suspend` in
  `viewModelScope`. *Files: `.../StatusDashboard/View/SettingsViewModel.kt`, `.../View/SettingsUiState.kt`.*
  *Verify: values pre-filled from the service flows (US4 AC1); a change while stopped is persisted by the
  service so the next start uses it (US4 AC4).*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `bd1d3292f61b9762bc8e37b9ee484169b84d7fb5`.*

- **State fields.** `View/SettingsUiState.kt` is an Android-free, immutable `data class` with
  `offsetPercent` (the active F3 offset), `canResetOffset` (false at
  `BrightnessOffsetRange.DEFAULT`, so the reset affordance is only offered when it does something),
  `offsetApplication` (F3's clamp outcome, so a limited offset can be indicated) and
  `responseSpeedPercent` + `canResetResponseSpeed` (same rule for F4 at `ResponseSpeed.DEFAULT`). No
  resource ids; every field's KDoc names the story/AC it serves.
- **Pre-filled, never a local copy.** `View/SettingsViewModel.kt` combines `offsetService.offset`,
  `offsetService.application` and `responseTuningService.responseSpeed` with
  `distinctUntilChanged()` + `stateIn(viewModelScope, WhileSubscribed(5 s), initialState())`. The
  values therefore come from the services (US4, AC4.1) and the class keeps no state of its own.
  `initialState()` is a neutral placeholder (`0`, `false`, `NotEvaluated`, `ResponseSpeed.DEFAULT`,
  `false`) that claims nothing before the first emission.
- **One offset parameter.** The constructor takes a single `BrightnessOffsetService` (it extends
  `BrightnessOffsetSource`, and the composition root always has the writable service), so the screen
  reads through the same object it writes to; the KDoc records that choice.
- **Clamping/snapping rules.** `setOffsetPercent(percent)` runs `snapToOffsetGrid` — `coerceIn(MIN,
  MAX)` then align to `BrightnessOffsetRange.STEP` around `DEFAULT` (`roundToInt`, re-clamped) — before
  `BrightnessOffset.of(...)`, so a continuous slider value can never throw; `120 → +50`, `-999 → -50`.
  `setResponseSpeedPercent(percent)` clamps through `ResponseSpeed.of(percent)` (0..100), so `150 →
  100`. `resetOffset()` calls `BrightnessOffsetService.reset()`; `resetResponseSpeed()` calls
  `setResponseSpeed(ResponseSpeed.of(ResponseSpeed.DEFAULT))`.
- **Writes are suspend.** All four intents launch the suspend service call in `viewModelScope`, so
  nothing blocks and no write runs on the main thread. Persistence is F3/F4's shared `SettingsStore`,
  so a change made while the controller is stopped is stored and takes effect on the next start
  (US4, AC4.4) — documented in the class KDoc.
- **Tests (JUnit 4, `:app` JVM).** Extended `testdoubles/DashboardFakes.kt` with a writable
  `FakeBrightnessOffsetService : BrightnessOffsetService` (records `setOffset`/`reset`/`adjustBy` and
  publishes into its `offset` flow) and made `FakeResponseTuningService` record `lastSetSpeed`.
  `SettingsViewModelTest` (6): `US4 AC1 - the state is pre-filled from the service flows` (offset
  `-17`, speed `80`, no intent called); `a change to the service flows is reflected without recreating
  the ViewModel`; `setOffsetPercent clamps a continuous value to the offset range before writing`
  (`120 → +50`, `-999 → -50`, `7 → 7`); `resetOffset returns the offset to the default and
  canResetOffset follows it`; `setResponseSpeedPercent clamps to the range and resetResponseSpeed
  writes the default` (`150 → 100`); `US4 AC4 - a change while nothing else runs is persisted by the
  service` (a fresh read reports the written value).
- Verification (from the repo root):
  `$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot'` then
  `.\gradlew.bat :app:testDebugUnitTest :app:assembleDebug --no-daemon --console=plain` → **BUILD
  SUCCESSFUL**, 12 classes / 55 tests, 0 failures (T3 adds 1 class / 6 tests). No `:core` change and
  no `com.ambientbrightness.infrastructure.*` import outside `Composition/**`.
