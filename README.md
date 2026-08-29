# kami-engine-rotor

Reduced-order rotor solver (`:rom-rotor`) — momentum theory plus a figure of
merit. Turns a multirotor's per-rotor **thrust** requirement into a sized rotor:
diameter, RPM, shaft power, torque, tip Mach, blade loading and blade mass.

It is the missing link in the clean-sheet design chain for rotorcraft. Sizing a
drone runs `thrust → rotor → motor → pack`, and the second arrow had no solver:
`kami-engine-motor` sizes a motor from a peak-kW target, but nothing computed
what that target should be. `size-for-thrust` produces exactly the input
`motor.solver/size-for-power` consumes.

Part of the clean-sheet design / CAE stack (purpose-split shared libs).
Zero-dep portable `.cljc`. Run `clojure -M:dev:test`.

## Two power models, kept honest against each other

| model | form | who uses it |
|---|---|---|
| `hover` | `P = T·vi / FM` | the designer — one number, FM, absorbs profile drag and non-uniform inflow |
| `forward` | `P = k_i·T·vi + P_profile·(1 + 4.65·μ²)`, `vi` from Glauert | cruise, where the two terms have to be separated |

Because `forward` degrades to the hover decomposition at `μ = 0`, `hover` also
reports **`:fm-implied`** — the figure of merit the decomposed model implies at
that same operating point. Pass an `:fm` far from `:fm-implied` and you have two
models disagreeing about one rotor; the discrepancy is on the result map rather
than averaged away.

## What it refuses to answer

- **Non-positive thrust, diameter or RPM** throw with the offending `:key` in
  `ex-data`, rather than returning a `NaN` that reads like a number.
- **Altitude outside the ISA troposphere** throws instead of extrapolating a
  lapse-rate law that has stopped applying.
- **Blade stall is reported, not hidden.** Above `CT/σ = 0.12` momentum theory
  still returns arithmetic; `:stalled? true` says the arithmetic is no longer
  about a rotor.

## Scope

One rotor, in isolation. It does **not** know how many rotors there are, and it
does not model rotor-on-rotor interference, ground effect, or airframe parasite
drag — that belongs to whatever sizes the airframe. Above roughly `μ = 0.5` the
edgewise model is outside its range; it does not say so, so the caller must.

There is **no battery/pack solver in this workspace** (`kami-engine-echem` is a
PEM fuel cell, not a Li-ion pack), so endurance is not computed here. A pack
solver would register its own kind on the same `cae.solver` contract.

## Reference point

A 15-inch two-blade prop making 10 N at sea level (a 1 kg-per-rotor quadrotor)
at 5000 rpm: **92.1 W shaft, 11.08 g/W, 0.176 N·m, tip Mach 0.293, CT/σ 0.090,
22.6 g of blade.** Every expected value in the test suite was computed by hand
from the governing equation before the code was run.
