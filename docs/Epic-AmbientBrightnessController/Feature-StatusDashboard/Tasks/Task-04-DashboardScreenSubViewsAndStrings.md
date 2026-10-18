# T4 — Dashboard screen, sub-views and strings

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-StatusDashboard.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T4 — Dashboard screen, sub-views and strings**: build `DashboardScreen` (`LiveReadoutPanel`:
  lux + `LightLevel` label; `ControllerControlPanel`: one Start/Stop button with reason text when
  disabled; `SetupSummaryPanel`: F6 summary via `SetupFlowNavigator`; `SettingsEntryPoint`) with
  Material 3 and `collectAsStateWithLifecycle`, no hardcoded text; add all strings incl. the `LightLevel`
  labels and clamp/error/stopped messages to `strings.xml` and `values-de/strings.xml`. *Files:
  `:app/.../StatusDashboard/View/DashboardScreen.kt`, `.../View/SubViews/*.kt`,
  `:app/src/main/res/values/strings.xml`, `.../values-de/strings.xml`.* *Verify: Compose test — stopped
  message, live readout, Start disabled with visible reason when `canStart == false`; label resolves from
  resources in a `values-de` locale.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `a6cd7c3aff9518e3a642af04b033eafb36f088dd`.*

- Added `View/DashboardScreen.kt` — a **stateless** `DashboardScreen(uiState, onStart, onStop,
  onSetupAction, onOpenSettings)`, so it renders in a test without a ViewModel or a device, plus
  `DashboardRoute(viewModel, …)`, the stateful wrapper that collects `DashboardViewModel.uiState` with
  `collectAsStateWithLifecycle`. Layout is a Material 3 `Scaffold` with
  `contentWindowInsets = WindowInsets.safeDrawing` and a scrollable column, because edge-to-edge
  enforcement cannot be opted out on API 36. The screen handles **no** back itself: navigation stays
  T5's `NavHost`, which keeps predictive back working (`onBackPressed()` is not used anywhere).
- Added `View/SubViews/`:
  - `LiveReadoutPanel` — numeric lux (`%1$.0f lx`) plus the `LightLevel` label resolved through T1's
    `LightLevelLabelResolver`; `LightReadout.Unavailable` renders "No current reading" (US1, AC1.4)
    and a `ClampHint.Limited` adds the non-blocking offset-limited hint (US3, AC1.1).
  - `ControllerControlPanel` — one Start/Stop button with a `contentDescription` for TalkBack; Start is
    disabled when F6 gates it and `ControllerUiState.Blocked` names the blockers next to it (US2,
    AC2.3); `Failed` shows the reason instead of a "running" label (US3, AC2); `StoppedBySystem` reads
    as such (US3, AC3); `Running` shows the applied brightness when present.
  - `SetupSummaryPanel` — each F6 requirement with its own name and state, and an action button only
    when `action != SetupAction.None`, forwarded through `onSetupAction` so F6 owns the platform step
    (US3, AC3.1/AC3.2).
  - `SettingsEntryPoint` — repeats the active offset and response speed and opens the settings screen
    (US4, AC1.1).
- Added `View/Task/SetupTextResolver.kt` — total `@StringRes` mapping for `SetupRequirementId` and
  `RequirementState` (incl. `Missing(Revoked)` vs `Missing(NotGranted)` and
  `Unsupported(NoAmbientLightSensor)`), so the panels stay layout-only and no state can render a blank
  reason. Each panel and button exposes a `testTag` constant for the Compose tests.
- Strings: extended `values/strings.xml` and `values-de/strings.xml` with every dashboard, control,
  setup and settings string (headings, states, blocker/error reasons, clamp hints, button labels and
  content descriptions) in **English and German**; the existing `light_level_*` keys were not
  duplicated. No user-visible text is hardcoded in a composable.
- Build (catalogue only, no hard-coded versions): added `lifecycle-runtime-compose` (for
  `collectAsStateWithLifecycle`), the Compose BOM to the **androidTest** source set,
  `androidx.compose.ui:ui-test-junit4` (androidTest) and `androidx.compose.ui:ui-test-manifest`
  (debug), so Compose UI tests can run on a device.
- Tests. `:app` JVM `SetupTextResolverTest` (4): every `SetupRequirementId` has a distinct real label;
  every `RequirementState` maps to a real, distinct label; `Missing(Revoked)` is not described as
  `Missing(NotGranted)`; `Unsupported` says the device cannot do it. `:app` androidTest
  `DashboardScreenTest` (6, `createComposeRule`) — the stopped message with no reading, the lux value
  plus its level and an enabled Stop while running, Start disabled with the visible blocker reason, the
  error state instead of "running", the clamp hint, and a German-locale resource assertion. It is
  **compile-verified only** here (no emulator/device attached); the same behaviours are covered on the
  JVM by T1/T2/T3, which is why the instrumented set is an addition rather than the only proof.
- Verification (from the repo root):
  `$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot'` then
  `.\gradlew.bat :app:testDebugUnitTest :app:assembleDebug :app:assembleDebugAndroidTest
  --no-daemon --console=plain` → **BUILD SUCCESSFUL**, `:app` 13 classes / 59 tests, 0 failures (T4
  adds 1 JVM class / 4 tests and 1 instrumented class / 6 tests). No `:core` change.
