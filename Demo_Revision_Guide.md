# Bumper Cars Fairground Simulator — Demo Revision Guide
### CMP-5010B Computer Graphics Coursework

---

## Quick Reference — Controls

| Key | Action | Notes |
|-----|--------|-------|
| `Arrow Up` | Accelerate player car forward | Ride must be ON |
| `Arrow Down` | Brake / reverse player car | Ride must be ON |
| `Arrow Left` | Turn player car left | Ride must be ON |
| `Arrow Right` | Turn player car right | Ride must be ON |
| `R` | Toggle ride ON/OFF | Enables/disables all cars |
| `1` | Free-fly camera | WASD + mouse look |
| `2` | First-person cockpit view | Follows player car |
| `3` | Third-person chase view | Follows player car |
| `4` | Operator's Booth view | Fixed elevated corner view |
| `W A S D` | Move free-fly camera | CAM_FREE only |
| `F` | Toggle walking / fly mode | CAM_FREE only |
| `Space` | Jump | Walking mode only |
| `P` | Print camera position to console | Debugging aid |
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
- `Camera.cpp:73–122` — `moveForward`, `moveBackward`, `moveLeft`, `moveRight`
- `Camera.cpp:124–135` — `addYaw`, `addPitch`
- `Camera.cpp:140–169` — `jump`, `updateJump`
- `Camera.cpp:181–185` — `getActiveViewMatrix` CAM_FREE branch

---

### 1.2 First-Person Cockpit View — Key `2`

**File:** `Camera.cpp` | **Mode:** `CAM_FIRST_PERSON`

Eye locked 14 units above the player car, offset 2 units forward along the car's facing direction. Target is 10 units ahead of the eye. Updates every frame from player position and rotation — gives the feeling of sitting inside the car cockpit.

```
eye    = playerPos + (0, 14, 0) + forward * 2
target = eye + forward * 10
```

**Key code:** `Camera.cpp:192–196`

---

### 1.3 Third-Person Chase View — Key `3`

**File:** `Camera.cpp` | **Mode:** `CAM_THIRD_PERSON`

Camera sits 18 units behind and 21.8 units above the car, looking at a point 9.8 units above the car. Tracks player rotation in real time, giving a classic racing game perspective.

```
eye    = playerPos - forward * 18 + (0, 21.8, 0)
target = playerPos + (0, 9.8, 0)
```

**Key code:** `Camera.cpp:199–203`

---

### 1.4 Operator's Booth View — Key `4`

**File:** `Camera.cpp` | **Mode:** `CAM_BOOTH`

Fixed elevated view from the NE corner post of the arena, above the roof line. Positioned so it can see over the arena roof — chosen deliberately because a true top-down camera was blocked by the roof geometry.

```
eye    = (60.8, 50.8, -6.5)
target = (-10.0, 9.8, -5.0)
```

**Key code:** `Camera.cpp:206–210`

---

## 2. Design & Movement (25 marks)

---

### 2.1 Arena & Scene Design

**Files:** `main.cpp` (rendering), OBJ models via `COBJLoader`

The scene contains:
- OBJ bumper car **arena model** (`Arena.obj`) — walls, floor structure, decoration
- **Sky dome** — a large sphere drawn inside-out so the texture faces inward
- **Textured ground** (`FIELD` object type) — large flat quad outside the arena
- **Six bumper cars** rendered via OBJ model instancing (`BumperCar.obj` / `BumperCarHunter.obj`)

The sky dome uses spherical UV mapping computed in the fragment shader:
```glsl
float u = atan(n.z, n.x) / (2.0 * 3.14159265) + 0.5;
float v = n.y * 0.5 + 0.5;
```
This gives a seamless wrap of the sky texture across the sphere.

**Key code:** `fragmentShader.glsl` — `SKYDOME` case | `vertexShader.glsl` — `SKYDOME` case (uses sphere coords, passes normals)

---

### 2.2 Player Car Physics — Velocity Steering & Grip

**File:** `BumperCar.cpp` — `update()`, `accelerate()`, `turn()`

The key design decision is **grip steering**: turning does not instantly redirect the car's velocity. Instead, each frame the velocity vector is gradually steered toward the car's facing direction:

