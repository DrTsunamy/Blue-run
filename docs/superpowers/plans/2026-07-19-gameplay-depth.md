# Blue Run Gameplay Depth Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the Kraken pursuer, 5 new obstacles, 4 new power-ups, and stronger visible track curves to index.html.

**Architecture:** All changes go into the single self-contained `index.html` (1,995 lines). Every feature follows existing patterns: obstacles push to `obs[]` with `userData.kind`, collectibles to `cols[]`, shared materials in `M`, state flags in `S` via `resetState()`, animation/collision branches in the main `loop()`. The Kraken is a singleton scene object with a distance-driven phase machine.

**Tech Stack:** Three.js r128 (CDN), Web Audio API, vanilla JS IIFE. No build, no imports, no test framework — verification is manual browser testing with a dev hotkey.

## Global Constraints

- Single file: ALL changes in `index.html`. No new files, no imports, no npm.
- Three.js r128 API only (no BufferGeometryUtils, no capsules — use spheres/cones/cylinders/torus).
- New shared materials go in the `M` object (line ~610). Per-instance materials only when animated individually.
- Every object removed from scene MUST call `disposeObj(obj)` first (line ~1356 pattern: `scene.remove(o);disposeObj(o);arr.splice(i,1)`).
- New state fields MUST be added to `resetState()` (line ~422) so restart fully resets.
- The curved-world system auto-bends all materials via the patched `scene.add` (line ~548) — nothing special needed for new meshes.
- Existing gameplay must stay intact: run the smoke test (Task 0 verify) after every task.
- Git: commit after each task with the exact message given. Working file is `index.html` on branch `master`.

## Existing code map (for orientation)

| Thing | Line (approx) | Notes |
|---|---|---|
| `C` config | 403 | freqs: obFreq .38, pearlFreq .42, goldFreq .10, shieldFreq .06, magnetFreq .05, x2Freq .045 |
| `resetState()` | 422 | S fields |
| `CURVE` + `bendMaterial` | 527 | curved world uniforms |
| `M` materials | 610 | shared materials |
| `SFX` | ~260 | osc(type,freq,freqEnd,dur,vol,delay), noise(dur,vol,filterFreq) |
| obstacle factories | 1150-1290 | makeMine, makeBarrier, makeSweeper, makeObs dispatch |
| collectible factories | 1292-1353 | makePearl, makeGold, makeShieldPickup, makeMagnetPickup, makeX2Pickup |
| `initWorld()` | 1362 | initial spawn rolls |
| input keydown | 1390 | |
| UI $ refs | 1438 | $mb, $x2b, $shb pattern |
| `takeDamage()` | 1555 | returns true if game over |
| main `loop()` | 1620 | spd calc 1630, curve retarget 1650, obstacle loop 1775, collectible loop 1836, isPowerup check 1847, collect dispatch 1861-1893 |
| segment recycle spawns | 1758-1771 | per-segment spawn rolls |

---

### Task 0: Dev test hotkey + smoke test baseline

**Files:**
- Modify: `index.html` keydown handler (~line 1390)

**Interfaces:**
- Produces: Shift+K hotkey that adds 250 to `S.dist` mid-run. Used by every later task to reach high-level content fast.

- [ ] **Step 1: Add the hotkey** — inside the keydown listener, after the fast-fall line, add:

```js
  // DEV: skip ahead to test later levels (Shift+K)
  if(e.code==='KeyK'&&e.shiftKey){S.dist+=250;}
```

- [ ] **Step 2: Verify** — Open index.html in Chrome, start a run, press Shift+K four times. Expected: level announcements fire in sequence (STORMY CURRENTS → ANCIENT ABYSS → TREASURE PALACE), sky/fog change, no console errors (F12).

- [ ] **Step 3: Smoke test baseline** — one full manual run: lanes, jump, charge-jump boost, collect pearls, hit an obstacle, die, restart. Everything works as before.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "dev: add Shift+K distance skip hotkey for testing"
```

---

### Task 1: Stronger, more visible track curves

**Files:**
- Modify: `index.html` curve retarget block (~line 1650) and curve ease (~line 1930)

**Interfaces:**
- Consumes: existing `S.curveTimer/curveTgtX/curveTgtY`, `CURVE` uniforms.
- Produces: nothing new — tuning only.

The current bend is too subtle (user feedback: "need at least two or three curves visible"). Increase amplitude, retarget more often, ease faster.

- [ ] **Step 1: Replace the retarget block** (~line 1652):

```js
    // Winding canyon: retarget the world bend every few seconds.
    // Negative Y targets are cascade drops — the track ahead dives away.
    S.curveTimer-=dt;
    if(S.curveTimer<=0){
      S.curveTimer=3.5+Math.random()*3.5;
      // Alternate direction more often than not so the canyon reads as S-curves
      const dir=(S.curveTgtX>=0?-1:1)*(Math.random()<0.75?1:-1);
      S.curveTgtX=dir*(1.6+Math.random()*1.8);
      S.curveTgtY=Math.random()<0.35?-(0.5+Math.random()*0.9):(Math.random()-0.5)*0.8;
    }
```

- [ ] **Step 2: Faster ease** (~line 1931) — change both `0.012` factors to `0.02`:

```js
  CURVE.x.value+=((curveOn?S.curveTgtX:0)-CURVE.x.value)*0.02;
  CURVE.y.value+=((curveOn?S.curveTgtY:0)-CURVE.y.value)*0.02;
```

- [ ] **Step 3: Verify** — start a run, watch 30 seconds. Expected: canyon ahead visibly snakes left/right (2-3 direction changes visible), occasional dives. Collisions still exact (bend is view-space only). No nausea — if it feels too strong, reduce 1.6+1.8 to 1.3+1.4.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: stronger S-curve winding canyon (visible curves)"
```

---

### Task 2: Speed Dash power-up

**Files:**
- Modify: `index.html` — C config, resetState, HUD CSS/HTML, $ refs, collectible factory, spawn rolls, collect dispatch, spd calc, damage guard, SFX

**Interfaces:**
- Consumes: `cols[]` pattern, `isPowerup` check line 1847, spd calc line 1630.
- Produces: `S.dashT` (seconds remaining), `makeDashPickup(seg)`, `C.dashFreq`, SFX.dash().

- [ ] **Step 1: Config + state** — in `C` add `dashFreq: 0.04,` after `x2Freq`. In `resetState()` add `dashT:0,` after `x2:false, x2T:0,`.

- [ ] **Step 2: HUD badge** — CSS after `#x2-badge.show` rule:

```css
    #dash-badge { top:280px; right:18px; background:linear-gradient(180deg,#ffa030,#e06010 50%,#a04000); border-color:#fff; font-size:14px; padding:5px 14px; opacity:0; transition:opacity .3s; }
    #dash-badge.show { opacity:1; animation:shieldPulse 1s ease-in-out infinite alternate; }
```

HTML after the x2-badge div: `<div id="dash-badge" class="hud-badge">💨 DASH</div>`
$ refs after `$x2b=...` line: `$db=document.getElementById('dash-badge'),`

- [ ] **Step 3: Factory** — after `makeX2Pickup` (~line 1347):

```js
function makeDashPickup(seg){
  const lane=Math.floor(Math.random()*3);
  const g=new THREE.Group();
  const aMat=new THREE.MeshStandardMaterial({color:0xffa030,roughness:0.15,metalness:0.7,emissive:0xe06010,emissiveIntensity:0.7});
  // Double chevron arrow
  for(let i=0;i<2;i++){
    const ar=new THREE.Mesh(new THREE.ConeGeometry(0.22,0.34,4),aMat);
    ar.rotation.x=-Math.PI/2; ar.position.z=-0.12-i*0.3; g.add(ar);
  }
  g.add(new THREE.Mesh(new THREE.TorusGeometry(0.45,0.035,6,20),new THREE.MeshBasicMaterial({color:0xffc060,transparent:true,opacity:0.45})));
  g.position.set(C.lanes[lane],2.0,seg.position.z+(Math.random()-0.5)*C.segLen*0.4);
  g.userData={kind:'dash',got:false,baseY:2.0,phase:Math.random()*Math.PI*2};
  scene.add(g); cols.push(g);
}
```

- [ ] **Step 4: Spawn rolls** — add `if(Math.random()<C.dashFreq) makeDashPickup(sg);` in `initWorld()` after the x2 roll, and `if(Math.random()<C.dashFreq) makeDashPickup(newSg);` in the segment-recycle block after its x2 roll.

- [ ] **Step 5: isPowerup + collect dispatch** — extend line 1847:

