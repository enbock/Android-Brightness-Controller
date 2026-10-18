# T2 — Response speed model, profile mapping, boundary service

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-ResponseTuning.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T2 — Response speed model, profile mapping, boundary service**: add `ResponseSpeed` (clamp,
  `DEFAULT = 50`), `ResponseProfile`, `ResponseProfileResolver` (fixed constants above) and the
  `ResponseTuningService` boundary with its `DefaultResponseTuningService(store, scope)`
  implementation, seeded from `SettingsStore.observe(RESPONSE_SPEED_PERCENT)` and persisting on
  `setResponseSpeed`, exposing `responseSpeed: StateFlow<ResponseSpeed>` for F7. The constructor is
  `(store: SettingsStore, scope: CoroutineScope)` — no `Clock` (see the T2 design resolution below).
  *Files: `:core/src/main/kotlin/com/ambientbrightness/core/ResponseTuning/**`,
  `:core/src/test/kotlin/com/ambientbrightness/core/testdoubles/FakeSettingsStore.kt`.* *Verify: unit
  tests for default-when-absent (AC4.2), clamped out-of-range values, derivation at `0/50/100`,
  reactive update after `setResponseSpeed` (AC1.4).*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `c286a396c9eca953e0edbfac53b831cbc3fe2b4b`.*

- `ResponseSpeed` is a data class with a private constructor and the `of(percent)` factory: the
  factory clamps into `FASTEST_PERCENT..SLOWEST_PERCENT` (`0..100`, both bounds public because the
  resolver interpolates over the range), and `DEFAULT = 50` stays the absent-case value. The private
  constructor keeps an unclamped `percent` unrepresentable; the class carries
  `@ConsistentCopyVisibility` so the generated `copy()` remains private (without it the compiler
  warns about the Kotlin 2.5 data-class visibility change).
- `ResponseProfile` is a plain data holder with exactly the three public properties
  (`reactionWindowMillis`, `debounceWindowMillis`, `maxStepFraction`) and no behaviour, as designed.
- `ResponseProfileResolver` is one concrete class in its own folder — no interface/`Default…` pair,
  because the mapping is pure, stateless and side-effect free, so there is nothing to substitute. It
  interpolates both windows linearly off `percent` and exposes the fixed constants publicly
  (`FASTEST_REACTION_WINDOW_MILLIS = 300`, `SLOWEST_REACTION_WINDOW_MILLIS = 3000`,
  `FASTEST_DEBOUNCE_WINDOW_MILLIS = 150`, `SLOWEST_DEBOUNCE_WINDOW_MILLIS = 600`,
  `MAX_STEP_FRACTION = 0.15f`) so T3/T4 reuse them instead of re-deriving them.
- `ResponseTuningService` (boundary) and `DefaultResponseTuningService` both live in `:core`,
  following the in-module precedent for platform-independent business logic (`AmbientLightService` +
  `DefaultAmbientLightService`, `HysteresisGate` + `DefaultHysteresisGate`): the implementation only
  depends on the injected `SettingsStore`, so it needs no Android type and `:infrastructure` gets
  nothing. The boundary stays exactly `responseSpeed: StateFlow<ResponseSpeed>` +
  `suspend fun setResponseSpeed(speed)`.
- Deviation (also recorded in the design resolution below): the implementation is
  `DefaultResponseTuningService(store: SettingsStore, scope: CoroutineScope)` and takes **no
  `Clock`**. A `StateFlow` needs a producer and `:app` must stay the only scope owner, so the scope
  is constructor-injected, mirroring the existing `AndroidDataStoreSettingsStore.create(context,
  scope)` and `BrightnessControllerUseCase(..., scope)`; the `Clock` was dropped because no T2 logic
  reads the wall clock (T3/T4 inject their own). Seeding is
  `store.observe(RESPONSE_SPEED_PERCENT).map { ResponseSpeed.of(it) }.stateIn(scope,
  SharingStarted.Eagerly, ResponseSpeed.of(DEFAULT))`, and `setResponseSpeed` writes to the store
  only — the store's emission updates the `StateFlow`, so there is no second (optimistic) source of
  truth.
- Deviation: `:core`'s first store-dependent tests add the test-source double `FakeSettingsStore`
  next to `FakeClock`, as T1 announced. It seeds through its constructor, stays reactive like both
  real stores and records `writes`, so a test can assert the persisted value and the value read back
  through `observe`. No production in-memory store was added to `:core`.
