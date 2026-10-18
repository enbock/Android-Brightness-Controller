# F2 — Automatic Brightness Control

| Field | Value |
|---|---|
| **Epic** | [EPIC: Ambient Brightness Controller](../Epic-AmbientBrightnessController.md) |
| **Depends on** | F1 (Ambient Light Sensing) |
| **Depended on by** | F3 (Manual Brightness Offset), F4 (Response Tuning), F5 (Background Service) |

## Overview

This feature turns the measured ambient light into an actual screen brightness value and applies it
to the system. It is the core of the app: the part that the user experiences as "the screen is
suddenly readable". It owns the response curve, the safety bounds, and the write to
`Settings.System.SCREEN_BRIGHTNESS`.

## User Story 1 — Automatic adjustment without user interaction

As a user, I want the screen brightness to be adjusted automatically to match the ambient light, so
that the screen stays readable without me touching the brightness slider.

#### Acceptance Criteria

- Given the controller is running and the `WRITE_SETTINGS` permission is granted, when the measured
  ambient light changes, then a new brightness value is computed and applied without any user
  interaction.
- Given a newly computed brightness value differs from the currently applied value by less than the
  configured minimum step, when it is computed, then no system write is performed, so the app does
  not perform needless writes.
- Given the app applies a new brightness, when the write succeeds, then the applied value is
  reflected as the current value in the app's own status, so the UI never reports a stale value.
- Given the app is running, when brightness is applied, then only `Settings.System.SCREEN_BRIGHTNESS`
  is written, and no other system setting is modified.

## User Story 2 — Safe and valid brightness values

As a user, I want the app to never leave my screen unreadably dark or invalidly bright, so that I can
always trust it to keep the display usable.

#### Acceptance Criteria

- Given a computed brightness would be below the minimum allowed value, when it is applied, then it
  is clamped to that minimum and never to 0, so the screen can never be driven to full darkness.
- Given a computed brightness would exceed the device's maximum brightness, when it is applied, then
  it is clamped to the maximum supported by the device.
- Given the device reports its own brightness range, when values are applied, then they are mapped
  into that range rather than assuming a fixed 0–255 scale.
- Given the brightness write fails (for example because the permission was revoked while running),
  when the failure occurs, then the app surfaces an error state, stops attempting writes, and does
  not crash.

## User Story 3 — Predictable lux-to-brightness behaviour

As a user, I want the relationship between light and brightness to be predictable and free of
flickering, so that the screen does not jump between two brightness levels when the light sits at a
boundary.

#### Acceptance Criteria

- Given two ambient light values where one is higher than the other, when both are mapped, then the
  brightness for the higher value is never lower than the brightness for the lower value (the curve
  is monotonically non-decreasing).
- Given the ambient light hovers at a value where the curve changes slope, when the light oscillates
  slightly around that value, then hysteresis prevents the applied brightness from rapidly alternating
  between two levels.
- Given the ambient light is very low, when it is mapped, then the result is a low but still
  readable brightness rather than the minimum, so the screen does not become unnecessarily dark.

## User Story 4 — Coexistence with the device's built-in adaptive brightness

As a user, I want to know when the app conflicts with the phone's own automatic brightness, so that
I understand why the screen does not behave as expected and can resolve it.

#### Acceptance Criteria

- Given the device's built-in adaptive brightness is enabled, when the controller is started, then the
  app informs the user that both systems will compete and offers to switch the built-in adaptive
  brightness off.
- Given the user declines to switch it off, when the controller runs, then it still functions but the
  UI keeps a persistent hint that the built-in adaptive brightness may override the app's value.
- Given the built-in adaptive brightness is disabled, when the controller runs, then the app does not
  show the conflict hint.

## Out of Scope

- The user-facing offset on top of the computed value (that is F3).
- Smoothing, damping and ramp timing (that is F4).
- Per-app or per-scenario brightness profiles.
- Colour temperature, blue-light filtering, or contrast adjustment.
- Screen dimming overlays or any mechanism other than the system brightness setting.

## Resolved Decisions

- Stopping the controller restores the brightness value that was in effect before it was started (see
  F5).
- The app never turns the device's built-in adaptive brightness off by itself; it only explains the
  conflict and links to the system setting.

## Technical Design

### Architecture & Components

Same three modules as F1 (`:core` pure Kotlin, `:infrastructure` Android adapters, `:app`
composition root; package root `com.ambientbrightness`). F2 adds the brightness pipeline and
consumes F1 **only** through `AmbientLightService` (`availability`, `luxStream()`/`start()`,
`stop()`). `AmbientLightSensorClient` is never referenced here.

