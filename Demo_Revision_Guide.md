# Bumper Cars Fairground Simulator — Demo Revision Guide
### CMP-5010B Computer Graphics Coursework

---

## Quick Reference — Controls

| Key | Action | Notes |
|-----|--------|-------|
| `W` | Accelerate player car | Ride must be ON |
| `A` | Turn left | Ride must be ON |
| `D` | Turn right | Ride must be ON |
| `R` | Toggle ride ON/OFF | Enables/disables all cars |
| `1` | Free-fly camera | WASD + mouse look |
| `2` | First-person cockpit view | Follows player car |
| `3` | Third-person chase view | Follows player car |
| `4` | Operator's Booth view | Fixed elevated corner view |
| `F` | Toggle walking mode | CAM_FREE only |
| `Space` | Jump | Walking mode only |
| Mouse drag | Look around | CAM_FREE only |
| `Esc` | Exit | — |

---

## Mark Scheme Overview

| Category | Marks | Weight |
|----------|-------|--------|
| Camera Views | 15 | 15% |
| Design & Movement | 25 | 25% |
| Lighting | 15 | 15% |
| Collisions | 25 | 25% |
| Overall Quality | 20 | 20% |
| **TOTAL** | **100** | **100%** |

---

## 1. Camera Views (15 marks)

Four distinct camera modes, selectable via keys 1–4.

---

### 1.1 Free-Fly Camera — Key `1`

**Files:** `Camera.cpp`, `Camera.h` | **Mode:** `CAM_FREE`

WASD movement with mouse look. Yaw and pitch are updated via `addYaw()` / `addPitch()`, with pitch clamped to ±89° to prevent gimbal flip. The view matrix is built using `glm::lookAt(position, position + viewDirection, UP)`.

**Walking mode (F key):** Constrains Y to ground level (`walkingHeight = 19.0f`). Enables jumping via `jump()` / `updateJump()` with gravity `-60.0f` and initial vertical velocity `+25.0f`.

**Collision:** Before each move, old position is saved. After moving, `checkCollision()` is called — if it returns true, position reverts. Prevents clipping through geometry.

**Key code locations:**
- `Camera.cpp:62–112` — `moveForward`, `moveBackward`, `moveLeft`, `moveRight`
- `Camera.cpp:114–124` — `addYaw`, `addPitch`
- `Camera.cpp:130–158` — `jump`, `updateJump`
- `Camera.cpp:171–175` — `getActiveViewMatrix` CAM_FREE branch

---

### 1.2 First-Person Cockpit View — Key `2`

**File:** `Camera.cpp` | **Mode:** `CAM_FIRST_PERSON`

Eye locked 14 units above the player car, offset 2 units forward along the car's facing direction. Target is 10 units ahead of the eye. Updates every frame from player position and rotation — gives the feeling of sitting inside the car cockpit.

```
eye    = playerPos + (0, 14, 0) + forward * 2
target = eye + forward * 10
```

**Key code:** `Camera.cpp:181–185`

---

### 1.3 Third-Person Chase View — Key `3`

**File:** `Camera.cpp` | **Mode:** `CAM_THIRD_PERSON`

Camera sits 18 units behind and 21.8 units above the car, looking at a point 9.8 units above the car. Tracks player rotation in real time, giving a classic racing game perspective.

```
eye    = playerPos - forward * 18 + (0, 21.8, 0)
target = playerPos + (0, 9.8, 0)
```

**Key code:** `Camera.cpp:188–192`

---

### 1.4 Operator's Booth View — Key `4`

**File:** `Camera.cpp` | **Mode:** `CAM_BOOTH`

Fixed elevated view from the NE corner post of the arena, above the roof line. Positioned so it can see over the arena roof — chosen deliberately because a true top-down camera was blocked by the roof geometry.

```
eye    = (60.8, 50.8, -6.5)
target = (-10.0, 9.8, -5.0)
```

**Key code:** `Camera.cpp:195–199`

---

## 2. Design & Movement (25 marks)

---

### 2.1 Arena & Scene Design

**File:** `main.cpp` (rendering), OBJ models via `COBJLoader`

The scene contains:
- OBJ bumper car **arena model** (walls, floor structure, decoration)
- **Sky dome** — a large sphere drawn inside-out so the texture faces inward
- **Textured floor** (FIELD object type) with a separate flat quad
- Decorative **light pole models**

The sky dome uses spherical UV mapping computed in the vertex shader:
```glsl
u = atan(n.z, n.x) / (2*pi) + 0.5
v = n.y * 0.5 + 0.5
```
This gives a seamless wrap of the sky texture across the sphere with no visible seam.

