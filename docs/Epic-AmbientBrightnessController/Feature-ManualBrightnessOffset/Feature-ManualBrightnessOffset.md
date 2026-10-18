# F3 — Manual Brightness Offset

| Field | Value |
|---|---|
| **Epic** | [EPIC: Ambient Brightness Controller](../Epic-AmbientBrightnessController.md) |
| **Depends on** | F2 (Automatic Brightness Control) |
| **Depended on by** | F7 (Status Dashboard) |

## Overview

This feature gives the user a single, precise correction knob on top of the automatic brightness: an
offset that shifts the computed brightness up or down. It lets the user say "the measurement may be
right, but I still want it brighter/darker" without giving up automation.

## User Story 1 — Adjust the offset

As a user, I want to shift the automatically computed brightness up or down, so that the final result
matches my personal preference even when the measurement is correct.

#### Acceptance Criteria

- Given the controller is running, when I change the offset, then the corrected brightness is applied
  within 1 s without waiting for the ambient light to change.
- Given no ambient light reading is available at the moment I change the offset, when a reading
  becomes available again, then the offset is applied to it without further user action.
- Given the offset is non-zero, when brightness is applied, then the applied value is the computed
  brightness plus the offset, clamped to the valid brightness bounds.

## User Story 2 — An offset that cannot break the screen

As a user, I want the offset to be unable to make my screen unusable, so that I can experiment with
it freely.

#### Acceptance Criteria

- Given the offset is at its most negative setting, when the result would fall below the minimum
  allowed brightness, then the applied value is clamped to that minimum and never 0.
- Given the offset is at its most positive setting, when the result would exceed the device maximum,
  then the applied value is clamped to the device maximum.
- Given the applied value has been clamped, when I look at the app, then it indicates that the offset
  could not be fully applied at the current light level.
- Given the offset is 0, when brightness is applied, then the result is exactly the unmodified
  computed brightness.

## User Story 3 — A usable control range

As a user, I want a control that is fine-grained enough to matter but small enough to stay safe, so
that I can tune the screen quickly.

#### Acceptance Criteria

- Given the offset control, when I use it, then it provides a symmetric range from −50 % to +50 % of
  the brightness span, adjustable in 1 % steps.
- Given the offset control, when I press and hold or drag, then the displayed value updates
  continuously and the brightness follows while I adjust it.
- Given I have moved the offset away from 0, when I use the reset affordance, then the offset returns
  to 0 and the brightness is recomputed immediately.

## User Story 4 — The offset survives restarts

As a user, I want my offset to still be in effect after I close the app or reboot my phone, so that I
do not have to set it again.

#### Acceptance Criteria

- Given I have set an offset, when the app process is restarted, then the offset is still in effect
  once the controller resumes.
- Given I have set an offset, when the device is rebooted and the controller resumes, then the offset
  is still in effect.
- Given I have never changed the offset, when the controller runs for the first time, then the offset
  is 0.

## Out of Scope

- Changing how quickly the offset is applied (that is F4).
- Multiple offset profiles, per-app offsets, or time-of-day dependent offsets.
- An offset on the measured lux value itself (the offset applies to brightness, not to the sensor
  reading).
- Automatic learning or suggestion of an offset.

## Resolved Decisions

- The offset is symmetric and expressed as a **percentage of the brightness span**, ranging from −50 %
  to +50 % in 1 % steps, so a positive and a negative offset of the same magnitude are equally strong.
- The offset control lives on the app's dedicated settings screen — not on the dashboard and not in the
  ongoing notification.

## Technical Design

### Architecture & Components

Modules unchanged: `:core` (pure Kotlin), `:infrastructure` (Android adapters), `:app` (composition
root, manual DI); package root `com.ambientbrightness`. F3 plugs into the `BrightnessTransform` hook
F2 already exposes (post-mapping, pre-clamp) — it does not touch F2's mapper or clamp and adds no
parallel pipeline. No UI here: the dedicated settings screen is F7, which only consumes the state
below.

Components to add (all `:core` unless noted):
- `BrightnessOffset` — value object: integer percent in `[-50, +50]`, step 1, symmetric; `0` is the
  default and the identity; rejects out-of-range or non-step values. `BrightnessOffsetRange` holds the
  constants (`MIN = -50`, `MAX = +50`, `STEP = 1`, `DEFAULT = 0`).
- `BrightnessOffsetApplication` (sealed) — `NotEvaluated`, `FullyApplied`, `ClampedToMinimum`,
  `ClampedToMaximum`; the observable "offset could not be fully applied" state.
