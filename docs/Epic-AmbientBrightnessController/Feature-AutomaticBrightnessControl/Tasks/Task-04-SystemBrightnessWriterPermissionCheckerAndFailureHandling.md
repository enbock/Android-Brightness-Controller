# T4 — System brightness writer, permission checker and failure handling

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-AutomaticBrightnessControl.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T4 — System brightness writer, permission checker and failure handling**: implement
  `AndroidSystemBrightnessClient`, `AndroidDeviceBrightnessRangeClient`,
  `AndroidWriteSettingsPermissionChecker` and the `Dummy*` variants in `:infrastructure`; surface
  typed `Failed` results without crashing. T5 owns consuming the failure, updating controller
  status and ceasing subsequent writes. For range discovery, do not use
  `Settings.System.SCREEN_BRIGHTNESS_MAX` or a `PowerManager` maximum API: neither exists on the
  verified SDK 36 public surface. Confine a runtime lookup of Android's
  `config_screenBrightnessSettingMaximum` integer resource to `AndroidDeviceBrightnessRangeClient`;
  use `Resources.getIdentifier(..., "integer", "android")`, then `getInteger`, accepting only a
  value `>= min`. Return `null` if the resource is missing, unreadable, or invalid, allowing the
  existing core resolver's 1–255 fallback to apply. For permission observation, own the AppOps
  watcher inside a collected flow and unregister it in `awaitClose`; ensure app composition supplies
  the app scope, and controller stop cancels its active collection. Recheck `Settings.System.canWrite`
  immediately before each write to close callback/revocation races. Treat `putInt == false`,
  `SecurityException`, and `IllegalArgumentException` as `Failed`; do not catch-and-hide arbitrary
  throwables. Do not implement controller/use-case failure-stop orchestration in T4; T5 must
  surface the failure and cease write attempts until explicitly restarted/re-authorized.
  *Files: `:infrastructure/src/main/kotlin/com/ambientbrightness/infrastructure/**`.*
  *Verify: resource lookup missing/invalid/valid cases and the `max >= min` invariant; permission
  watcher registration/unregistration and grant refresh; instrumented write-then-read-back; and
  unit-tested failure path (AC2.4) with a fake client.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `cdf7689e8f8efaa483c85fcd3297485d8987abbd`.*

- Added `AndroidSystemBrightnessClient`, using the application `ContentResolver` to read only
  `Settings.System.SCREEN_BRIGHTNESS` and write only that setting. It checks the live
  `Settings.System.canWrite(context)` before each write and returns `Applied` only when `putInt`
  succeeds. Permission denial, a false `putInt` result, `SecurityException`, and
  `IllegalArgumentException` become nonempty diagnostic `Failed` results; expected read failures
  return `null`.
- Added `AndroidDeviceBrightnessRangeClient`, which reads the SDK-36 system integer resource
  `config_screenBrightnessSettingMaximum` via `Resources.getIdentifier/getInteger`, maps the
  minimum to `BrightnessValue(1)`, validates `max >= min`, and returns `null` for missing,
  unreadable or invalid resource values. The existing core resolver then uses its `1..255`
  fallback. The API-specific design was corrected after inspection of the actual `android-36`
  `android.jar`: neither `Settings.System.SCREEN_BRIGHTNESS_MAX` nor a public `PowerManager`
  maximum API exists there. The guarded system-resource lookup is intentionally the only Android
  range probe.
- Added `AndroidWriteSettingsPermissionChecker`, whose app-scoped flow starts with an accurate
  `canWrite` value, watches `OPSTR_WRITE_SETTINGS` only while collected, refreshes on AppOps changes,
  and unregisters the watcher on cancellation. Added deterministic dummy variants for the system
  brightness, device range and permission clients. Declared `WRITE_SETTINGS` in the app manifest.
