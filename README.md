# Ambient Brightness Controller

An Android app that reads the device's ambient light sensor and adjusts the **system screen
brightness** in real time, so the display stays comfortably readable in any lighting condition.

Stock adaptive brightness reacts slowly and undershoots in bright light. This app instead runs a
continuous sensor-driven control loop as a foreground service, keeps working in the background, and
lets you fine-tune the result with a manual offset and a configurable response speed.

---

## Features

| # | Feature | What it does |
|---|---|---|
| F1 | Ambient light sensing | Reads the light sensor continuously and exposes a stabilised, throttled lux stream |
| F2 | Automatic brightness control | Maps lux to a system brightness value and applies it via `WRITE_SETTINGS` |
| F3 | Manual brightness offset | Shifts the computed brightness by −50 % … +50 %, persisted across restarts |
| F4 | Response tuning | One slider that controls how quickly the screen follows a light change |
| F5 | Background service | A foreground service with a persistent notification that survives system/OEM kills and restarts after reboot |
| F6 | Permissions & setup flow | A skippable checklist that explains and drives every required permission, and reports what is blocking |
| F7 | Status dashboard | Live lux + brightness readout, start/stop control, settings screen for offset and response speed |

The dashboard shows the current reading and applied brightness, the active offset/response speed, and
the setup state; from there you start and stop the controller and reach the settings screen.

---

## Requirements

| | |
|---|---|
| **Device** | **Android 16 (API 36) or newer** — `minSdk = targetSdk = compileSdk = 36` |
| **JDK** | 21 (the build targets `jvmToolchain(21)`) |
| **Android SDK** | Platform `android-36` and build-tools `36.0.0` |
| **Gradle** | 9.1.0 (the wrapper is committed — always use `gradlew`) |

> **Note:** the minimum SDK is deliberately API 36. The app **cannot** be installed on Android 15 or
> older devices. This is an intentional scope decision of the project (see
> [`docs/Epic-AmbientBrightnessController/Epic-AmbientBrightnessController.md`](docs/Epic-AmbientBrightnessController/Epic-AmbientBrightnessController.md), decision D2).

The toolchain versions (AGP, Kotlin, Compose BOM) are pinned in
[`gradle/libs.versions.toml`](gradle/libs.versions.toml), which is the single source of truth.

---

## Build

```powershell
# From the repository root. Set JAVA_HOME in the same shell if it is not
# available to the freshly spawned process.
$env:JAVA_HOME = '<path to your JDK 21>'

.\gradlew.bat :app:assembleDebug
```

The APK is written to `app/build/outputs/apk/debug/app-debug.apk`.

A **release** build (`:app:assembleRelease`) currently produces an *unsigned* APK, because no release
signing configuration is committed. Sign it with your own keystore before distributing it. Debug APKs
are signed with the standard Android debug keystore and are directly installable.

## Install on a device

**Option A — USB / ADB:**

```powershell
adb install -r app\build\outputs\apk\debug\app-debug.apk
# or simply:
.\gradlew.bat :app:installDebug
```

**Option B — sideload:** copy `app-debug.apk` (or a pre-built copy from `dist/`) to the phone and open
it. You may have to allow "Install unknown apps" for the app you open it from.

### First-run: grant "Modify system settings"

`WRITE_SETTINGS` is a **special permission** — it has no runtime dialog. The app's setup checklist
opens the correct system screen for you (tap *Fix* next to the brightness permission), or you can grant
it directly:

```powershell
adb shell appops set com.ambientbrightness.app WRITE_SETTINGS allow
```

> The grant is discarded when the app is reinstalled (including by `connectedAndroidTest`), so
> re-apply it after every reinstall.

---

## Permissions

| Permission | Why |
|---|---|
| `WRITE_SETTINGS` | The only way to change `Settings.System.SCREEN_BRIGHTNESS`. Granted via a system settings screen, not a dialog |
| `FOREGROUND_SERVICE` + `FOREGROUND_SERVICE_SPECIAL_USE` | Keeps the control loop alive while you use other apps |
| `POST_NOTIFICATIONS` | Shows the persistent notification with the stop action. Advisory: denial keeps the service running but hides the notification |
| `RECEIVE_BOOT_COMPLETED` | Restarts the controller after a reboot, but **only** if it was running before |

No network permission is requested — the app has no backend and no analytics. Nothing leaves the device.

---

## Architecture

