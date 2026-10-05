# Modular Car Controller

A small, self-contained Unity car controller built on `WheelCollider` physics
and the new Input System. The five scripts under
`Assets/Modular Car Controller/Scripts/` are deliberately decoupled — drop the
prefab into a scene, wire the inspector slots, and each component (driving,
lights, audio, UI, camera) can be used or removed independently.

- Unity version: **6000.3.10f1** (Unity 6, URP project template)
- Input: **Unity Input System** package (`UnityEngine.InputSystem`)
- Physics: built-in `WheelCollider`
- Pipeline-agnostic: emission uses the standard `_EmissionColor` property, which
  works in Built-in, URP, and HDRP

---

## Folder layout

```
Assets/Modular Car Controller/
  Audio/        Engine.mp3, Hit.mp3, Signal.mp3
  Models/       CarModel/Car 3.fbx + CarMaterials/Material.NNN.mat
  Prefabs/      CarController.prefab
  Scenes/       DemoScene.unity
  Scripts/      CarController.cs
                CarLights.cs
                CarAudio.cs
                CarUI.cs
                CameraFollow.cs
```

Anything outside this folder (terrain, default URP settings, the input-system
defaults) is the surrounding sample project — it is not part of the controller.

---

## Quick start

1. Open `Assets/Modular Car Controller/Scenes/DemoScene.unity`.
2. Press Play and drive with the controls below.

To add the controller to your own scene:

1. Drag `Assets/Modular Car Controller/Prefabs/CarController.prefab` into the
   scene.
2. The prefab has its own `Camera`. If you want to keep it, delete the
   `Main Camera` from the scene. Otherwise, unpack the prefab, delete the
   prefab's camera, attach `CameraFollow.cs` to your `Main Camera`, and set
   its `target` to the prefab root.
3. If you have your own `CarUI` overlay, delete the prefab's `CarUI` overlay,
   set its `car` slot to the prefab root, and wire the `speedText` to your
   canvas text element.

---

## Controls

| Key | Action |
| --- | --- |
| `W` / Up arrow | Accelerate forward (or motor-brake when rolling backward) |
| `S` / Down arrow | Reverse (or motor-brake when rolling forward) |
| `A` / `D`, Left / Right | Steer |
| `Space` | Handbrake |
| `L` | Toggle head lights |
| `Q` | Toggle left turn signal |
| `E` | Toggle right turn signal |
| `1` / `2` / `3` / `4` | Shift to R / N / P / D (manual gears only) |
| Mouse move | Orbit camera around the car |
| Mouse wheel | Zoom camera |
| `Esc` | Release cursor; left-click the window to re-lock |

Release the throttle and the car decelerates progressively to a stop via a
configurable coast brake — it never rolls forever.

---

## Components

### `CarController` (required)

Drives the four `WheelCollider`s. Front-wheel drive. Handles steering,
throttle, handbrake, motor brake, coast brake, anti-roll, optional manual P/R/N/D
gears, and forwards driving signals to `CarLights` each frame.

Key inspector fields (everything has a tooltip — these are the ones you will
actually tune):

| Field | What it does | Sensible default |
| --- | --- | --- |
| `motorForce` | Torque per unit throttle on the driven wheels | `100` |
| `maxSteeringAngle` | Front-wheel steer angle at low speed | `30°` |
| `steeringFalloffSpeed` / `highSpeedSteeringFactor` | Steering shrinks above this speed down to this fraction | `20 m/s`, `0.35` |
| `brakeForce` | Handbrake torque (Space) | `4000` |
| `motorBrakeForce` | Brake torque when pressing the opposite-direction key | `500` |
| `coastBrakeForce` | Brake torque applied when no throttle is held | `400` |
| `coastBrakeStopSpeed` | Below this speed coast brake doubles, so the car reaches 0 cleanly | `1.5 m/s` |
| `frontBrakeBias` | Fraction of brake torque on the front wheels (0..1) | `0.6` |
| `maxForwardSpeed` / `maxReverseSpeed` | Top speed in each direction; motor torque cuts off above | `30` / `10 m/s` |
| `useManualGears` + `currentGear` | Enables P/R/N/D mode and starting gear | off / `Park` |
| `centerOfMassOffset` | Local offset of the Rigidbody centre of mass; **negative Y is what stops the car flipping** | `(0, -0.5, 0)` |
| `antiRollForce` | Per-axle anti-roll bar force | `5000` |

