# EPIC: Ambient Brightness Controller (Android)

| Field | Value |
|---|---|
| **Owner** | Product Owner |
| **Created** | 2026-10-11 |
| **Platform** | Android, minimum and target SDK: Android 16 (API 36) |
| **Brightness mechanism** | System brightness via `WRITE_SETTINGS` (`Settings.System.SCREEN_BRIGHTNESS`) |

## Overview

An Android application that continuously reads the device's ambient light sensor and automatically
adjusts the system screen brightness in real time, so that content stays comfortably readable in any
lighting condition. It runs continuously in the background as a foreground service that resists
system- and OEM-initiated process termination, and lets the user fine-tune the result through a
manual brightness offset and a configurable response speed.

**Target users:** Android 16 users who want display brightness that tracks their environment more
accurately and more responsively than the stock adaptive brightness, and who want that behaviour to
keep working reliably in the background.

## Problem Statement

Stock adaptive brightness on Android is deliberately conservative: it reacts slowly, tends to
undershoot in bright outdoor light, and lets the user correct it only with a global, device-wide
bias slider. Users who read on their phone while moving between environments (indoors → outdoors →
car → dark room) frequently find the screen either too dim to read or uncomfortably bright.

In addition, Android aggressively reclaims background processes. A naive background app that
adjusts brightness is killed within minutes on many devices, so brightness control silently stops
working — often without the user noticing.

## Goals

1. Keep the effective screen brightness matched to the measured ambient light at all times.
2. React fast enough to feel immediate, but without flickering or "pumping" the display.
3. Give the user precise, predictable manual control over the final result (offset + response speed).
4. Keep running in the background indefinitely, surviving system and OEM process termination.
5. Be transparent: the user can always see what the app is measuring, what it is doing, and can stop it.

## Success Metrics

| Metric | Target |
|---|---|
| End-to-end reaction to a step change in ambient light | Noticeable adjustment begins within 1 s; settles within the configured response window |
| Visible brightness flicker / oscillation | None during steady ambient conditions |
| Background service survival | Service still alive and controlling brightness after 24 h of normal use with screen off |
| Unintended brightness changes per hour | 0 (no changes when the user has stopped the service) |
| User-reported "screen unreadable" incidents | 0 in field testing |

## In Scope

- Reading ambient light (lux) from the device light sensor in real time.
- Mapping lux to a system screen brightness value and applying it.
- A user-configurable manual offset on top of the computed brightness.
- A user-configurable response speed (smoothing / damping of brightness changes).
- Running as a foreground service with a persistent notification, plus battery-optimization exemption.
- Guiding the user through the required permissions (`WRITE_SETTINGS`, `POST_NOTIFICATIONS`,
  battery optimization exemption, sensor availability).
- A status screen showing live lux, applied brightness, and service state, with start/stop control.
- A settings screen where the manual brightness offset and the response speed are configured.

## Out of Scope

- Any mechanism other than system brightness (no overlay/dimming windows, no screen filters, no
  colour temperature or blue-light adjustment).
- Automatic scheduling by time of day, geofencing, or location-based profiles.
- Per-app brightness rules or profiles.
- Cloud sync, accounts, analytics, or any network/backend component.
- Root-only or ADB-only features.
- Supporting Android versions below Android 16 (API 36).
- Tablets, foldables, TVs and Wear form factors — phone form factor only.

## Feature Breakdown

Each feature below becomes its own `docs/Epic-<Slug>/Feature-<Slug>/Feature-<Slug>.md` document, written as user stories with
acceptance criteria, and is planned/implemented individually.

### [F1 — Ambient Light Sensing](./Feature-AmbientLightSensing/Feature-AmbientLightSensing.md)
Read the ambient light sensor continuously and expose a reliable, throttled lux stream.
*Covers: sensor availability detection, reading cadence, noise rejection.*

### [F2 — Automatic Brightness Control](./Feature-AutomaticBrightnessControl/Feature-AutomaticBrightnessControl.md)
Convert the measured lux into a system screen brightness value and apply it through
`WRITE_SETTINGS`.
*Covers: the lux → brightness response curve, clamping to valid device ranges (min/max, and never 0),
and writing to `Settings.System.SCREEN_BRIGHTNESS`.*

