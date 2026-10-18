# T3 — Offset transform (F2 hook)

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-ManualBrightnessOffset.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T3 — Offset transform (F2 hook)**: implement
  `BrightnessOffsetTransform : BrightnessTransform` — `fraction + percent / 100`, identity at `0` —
  reading `BrightnessRange` from `BrightnessContext` and publishing the clamp indicator via
  `OffsetApplicationSink`; add `range` to F2's `BrightnessContext`.
  *Files: `:core/.../BrightnessOffsetTransform/**`, `:core/.../AutomaticBrightnessControl/BrightnessContext/**` (additive).*
  *Verify: unit tests for identity (AC2.4), span arithmetic (AC3.1) and both clamp directions
  (AC2.1–AC2.3).*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `781dd2d736a129753fa71d256f7d723b88543808`.*

- Implemented `BrightnessOffsetTransform(offsetSource, applicationSink) : BrightnessTransform` in
  `:core`, Android-free. `apply` reads `offsetSource.offset.value.percent` and returns the **raw**
  shifted fraction `fraction + percent / 100f`, so F2's `BrightnessRangeScaler` stays the single
  clamp. At `0` the shift is a no-op, so the returned value equals the input exactly (AC2.4
  identity).
- The clamp indicator (AC2.3) is computed from `context.range` in **raw device units** — the same
  arithmetic F2's scaler performs — so no out-of-contract `BrightnessValue` is ever constructed for a
  fraction outside `0f..1f`: `unclamped = min + (span * shifted).roundToInt()` and
  `clamped = min + (span * shifted.coerceIn(0f,1f)).roundToInt()`. When they are equal the offset was
  fully applied; a smaller `unclamped` publishes `ClampedToMinimum`, otherwise
  `ClampedToMaximum`. Exactly one `BrightnessOffsetApplication` is published per `apply` call, so the
  observable state always reflects the latest sample.
- Collision note: the task also lists adding `range` to F2's `BrightnessContext`, but F4 already
  introduced `BrightnessContext.range` and its KDoc already describes it, so **no** change was needed
  there (`:core/.../AutomaticBrightnessControl/BrightnessContext/**` is untouched).
- Files added:
  `core/src/main/kotlin/com/ambientbrightness/core/BrightnessOffsetTransform/BrightnessOffsetTransform.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/BrightnessOffsetTransform/BrightnessOffsetTransformTest.kt`.
- The test uses local fakes (`FakeOffsetSource` over a `MutableStateFlow<BrightnessOffset>`,
  `RecordingApplicationSink` recording into a list) and covers identity at `0` (AC2.4), the `+50`/
  `-50` half-span shift with no clamping (AC3.1), both clamp directions with the published indicator
  (AC2.1–AC2.3), the floor-never-zero guarantee, and that each call republishes the latest state.
- No deviation from the design. `:core` gained no `android.`/`androidx.datastore` import (the
  `ArchitectureStructureTest` guard stays green); `BrightnessContext` was left as-is.
- Verification (from the repo root):
  `$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot'` then
  `.\gradlew.bat :core:test --no-daemon --console=plain` → BUILD SUCCESSFUL, 26 test classes /
  157 tests, 0 failures (151 before T3: +1 class / +6 tests, all in `BrightnessOffsetTransformTest`).
