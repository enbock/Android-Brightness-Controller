# T1 — UI state model and pure mapper

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-StatusDashboard.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T1 — UI state model and pure mapper**: add `DashboardUiState`, `ControllerUiState`,
  `LightReadout`, `ClampHint`, `DashboardSnapshot`, the pure `DashboardUiStateMapper` and
  `LightLevelLabelResolver`, all Android-free (no `@StringRes` in the mapper). *Files:
  `:app/.../Application/StatusDashboard/View/DashboardUiState.kt`, `.../View/Task/DashboardUiStateMapper.kt`,
  `.../View/Task/LightLevelLabelResolver.kt`.* *Verify: unit tests for every `BackgroundServiceState`→
  `ControllerUiState` mapping (incl. `Running(null)`, `Blocked`, `Failed`, `StoppedBySystem`) and each
  `LightLevel`→resource id.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `b03666e0efabe7e7eeb64dea782c99789095ba82`.*

- **Package placement.** The design names `:app/.../Application/StatusDashboard/...`; the binding
  architecture document prescribes `.../app/<Responsibility>/...` (and F6 already shipped
  `com.ambientbrightness.app.SetupFlow`), so F7 lands in `com.ambientbrightness.app.StatusDashboard`
  with the `View/` and `View/Task/` sub-levels the design specifies. Behaviour is unchanged; this is a
  path convention only.
- Added the Android-free presentation model in `View/`: `DashboardUiState`, the sealed
  `ControllerUiState` (`Stopped`, `Starting`, `Running(appliedBrightness)`, `Blocked(blockers)`,
  `Failed(reason)`, `StoppedBySystem`), the sealed `LightReadout` (`Unavailable`, `Reading(lux, level)`)
  and the sealed `ClampHint` (`None`, `Limited(atMaximum)`). Each variant's KDoc names the story/AC it
  serves.
- Added `View/DashboardSnapshot.kt`: one immutable read of the five service values the dashboard needs
  (F5 state, F1 availability + latest lux, F6 status + gate, F3 offset + offset application, F4 speed).
  It carries plain service values and no presentation decision, which is what keeps the mapper a pure
  function over the full service-state matrix.
- Added `View/Task/DashboardUiStateMapper.kt` — pure `(DashboardSnapshot) -> DashboardUiState`, no
  Android and no resource id. Documented precedence: `Failed` wins over everything (US3 AC3.2), then
  `StoppedBySystem` over the stale start request (US3 AC3.3), then F5's `Starting`/`Running` (the F6
  gate never overrides a real service state), and an idle controller becomes `Blocked` (naming
  `SetupStatus.blockers`) when the gate is false, else `Stopped` (US2 AC2.3). A reading exists only while
  running **with** an available sensor **and** a finite sampled lux, so a stopped controller never shows
  a stale value (US1 AC1.4). `NotEvaluated` maps to `ClampHint.None`, so the app never claims a clamp
  that did not happen (US3 AC1.1).
- Added `View/Task/LightLevelLabelResolver.kt` — `@StringRes` mapping for all six `LightLevel` values,
  kept out of the mapper so the state mapping stays localisation-agnostic (US1 AC1.2).
- Strings: created `app/src/main/res/values/strings.xml` and `values-de/strings.xml` with the six
  light-level labels this resolver needs (later tasks add their own entries to the same files).
- Build: enabled `testOptions.unitTests.isIncludeAndroidResources = true` in `app/build.gradle.kts` so
  the resolver can be asserted against real resource ids in a JVM test (consistent with the project's
  preference for JVM tests over instrumented ones).
- Tests (JUnit 4, JVM): `DashboardUiStateMapperTest` (14 tests) walks the design's matrix — every
  `BackgroundServiceState` -> `ControllerUiState` mapping including `Running(null)`, `Blocked` with named
  blockers, `Failed` never rendering as running, the gate not overriding a real state, the light readout
  for running/stopped/no-sensor/no-sample, all four offset applications -> clamp hint, and pass-through
  of offset/speed/setup summary. `LightLevelLabelResolverTest` (2 tests) asserts each level's resource
  and that the mapping is total with distinct labels.
- Verification (from the repo root):
  `$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot'` then
  `.\gradlew.bat :app:testDebugUnitTest --no-daemon --console=plain` → **BUILD SUCCESSFUL**, 8 classes /
  36 tests, 0 failures (T1 adds 2 classes / 16 tests). `:core` was not touched.