```js
      const isPowerup=['shield','magnet','x2','dash'].includes(c.userData.kind);
```

Add branch after the `'x2'` branch in the collect dispatch:

```js
        } else if(c.userData.kind==='dash'){
          S.dashT=0.9;
          $db.classList.add('show');
          SFX.dash();
          flash('#ffa030',120);
          PS.burst(c.position.clone(),[0xffa030,0xffd060,0xffffff],14,5,4);
          showPop('DASH!',screenX,screenY,'#ffa030');
```

- [ ] **Step 6: Dash physics** — in the spd calc (line ~1630) replace `const spd=S.boost?S.speed*1.6:S.speed;` with:

```js
    let spd=S.boost?S.speed*1.6:S.speed;
    if(S.dashT>0){ spd*=2.2; S.dashT-=dt; if(S.dashT<=0)$db.classList.remove('show'); }
```

Damage guard — in the obstacle collision branch (line ~1816) change `if(!S.invincible){ if(takeDamage()) return; }` to:

```js
        if(!S.invincible&&S.dashT<=0){ if(takeDamage()) return; }
```

Also in the barrier branch (~1802) change `if(playerLane!==o.userData.safeLane&&!S.invincible)` to `if(playerLane!==o.userData.safeLane&&!S.invincible&&S.dashT<=0)`.

Orange trail during dash — in the trail block (~1902) change `if(S.boost){` to `if(S.boost||S.dashT>0){` and the trail opacity line `const tgt=S.boost?tr.userData.tgtOp:0;` to `const tgt=(S.boost||S.dashT>0)?tr.userData.tgtOp:0;`

- [ ] **Step 7: SFX** — in the SFX return object after `boost(){...},`:

```js
    dash(){ osc('sawtooth', 200, 900, 0.3, 0.1); noise(0.2, 0.1, 900); },
```

- [ ] **Step 8: Verify** — run, find the orange chevron pickup (Shift+K if needed; freq is 4% per segment so ~1 per 25 segments — temporarily set dashFreq to 0.5 to test, then RESTORE to 0.04). Expected: 0.9s burst of extreme speed, immune to obstacles, orange-ish trail, badge shows/hides, no errors.

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "feat: speed dash power-up (0.9s burst + invincibility)"
```

---

### Task 3: Time Slow power-up

**Files:**
- Modify: `index.html` — same integration points as Task 2

**Interfaces:**
- Consumes: spd calc (after Task 2 it is `let spd=...`).
- Produces: `S.timeSlowT`, `makeTimeSlowPickup(seg)`, `C.timeSlowFreq`, SFX.timeSlow().

Implementation note: "obstacles slow 25%, player unaffected" translates in a scroll-runner to slowing the WORLD scroll `spd` by 25% while keeping score/dist accrual on the un-slowed value (so the pickup never costs the player points).

- [ ] **Step 1: Config + state** — `C`: add `timeSlowFreq: 0.035,`. `resetState()`: add `timeSlowT:0,`.

- [ ] **Step 2: HUD badge** — CSS:

```css
    #slow-badge { top:346px; right:18px; background:linear-gradient(180deg,#b070ff,#7030d0 50%,#4a1898); border-color:#fff; font-size:14px; padding:5px 14px; opacity:0; transition:opacity .3s; }
    #slow-badge.show { opacity:1; animation:shieldPulse 1s ease-in-out infinite alternate; }