The project follows clean architecture with dependencies pointing inwards:

```
app/             Composition root, Android lifecycles, Jetpack Compose UI (F7)
infrastructure/  Android adapters implementing the core interfaces (+ Dummy variants)
core/            Pure Kotlin domain: entities, interfaces, use cases and services
```

Rules enforced by tests:

- **`core` is Android-free** — no `android.*` or `androidx.datastore` import; persistence goes through
  the `SettingsStore` boundary.
- **Folder == package** — a file's package must match its directory
  (`.../core/<Responsibility>/<Type>.kt` ↔ `com.ambientbrightness.core.<Responsibility>`).
- **Composition-root isolation** — only `app/.../Composition/**` imports `infrastructure`.

Every interface is owned by `core` and implemented in `infrastructure`; the composition root wires the
concrete implementations and injects them inwards. See
[`docs/Architecture-StructureAlignment.md`](docs/Architecture-StructureAlignment.md) for the full
rationale.

---

## Tests

| Suite | What it covers |
|---|---|
| `:core:test` (JUnit 5) | Domain rules: mapping curves, stabiliser, offset, response tuning, lifecycle orchestration, setup aggregation, dashboard state mapping |
| `:infrastructure:test` (JUnit 4) | Dummy adapters, folder/package guard, brightness adapter helpers |
| `:app:testDebugUnitTest` (JUnit 4) | ViewModels, collectors, navigation decisions, end-to-end graphs over Dummy adapters |
| `connectedDebugAndroidTest` | Real service/notification, DataStore round-trip, Compose screens, app smoke test |

```powershell
.\gradlew.bat :core:test :infrastructure:test :app:testDebugUnitTest
.\gradlew.bat :connectedDebugAndroidTest          # needs a device/emulator
```

Compose and service instrumented tests need an emulator. One caveat: the adapter tests that write
`SCREEN_BRIGHTNESS` require the `WRITE_SETTINGS` appop to be granted **while the test run is in
progress**, because the connected task reinstalls the app:

```powershell
# in one shell, keep re-applying the grant
1..150 | ForEach-Object { adb shell appops set com.ambientbrightness.app WRITE_SETTINGS allow; Start-Sleep 2 }
# then in another shell
.\gradlew.bat :connectedDebugAndroidTest
```

---

## Documentation

The `docs/` folder is the design record: an epic plus one EPIC folder containing one folder per feature, each with a feature
document containing user stories, acceptance criteria and technical design. Individual implementation
tasks and their notes live in that feature's `Tasks/` folder.

- [EPIC — Ambient Brightness Controller](docs/Epic-AmbientBrightnessController/Epic-AmbientBrightnessController.md) — scope, goals, key decisions
- [Architecture structure alignment](docs/Architecture-StructureAlignment.md) — layering, conventions, guards
- [F1 — Ambient Light Sensing](docs/Epic-AmbientBrightnessController/Feature-AmbientLightSensing/Feature-AmbientLightSensing.md)
- [F2 — Automatic Brightness Control](docs/Epic-AmbientBrightnessController/Feature-AutomaticBrightnessControl/Feature-AutomaticBrightnessControl.md)
- [F3 — Manual Brightness Offset](docs/Epic-AmbientBrightnessController/Feature-ManualBrightnessOffset/Feature-ManualBrightnessOffset.md)
- [F4 — Response Tuning](docs/Epic-AmbientBrightnessController/Feature-ResponseTuning/Feature-ResponseTuning.md)
- [F5 — Background Service](docs/Epic-AmbientBrightnessController/Feature-BackgroundService/Feature-BackgroundService.md)
- [F6 — Permissions & Setup Flow](docs/Epic-AmbientBrightnessController/Feature-PermissionsSetupFlow/Feature-PermissionsSetupFlow.md)
- [F7 — Status Dashboard & Service Control](docs/Epic-AmbientBrightnessController/Feature-StatusDashboard/Feature-StatusDashboard.md)

---

## Known limitations

- **Android 16 only** (API 36). No backward compatibility.
- **Phone form factor only** — no tablet, foldable, TV or Wear layouts.
- **No force-stop survival** — no app can be restarted after a user-initiated force-stop.
- The app changes a **global device setting**. Stopping the controller restores the brightness that was
  in effect before it was started.
- The app does not touch the device's built-in adaptive brightness; it only points out the conflict.

---

## License

[MIT](LICENSE) © 2026 Endre Bock