Components to add (all `:core` unless noted):

- `AutomaticBrightnessController` — boundary/service and single public entry point for F2
  (`start()`, `stop()`, `status: StateFlow<BrightnessStatus>`).
- `BrightnessControllerUseCase` — orchestrates sample → map → transform hooks → clamp → dead-band →
  write, and owns the running flag.
- `BrightnessMappingCurve` (interface) + `DefaultBrightnessMappingCurve` — monotonic lux→fraction.
- `BrightnessRange` (value object: `min`, `max`) + `BrightnessRangeResolver` — resolves the real
  device range.
- `BrightnessValue` (value object, integer in device units, invariant `>= 1`).
- `MinimumStepDeadBand` — suppresses writes when |new − applied| < min step.
- `HysteresisGate` — directional hysteresis against curve oscillation (US3).
- `SystemBrightnessClient` (interface) — read/write `Settings.System.SCREEN_BRIGHTNESS`.
- `DeviceBrightnessRangeClient` (interface) — discovers the device maximum using the guarded
  Android resource lookup described below; returns `null` only when that lookup yields no valid
  maximum.
- `WriteSettingsPermissionChecker` (interface) — reports `WRITE_SETTINGS` grant state as a flow.
- `BuiltInAdaptiveBrightnessState` (value object) + `AdaptiveBrightnessConflictDetector` —
  observable conflict state for F6/F7 (D4); no UI here.
- `BrightnessTransform` (interface) — **extension point**: post-mapping, pre-clamp hooks so F3
  (offset) and F4 (smoothing/ramp) plug into the pipeline without touching it.
- `BrightnessSnapshot` — pre-controller brightness snapshot F5 restores on stop (D3).
- `BrightnessWriteResult` (sealed) — `Applied` / `Suppressed` / `Failed(reason)`.

`:infrastructure`: `AndroidSystemBrightnessClient`, `AndroidDeviceBrightnessRangeClient`,
`AndroidWriteSettingsPermissionChecker`, plus `Dummy*` variants for tests/development.
`:app`: `BrightnessStatusCollector` (thin, forwards status; no business logic).

### Data Flow

`AmbientLightService.luxStream()` → `BrightnessControllerUseCase`: on each `LuxSample`, if not
running or permission missing, skip. Otherwise `DefaultBrightnessMappingCurve` maps lux → fraction
`[0,1]`; `BrightnessTransform` hooks (empty in F2) may modify the fraction; `BrightnessRangeResolver`
+ `BrightnessRange` scale it to device units and clamp to `[max(1,min), max]`; `HysteresisGate`
smooths breakpoint oscillation; `MinimumStepDeadBand` compares against the last applied value and
drops writes below the step. A surviving value is written via `SystemBrightnessClient`; the result
updates `status` (`appliedBrightness`, `lastLux`, `writeState`). `AdaptiveBrightnessConflictDetector`
runs in parallel off the permission/system state and publishes `BuiltInAdaptiveBrightnessState`.

Interfaces (`:core`):

```kotlin
interface AutomaticBrightnessController {
    fun start()
    fun stop()
    val status: StateFlow<BrightnessStatus>
    val adaptiveBrightnessConflict: StateFlow<BuiltInAdaptiveBrightnessState>
}
interface BrightnessMappingCurve { fun toFraction(lux: Float): Float }
interface SystemBrightnessClient {
    fun readCurrent(): Int?
    fun write(value: BrightnessValue): BrightnessWriteResult
}
interface BrightnessTransform { fun apply(fraction: Float, context: BrightnessContext): Float }
```

### Key Interfaces & Classes

- **Curve (US3):** piecewise-linear in log10(lux) space (~1–100 000 lux) through fixed anchor
  points, computed to be monotonically non-decreasing; low-light endpoint returns a low-but-readable
  value (> 1), never the device minimum (AC3.3). Chosen over a lookup table (device-specific, hard to
  keep monotonic) and over a power function (harder to satisfy the "very low light" floor and to
  reason about at breakpoints).
- **Hysteresis (US3/AC3.2):** `HysteresisGate` keeps a dead-band in lux around the current anchor;
  the value only re-maps once the sample leaves the band by a configured margin, so oscillation at a
  slope change cannot alternate the applied value.
