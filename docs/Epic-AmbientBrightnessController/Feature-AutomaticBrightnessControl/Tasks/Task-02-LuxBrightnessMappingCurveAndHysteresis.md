# T2 — Lux→brightness mapping curve and hysteresis

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-AutomaticBrightnessControl.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T2 — Lux→brightness mapping curve and hysteresis**: implement `DefaultBrightnessMappingCurve`
  (log-lux piecewise anchors, monotonic, low-light floor) and `HysteresisGate`.
  *Files: `:core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BrightnessMappingCurve/**`,
  `.../HysteresisGate/**`.*
  *Done: implementation commit `db5aeb2a42b0fb04ff8c667bf18f31a936c9f94f`.*
  *Verify: monotonicity property test, low-light floor (AC3.3) and breakpoint oscillation (AC3.2).*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `db5aeb2a42b0fb04ff8c667bf18f31a936c9f94f`.*

- Added `DefaultBrightnessMappingCurve`, interpolating linearly in `log10(1 + lux)` space across
  the approved anchors: `0→0.10`, `1→0.12`, `10→0.18`, `100→0.35`, `1,000→0.60`,
  `10,000→0.85`, `100,000→1.00`; higher lux saturates at `1.0`. Negative and NaN inputs map to
  the low-light floor, and positive infinity saturates high.
- Added per-instance `DefaultHysteresisGate` with 10% directional transition bands: upward
  transitions at `1.10×` a boundary and downward transitions at `0.90×`; invalid readings are
  rejected without changing gate state.
- Added focused curve and hysteresis JVM tests covering AC3.1/3.2/3.3, anchor/interpolation
  outputs, monotonicity/bounds, invalid values, threshold jitter, and multi-segment jumps.
- Files:
  `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/BrightnessMappingCurve/DefaultBrightnessMappingCurve.kt`,
  `core/src/main/kotlin/com/ambientbrightness/core/AutomaticBrightnessControl/HysteresisGate/DefaultHysteresisGate.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/BrightnessMappingCurve/DefaultBrightnessMappingCurveTest.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/HysteresisGate/DefaultHysteresisGateTest.kt`.
- Verification passed: `:core:test --no-daemon`;
  `:core:dependencies --configuration compileClasspath --no-daemon` (no Android artifacts);
  `:core:test :infrastructure:test :app:assembleDebug --no-daemon`. No deviations; T1 interfaces
  were unchanged.