- `BrightnessOffsetSource` (interface) — read-only side F2 and F7 depend on.
- `BrightnessOffsetService` (interface, extends `BrightnessOffsetSource`) — boundary/entry point F7
  calls to change the offset.
- `OffsetApplicationSink` (interface) — write-side port the transform uses to publish the clamp
  indicator without widening F2's `apply(...): Float` signature.
- `BrightnessOffsetTransform` — the `BrightnessTransform` implementation (F2 hook).
- `BrightnessOffsetStateHolder` — implementation backing `BrightnessOffsetService`.

Persistence reuses F4's **shared `SettingsStore`** (no bespoke port): F3 registers its
`SettingsKey<Int>` (`brightness_offset_percent`) in F4's `SettingsKeys` and persists through
`InMemorySettingsStore` (Dummy) and `AndroidDataStoreSettingsStore` (persistent) — both
`:infrastructure` implementations already provided by F4; `:core` holds only the contract. `:app`:
register the transform with F2's controller and expose the service; no logic.

### Data Flow

Write path (F7 → shared store → pipeline): `BrightnessOffsetService.setOffset` / `adjustBy` /
`reset` → `BrightnessOffsetStateHolder` normalises via `BrightnessOffset` →
`SettingsStore.set(SettingsKeys.BRIGHTNESS_OFFSET_PERCENT, offset.percent)` → updates
`offset: StateFlow`. F2's `BrightnessControllerUseCase` observes
`BrightnessOffsetSource.offset` and re-applies using the **last known lux**, so a change lands within
1 s (AC1.1) with no new sample; if no lux has ever arrived it writes nothing and the stored offset
applies on the first `LuxSample` (AC1.2).

Read path (per sample): `AmbientLightService.luxStream()` → `DefaultBrightnessMappingCurve` (F2) →
F2 runs `BrightnessTransform` hooks, including `BrightnessOffsetTransform`, returning
`fraction + percent / 100` → F2 `BrightnessRangeResolver`/`BrightnessRange` scales and clamps to
`[max(1, min), max]`. The offset thus applies to the computed brightness after mapping and before the
final clamp. At `0` the transform returns the fraction unchanged (AC2.4 identity).

Clamp observability: since `apply` returns only a `Float`, `BrightnessOffsetTransform` uses the
resolved `BrightnessRange` from `BrightnessContext` to compute the pre-clamp result and its clamped
counterpart; when they differ it publishes `ClampedToMinimum`/`ClampedToMaximum` to
`OffsetApplicationSink`. `BrightnessOffsetSource.application` surfaces this to F7 instead of a silent
no-op.

### Key Interfaces & Classes

```kotlin
// reused verbatim from F2
interface BrightnessTransform { fun apply(fraction: Float, context: BrightnessContext): Float }
interface BrightnessOffsetSource {
    val offset: StateFlow<BrightnessOffset>
    val application: StateFlow<BrightnessOffsetApplication>
}
interface BrightnessOffsetService : BrightnessOffsetSource {
    suspend fun setOffset(offset: BrightnessOffset)
    suspend fun adjustBy(deltaPercent: Int)
    suspend fun reset()                       // == setOffset(BrightnessOffset.DEFAULT)
}
interface OffsetApplicationSink { fun publish(application: BrightnessOffsetApplication) }
// Persistence reuses F4's shared store (not redefined here):
//   SettingsStore.observe(SettingsKeys.BRIGHTNESS_OFFSET_PERCENT): Flow<Int>
//   SettingsStore.set(SettingsKeys.BRIGHTNESS_OFFSET_PERCENT, offset.percent)
```

The only additive change to F2 is exposing the already-resolved `BrightnessRange` on
`BrightnessContext` and driving a re-apply when `BrightnessOffsetSource.offset` changes; no other F2
member is altered. `BrightnessRange` min is F2's floor `>= 1`, which is why the offset can never
drive the screen to 0 nor invert the range (AC2.1).

### State & Persistence

Mechanism is not re-decided here: F3 **adopts F4's shared `SettingsStore`** (AndroidX Preferences
DataStore over the single app-private file `settings_store.preferences_pb`) rather than inventing its
own port. F3 contributes exactly one entry to F4's `SettingsKeys` registry —
`BRIGHTNESS_OFFSET_PERCENT: SettingsKey<Int>` (stored key `brightness_offset_percent`) — and stores
the offset as an `Int` percent; `BrightnessOffset` stays the domain type and is reconstructed from the
stored `Int` in `BrightnessOffsetStateHolder`. The `Flow<T>` surface, off-main-thread suspend writes,
transactional consistency and reboot survivability (AC4.1/AC4.2) are inherited from F4's store by
construction — no duplicate DataStore, no bespoke key handling.

