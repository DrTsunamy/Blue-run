# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project overview

**Blue Run – Underwater Odyssey** is a browser-based 3D endless-runner built as a **single self-contained HTML file**. No build step, no server, no package manager — open in Chrome and it runs.

- **ACTIVE FILE: `index.html`** — deployed via GitHub Pages (https://drtsunamy.github.io/Blue-run/) from `master`
- **Older reference versions:** `blue_run_v3.html`, `blue_run_improved.html` (do not overwrite; sections below describing v3 may lag behind index.html)

### July 2026 AAA update (branch `gameplay-depth`, see git log)

Beyond everything documented below, `index.html` adds:

- **Curved world**: vertex-shader view-space bend (`CURVE` uniforms, `bendMaterial`, patched `scene.add`); winding S-curve retargeting in the loop
- **The Kraken** (`updateKraken`, phase machine by `S.dist`): silhouette 350m → hunt 600m (roars, sweeping seabed shadow, +50/s survival bonus) → tentacle slams 1000m → full assault 1500m (double slams, ink blasts, shockwave rings). Giant slam tentacles (`bigTents`) are INSTANT GAME OVER on direct hit; shield/decoy absorb one. Warning = growing floor shadow + lane flash, 1.2s.
- **Obstacles**: `stalactite` (falls at z>-13, shadow warning), `crabClaw` (2s snap cycle), `inkCloud` (1.5s blindness via #ink-overlay, no damage), `eelZap` (charges 1.5s then electrifies lane 2s), `torpedo` (timer-spawned cross-lane projectile, level 3+, NOT segment-scrolled), plus v3's `sweeper` moray eel
- **Power-ups**: `dash` (0.9s, 2.2x speed, invincible), `timeSlow` (world 75% for 3s, dist accrual unpenalized), `decoy` (orbiting mini-Warplet absorbs one hit), `shrink` (S.sizeF 0.5 for 5s, hitR() helper, +5/pearl), plus v3-era `magnet`/`x2`
- **Game feel**: coyote time (S.airT<0.1 + vy guard in doJump), jump buffering (S.jumpBufT 0.18s, consumed on landing; held Space converts to charge), hit-stop (S.hitStopT=0.09 freezes rawDt in takeDamage)
- **Graphics phase 1**: waterSurf (additive ShaderMaterial sheet overhead, brightness tied to sunL.intensity), marine snow (THREE.Points, 420 motes), 26 god rays, camera drift + curve lean, film grain overlay div, CSP includes media-src 'self'
- **Level cinematics**: `playCinematic(lv)` plays `lv<N>.mp4` (same folder) fullscreen at level transitions + run start; tap-to-skip; missing files fail silently. Prompts for generating clips: `blue-run-references/level-cinematic-prompts.md`
- **Dev hotkey**: Shift+K adds 250 dist mid-run
- **Reference assets:** `blue-run-references/` — video clips, keyframe stills, environment crops, character spec, Meshy.ai build guide

**Tech stack:** Three.js r128 (CDN), Web Audio API, Canvas 2D, CSS animations. Google Fonts: Luckiest Guy + Nunito 700/900.

**Hard constraints:** Stay single-file for all code. No imports. No npm. No build. All geometry procedural. Sole external-asset exception: optional `lv0-4.mp4` cinematic clips beside index.html — the game must run perfectly without them.

---

## Current state: what works

Everything in this list is fully implemented and functional in `blue_run_v3.html`.

**Core gameplay loop**
- 3-lane endless runner: player swims forward, world scrolls toward camera
- Lane switching: ← → / A D / swipe left-right
- Jump + charge-jump: hold Space/tap → charge bar fills (0–100%) → release → powered jump. Charge > 88% triggers Rainbow Boost
- Rainbow Boost: 2.5 s at 1.6× speed with 24-sphere rainbow trail, FOV jumps to 88, camera micro-shakes, "RAINBOW BOOST!" banner
- Physics: gravity 38 units/s², `player.userData.vy`, `player.userData.grounded`. Jump velocity 13.5–24 units/s scaled by charge
- Squash-and-stretch: player scale driven by `vy / maxJumpV`, rotation.x leans with jump arc
- Camera: soft-follows player X with lag 0.07; Y tracks jump with 0.38 multiplier; FOV lerps 60→78 with speed, 88 on boost

**Obstacles (5 types)**
- `urchin` — spinning icosahedron with 24 red spikes; rotates on x+y axes
- `rock` — dodecahedron, static
- `jelly` — jellyfish that bobs, pulses, drifts laterally across lanes; tentacles animate
- `mine` — icosahedron body with 14 Fibonacci-distributed spikes that pulse emissive orange; also rotates
- `barrier` — 3-lane wall with 1 safe gap; green arrows + arch mark the safe lane. Collision checks player lane vs `safeLane`

Obstacles spawn per-segment at probability `C.obFreq = 0.38`. `findSpawnZ()` tries 8 times to find a Z clear of other obstacles by 12 units. `makeObs()` weights by level: level 0 = urchin/rock/jelly only; level 1 adds mines; level 2+ adds barriers. Lane warning flash (bottom gradient) triggers when obstacle enters z > -14.

**Collectibles (3 types)**
- `pearl` chain (`makePearlChain`) — 4–6 pearls in a line on one lane, 4.5 units apart; 22% chance a gold coin follows the chain
- `gold` coin — worth 100 × combo multiplier
- `shield` pickup — 8 s shield; also doubles collect radius (magnet mode) while active. Shield absorbs one hit then grants 0.8 s invincibility

Collectibles bob (sin wave Y), rotate on Y, and sparkle (4 orbiting sphere meshes). Magnet pull activates at 3× collect radius, interpolates toward player at rate 0.25 per frame.

**Scoring**
- `S.score = Math.floor(S.dist) + S.pearls * 10` — recalculated every tick
- Point pickups from pearls (25 × combo) and gold (100 × combo) are added directly to `S.score` on collect
- Combo multiplier: `S.combo` increments on any collect, decays after 2.2 s of no collect. Capped at 6×. Badge shows when ≥ 2×
- High scores: top 5 stored in `localStorage` key `bluerun_scores`. Game-over screen shows top 3 with medal emojis; new best shown with ★

**Lives and damage**
- 3 lives (hearts HUD top-left)
- `takeDamage()`: returns `true` if game ends. Decrements `S.lives`, triggers red flash + camera shake (0.35 s), grants 1.5 s invincibility. During invincibility, player flashes at 22 Hz and blue border overlay pulses
- `S.lives === 1` triggers red vignette + pulse animation
- Shield absorbs a hit (no life lost), triggers cyan flash + particle burst

**Level progression (5 levels)**

| # | Name | `dist` threshold | Visual character |
|---|------|------|------|
| 0 | CALM REEFS | 0 | Warm sandy sky, teal water, bright sun |
| 1 | STORMY CURRENTS | 350 | Dark blue-grey sky, heavier fog, dim sun |
| 2 | ANCIENT ABYSS | 750 | Near-black deep sea, blue bioluminescence, very dense fog |
| 3 | TREASURE PALACE | 1200 | Purple twilight, warm golden accent lights |
| 4 | BOSS ESCAPE | 1800 | Blood red sky and fog, intense red ambient |

Level transitions: sky background is a fresh `makeSky()` canvas gradient texture (2×512 px). Fog color, fog density, ambient light color, sun color/intensity, and wall color all lerp via `applyLevelLerp(frac, prev, cur)` using `S.levelFrac` (0→1 over ~1.25 s). Level-up fires white flash + stinger SFX + `announceLv()` overlay.

Bioluminescence scaling: `coralPurple`, `coralCyan`, `anemone`, `anemone2` emissive intensity scales with `(S.level-1)*0.4`, so deeper levels glow brighter.

**Sound (all Web Audio API, procedural)**

`SFX` is an IIFE object initialized on first user interaction (required by browsers). No audio files.

| Method | What it plays |
|--------|--------------|
| `jump(power)` | Sine sweep down (180 + power×350 Hz → 75 Hz) |
| `land()` | Soft sine thud |
| `swish()` | Quick sine sweep up |
| `collect(combo)` | Sine sweep up, pitch scales with combo |
| `gold()` | Three-note triangle chord staggered 60 ms |
| `hit()` | Bandpass noise burst + sine drop |
| `boost()` | Sawtooth sweep up |
| `levelUp()` | Four-note square chord staircase |
| `shield()` | Sine up sweep |
| `death()` | Bandpass noise + sawtooth sweep down |
| `ambientStart()` | Looping white-noise lowpass drone at 0.04 gain (80 Hz cutoff) |
| `ambientStop()` | Stops the ambient drone |

**Rendering — Three.js r128** *(graphics overhaul July 2026)*
- `ACESFilmicToneMapping`, exposure 1.05
- **Bloom post-processing**: EffectComposer + UnrealBloomPass (0.42/0.6/0.85) loaded from jsdelivr CDN examples/js with try/catch fallback to plain `ren.render`
- **Soft shadows**: PCFSoftShadowMap; sun casts (1024 map, ortho ±30); player, obstacles, wall blocks cast; track floor receives
- **Environment cubemap**: 6 tiny gradient canvases → `scene.environment` for PBR reflections (deliberately dark — bright values wash the whole scene)
- **Procedural bump textures**: `rockBump` (walls/rocks/bg), `scaleBump` (fish-scale rows on Warplet body). Character body materials are now `MeshPhysicalMaterial` with mild clearcoat
- **Background spires**: `makeSpire()` LatheGeometry towers with coral shelf discs (replaces plain bg rocks; 10 objects, `userData.side` for recycle)
- **Sunken temple landmark**: `makeTemple()` — stepped base, columns (some broken), golden pediment + PointLight; scrolls at 0.5× parallax, recycles ~every 650 dist
- Wall blocks: subdivided BoxGeometry(2,2,4), position-hashed displacement (keeps seams welded), smooth normals — no more flatShading
- Caustic floor canvas: sandy-teal solid base + fixed grain speckles + brighter caustic webs (solid base only — gradients tile as visible squares)
- `sortObjects = true` (needed for transparent jellyfish/shield/trail)
- Pixel ratio capped at 2
- 5 lights: `ambL` (ambient), `sunL` (directional sun, pos 3,80,-40), `rimL` (directional rim from behind camera), `charL` (point light follows player, 0x60c0ff, radius 12), `bounceL` (teal point below scene, radius 60)
- Camera: `PerspectiveCamera(70)` at position (0, 5.5, 11) looking at (0, 2.2, -30)

**Atmospheric layers (CSS + Three.js)**
- `#water-surface` div: gradient + `surfRipple` animation (top 20% of screen)
- `#caustics` div: static CSS radial gradients drifting with `caustDrift` animation (pure CSS, overlay blend)
- `caustic` IIFE: **separate Canvas 2D texture** updated every other frame — 16 animated ellipses on a dark teal base. Applied to `M.track.map` so the floor looks like it has light caustics. The CSS `#caustics` div and this Three.js texture are independent.
- `#vignette` div: dark elliptical gradient; becomes red + pulsing at 1 life
- `#screen-flash` div: full-white or full-red flash with 0.1 s CSS transition

**World / environment**
- 14 segments (`C.segCount`), each `C.segLen = 35` units long, recycled as they pass z > 35
- Each segment: floor track (8×0.85×35 box with caustic texture), 7 cross-lines, 3 lane lines, 2 glowing edge strips, random sand patches + small rocks, 2 canyon walls, 2–4 floor coral clusters, 3–7 floor seaweed stems
- Canyon walls (`makeCanyonWall`): 3–5 vertex-displaced box blocks + 2–3 mossy ledges + 2–5 coral clusters + 3–6 hanging kelp groups
- Coral variants (`makeWallCoral`): 5 procedural types — bristle tube cluster, fan disc, branching tree, icosahedron blob, tube cluster — using 8 shared material colors
- Hanging kelp (`makeHangingKelp`): 2–4 strand groups of 4–7 leaf segments that sway per-segment in tick. All kelp leaves pushed into `decos[]`
- 8 background rocks (`makeBgRocks`): large displaced-box rocks on both sides at fixed Z intervals, parallax-scroll at 0.5× speed; recycle when z > 50
- 8 ambient jellyfish: bob and drift in far background, never scroll
- 120 plankton particles: slowly drift, opacity pulses
- 18 god rays: translucent cone meshes, slide slowly forward, wrap when z > 0
- 80 bubbles: rise with wobble, wrap when y > 32
- 5 fish schools (10–18 fish each): sinusoidal formation swim, scroll at 0.28× game speed

**HUD elements**
- Lives badge (top-left): 3 heart emojis; `lost` class fades/shrinks missing hearts
- Score badge (top-right): SCORE label + value
- Level badge (top-right, below score): level name
- Combo badge (top-left, below lives): shows "×N" when combo ≥ 2
- Speed bar (right edge, vertical): fill height = `(speed - baseSpeed) / (maxSpeed - baseSpeed)`
- Charge bar (bottom center): appears on hold, fills to 100%
- Boost banner (bottom center): "RAINBOW BOOST!" with multi-color glow; pulses while active
- Shield badge (top-right, below level): cyan, pulses while active
- Lane warning flashes: 3 `#lane-warns` divs, left-column gradient flashes on obstacle warning
- Score popups (`showPop`): positioned at center-screen, float up and fade in 0.9 s
- Level announce overlay: scales in from 2× then fades out over 2.2 s

---

## What is NOT implemented (planned or mentioned elsewhere)

**Runway ML video backgrounds** — The user mentioned these but they do **not exist in `blue_run_v3.html`**. There is no `<video>` element, no Runway API calls, no video texture on any material. This is a planned future feature. If implementing: the approach would be to use a Three.js `VideoTexture` on the scene background or a fullscreen background mesh, fed by a `<video>` element playing a Runway-generated loop. Needs to happen before `scene.background = makeSky()` or replace it.

**Sunken temple / hero landmarks** — Referenced extensively in `Meshy-Unity-Build-Guide.md` and reference art, but no temple geometry exists in the HTML game. The canyon walls use only abstract displaced boxes.

**Boss chase sequence** — Mentioned in `blue-run-references/purpoe and contexts.txt` ("boss chase sequence" in the HTML-only version). Not in v3.

**Bipedal walking** — The Warplet spec describes stubby webbed legs. `buildWarplet()` includes two tiny pelvic fin proxies (`for(let s=-1;s<=1;s+=2)` two `bodyLight` cones at y=-0.6) but no legs or leg animation. The character is purely a swimmer.

**Level-specific wall materials** — `C.levels[].wallColor` exists and lerps during transition, but `M.wallDark`, `M.wallMoss`, and coral materials do not change with level. Only `M.wall.color` lerps. Deep level visual differentiation is mostly fog + lighting.

**`allDisposable` array** — Declared at line 484 (`const allDisposable = [];`) but never populated or used. Dead code.

**`dist2()` function** — Declared at line 1075 but never called. Dead code.

---

## The Warplet — character spec and `buildWarplet()` details

The Warplet is a chunky bipedal fish with cartoon Pixar energy: spherical blue body, googly eyes, toothy grin, spiny dorsal mohawk, forked tail, pectoral fins like little arms. It faces **-Z** (into the scene) and swims forward.

### `buildWarplet()` — mesh anatomy

All geometry assembled into `w` (outer group) → `inner` (animated sub-group that bobs and body-undulates):

| Part | Geometry | Material | Notes |
|------|----------|----------|-------|
| Body | `SphereGeometry(0.82, 32, 24)` scaled (0.95, 1, 1.15) | `M.body` (0x3090d0) | Main blue sphere |
| Belly | `SphereGeometry(0.68, 20, 16)` scaled (0.78, 0.65, 0.95) | `M.belly` (0x80c8e8) | Light belly patch, offset y=-0.22 |
| Mouth cavity | `SphereGeometry(0.42, 16, 12)` scaled (1.1, 0.8, 0.7) | `M.mouthInside` (dark red) | Position z=-0.72 forward |
| Tongue | `SphereGeometry(0.22, 10, 8)` scaled (1, 0.5, 0.8) | `M.tongue` (pink) | Inside mouth at y=-0.30 |
| Upper jaw | `TorusGeometry(0.38, 0.08, 8, 16, Math.PI)` | `M.body` | Half-torus forming upper lip |
| Lower jaw | `TorusGeometry(0.36, 0.07, 8, 16, Math.PI)` | `M.body` | Half-torus forming lower lip |
| Upper teeth | 7 `ConeGeometry(0.035, 0.14, 5)` | `M.tooth` (ivory) | Arrayed across upper jaw, rotated down |
| Lower teeth | 5 `ConeGeometry(0.03, 0.11, 5)` | `M.tooth` | Arrayed across lower jaw |
| Eyes (×2) | `SphereGeometry(0.22, 16, 14)` white + `SphereGeometry(0.10, 12, 10)` pupil + `SphereGeometry(0.045, 8, 8)` highlight | `M.eyeW`, `M.eyeP`, `M.eyeH` | Left side = -1, right = +1; highlight offset toward camera |
| Brows (×2) | `BoxGeometry(0.22, 0.05, 0.07)` | `M.brow` (dark blue) | rotation.z ±0.25 at rest; animate with `S.boost` |
| Dorsal mohawk | 8 `ConeGeometry(0.05, h, 5)` | `M.bodyDark` | Graded heights along top-back, z 0.35→-0.49 |
| Tail base | `SphereGeometry(0.22, 8, 6)` scaled (0.5, 0.7, 1.2) | `M.body` | Wrist connecting body to tail at z=0.85 |
| Tail forks (×2) | `ConeGeometry(0.12, 0.45, 5)` | `M.bodyDark` | Upper fork y=+0.18 rotation.x=-0.6; lower y=-0.18 rotation.x=+0.6 |
| Pectoral fins (×2) | `ConeGeometry(0.14, 0.45, 6)` | `M.bodyLight` | Left at x=-0.68, right at x=+0.68; principal animation joints |
| Scale arcs (×5) | `TorusGeometry(0.62-i*0.07, 0.012, 4, 24, Math.PI*1.2)` | `M.bodyDark` | Decorative scale arcs along body |
| Pelvic stubs (×2) | `ConeGeometry(0.06, 0.22, 4)` | `M.bodyLight` | Tiny leg/foot proxies at y=-0.6 |
| Shield aura | `SphereGeometry(1.3, 16, 12)` | `M.shieldAura` (transparent cyan) | `visible=false` until shield active |

### `player.userData` refs (for animation in tick loop)

```
inner        — inner group; body-undulation bob Y, subtle rotation.y
tailG        — tail fork group; rotation.y = main swim wag
lFin/rFin    — pectoral fins; rotation.z + rotation.y alternate with swimPhase
dorsalG      — dorsal fin group; subtle rotation.z
lBrow/rBrow  — brow boxes; rotation.z animates with boost/swim
mouth        — mouth sphere; scale.y breathes
lEye/rEye    — {e: eyeWhite mesh, p: pupil mesh}; scale on scaredFrac
vy           — current vertical velocity
baseY        — rest Y position (2.0)
grounded     — jump gate
swimPhase    — accumulates dt*7 (normal) or dt*14 (boost)
aura         — shield bubble mesh
```

### Player animation (tick loop)

```js
// swimPhase drives all animation
ud.inner.rotation.y = sin(sp) * 0.08          // body undulation
ud.tailG.rotation.y = sin(sp - 0.8) * 0.45   // tail wag
ud.tailG.scale.y    = 1 + sin(sp*2) * 0.12   // tail flex
ud.lFin.rotation.z  = PI/2+0.5+sin(sp)*0.4   // left pec stroke
ud.rFin.rotation.z  = -PI/2-0.5+sin(sp+PI)*0.4 // right pec (antiphase)
ud.dorsalG.rotation.z = sin(sp-0.3) * 0.06   // dorsal sway
ud.inner.position.y = sin(sp*0.5) * 0.04     // gentle bob
// Brow driven by boost
ud.lBrow.rotation.z = -(S.boost?0.4:0.25) + sin(sp*0.3)*0.05
// Mouth breathes
ud.mouth.scale.y = 0.75 + sin(sp*0.8) * 0.08
// Eye fright scales with nearest obstacle distance
scaredFrac = max(0, 1 - nearestObstDist/8)
lEye.p.scale → 1 + scaredFrac*0.5
lEye.e.scale → 1 + scaredFrac*0.15
// Squash-stretch from jump
vr = vy / maxJumpV
stretch = 1 + |vr|*0.3; squash = 1/sqrt(max(0.5, stretch))
player.scale.set(squash, max(0.65, stretch), squash)
player.rotation.x = vr * 0.35
// Bank on lane change
player.rotation.z = -(targetLaneX - player.position.x) * 0.08
```

---

## Architecture — full code map

Single `<script>` IIFE. Section order:

1. **Title bubbles** (25 animated `div.tbubble` elements)
2. **`SFX`** — Web Audio API sound system IIFE
3. **`ParticleSystem` class** — `PS.burst(pos, colors, count, spread, upBias)` / `PS.update(dt)` / `PS.clear()`
4. **`C`** — config object (immutable constants)
5. **`S` / `resetState()`** — mutable game state
6. **Three.js setup** — `scene`, `cam`, `ren`, `makeSky()`, fog, 5 lights
7. **`M`** — materials object (all ~42 shared materials; do not create per-mesh materials)
8. **`buildWarplet()`** / `player` — character construction
9. **Array declarations** — `segs, obs, cols, godRays, bubbles, fishArr, trail, plankton, ambJellies, decos, allDisposable`
10. **`makeWallCoral()`** — 5 coral type variants
11. **`makeHangingKelp()`** — multi-strand hanging kelp; pushes leaves to `decos[]`
12. **`makeCanyonWall(side, z)`** — wall group with blocks, ledges, coral, kelp
13. **`makeBgRocks()`** / `bgRocks[]` — 8 large static-then-parallax rocks
14. **`caustic` IIFE** — Canvas 2D animated floor texture
15. **`initAmbJellies()`, `initPlankton()`, `initGodRays()`, `initBubbles()`, `makeFishBody()`, `initFish()`, `initTrail()`** — one-time decorative pool init
16. **`makeSeg(z)`** — segment construction + populates `decos[]` with floor kelp
17. **`findSpawnZ(seg)`** — spacing helper
18. **`makeMine(seg)`**, **`makeBarrier(seg)`**, **`makeObs(seg)`** — obstacle factory
19. **`makePearl()`, `makeGold()`, `makeShieldPickup()`, `makePearlChain()`** — collectible factories
20. **`disposeObj(obj)`**, **`initWorld()`** — world lifecycle
21. **Input** — keyboard listeners, touch listeners, `doJump()`
22. **UI helpers** — `$s`, `$lv`, etc. DOM refs; `triggerLW()`, `popB()`, `showPop()`, `flash()`, `announceLv()`, `updateLivesHUD()`, `updateHUD()`
23. **High scores** — `getScores()`, `saveScore()`
24. **`lerpHex()`, `applyLevelLerp()`** — level color transition
25. **Game flow** — `startGame()`, `takeDamage()`, `endGame()`, button wires
26. **`dist2()`, `dist3()`** — math helpers (`dist2` unused)
27. **`loop(now)` / `requestAnimationFrame`** — main tick (dt capped 80 ms)
28. **Startup** — `initWorld()`, load best score, `loop(0)`

### Global arrays: which persist vs reset

| Array | On `initWorld()` | On session start |
|-------|------------------|-----------------|
| `segs` | Cleared + rebuilt | Same |
| `obs` | Cleared | Same |
| `cols` | Cleared | Same |
| `decos` | Cleared (kelp refs from old segs) | Same |
| `godRays` | Not cleared (init-once guard) | Persistent |
| `bubbles` | Not cleared | Persistent |
| `fishArr` | Not cleared | Persistent |
| `trail` | Not cleared | Persistent |
| `plankton` | Not cleared | Persistent |
| `ambJellies` | Not cleared | Persistent |
| `bgRocks` | Never cleared | Persistent |

### Tick loop structure (what runs each frame)

```
if (S.on):
  update S.dist, S.speed, S.score
  boost timer, combo decay, shield timer, invincibility
  level detection → applyLevelLerp
  charge accumulation
  lane smoothing (lerp toward target lane)
  physics (gravity, ground check)
  squash-stretch + rotation
  animation (swimPhase, all ud.* refs)
  scared-eye scaling
  charL position sync
  segment recycling (z > 35 → disposeObj + makeSeg)
  obstacle update (per-kind animation + z advance + collision)
  collectible update (bob + sparkle + magnet + collision)
  trail update (S.boost → position trail spheres)
  PS.update(dt)
  updateHUD()
  camera FOV lerp

always (even on title/game-over screens):
  caustic.update(t)
  bioluminescence emissive scaling
  godRays opacity/drift
  bubbles rise + wrap
  fishArr sinusoidal swim
  plankton drift + opacity
  ambJellies bob
  decos kelp sway
  bgRocks parallax scroll
  camera follow + shake
  ren.render(scene, cam)
```

---

## Visual style and design decisions

**Aesthetic target:** Pixar-adjacent stylized underwater — saturated teal-to-gold color palette, soft rounded forms, warm sun shafts from above, bioluminescent accents in deeper levels. Reference images in `blue-run-references/` are the quality benchmark. More detail is always better.

**Color philosophy:**
- Surface: warm amber/gold sky (`#ffeecc` top) bleeding into bright teal
- Mid-water: teal `#30b8c8` → deep navy `#0c4868`
- Deep levels go pure dark; BOSS ESCAPE is blood-red
- Character: bright `#3090d0` blue — high saturation to pop against teal environment
- Collectibles: pearl = white/silver, gold coin = `#ffd700`, shield = cyan
- Obstacles: dark/threatening — urchin black-red, rock grey, jelly pink/translucent, mine dark metal with orange-red spikes

**Typography:** Luckiest Guy for all game text (chunky, bold, arcade feel). Nunito 900 for subtitles/labels. Gold `#ffd700` as accent everywhere.

**HUD design:** Angry Birds-style chunky pill badges with drop-shadow + inner highlight. Red gradient for lives/score, blue for level, orange for combo, cyan for shield. All badges pop-scale on update.

**Score popup:** floating text that scales up then drifts upward and fades — explicitly preserved feature, do not remove or change behavior.

**Camera:** Slightly above and behind player (y=5.5, z=11) looking down-forward. Soft horizontal follow prevents whipping. Vertical tracks jump. FOV increase on speed/boost sells velocity without moving camera.

---

## Future direction: Unity rebuild with Meshy.ai

The HTML game is a prototype. The production version is planned as a **Unity URP project** using Meshy.ai-generated 3D assets.

**Asset pipeline:** `blue-run-references/Meshy-Unity-Build-Guide.md` specifies every environment asset with reference images and Meshy prompt starters. Key assets: bioluminescent jellyfish, anemone/coral clusters, table corals, canyon wall chunks, sunken golden temple (hero landmark), lantern power-up, pearl currency orb, stone columns.

**Warplet in Unity:** `blue-run-references/Main-Character-Build.md` specifies the full Meshy.ai Image-to-3D recipe:
- Mode: Image-to-3D from `warplet_meshy_white_1200.png`
- Style: Stylized (not Realistic)
- Symmetry: On
- Target: 8–15k tris, quad-dominant topology
- Export: FBX with PBR textures into `Assets/Characters/BlueFish/`
- Rig: lightweight custom (~12 bones): root → spine → head/jaw + pec fins + tail + dorsal + legs
- Animations needed: swim idle, turn lean, chomp/collect, hit recoil, dash/boost, idle bob
- Material: URP/Lit; optional Fresnel Shader Graph for iridescent purple-blue sheen; emissive eyes

**Unity scene structure:**
- URP pipeline with global Volume: Bloom + Color Grading (saturated, warm highlights) + Vignette + soft DoF
- Exponential fog: saturated teal `~#1FA6A6`, density tuned to fog canyon at mid-distance
- One directional sun light (warm gold, angled steeply from above) + accent point lights for temple/bioluminescence
- Modular canyon segments (same streaming/recycling concept as HTML) dressed with Meshy coral/rock scatter
- Hero landmarks: golden temple as gateway beat at level intervals
- GPU-instanced coral/rock for performance; static batching on non-moving environment

**Runway ML video backgrounds:** Planned feature not yet started. The HTML game currently has no video. When implemented: `<video>` element playing a Runway-generated underwater loop → `THREE.VideoTexture` → applied to `scene.background` or a fullscreen background mesh, replacing the current `makeSky()` canvas gradient. The CSS `#water-surface` and `#caustics` overlays would stack on top as before.

---

## Developer workflow notes

**To test:** Open `blue_run_v3.html` in Chrome. No server needed. F12 console shows any JS errors.

**To iterate:** Edit `blue_run_v3.html` directly. Each major rework is a full rebuild — do not incrementally patch if the change is architectural. The reference images are the quality target.

**Memory management:** Every Three.js object removed from the scene must go through `disposeObj(obj)` to free GPU memory. Segment disposal recursively disposes the entire group tree. Never remove a mesh without disposing it.

**Materials:** All shared materials live in `M`. Adding a new material: add it to `M`, never instantiate inside a geometry-building function (except for per-instance needs like the fish schools and particle system which need individual clones). Shared materials update globally — useful for level color lerps.

**Adding a new obstacle type:** (1) Write a `makeXxx(seg)` factory that pushes to `obs[]` with `userData.kind = 'xxx'`. (2) Add animation branch in the obstacle update section of `tick`. (3) Add collision handling (likely reuse `dist3 < C.hitRadius`). (4) Add to `makeObs()` weighted dispatch. (5) Handle `userData.warned` for lane flash.

**`decos[]` contract:** Any mesh that needs per-frame animation but isn't an obstacle/collectible should be pushed to `decos[]` with identifying `userData` flags. The tick loop at line 1307 reads `userData.kelpSeg` or `userData.isKelp`. Add new animation branches there. Clear `decos.length = 0` in `initWorld()` (already done).