```cpp
// Redirect velocity toward the facing direction — simulates wheel grip
float grip = 3.0f;
glm::vec3 newDir = glm::normalize((velocity / speed) + facing * grip * deltaTime);
velocity = newDir * speed;
```

This makes the car drift slightly when turning sharply — mimicking real wheel grip. Without this, cars would slide sideways through turns.

**Full physics loop per frame:**
1. Integrate velocity → new position
2. Bounce off walls (see §4.2)
3. Apply grip steering (redirect velocity toward facing)
4. Decrement collision flash timer
5. Apply exponential friction: `velocity *= pow(base, deltaTime)`

---

### 2.3 Frame-Rate Independence

**Files:** `BumperCar.cpp` (`update`), `main.cpp` (deltaTime calculation in `animate()`)

All movement, AI timers, and animation multiply by `deltaTime`. Friction uses **exponential decay** rather than a linear subtraction, so the decay rate is identical regardless of frame rate:

```cpp
// Exponential friction — much stronger when the ride is stopped
float friction = active ? pow(0.5f, deltaTime) : pow(0.05f, deltaTime);
velocity *= friction;
```

Velocity snaps to zero below 0.5 units/s when the ride is off, to avoid an infinite crawl.

---

### 2.4 Wall Bouncing

**File:** `BumperCar.cpp` — `update()`

The arena limits are **asymmetric** — they match the actual black floor boundary in world space after model transforms:

| Axis | Negative limit | Positive limit |
|------|----------------|----------------|
| X | -60.0 | +46.0 |
| Z | -60.0 | +46.0 |

On wall contact:
- Position is clamped to the limit
- Perpendicular velocity component is **negated and halved** (0.5× restitution — bouncy but lossy)
- Parallel velocity component is **reduced by 15%** (0.85× damping)
- `triggerCollisionFlash()` is called for the visual flash effect

---

### 2.5 AI — Wander Behaviour

**File:** `BumperCar.cpp` — `updateWanderAI()`

Every **0.5–1.5 seconds** the car picks a new heading: current rotation ± up to 135°. It then turns toward that heading at 120°/s along the **shorter arc** (angle difference normalised to −180..+180):

```cpp
// Shorter-arc angle difference
float diff = aiTargetRotation - rotationDegrees;
while (diff >  180.0f) diff -= 360.0f;
while (diff < -180.0f) diff += 360.0f;
// turn at up to 120 deg/s
```

The car always accelerates at 30 units/s², producing looping organic paths.

**Wall avoidance** (`checkAndAvoidWalls()`): if within 10 units of any wall for more than 0.25 seconds, the AI overrides normal behaviour and steers toward the arena centre using `atan2`.

---

### 2.6 AI — Hunter Behaviour

**File:** `BumperCar.cpp` — `updateHunterAI()`

Hunters use **pursuit steering**: compute the bearing from self to target using `atan2`, then turn toward it along the shorter arc.

```cpp
// Bearing toward target: facing is (cos, 0, -sin) so angle = atan2(-dz, dx)
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

**Files:** `main.cpp` (keyboard handler), `BumperCar.cpp` — `accelerate()`, `turn()`

`R` toggles the global ride state. When `active == false`:
- `accelerate()` and `turn()` early-return — no input applied
- Friction jumps to `pow(0.05, deltaTime)` — very strong braking
- Cars coast to a natural halt rather than freezing instantly

---

## 3. Lighting (15 marks)

Four light sources: directional sun, overhead point light, spotlight, and a pulsing animated LED. Two shader programs share responsibility.

---

### 3.1 Primary Directional Sun — Light 0

**Files:** `Lighting.cpp`, `basicTransformations.frag`

Standard directional Phong light. `coords.w = 0.0` signals a directional light (no position, no attenuation). The fragment shader computes:

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

**File:** `basicTransformations.frag`

A second directional light from the **opposite side** of the sun prevents the back faces of car models going completely dark. Computed per-fragment:

```glsl
vec3 Lf = normalize(vec3(ViewMatrix * FillLightPos));
float NdotLf = max(dot(n, Lf), 0.0);
color += fill_diffuse * material_diffuse * NdotLf;
```

**Uniforms:** `FillLightPos`, `fill_diffuse`, `fill_specular`

---

### 3.3 Overhead Warm Point Light — Light 1

**Files:** `basicTransformations.frag`, `main.cpp` (upload of `PointLightPosEye`)

Per-fragment overhead point light positioned above the arena centre. The light position is transformed to eye space before upload so the direction to the light is computed **per-fragment** correctly across the mesh surface.

Uses **quadratic distance attenuation**:

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

**Files:** `fragmentShader.glsl` (`phongContribution`), `Lighting.cpp`

Warm yellow spotlight above the arena entrance sign, aimed down onto the ramp. Uses a **cone cutoff** and **exponent falloff**:

```glsl
float cosAngle = dot(-lightDir, normalize(vec3(L.spotDir)));
if (cosAngle < L.spotCutoff)
    attenuation = 0.0;           // outside cone — no contribution