- **Range discovery (US2/AC2.3):** on the verified compile/target/min SDK 36 surface,
  `Settings.System` has no `SCREEN_BRIGHTNESS_MAX` constant and `PowerManager` has no public
  brightness-maximum API. Do not reference either API or assume a public power-profile API.
  `AndroidDeviceBrightnessRangeClient` may make one narrowly-scoped runtime lookup for Android's
  internal integer resource `config_screenBrightnessSettingMaximum`, using the public
  `Resources.getIdentifier(name, "integer", "android")` and `Resources.getInteger(id)` APIs.
  This resource is not a public SDK contract: use it only if the identifier exists, retrieval
  succeeds, and the returned maximum is at least the configured positive minimum. Return `null`
  otherwise. The existing core `BrightnessRangeResolver` then supplies its established
  `BrightnessRange(1, 255)` fallback; do not invent additional Settings or PowerManager sources.
  Validate `max >= min` before constructing a `BrightnessRange`, so a malformed resource value can
  never violate the range invariant.
- **Dead-band (US1/AC1.2):** `MinimumStepDeadBand` is the sole reason a computed value is not
  written; it is size-independent of F4's ramp timing.
- **Write failures (US2/AC2.4):** `SystemBrightnessClient.write` returns `Failed` (permission
  revoked, `SecurityException`, `IllegalArgumentException`); the use-case sets an error state, stops
  attempting writes, and never crashes.

### State & Persistence

In-memory only: `status`, `adaptiveBrightnessConflict`, last-applied value and the
`BrightnessSnapshot` are held by `AutomaticBrightnessController` (app-scoped). No database or files;
F2 does not persist the offset (F3) or response settings (F4). The snapshot exists solely so F5 can
restore the pre-controller value on stop (D3); F2 exposes it and does not act on stop itself.

### Android Platform Specifics

- Write path: `Settings.System.putInt(contentResolver, Settings.System.SCREEN_BRIGHTNESS, v)` —
  the only setting modified (AC1.4); no overlay.
- `WRITE_SETTINGS` is a special permission: check with
  `Settings.System.canWrite(context)`. For T4, implement the checker as an app-scoped
  `StateFlow` backed by an `AppOpsManager` `OPSTR_WRITE_SETTINGS` mode watcher. Register the
  listener only while the flow is actively collected (for example, `callbackFlow` with
  `awaitClose` unregistering via `stopWatchingMode`, shared with `stateIn` using
  `SharingStarted.WhileSubscribed`); do not leave a process-lifetime listener registered. Re-read
  `canWrite(context)` initially and on each matching callback. The controller's active collection
  owns observation, and `AndroidSystemBrightnessClient.write` must still check the live grant and
  handle `SecurityException` because revocation and callback delivery can race. The current
  app-scoped `AmbientBrightnessComponent` has no T4 permission-checker lifecycle wiring; add its
  dependency and scope when T4 is composed, and cancel/stop the collection with the controller.