Wheel slots: drag the four `WheelCollider`s into `frontLeftWheel` etc., and the
four visible wheel meshes into the `*WheelTransform` slots. The script captures
the rotation offset between collider and mesh at `Start()` so the mesh keeps its
authored orientation as the collider rotates.

Public API used by the other components:

- `CurrentSpeed` (m/s, unsigned) — for UI and audio.
- `ForwardSpeed` (m/s, signed) — positive forward, negative reverse.
- `Throttle` (-1..+1) — raw input; drives engine pitch.
- `UseManualGears` / `CurrentGear` / `EffectiveGear` — for UI gear letter.

### `CarLights` (optional)

Drives `Light` components and emissive materials for stop, head, tail/reverse,
and turn-signal lamps. Each emissive material slot is resolved at `Start()` to
a per-renderer instance so emission changes don't leak into the shared project
asset.

The script accepts the `.mat` asset you've assigned in the inspector, then walks
all child `Renderer`s looking for that exact material. If the FBX has been
reimported and the asset reference no longer matches but the material name does,
it falls back to a name-based match and logs a one-line note. When neither pass
finds a renderer it logs a warning and dumps every material in use under the
prefab — that diagnostic is what catches a broken folder restructure.

| Slot | Notes |
| --- | --- |
| `stopLightLeft` / `Right` | `Light` components used as brake lamps |
| `headLightLeft` / `Right` | Front spotlight `Light` components |
| `tailLightLeft` / `Right` | Reverse / rear-running lamp `Light` components |
| `turnLightFrontLeft` … `RearRight` | The four indicator lamp `Light` components |
| `*Mat` for each lamp | Optional emissive material driven in lock-step with its `Light` |
| `*Color` and `*Intensity` | Per-channel emission tint and brightness |
| `signalBlinkInterval` | Seconds per half-cycle of the turn signal |

Public API (read by `CarAudio` for the turn-signal loop):

- `HeadLightsOn`, `SignalLeftOn`, `SignalRightOn`
- `SetDrivingState(braking, motorBraking, reversing)` — called by `CarController`
- `ToggleHeadLights()`, `ToggleSignalLeft()`, `ToggleSignalRight()`

Stop-lamp behaviour: bright while braking, dim "running light" mode while head
lights are on, otherwise off. Reverse lamps follow the reverse signal. Turn
signals blink at `signalBlinkInterval` independent of frame rate.

### `CarAudio` (optional, requires `CarController`)

Three `AudioSource` slots — `engineSource`, `signalSource`, `hitSource`.

- **Engine** is one looping clip whose pitch is swept from `minThrottlePitch`
  (idle) to `maxThrottlePitch` (redline) by a simulated rpm. Holding throttle
  ramps rpm at `rpmRiseRate`; when it hits 1 the script "upshifts" — rpm snaps
  back to `postShiftRpm` and the gear counter increments until `gearCount` is
  exhausted, at which point rpm holds at redline. Releasing the throttle decays
  rpm at `rpmDecayRate` back to idle. The absolute value of `Throttle` is used,
  so reverse revs the engine the same way forward does.
- **Signal** is an independent loop, started only while `CarLights` reports a
  turn signal active.
- **Hit** is a one-shot on `OnCollisionEnter`. Volume and pitch are interpolated
  between `hitMin*` and `hitMax*` from current car speed (mapped between
  `hitSpeedThreshold` and `hitReferenceSpeed`), plus a small random jitter so
  repeated impacts don't sound identical. Below `hitSpeedThreshold` no sound
  plays — that suppresses resting contacts and light scrapes.

### `CarUI` (optional)

Displays speed and gear letter on a legacy uGUI `Text`. Reads `CurrentSpeed` and
`EffectiveGear` from the controller every frame.

- `speedUnitMultiplier` — `3.6` for km/h, `2.237` for mph.
- `speedUnitSuffix` — string appended after the number.
- `manualModeText` / `automaticModeText` — optional labels toggled by the
  controller's `UseManualGears`. Drop in two pre-made banners and the UI shows
  the right one based on the current driving mode.