```

HTML: `<div id="slow-badge" class="hud-badge">⏳ SLOW-MO</div>` after dash-badge.
$ ref: `$slb=document.getElementById('slow-badge'),`

- [ ] **Step 3: Factory** — after `makeDashPickup`:

```js
function makeTimeSlowPickup(seg){
  const lane=Math.floor(Math.random()*3);
  const g=new THREE.Group();
  const hMat=new THREE.MeshStandardMaterial({color:0xb070ff,roughness:0.1,metalness:0.6,emissive:0x7030d0,emissiveIntensity:0.7,transparent:true,opacity:0.9});
  // Hourglass: two cones tip-to-tip
  const top=new THREE.Mesh(new THREE.ConeGeometry(0.22,0.3,8),hMat); top.rotation.x=Math.PI; top.position.y=0.15; g.add(top);
  const bot=new THREE.Mesh(new THREE.ConeGeometry(0.22,0.3,8),hMat); bot.position.y=-0.15; g.add(bot);
  g.add(new THREE.Mesh(new THREE.TorusGeometry(0.45,0.035,6,20),new THREE.MeshBasicMaterial({color:0xc898ff,transparent:true,opacity:0.45})));
  g.position.set(C.lanes[lane],2.0,seg.position.z+(Math.random()-0.5)*C.segLen*0.4);
  g.userData={kind:'timeSlow',got:false,baseY:2.0,phase:Math.random()*Math.PI*2};
  scene.add(g); cols.push(g);
}
```

- [ ] **Step 4: Spawn rolls** — `if(Math.random()<C.timeSlowFreq) makeTimeSlowPickup(sg);` in initWorld + same for newSg in recycle block.

- [ ] **Step 5: isPowerup + dispatch** — add `'timeSlow'` to the isPowerup array. Collect branch:

```js
        } else if(c.userData.kind==='timeSlow'){
          S.timeSlowT=3;
          $slb.classList.add('show');
          SFX.timeSlow();
          flash('#b070ff',150);
          PS.burst(c.position.clone(),[0xb070ff,0xd0a0ff,0xffffff],14,4,4);
          showPop('SLOW-MO!',screenX,screenY,'#c898ff');
```

- [ ] **Step 6: World-slow physics** — in the spd calc right after the dash line add:

```js
    if(S.timeSlowT>0){ S.timeSlowT-=dt; spd*=0.75; if(S.timeSlowT<=0)$slb.classList.remove('show'); }
```

IMPORTANT: keep `S.dist+=spd*dt;` as-is BUT move it ABOVE the timeSlow multiply so distance/score accrual is not penalized. Final order in the block:

```js
    let spd=S.boost?S.speed*1.6:S.speed;
    if(S.dashT>0){ spd*=2.2; S.dashT-=dt; if(S.dashT<=0)$db.classList.remove('show'); }
    S.dist+=spd*dt;
    if(S.timeSlowT>0){ S.timeSlowT-=dt; spd*=0.75; if(S.timeSlowT<=0)$slb.classList.remove('show'); }
```

- [ ] **Step 7: SFX** — `timeSlow(){ osc('sine', 800, 200, 0.5, 0.09); },`

- [ ] **Step 8: Verify** — collect the purple hourglass (bump freq to 0.5 temporarily, restore after). Expected: world visibly eases to ~75% scroll for 3s, purple flash, badge, score keeps climbing at full rate, no drift between obstacles and track.

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "feat: time-slow power-up (25% world slowdown, 3s)"
```

---

### Task 4: Decoy Fish power-up

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `buildWarplet()` (reusable — shared materials), obstacle hit branch (after Task 2 edit).
- Produces: `S.decoy` (bool), `decoyFish` scene object (module-level let), `makeDecoyPickup(seg)`, `C.decoyFreq`.

- [ ] **Step 1: Config + state** — `C`: `decoyFreq: 0.03,`. `resetState()`: `decoy:false,`.

- [ ] **Step 2: Decoy object** — after `const player = buildWarplet(); ... scene.add(player);` add:

```js
// Decoy fish: a mini Warplet that orbits the player and absorbs one hit
const decoyFish = buildWarplet();
decoyFish.scale.setScalar(0.4);
decoyFish.visible = false;
scene.add(decoyFish);
```

- [ ] **Step 3: Factory** — after `makeTimeSlowPickup`:

```js
function makeDecoyPickup(seg){
  const lane=Math.floor(Math.random()*3);
  const g=new THREE.Group();
  const fMat=new THREE.MeshStandardMaterial({color:0x40e0ff,roughness:0.2,metalness:0.4,emissive:0x00a0d0,emissiveIntensity:0.8});
  const body=new THREE.Mesh(new THREE.SphereGeometry(0.24,12,10),fMat); body.scale.set(0.9,0.8,1.3); g.add(body);
  const tail=new THREE.Mesh(new THREE.ConeGeometry(0.14,0.3,5),fMat); tail.position.z=0.38; tail.rotation.x=-Math.PI/2; g.add(tail);
  g.add(new THREE.Mesh(new THREE.TorusGeometry(0.45,0.035,6,20),new THREE.MeshBasicMaterial({color:0x80f0ff,transparent:true,opacity:0.45})));
  g.position.set(C.lanes[lane],2.0,seg.position.z+(Math.random()-0.5)*C.segLen*0.4);
  g.userData={kind:'decoy',got:false,baseY:2.0,phase:Math.random()*Math.PI*2};
  scene.add(g); cols.push(g);
}
```

- [ ] **Step 4: Spawn rolls** — same pattern as Tasks 2-3 with `C.decoyFreq`.

- [ ] **Step 5: isPowerup + dispatch** — add `'decoy'` to isPowerup array. Branch:

```js
        } else if(c.userData.kind==='decoy'){
          S.decoy=true; decoyFish.visible=true;
          SFX.powerup();
          PS.burst(c.position.clone(),[0x40e0ff,0x80f0ff,0xffffff],12,4,4);
          showPop('DECOY!',screenX,screenY,'#80f0ff');
```

- [ ] **Step 6: Orbit animation** — in the player animation section of the loop (after the `charL.position.set(...)` line ~1745) add:

```js
    // Decoy fish orbits the player
    if(S.decoy){
      const oa=t*2.4;
      decoyFish.position.set(player.position.x+Math.cos(oa)*1.5, player.position.y+0.4+Math.sin(oa*1.7)*0.3, player.position.z+Math.sin(oa)*1.0);
      decoyFish.rotation.y=Math.sin(oa)*0.5;
      decoyFish.userData.swimPhase+=dt*10;
      const dsp=decoyFish.userData.swimPhase;
      decoyFish.userData.tailG.rotation.y=Math.sin(dsp-0.8)*0.45;
    }
```

- [ ] **Step 7: Hit absorption** — in the obstacle hit branch, BEFORE the shield check, add:

```js
        if(S.decoy){
          S.decoy=false; decoyFish.visible=false;
          S.invincible=true; S.invincibleT=0.8;
          flash('#40e0ff',150); SFX.hit();
          PS.burst(decoyFish.position.clone(),[0x40e0ff,0x80f0ff,0xffffff],16,5,4);
          showPop('DECOY SAVED YOU!',innerWidth/2,innerHeight*0.4,'#80f0ff');
          scene.remove(o); disposeObj(o); obs.splice(i,1); continue;
        }
```

Also in `takeDamage()` nothing changes (decoy intercepts before damage). And in the barrier branch add the same decoy check before its damage line:

```js
          if(playerLane!==o.userData.safeLane&&!S.invincible&&S.dashT<=0){
            if(S.decoy){ S.decoy=false; decoyFish.visible=false; S.invincible=true; S.invincibleT=0.8; flash('#40e0ff',150); SFX.hit(); }
            else if(takeDamage())return;
          }
```

- [ ] **Step 8: Reset on restart** — in `startGame()` (line ~1534) after `S=resetState()`-equivalent logic runs, add `decoyFish.visible=false;` (find where player state is reset and add there).

- [ ] **Step 9: Verify** — collect cyan fish pickup (temp freq bump), see mini Warplet orbiting; run into an urchin: decoy vanishes with burst, no heart lost; second hit costs a heart. Restart clears decoy.

- [ ] **Step 10: Commit**

```bash
git add index.html
git commit -m "feat: decoy fish power-up (orbiting mini-Warplet absorbs one hit)"
```

---

### Task 5: Mini Shrink power-up

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: player scale logic (squash-stretch line ~1714 overwrites `player.scale` every frame — shrink must multiply INTO that line).
- Produces: `S.shrinkT`, `makeShrinkPickup(seg)`, `C.shrinkFreq`, effective-radius helper `hitR()`.

- [ ] **Step 1: Config + state** — `C`: `shrinkFreq: 0.03,`. `resetState()`: `shrinkT:0,`.

- [ ] **Step 2: HUD badge** — CSS:

```css
    #tiny-badge { top:412px; right:18px; background:linear-gradient(180deg,#ff70b8,#d03080 50%,#981858); border-color:#fff; font-size:14px; padding:5px 14px; opacity:0; transition:opacity .3s; }
    #tiny-badge.show { opacity:1; animation:shieldPulse 1s ease-in-out infinite alternate; }
```

HTML: `<div id="tiny-badge" class="hud-badge">🧪 TINY</div>`
$ ref: `$tb=document.getElementById('tiny-badge'),`

- [ ] **Step 3: Factory** —

```js
function makeShrinkPickup(seg){
  const lane=Math.floor(Math.random()*3);
  const g=new THREE.Group();
  const pMat=new THREE.MeshStandardMaterial({color:0xff70b8,roughness:0.1,metalness:0.3,emissive:0xd03080,emissiveIntensity:0.7,transparent:true,opacity:0.9});
  // Potion bottle: sphere body + cylinder neck + cork
  const body=new THREE.Mesh(new THREE.SphereGeometry(0.22,12,10),pMat); g.add(body);
  const neck=new THREE.Mesh(new THREE.CylinderGeometry(0.07,0.09,0.16,8),pMat); neck.position.y=0.24; g.add(neck);
  const cork=new THREE.Mesh(new THREE.CylinderGeometry(0.08,0.08,0.07,8),new THREE.MeshStandardMaterial({color:0xb08048,roughness:0.9})); cork.position.y=0.35; g.add(cork);
  g.add(new THREE.Mesh(new THREE.TorusGeometry(0.45,0.035,6,20),new THREE.MeshBasicMaterial({color:0xff98d0,transparent:true,opacity:0.45})));
  g.position.set(C.lanes[lane],2.0,seg.position.z+(Math.random()-0.5)*C.segLen*0.4);
  g.userData={kind:'shrink',got:false,baseY:2.0,phase:Math.random()*Math.PI*2};
  scene.add(g); cols.push(g);
}
```

- [ ] **Step 4: Spawn rolls** — same pattern, `C.shrinkFreq`.

- [ ] **Step 5: isPowerup + dispatch** — add `'shrink'`. Branch:

```js
        } else if(c.userData.kind==='shrink'){
          S.shrinkT=5;
          $tb.classList.add('show');
          SFX.shrink();
          flash('#ff70b8',120);
          PS.burst(c.position.clone(),[0xff70b8,0xff98d0,0xffffff],12,4,4);
          showPop('TINY!',screenX,screenY,'#ff98d0');
```

- [ ] **Step 6: Shrink scale + hitbox** — add a scale factor into the squash-stretch line. Replace (~line 1714):

```js
    const vr=player.userData.vy/C.maxJumpV;
    const stretch=1+Math.abs(vr)*0.3, squash=1/Math.sqrt(Math.max(0.5,stretch));
    player.scale.set(squash,Math.max(0.65,stretch),squash);
```

with:

```js
    if(S.shrinkT>0){ S.shrinkT-=dt; if(S.shrinkT<=0)$tb.classList.remove('show'); }
    S.sizeF+=((S.shrinkT>0?0.5:1)-S.sizeF)*0.1;
    const vr=player.userData.vy/C.maxJumpV;
    const stretch=1+Math.abs(vr)*0.3, squash=1/Math.sqrt(Math.max(0.5,stretch));
    player.scale.set(squash*S.sizeF,Math.max(0.65,stretch)*S.sizeF,squash*S.sizeF);
```

Add `sizeF:1,` to `resetState()`.

Hitbox: define once near dist3 (~line 1616): `function hitR(){return C.hitRadius*(S.sizeF<0.8?0.6:1);}` and replace the one use of `C.hitRadius` in the obstacle collision (line ~1807) with `hitR()`.

- [ ] **Step 7: Pearl bonus while tiny** — in the pearl collect branch, after `let pts=25*mult;` add `if(S.shrinkT>0)pts+=5;`

- [ ] **Step 8: SFX** — `shrink(){ osc('sine', 400, 1200, 0.25, 0.08); },`

- [ ] **Step 9: Verify** — collect pink potion: player smoothly shrinks to half, squeezes past obstacles that would have hit, pearls give +5 extra, grows back after 5s. Badge toggles.

- [ ] **Step 10: Commit**

```bash
git add index.html
git commit -m "feat: mini-shrink power-up (half size, half hitbox, pearl bonus)"
```

---

### Task 6: Stalactite obstacle

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `findSpawnZ(seg)`, `M.rock`, obs[] pattern, `triggerLW`.
- Produces: `makeStalactite(seg)` with `userData.kind='stalactite'`, states `hang → falling → landed`.

- [ ] **Step 1: Factory** — after `makeSweeper` (~line 1236):

```js
// ─── STALACTITE (hangs above a lane, drops when you get close) ───
function makeStalactite(seg){
  const lane=Math.floor(Math.random()*3);
  const o=new THREE.Group();
  const rockG=new THREE.ConeGeometry(0.45,2.2,7);
  const pos=rockG.attributes.position, seed=Math.random()*100;
  for(let v=0;v<pos.count;v++){
    const x=pos.getX(v),y=pos.getY(v),z=pos.getZ(v);
    const n=1+Math.sin(x*8.3+y*5.1+z*7.7+seed)*0.13;
    pos.setXYZ(v,x*n,y,z*n);
  }
  pos.needsUpdate=true; rockG.computeVertexNormals();
  const spike=new THREE.Mesh(rockG,M.rock);
  spike.rotation.x=Math.PI; // point down
  o.add(spike);
  // Warning shadow on the floor (scales up as it primes)
  const shadow=new THREE.Mesh(new THREE.CircleGeometry(0.55,16),
    new THREE.MeshBasicMaterial({color:0x000000,transparent:true,opacity:0.0,depthWrite:false}));
  shadow.rotation.x=-Math.PI/2;
  o.add(shadow);
  o.position.set(C.lanes[lane],0,findSpawnZ(seg));
  spike.position.y=7.5;
  shadow.position.y=0.46-o.position.y+0.02;
  o.traverse(ch=>{ if(ch.isMesh&&ch.material&&!ch.material.transparent) ch.castShadow=true; });
  o.userData={kind:'stalactite',lane,warned:false,state:'hang',fallV:0,spike,shadow,landT:0};
  scene.add(o); obs.push(o);
}
```

- [ ] **Step 2: Animation + trigger branch** — in the obstacle update loop after the `'sweeper'` branch:

```js
      } else if(o.userData.kind==='stalactite'){
        const st=o.userData;
        if(st.state==='hang'){
          st.spike.position.y=7.5+Math.sin(t*2+o.position.z)*0.06;
          const prox=Math.max(0,1-Math.abs(o.position.z+13)/10);
          st.shadow.material.opacity=prox*0.4;
          if(o.position.z>-13){ st.state='falling'; SFX.stalactite(); }
        } else if(st.state==='falling'){
          st.fallV+=55*dt;
          st.spike.position.y-=st.fallV*dt;
          st.shadow.material.opacity=0.45;
          if(st.spike.position.y<=1.1){
            st.spike.position.y=1.1; st.state='landed'; st.landT=0;
            S.shakeT=0.18; S.shakeAmp=0.06;
            PS.burst(new THREE.Vector3(o.position.x,0.5,o.position.z),[0x585858,0x8a8a8a,0xc8b898],10,4,3);
          }
        } else { // landed — stays as a ground hazard, shadow fades
          st.landT+=dt;
          st.shadow.material.opacity=Math.max(0,0.45-st.landT*0.5);
        }
```

- [ ] **Step 3: Collision** — the stalactite hits only while falling or landed AND spike near player height. Its collision reuses the default `dist3` path, but the mesh position lives on the child. Simplest correct approach: in the collision condition (the `else if` at ~1805 after sweeper), extend the ternary chain to compute a per-kind test. Replace the collision `else if` condition with:

```js
      } else if(
          o.userData.kind==='sweeper'
          ? (Math.abs(player.position.z-o.position.z)<0.85 && player.position.y-player.userData.baseY<1.15 && Math.abs(player.position.x-o.position.x)<1.9)
          : o.userData.kind==='stalactite'
          ? (o.userData.state!=='hang' && Math.abs(player.position.z-o.position.z)<0.8 && Math.abs(player.position.x-o.position.x)<0.9 && o.userData.spike.position.y-player.position.y<2.0)
          : dist3(player.position,o.position)<hitR()){
```

- [ ] **Step 4: Spawn dispatch** — see Task 10 Step 5 for the final `makeObs` weights table (all new obstacles integrated in one edit there). For NOW, test-spawn by temporarily adding at the top of `makeObs`: `if(S.level>=1&&Math.random()<0.3){makeStalactite(seg);return;}` — REMOVE after verify (Task 10 integrates properly).

- [ ] **Step 5: SFX** — `stalactite(){ noise(0.3, 0.2, 300); osc('square', 90, 45, 0.25, 0.07); },`

- [ ] **Step 6: Verify** — Shift+K to level 1+. Watch a stalactite: hangs high with growing floor shadow → falls fast with rumble → slams with dust + shake → remains as ground hazard. Colliding while falling or with the landed spike costs a heart. Jumping over the landed spike clears it.

- [ ] **Step 7: Commit** (after removing the temp spawn line)

```bash
git add index.html
git commit -m "feat: stalactite obstacle (shadow warning, drop trigger, ground hazard)"
```

---

### Task 7: Crab Claws obstacle

**Files:**
- Modify: `index.html`

**Interfaces:**
- Produces: `makeCrabClaw(seg)` `kind='crabClaw'`, timer cycle open(1.4s)→snap(0.2s)→hold(0.4s).

- [ ] **Step 1: Factory** —

```js
// ─── CRAB CLAWS (two arms snap shut across a lane on a rhythm — time your pass) ───
function makeCrabClaw(seg){
  const lane=Math.floor(Math.random()*3);
  const o=new THREE.Group();
  const shellMat=new THREE.MeshStandardMaterial({color:0xd86030,roughness:0.5,emissive:0x501008,emissiveIntensity:0.3});
  const arms=[];
  for(let s=-1;s<=1;s+=2){
    const arm=new THREE.Group();
    const seg1=new THREE.Mesh(new THREE.CylinderGeometry(0.14,0.18,1.1,8),shellMat);
    seg1.rotation.z=Math.PI/2; seg1.position.x=s*0.55; arm.add(seg1);
    // Pincer: two curved cones
    const pin1=new THREE.Mesh(new THREE.ConeGeometry(0.16,0.7,6),shellMat);
    pin1.position.set(s*1.2,0.18,0); pin1.rotation.z=-s*1.9; arm.add(pin1);
    const pin2=new THREE.Mesh(new THREE.ConeGeometry(0.12,0.5,6),shellMat);
    pin2.position.set(s*1.18,-0.15,0); pin2.rotation.z=-s*1.25; arm.add(pin2);
    arm.position.set(s*1.55,1.1,0);
    o.add(arm); arms.push(arm);
  }
  o.position.set(C.lanes[lane],0,findSpawnZ(seg));
  o.traverse(ch=>{ if(ch.isMesh) ch.castShadow=true; });
  o.userData={kind:'crabClaw',lane,warned:false,cycle:Math.random()*2,arms};
  scene.add(o); obs.push(o);
}
```

- [ ] **Step 2: Animation branch** —

```js
      } else if(o.userData.kind==='crabClaw'){
        o.userData.cycle=(o.userData.cycle+dt)%2.0;
        const cy=o.userData.cycle;
        // 0-1.4 open, 1.4-1.6 snapping shut, 1.6-2.0 held shut
        let openF; // 1 = fully open, 0 = shut
        if(cy<1.4) openF=1;
        else if(cy<1.6){ openF=1-(cy-1.4)/0.2; if(!o.userData.snapped){o.userData.snapped=true; if(o.position.z>-25)SFX.crabSnap();} }
        else openF=0;
        if(cy<1.4) o.userData.snapped=false;
        o.userData.arms[0].position.x=-1.55+(1-openF)*1.05;
        o.userData.arms[1].position.x= 1.55-(1-openF)*1.05;
```

- [ ] **Step 3: Collision** — crabClaw hurts only while mostly shut. Extend the collision ternary chain (matching Task 6 pattern) with:

```js
          : o.userData.kind==='crabClaw'
          ? (o.userData.cycle>=1.45 && Math.abs(player.position.z-o.position.z)<0.7 && Math.abs(player.position.x-o.position.x)<1.3 && player.position.y-player.userData.baseY<1.3)
```

(Jumping over the claws also works since they sit at y≈1.1.)

- [ ] **Step 4: Temp spawn for testing** — same pattern as Task 6 (temp line in makeObs, removed before commit).

- [ ] **Step 5: SFX** — `crabSnap(){ osc('square', 320, 90, 0.09, 0.12); noise(0.06, 0.12, 1400); },`

- [ ] **Step 6: Verify** — claws visibly cycle open/shut with snap sound; passing during open phase is safe; getting caught in the snap costs a heart; jumping over always works.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: crab claws obstacle (rhythmic snap timing)"
```

---

### Task 8: Ink Cloud obstacle

**Files:**
- Modify: `index.html`

**Interfaces:**
- Produces: `makeInkCloud(seg)` `kind='inkCloud'`, `S.inkedT` blackout state, `#ink-overlay` DOM element.

- [ ] **Step 1: Ink overlay** — CSS:

```css
    #ink-overlay { position:absolute; top:0; left:0; width:100%; height:100%; pointer-events:none; z-index:57; opacity:0; transition:opacity 0.25s;
      background:radial-gradient(ellipse at 50% 45%, rgba(5,8,12,0.55) 0%, rgba(2,4,8,0.97) 62%, #000 100%); }
```

HTML after `#screen-flash` div: `<div id="ink-overlay"></div>`
$ ref: `$ink=document.getElementById('ink-overlay'),`
State: `inkedT:0,` in `resetState()`.

- [ ] **Step 2: Factory** —

```js
// ─── INK CLOUD (blinds you for 1.5s on contact — dodge it) ───
function makeInkCloud(seg){
  const lane=Math.floor(Math.random()*3);
  const o=new THREE.Group();
  for(let i=0;i<7;i++){
    const s=0.35+Math.random()*0.4;
    const blob=new THREE.Mesh(new THREE.SphereGeometry(s,10,8),
      new THREE.MeshBasicMaterial({color:0x0a0a14,transparent:true,opacity:0.55+Math.random()*0.25,depthWrite:false}));
    blob.position.set((Math.random()-0.5)*1.2,(Math.random()-0.5)*1.0,(Math.random()-0.5)*1.0);
    blob.userData.puffPhase=Math.random()*Math.PI*2;
    o.add(blob);
  }
  o.position.set(C.lanes[lane],1.9,findSpawnZ(seg));
  o.userData={kind:'inkCloud',lane,warned:false};
  scene.add(o); obs.push(o);
}
```

- [ ] **Step 3: Animation branch** —

```js
      } else if(o.userData.kind==='inkCloud'){
        o.children.forEach(b=>{const pf=1+Math.sin(t*1.8+b.userData.puffPhase)*0.15;b.scale.setScalar(pf);});
        o.rotation.y+=0.3*dt;
```

- [ ] **Step 4: Collision** — ink does NOT damage; it blinds. It needs its own branch BEFORE the damage collision chain (inside the obstacle loop, after the animation branches):

```js
      // Ink cloud: blind, no damage, cloud dissipates
      if(o.userData.kind==='inkCloud' && dist3(player.position,o.position)<1.4){
        S.inkedT=1.5;
        SFX.inkHit();
        PS.burst(o.position.clone(),[0x0a0a14,0x202030,0x101020],16,4,2);
        scene.remove(o); disposeObj(o); obs.splice(i,1); continue;
      }
```

And EXCLUDE inkCloud from the damage chain by adding to its condition: the chain ternary must not run for inkCloud (it was removed above, so no change needed — but ensure barrier branch is not affected).

- [ ] **Step 5: Blackout tick** — in the loop near the invincibility block:

```js
    // Ink blindness
    if(S.inkedT>0){
      S.inkedT-=dt;
      $ink.style.opacity=Math.min(1,S.inkedT/0.4).toString();
      if(S.inkedT<=0)$ink.style.opacity='0';
    }
```

Also add `$ink.style.opacity='0';` in `startGame()` and `endGame()` housekeeping.

- [ ] **Step 6: SFX** — `inkHit(){ noise(0.35, 0.25, 150); osc('sine', 220, 70, 0.3, 0.06); },`

- [ ] **Step 7: Verify** — swim into the black blob cloud: screen fades to near-black leaving only a small visibility circle at center, fades back over 1.5s, no heart lost. Restart clears overlay.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat: ink cloud obstacle (1.5s blindness, no damage)"
```

---

### Task 9: Torpedo Fish obstacle

**Files:**
- Modify: `index.html`

**Interfaces:**
- Produces: `makeTorpedo()` (NOT segment-based — spawned by a timer in the loop), `S.torpedoTimer`.

Torpedoes cross the track laterally, so they do not scroll with segments. They spawn from a loop timer at level 3+ and live in `obs[]` with their own motion.

- [ ] **Step 1: State** — `resetState()`: add `torpedoTimer:8,`.

- [ ] **Step 2: Factory** —

```js
// ─── TORPEDO FISH (shoots across all lanes from the side — jump or gap it) ───
function makeTorpedo(){
  const side=Math.random()<0.5?-1:1;
  const o=new THREE.Group();
  const tMat=new THREE.MeshStandardMaterial({color:0x506878,roughness:0.3,metalness:0.6});
  const bodyG=new THREE.SphereGeometry(0.32,12,10); bodyG.scale(2.8,0.9,0.9);
  o.add(new THREE.Mesh(bodyG,tMat));
  const nose=new THREE.Mesh(new THREE.SphereGeometry(0.2,10,8),
    new THREE.MeshStandardMaterial({color:0xff3020,roughness:0.3,emissive:0xc01000,emissiveIntensity:0.9}));
  nose.position.x=-side*0.85; o.add(nose);
  const finMat=new THREE.MeshStandardMaterial({color:0x384858,roughness:0.5});
  for(let a=0;a<3;a++){
    const fin=new THREE.Mesh(new THREE.ConeGeometry(0.12,0.32,4),finMat);
    fin.position.x=side*0.75; fin.rotation.z=side*Math.PI/2; fin.rotation.x=a*Math.PI*2/3; o.add(fin);
  }
  const zAt=-8-Math.random()*6;
  o.position.set(side*16,1.15,zAt);
  o.userData={kind:'torpedo',lane:1,warned:false,vx:-side*22,streakT:0};
  // Red warning streak at the entry edge
  triggerLW(side<0?0:2);
  SFX.torpedoWarn();
  scene.add(o); obs.push(o);
}
```

- [ ] **Step 3: Spawn timer** — in the loop, inside `if(S.on&&!S.paused){`, after the curve retarget block:

```js
    // Torpedo volleys from level 3 (TREASURE PALACE) onward
    if(S.level>=3){
      S.torpedoTimer-=dt;
      if(S.torpedoTimer<=0){
        S.torpedoTimer=(S.level>=4?3.5:6)+Math.random()*3;
        makeTorpedo();
        if(S.level>=4&&Math.random()<0.5) setTimeout(()=>{if(S.on&&!S.paused)makeTorpedo();},600);
      }
    }
```

- [ ] **Step 4: Motion + bubbles branch** — in the obstacle loop. IMPORTANT: torpedoes must NOT get `o.position.z+=spd*dt` scroll. At the TOP of the obstacle loop change `const o=obs[i]; o.position.z+=spd*dt;` to:

```js
      const o=obs[i];
      if(o.userData.kind!=='torpedo') o.position.z+=spd*dt;
```

Animation branch:

```js
      } else if(o.userData.kind==='torpedo'){
        o.position.x+=o.userData.vx*dt;
        o.position.z+=spd*dt*0.25; // slight drift so it does not feel glued
        o.userData.streakT-=dt;
        if(o.userData.streakT<=0){
          o.userData.streakT=0.05;
          PS.burst(new THREE.Vector3(o.position.x-Math.sign(o.userData.vx)*-1.0,o.position.y,o.position.z),[0xffffff,0xcceeff],1,0.5,0.5);
        }
        if(Math.abs(o.position.x)>18){scene.remove(o);disposeObj(o);obs.splice(i,1);continue;}
```

- [ ] **Step 5: Collision** — extend the damage ternary chain:

```js
          : o.userData.kind==='torpedo'
          ? (Math.abs(player.position.z-o.position.z)<0.9 && Math.abs(player.position.x-o.position.x)<1.1 && Math.abs(player.position.y-o.position.y)<0.9)
```

(Jumping clears it: torpedo rides at y=1.15, a jump peaks ~4.5.)

- [ ] **Step 6: SFX** — `torpedoWarn(){ osc('square', 1200, 900, 0.1, 0.07); osc('square', 1200, 900, 0.1, 0.07, 0.15); },`

- [ ] **Step 7: Verify** — Shift+K to TREASURE PALACE. Torpedoes streak across with edge-lane warning flash + two warning beeps; jump or lane-gap them; hit costs a heart; level 5 fires doubles. Check none linger in `obs` (console: `obs.length` stays bounded).

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat: torpedo fish cross-lane projectile (level 3+)"
```

---

### Task 10: Electric Eel obstacle + final makeObs weights

**Files:**
- Modify: `index.html`

**Interfaces:**
- Produces: `makeEelZap(seg)` `kind='eelZap'` (charge → zap leaves a 2s lane hazard), FINAL `makeObs` dispatch integrating stalactite, crabClaw, inkCloud, eelZap.

- [ ] **Step 1: Factory** —

```js
// ─── ELECTRIC EEL (coils on a lane, charges, then electrifies the lane 2s) ───
function makeEelZap(seg){
  const lane=Math.floor(Math.random()*3);
  const o=new THREE.Group();
  const eMat=new THREE.MeshStandardMaterial({color:0x30c8c0,roughness:0.4,emissive:0x087868,emissiveIntensity:0.5});
  const segsArr=[];
  for(let i=0;i<8;i++){
    const r=0.22-i*0.014;
    const sg=new THREE.Mesh(new THREE.SphereGeometry(r,10,8),eMat);
    sg.position.set(Math.sin(i*0.9)*0.35,1.2+i*0.16,Math.cos(i*0.9)*0.2);
    o.add(sg); segsArr.push(sg);
  }
  const head=new THREE.Mesh(new THREE.SphereGeometry(0.26,12,10),eMat);
  head.scale.set(1.1,0.85,1.3); head.position.set(0,1.1,0.15); o.add(head);
  for(let s=-1;s<=1;s+=2){
    const eye=new THREE.Mesh(new THREE.SphereGeometry(0.06,6,5),M.eyeH); eye.position.set(s*0.14,1.2,0.36); o.add(eye);
  }
  // Zap field: vertical glowing plane across the lane (hidden until zap)
  const field=new THREE.Mesh(new THREE.PlaneGeometry(2.2,2.6),
    new THREE.MeshBasicMaterial({color:0x60f0ff,transparent:true,opacity:0,side:THREE.DoubleSide,depthWrite:false,blending:THREE.AdditiveBlending}));
  field.position.y=1.6; o.add(field);
  o.position.set(C.lanes[lane],0,findSpawnZ(seg));
  o.traverse(ch=>{ if(ch.isMesh&&ch.material&&!ch.material.transparent) ch.castShadow=true; });
  o.userData={kind:'eelZap',lane,warned:false,state:'idle',chargeT:0,zapT:0,eMat,field,segsArr};
  scene.add(o); obs.push(o);
}
```

- [ ] **Step 2: Animation branch** —

```js
      } else if(o.userData.kind==='eelZap'){
        const ez=o.userData;
        ez.segsArr.forEach((sg2,si)=>{sg2.position.x=Math.sin(si*0.9+t*3)*0.35;});
        if(ez.state==='idle'&&o.position.z>-22){ ez.state='charging'; ez.chargeT=1.5; SFX.eelCharge(); }
        if(ez.state==='charging'){
          ez.chargeT-=dt;
          ez.eMat.emissiveIntensity=0.5+ (1.5-ez.chargeT)*2.2 + Math.sin(t*30)*0.4;
          if(ez.chargeT<=0){ ez.state='zapping'; ez.zapT=2.0; SFX.eelZap(); }
        } else if(ez.state==='zapping'){
          ez.zapT-=dt;
          ez.field.material.opacity=0.35+Math.sin(t*40)*0.2;
          ez.eMat.emissiveIntensity=2.5+Math.sin(t*25)*1.2;
          if(Math.random()<0.3) PS.burst(new THREE.Vector3(o.position.x+(Math.random()-0.5)*1.6,1+Math.random()*1.6,o.position.z),[0x60f0ff,0xffffff,0xaaffff],1,1,1);
          if(ez.zapT<=0){ ez.state='done'; ez.field.material.opacity=0; ez.eMat.emissiveIntensity=0.5; }
        }
```

- [ ] **Step 3: Collision** — extend the damage ternary chain:

```js
          : o.userData.kind==='eelZap'
          ? (o.userData.state==='zapping' && Math.abs(player.position.z-o.position.z)<0.8 && Math.abs(player.position.x-o.position.x)<1.2)
```

(While idle/charging/done the eel is harmless — you swim through its lane freely; only the 2s zap field kills. Jumping does NOT clear it — field is 2.6 tall — you must lane-dodge.)

- [ ] **Step 4: FINAL makeObs dispatch** — replace the whole weight block at the top of `makeObs` (~line 1248):

```js
function makeObs(seg){
  const lvl=S?S.level:0, r=Math.random(); let t;
  // t: 0 urchin 1 rock 2 jelly 3 mine 4 barrier 5 sweeper 6 stalactite 7 crabClaw 8 inkCloud 9 eelZap
  if(lvl>=4){      if(r<0.10)t=0;else if(r<0.18)t=1;else if(r<0.28)t=2;else if(r<0.40)t=3;else if(r<0.52)t=4;else if(r<0.62)t=5;else if(r<0.74)t=6;else if(r<0.84)t=7;else if(r<0.92)t=8;else t=9; }
  else if(lvl>=3){ if(r<0.12)t=0;else if(r<0.22)t=1;else if(r<0.34)t=2;else if(r<0.46)t=3;else if(r<0.58)t=4;else if(r<0.68)t=5;else if(r<0.80)t=6;else if(r<0.88)t=7;else if(r<0.94)t=8;else t=9; }
  else if(lvl>=2){ if(r<0.14)t=0;else if(r<0.26)t=1;else if(r<0.40)t=2;else if(r<0.52)t=3;else if(r<0.66)t=4;else if(r<0.76)t=5;else if(r<0.86)t=6;else if(r<0.94)t=8;else t=9; }
  else if(lvl>=1){ if(r<0.20)t=0;else if(r<0.38)t=1;else if(r<0.54)t=2;else if(r<0.68)t=3;else if(r<0.80)t=5;else if(r<0.90)t=7;else t=9; }
  else{            if(r<0.33)t=0;else if(r<0.66)t=1;else t=2; }
  if(t===9){ makeEelZap(seg); return; }
  if(t===8){ makeInkCloud(seg); return; }
  if(t===7){ makeCrabClaw(seg); return; }
  if(t===6){ makeStalactite(seg); return; }
  if(t===5){ makeSweeper(seg); return; }
  if(t===4){ makeBarrier(seg); return; }
  if(t===3){ makeMine(seg); return; }
```

(Remove any temp spawn lines left from Tasks 6-8.)

- [ ] **Step 5: SFX** —

```js
    eelCharge(){ osc('sawtooth', 80, 400, 1.4, 0.05); },
    eelZap(){ noise(0.4, 0.3, 2400); osc('square', 1800, 300, 0.35, 0.09); },
```

- [ ] **Step 6: Verify** — play through levels 0-4 with Shift+K. Level 0: only classic three. Level 1: crabs + eels appear. Level 2: +stalactites, ink, barriers. Level 3+: everything + torpedoes. Eel: teal coil glows brighter over 1.5s then the lane wall of light kills on touch for 2s, then goes dark and harmless.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: electric eel lane-zap + final per-level obstacle weights"
```

---

### Task 11: Kraken — build, silhouette, pursuit

**Files:**
- Modify: `index.html`

**Interfaces:**
- Produces: `kraken` scene object (module-level), `updateKraken(dt,t)` called from the loop, `S.krakenPhase` ('none'|'silhouette'|'pursuing'|'attacking'|'fullAssault'), `S.krakenZ`.

GEOMETRY NOTE (deviation from spec): the spec placed the pursuer at negative z (ahead of the player) — that reads as running TOWARD the monster. Correct pursuer staging for this camera (cam z=11 looking at -z): the Kraken lives BETWEEN camera and player at positive z. Safe = z 16 (hidden behind camera). Danger = z 5-7 (tentacles and eyes loom into frame around the player). Catch at z < 4.5. The silhouette phase keeps the spec look: a huge dark mass far ahead in the fog that vanishes, then ROARS in behind you.

- [ ] **Step 1: State** — `resetState()`: add `krakenPhase:'none', krakenZ:16, krakenAtkT:6, krakenInkT:5, krakenWaveT:9, survT:0,`.

- [ ] **Step 2: Build** — after the temple code (~line 970):

```js
// ─── THE KRAKEN ───
function makeKraken(){
  const g=new THREE.Group();
  const kMat=new THREE.MeshPhysicalMaterial({color:0x1a0a2e,roughness:0.55,metalness:0.1,emissive:0x40106a,emissiveIntensity:0.4,clearcoat:0.3});
  const kMatLight=new THREE.MeshPhysicalMaterial({color:0x2a1445,roughness:0.5,emissive:0x6020a0,emissiveIntensity:0.5});
  // Body
  const bodyG=new THREE.SphereGeometry(4,20,16); bodyG.scale(1,0.75,1.2);
  const body=new THREE.Mesh(bodyG,kMat); g.add(body);
  // Mantle
  const mantle=new THREE.Mesh(new THREE.ConeGeometry(3.2,5.5,10),kMat);
  mantle.position.y=4.2; mantle.rotation.x=-0.15; g.add(mantle);
  // Eyes: huge, yellow slits
  const eyes=[];
  for(let s=-1;s<=1;s+=2){
    const ew=new THREE.Mesh(new THREE.SphereGeometry(1.0,14,12),new THREE.MeshStandardMaterial({color:0xffd820,roughness:0.15,emissive:0xd0a000,emissiveIntensity:1.4}));
    ew.position.set(s*1.9,1.1,-3.1);
    const pu=new THREE.Mesh(new THREE.SphereGeometry(0.55,10,8),new THREE.MeshBasicMaterial({color:0x000000}));
    pu.scale.set(0.35,1,0.35); pu.position.set(s*1.9,1.1,-3.85);
    g.add(ew,pu); eyes.push({ew,pu});
  }
  // Bioluminescent vein rings on the mantle
  for(let i=0;i<4;i++){
    const ring=new THREE.Mesh(new THREE.TorusGeometry(2.6-i*0.5,0.06,6,20),new THREE.MeshBasicMaterial({color:0xa050ff,transparent:true,opacity:0.6}));
    ring.rotation.x=Math.PI/2; ring.position.y=2.6+i*1.1; g.add(ring);
  }
  // 8 ambient tentacles
  const tents=[];
  for(let i2=0;i2<8;i2++){
    const tg=new THREE.Group();
    const a=(i2/8)*Math.PI*2;
    let r0=0.55;
    for(let j=0;j<6;j++){
      const jr=r0*(1-j*0.14);
      const joint=new THREE.Mesh(new THREE.SphereGeometry(jr,8,6),j%2?kMat:kMatLight);
      joint.position.set(0,-j*0.9,-j*0.35);
      tg.add(joint);
    }
    tg.position.set(Math.cos(a)*3.2,-1.8,Math.sin(a)*2.2);
    tg.userData={phase:i2*0.8,baseA:a};
    g.add(tg); tents.push(tg);
  }
  // 2 long attack tentacles (bigger, articulated later)
  const atkTents=[];
  for(let s=-1;s<=1;s+=2){
    const tg=new THREE.Group();
    for(let j=0;j<9;j++){
      const jr=0.7*(1-j*0.09);
      const joint=new THREE.Mesh(new THREE.SphereGeometry(jr,8,6),j%2?kMat:kMatLight);
      joint.position.set(0,-j*1.0,0);
      // Suckers
      if(j>2){const su=new THREE.Mesh(new THREE.SphereGeometry(jr*0.3,6,5),new THREE.MeshBasicMaterial({color:0xc080ff,transparent:true,opacity:0.7}));su.position.set(0,-j*1.0,jr*0.8);tg.add(su);}
      tg.add(joint);
    }
    tg.position.set(s*4.5,0,-1);
    tg.userData={side:s,attacking:false,atkLane:1,atkT:0};
    g.add(tg); atkTents.push(tg);
  }
  g.userData={eyes,tents,atkTents,roarT:0};
  g.visible=false;
  return g;
}
const kraken=makeKraken();
scene.add(kraken);
// Silhouette double: giant dark ghost far ahead in the fog for the teaser phase
const krakenSil=(()=>{
  const m=new THREE.Mesh(new THREE.SphereGeometry(9,12,10),
    new THREE.MeshBasicMaterial({color:0x02030a,transparent:true,opacity:0,depthWrite:false}));
  m.scale.set(1,1.4,1); m.position.set(6,10,-190); m.visible=false;
  scene.add(m); return m;
})();
```

- [ ] **Step 3: updateKraken** — new function placed right before the main loop:

```js
function updateKraken(dt,t){
  // Phase machine driven by distance
  const d=S.dist;
  let ph='none';
  if(d>=1800) ph='fullAssault';
  else if(d>=1200) ph='attacking';
  else if(d>=1000) ph='pursuing';
  else if(d>=750) ph='silhouette';
  if(ph!==S.krakenPhase){
    const was=S.krakenPhase; S.krakenPhase=ph;
    if(ph==='silhouette'){ krakenSil.visible=true; }
    if(ph==='pursuing'&&(was==='silhouette'||was==='none')){
      krakenSil.visible=false;
      kraken.visible=true; S.krakenZ=16;
      SFX.krakenRoar(); flash('#40106a',300); S.shakeT=0.6; S.shakeAmp=0.14;
      announceLv('IT FOUND YOU!');
    }
    if(ph==='fullAssault'){ SFX.krakenRoar(); flash('#ff2020',200); }
  }
  if(S.krakenPhase==='none') return;
  if(S.krakenPhase==='silhouette'){
    // Looming dark mass drifting in the fog ahead
    const f=Math.min(1,(d-750)/250);
    krakenSil.material.opacity=0.25+f*0.4;
    krakenSil.position.x=6*Math.sin(t*0.13);
    krakenSil.position.y=9+Math.sin(t*0.3)*1.5;
    return;
  }
  // ── Pursuit (all phases >= pursuing) ──
  // Kraken hangs between camera and player; closes in when you are slow.
  const spdFrac=(S.speed-C.baseSpeed)/(C.maxSpeed-C.baseSpeed);
  const tgtZ=S.boost||S.dashT>0?17:(5.5+spdFrac*10.5);
  S.krakenZ+=(tgtZ-S.krakenZ)*0.014;
  kraken.position.set(player.position.x*0.5, 2.2+Math.sin(t*0.9)*0.5, S.krakenZ);
  kraken.rotation.y=Math.PI+Math.sin(t*0.5)*0.08; // face down-track (eyes at -z face player)
  kraken.rotation.z=Math.sin(t*0.7)*0.05;
  // Ambient tentacle writhe
  kraken.userData.tents.forEach(tg=>{
    tg.rotation.x=0.5+Math.sin(t*1.4+tg.userData.phase)*0.35;
    tg.rotation.z=Math.sin(t*1.1+tg.userData.phase)*0.3;
  });
  // Caught?
  if(S.krakenZ<4.5&&!S.invincible&&S.dashT<=0){
    SFX.krakenRoar();
    S.krakenZ=13; // recoil after the bite so you get a chance
    if(S.decoy){ S.decoy=false; decoyFish.visible=false; S.invincible=true; S.invincibleT=1.0; flash('#40e0ff',150); }
    else if(takeDamage()) return;
  }
  // Survival bonus: +50/s while pursued
  S.survT+=dt;
  if(S.survT>=1){ S.survT-=1; S.score+=50; }
}
```

- [ ] **Step 4: Call it** — in the loop inside `if(S.on&&!S.paused){`, right after the level-detection block: `updateKraken(dt,t);`

- [ ] **Step 5: Hide on restart** — in `startGame()`: `kraken.visible=false; krakenSil.visible=false;`

- [ ] **Step 6: SFX** —

```js
    krakenRoar(){ noise(1.2, 0.4, 90); osc('sawtooth', 55, 28, 1.1, 0.16); osc('sawtooth', 82, 40, 0.9, 0.1, 0.1); },
```

- [ ] **Step 7: Verify** — Shift+K to dist 750+: dark looming mass ahead in fog. At 1000: white-purple flash, roar, "IT FOUND YOU!", the Kraken (yellow eyes, purple rings, writhing tentacles) looms into frame from behind whenever speed drops; boosting pushes it out of frame. Getting caught: roar + heart lost + kraken recoils. Score ticks +50/s while pursued.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat: the Kraken - build, silhouette teaser, pursuit with catch"
```

---

### Task 12: Kraken tentacle lane attacks

**Files:**
- Modify: `index.html` — extends `updateKraken`

**Interfaces:**
- Consumes: `kraken.userData.atkTents`, `triggerLW(lane)`, `S.krakenAtkT`.
- Produces: tentacle slam attack cycle at phase 'attacking'+.

- [ ] **Step 1: Attack logic** — append inside `updateKraken`, after the survival bonus:

```js
  if(S.krakenPhase!=='attacking'&&S.krakenPhase!=='fullAssault') return;
  // ── Tentacle lane slams ──
  S.krakenAtkT-=dt;
  if(S.krakenAtkT<=0){
    S.krakenAtkT=(S.krakenPhase==='fullAssault'?3.5:5.5)+Math.random()*2.5;
    const lane=Math.floor(Math.random()*3);
    const tent=kraken.userData.atkTents[lane<=1?0:1];
    if(!tent.userData.attacking){
      tent.userData.attacking=true; tent.userData.atkLane=lane; tent.userData.atkT=0;
      triggerLW(lane); SFX.tentacleWarn();
      setTimeout(()=>{if(S.on)triggerLW(lane);},400);
      setTimeout(()=>{if(S.on)triggerLW(lane);},800);
    }
  }
  kraken.userData.atkTents.forEach(tent=>{
    const tu=tent.userData;
    if(!tu.attacking){
      // Rest pose: curled at the kraken sides
      tent.rotation.x=0.8+Math.sin(t*1.2+tu.side)*0.2;
      tent.position.z=-1;
      return;
    }
    tu.atkT+=dt;
    const laneX=C.lanes[tu.atkLane];
    // NOTE: kraken.rotation.y = PI mirrors child local X and Z in world space.
    // Local x must be NEGATED and local z POSITIVE to land at world laneX / world z -2.
    const kx=-(laneX-kraken.position.x), kz=S.krakenZ+2;
    if(tu.atkT<1.2){
      // Wind-up: tentacle rises above the frame
      const f=tu.atkT/1.2;
      tent.position.set(kx, 6+f*6, 4);
      tent.rotation.x=0.2;
    } else if(tu.atkT<1.45){
      // SLAM: drops through the lane at the player z plane
      const f=(tu.atkT-1.2)/0.25;
      tent.position.set(kx, 12-f*13, kz); // world z ~= -2
      if(f>0.5&&!tu.hitDone){
        tu.hitDone=true;
        S.shakeT=0.3; S.shakeAmp=0.1; SFX.tentacleSlam();
        PS.burst(new THREE.Vector3(laneX,0.6,-2),[0x40106a,0x6020a0,0xc8b898],16,5,4);
        // Collision: player on that lane at ground height
        const playerLane=C.lanes.reduce((best,lx,li)=>Math.abs(player.position.x-lx)<Math.abs(player.position.x-C.lanes[best])?li:best,0);
        if(playerLane===tu.atkLane&&player.position.y-player.userData.baseY<2.2&&!S.invincible&&S.dashT<=0){
          if(S.decoy){ S.decoy=false; decoyFish.visible=false; S.invincible=true; S.invincibleT=1.0; flash('#40e0ff',150); SFX.hit(); }
          else if(S.shieldActive){ S.shieldActive=false; player.userData.aura.visible=false; $shb.classList.remove('show'); S.invincible=true; S.invincibleT=0.8; flash('#00ddff',150); SFX.hit(); }
          else takeDamage();
        }
      }
    } else if(tu.atkT<2.1){
      // Retract
      const f=(tu.atkT-1.45)/0.65;
      tent.position.set(kx*(1-f), -1+f*1, 4-f*3);
    } else {
      tu.attacking=false; tu.hitDone=false;
    }
  });
```

- [ ] **Step 2: SFX** —

```js
    tentacleWarn(){ osc('sine', 180, 120, 0.35, 0.1); osc('sine', 180, 120, 0.35, 0.1, 0.45); },
    tentacleSlam(){ noise(0.4, 0.35, 120); osc('sine', 70, 30, 0.35, 0.14); },
```

- [ ] **Step 3: Verify** — Shift+K to dist 1200+. Every ~6s: a lane flashes red 3 times with warning tones → a huge sucker-studded tentacle crashes down through that lane with dust + shake → retracts. Standing in the lane (grounded or low jump) costs a heart; being on another lane or high in a charge-jump is safe. Shield/decoy absorb it.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: kraken tentacle lane-slam attacks (level 4+)"
```

---

### Task 13: Kraken full assault — ink blasts + shockwaves

**Files:**
- Modify: `index.html` — extends `updateKraken`, new projectile arrays

**Interfaces:**
- Produces: `krakenProj[]` module array (ink orbs + shockwave rings), full-assault attack cycle, cleanup in `initWorld()`.

- [ ] **Step 1: Array** — add `krakenProj` to the pools line (~line 700): `const segs=[], obs=[], ..., krakenProj=[];`
In `initWorld()` add: `krakenProj.forEach(o=>{disposeObj(o);scene.remove(o);}); krakenProj.length=0;`

- [ ] **Step 2: Assault logic** — append inside `updateKraken` after the tentacle block:

```js
  if(S.krakenPhase!=='fullAssault') return;
  // ── Ink blast ──
  S.krakenInkT-=dt;
  if(S.krakenInkT<=0){
    S.krakenInkT=4+Math.random()*3;
    const orb=new THREE.Mesh(new THREE.SphereGeometry(0.55,12,10),
      new THREE.MeshBasicMaterial({color:0x0a0a14,transparent:true,opacity:0.85}));
    orb.position.set(kraken.position.x,3,S.krakenZ-2);
    // Aim at the player's current lane, flying forward (negative z)
    orb.userData={pKind:'ink',vx:(player.position.x-orb.position.x)*0.6,vz:-(S.speed*1.5+14)};
    scene.add(orb); krakenProj.push(orb);
    SFX.inkShot();
  }
  // ── Shockwave ring ──
  S.krakenWaveT-=dt;
  if(S.krakenWaveT<=0){
    S.krakenWaveT=8+Math.random()*3;
    const ring=new THREE.Mesh(new THREE.TorusGeometry(1.2,0.22,8,24),
      new THREE.MeshBasicMaterial({color:0xa050ff,transparent:true,opacity:0.85}));
    ring.rotation.x=Math.PI/2;
    ring.position.set(0,0.55,S.krakenZ-3);
    ring.userData={pKind:'wave',vz:-(S.speed+10),grow:4.5};
    scene.add(ring); krakenProj.push(ring);
    SFX.shockwave();
    announceLv('JUMP!');
  }
  // ── Projectile update ──
  for(let i=krakenProj.length-1;i>=0;i--){
    const pr=krakenProj[i];
    pr.position.z+=pr.userData.vz*dt;
    if(pr.userData.pKind==='ink'){
      pr.position.x+=pr.userData.vx*dt;
      pr.scale.setScalar(1+Math.sin(t*10)*0.12);
      if(Math.abs(pr.position.z-player.position.z)<1.0&&Math.abs(pr.position.x-player.position.x)<1.0&&Math.abs(pr.position.y-player.position.y)<1.4){
        S.inkedT=1.5; SFX.inkHit();
        PS.burst(pr.position.clone(),[0x0a0a14,0x202030],14,4,2);
        scene.remove(pr); disposeObj(pr); krakenProj.splice(i,1); continue;
      }
    } else { // wave
      pr.scale.x+=pr.userData.grow*dt; pr.scale.y+=pr.userData.grow*dt;
      pr.material.opacity=Math.max(0.2,pr.material.opacity-dt*0.12);
      if(Math.abs(pr.position.z-player.position.z)<0.7&&player.position.y-player.userData.baseY<0.9&&!S.invincible&&S.dashT<=0){
        if(S.decoy){ S.decoy=false; decoyFish.visible=false; S.invincible=true; S.invincibleT=1.0; flash('#40e0ff',150); SFX.hit(); }
        else if(S.shieldActive){ S.shieldActive=false; player.userData.aura.visible=false; $shb.classList.remove('show'); S.invincible=true; S.invincibleT=0.8; flash('#00ddff',150); SFX.hit(); }
        else if(takeDamage()){ return; }
        scene.remove(pr); disposeObj(pr); krakenProj.splice(i,1); continue;
      }
    }
    if(pr.position.z<-60){ scene.remove(pr); disposeObj(pr); krakenProj.splice(i,1); }
  }
```

- [ ] **Step 3: SFX** —

```js
    inkShot(){ osc('sine', 300, 90, 0.3, 0.1); noise(0.15, 0.12, 400); },
    shockwave(){ osc('sawtooth', 45, 130, 0.7, 0.13); noise(0.5, 0.15, 200); },
```

- [ ] **Step 4: Verify** — Shift+K to dist 1800 (BOSS ESCAPE). Expected: red flash + roar entering the level. Ink orbs arc toward you (dodge laterally, or eat 1.5s blindness). Purple shockwave rings sweep forward with "JUMP!" callout — jumping clears, standing costs a heart. Shield/decoy absorb both. Full run 0→death with no console errors, obs/cols/krakenProj all bounded, restart clean.

- [ ] **Step 5: Full regression run** — one complete game from CALM REEFS to death in BOSS ESCAPE without Shift+K if patience allows (or with sparing use). Confirm: fps stays smooth, all old features intact, high-score flow works.

- [ ] **Step 6: Commit + push**

```bash
git add index.html
git commit -m "feat: kraken full assault - ink blasts and shockwave rings (final level)"
git push
```

---

## Deferred / explicitly not in this plan

- Double Points power-up: already exists in index.html as the `x2` pickup. No work needed.
- Spec's "collapsing walls / debris" for level 4: covered by stalactite waves (higher stalactite weight at lvl>=3 in the makeObs table).
- CLAUDE.md refresh to document index.html as the active file — do after implementation lands.