else
    attenuation *= pow(cosAngle, L.spotExponent);  // soft edge falloff
```

`spotDir.w = 1.0` is used as the "is spotlight" flag in the shader.

---

### 3.5 Pulsing Animated LED — Light 3

**Files:** `Lighting.cpp` (`uploadLights`), `main.cpp` (`lightPulseTime`, light marker render loop)

Light 3 is animated using a sine wave each frame:

```cpp
float pulse = 0.65f + 0.35f * sin(lightPulseTime * 1.5f * 6.2832f);
sceneLights[3].difCols = glm::vec4(0.0f, 0.5f * pulse, 1.5f * pulse, 1.0f);
```

Both the **core sphere radius** and the **corona halo radius** in the light marker also pulse in sync:

```cpp
float coreRadius = 2.5f * pulse;
float haloRadius = 7.0f + 5.0f * pulse;
```

This creates a breathing LED effect and demonstrates time-based animation tied to a light source.

---

### 3.6 Visual Light Markers — Two-Pass Rendering

**File:** `main.cpp` — light marker render loop in `drawScene()`

Each light source is visualised as a glowing orb using **two render passes**:

**Pass 1 — Opaque core sphere:**
- Object type: `LIGHT_MARKER`
- Small radius (~2.0 units), depth write ON
- The `LIGHT_MARKER` shader path outputs the light's colour directly — no Phong calculation — so it always looks bright regardless of scene lighting

**Pass 2 — Corona/halo:**
- Large flattened sphere at the same position
- `glBlendFunc(GL_ONE, GL_ONE)` — additive blending, so the glow *adds* to whatever is behind it
- `glDepthMask(GL_FALSE)` — depth write OFF so the halo doesn't occlude geometry behind it
- Depth mask restored after

This gives each light source a visible neon glow without needing a post-processing pass.

---

### 3.7 Normal Matrix

**Files:** `vertexShader.glsl`, `basicTransformations.vert`, `main.cpp` (`setModelView`)

Normals must be transformed by the **inverse-transpose** of the upper-left 3×3 of the model-view matrix. This ensures normals remain perpendicular to surfaces after non-uniform scaling:

```cpp
// main.cpp — setModelView()
glm::mat3 normalMatrix = glm::inverseTranspose(glm::mat3(mv));
glUniformMatrix3fv(normalMatLoc, 1, GL_FALSE, glm::value_ptr(normalMatrix));
```

If the plain model-view matrix were used, normals would be distorted under non-uniform scale.

---

### 3.8 Emissive Neon Glow on Cars

**File:** `basicTransformations.frag`

```glsl
out_Color = clamp(litResult + emissiveColor * texSample, 0.0, 1.0);
```

`emissiveColor` is added on top of the Phong result, multiplied by the texture sample so dark texture areas don't glow. During normal driving it is a low-intensity version of the car's body colour; it briefly boosts on collision (driven by `collisionFlashTimer`).

**Uniform:** `emissiveColor` (vec4)

---

### 3.9 Per-Car Colour Tint

**Files:** `basicTransformations.frag`, `main.cpp` (carTints array, carTint upload)

```glsl
vec4 litResult = clamp(color, 0.0, 1.0) * texSample * carTint;
```

`carTint` multiplies over the diffuse texture after all lighting is computed. Each car has a unique tint (red, blue, yellow, green, purple, orange). `(1,1,1,1)` = unchanged texture colours. The tint array is defined in `main.cpp`:

```cpp
static const glm::vec4 carTints[6] = {
    glm::vec4(1.0f, 0.4f, 0.4f, 1.0f),   // 0: red   (wander)
    glm::vec4(0.4f, 0.6f, 1.0f, 1.0f),   // 1: blue  (wander)
    glm::vec4(1.0f, 1.0f, 0.3f, 1.0f),   // 2: yellow(wander)
    glm::vec4(0.4f, 1.0f, 0.4f, 1.0f),   // 3: green (player)
    glm::vec4(0.8f, 0.3f, 1.0f, 1.0f),   // 4: purple(hunter)
    glm::vec4(1.0f, 0.55f, 0.1f, 1.0f),  // 5: orange(hunter)
};
```

---

## 4. Collisions (25 marks)

---

### 4.1 Sphere–Sphere Car Collisions

**File:** `main.cpp` — `resolveCarCollisions()`

Every pair of cars is tested each frame. If the distance between centres is less than the sum of their radii (both 6.0 units), a collision is resolved:

```cpp
glm::vec3 norm   = glm::normalize(posA - posB);
float     overlap = (minDist - dist) * 0.5f;