- Android SDK 36 does not expose `SCREEN_BRIGHTNESS_MAX` on `Settings.System` or a public maximum
  brightness API on `PowerManager` (verified against the project's `android-36/android.jar`).
  Do not add compile-time references to those nonexistent symbols. The guarded runtime resource
  lookup above is the only device-maximum probe in this design; when absent or invalid,
  `DeviceBrightnessRangeClient.readRange()` returns `null` and the existing core resolver uses
  1–255. The lookup is deliberately confined to the infrastructure adapter and is not treated as
  a stable public Android API.
- `Settings.System.SCREEN_BRIGHTNESS_MODE` is **read only** (D4) — reading it feeds the conflict
  state, never a write.
- Screen-off is already gated upstream in F1; F2 still ignores samples while stopped.

### Edge Cases & Failure Handling

- Permission revoked mid-run → first failed write flips to `Failed`, writes stop, state surfaced.
- Computed value < minimum → clamp to floor `>= 1`, never 0 (AC2.1); above max → clamp (AC2.2).
- Device range resource unavailable, unreadable, or less than the configured minimum → return no
  reported range and use the core 1–255 fallback; log the fallback without crashing. A reported
  range must have `max >= min`.
- `LuxSample` invalid/unavailable → skip; do not write.
- Rapid oscillation at a breakpoint → absorbed by `HysteresisGate` + dead-band.
- Built-in adaptive brightness on → conflict state `Enabled`, controller still runs (AC4.2); hint
  is a downstream concern (F6/F7).
- `stop()` → controller stops writing; restoration is deferred to F5 via `BrightnessSnapshot`.
- `start()`/`stop()` idempotent.

### Test Strategy

`:core` is pure Kotlin: `DefaultBrightnessMappingCurve` gets a property/monotonicity test (random
lux pairs never decrease) plus low-light floor and breakpoint cases; `HysteresisGate` and
`MinimumStepDeadBand` are unit-tested with a fixed `Clock`; `BrightnessControllerUseCase` is driven
by fake `AmbientLightService`, `SystemBrightnessClient`, `DeviceBrightnessRangeClient` and
`WriteSettingsPermissionChecker`, asserting clamp, dead-band suppression and failure handling
(AC1.2, AC2.1–2.4, AC3.1). `AdaptiveBrightnessConflictDetector` is unit-tested for the three AC4
states. `:infrastructure` adapters are covered by one instrumented test that writes and reads back a
value on a device/emulator with the permission granted; `Dummy*` adapters let `:app` run without
hardware. No Compose tests.

### Reported bug 1 — competing ambient-light collections

The current composition has several callers independently collecting the cold
`AmbientLightService.start()` flow: the F2 control loop, the Activity's F1 `AmbientLightCollector`,
and F7's dashboard lux collector. `DefaultAmbientLightService` serializes those sessions with one
mutex and holds the lock until the active stream is cancelled; it does not multicast samples.
Consequently, an earlier long-lived UI collection can hold the sensor session while F2 waits for
the next session and receives no lux samples. `MainActivity.onStop()` also calls the component's
global `service.stop()`, which can cancel a session used by F2 without restarting it until its screen
state collector receives another transition. The user clarified that the reported off/on cycle means
locking and unlocking the device. These mechanisms are consistent with control beginning only after
that lifecycle/screen-state cycle; no device reproduction is claimed.

Source evidence: `app/src/main/kotlin/com/ambientbrightness/app/MainActivity.kt` (`onStart`,
`onStop`); `app/src/main/kotlin/com/ambientbrightness/app/AmbientLightSensing/Collector/AmbientLightCollector.kt`
(`collect`); `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BrightnessControllerUseCase/BrightnessControllerUseCase.kt`
(`controlLoop`); `app/src/main/kotlin/com/ambientbrightness/app/StatusDashboard/Adapter/DashboardStateCollector.kt`
(`latestLux`); and `core/src/main/kotlin/com/ambientbrightness/core/AmbientLightSensing/AmbientLightService/DefaultAmbientLightService.kt`
plus `.../Task/CollectionSessionRegistry.kt` (serialized session lock and cancellation).

The F2/F1/F7 integration must have one explicit collection owner and fan out each accepted sample to
the controller and live dashboard, rather than independently collecting the cold sensor flow.
Lifecycle cancellation must target only the UI subscription, not stop the service-wide sensor session
while F5 still owns the controller. Keep the existing sensor cleanup guarantee when the final owner
stops. Verify with a fake that starts the Activity/dashboard and controller consumers in either order:
one sensor registration, each sample reaches F2 and F7, Start yields a brightness write on the first
valid sample without a screen cycle, and stopping the UI consumer does not cancel F5's active control.

## Task Breakdown

- [x] [T1 — Domain model & boundary interfaces](./Tasks/Task-01-DomainModelBoundaryInterfaces.md)
- [x] [T2 — Lux→brightness mapping curve and hysteresis](./Tasks/Task-02-LuxBrightnessMappingCurveAndHysteresis.md)
- [x] [T3 — Device range resolution, scaling, clamping and dead-band](./Tasks/Task-03-DeviceRangeResolutionScalingClampingAndDeadBand.md)
- [x] [T4 — System brightness writer, permission checker and failure handling](./Tasks/Task-04-SystemBrightnessWriterPermissionCheckerAndFailureHandling.md)
- [ ] [T5 — Controller orchestration, conflict state, snapshot and composition root](./Tasks/Task-05-ControllerOrchestrationConflictStateSnapshotAndCompositionRoot.md) — follow-up: establish a single shared ambient-light collection for controller and dashboard consumers.

## Suggested Implementation Order

`T1 → T2 → T3 → T4 → T5`. T1 fixes the contract F3/F4/F5 extend, so it lands first. T2 and T3 are
pure and can proceed in parallel once T1 exists; T3 depends on T2's fraction output but only via the
interface, so the clamp/dead-band work can be built against a stub curve. T4 is independent of T2/T3
and can run alongside them (adapters only). T5 depends on all of them because it is the only place
that connects the F1 stream, the status surface and the composition root.
