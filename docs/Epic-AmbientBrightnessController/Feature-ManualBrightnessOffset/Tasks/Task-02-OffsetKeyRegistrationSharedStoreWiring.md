# T2 — Offset key registration & shared-store wiring

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-ManualBrightnessOffset.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T2 — Offset key registration & shared-store wiring**: register
  `BRIGHTNESS_OFFSET_PERCENT: SettingsKey<Int>` (stored key `brightness_offset_percent`) in F4's
  `SettingsKeys`, and make `BrightnessOffsetStateHolder` (T4) read/write the offset through the
  injected `SettingsStore` — no new port or adapter. Depends on F4's `SettingsStore`/`SettingsKey`/
  `SettingsKeys`/`AndroidDataStoreSettingsStore` existing. *Files: `:core/.../Settings/**` (additive
  key only).* *Verify: absent key yields `0`; a write is observable via `SettingsStore.observe`;
  compiles against F4's store API.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `8be20111b13ab39f44fb3cb2fffc699ef3d9c1ee`.*

- Registered F3's single additive key in F4's registry: added
  `BRIGHTNESS_OFFSET_PERCENT: SettingsKey<Int> = SettingsKey.IntKey("brightness_offset_percent", 0)`
  to `SettingsKeys` and aligned its KDoc (F3 *owns* the key now, rather than "adds" it). The stored
  name and default are exactly as the design fixes them, so a first run reports `0` without a write
  (AC4.3) and F4's `observe`/`set` API is unchanged.
- Extended the existing `SettingsKeysTest` (no parallel class, F4 assertions kept intact) with four
  tests: freeze the new key's stored `name` and `default`; prove the absent case yields `0` via
  `key.coerce(null)` and a foreign value (`String`, `Float`); prove a stored `Int` round-trips
  (including both `−50`/`+50` bounds). The fourth reuses the existing `:core`
  `com.ambientbrightness.core.FakeSettingsStore` test double — no new fake was written, `:core` has no
  reachable store implementation — and asserts `observe(BRIGHTNESS_OFFSET_PERCENT)` emits `0` before
  any write and `25` after `set`, plus that the write was recorded under the stored name.
- No new port, adapter or state holder: persistence is F4's shared `SettingsStore` unchanged, and
  `BrightnessOffsetStateHolder` (T4) is deliberately untouched. `:core` gained no
  `android.`/`androidx.datastore` import (the `ArchitectureStructureTest` guard stays green).
- Files changed:
  `core/src/main/kotlin/com/ambientbrightness/core/SettingsStore/SettingsKeys.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/SettingsStore/SettingsKeysTest.kt`.
- Verification (from the repo root):
  `$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot'` then
  `.\gradlew.bat :core:test --no-daemon --console=plain` → BUILD SUCCESSFUL, 25 test classes /
  151 tests, 0 failures (147 before T2: +4 for the extended `SettingsKeysTest`).