**Key code:** `vertexShader.glsl` — `FIELD` and `SKY_DOME` cases | `fragmentShader.glsl` — `SKY_DOME` case (texture lookup only, no lighting applied)

---

### 2.2 Player Car Physics — Velocity Steering & Grip

**File:** `BumperCar.cpp:68–158` (`update`), `BumperCar.cpp:160–180` (`accelerate`, `turn`)

The key design decision is **grip steering**: turning does not instantly redirect the car's velocity. Instead, each frame the velocity vector is gradually steered toward the car's facing direction:

```cpp
// grip = how quickly velocity direction tracks facing direction
float grip = 3.0f;
glm::vec3 newDir = glm::normalize(currentDir + facing * grip * deltaTime);
velocity = newDir * speed;
```

This makes the car drift slightly when turning sharply — mimicking real wheel grip. Without this, cars would slide sideways through turns.

**Full physics loop per frame:**
1. Integrate velocity → new position
2. Bounce off walls (see §4.2)
3. Apply grip steering (redirect velocity toward facing)
4. Apply exponential friction: `velocity *= pow(base, deltaTime)`

---

### 2.3 Frame-Rate Independence

**Files:** `BumperCar.cpp:68` (`update`), `main.cpp` (deltaTime calculation)

All movement, AI timers, and animation multiply by `deltaTime`. Friction uses **exponential decay** rather than a linear subtraction, so the decay rate is identical regardless of frame rate:

```cpp
float friction = active ? pow(0.5f, deltaTime)   // gentle drag while driving
                        : pow(0.05f, deltaTime);  // strong brake when ride OFF
velocity *= friction;
```

Velocity snaps to zero below 0.5 units/s when the ride is off, to avoid an infinite crawl.

---

### 2.4 Wall Bouncing

**File:** `BumperCar.cpp:76–117`

The arena limits are **asymmetric** — they match the actual black floor boundary in world space after model transforms:

| Axis | Negative limit | Positive limit |
|------|---------------|----------------|
| X | -60.0 | +46.0 |
| Z | -60.0 | +46.0 |

On wall contact:
- Position is clamped to the limit
- Perpendicular velocity component is **negated and halved** (0.5× restitution — bouncy but lossy)
- Parallel velocity component is **reduced by 15%** (0.85× damping)
- `triggerCollisionFlash()` is called for the visual flash effect

---

### 2.5 AI — Wander Behaviour

**File:** `BumperCar.cpp:244–275` (`updateWanderAI`)

Every **0.5–1.5 seconds** the car picks a new heading: current rotation ± up to 135°. It then turns toward that heading at 120°/s along the **shorter arc** (angle difference normalised to -180..+180):

```cpp
float diff = aiTargetRotation - rotationDegrees;
while (diff > 180.0f)  diff -= 360.0f;
while (diff < -180.0f) diff += 360.0f;
// turn at up to 120 deg/s
```

The car always accelerates at 30 units/s², producing looping organic paths.

**Wall avoidance** (`BumperCar.cpp:197–238`): if within 10 units of any wall for more than 0.25 seconds, the AI overrides normal behaviour and steers toward the arena centre using `atan2`.

---

### 2.6 AI — Hunter Behaviour

**File:** `BumperCar.cpp:289–368` (`updateHunterAI`)

Hunters use **pursuit steering**: compute the bearing from self to target using `atan2`, then turn toward it along the shorter arc.

```cpp
float targetAngleDeg = glm::degrees(atan2(-toTarget.z, toTarget.x));
```

The sign convention (`-toTarget.z`) matches the facing convention used everywhere else: `facing = (cos(rot), 0, -sin(rot))`.

**Parameters vs wanderers:**
- Turn rate: **150°/s** (vs 120 for wanderers) — conveys active pursuit
- Acceleration: **40 units/s²** (vs 30) — slightly faster

**Retargeting policy** (every 3–6 seconds):
- **60%** chance: pick the **nearest** non-hunter car
- **40%** chance: pick a **random** non-hunter car
- Hunters never target each other

Wall avoidance works identically to wanderers.

---

### 2.7 Ride On/Off Toggle — Key `R`

**Files:** `main.cpp` (keyboard handler), `BumperCar.cpp:160,171` (active guard)

`R` toggles the global ride state. When `active == false`:
- `accelerate()` and `turn()` are early-return no-ops
- Friction jumps to `pow(0.05, deltaTime)` — very strong braking
- Cars coast to a natural halt rather than freezing instantly

---

## 3. Lighting (15 marks)

Three light sources plus spotlight and animated LED. Two shader programs share responsibility.

---

### 3.1 Primary Directional Sun — Light 0