Absent key ⇒ default `BrightnessOffset.DEFAULT` (`0`), so a first run reports `0` without writing
(AC4.3): `BrightnessOffsetStateHolder` seeds `offset` from
`SettingsStore.observe(BRIGHTNESS_OFFSET_PERCENT)` and falls back to `0` when unset.
`BrightnessOffsetStateHolder` is app-scoped and is the single source of truth for `offset`. The
UI-independent `application` state is in-memory only and resets to `NotEvaluated` on process start
until the next sample. Out-of-range stored `Int` values are clamped via `BrightnessOffsetRange` on
read (defence in depth; the shared store performs no domain validation).

### Android Platform Specifics

- DataStore needs an Android `Context`; `:core` depends only on the `SettingsStore` boundary F4
  introduces, and the `Context`-dependent `AndroidDataStoreSettingsStore` lives in `:infrastructure`
  (F4 owns it — F3 adds no adapter).
- Boot/restart persistence is inherent — no `BOOT_COMPLETED` receiver or foreground work in F3 (F5 owns that).
- API 36 / minSdk 36: use the current stable `androidx.datastore` artifact; no legacy APIs.
- No new permission and no new system setting; F3 never touches `SCREEN_BRIGHTNESS` itself, and ships
  no Compose/UI (F7 builds the settings screen).

### Edge Cases & Failure Handling

Verdict: **feasible with caveats** (see the `BrightnessContext` coordination note).

- Offset changed with no ambient reading → value persisted and emitted; no write; applied on the next
  sample (AC1.2). `application` stays `NotEvaluated`, so F7 must not claim clamping yet.
- Most negative / positive offset overflows the range → F2's clamp handles it; the transform publishes
  `ClampedToMinimum`/`ClampedToMaximum`, applied value is the floor/max, never `0` (AC2.1/AC2.2).
- Offset `0` → identity; transform output equals the unmodified fraction (AC2.4); `application` is
  `FullyApplied`.
- Offset already at a clamped value (e.g. range max) → `FullyApplied` only when the clamped result
  equals the pre-clamp result; otherwise the clamped state stands.
- `SettingsStore` read error/corruption → fall back to `DEFAULT` (`0`), log, no crash; a failed write
  keeps the in-memory value, reports to F7 and does not block brightness.
- Dead-band suppression (F2) is F2 behaviour, not an F3 clamp: it stays `Suppressed`, not a clamp
  indication.

### Test Strategy

`:core` is pure Kotlin. `BrightnessOffset` gets boundary tests (`-50/+50` valid; `-51/+51` and
non-step rejected; `0` identity). `BrightnessOffsetStateHolder` is unit-tested with a `:core`
test-double settings store — F4's `InMemorySettingsStore` is an `:infrastructure` implementation, out
of `:core`'s reach — for default-on-absent (AC4.3), set/adjust/reset (AC3.3), restore after a
simulated restart (AC4.1/4.2) and no-reading change (AC1.2); an out-of-range seeded `Int` clamps to
the `BrightnessOffsetRange` bounds. `BrightnessOffsetTransform` is tested with a fixed
`BrightnessContext`/`BrightnessRange` for identity at `0` (AC2.4), the `±50 %`-of-span arithmetic
(AC3.1) and both clamp directions plus the published indicator (AC2.1–AC2.3). The DataStore round-trip
(absent key ⇒ `0`, recreate ⇒ value persists) is already covered by F4's
`AndroidDataStoreSettingsStore` instrumented test, so F3 adds no new instrumented test. A fake-driven
integration test asserts the offset is applied within the 1 s budget after a change (AC1.1). No
Compose tests.

## Task Breakdown

- [x] [T1 — Offset domain model & boundary interfaces](./Tasks/Task-01-OffsetDomainModelBoundaryInterfaces.md)
- [x] [T2 — Offset key registration & shared-store wiring](./Tasks/Task-02-OffsetKeyRegistrationSharedStoreWiring.md)
- [x] [T3 — Offset transform (F2 hook)](./Tasks/Task-03-OffsetTransformF2Hook.md)
- [x] [T4 — Offset state service](./Tasks/Task-04-OffsetStateService.md)
- [x] [T5 — Pipeline integration & composition root](./Tasks/Task-05-PipelineIntegrationCompositionRoot.md)

## Suggested Implementation Order

`T1 → T2 → {T3, T4} → T5`. T1 fixes the contract every part and F7 depend on, so it lands first. T2
is independent of T3/T4 and can run alongside them (adapter only). T3 and T4 touch different units
and can proceed in parallel once T1 exists; T5 is last because it is the only place that connects the
transform to F2's running controller, the 1 s re-apply and the `:app` composition root.