- Files changed:
  `infrastructure/src/main/kotlin/com/ambientbrightness/infrastructure/AutomaticBrightnessControl/SystemBrightnessClient/Android/AndroidSystemBrightnessClient.kt`,
  `infrastructure/src/main/kotlin/com/ambientbrightness/infrastructure/AutomaticBrightnessControl/DeviceBrightnessRangeClient/Android/AndroidDeviceBrightnessRangeClient.kt`,
  `infrastructure/src/main/kotlin/com/ambientbrightness/infrastructure/AutomaticBrightnessControl/WriteSettingsPermissionChecker/Android/AndroidWriteSettingsPermissionChecker.kt`,
  `infrastructure/src/main/kotlin/com/ambientbrightness/infrastructure/AutomaticBrightnessControl/SystemBrightnessClient/Dummy/DummySystemBrightnessClient.kt`,
  `infrastructure/src/main/kotlin/com/ambientbrightness/infrastructure/AutomaticBrightnessControl/DeviceBrightnessRangeClient/Dummy/DummyDeviceBrightnessRangeClient.kt`,
  `infrastructure/src/main/kotlin/com/ambientbrightness/infrastructure/AutomaticBrightnessControl/WriteSettingsPermissionChecker/Dummy/DummyWriteSettingsPermissionChecker.kt`
  (regrouped by T8),
  `infrastructure/src/test/kotlin/com/ambientbrightness/infrastructure/AndroidSystemBrightnessClientTest.kt`,
  `infrastructure/src/test/kotlin/com/ambientbrightness/infrastructure/AndroidDeviceBrightnessRangeClientTest.kt`,
  `infrastructure/src/test/kotlin/com/ambientbrightness/infrastructure/DummyBrightnessAdaptersTest.kt`,
  `app/src/androidTest/kotlin/com/ambientbrightness/app/AndroidBrightnessAdaptersTest.kt`,
  and `app/src/main/AndroidManifest.xml`.
- Focused tests: infra JVM 26/26 passed (including dummy behavior, typed writer failure conversion,
  and range validity); core JVM 80/80 passed. Required builds passed:
  `:infrastructure:test :infrastructure:assembleDebug :infrastructure:assembleDebugAndroidTest`
  and `:core:test :infrastructure:test :app:assembleDebug` (all with `--no-daemon`).
- Emulator: API 36 `medium_phone` (`emulator-5554`). `:infrastructure:connectedDebugAndroidTest`
  passed 1/1 (`AndroidAmbientLightSensorClientTest.lightEventReachesCallback`).
  The actual brightness tests are in `app/src/androidTest/.../AndroidBrightnessAdaptersTest.kt`
  because the infrastructure library instrumentation target is
  `com.ambientbrightness.infrastructure.test`, not the required app package. After
  `adb shell appops set com.ambientbrightness.app WRITE_SETTINGS allow`,
  `:app:connectedDebugAndroidTest` passed 5/5: the existing two component tests plus
  `deviceRangeIsValidWhenTheSystemResourceIsAvailable`,
  `systemWriteReadsBackAndRestoresBrightness`, and
  `permissionStateStartsWithTheLiveGrantAndObservationCanStop` (the latter toggled AppOps
  `ignore`/`allow` and observed both flow updates). The discovered maximum was 255.
  These brightness tests must stay in `:app`: the `:infrastructure` library manifest declares no
  `WRITE_SETTINGS` and the adb `appops` grant is bound to the app package, so relocating them into the
  library androidTest would make them skip (`assumeTrue`) or fail on a device, with no compile-time signal.
- The write/read-back test saved brightness `102`, wrote a nearby distinct value, read it back and
  restored `102` in `finally`; the outer adb cleanup also restored system brightness to `102`,
  reset `WRITE_SETTINGS` to `default` while the app was installed, verified that state, and removed
  the app package to restore its initially absent installation state.
- **Failure boundary:** T4 adapters return `Failed`; they do not orchestrate controller status or
  stop subsequent writes. T5 must consume `Failed`, set `BrightnessStatus.writeState/writeError`,
  and cease future writes. No retry-stop behavior is claimed before T5.