**Files:** `Lighting.cpp`, `fragmentShader.glsl`, `basicTransformations.frag`

Standard directional Phong light. The light direction is transformed to eye space in the vertex shader and passed as `ex_LightDir`. The fragment shader computes:

```glsl
float NdotL = max(dot(n, L), 0.0);
float RdotV = max(dot(r, v), 0.0);
color  = light_ambient * material_ambient;
color += light_diffuse * material_diffuse * NdotL;
color += material_specular * light_specular * pow(RdotV, material_shininess);
```

**Uniforms:** `light_ambient`, `light_diffuse`, `light_specular`, `material_ambient`, `material_diffuse`, `material_specular`, `material_shininess`

---

### 3.2 Fill Light (Opposite Direction)

**File:** `basicTransformations.frag:69–77`

A second directional light from the **opposite side** of the sun prevents the back faces of car models going completely dark. Computed per-fragment:

```glsl
vec3 Lf = normalize(vec3(ViewMatrix * FillLightPos));
float NdotLf = max(dot(n, Lf), 0.0);
color += fill_diffuse * material_diffuse * NdotLf;
// + specular if NdotLf > 0
```

**Uniforms:** `FillLightPos`, `fill_diffuse`, `fill_specular`

---

### 3.3 Overhead Warm Point Light — Light 1

**Files:** `basicTransformations.frag:79–94`, `main.cpp` (upload of `PointLightPosEye`)

Per-fragment overhead point light. The light position is already in eye space when passed in, and the direction to the light is computed **per-fragment** so illumination changes correctly across the mesh surface.

Uses **quadratic distance attenuation** (lecture slide 41 formula):

```glsl
vec3  toPoint   = vec3(PointLightPosEye) - ex_PositionEye;
float pointDist = length(toPoint);
float c1 = 1.0, c2 = 0.014, c3 = 0.0007;
float pointAtten = min(1.0 / (c1 + c2*pointDist + c3*pointDist*pointDist), 1.0);
```

Gives warm top-down lighting across the cars and arena surface.

**Uniforms:** `PointLightPosEye`, `point_light_diffuse`, `point_light_specular`

---

### 3.4 Spotlight — Light 2

**Files:** `fragmentShader.glsl` (`phongContribution` function), `Lighting.cpp`

Warm yellow spotlight with a cone cutoff angle and **exponent falloff**:

```glsl
spot factor = max(dot(L_to_fragment, spotDir), 0.0) ^ exponent
// = 0 outside cone
```

Applied via the `phongContribution()` helper function which handles the full Phong calculation including spot attenuation.

---

### 3.5 Pulsing Animated LED — Light 3

**File:** `main.cpp` (light marker render loop, `lightPulseTime`)

Light 3 is animated using a sine wave:

```cpp
float pulse = 0.65f + 0.35f * sinf(lightPulseTime * 1.5f * 6.2832f);
float coreRadius = 2.5f * pulse;
float haloRadius = 7.0f + 5.0f * pulse;
```

Both the **core sphere radius** and the **corona halo radius** pulse in sync each frame, creating a breathing LED effect. Demonstrates time-based animation tied to a light source.

---

### 3.6 Visual Light Markers — Two-Pass Rendering

**File:** `main.cpp` (light marker render loop)

Each light source is visualised as a glowing orb using **two render passes**:

**Pass 1 — Opaque core sphere:**
- Object type: `LIGHT_MARKER`
- Small radius (~2.0 units), depth write ON
- The `LIGHT_MARKER` shader path outputs the light's colour directly — no Phong calculation — so it always looks bright regardless of scene lighting

**Pass 2 — Corona/halo:**
- Large sphere at the same position
- `glBlendFunc(GL_ONE, GL_ONE)` — additive blending, so the glow *adds* to whatever is behind it
- `glDepthMask(GL_FALSE)` — depth write OFF so the transparent halo doesn't occlude geometry
- Depth mask restored after

This gives each light source a visible neon glow without needing a post-processing pass.

---

### 3.7 Normal Matrix

**Files:** `vertexShader.glsl`, `basicTransformations.vert`

Normals must be transformed by the **inverse-transpose** of the upper-left 3×3 of the model-view matrix. This ensures normals remain perpendicular to surfaces after non-uniform scaling. If the plain model-view matrix were used, normals would be distorted.

```cpp
// CPU side (main.cpp / shader setup)
glm::mat3 normalMatrix = glm::inverseTranspose(glm::mat3(modelViewMatrix));
```

---

### 3.8 Emissive Neon Glow on Cars

**File:** `basicTransformations.frag:99–101`