// Separate the cars
bumperCars[i].setPosition(posA + norm * overlap);
bumperCars[j].setPosition(posB - norm * overlap);

// Exchange velocity components along the collision normal (elastic, equal mass)
float aAlong = glm::dot(vA, norm);
float bAlong = glm::dot(vB, norm);
bumperCars[i].setVelocity((vA + (bAlong - aAlong) * norm) * 0.9f);
bumperCars[j].setVelocity((vB + (aAlong - bAlong) * norm) * 0.9f);
```

This is an **elastic impulse response** — velocity components along the collision normal are exchanged, giving a billiard-ball style bounce. The 0.9 factor introduces a small energy loss.

---

### 4.2 Car–Wall Bouncing (AABB)

**File:** `BumperCar.cpp` — `update()`

Axis-aligned bounding box test against the asymmetric arena limits. Each axis (X and Z) is tested independently. On contact:

```cpp
velocity.x = -velocity.x * 0.5f;   // reflect + halve perpendicular component
velocity.z *= 0.85f;                // damp parallel component
```

(Axes swap roles depending on which wall is hit.)

---

### 4.3 Sphere–AABB Camera Collision

**Files:** `Camera.cpp` — movement functions, `main.cpp` — `checkCollision()`

The free-fly camera carries a sphere collider (radius 2.0 units). Before each movement step the old position is saved, then after moving `checkCollision()` is called:

```cpp
glm::vec3 oldPos = position;
position += dir * amount;
if (checkCollision(position)) position = oldPos;  // revert on hit
```

`checkCollision()` in `main.cpp` tests:
- Ground plane
- Arena raised floor
- Arena platform ledge faces
- Arena walls (inner faces)
- All bumper car OBBs (sphere vs OBB)

---

### 4.4 Sphere–OBB Camera vs Car Collision

**File:** `main.cpp` — `sphereOBBCollision()`, called from `checkCollision()`

Each bumper car is represented as an Oriented Bounding Box. The camera sphere is tested against it by transforming the sphere into the box's local space, then performing a standard sphere–AABB test:

```cpp
glm::vec3 d = sphereCenter - obbCenter;
glm::vec3 local(
    glm::dot(d, orientation[0]),
    glm::dot(d, orientation[1]),
    glm::dot(d, orientation[2])
);
return glm::distance(local, glm::clamp(local, -halfExtents, halfExtents)) < radius;
```

---

### 4.5 Collision Flash Effect

**Files:** `BumperCar.cpp` — `triggerCollisionFlash()`, `update()` (timer decrement) | `main.cpp` — uses timer for emissive/tint

`triggerCollisionFlash()` sets `collisionFlashTimer` to 0.4 seconds. While the timer is above zero, `main.cpp` uploads a boosted emissive colour and brighter tint for that car, producing a visible white flash on impact. The timer decrements by `deltaTime` each frame, and for floor glow discs the intensity interpolates between bright (flash) and soft (ambient):

```cpp
float intensity = (flash > 0.0f)
    ? glm::mix(0.3f, 1.0f, flash / 0.4f)  // bright flash fading out
    : 0.15f;                                // soft ambient glow
