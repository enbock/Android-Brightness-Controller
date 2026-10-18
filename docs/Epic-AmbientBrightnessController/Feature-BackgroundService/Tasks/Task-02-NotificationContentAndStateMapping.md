# T2 — Notification content and state mapping

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-BackgroundService.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T2 — Notification content and state mapping**: implement `BackgroundNotificationBuilder`
  (state → title/text, `appliedBrightness` from F2 `BrightnessStatus`, single Stop action, ongoing
  flag) as pure Kotlin. *Files: `:core/.../BackgroundNotificationBuilder/**`.*
  *Verify: unit tests per state; AC4.1 (state + applied brightness), AC4.2 (stop action).*
  *Depends on: T1.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `c395b04`.*

- Added `BackgroundNotificationBuilder` — a stateless, Android-free, side-effect-free class with a
  no-arg constructor (so F7/T4 can construct, inject or substitute it) exposing
  `fun build(state: BackgroundServiceState): BackgroundNotificationContent`.
- Mapping is an exhaustive `when` over all five variants (US4, AC4.1–AC4.3):
  - `Starting` → title "Ambient brightness control is starting", text "Starting the brightness
    controller...", `ongoing = true`, `showStopAction = true`.
  - `Running(brightness)` → title "Ambient brightness control is running", text
    "Applied brightness: N" from the state's F2 `BrightnessValue` (or "Waiting for the first light
    reading" when null), `ongoing = true`, `showStopAction = true`.
  - `Stopped` → title "Ambient brightness control is stopped", text "The brightness controller is
    stopped.", `ongoing = false`, `showStopAction = false`.
  - `StoppedBySystem` → title "Ambient brightness control is stopped", text "Android stopped the
    brightness controller. Open the app to start it again." (so a system stop is not mistaken for a
    user stop), `ongoing = false`, `showStopAction = false`.
  - `Failed(reason)` → title "Ambient brightness control stopped", text
    "The brightness controller stopped: <reason>", `ongoing = false`, `showStopAction = false`.
- The applied-brightness text lives in one documented private helper (`brightnessText`) and comes
  only from `state` — the builder never re-reads the system brightness setting, so it cannot
  disagree with the controller. `ongoing` is documented as the flag that makes the notification
  non-dismissible (AC4.3); the platform `setOngoing(true)` call stays in T4.
- Files added:
  `core/src/main/kotlin/com/ambientbrightness/core/BackgroundNotificationBuilder/BackgroundNotificationBuilder.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/BackgroundNotificationBuilder/BackgroundNotificationBuilderTest.kt`.
- Verification (from the repo root):
  `$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot'` then
  `.\gradlew.bat :core:test --no-daemon --console=plain`
  → **BUILD SUCCESSFUL**, 29 test classes / 190 tests, 0 failures, 0 errors, 0 skipped
  (`BackgroundNotificationBuilderTest` adds 9 tests covering every state, AC4.1, AC4.2 and the
  `ongoing` flag). `:core` gained **no** `android.`/`androidx.datastore` import.
