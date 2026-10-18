# T4 — Responsive brightness ramp on F2's transform hook

**Completion:** Complete

## Dependencies

See the [Suggested Implementation Order](../Feature-ResponseTuning.md#suggested-implementation-order) in the parent feature document.

## Size

Not estimated in the existing documentation.

## Implementation Instructions

- [x] **T4 — Responsive brightness ramp on F2's transform hook**: implement
  `ResponsiveBrightnessRamp : BrightnessTransform` with `currentFraction`/`targetFraction`/
  `lastCallMillis` state, the time-based step bound (`rate × elapsed`, capped at `maxStepFraction`)
  and mid-ramp retargeting; register it in F2's transform hook list in `:app`, with F2's
  `MinimumStepDeadBand` ≤ 1 device unit so it cannot suppress ramp progress. *Files:
  `:core/.../core/ResponsiveBrightnessRamp/**`, `:app/.../app/**`.* *Verify: fake-clock unit tests
  that a full-span change becomes several bounded steps not one jump (AC2.1, AC2.2) and that a
  mid-ramp retarget never overshoots and stays monotone (AC2.3).*

## Verification Expectations

The verification criteria are recorded in the `Verify:` clause of the implementation instructions above.

## Implementation Notes


*Done: implementation commit `7591737649a2978690593e6303e63f91b915efef`.*

- Built `ResponsiveBrightnessRamp` as F2's post-mapping `BrightnessTransform`, so it runs before F2
  clamps and dead-bands: every intermediate value is clamped on its own and offered to the dead-band
  on its own. One invocation yields at most one step — no ticker — and the step is
  `clamp(elapsedMillis / reactionWindowMillis, one device unit, maxStepFraction)` of the fraction
  span, with `elapsedMillis` measured on the injected `Clock` since the previous invocation, so a
  full-span change becomes several bounded steps rather than a jump (AC2.1) and a step is capped at
  `maxStepFraction` (0.15) even after a long gap such as a screen-off period (AC2.2). The floor of
  one device unit keeps a step strictly *above* F2's `MinimumStepDeadBand`, which stays at its
  default of 1 device unit (`BrightnessControllerUseCase`'s `deadBand = MinimumStepDeadBand()`), so
  the dead-band can never swallow ramp progress.
- The profile is re-read from the current `ResponseSpeed` on every invocation (AC1.4), and the state
  is exactly the designed `currentFraction` / `targetFraction` / `lastCallMillis`; `currentFraction`
  and `targetFraction` are exposed read-only (the pattern `MinimumStepDeadBand.lastApplied` already
  uses) so tests and a later dashboard can observe the ramp. A step is never larger than the distance
  remaining to the target, so a target arriving mid-ramp — including a reversal — retargets from the
  current value, stays monotone and cannot overshoot (AC2.3).
- `:app` wiring: `AmbientBrightnessComponent` registers the ramp as a transform hook
  (`BrightnessControllerUseCase(...).also { it.registerTransform(ramp) }`) and keeps exposing the
  controller as `AutomaticBrightnessController`, so the public surface and the F5 ownership of the
  loop are unchanged.
- Files added: `core/src/main/kotlin/com/ambientbrightness/core/ResponsiveBrightnessRamp/ResponsiveBrightnessRamp.kt`,
  `core/src/test/kotlin/com/ambientbrightness/core/ResponsiveBrightnessRamp/ResponsiveBrightnessRampTest.kt`.
  Files changed: `app/src/main/kotlin/com/ambientbrightness/app/Composition/AmbientBrightnessComponent.kt`.
  F2's pipeline, curve, hysteresis, dead-band and clients are unmodified.
- Deviation 1: the first invocation takes its baseline from `BrightnessContext.currentBrightness`
  (the brightness actually in effect, i.e. F2's `deadBand.lastApplied`) instead of adopting the
  computed value, so a ramp does not start with a jump either; only when no applied brightness is
  known does it fall back to adopting the first target. This makes the design's "steps from the
  currently *applied* fraction" true at the start of a run as well as mid-run.
- Deviation 2: the design describes `clamp(rate × elapsedMillis, 1 span-unit, maxStepFraction × span)`
  without saying how a span-unit is expressed. It is implemented as `1 device unit / span` in
  fraction terms, which is the quantity F2's dead-band compares against, and the floor is applied
  after the cap so `coerceIn` cannot fail on a device range narrower than seven units.
- Note on the reaction window in the ramp: at F1's 500 ms cadence the anti-flash cap (0.15) binds for
  every response speed, because `500 / reactionWindowMillis` is ≥ 0.15 for all windows (300 ms →
  1.67, 3000 ms → 0.167). The response speed therefore shapes the ramp only at a finer cadence; the
  visible difference between fast and slow comes from the stabiliser (T3). Recorded so the behaviour
  is not mistaken for a bug; both extremes are covered by tests at a 50 ms cadence.
- Verification (from the repo root): `.\gradlew.bat :core:test --no-daemon --console=plain` →
  BUILD SUCCESSFUL, 24 test classes / 136 tests, 0 failures, 0 errors (23 / 128 before T4: +1 class,
  +8 tests, all in the new `ResponsiveBrightnessRampTest`); `.\gradlew.bat :app:assembleDebug
  --no-daemon --console=plain` → BUILD SUCCESSFUL.
- Test mapping: AC2.1 → a full-span change yields distinct bounded steps (first step exactly 0.15 =
  15 of 100 device units) and reaches the target; AC2.2 → a 60 s gap still produces one capped step;
  AC2.3 → a retarget mid-ramp (0 → 1 retargeted to 0.2) never exceeds the value at retarget time,
  never undershoots the new target and stays monotone; plus the baseline-from-applied-brightness
  case, the rate at both extremes, a live speed change, the one-device-unit floor and the
  non-finite-fraction pass-through.