```

---

## 5. Overall Quality (20 marks)

---

### 5.1 Shader Architecture — Two Programs

**Main shaders:** `vertexShader.glsl` + `fragmentShader.glsl`

Handle: textured ground (`FIELD`), sky dome (`SKYDOME`), spheres (`SPHERE`), light markers (`LIGHT_MARKER`), and solid-colour objects. Object type is selected via `uniform uint object` with `#define` constants:

```glsl
#define FIELD        0
#define SKYDOME      4
#define ARENA_FLOOR  5
#define BUMPER_CAR   7
#define LIGHT_MARKER 8
```

Each case has its own code path in the shader, avoiding separate programs for minor variants.

**Lab shaders:** `basicTransformations.vert` + `basicTransformations.frag`

Handle all OBJ textured models (bumper cars, arena). Full Phong with three light sources (sun, fill, overhead point), per-car colour tint, and emissive glow. Based on the lab framework but extended with the fill light, overhead point light, `carTint` and `emissiveColor` uniforms.

---

### 5.2 Texture Usage

**Files:** `main.cpp` (loading), `palletes.png`

- **Car texture atlas** (`palletes.png`): `GL_NEAREST` filtering — preserves pixel-art crispness without blurring between colour blocks
- **Sky dome / ground textures**: `GL_LINEAR` + mipmaps — smooth gradients
- UV coordinates come from the OBJ file for models; computed in the fragment shader for the sky dome sphere

---

### 5.3 Linear Fog

**File:** `fragmentShader.glsl` — `FIELD` case

Distance fog applied in eye space on the ground quad:

```glsl
float fogStart  = -20.0;
float fogEnd    = -400.0;
float fogFactor = clamp((fogEnd - positionEyeExport.z) / (fogEnd - fogStart), 0.0, 1.0);
colorsOut = mix(fogColor, texColor, fogFactor);
```

Also applied to the arena geometry (`ARENA_FLOOR`, `ARENA_WALL`, `BUMPER_CAR` cases) with tighter fog bounds (`-50` to `-300`).

---

### 5.4 Code Organisation

**Classes:**
- `BumperCar` (`BumperCar.h` / `BumperCar.cpp`) — physics, AI, wall bounce, grip steering
- `Camera` (`Camera.h` / `Camera.cpp`) — all four camera modes, walking, jumping
- `Lighting` (`Lighting.h` / `Lighting.cpp`) — light source data, setup and per-frame GPU upload

**Section structure in `main.cpp`** (marked with `// --- dividers ---`):

| Section | Contents |
|---------|----------|
| `// --- Geometry ---` | Vertex/index data for floor, arena floor, unit cube |
| `// --- Matrices ---` | modelViewMat, projMat, normalMat |
| `// --- Collision Primitives ---` | sphere-sphere, sphere-AABB, car-car, sphere-OBB, checkCollision |
| `// --- Rendering Helpers ---` | `setModelView()` |
| `// --- Setup ---` | `setup()` — GL init, VAO/VBO, textures, model loading, car init |
| `// --- Rendering ---` | `drawScene()` — full scene draw |
| `// --- Animation ---` | `animate()` — delta-time, input, physics, AI, collision |
| `// --- Input & Window ---` | `resize()`, `keyInput()`, `specialKeyInput()`, `mouseMotion()` |
| `// --- Entry Point ---` | `printInteraction()`, `main()` |

**Main loop order in `animate()`:**
1. Compute delta-time
2. Process WASD camera movement
3. Update camera jump physics
4. Process arrow key player car input
5. AI tick (`BumperCar::updateAI`)
6. Physics tick (`BumperCar::update`)
7. Car-car collision resolution (`resolveCarCollisions`)
8. Post redisplay

---

## 6. File Reference Summary

