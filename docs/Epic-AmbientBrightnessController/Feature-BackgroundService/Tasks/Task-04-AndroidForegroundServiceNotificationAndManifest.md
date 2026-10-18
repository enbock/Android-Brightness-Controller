# T4 — Android foreground service, notification and manifest

**Completion:** Not completed — reported bug follow-up required

## Dependencies

See the [Suggested Implementation Order](../Feature-BackgroundService.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T4 — Android foreground service, notification and manifest**: implement
  `ForegroundBrightnessService`, `AndroidBrightnessNotificationFactory` and the manifest entries
  (`FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_SPECIAL_USE`, `POST_NOTIFICATIONS`,
  `foregroundServiceType="specialUse"` + subtype `<property>`, notification channel). Handle a `null`
  restart intent and `POST_NOTIFICATIONS` denial without crashing.
  *Files: `:infrastructure/src/main/kotlin/com/ambientbrightness/infrastructure/**`, `:app/src/main/AndroidManifest.xml`.*
  *Verify: instrumented test — service is foreground, notification is ongoing, stop action works;
  AC1.1, AC4.3 (not swipe-dismissible).*
  *Depends on: T1, T2.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `704485d817d11890a5745be3f2622525b0350850`.*

- Added the only Android `Service` in the project,
  `ForegroundBrightnessService : Service(), ForegroundServiceHost`
  (`:infrastructure/.../BackgroundService/ForegroundServiceHost/`). It is the platform host for T3's
  `BackgroundServiceUseCase`, so `:core` keeps driving the notification through the
  `ForegroundServiceHost` interface and never touches Android:
  - `onCreate` creates one `NotificationChannel` (`AndroidBrightnessNotificationFactory.CHANNEL_ID`)
    at `IMPORTANCE_LOW` (no sound/vibration); updates reuse one stable id
    (`AndroidBrightnessNotificationFactory.NOTIFICATION_ID`).
  - `onStartCommand` promotes the service with
    `ServiceCompat.startForeground(this, NOTIFICATION_ID, notification, ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE)`
    promptly — for a `null` restart intent (`START_STICKY` after a system kill) it falls back to T2's
    `BackgroundNotificationBuilder.build(BackgroundServiceState.Starting)` content instead of a
    duplicated string — and returns `START_STICKY`.
  - `startForeground(content)` / `updateNotification(content)` / `stopForeground()` are the interface
    methods the use case calls; `updateNotification` posts only when already in the foreground and
    tolerates a denied `POST_NOTIFICATIONS` (`NotificationManagerCompat.areNotificationsEnabled()`
    guard plus a `SecurityException` guard), so the FGS still runs and nothing crashes (API 36 spec:
    the service does not need the permission to start, only to show the notice).
  - `onBind` returns a `LocalBinder` exposing the instance, which is how the composition root (T6) and
    the instrumented test obtain the `ForegroundServiceHost` without constructing the framework-owned
    service.
- Added `AndroidBrightnessNotificationFactory` (`.../BackgroundService/AndroidBrightnessNotificationFactory/`):
  `BackgroundNotificationContent` → `NotificationCompat` with `setOngoing(content.ongoing)`,
  `setAutoCancel(false)`, a content intent that opens the app's launcher activity
  (`PackageManager.getLaunchIntentForPackage` → `MainActivity`, resolved without an `:app` import,
  which the dependency rule forbids), and exactly one **Stop** action when `content.showStopAction`.
- **Stop action path:** the Stop action is a `PendingIntent.getService` carrying
  `ForegroundBrightnessService.ACTION_STOP`. Tapping it re-enters `onStartCommand(ACTION_STOP)`,
  which calls `stopRequestListener` — a seam set by the composition root (T6) to
  `BrightnessBackgroundService.stop()` — so the command reaches `BackgroundServiceUseCase.stop()`
  (F2 stop → snapshot restore → `host.stopForeground()`, which removes the notification and calls
  `stopSelf()`). When the listener is not wired (dev), the service stops itself without restoring.
- Added `DummyForegroundServiceHost` (same `ForegroundServiceHost/` folder) so `:app` can compose the
  lifecycle without a device; it tracks the current content and foreground flag in memory.
- Manifest (`app/src/main/AndroidManifest.xml`): added `FOREGROUND_SERVICE`,
  `FOREGROUND_SERVICE_SPECIAL_USE` and `POST_NOTIFICATIONS`, and declared
  `ForegroundBrightnessService` with `android:exported="false"` and
  `android:foregroundServiceType="specialUse"` plus the
  `android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE` `<property>` child (value
  "ambient light based screen brightness control"), per *Android Platform Specifics*. Verified in the
  merged manifest of `:app:assembleDebug`.
- Added the instrumented test
  `infrastructure/src/androidTest/kotlin/com/ambientbrightness/infrastructure/ForegroundBrightnessServiceTest.kt`
  (JUnit 4 / `AndroidJUnit4`): starts the service, binds through the `LocalBinder`, asserts
  `isInForeground`, asserts the built notification is `FLAG_ONGOING_EVENT`, is not
  `FLAG_AUTO_CANCEL` and carries exactly one "Stop" action, and (when notifications are visible)
  asserts the same on the posted notification. Device-dependent assertions use `assumeTrue`, matching
  the module's existing smoke tests; here it only has to compile.
  *(Superseded: the test was later relocated to `:app`'s androidTest because it never actually ran from
  here — see the Correction note below.)*
- `:infrastructure` gained `implementation(libs.androidx.core.ktx)` (for `NotificationCompat`,
  `ServiceCompat`, `ContextCompat`, `NotificationManagerCompat`); the instrumented test deliberately
  uses only platform APIs so it does not depend on the androidx test classpath. No `:core` change:
  T1's `BackgroundNotificationContent`/`ForegroundServiceHost` and T2's builder were consumed as-is.
- Verification (from the repo root):
  `$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot'` then
  `.\gradlew.bat :infrastructure:test :infrastructure:assembleDebugAndroidTest :app:assembleDebug --no-daemon --console=plain`
  → **BUILD SUCCESSFUL** (92 actionable tasks); `:infrastructure` unit tests 8 classes / 32 tests,
  0 failures; `:infrastructure:assembleDebugAndroidTest` and `:app:assembleDebug` compile/link.

#### Correction (after the on-device run)

*Fix commit `1cd07b8`.*

- **The T4 instrumented test never actually ran.** It was placed in `:infrastructure`'s androidTest, but
  the `<service>` is declared in the **app** manifest and `infrastructure/src/main/AndroidManifest.xml`
  is empty, so the infrastructure test target
  (`com.ambientbrightness.infrastructure.test`) has no such component. On the emulator every attempt
  failed in `ActivityManager` with *"Unable to start service … not found"*, the `assumeTrue` guard then
  skipped the test, and the suite reported green while proving nothing. This is exactly the failure mode
  the architecture document records for the F2 adapter tests, which had to move to `:app` for the same
  reason.
- The test was **relocated** to `app/src/androidTest/kotlin/com/ambientbrightness/app/ForegroundBrightnessServiceTest.kt`
  (same class, same assertions, new package). In `:app`'s androidTest the app manifest is in effect, so
  the service exists and the test executes for real.
- Verified on the `medium_phone` AVD (Android 16 / API 36):
  `:infrastructure:connectedDebugAndroidTest` → **6 tests, 0 failures, 0 skipped**;
  `:app:connectedDebugAndroidTest` → **16 tests, 0 failures, 0 skipped** (the class moved between the two
  suites, and *no* test is skipped any more). This is the first run in which any F5 instrumented
  assertion was actually executed.
- **Precondition for a green on-device run:** the F2 adapter tests need
  `adb shell appops set com.ambientbrightness.app WRITE_SETTINGS allow`. The connected task reinstalls
  the app, which discards a previously set appop, so the grant must be applied *while the run is in
  progress* — the F2 document already records the same finding.

## Follow-up — sticky restart lifecycle handoff

- [ ] Add an explicit, idempotent handoff from `ForegroundBrightnessService.onStartCommand()` on a
  sticky/null-intent recreation to the app/core lifecycle recovery path. The current implementation
  only promotes a `Starting` notification and returns `START_STICKY`; notification recreation must
  not be treated as proof that the controller resumed.
- [ ] Keep foreground promotion within Android's startup window, distinguish a genuine platform
  restart from a normal duplicate start, and report failure to bind/recover so the core/dashboard
  cannot claim Running based solely on the notification.
- **Dependencies:** F5 T6 restart reconciliation and persistent `ControllerStateStore`; coordinate the
  event contract with the app composition/bridge without adding Android dependencies to `:core`.
- **Size:** M.
- **Verification:** platform-facing tests cover null-intent service recreation with both
  `desiredRunning=true` and false through the recovery contract; verify one foreground promotion,
  idempotent repeated callbacks, and no controller action when false. Run on-device/emulator as part
  of T6's process-recreation scenario.