### [F3 — Manual Brightness Offset](./Feature-ManualBrightnessOffset/Feature-ManualBrightnessOffset.md)
Let the user shift the automatically computed brightness up or down by a configurable amount, so the
perceived result can be corrected without giving up automation.
*Covers: offset range, immediate re-application, persistence across restarts.*

### [F4 — Response Tuning](./Feature-ResponseTuning/Feature-ResponseTuning.md)
Let the user control how quickly the brightness follows changes in ambient light.
*Covers: smoothing/damping parameter, debounce against sensor noise, immediate vs. gradual ramping,
and avoiding oscillation.*

### [F5 — Background Service](./Feature-BackgroundService/Feature-BackgroundService.md)
Run the whole control loop as a foreground service with a persistent notification so the system does
not reclaim it.
*Covers: foreground service + service type declaration, persistent notification, battery-optimization
exemption request, restart-on-task-removed behaviour, and clean shutdown.*

### [F6 — Permissions & Setup Flow](./Feature-PermissionsSetupFlow/Feature-PermissionsSetupFlow.md)
Guide the user through granting `WRITE_SETTINGS`, `POST_NOTIFICATIONS` and the battery-optimization
exemption, and explain what each one is for.
*Covers: first-run onboarding, per-permission explain/request/retry, degraded-mode messaging when a
permission is missing, and detecting devices without a light sensor.*

### [F7 — Status Dashboard & Service Control](./Feature-StatusDashboard/Feature-StatusDashboard.md)
Show live lux, the currently applied brightness, the active offset/response settings and the service
state, and let the user start and stop the controller.
*Covers: start/stop control, live readouts, clear indication of whether the app is currently allowed
to change brightness, and confirmation that stopping really stops all changes.*

## Key Decisions