| File | Purpose | Key things to show |
|------|---------|--------------------|
| `main.cpp` | Scene setup, render loop, input, collision resolution | `resolveCarCollisions()`, `checkCollision()`, `sphereOBBCollision()`, light marker two-pass render, section dividers |
| `BumperCar.h/.cpp` | Car physics, AI, wall bounce, grip steering | `update()`, `updateWanderAI()`, `updateHunterAI()`, `checkAndAvoidWalls()` |
| `Camera.h/.cpp` | All four camera modes, walking, jumping | `getActiveViewMatrix()`, `moveForward/Back/Left/Right()`, `jump()`, `updateJump()` |
| `Lighting.h/.cpp` | Light source setup and per-frame upload | `setupLights()`, `uploadLights()`, eye-space transform, pulse animation |
| `vertexShader.glsl` | Main vertex stage | `object` uniform dispatch, `SKYDOME` spherical UV, eye-space position export |
| `fragmentShader.glsl` | Main fragment stage | `phongContribution()`, spotlight cone test, `FIELD` fog, `LIGHT_MARKER` output |
| `basicTransformations.vert` | Lab shader vertex | Normal matrix, `ex_LightDir`, `ex_PositionEye` |
| `basicTransformations.frag` | Lab shader fragment | Sun, fill light, overhead point light, `carTint`, `emissiveColor` |
| `TestModels/Arena.obj` | Arena scene geometry | Arena structure, walls, decorations |
| `TestModels/BumperCar.obj` | Wander car model | Used for cars 0, 1, 2, 3 |
| `TestModels/BumperCarHunter.obj` | Hunter car model | Used for cars 4 and 5 |
| `Textures/palletes.png` | Car texture atlas | `GL_NEAREST` filtering — explain why |

---

## 7. Demo Talking Points

Use these during the viva to explain the key decisions:

**Controls:**
> "The arrow keys drive the green player car. R starts and stops the ride — when stopped, all cars decelerate with strong friction and coast to a halt. Keys 1 to 4 switch between the four camera modes at any time."

**Cameras:**
> "I implemented four distinct camera modes: free-fly with WASD and mouse look — which also has a walking and jumping submode — first-person cockpit, third-person chase, and a fixed operator's booth view at the corner of the arena. I had to use a corner angle rather than top-down because the arena roof blocks the view from directly above."

**Lighting:**
> "There are four light sources. The main shaders handle a directional sun, a spotlight at the arena entrance, and a pulsing blue point light. The lab shaders for the OBJ models also have a fill light from the opposite side of the sun so cars never go completely dark, and an overhead warm point light with quadratic distance attenuation."

**Light markers:**
> "The light markers use two-pass rendering. First an opaque core sphere, then a large additive-blended corona with depth write disabled — so the glow adds on top of everything without occluding geometry behind it. Light 3 pulses with a sine wave, which also drives the corona radius and the diffuse colour intensity."

**Car collisions:**
> "Car-car collisions use sphere-sphere testing. When two cars overlap I separate them by pushing each half the penetration depth, then exchange their velocity components along the collision normal — that's the elastic impulse formula which gives the billiard-ball bounce. I multiply by 0.9 to add a small energy loss. Wall bouncing reflects the perpendicular component at half strength and damps the parallel component."

**Grip steering:**
> "Turning doesn't instantly redirect the car's velocity — each frame the velocity vector is gradually steered toward the car's facing direction at a grip rate of 3 units per second. This means the car drifts slightly on sharp turns, which feels more like a real bumper car. Without this, cars would just slide sideways regardless of which way they're facing."

**AI:**
> "Wandering cars pick a random heading every 0.5 to 1.5 seconds and turn toward it along the shorter arc at 120 degrees per second. Hunter cars compute the bearing to their target using atan2 and turn at 150 degrees per second. Hunters retarget every 3 to 6 seconds with a 60/40 bias toward the nearest car. Both AI types share the same wall avoidance logic — if they're within 10 units of a wall for more than 0.25 seconds, they steer toward the arena centre."

**Frame-rate independence:**
> "All physics multiply by delta-time. Friction specifically uses pow(base, deltaTime) for exponential decay that behaves identically at any frame rate — if I used linear subtraction the decay rate would vary with the frame rate."

**Shaders:**
> "I use two shader programs. The main shaders handle the floor, sky dome, and light markers, with the object type selected by a uniform uint. The lab shaders handle all the OBJ models and have full three-light Phong plus per-car colour tint and emissive neon glow."

**Normal matrix:**
> "Normals can't just be multiplied by the model-view matrix because non-uniform scaling would distort them — they'd no longer be perpendicular to the surface. I compute the inverse-transpose of the upper-left 3x3 on the CPU and upload it as a separate uniform."
