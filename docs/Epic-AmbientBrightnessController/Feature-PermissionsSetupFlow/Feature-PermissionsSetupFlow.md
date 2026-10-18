# F6 — Permissions & Setup Flow

| Field | Value |
|---|---|
| **Epic** | [EPIC: Ambient Brightness Controller](../Epic-AmbientBrightnessController.md) |
| **Depends on** | F5 (Background Service) |
| **Depended on by** | F7 (Status Dashboard) |

## Overview

This feature gets the user from "installed" to "working" with the minimum of confusion. Because the
app needs a *special* permission that cannot be granted by a normal dialog, plus a runtime
notification permission and a battery-optimization exemption, the setup flow has to explain each step,
open the right system screen, detect the outcome, and clearly report what is still missing.

## User Story 1 — Understand what the app needs and why

As a first-time user, I want a short explanation of what the app does and which permissions it needs
before it asks for anything, so that I can decide whether to trust it.

#### Acceptance Criteria

- Given the app is opened for the first time after installation, when the first screen is shown, then
  it briefly explains what the app does and lists the permissions it will ask for together with the
  reason for each one.
- Given the setup flow lists a permission, when it is described, then the description is written in
  plain, non-technical language and does not rely on Android jargon.
- Given the user is in the setup flow, when they complete or skip it, then the app does not ask for
  the same thing again on every subsequent start.

## User Story 2 — Grant the required permissions

As a user, I want each permission request to take me straight to the right system screen, so that I do
not have to figure out where the setting lives.

#### Acceptance Criteria

- Given the `WRITE_SETTINGS` permission has not been granted, when I activate the corresponding setup
  step, then the system screen for "Modify system settings" for this app is opened directly.
- Given the notification permission has not been granted, when I activate the corresponding setup
  step, then the system runtime permission dialog for notifications is shown.
- Given battery optimization is active for this app, when I activate the corresponding setup step,
  then the system screen to exempt the app from battery optimization is offered.
- Given I return to the app after visiting a system screen, when the app resumes, then it re-checks
  the permission state and updates the UI immediately without requiring a restart.

## User Story 3 — Always know what is blocking

As a user, I want to see exactly which requirement is still missing, so that I can tell the difference
between "not set up yet" and "broken".

#### Acceptance Criteria

- Given at least one required permission is missing, when I look at the app, then each missing
  requirement is listed with its own state, and the Start action is disabled.
- Given I try to start the controller while a requirement is missing, when I attempt it, then the app
  explains exactly which requirement blocks the start instead of failing silently.
- Given all requirements are satisfied, when I look at the app, then the setup state is clearly shown
  as complete and the Start action is enabled.

## User Story 4 — Handle uncovered and degraded situations

As a user, I want the app to be honest when my device or configuration cannot support it, so that I do
not keep troubleshooting something that cannot work.

#### Acceptance Criteria

- Given the device has no ambient light sensor, when the app starts, then it reports the device as
  unsupported, explains why, and does not offer to start the controller.
- Given the `WRITE_SETTINGS` permission is revoked while the controller is running, when the next
  brightness write fails, then the app detects this, stops attempting writes, and tells the user which
  permission needs to be re-granted.
- Given the notification permission is not granted, when the controller is started, then the app warns
  that the ongoing-service notification may not be visible to the user.

## Out of Scope

- Requesting permissions that the app does not need.
- Root, ADB or device-owner based permission grants.
- Automated permission granting without user interaction.
- Onboarding videos, account creation, or any sign-in flow.
- Explaining Android internals beyond what is needed to complete the setup.

## Resolved Decisions

- The setup flow is a **single checklist screen** that also serves as the permanent "what is still
  missing" view; there is no multi-step wizard.
- The checklist is **skippable**, so the user can look around the app before granting anything.
- Granting the battery-optimization exemption is **advisory**: it is recommended and explained, but it
  never blocks Start. Only the light sensor and `WRITE_SETTINGS` gate Start.

## Technical Design

### Architecture & Components

Same three modules as F1/F2/F5 (`:core` pure Kotlin, `:infrastructure` Android adapters, `:app` composition root; package root `com.ambientbrightness`). F6 owns the **aggregated requirement-state model** and the platform navigation glue; **F7 owns the presentation** (Compose checklist screen, copy, Start button). F6 does not re-detect permissions: it composes existing checkers.

`:core` (Android-free):
- `AppSetupService` (boundary interface) — the single surface **F7 consumes**; exposes aggregated `SetupStatus` and the Start gate `isReadyToStart`, plus `refresh()` / `acknowledgeSetup()`.
- `SetupRequirementAggregator` — the aggregation use-case/service implementing `AppSetupService`. Breaks down into per-requirement `Task/` classes: `SensorRequirementTask`, `WriteSettingsRequirementTask`, `NotificationRequirementTask`, `BatteryRequirementTask` (each maps one source into a `SetupRequirement`).
- Model VOs/enums: `SetupStatus`, `SetupRequirement`, `SetupRequirementId` (`AmbientLightSensor`, `WriteSettings`, `Notifications`, `BatteryOptimization`), `RequirementSeverity` (`Blocking`, `Advisory`), `RequirementState` (sealed: `Satisfied`, `Missing(reason)`, `Unsupported(reason)`, `Unknown`), `SetupAction` (sealed: `RequestNotificationPermission`, `OpenWriteSettings`, `OpenBatteryOptimizationSettings`, `None`).
- `NotificationPermissionChecker` (new `:core` interface) — `val isGranted: Flow<Boolean>`.
- `SetupStateStore` (new `:core` interface) — persistence of `setupAcknowledged` + `lastShownAtMillis` only.