| # | Decision |
|---|---|
| D1 | Brightness is controlled through the system brightness setting (`WRITE_SETTINGS`); no overlay or screen filter. |
| D2 | Minimum **and** target Android version are both Android 16 (API 36). |
| D3 | Stopping the controller restores the brightness that was in effect before it was started. |
| D4 | The app never changes the device's built-in adaptive brightness itself; it only explains the conflict and links to the system setting. |
| D5 | The controller starts automatically after a device reboot if it was running before the reboot. |
| D6 | The offset is symmetric and expressed as a percentage of the brightness span, from −50 % to +50 % in 1 % steps. |
| D7 | Response speed is a single control with a sensible default — no separate smoothing/damping controls. |
| D8 | Setup is a skippable checklist that doubles as the permanent "what is still missing" view. |
| D9 | The offset and response-speed controls live on a dedicated settings screen; the dashboard shows live values and start/stop. |
| D10 | The dashboard shows ambient light as a numeric lux value plus a qualitative description of the light level. |
| D11 | The battery-optimization exemption is recommended but **advisory** — it never blocks starting the controller. |
| D12 | Every completed implementation task is committed to git as its own commit (see [Development Workflow](#development-workflow)). |

## Development Workflow

- **One commit per completed task.** As soon as a task's stated verification passes, stage the
  changes (`git add -A`) and commit them before starting the next task. Never batch several tasks
  into one commit, and never commit a task whose verification has not passed.
- **`.gitignore` must keep build output out of the repository:** `build/`, `.gradle/`,
  `local.properties`, `.idea/`, `*.iml`. `local.properties` is machine-local (it holds `sdk.dir`)
  and must never be committed.
- **Commit messages follow Conventional Commits:** `<type>(<scope>): <summary>`, where `<scope>` is
  the feature slug or module (`build`, `core`, `sensor`, `brightness`, `offset`, `tuning`,
  `service`, `setup`, `ui`). The body lists what changed, the task ID, and the exact verification
  command that was run.

- **Task progress is recorded in the feature document**, not only in the commit: flip the task from
  `- [ ]` to `- [x]` and append a short `*Done:` note with the commit SHA and the verification
  result. The feature documents are the source of truth for what is complete.

## Non-Functional Requirements

- **Battery:** the control loop must not be a wakelock; it must run only while the screen is on and
  must idle the sensor when the service is stopped.
- **Reliability:** brightness changes must only ever be applied by the running service; stopping the
  service must leave no residual behaviour.
- **Privacy:** no data leaves the device; no network permission is required by the core feature.
- **Localisation:** UI strings in English and German.

## Dependencies & Risks

| Item | Impact | Notes |
|---|---|---|
| `WRITE_SETTINGS` is a special permission | High | User must grant it via a system Settings screen; cannot be granted by a dialog |
| OEM battery managers (e.g. aggressive "app sleep") | High | Mitigated by foreground service + battery-optimization exemption guidance |
| Light sensor absent or unreliable on some devices | Medium | Must be detected and reported clearly (F6) |
| Android 16 only | Medium | Deliberate scope decision; excludes the majority of installed devices |
| Changing system brightness affects global device state | Medium | Must be reversible/restorable and clearly communicated to the user |

## Platform Verification (Android 16 / API 36)

Every platform-specific assumption in the feature designs was checked against the official Android 16
documentation. All items are **closed**; only the Play Console review of the foreground-service subtype
depends on something outside the documentation.

| Topic | Verified fact | Impact |
|---|---|---|
| Foreground service type | `specialUse` is declared as `android:foregroundServiceType="specialUse"`, requires the `FOREGROUND_SERVICE_SPECIAL_USE` permission and the `ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE` constant, and has **no runtime prerequisites**. The subtype is declared with `<property android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE" android:value="…"/>`; the value is free-form but is reviewed in the Google Play Console. | F5 — the chosen type is correct; the property value must justify why no other type fits |
| Auto-start after reboot | `ACTION_BOOT_COMPLETED` is an allowed exemption from background-start restrictions. For apps targeting API 35+, a boot receiver may not launch `dataSync`, `camera`, `mediaPlayback`, `phoneCall`, `mediaProjection` or `microphone`; `specialUse` is **not** restricted. `ACTION_LOCKED_BOOT_COMPLETED` is unusable because the persisted state is read from DataStore. | F5 — D5 (auto-start after reboot) is achievable |
| Notification permission | `POST_NOTIFICATIONS` is **not** required to launch a foreground service, but a notification must still be posted. If the user denies it, that notification is invisible in the drawer while the service stays visible in the system Task Manager. | F5/F6 — denial is advisory; the stop affordance moves to the Task Manager |
| `WRITE_SETTINGS` grant | `Settings.ACTION_MANAGE_WRITE_SETTINGS` still accepts the `package:` data URI for the per-app screen, and `Settings.System.canWrite()` is still the documented check. The platform explicitly warns that a matching activity may not exist. | F6 — the fallback chain stays mandatory |
| Predictive back | For targetSdk 36 on Android 16+, predictive back animations are enabled by default and `onBackPressed()` / `KEYCODE_BACK` are no longer delivered. | F7 — use `NavHost` + Compose `BackHandler`; never override `onBackPressed()` |
| Edge-to-edge | `windowOptOutEdgeToEdgeEnforcement` is ignored for apps targeting API 36. | F7 — both screens must be inset-aware |
| UI artifacts | `gradle/libs.versions.toml` (created by F1/T0) is the single source of truth for versions. T0 pinned the Google-validated set for `compileSdk = 36`: AGP 9.0.1, Kotlin 2.3.20, Gradle 9.1.0, `compose-bom 2026.03.01`, `activity-compose 1.13.0`, `core-ktx 1.18.0`, `lifecycle-runtime-ktx 2.10.0`. Latest known stable for the artifacts still to be added: `navigation-compose 2.10.2` (type-safe routes), `lifecycle-runtime-compose 2.11.0`, `datastore-preferences 1.2.1`. | F3/F4/F7 — add new versions **through the catalog only**, never hard-coded in a module build file, and keep the build green |

**Open external dependency:** the `specialUse` subtype declaration is reviewed by Google Play at
submission. This is the only item that cannot be closed by the Android documentation alone.

## Glossary

| Term | Meaning |
|---|---|
| **Lux** | SI unit of illuminance; the value reported by the ambient light sensor |
| **Offset** | A constant added to the computed brightness value before it is applied |
| **Response speed** | How quickly brightness follows a change in ambient light (smoothing/damping) |
| **Foreground service** | An Android service with a persistent, user-visible notification, which the system is reluctant to kill |