```glsl
out_Color = clamp(litResult + emissiveColor * texSample, 0.0, 1.0);
```

`emissiveColor` is added on top of the Phong result, multiplied by the texture sample so dark areas of the texture don't glow. During normal driving it is set to a low-intensity version of the car's body colour; it briefly boosts on collision (driven by `collisionFlashTimer` in `main.cpp`).

**Uniforms:** `emissiveColor` (vec4)

---

### 3.9 Per-Car Colour Tint

**Files:** `basicTransformations.frag:97`, `main.cpp` (carTint upload)

```glsl
vec4 litResult = clamp(color, 0.0, 1.0) * texSample * carTint;
```

`carTint` multiplies over the diffuse texture after all lighting is computed. Each car has a unique `bodyColor` (red, blue, yellow, green, etc.) that drives this uniform. `(1,1,1,1)` = unchanged texture colours. The player car flashes `(1,1,1,1)` white on collision.

---

## 4. Collisions (25 marks)

---

### 4.1 Sphere–Sphere Car Collisions

**File:** `main.cpp` (collision detection loop)

Every pair of cars is tested each frame. If the distance between centres is less than the sum of their radii (both 6.0 units), a collision is resolved:

```
normal  = normalize(posA - posB)
relVel  = velA - velB
impulse = dot(relVel, normal)

velA -= impulse * normal    // A slows/reverses along normal
velB += impulse * normal    // B gains that velocity

// Separate: push each car half the penetration depth
```

This is an **elastic impulse response** — velocity components along the collision normal are exchanged, giving a billiard-ball style bounce.

---

### 4.2 Car–Wall Bouncing (AABB)

**File:** `BumperCar.cpp:76–117`

Axis-aligned bounding box test against the asymmetric arena limits. Each axis (X and Z) is tested independently. On contact:

```cpp
velocity.x = -velocity.x * 0.5f;   // reflect + halve
velocity.z *= 0.85f;                // damp parallel
```

(Axes swap roles depending on which wall is hit.)

---

### 4.3 Sphere–OBB Camera Collision

**Files:** `Camera.cpp:75,91,100,107` (checkCollision calls), `main.cpp` (checkCollision implementation)

The free-fly camera carries a sphere collider (radius 2.0 units). Before each movement step:

```cpp
glm::vec3 oldPos = position;
position += dir * amount;
if (checkCollision(position)) position = oldPos;  // revert on hit
```

This prevents the camera clipping through arena walls and car geometry.

---

### 4.4 Collision Flash Effect

**Files:** `BumperCar.cpp:140–143` (timer), `main.cpp` (uses timer for emissive/tint)

`triggerCollisionFlash()` sets `collisionFlashTimer` to ~0.3 seconds. While the timer is above zero, `main.cpp` uploads a boosted `emissiveColor` and white `carTint` for that car, producing a visible white flash on impact. The timer decrements by `deltaTime` each frame.

---

## 5. Overall Quality (20 marks)

---

### 5.1 Shader Architecture — Two Programs

**Main shaders:** `vertexShader.glsl` + `fragmentShader.glsl`

Handle: textured floor (`FIELD`), sky dome (`SKY_DOME`), spheres (`SPHERE`), light markers (`LIGHT_MARKER`), solid-colour objects.

Object type is selected via `uniform int objectType` with `#define` constants:
```glsl
#define FIELD        0
#define SKY_DOME     1
#define SPHERE       2
#define LIGHT_MARKER 8
// etc.
```
Each case has its own code path in the shader, avoiding separate programs for minor variants.

**Lab shaders:** `basicTransformations.vert` + `basicTransformations.frag`

Handle: all OBJ textured models (bumper cars, arena). Full Phong with three light sources (sun, fill, overhead point), per-car colour tint, and emissive glow. Based on the provided lab framework but extended with the fill light and overhead point light.

---

### 5.2 Texture Usage

**Files:** `main.cpp` (loading), `palletes.png`

- **Car texture atlas** (`palletes.png`): `GL_NEAREST` filtering — preserves pixel-art crispness without blurring
- **Sky dome texture**: `GL_LINEAR` — smooth gradients look better for a sky image
- UV coordinates come from the OBJ file for models; computed manually in the vertex shader for the sky dome sphere

---

### 5.3 Linear Fog

**File:** `fragmentShader.glsl` — `FIELD` case

Distance fog applied in eye space on the floor surface:

```glsl
float dist      = length(ex_PositionEye);
float fogFactor = clamp((fogEnd - dist) / (fogEnd - fogStart), 0.0, 1.0);
out_Color       = mix(fogColor, litColor, fogFactor);
```