`:infrastructure`: `AndroidNotificationPermissionChecker`, `AndroidSetupStateStore`, and their `Dummy*` variants. **Reuse as-is** (do not re-implement): F1 `AmbientLightService`, F2 `WriteSettingsPermissionChecker` + `AndroidWriteSettingsPermissionChecker`, F5 `BatteryOptimizationClient` + `AndroidBatteryOptimizationClient`.

`:app`: `SetupFlowCollector` (thin pass-through of `AppSetupService`; no logic), `SetupFlowNavigator` (turns a `SetupAction` into an `Intent`/permission request; Activity-scoped), `SetupLifecycleObserver` (ON_RESUME → `refresh()`), manual DI wiring, and the `POST_NOTIFICATIONS` manifest entry. The Compose checklist screen itself is F7.

### Data Flow

Reads: F1 `AmbientLightService.availability` → sensor requirement; F2 `WriteSettingsPermissionChecker` flow → `WriteSettings`; F5 `BatteryOptimizationClient.isExempt` → `BatteryOptimization`; `NotificationPermissionChecker.isGranted` → `Notifications`. `SetupRequirementAggregator` subscribes to all four, maps each through its `Task` class, and emits `SetupStatus`; `isReadyToStart` is true only when every `Blocking` requirement is `Satisfied`.

Actions: F7 renders the list and calls `SetupFlowNavigator.perform(action)` from the tapped row (so the request originates from the Compose screen in `:app`, which has the Activity). `SetupFlowNavigator` fires the correct platform intent / permission launcher. When the user returns, `SetupLifecycleObserver` (ON_RESUME) calls `AppSetupService.refresh()`; the sources re-emit and `SetupStatus` updates without a restart (US2 AC4, US3). Start is invoked on F5 `BrightnessBackgroundService` only when `isReadyToStart`; when blocked, F7 shows `SetupStatus.blockers` reasons (US3 AC2).

### Key Interfaces & Classes

```kotlin
// :core — boundary consumed by F7
interface AppSetupService {
    val status: StateFlow<SetupStatus>
    val isReadyToStart: StateFlow<Boolean>
    fun refresh()                 // re-derive from live sources (called on ON_RESUME)
    fun acknowledgeSetup()        // persists "setup completed/skipped" (US1 AC3)
}
interface NotificationPermissionChecker { val isGranted: Flow<Boolean> }
interface SetupStateStore { var setupAcknowledged: Boolean; var lastShownAtMillis: Long? }

data class SetupStatus(
    val requirements: List<SetupRequirement>,
    val isReadyToStart: Boolean,
    val blockers: List<SetupRequirementId>,   // Blocking requirements that are not Satisfied
)
data class SetupRequirement(
    val id: SetupRequirementId,
    val severity: RequirementSeverity,
    val state: RequirementState,
    val action: SetupAction,                  // what the row should trigger
)
sealed interface RequirementState {
    data object Satisfied : RequirementState
    data class Missing(val reason: MissingReason) : RequirementState          // NotGranted, Revoked
    data class Unsupported(val reason: UnsupportedReason) : RequirementState  // NoAmbientLightSensor
    data object Unknown : RequirementState
}
// :app — platform navigation, Activity-scoped
interface SetupFlowNavigator { fun perform(action: SetupAction); }
```

Severity mapping: `AmbientLightSensor` → `Blocking` (`Unsupported` when no/!usable sensor, US4 AC1); `WriteSettings` → `Blocking`; `BatteryOptimization` → `Advisory` (recommended, but never gates Start); `Notifications` → `Advisory` (never blocks Start, US4 AC3).

### State & Persistence

Only two facts are persisted, via `SetupStateStore` (DataStore/SharedPreferences in `:infrastructure`): `setupAcknowledged` (US1 AC3 — "do not ask again") and `lastShownAtMillis` (analytics/debug only). All requirement states are **re-derived live** on every `refresh()` and are never cached — caching would contradict US2 AC4 (re-check on resume). `SetupStateStore` is a separate concern from F5's `ControllerStateStore`; **rejected**: merging them (different lifetimes and single-responsibility). In-memory only: the current `SetupStatus`.

### Android Platform Specifics

