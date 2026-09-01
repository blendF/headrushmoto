# headrushmoto

**Duck to accelerate, lean to steer — an endless motorbike runner you play with your head.**

No keyboard, no gamepad. Sit up straight and you stop; duck and you fly. Your nose is the
throttle and the handlebars, tracked through your webcam, driving a first-person bike down an
endless highway in Three.js.

One HTML file. No build step, no `npm install`, no dependencies to vendor.

---

## Quick start

The webcam requires a **secure context**, so this will *not* work by double-clicking the file
(`file://`). Serve it over `localhost` (or HTTPS):

```bash
git clone https://github.com/blendF/headrushmoto.git
cd headrushmoto
python -m http.server 8080          # or: npx serve .
```

Then open **<http://localhost:8080/highway.html>** and allow camera access when prompted.

First load pulls Three.js and the MediaPipe face model from CDN (~10 MB), so give it a moment
on a cold cache. After that it runs entirely on your machine — **no video ever leaves the
browser**, there is no server component and nothing is uploaded.

---

## Controls

Your **nose tip** is the control point. The neutral zero is the **centre of the frame**, drawn
as a white ring on the webcam preview in the corner. Put your nose on that ring and you are
stopped and going straight.

| Do this | Get this |
|---|---|
| Nose **on** the centre ring | 0 speed, straight ahead |
| **Duck / point your nose down** | Accelerate — *how far you duck sets your top speed* |
| Head **up** from centre | Brake, all the way to a stop |
| Lean **left / right** | Move across the road — a lane change takes ~0.4 s at any speed |

| Key | Action |
|---|---|
| `←` `→` `↑` `↓` | Full fallback controls if the camera or model fails |
| `C` | Re-zero the neutral point on your current pose |
| `R` | Restart the run |

**The throttle is a ceiling, not a switch.** A small duck can only ever reach a small speed no
matter how long you hold it — you have to commit to the full duck to see VMAX. The bike then
ramps toward that ceiling procedurally rather than snapping to it:

| duck depth | 10% | 25% | 50% | 75% | 100% |
|---|---|---|---|---|---|
| **top speed** | 1 | 8 | 27 | 51 | 80 |

Dodge the barrels, crates, stalled cars and cones. A hit costs you 65% of your speed. The
gravel shoulders scrub speed off, and the barriers are a hard wall. Difficulty ramps over the
first 2500 m — obstacle gates get denser, but **never block all three lanes**, so a clean line
always exists.

---

## How the control model works

This went through several rewrites before it felt right, and the failures are worth recording
because they are all tempting:

**Steering is arcade, not simulation.** Head deflection commands *sideways speed* directly, and
that speed does **not** shrink as you go faster — infinite grip, Traffic Racer style. You never
lose traction; you only lose by hitting something. An earlier version modelled a real cornering
grip budget (`GRIP_LAT / speed`), which throttled the turn rate to 0.16 rad/s at top speed: the
bike leaned beautifully and refused to move. Deleted.

**The input is integrated, never differentiated.** Head position sets a velocity which is
integrated into road position. Mapping head position *straight onto* road position feels like
the obvious "zero latency" answer and is unusable — the landmark feed is a noisy 30 Hz signal,
and taking a derivative of it (for yaw or lean) amplifies that noise into a violent shake.
Integration is itself a low-pass filter, which is why this version is stable.

**Response is linear.** `STEER_EXP = 1.0`, deadzone as small as the tracker allows. A 5% head
movement is worth exactly 5%, because at 80 units/s a small correction matters enormously.

**Yaw and lean are cosmetic.** They are read *off* the smooth steer input — body positioning,
mirroring you 1:1 — and never feed back into how fast the bike crosses the road, so they cannot
add lag or fight the input.

**The camera is the bike.** It is parented to the bike's lean group, so it inherits position,
yaw and roll in the same matrix update. There is no camera easing to catch up, and the bike
cannot outrun its own viewpoint at any speed.

