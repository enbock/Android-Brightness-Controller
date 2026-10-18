# T4 — Offset state service

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-ManualBrightnessOffset.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T4 — Offset state service**: implement `BrightnessOffsetStateHolder` backing
  `BrightnessOffsetService` — reactive `offset`/`application` `StateFlow`, injected shared
  `SettingsStore` (seed from `observe(BRIGHTNESS_OFFSET_PERCENT)`, restore at construction, persist on
  change), `setOffset`/`adjustBy`/`reset`, default `0`, out-of-range stored `Int` clamped via
  `BrightnessOffsetRange`.
  *Files: `:core/.../BrightnessOffsetStateHolder/**`.*
  *Verify: with the `:core` test-double settings store — restore-after-restart (AC4.1/4.2), default (AC4.3), reset
  (AC3.3), no-reading change (AC1.2), out-of-range seed clamps.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `3091690ba8cb4708e075bcff70fce937817ae4fe`.*

- Implemented `BrightnessOffsetStateHolder(store: SettingsStore, scope: CoroutineScope) :
  BrightnessOffsetService, OffsetApplicationSink` in `:core`, Android-free — the app-scoped single
  source of truth for the offset. `offset: StateFlow<BrightnessOffset>` derives from
  `store.observe(SettingsKeys.BRIGHTNESS_OFFSET_PERCENT)`, clamps the stored `Int` into
  `BrightnessOffsetRange.MIN..MAX` before `BrightnessOffset.of(...)` (defence in depth, since the
  shared store performs no domain validation) and is built with
  `stateIn(scope, SharingStarted.Eagerly, initialValue = BrightnessOffset.DEFAULT)` — the exact
  `DefaultResponseTuningService` pattern, so an absent key reports `0` immediately without a write
  (AC4.3) and a value already in the store is restored at construction (AC4.1/AC4.2).
- `setOffset`/`adjustBy`/`reset` only persist through the store; the store emission updates `offset`
  in turn. `adjustBy` computes `current + delta` in `Long`, coerces into
  `BrightnessOffsetRange.MIN..MAX` and stores the result, so a large delta saturates instead of
  throwing or overflowing; `reset()` delegates to `setOffset(BrightnessOffset.DEFAULT)` (AC3.3).
  Because the holder never depends on an ambient reading, a change is emitted even with no reading
  and `application` stays `NotEvaluated` (AC1.2).
- `application` is held as a private in-memory `MutableStateFlow` initialised to
  `BrightnessOffsetApplication.NotEvaluated` (it is not persisted and resets on process start),
  exposed read-only via `asStateFlow()` and updated by the `OffsetApplicationSink.publish(...)`
  write-side port, so F7 can surface the transform's clamp indicator (AC2.3).
- Files added:
  `core/src/main/kotlin/com/ambientbrightness/core/BrightnessOffsetStateHolder/BrightnessOffsetStateHolder.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/BrightnessOffsetStateHolder/BrightnessOffsetStateHolderTest.kt`.
- The tests reuse the existing `:core` test double `com.ambientbrightness.core.FakeSettingsStore` (no
  duplicate fake) and cover default-on-absent (AC4.3), restore of `25`/`-50` after a simulated
  restart (AC4.1/4.2), set/adjust/reset with saturation at both bounds (AC3.3), an out-of-range
  seeded `±999` clamping to `MAX`/`MIN`, a `setOffset` change with no ambient reading still updating
  `offset` while `application` stays `NotEvaluated` (AC1.2), and `publish(...)` surfacing through
  `application`.
- No deviation from the design. `:core` gained no `android.`/`androidx.datastore` import (the
  `ArchitectureStructureTest` guard stays green); `:core`'s JUnit 5 style is used, and the test class
  carries `@OptIn(ExperimentalCoroutinesApi::class)` for `runCurrent`, matching the existing tests.
- Verification (from the repo root):
  `$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot'` then
  `.\gradlew.bat :core:test --no-daemon --console=plain` → BUILD SUCCESSFUL, 27 test classes /
  168 tests, 0 failures (157 before T4: +1 class / +11 tests, all in `BrightnessOffsetStateHolderTest`).