- **`WRITE_SETTINGS` (US2 AC1) — confirmed:** open `Settings.ACTION_MANAGE_WRITE_SETTINGS` with `Uri.parse("package:$packageName")`. The platform documents the `package:` data URI as the way to "directly invoke the management GUI specific to the package name", so the per-app screen is still the supported path, and `Settings.System.canWrite(context)` (API 23+) remains the documented way to read the grant. The same docs state that "in some cases, a matching Activity may not exist, so ensure you safeguard against this" — hence the required `try/catch (ActivityNotFoundException)` and the fallback chain to `ACTION_MANAGE_WRITE_SETTINGS` without a URI, then `Settings.ACTION_APPLICATION_DETAILS_SETTINGS`. Always use the `Settings` constant, never a literal: its value is `android.settings.action.MANAGE_WRITE_SETTINGS` (not `android.settings.MANAGE_WRITE_SETTINGS`).
- **`POST_NOTIFICATIONS` (US2 AC2) — confirmed:** request from the Compose screen in `:app` via `rememberLauncherForActivityResult(ActivityResultContracts.RequestPermission())` (needs an Activity). Denial is **advisory** because the platform states that apps do not need `POST_NOTIFICATIONS` to launch a foreground service, and that with the permission denied the foreground-service notice is visible in the system Task Manager but not in the notification drawer. The controller therefore keeps running and only the stop affordance disappears (F5 behaviour). Missing permission does **not** block Start (US4 AC3).
- **Battery optimization (US2 AC3):** detect via F5's `BatteryOptimizationClient.isExempt` (`PowerManager.isIgnoringBatteryOptimizations`). Guide with `ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS`. **Play Store policy risk:** `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` requires the `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` permission and is restricted by Play to apps whose core function genuinely needs exemption; the safer alternative is the guiding settings intent (no special permission). Prefer the guide screen.
- **Lifecycle re-check (US2 AC4):** `SetupLifecycleObserver` as a `DefaultLifecycleObserver`/`LifecycleEventObserver` on ON_RESUME calls `refresh()`; F7 may also use `LifecycleResumeEffect`. No polling.
- **Sensor (US4 AC1):** read `AmbientLightService.availability`; `Unavailable(NoSensor|NotUsable)` → `Unsupported`, and F7 must not offer Start. No runtime permission is involved.

### Edge Cases & Failure Handling

- No ambient-light sensor → `AmbientLightSensor = Unsupported(NoAmbientLightSensor)`, `Blocking`, Start hidden/disabled with the reason shown (US4 AC1).
- `WRITE_SETTINGS` revoked at runtime → F2's `AutomaticBrightnessController` reports failure → F5 `BackgroundServiceState.Failed`; on the next ON_RESUME the checker flow reports `Missing(Revoked)` and F6 surfaces it (US4 AC2). F6 only reports which requirement is missing; F2/F5 own stopping the writes.
- `POST_NOTIFICATIONS` denied → `Notifications = Missing(NotGranted)`, `Advisory`; Start stays enabled and F6 marks the warning so F7 can explain the invisible notification (US4 AC3).
- Setup skipped → `acknowledgeSetup()` persists `setupAcknowledged = true`; the checklist remains the permanent "what is missing" view (Resolved Decisions), still reachable.
- Source emits nothing yet → `Unknown` (treated as not-ready only for `Blocking` requirements until first real value); avoid a false "ready".
- `ActivityNotFoundException` from any launch → fall back chain above; never crash.
- F5 force-stop left `desiredRunning = true` with nothing running → F6 does not act on it (out of scope); the checklist reconciles the visible state on open.

### Test Strategy

`:core` (JVM): `SetupRequirementAggregator` unit-tested with fakes of F1 `AmbientLightService`, F2 `WriteSettingsPermissionChecker`, F5 `BatteryOptimizationClient` and `NotificationPermissionChecker`, plus a `DummySetupStateStore`. Cover the full matrix (satisfied / missing / unsupported / unknown), `isReadyToStart` gating, `blockers` contents, and that `Notifications` and `BatteryOptimization` never block. `:infrastructure`: instrumented tests for `AndroidNotificationPermissionChecker` and `AndroidSetupStateStore` (persistence + `Dummy*` keep `:app` runnable without a device). `:app`: `SetupFlowNavigator` intent/fallback tests (Robolectric or instrumented) incl. `ActivityNotFoundException`. Do not duplicate F1/F2/F5 tests; no Compose tests here (F7 owns UI tests).

## Task Breakdown

- [x] [T1 — Core requirement model & boundary interfaces](./Tasks/Task-01-CoreRequirementModelBoundaryInterfaces.md)
- [x] [T2 — Aggregation service and per-requirement tasks](./Tasks/Task-02-AggregationServiceAndPerRequirementTasks.md)
- [x] [T3 — Infrastructure adapters (real + Dummy)](./Tasks/Task-03-InfrastructureAdaptersRealDummy.md)
- [x] [T4 — App navigation, lifecycle and wiring](./Tasks/Task-04-AppNavigationLifecycleAndWiring.md)
- [x] [T5 — First-run acknowledgement and degraded-path integration](./Tasks/Task-05-FirstRunAcknowledgementAndDegradedPathIntegration.md)
