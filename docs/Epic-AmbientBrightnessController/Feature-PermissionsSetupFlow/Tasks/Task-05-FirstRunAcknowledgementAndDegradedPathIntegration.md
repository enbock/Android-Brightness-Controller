# T5 — First-run acknowledgement and degraded-path integration

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-PermissionsSetupFlow.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T5 — First-run acknowledgement and degraded-path integration**: persist `setupAcknowledged`/`lastShownAtMillis` on complete **and** skip (US1 AC3), expose the persisted flag so F7 shows the checklist once and thereafter keeps it as the permanent view, and integrate the degraded paths (no sensor → unsupported/Start not offered; runtime-revoked `WRITE_SETTINGS`; advisory notification warning) end to end with F5's state. *Files: `:core/.../SetupRequirementAggregator/**`, `:app/.../SetupFlow*/**`.* *Verify: US1 AC3, US3 AC1–AC3, US4 AC1–AC3 covered by tests.* (depends on: T4)

### Suggested Implementation Order

`T1 → (T2 ∥ T3) → T4 → T5`. T1 fixes the `AppSetupService` contract F7 consumes and gates Start on, so it lands first and is reviewed with F7. T2 and T3 are independent once T1 exists (T2 tests use fakes, T3 touches only infrastructure). T4 depends on both because navigation needs the model and the notification checker. T5 closes the loop for the first-run and degraded paths and must run against real adapters. Ship T1–T2 as a vertical model slice testable with fakes before any UI wiring.

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `c820024`.* **F6 is complete with this task.**

- **Explicit terminate pair (US1 AC3).** `AppSetupService` gained `completeSetup()` and `skipSetup()`, both documented at the boundary: completing and skipping persist the same acknowledgement (`setupAcknowledged = true` + `lastShownAtMillis`). `acknowledgeSetup()` is kept as the shared persistence path the pair funnels into, so existing callers keep working. Only the two facts from the design are stored; nothing else changed in the store contract.
- **Persisted flag exposed (US1 AC3).** `AppSetupService` now exposes `setupAcknowledged: StateFlow<Boolean>`, seeded from `SetupStateStore.setupAcknowledged` when the aggregator is built and flipped to `true` on complete/skip/acknowledge. This is the signal F7 uses to decide between the first-run view and the permanent "what is missing" view; a fresh aggregator over the same store starts acknowledged.
- **F5 degraded-path integration (US4 AC2).** `SetupRequirementAggregator` takes an optional, nullable `backgroundService: BrightnessBackgroundService?` collaborator (the fifth source) and observes its `state`. `WriteSettingsRequirementTask.map(granted, controllerFailed)` forces `Missing(Revoked)` while F5 reports `BackgroundServiceState.Failed` — so a `WRITE_SETTINGS` revoked at runtime surfaces as needing re-granting and lands in `blockers` even before the checker re-reads the permission; the observed-satisfied memory is set, so a later `false` stays `Revoked`. Once the controller recovers to `Running`/`Starting` and the checker reports `true` again, the requirement returns to `Satisfied`.
- **Advisory notification warning observable (US4 AC3).** `AppSetupService` now exposes `notificationWarning: StateFlow<Boolean>`, derived by the aggregator as `Notifications` is `Missing` **and** F5 has been started (`Starting`/`Running`). It is advisory only: it is computed from the same `SetupStatus` but never contributes to `blockers`/`isReadyToStart`, so a missing `POST_NOTIFICATIONS` cannot block Start. With no `backgroundService` the warning stays `false`.
- **Wiring.** `Composition.AmbientBrightnessComponent` passes its existing `backgroundService` into the aggregator, so the degraded paths are integrated end to end through the real F5 service. `SetupFlowCollector` forwards the two new streams (`onSetupAcknowledged`, `onNotificationWarning`) as trailing, defaulted callbacks, keeping it a logic-free pass-through.
- **Tests — coverage by AC.** `:core` `SetupRequirementAggregatorTest` (JUnit 5, now 25 tests) names each AC: US1 AC3 (`AC1_3 - completing…`, `AC1_3 - skipping…`, `AC1_3 - a fresh aggregator…`, `AC1_3 - acknowledgeSetup keeps…`), US3 AC1 (`AC3_1 - a missing blocking requirement…`), US3 AC2 (`AC3_2 - the blockers name exactly…`), US3 AC3 (`AC3_3 - all requirements satisfied…`), US4 AC1 (`AC4_1 - no sensor reports unsupported…`), US4 AC2 (`AC4_2 - an F5 failed controller…`, `AC4_2 - the revoked report survives…`), US4 AC3 (`AC4_3 - the notification warning is observable…`, `AC4_3 - …stays off when the grant is present`). New test double `FakeBrightnessBackgroundService` (settable state) in `SetupFlowFakes.kt`. `:app` (JUnit 4) `AmbientBrightnessSetupFlowTest` verifies the same ACs end to end over the real composition + real F5 lifecycle: US1 AC3 (complete/skip persist, a fresh component starts acknowledged), US4 AC2 (a failing write → F5 `Failed` → `Missing(Revoked)` + in `blockers`), US4 AC3 (notifications missing while `Running` → `notificationWarning` true, Start still enabled). `SetupLifecycleObserverTest`'s recording service was extended for the new interface members.
- Verification (from the repo root): `.\gradlew.bat :core:test :infrastructure:test :app:testDebugUnitTest :app:assembleDebug --no-daemon --console=plain` → BUILD SUCCESSFUL, 0 failures; `:core` 33 suites / 237 tests (T5 +11), `:infrastructure` 13 / 48, `:app` 6 / 20 (T5 adds `AmbientBrightnessSetupFlowTest`, 4 tests). `git grep "com\.ambientbrightness\.infrastructure\." -- app/src/main/kotlin` still matches only `Composition/**`; no `android.*`/`androidx.*` import was added to `:core`.
