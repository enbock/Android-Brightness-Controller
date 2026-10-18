# F7 — Status Dashboard & Service Control

| Field | Value |
|---|---|
| **Epic** | [EPIC: Ambient Brightness Controller](../Epic-AmbientBrightnessController.md) |
| **Depends on** | F3 (Manual Brightness Offset), F4 (Response Tuning), F5 (Background Service), F6 (Permissions & Setup Flow) |
| **Depended on by** | — |

## Overview

This feature is the app's main screen and the user's single place of truth: it shows what the sensor is
measuring, what brightness the app is applying, which settings are active, and whether the controller
is actually running — and it is where the user starts and stops everything.

## User Story 1 — See what the app is doing right now

As a user, I want to see the current light reading and the brightness being applied, so that I can
verify the app is working correctly.

#### Acceptance Criteria

- Given the controller is running, when I open the app, then I see the current ambient light reading
  and the currently applied brightness, both updating live without me refreshing anything.
- Given the controller is running, when I look at the light reading, then it is shown as a numeric lux
  value together with a qualitative description of the light level (for example "dark", "overcast",
  "bright sunlight").
- Given the controller is running, when I look at the dashboard, then I can see the currently active
  offset and response speed values.
- Given the controller is stopped, when I open the app, then the dashboard clearly shows the stopped
  state and does not display a live reading as if it were current.

## User Story 2 — Start and stop the controller

As a user, I want one obvious control to start and stop the controller, so that I am always in charge
of whether the app changes my screen.

#### Acceptance Criteria

- Given all required permissions are granted, when I activate Start, then the controller begins and the
  dashboard shows it as running within 2 s.
- Given the controller is running, when I activate Stop, then brightness control ends, the ongoing
  notification disappears, and the brightness is handled according to the configured stop behaviour.
- Given a required permission is missing, when I look at the Start control, then it is disabled and the
  reason is shown next to it.

## User Story 3 — Honest feedback instead of stale state

As a user, I want the app to tell me when it cannot do what it claims, so that I do not trust a
"running" indicator that is lying to me.

#### Acceptance Criteria

- Given the applied brightness has been limited because the offset could not be fully applied, when I
  look at the dashboard, then this is indicated rather than hidden.
- Given a brightness write fails while the controller is running, when the failure occurs, then the
  dashboard switches to an error state instead of continuing to show "running".
- Given the service has been stopped by the system, when I return to the app, then the dashboard
  reflects the actual current state rather than the last known state.

## User Story 4 — Reach the settings without losing sight of the state

As a user, I want the offset and response-speed settings to be reachable from the dashboard, so that
tuning stays a quick feedback loop even though the controls live on their own screen.

#### Acceptance Criteria

- Given I am on the dashboard, when I activate the settings entry point, then the settings screen opens
  with the currently active offset and response-speed values pre-filled.
- Given the controller is running, when I change the offset on the settings screen, then the change is
  reflected in the applied brightness and the dashboard shows the new value when I return to it.
- Given the controller is running, when I change the response speed on the settings screen, then the
  new behaviour applies to the next adjustment.
- Given I change either setting while the controller is stopped, when I later start the controller,
  then the stored values are the ones that take effect.

## Out of Scope

- A separate settings screen for values other than the offset and response speed.
- History, charts or statistics of past light readings and brightness values.
- Exporting or importing configuration.
- Theming, dark mode, or visual customisation beyond basic readability.
- Widgets, tiles or shortcuts outside the app.

## Resolved Decisions

- The dashboard shows live values and the start/stop control; the offset and response-speed controls
  live on a separate settings screen reachable from the dashboard.
- The light reading is shown as a numeric lux value plus a qualitative description of the light level.

## Technical Design

### Architecture & Components