### `CameraFollow`

Orbit/chase camera with mouse look, scroll-wheel zoom, sphere-cast collision,
and auto-recenter. No button to hold — moving the mouse swings the
camera around the car directly.

Behaviour worth knowing:

- **Auto-recenter** — after `recenterDelay` seconds without mouse input, yaw
  eases back behind the car. Recenter speed is scaled by the car's forward
  speed up to `recenterFullSpeed`, so the camera holds still when parked and
  gathers behind the car only as it actually drives forward.
- **Collision** — a sphere-cast from the look anchor to the desired camera
  position pulls the camera in when something is in the way. Anything in the
  target's own hierarchy is ignored, so the car never blocks its own camera.
- **Cursor** — locked and hidden at start. `Esc` releases the cursor;
  left-click the window to re-lock.

Set `target` to the car's root transform. If that transform also has a
`CarController`, the camera fetches it automatically and uses `ForwardSpeed`
for the auto-recenter ramp.

---

## How the components connect

```
                   CarController
                   /     |      \
            (drives)  (input)   (queries)
              /          |           \
         CarLights       |        CameraFollow
            /\           |
       (queries)    (queries)
          |              |
       CarAudio       CarUI
```

- `CarController` owns input and physics. It writes the current braking and
  reverse state into `CarLights` every frame via `SetDrivingState`.
- `CarLights` is the source of truth for which lamps are lit and which turn
  signals are blinking. `CarAudio` polls it to know when to start the turn-
  signal sound loop.
- `CarAudio` polls the controller for `Throttle` (engine pitch) and
  `CurrentSpeed` (collision intensity).
- `CarUI` polls `CurrentSpeed`, `EffectiveGear`, and `UseManualGears` for the
  HUD.
- `CameraFollow` reads `ForwardSpeed` from the controller to scale the
  auto-recenter.

Each component does a `GetComponent` fallback when its optional slot is left
empty, so the typical configuration — all five components on the same prefab —
needs almost no wiring beyond the wheel slots and the audio clips.

---

## Tuning notes
- **The car is moving too slow.** Increase the `motorForce` / the top speed.
- **The car brakes too weakly.** Increase the `motorBrakeForce` / the `brakeForce`.
- **The car is flipping while steering.** Decrease the angular damping of the Rigidbody component / set the center-of-mass-offset Y to `-0.5` / decrease the WheelCollider's extreme slip (!!! This causes understeering! To prevent that, increase the max steering angle, for example. !!!)

- **Steering feels twitchy at speed.** Lower `highSpeedSteeringFactor` so the
  front wheels turn less above `steeringFalloffSpeed`.
- **Car rolls forever after I release W.** Raise `coastBrakeForce`. Set it to
  `0` to disable coast braking entirely and get a free-coast feel.
- **Brakes lock and the car spins.** Lower `frontBrakeBias` so the front
  wheels keep more lateral grip while braking.
- **Engine sound never shifts.** `gearCount` is `5` by default — make sure
  `rpmRiseRate` is high enough to reach redline in reasonable time, or lower
  `gearCount` to hear shifts sooner.
- **Emissive lamps don't glow after a folder reorganise.** The `CarLights`
  inspector slots store material asset references by GUID. If you delete and
  recreate the materials, drag the new `.mat` assets back into the slots — the
  script logs a warning listing every material it found under the prefab so you
  can match them up.
- **NOTE: The prefab's settings already have values that make the car's controls stable.**
---

## Public API summary

```csharp
// CarController
public float CurrentSpeed   { get; }   // m/s, unsigned
public float ForwardSpeed   { get; }   // m/s, signed
public float Throttle       { get; }   // -1..+1
public bool  UseManualGears { get; }
public GearMode CurrentGear { get; }
public GearMode EffectiveGear { get; } // P / R / N / D for HUD

// CarLights
public bool HeadLightsOn  { get; }
public bool SignalLeftOn  { get; }
public bool SignalRightOn { get; }
public void SetDrivingState(bool braking, bool motorBraking, bool reversing);
public void ToggleHeadLights();
public void ToggleSignalLeft();
public void ToggleSignalRight();
```

These are the only members the components use to talk to each other. Anything
else is internal.
#   M o b i l  
 