# T5 — Pipeline integration & composition root

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-ManualBrightnessOffset.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T5 — Pipeline integration & composition root**: register `BrightnessOffsetTransform` as F2's
  `BrightnessTransform`, make `BrightnessControllerUseCase` re-apply on
  `BrightnessOffsetSource.offset` change using the last known lux (≤ 1 s), and bind
  `BrightnessOffsetService` in `:app` for F7.
  *Files: `:core/.../BrightnessControllerUseCase/**` (additive),*
  *`:app/src/main/kotlin/com/ambientbrightness/app/**`.*
  *Verify: fake-driven integration test asserts the applied value changes within 1 s (AC1.1) and the
  clamped state reaches `application` (AC2.3).*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `5af78e6a6c0ae801fa4a646bcf7071fa344a06d0`.*

- `BrightnessControllerUseCase` (F2) gained a **purely additive** `registerOffsetSource(source:
  BrightnessOffsetSource)` API — no constructor parameter and no existing member or signature was
  altered, so every F2 call site and test keeps compiling. A change on
  `BrightnessOffsetSource.offset` calls the existing `reapply()`, so it is applied with the last
  accepted lux within 1 s and **without a new ambient sample** (AC1.1). The observation is a job
  (`offsetCollectionJob`) launched in `start()` inside the same `lifecycleLock`-guarded block and
  cancelled in `stop()`; `drop(1)` makes the value present at start not count as a change. With no
  reading yet, `reapply()` already no-ops, so the stored offset simply applies on the first sample
  (AC1.2) — no extra logic was added for that case.
- `AmbientBrightnessComponent` (composition root) now builds **one** app-scoped
  `BrightnessOffsetStateHolder(store = settingsStore, scope = controllerScope)` and exposes it as
  `val brightnessOffsetService: BrightnessOffsetService` — F7's boundary. The same holder is handed to
  the transform as **both** its `offsetSource` and its `applicationSink`, so the running pipeline and
  the UI share a single offset state and the clamp indicator reaches `application`.
- **Transform order (the critical decision):** the controller registers F3's
  `BrightnessOffsetTransform` **first** and F4's `ResponsiveBrightnessRamp` **second**, matching the
  `BrightnessPipeline` fold order. Because registration order is fold order, the offset shifts the
  mapped fraction and the ramp then governs *how fast that shift is applied* — which is exactly what
  F3's "Out of Scope" requires ("Changing how quickly the offset is applied (that is F4)"). The
  `registerOffsetSource(holder)` call sits in the same `also { … }` block, before `start()`, so the
  change is observed from the first run.
- Tests added. `:core` (JUnit 5, `BrightnessControllerUseCaseTest` extended, no parallel class): an
  `AC1_1 - an offset change while running re-applies using the last known lux` test (a write happens
  from the offset change alone, with the offset transform registered), an `AC1_2 - an offset change
  before the first reading writes nothing and applies on the first sample` test, and a
  `stop cancels the offset observation` test. A reusable `FakeBrightnessOffsetSource` was added to the
  existing `:core` test doubles rather than duplicating a local fake. `:app` (JUnit 4, new)
  `AmbientBrightnessOffsetIntegrationTest` drives the real composition root (dummy sensor → F1
  stabiliser → F2 pipeline with the offset transform + F4 ramp → recording `SystemBrightnessClient`)
  with the injected `Clock` and virtual time advanced together: `AC1.1` asserts a system write within
  the 1 s budget (2 × `SAMPLING_INTERVAL_MILLIS = 500` tick arithmetic) with **no** new sample, and
  `AC2.3` asserts that at a low light level a `-50` offset surfaces
  `BrightnessOffsetApplication.ClampedToMinimum` through `application` and drives the applied value to
  the range minimum (`1`), never `0`. The tests assert the observable outcome (applied value and
  `application`), not the ramp's internal steps.
- No deviation from the design; `:core` gained no `android.`/`androidx.datastore` import (the
  `ArchitectureStructureTest` guard stays green). The existing F4 `:app` test stays green.
- Verification (from the repo root, all with `--no-daemon --console=plain`):
  `.\gradlew.bat :core:test` → BUILD SUCCESSFUL, 27 classes / 171 tests, 0 failures (168 before T5:
  +3 tests, all in `BrightnessControllerUseCaseTest`); `.\gradlew.bat :infrastructure:test` →
  BUILD SUCCESSFUL, 8 classes / 32 tests, 0 failures; `.\gradlew.bat :app:testDebugUnitTest` →
  BUILD SUCCESSFUL, 2 classes / 5 tests, 0 failures (1 class / 3 tests before T5: +1 class / +2 tests,
  both in `AmbientBrightnessOffsetIntegrationTest`); `.\gradlew.bat :app:assembleDebug` and
  `.\gradlew.bat :app:assembleDebugAndroidTest` → BUILD SUCCESSFUL (the existing instrumented tests
  still compile against the changed component). Not run on a device: the `androidTest` suites are
  compiled only — no emulator or device is attached in this environment.
- Feature status: T1–T5 are complete, so every user story of F3 has a verified implementation. AC1.1
  is covered by T5's `:core` and `:app` tests plus T4's, AC1.2 by T4's and T5's, AC2.1–AC2.3 by T3's
  and T5's, AC2.4 by T3's, AC3.1 by T3's, AC3.3 by T4's, and AC4.1–AC4.3 by T2's and T4's.
