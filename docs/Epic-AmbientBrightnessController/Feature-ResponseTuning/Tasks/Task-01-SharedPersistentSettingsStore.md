# T1 — Shared persistent settings store

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-ResponseTuning.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T1 — Shared persistent settings store**: add `SettingsStore` + `SettingsKey<T>`/`SettingsKeys`
  in `:core` (contract and value objects only); implement `InMemorySettingsStore` (Dummy) and
  `AndroidDataStoreSettingsStore` (`:infrastructure`, Preferences DataStore on `Dispatchers.IO`);
  register one app-scoped instance in `:app`'s composition root. *Files:
  `:core/.../core/SettingsStore/**` (contract only),
  `:infrastructure/.../infrastructure/SettingsStore/**` (both implementations),
  `:app/src/main/kotlin/com/ambientbrightness/app/**`.* *Verify: unit test for round-trip + reactive
  emission on `InMemorySettingsStore`; instrumented test that a value written by one store instance
  is read by a fresh instance.*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `399d0286fd68b7b85995061e5a69985dce6791a6`.*

- Built the shared key contract in `:core`: `SettingsStore` (`observe`/`set` exactly as designed),
  `SettingsKey<T>` as a sealed hierarchy whose subtype is the Android-free type tag (`IntKey`,
  `LongKey`, `FloatKey`, `BooleanKey`, `StringKey`) and `SettingsKeys` as the single registry, holding
  `RESPONSE_SPEED_PERCENT` with the stored name `response_speed_percent` and the default `50`. A key
  carries its own absent-case value, so `observe` can stay non-null (`T`), which is what F3's
  "default when absent" needs. `:core` imports no `android.`/`androidx.datastore` type.
- Built both implementations in `:infrastructure`. `InMemorySettingsStore` is reactive and
  thread-safe (`MutableStateFlow<Map<String, Any?>>`, writes through `update`), serves the Dummy
  composition and tests. `AndroidDataStoreSettingsStore` maps each sealed subtype onto the matching
  `Preferences` accessor, suspends writes onto the injected `Dispatchers.IO`, and emits empty
  preferences (i.e. the key defaults) on an `IOException`; reads observe `DataStore.data`, so a
  collector sees a `set` from any instance immediately. It takes its `DataStore<Preferences>` and
  dispatcher through the constructor — no singleton — and offers
  `create(context, scope, fileName = "settings_store.preferences_pb", ioDispatcher)` for the
  composition root.
- `:app` registers exactly one app-scoped store over `settings_store.preferences_pb`
  (`AndroidDataStoreSettingsStore.create(context = this, scope = appScope)`), and the component's new
  `settingsStore: SettingsStore` parameter defaults to the in-memory store. F3/F4/F7 consume the
  contract unchanged; `SettingsKeys` is where F3 adds `BRIGHTNESS_OFFSET_PERCENT`.
- Files added:
  `core/src/main/kotlin/com/ambientbrightness/core/SettingsStore/SettingsStore.kt`,
  `.../core/SettingsStore/SettingsKey.kt`,
  `.../core/SettingsStore/SettingsKeys.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/SettingsStore/SettingsKeysTest.kt`,
  `infrastructure/src/main/kotlin/com/ambientbrightness/infrastructure/SettingsStore/SettingsStore/InMemory/InMemorySettingsStore.kt`,
  `.../infrastructure/SettingsStore/SettingsStore/Android/AndroidDataStoreSettingsStore.kt`,
  `infrastructure/src/test/kotlin/com/ambientbrightness/infrastructure/InMemorySettingsStoreTest.kt`,
  `infrastructure/src/androidTest/kotlin/com/ambientbrightness/infrastructure/AndroidDataStoreSettingsStoreTest.kt`.
  Files changed: `gradle/libs.versions.toml` (`androidxDatastore = "1.2.1"` +
  `androidx-datastore-preferences`), `infrastructure/build.gradle.kts`
  (`implementation(libs.androidx.datastore.preferences)`, `testImplementation(libs.kotlinx.coroutines.test)`),
  `app/.../app/Composition/AmbientBrightnessComponent.kt` (`settingsStore` parameter) and
  `.../app/Composition/AmbientBrightnessApplication.kt` (the one real store).
- Deviation 1 (corrected after the first review): `InMemorySettingsStore` lives in
  `:infrastructure` (`SettingsStore/SettingsStore/InMemory/`), **not** in `:core`. Core holds
  entities, value objects and interfaces only; every implementation of an interface — including the
  Dummy/in-memory one — belongs to Infrastructure. Its test therefore lives in `:infrastructure`'s
  JVM test source set (JUnit 4 + `kotlinx-coroutines-test`, added as `testImplementation`), because
  `:core` cannot depend on `:infrastructure`. The original task text and this document said
  "(`:core`, tests/dummy)"; both were wrong and are corrected here and in
  `Feature-ManualBrightnessOffset/Feature-ManualBrightnessOffset.md`. A `:core` test that needs a store uses a `:core`
  test-source double, like the existing `FakeClock` — T1 has no such `:core` test, so none was added
  yet; the first store-dependent `:core` test (F3 T4 / F4 T2) adds it.
- Deviation 2: `SettingsKey` is a sealed class hierarchy rather than an enum-tagged holder, and not a
  `data class`. The sealed subtype is the type tag that carries `T` into the store, which keeps
  `:core` free of storage types; identity (not value) equality prevents two features' keys from
  collapsing into one entry.
- Deviation 3: the instrumented test lives in `:infrastructure`'s `androidTest`, not `:app`'s —
  the adapter under test is `:infrastructure`, the module already resolves DataStore, and
  `InstrumentationRegistry.targetContext` supplies the `Context`. It writes to a unique file name per
  run and cancels the first store's scope before opening the fresh instance, since DataStore refuses
  two active instances on one file; it asserts absent ⇒ default, then a write read back by a fresh
  instance (simulated restart).
- Deviation 4: the store tolerates a corrupt file by reading defaults
  (`ReplaceFileCorruptionHandler { emptyPreferences() }`) instead of failing; a missing/corrupt key
  yields the key's own default, which is AC4.2 and F3's AC4.3.
- Verification (from the repo root, all with `--no-daemon --console=plain`):
  `.\gradlew.bat :core:test` → BUILD SUCCESSFUL, 19 test classes / 104 tests, 0 failures (102 before
  T1: +2 for `SettingsKeysTest`; the in-memory tests moved to `:infrastructure`);
  `.\gradlew.bat :infrastructure:test` → BUILD SUCCESSFUL, 8 test classes / 32 tests, 0 failures
  (28 before T1: +4 for `InMemorySettingsStoreTest`); `.\gradlew.bat :app:assembleDebug` → BUILD
  SUCCESSFUL; `.\gradlew.bat :infrastructure:assembleDebugAndroidTest` → BUILD SUCCESSFUL. The
  folder/package guards in both modules stay green.
- Not verified on device: `AndroidDataStoreSettingsStoreTest` (the DataStore round-trip across a
  fresh instance, AC4.1) is compiled only — no emulator or device is attached in this environment.
