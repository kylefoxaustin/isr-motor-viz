# CLAUDE.md — Arm Servo (3-axis motor control visualizer)

Handoff notes for a Claude Code session. The **motor-control sibling** of the
ISR loop / GaN visualizer (`../isr-loop-visualizer`, live at
kylefoxaustin.github.io/isr-loop-viz). Same single-file canvas + vanilla JS,
same design system; the power stage is replaced by a 3-DOF robot arm.

## What this is

A self-contained `index.html` animating a 3-axis revolute arm under sampled
servo control: encoder → servo ISR (PID) → driver → motor → joint → forward
kinematics → tool point. The point: the **interrupt period decides whether the
arm tracks its command or rings into instability**. 20 µs → UNSTABLE (the arm
slams its stops); 1 µs → SMOOTH.

## Architecture (one canvas, 1040×680)

- **Pipeline** (`drawPipeline`) — ENCODER → SERVO ISR → DRIVER → MOTOR → ARM.
- **Robot arm** (`drawArm`, `fkScreen`) — left panel. Actual arm (colored by
  verdict) + dashed commanded ghost + TCP trails (actual green / commanded faint).
- **3 joint scopes** (`drawJointScope`, `SC[0..2]`) — per axis: commanded angle
  (cyan), encoder sample-and-hold (amber), actual angle colored by following
  error (green/amber/red). Overrun blown periods hatched magenta; collision marker.
- **Metrics** (`drawMeta`) — chips + SMOOTH/MARGINAL/UNSTABLE verdict.

### Simulation model (the honest part)

- Each joint: 2nd-order plant `J·θ̈ + b·θ̇ = τ`, integrated at a 0.5 µs sub-step.
- Controller: sampled **PD** at every `nextTick` (`isrUs` apart):
  `u = Kp·(θcmd − θmeas) − Kd·θ̇meas`, where θmeas is the **encoder-quantized**
  angle and θ̇meas is the sampled difference. Applied with a **1-period transport
  delay** (`uApp`/`uPend`) — this delay is what destabilizes slow loops.
- Torque saturates at `cfg.umax`; joints hard-stop at `±cfg.thlim` (so an unstable
  loop slams the limits and stays bounded on-screen, which is also realistic).
- **Gains are tuned** (`cfg.Kp=0.008, Kd=0.12, J=1, b=0.02`) against a SLOW command
  (period ~440–600 µs) so the loop tracks at 1 µs but the sample delay destabilizes
  it by ~10 µs. Tune in node before touching these — see the tuning sweeps in the
  build history; the gradient is delicate (1/2 µs smooth, 5 µs marginal, ≥10 unstable).
- `cfg.windowUs = 360` (wider than the GaN tool — the motion is slow vs the loop).
- `state.speed` is µs of virtual time per real second (default 70).

### Ported sub-models

- **Encoder bits** = ADC bits: `encQuant` snaps θ to `2π/2^bits`. Like the ADC LSB,
  it's sub-pixel on the trace — surfaced honestly via the `ENCODER LSB` chip (°/arcsec).
- **ISR overrun** (`isrCost`/`isrOverran`/`missTicks`): cost includes `3×isrAxisCostUs`
  (three joints per tick → tight budget), jitter, and a collision surcharge. Blown
  deadline → the update is dropped (motor holds), drawn hatched magenta.
- **Collision** (= the GaN "disturbance kick"): impulse torque on a random joint.
- **Motor thermal** (`motorLoss`/`motorTargetC`/`motorTempC`): heat ∝ Σ torque²,
  integrated with thermal mass (`cfg.motorTau` real seconds) in `tick()`; drives the
  MOTOR glyph color + MOTOR TEMP / MOTOR AGING chips. Unstable loop saturates torque
  → cooks the motors.

### GIF recording

Same as the GaN tool: `gif.js` vendored, worker base64-inlined as `GIF_WORKER_B64`
(copied verbatim from the sibling). Record = 15 fps × 3 s → `arm-servo-<period>us.gif`.

## Gotchas

- **Headless rendering can't show the motor heating** — `--virtual-time-budget`
  doesn't advance `performance.now()` like real wall-clock, so the thermal model
  (real-seconds time constant) barely moves in a screenshot. Verify thermal logic in
  node; it works at 60 fps in a real browser.
- **Encoder bit depth is sub-trace** (same as ADC LSB in the sibling) — don't "fix"
  the invisible quantization; it's surfaced in the chip.
- Don't retune gains casually — the smooth→marginal→unstable gradient is sensitive to
  Kp/Kd/command-period. Sweep in node first.

## Status / not yet done

- v1 built + verified (1 µs SMOOTH / 5 µs MARGINAL / 20 µs UNSTABLE; collision,
  overrun, encoder, motor thermal all working). Local git only — **no GitHub repo yet**
  (Kyle's call: "new folder, repo later"). To publish, mirror the sibling: `gh repo
  create isr-motor-viz --public --source=. --push`, then enable Pages (main / root).
- `docs/screenshot.png` is the hero (1 MHz default config).
- Possible next: contour/path test mode, per-joint inertia differences, a
  "max stable loop period" auto-finder.

## Owner voice

Kyle signs off **TTA** (Trust the Awesomeness). Match it: casual, technical, direct.

---
TTA