- Files added: `core/src/main/kotlin/com/ambientbrightness/core/ResponseTuning/ResponseSpeed.kt`,
  `.../core/ResponseTuning/ResponseProfile.kt`,
  `.../core/ResponseTuning/ResponseProfileResolver/ResponseProfileResolver.kt`,
  `.../core/ResponseTuning/ResponseTuningService/ResponseTuningService.kt`,
  `.../core/ResponseTuning/ResponseTuningService/DefaultResponseTuningService.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/testdoubles/FakeSettingsStore.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/ResponseTuning/ResponseSpeedTest.kt`,
  `.../core/ResponseTuning/ResponseProfileResolver/ResponseProfileResolverTest.kt`,
  `.../core/ResponseTuning/ResponseTuningService/DefaultResponseTuningServiceTest.kt`.
  Files changed: none — F1, F2, `SettingsStore`, the key names and the key defaults are untouched
  (`SettingsKeys.RESPONSE_SPEED_PERCENT` keeps T1's literal `50`), and neither `:infrastructure` nor
  `:app` is affected.
- Tests cover default when the key is absent (AC4.2), a stored value clamped into the range, in-range
  pass-through, the derivation at `0/50/100` (300/1650/3000 ms and 150/375/600 ms, `maxStepFraction`
  0.15 at every speed), monotonicity over all 101 percents, the live emission to a running collector
  after `setResponseSpeed` (AC1.4), a value persisted before construction seeding the flow and
  reaching a late collector, and the clamped write round-tripping through the store.
- Verification (from the repo root): `.\gradlew.bat :core:test --no-daemon --console=plain` →
  BUILD SUCCESSFUL, 22 test classes / 118 tests, 0 failures, 0 errors, no compiler warnings
  (19 classes / 104 tests before T2: +3 classes, +14 tests). `ArchitectureStructureTest`
  (folder/package + Android-import guard) is green, so the new files declare
  `com.ambientbrightness.core.ResponseTuning…` and `:core` imports no Android type. No other module
  was built because no other module changed.

### T2 — Design resolution (design only)


*Design only — the two under-specified points in the T2 text were resolved before implementation; no
acceptance criterion changes and the boundary surface does not grow.*

- **Gap 1 — who owns the scope feeding `responseSpeed`.** Decision: **option (a)**, a constructor
  scope. The boundary keeps exactly the two designed members:

  ```kotlin
  interface ResponseTuningService {
      val responseSpeed: StateFlow<ResponseSpeed>
      suspend fun setResponseSpeed(speed: ResponseSpeed)
  }
  ```

  The concrete implementation is
  `class DefaultResponseTuningService(store: SettingsStore, scope: CoroutineScope) :
  ResponseTuningService` — collection starts at construction, and the class creates no scope of its
  own, so `:app` (T5) remains the only scope owner and passes the existing `appScope`. This mirrors
  code already in the repo: `AndroidDataStoreSettingsStore.create(context, scope)` in
  `:infrastructure` and `BrightnessControllerUseCase(..., private val scope: CoroutineScope)` in
  `:core`. Seeding is a single declaration
  `store.observe(SettingsKeys.RESPONSE_SPEED_PERCENT).map(ResponseSpeed::of)
  .stateIn(scope, SharingStarted.Eagerly, ResponseSpeed.DEFAULT)`, so a value persisted before
  construction is picked up (both store implementations emit immediately), `responseSpeed` is always
  a valid `StateFlow`, and a steady default is exposed when the key is absent (AC4.2).
  `setResponseSpeed` does only `store.set(SettingsKeys.RESPONSE_SPEED_PERCENT, speed.percent)`; the
  new value reaches F7 through the store's own emission, so there is no second (optimistic) source of
  truth and the live update of AC1.4 falls out of the store contract.
  - *Rejected (b) `start(scope)`/`stop()`* (the `AmbientLightService` / `AutomaticBrightnessController`
    precedent): those own device resources that must be released deterministically, whereas this
    service only mirrors an app-scoped store; a lifecycle hook would force F7 to know when to
    `start`, keep the surface larger than designed, and expose a stuck default until it is called.
  - *Rejected (c) a cold `Flow`/`SharingStarted` boundary*: F7 is specified to receive a
    `StateFlow`, which `:app` must not have to build itself.
- **Gap 2 — the `Clock`.** Decision: **omit it**; `DefaultResponseTuningService` takes no `Clock`.
  Reading and writing a percent is not time-based, so a `Clock` parameter would be dead surface; T3
  and T4 still inject the `Clock` they need themselves. The design's "one injected `Clock` serves
  both — never `System.currentTimeMillis()` directly" invariant is about the two time-based pipeline
  components (`UserConfigurableStabiliser`, `ResponsiveBrightnessRamp`), which take the same `Clock`
  instance from `:app`; the service is not one of them. Recorded as a deliberate deviation from the
  earlier T2 wording.

Consequences for the developer: the service is one small class, so no `Task/` split is warranted. The
T2 `:core` unit test is the first store-dependent `:core` test (noted in T1) and therefore adds the
`:core` test-source double `FakeSettingsStore` under `core/src/test/.../core/testdoubles/`; because
the `stateIn` upstream never completes, the test must collect under `backgroundScope` (advancing with
`runCurrent()`/`advanceUntilIdle()`) rather than let `runTest` await it.