**Filtering is ~55 ms total** (`HEAD_TAU + STEER_TAU`), plus MediaPipe's own detection latency.
That is anti-jitter only. Cascading smoothing stages *adds* their time constants — the snappiness
knob is `LAT_SPEED`, never the filters.

The world is 16 recycled 60-unit road chunks: when one falls behind you it jumps to the far end
and re-rolls its obstacles. World Z rebases by 4000 units periodically so floats stay clean on
long runs, and the ground plane follows you with a sliding texture offset so it still reads as
fixed.

---

## Tuning

Every knob is a documented `const` at the top of [`highway.html`](highway.html). The ones worth
touching first:

| Constant | Default | What it does |
|---|---|---|
| `LAT_SPEED` | `15.0` | **The feel knob.** Sideways speed at full head deflection. Higher = snappier. |
| `VMAX` | `80.0` | Top speed at a full duck. Accel/coast/brake are expressed as *times*, so they rescale with it automatically. |
| `HALF_X` | `0.15` | Head travel needed for full steering lock. **Bigger = less sensitive.** |
| `HALF_Y` | `0.15` | Head travel needed for a full duck. |
| `DEADZONE` | `0.012` | Neutral pocket. Raise only if you see jitter at rest. |
| `ACCEL_TIME` | `3.5` | Seconds from a standstill to `VMAX` at a full duck. |
| `BRAKE_TIME` | `2.0` | Seconds to stop from `VMAX` with your head fully up. |
| `LEAN_MAX` | `0.62` | Lean angle (rad) at full deflection — pure body positioning. |
| `FP_ROLL` | `1.0` | How much of the bike's roll the camera keeps. Lower toward 0 for a level horizon. |
| `OBST_SLOTS` | `3` | Obstacle gates per 60-unit chunk. |
| `DIFF_DIST` | `2500` | Metres over which difficulty ramps to full. |
| `STEER_SIGN` / `PITCH_SIGN` | `1` | Flip to `-1` if a control feels inverted. |

---

## Troubleshooting

**Camera never starts / black preview.** You are almost certainly on `file://`. Serve over
`localhost` — see Quick start. Also check no other app is holding the camera.

**Steering feels mirrored.** Set `STEER_SIGN = -1`. Same for `PITCH_SIGN` if ducking brakes.

**Zero point is at my chin.** The neutral is fixed at the centre of the frame, so re-frame your
camera rather than your posture — or press `C` to re-zero on the pose you are actually sitting
in. `R` restores the centre zero.

**It jitters at rest.** Raise `DEADZONE` slightly, or `HEAD_TAU` to `0.05`. Poor lighting makes
the landmarks noisier — face a light source, not a window behind you.

**It feels sluggish.** Raise `LAT_SPEED`. Do *not* reach for the `_TAU` values; they are already
near the floor, and dropping them below ~0.02 just exposes the 30 Hz detection stepping.

**Low frame rate.** The MediaPipe GPU delegate may have fallen back to CPU — check the console.
Lowering `NUM_SEG` (fewer live road chunks) is the cheapest win.

---

## Stack

| Layer | Used |
|---|---|
| Language | Vanilla JS as an ES module, plain HTML + CSS — no build, no bundler |
| 3D | [Three.js](https://threejs.org) r128 (WebGL, `BufferGeometry`, camera parented to the bike) |
| Head tracking | [MediaPipe Tasks Vision](https://ai.google.dev/edge/mediapipe) 0.10.12 — `FaceLandmarker`, GPU delegate, `VIDEO` mode |
| Camera | `getUserMedia`, with a Canvas 2D overlay for the tracking marker |
| Loop | Two independent `requestAnimationFrame` loops — render at display rate, detection at video rate |

Tested in Chrome and Edge on Windows. Any Chromium or Firefox build with WebGL2 and
`getUserMedia` should work; Safari is untested.

---

## Licence

MIT — see [LICENSE](LICENSE).
