# T5 — Settings screen, navigation and reset affordance

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-StatusDashboard.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T5 — Settings screen, navigation and reset affordance**: build `SettingsScreen`
  (`OffsetControl` Slider −50…+50 step 1, `ResponseSpeedControl` Slider 0…100, reset affordance) and wire
  `DashboardNavigator` + `NavHost` (`Dashboard` ⇄ `Settings` type-safe routes) so the entry point opens
  pre-filled settings
  and back returns to the dashboard without losing state. *Files:
  `:app/.../StatusDashboard/View/SettingsScreen.kt`, `.../View/SubViews/OffsetControl.kt`,
  `.../View/SubViews/ResponseSpeedControl.kt`, `.../Navigator/DashboardNavigator.kt`,
  `:app/.../MainActivity.kt`.* *Verify: Compose test — navigation between the type-safe `Dashboard` and
  `Settings` routes, current values shown, slider calls the service.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `5d0edef463d43fcbd7e8fa90d490f8dbbab6736d`.*

- Added `View/SubViews/SliderSnapping.kt` — pure `snapOffsetPercent` / `snapResponseSpeedPercent` that
  round a continuous slider value and clamp it to `BrightnessOffsetRange` / `ResponseSpeed`. Material 3
  sliders emit every value a drag passes over, while `BrightnessOffset.of(...)` rejects anything off the
  1 % grid, so snapping here is what keeps a drag from throwing (US4, AC4.1).
- Added `View/SubViews/OffsetControl.kt` — a `Slider` over `−50..50` with `steps = 99` (100 one-percent
  intervals), the live percent, a `contentDescription` for TalkBack, and a reset button enabled only
  when `canResetOffset` (US3, AC3.1/AC3.3).
- Added `View/SubViews/ResponseSpeedControl.kt` — one `Slider` over `0..100` with localized
  Faster/Slower endpoint labels, the same snapping, and its own reset affordance.
- Added `View/SettingsScreen.kt` — an inset-aware Material 3 `Scaffold` with both controls, the
  offset-limited message when F3 reports `ClampedToMinimum`/`ClampedToMaximum` (US3, AC1.1) and a back
  affordance. `SettingsScreen` is stateless (renderable from a plain `SettingsUiState`);
  `SettingsRoute` collects `SettingsViewModel.uiState` with `collectAsStateWithLifecycle` and forwards
  the intents, so values are **pre-filled from the services** (US4, AC1.1).
- Added `Navigator/DashboardNavigator.kt` — `@Serializable` type-safe destinations
  (`DashboardDestination`, `SettingsDestination`) and a `DashboardNavHost` that wires `DashboardRoute`
  and `SettingsRoute`: the dashboard's entry point navigates to the settings screen and `popBackStack`
  returns with the dashboard ViewModel retained, so its state survives the round trip (US4, AC1.1).
  **Back is handled only by `NavHost`/Compose** — `onBackPressed()` is never overridden and no
  `enableOnBackInvokedCallback="false"` is set, so predictive back keeps working on API 36.
- Wired `MainActivity`: it now hosts `DashboardNavHost` instead of the F1 placeholder
  (`AmbientLightSensing/View/AmbientBrightnessView.kt` is left in place but no longer used, to avoid
  invalidating the paths F1's document records), forwards setup actions to F6's `SetupFlowNavigator`,
  and keeps the existing F1 collector lifecycle behaviour. Added
  `Composition/StatusDashboardViewModelFactories.kt` (the only place that builds the collector and both
  ViewModels from the app-scoped component), plus an additive read-only
  `AmbientBrightnessComponent.ambientLightService` accessor — the same instance F1's collector drives,
  so there is still exactly one ambient-light pipeline.
- **Dependency choice (deviation from the doc, within its own rule).** The design names
  `navigation-compose:2.10.2`, but that artifact (and the `lifecycle-*-compose:2.11.0` it pulls in)
  requires **AGP 9.1.0 and compileSdk 37**, which the verified toolchain (AGP 9.0.1 / compileSdk 36)
  cannot satisfy — `:app:checkDebugAarMetadata` fails. The design explicitly allows the catalogue to win
  and requires the build to stay green, so `navigation-compose` is pinned to **2.9.0** (type-safe routes
  are stable since 2.8.0) and the Kotlin serialization plugin (`org.jetbrains.kotlin.plugin.serialization`,
  version-ref'd to `kotlin = 2.3.20`) was added; `lifecycle-runtime-compose` stays on the catalog's
  `androidxLifecycle = 2.10.0`.
- Strings: settings-screen strings (title, headings, slider descriptions, reset labels, back, clamp
  messages, Faster/Slower) added to **both** `values/strings.xml` and `values-de/strings.xml`.
- Tests. `:app` JVM `SliderSnappingTest` (3): snapping to whole percent both directions, clamping to the
  offset bounds including ±Infinity, and snapping/clamping the response speed. `:app` androidTest
  `DashboardNavigationTest` (1, `createComposeRule`) drives the real `DashboardNavHost` over the real
  composition root with Dummy adapters: dashboard → settings, the pre-filled `Offset: -12%`, the
  response-speed slider, a slider drag writing `25` through the service, reset clearing it to `0`, and
  back returning to the dashboard. It is **compile-verified only** here (no device attached); the same
  behaviour is covered on the JVM by T3's `SettingsViewModelTest`.
- Verification (from the repo root):
  `$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot'` then
  `.\gradlew.bat :core:test :infrastructure:test :app:testDebugUnitTest :app:assembleDebug
  :app:assembleDebugAndroidTest --no-daemon --console=plain` → **BUILD SUCCESSFUL**, 0 failures;
  `:core` 33 classes / 237 tests, `:infrastructure` 13 / 48, `:app` 14 / 62. No `:core` change.