F7 is **presentation only** and lives in the `Application/` layer of `:app` (Android 16 / API 36,
minSdk = targetSdk = 36, Kotlin, Jetpack Compose, Material 3). It owns no business logic and adds no
`:core` type: every value shown is read from the five existing boundaries — F1 `AmbientLightService`,
F5 `BrightnessBackgroundService`, F6 `AppSetupService`, F3 `BrightnessOffsetService`, F4
`ResponseTuningService`. Applied brightness and the write-failure signal come **only** from F5's
`BackgroundServiceState` (re-publishing F2's `BrightnessStatus`); F7 never touches F2/F1 clients.

Added components, all under `:app` / `com.ambientbrightness.app.Application`:

- `StatusDashboard/View/` — `DashboardViewModel` + `DashboardUiState`, `SettingsViewModel` +
  `SettingsUiState`; `View/Task/` holds the pure helpers `DashboardUiStateMapper` (service-state →
  UI-state, no Android), `LuxReadoutSampler` (display-rate throttling) and `LightLevelLabelResolver`
  (F1 `LightLevel` → string resource id).
- `StatusDashboard/Adapter/` — `DashboardStateCollector`: subscribes to the five services and emits one
  combined `DashboardSnapshot`; `Controller/` — `ControllerCommandHandler`: `Start`/`Stop` intents → F5
  calls plus the *optimistic* `Starting` transition (US2 AC1 ≤ 2 s).
- `StatusDashboard/Navigator/` — `DashboardNavigator` (Compose-Navigation `dashboard` ⇄ `settings`);
  `View/SubViews/` — `LiveReadoutPanel`, `ControllerControlPanel`, `SetupSummaryPanel`,
  `SettingsEntryPoint`, plus `Settings/SubViews/`: `OffsetControl`, `ResponseSpeedControl`,
  `ResetAffordance`.
- `res/values/strings.xml` + `res/values-de/strings.xml` — every user-visible string incl. the five `LightLevel` labels.

### Data Flow

**Read (per emission).** `DashboardStateCollector` combines `AmbientLightService.luxStream()`
(→ `LuxReadoutSampler`), `BrightnessBackgroundService.state`, `AppSetupService.status`/
`isReadyToStart`, `BrightnessOffsetService.offset` + `.application`, `ResponseTuningService`
`.responseSpeed` → `DashboardSnapshot` → `DashboardStateMapper` → `DashboardUiState` (`StateFlow`) →
Compose via `collectAsStateWithLifecycle()`. The settings screen observes the same F3/F4 `StateFlow`s
directly, so values are **pre-filled** from the services (US4 AC1) with no manual refresh.

**Lux sampling (anti-recomposition).** F1's stream is high-frequency; the raw value never reaches
Compose. `LuxReadoutSampler` applies `conflate()` + `distinctUntilChanged()` +
`sample(DISPLAY_SAMPLE_INTERVAL_MS = 500)` and pauses while the app is not resumed (lifecycle-aware
collection), so the readout repaints ≤ 2×/s regardless of sensor rate. Classification is F1's
`LightLevelClassifier`, run once per *sampled* value.

**Write (user action).** Dashboard: `ControllerCommandHandler` → `BrightnessBackgroundService.start()`/
`.stop()`; on `Start` the UI shows `Starting` at once and resolves to `Running`/`Failed` from the
service flow. Settings: `setOffset(...)`/`reset()` on F3 and `setResponseSpeed(...)` on F4; the resulting
`StateFlow` round-trips into the dashboard (US4 AC2–AC4). Writes are `suspend` on `viewModelScope`.

### Key Interfaces & Classes

Boundary types consumed (all existing — no redefinition): `AmbientLightService`,
`BrightnessBackgroundService`, `AppSetupService`, `BrightnessOffsetSource`/`BrightnessOffsetService`,
`ResponseTuningService`, plus VOs `LuxSample`, `SensorAvailability`, `LightLevel`,
`BackgroundServiceState`, `SetupStatus`/`SetupRequirementId`, `BrightnessOffset`,
`BrightnessOffsetApplication` and `ResponseSpeed`. UI-side types F7 introduces (Android-free):

```kotlin
// Application/StatusDashboard/View
interface DashboardStateProvider { fun snapshots(): Flow<DashboardSnapshot> }   // Collector impl.
sealed interface ControllerUiState {                     // maps F5 BackgroundServiceState (US2/US3)
    data object Stopped : ControllerUiState
    data object Starting : ControllerUiState
    data class Running(val appliedBrightnessPercent: Int?) : ControllerUiState
    data class Blocked(val blockers: List<SetupRequirementId>) : ControllerUiState  // F6 gate (US2 AC3)
    data class Failed(val reason: String) : ControllerUiState                       // (US3 AC2)
    data object StoppedBySystem : ControllerUiState                                 // (US3 AC3)
}
sealed interface LightReadout {                          // (US1 AC1/AC2/AC4)
    data object Unavailable : LightReadout               // F1 Unavailable → no stale value
    data class Reading(val lux: Float, val level: LightLevel) : LightReadout
}
sealed interface ClampHint { data object None : ClampHint; data class Limited(val atMaximum: Boolean) : ClampHint }
data class DashboardUiState(
    val controller: ControllerUiState, val light: LightReadout, val setupSummary: SetupStatus?,
    val canStart: Boolean /* AppSetupService.isReadyToStart (gate only) */,
    val offsetPercent: Int, val responseSpeedPercent: Int, val clampHint: ClampHint,
)
```

`DashboardStateMapper` is a pure `(DashboardSnapshot) -> DashboardUiState` function; the ViewModel is a
thin `stateIn`/`viewModelScope` shell. Strings resolve in the View via `LightLevelLabelResolver`
(`@StringRes`), keeping the mapper localisation-agnostic; the settings mapping is trivial (percent ↔
control value).

### State & Persistence

No new persistence: offset and response speed are stored by F3/F4 through F4's shared `SettingsStore`,
running state by F5's `ControllerStateStore`; F7 keeps only ephemeral `StateFlow`s rebuilt at process
start. UI state is immutable and `distinctUntilChanged` (equal re-emissions do not recompose); `Starting`
is a short-lived optimistic state reconciled by the next F5 state (no timeout-based lie).

### Android Platform Specifics

- **UI artifacts — the version catalog is the single source of truth.** `gradle/libs.versions.toml`
  (created by F1/T0) is authoritative; do not hard-code a version in a module build file and do not
  re-derive versions from this document. T0 pinned the Google-validated set for `compileSdk = 36`:
  AGP 9.0.1, Kotlin 2.3.20, Gradle 9.1.0, `androidx.compose:compose-bom:2026.03.01` (brings
  `androidx.compose.material3`), `androidx.activity:activity-compose:1.13.0`,
  `androidx.core:core-ktx:1.18.0`, `androidx.lifecycle:lifecycle-runtime-ktx:2.10.0`.
  T5 must **add** `androidx.lifecycle:lifecycle-runtime-compose` (for `collectAsStateWithLifecycle`)
  and `androidx.navigation:navigation-compose` to that catalog, choosing versions that are compatible
  with the pinned AGP 9.0.1 / Kotlin 2.3.20 toolchain and with the Compose BOM already in the catalog.
  Latest known stable are `lifecycle-runtime-compose:2.11.0` and `navigation-compose:2.10.2`, but the
  catalog's existing Compose BOM wins if a conflict appears — the build must stay green.
  F3/F4 add `androidx.datastore:datastore-preferences` (latest known stable 1.2.1) the same way.
  Sampling stops when backgrounded — do **not** start a second coroutine for the sensor.
- **Navigation between screens (confirmed):** `androidx.navigation:navigation-compose` with
  **type-safe routes** — `@Serializable` route objects passed to `NavHost(navController, startDestination = Dashboard)`
  and `composable<Dashboard> { }` instead of route strings. The type-safe API is documented as available
  since Navigation 2.8.0 and is stable in 2.10.2; it requires the Kotlin Serialization plugin, which
  must also be added to the catalog and to `:app`'s plugins by T5.
- **Predictive back (confirmed, targetSdk 36):** on Android 16+ devices the predictive back system
  animations are enabled **by default**, and `onBackPressed()` is no longer called and
  `KeyEvent.KEYCODE_BACK` is no longer dispatched. F7 must use only supported back APIs — `NavHost`'s
  own back handling plus Compose `BackHandler`/`PredictiveBackHandler` — and must never override
  `onBackPressed()`. Do not set `android:enableOnBackInvokedCallback="false"`: that is an opt-out, not a
  fix. Android 16 additionally brings predictive back to 3-button navigation (long-press Back).
- **Edge-to-edge (confirmed, targetSdk 36):** `windowOptOutEdgeToEdgeEnforcement` is ignored for apps
  targeting API 36 on Android 16, so both screens must be inset-aware (`Scaffold` with
  `WindowInsets.safeDrawing` / `contentWindowInsets`) rather than relying on an opt-out.
- `OffsetControl` = Material 3 `Slider`, `steps` from `BrightnessOffsetRange` (−50…+50, step 1 → 100
  intervals); `ResponseSpeedControl` = single `Slider` 0…100 labelled "fastest/slowest". Both are
  continuous, so snap emitted values to the domain step before calling the service. Accessibility:
  `contentDescription` on the Start/Stop button and sliders (TalkBack).
- No new permission and no manifest entry (F5/F6 own those); F7 must not request permissions.

### Edge Cases & Failure Handling

- `Stopped` → stopped state, **no** live lux/brightness (US1 AC4): `LightReadout` blanked, never the last known value.
- `Blocked` (Blocking requirement `Missing`/`Unsupported`, or `canStart == false`) → Start disabled with
  `SetupStatus.blockers` mapped to a localized reason (US2 AC3); tapping it delegates to F6.
- `Failed(reason)` → error state, "running" hidden (US3 AC2); Stop stays available (F5 keeps the service alive so the user can stop it).
- `StoppedBySystem` / service gone while `desiredRunning` was true → UI follows F5's actual state,
  re-read on `ON_RESUME` (US3 AC3); never a cached "running".
- `ClampedTo(Maximum|Minimum)` → non-blocking "offset limited" hint (US3 AC1); `NotEvaluated` → no hint
  (F3: not applied yet, so no clamp claim).
- `SensorAvailability.Unavailable(...)` → `LightReadout.Unavailable` + localized message; Start is gated
  by F6, not F7. Returning from the settings screen rebuilds the dashboard from the services.

### Test Strategy

- `:app` JVM tests: `DashboardUiStateMapper` over every `BackgroundServiceState` × `SensorAvailability`
  × `BrightnessOffsetApplication` combination (incl. `Running(appliedBrightness = null)`);
  `LuxReadoutSampler` on a virtual-time `TestScope` (≤ `DISPLAY_SAMPLE_INTERVAL_MS` cadence, stable
  value); `DashboardViewModel`/`SettingsViewModel` with fakes (`FakeAmbientLightService`,
  `FakeBrightnessBackgroundService`, `FakeAppSetupService`, `FakeBrightnessOffsetService`,
  `FakeResponseTuningService`, `InMemorySettingsStore`).
- Compose UI tests (androidTest, `createComposeRule`): one per screen — dashboard (stopped message, live
  readout, Start disabled + reason) and settings (pre-filled values, slider → service call, reset).
- Instrumented smoke test: app boots with `Dummy*` adapters and the dashboard renders without hardware.
- **Cannot be unit-tested**: real foreground-service lifecycle / system-initiated stop, physical sensor
  behaviour, on-device DataStore I/O and the real `SCREEN_BRIGHTNESS` write (covered by F1/F2/F5 tests).

### Reported bugs 1 and 2 — live readout and process-resume state

F7's `DashboardStateCollector.latestLux()` independently calls `AmbientLightService.start()` while
the service is Starting/Running. Because F2 and the Activity collector also collect that cold flow
and `DefaultAmbientLightService` serializes sessions instead of broadcasting samples, the dashboard
can starve the controller as well as fail to show live lux. The dashboard must consume the shared
sample stream defined by the F2/F1 integration task; it must not create a competing sensor
registration. Test consumer startup order permutations and prove the first sensor sample is available
to both the brightness writer and dashboard while maintaining only one physical sensor listener.
Source evidence: `app/src/main/kotlin/com/ambientbrightness/app/StatusDashboard/Adapter/DashboardStateCollector.kt`
(`latestLux`) independently calls `ambientLightService.start()`, while F2's
`core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BrightnessControllerUseCase/BrightnessControllerUseCase.kt`
also collects that service flow.

The user's reported reopening symptom is uncertain: it may have been a service-unavailable dashboard
state or general app unresponsiveness, and is not claimed as reproduced. F7's ON_RESUME hook currently
invokes only `AppSetupService.refresh()`. If a new process was created, F5's fresh state flow starts at
`Stopped`; observing that StateFlow directly is not reconciliation with persisted `desiredRunning` or
an Android service restarted with a null intent. After F5 implements restart recovery, the dashboard
must collect that recovered live state and display current applied brightness/lux, or an honest
stopped/error state when recovery does not succeed. Verify process/service recovery and UI
responsiveness separately after close/reopen following a platform process kill with
`desiredRunning=true`, and separately with it false; assert no stale running label and no invented
live readout in the false case.

## Task Breakdown

- [x] [T1 — UI state model and pure mapper](./Tasks/Task-01-UiStateModelAndPureMapper.md)
- [ ] [T2 — Collector, lux sampler and dashboard ViewModel](./Tasks/Task-02-CollectorLuxSamplerAndDashboardViewmodel.md) — follow-up: observe the shared lux stream without opening a competing sensor session.
- [x] [T3 — Settings ViewModel](./Tasks/Task-03-SettingsViewmodel.md)
- [x] [T4 — Dashboard screen, sub-views and strings](./Tasks/Task-04-DashboardScreenSubViewsAndStrings.md)
- [x] [T5 — Settings screen, navigation and reset affordance](./Tasks/Task-05-SettingsScreenNavigationAndResetAffordance.md)
- [ ] [T6 — Resume reconciliation, error/clamp integration and DI](./Tasks/Task-06-ResumeReconciliationErrorClampIntegrationAndDi.md) — follow-up: reconcile F5 after process recreation before presenting dashboard state on resume.

## Suggested Implementation Order

T1 → T2 → T3 first (pure Kotlin / fakes-only) to lock the state mapping before any pixels; T4 then T5
build the two screens on those ViewModels (T5 needs T4's `SettingsEntryPoint`/`NavHost` and the T4
strings); T6 closes the loop with resume reconciliation, DI and end-to-end verification once the screens
exist. Rationale: the dashboard mapping is the riskiest part (four controller states, a high-frequency
source, F3/F5 semantics), so it is designed and tested first; settings/navigation are lower-risk and land
last; every platform-specific item has been verified against the Android 16 docs (see Android Platform Specifics).
