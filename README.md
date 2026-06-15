# Arm Servo — 3-Axis Motor Control Visualizer

A single-file interactive explainer for a **3-axis robot arm servo loop** — the motor-control sibling of the [ISR loop / GaN visualizer](https://kylefoxaustin.github.io/isr-loop-viz/).

> **each joint's angle is read by an encoder → a servo ISR (PID) computes torque → the motor driver pushes it out → the joint moves → forward kinematics gives the tool point**

Same lesson, new domain: *the interrupt period isn't a "speed" knob — it decides whether the arm tracks its command or rings itself into instability.* Slide the ISR period from **20 µs** to **1 µs** and watch the three joints go from a flailing, limit-slamming mess to silky tracking.

![screenshot](docs/screenshot.png)

## Why a slow loop wrecks a servo

Unlike the power-rail case (where a slow loop just *droops*), a position loop has its own dynamics, so a slow loop is actively **dangerous**:

- The sampled controller adds **delay** (zero-order hold + one-period transport delay). That delay eats the loop's **phase margin**.
- Once the loop period is large relative to the joint's mechanical time constant, the closed loop **rings**, then goes **unstable** — the arm oscillates harder and harder until it slams its hard stops.

The model is a sampled **PD controller** on a 2nd-order joint (`J·θ̈ + b·θ̇ = τ`, `τ = Kp·e − Kd·θ̇`), tuned so:

| T_isr | T_isr / τ_mech | behavior | verdict |
|------:|:--------------:|:---------|:--------|
| 1–2 µs | ≤0.18× | crisp tracking | **SMOOTH** |
| 5 µs  | 0.45× | visible ringing | **MARGINAL** |
| 10–20 µs | ≥0.9× | rings → slams the stops | **UNSTABLE** |

## What it shows

- **Pipeline** — encoder → servo ISR (3 axes) → driver → motor (heats up) → arm.
- **Robot arm** — the 3-link arm at its actual pose (colored by health), a faint **commanded ghost** arm, and the tool-point trail (actual vs commanded). At 1 µs they overlap; at 20 µs the actual path is chaos.
- **3 joint scopes** — per axis: commanded angle (cyan), encoder sample-and-hold (amber), and the actual angle colored by following error (green→amber→red). Independent profiles per joint.
- **Metrics + verdict** — loop rate, worst following error, `T_isr/τ_mech`, encoder LSB, motor temperature, motor aging, and a SMOOTH / MARGINAL / UNSTABLE verdict.

## Controls

- **Interrupt period `T_isr`** — 20 / 10 / 5 / 2 / 1 µs.
- **Encoder** — 12 / 16 / 20-bit (resolution → `ENCODER LSB`, in ° / arcsec).
- **ISR overrun** — Off / On. The ISR now runs **3 axes per tick**, so the compute budget is tight: a 1 µs loop occasionally blows its deadline even idle, and a collision blows a burst. Blown deadlines = the update is dropped (motor holds), drawn hatched magenta.
- **Slow-mo** — 0.5× / 1× / 2×.
- **⚡ Collision** — knock a random joint with an impulse torque. At 1 µs it recovers; at 20 µs it can kick the loop into divergence.
- **Pause**, **● Record 3s GIF** (downloads `arm-servo-<period>us.gif`).

## Motor heating

Heat ∝ Σ torque² with thermal mass (it heats/cools on a time constant). A smooth loop barely warms the motors; an unstable loop saturates the torque and **cooks them** — temperature and an aging multiplier climb. Same idea as the battery in the GaN visualizer.

## Running it

Just open `index.html`. No build, no dependencies, no network. Keep `vendor/` next to it. (`gif.js` is vendored and the worker is base64-inlined, so recording works from `file://`.)

## License

MIT — see `LICENSE`.

TTA