Gives atmospheric depth to the arena floor — most visible at the far edges.

---

### 5.4 Code Organisation

**Classes:**
- `BumperCar` (`BumperCar.h` / `BumperCar.cpp`) — physics, AI, collision
- `Camera` (`Camera.h` / `Camera.cpp`) — all four camera modes, walking, jumping
- `Lighting` (`Lighting.h` / `Lighting.cpp`) — light source data and setup

**Main loop order in `main.cpp`:**
1. Process input
2. Physics tick (`BumperCar::update`)
3. AI tick (`BumperCar::updateAI`)
4. Car-car collision resolution
5. Render: floor → arena → sky → cars → light markers

**Dead code removed:**
- `Arena.h` / `Arena.cpp` — class was never instantiated or called anywhere
- `BumperCar::draw()` — rendering moved entirely to `main.cpp`
- Duplicate `sceneLights[0].ambCols` assignment in `Lighting.cpp`
- Unused shader files from `glslfiles/` (`basic.vert/frag`, `displacement.vert/frag`, outdated copies)
- No-op `glm::rotate(carModel, glm::radians(360.0f), ...)` call

---

## 6. File Reference Summary

| File | Purpose | Key things to show |
|------|---------|--------------------|
| `main.cpp` | Scene setup, render loop, input, collision resolution | `draw()`, `keyboard()`, car-car collision loop, light marker two-pass render, `PointLightPosEye` upload |
| `BumperCar.h/.cpp` | Car physics, AI, wall bounce, grip steering | `update()`, `updateAI()`, `updateWanderAI()`, `updateHunterAI()`, `checkAndAvoidWalls()` |
| `Camera.h/.cpp` | All four camera modes, walking, jumping | `getActiveViewMatrix()`, `moveForward/Back/Left/Right()`, `jump()`, `updateJump()` |
| `Lighting.h/.cpp` | Light source setup | Scene lights array, positions, colours, types |
| `vertexShader.glsl` | Main vertex stage | `objectType` dispatch, `FIELD`/`SKY_DOME`/`SPHERE`/`LIGHT_MARKER` cases, sky dome UV, eye-space exports |
| `fragmentShader.glsl` | Main fragment stage | `phongContribution()`, `FIELD` fog, `SKY_DOME` texture, `LIGHT_MARKER` colour output |
| `basicTransformations.vert` | Lab shader vertex | Normal matrix, `ex_LightDir`, `ex_PositionEye` |
| `basicTransformations.frag` | Lab shader fragment | Sun (lines 54–66), fill light (69–77), overhead point light (79–94), `carTint` (97), `emissiveColor` (99–101) |
| `palletes.png` | Car texture atlas | GL_NEAREST filtering |

---

## 7. Demo Talking Points

Use these during the viva to explain the key decisions:

**Cameras:**
> "I implemented four distinct camera modes — free-fly with walking and jumping, first-person cockpit, third-person chase, and a fixed operator's booth at the corner of the arena. Each one gives a different perspective on the scene."

**Lighting:**
> "There are three light sources on the OBJ models: a directional sun, a fill light from the opposite side so cars never go completely dark, and an overhead warm point light with quadratic distance attenuation — that's the formula from the lecture slides. There's also a spotlight and a pulsing animated LED."

**Light markers:**
> "The light markers use two-pass rendering. First an opaque core sphere, then a large additive-blended corona with depth write disabled, so the glow accumulates on top of everything without occluding geometry. Light 3 pulses with a sine wave."

**Car collisions:**
> "Car-car collisions use sphere-sphere testing with an elastic impulse response — I exchange the velocity components along the collision normal, which gives the billiard-ball bounce. Wall bouncing reflects the perpendicular component and halves it, and damps the parallel component."

**Grip steering:**
> "Turning doesn't instantly redirect the car — each frame the velocity vector is steered toward the facing direction at a grip rate of 3 units per second, so the car drifts slightly on sharp turns. Without this the car would just slide sideways."

**AI:**
> "Wandering AIs pick a random heading every 0.5 to 1.5 seconds. Hunter AIs compute the bearing to their target using atan2 and turn toward it at 150 degrees per second. Hunters retarget every 3 to 6 seconds with a 60/40 bias toward the nearest car versus a random car."

**Frame-rate independence:**
> "All physics use delta-time multiplication. Friction specifically uses pow(base, deltaTime) for true exponential decay that behaves the same at any frame rate."

**Shaders:**
> "I use two shader programs. The main shaders handle the floor, sky dome, and light markers, with the object type selected by a uniform int. The lab shaders handle all the OBJ models and have the full three-light Phong plus per-car colour tint and emissive glow."
