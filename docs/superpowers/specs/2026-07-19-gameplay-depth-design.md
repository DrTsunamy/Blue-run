# Blue Run – Gameplay Depth Design
**Date:** 2026-07-19  
**Scope:** Gameplay additions for AAA quality upgrade (Phase 1 of 2 — visuals are Phase 2)  
**File:** `blue_run_v3.html`

---

## 1. Level Restructure

The 5 existing levels (indices 0-4 in C.levels[]) keep their names, distance thresholds, and visual palettes. The Kraken is controlled by a separate S.krakenPhase state that evolves by S.dist independently of S.level:

| S.level | Name | Dist | New Gameplay Identity |
|---------|------|------|-----------------------|
| 0 | CALM REEFS | 0 | Tutorial feel - current gameplay, gentle intro |
| 1 | STORMY CURRENTS | 350 | Obstacle density +20%; eel + crab claws introduced |
| 2 | ANCIENT ABYSS | 750 | Ink clouds + stalactites; Kraken silhouette fades in at screen edge (no combat) |
| 3 | TREASURE PALACE | 1200 | Gauntlet - stalactite waves, torpedo fish, collapsing debris. Kraken pursuit + tentacle attacks active. |
| 4 | BOSS ESCAPE | 1800 | Full Kraken assault - ink blasts + shockwaves + maximum obstacle density. Speed cap raised to 32. |

Kraken phase transitions (by dist, independent of S.level):
- dist >= 750  -> krakenPhase = 'silhouette' (faint shadow at z = -200, no gameplay impact)
- dist >= 1000 -> krakenPhase = 'pursuing'   (Kraken at z = -40, speed pressure active)
- dist >= 1200 -> krakenPhase = 'attacking'  (tentacle lane sweeps begin)
- dist >= 1800 -> krakenPhase = 'fullAssault' (ink blasts + shockwaves)

---

## 2. Track Curvature

### Approach: Bend Segments + Canyon Drift

**Bend segments** are periodic special segments (every 4–6 regular segments) that rotate the world direction by ±15–20°. The camera banks 12° in the same direction over 0.8s. Implementation: a `worldYaw` accumulator rotates the spawn direction of new segments; existing segments are unaffected.

**Canyon drift** runs continuously — sinusoidal lateral sway applied to the camera lookAt target (amplitude ±0.6 units, frequency 0.4 Hz).

**New state fields:**
```
S.worldYaw       // current world yaw in radians
S.yawTarget      // target yaw after a bend
S.camBankAngle   // current camera roll in degrees
S.nextBendIn     // countdown in segments until next bend
```

---

## 3. New Obstacles (5 types)

### Electric Eel (kind: 'eel')
Sinuous chain of 8 sphere segments across one lane, glowing cyan-yellow. Charges 1.5s before zapping — that lane becomes impassable for 2s. Introduced level 1.

### Ink Cloud (kind: 'inkCloud')
5–8 translucent black spheres. Collision triggers `S.inked = true` for 1.5s — screen goes near-black, player can still move. Introduced level 2.

### Crab Claws (kind: 'crabClaw')
Two claw arms that close on a 2s timer (open 1.4s → snap 0.2s → reopen 0.4s). Shadow projected downward as warning. Safe window is the open phase. Introduced level 1.

### Torpedo Fish (kind: 'torpedo')
Spawns off-screen at x = ±20, fires horizontally across all 3 lanes in 0.9s. Red streak appears at lane edge 1.2s before impact. Can fire 1–3 in sequence. Introduced level 4.

### Stalactite (kind: 'stalactite')
Rock cone hanging from top, falls when player enters z > -12. Circular shadow on floor is the warning. Level 4 spawns in waves of 2–3. Introduced level 2.

---

## 4. New Power-ups (5 types)

### Speed Dash (kind: 'dash')
Gold-orange arrow. 3× speed + full invincibility for 0.8s. Trail turns orange-gold. No lane change during dash.

### Time Slow (kind: 'timeSlow')
Purple hourglass. All obstacles/collectibles slow 25% for 3s. Player speed unaffected. Subtle blue-purple screen tint. `S.timeSlow` flag multiplies obstacle velocity by 0.75.

### Double Points (kind: 'doublePoints')
Glowing green ×2 badge. All collectible point values doubled for 8s. "×2 POINTS" banner below combo badge.

### Decoy Fish (kind: 'decoy')
Miniature Warplet (0.4× scale, cyan glow) orbiting player at radius 1.5. Absorbs one hit then disappears with burst. Not mutually exclusive with shield.

### Mini Shrink (kind: 'shrink')
Pink potion. Player shrinks to 0.5× scale for 5s. Collision radius halved. Pearls collected while tiny grant +5 bonus each. "TINY" badge in pink.

---

## 5. The Kraken

### Construction (makeKraken())
- **Body**: SphereGeometry(4, 16, 12) scaled (1, 0.7, 1.4) — squat indigo ovoid
- **Eyes** (×2): white sphere + yellow slit pupil (SphereGeometry scaled 0.3, 1, 0.3)
- **Mantle**: ConeGeometry(3.5, 5, 8) above body — pointed squid head
- **8 tentacles**: Chain of 6 sphere joints with decreasing radius, sinusoidal phase offsets
- **2 attack tentacles**: Longer arms that reach lane positions during attacks
- Material: MeshPhysicalMaterial dark indigo (0x1a0a2e), emissive purple (0x6020a0)

### Phases

| Phase | krakenPhase | Behavior | Trigger |
|-------|-------------|----------|---------|
| 0 | none | No Kraken | dist < 750 |
| 1 | silhouette | Faint shadow at z = -200, grows slowly | dist 750–1000 |
| 2 | pursuing | At z = -40, closes gap if speed drops | dist 1000+ |
| 3 | attacking | Tentacles sweep individual lanes (1.2s warning) | level 4 |
| 4 | fullAssault | Ink blasts + shockwave rings | level 5 |

### Z Position Logic
```
speed < maxSpeed * 0.7  → krakenZ lerps toward -25 (danger close)
speed > maxSpeed * 0.85 → krakenZ lerps toward -55 (breathing room)
krakenZ never closer than -18
krakenZ > -15 → takeDamage() (caught)
```

### Tentacle Lane Attack (phase 3+)
Flash lane warn 1.2s before → attack tentacle slams across that lane's X at y = 1 → -1 → collision check → retracts over 0.6s.

### Ink Blast (phase 4)
Travels at 1.8× player speed. Triggers `S.inked`. 3s cooldown.

### Shockwave (phase 4)
Expanding flat torus from Kraken. Sweeps y = 0.5 — player must jump to clear it. 8s cooldown.

---

## 6. Scoring Adjustments

- Pursuer survival bonus: +50 pts/second while `krakenPhase >= 2`
- Mini Shrink pearl bonus: +5 per pearl collected while tiny
- Everything else unchanged

---

## 7. Implementation Order

1. Canyon drift + bend segments
2. Kraken silhouette (phase 1 — visual only)
3. New power-ups (additive to cols[] system)
4. New obstacles — stalactite + crab claws first, then eel + ink cloud + torpedo
5. Kraken pursuit (phase 2)
6. Kraken tentacle attacks (phase 3)
7. Kraken full assault (phase 4)

---

## 8. Constraints (unchanged)

- Single HTML file, no imports, no npm, no build
- Three.js r128 CDN
- All existing gameplay preserved
- New materials go into M object
- New animated decoratives go into decos[]
- All removed objects go through disposeObj()

