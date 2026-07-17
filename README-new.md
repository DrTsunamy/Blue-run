# 🌊 Blue Run – Underwater Odyssey

A fast, juicy 3D endless-runner that lives in **a single HTML file**. Swim a coral canyon,
dodge urchins, mines, jellyfish and moray eels, chain pearls into combos, and chase your
best score across five biomes — from sunlit reefs to the boss escape.

**Play it:** just open `index.html` in any modern browser (desktop or mobile).
No build step, no install.

## Controls

| Action | Keyboard | Touch |
|---|---|---|
| Change lane | ← → or A / D | Swipe left / right |
| Jump (hold to charge) | Space | Tap & hold |
| Dive-slam | ↓ or S (mid-air) | Swipe down |
| Pause | P or Esc | ⏸ button |

A fully-charged jump triggers the **Rainbow Boost**. The dive-slam sends out a shockwave
that pulls in nearby pearls.

## Features

- **5 biomes** with smooth colour-graded transitions, animated caustics, god rays and a
  procedural sunken temple that drifts by
- **Power-ups:** shield 🛡, magnet 🧲, and 2× score ✨
- **Skill rewards:** near-miss bonuses ("CLOSE!"), combo multiplier, pearl arcs over obstacles
- **One revive per run**, slow-motion last-life moments, distance milestones, live NEW BEST chase
- **Procedural audio** — all SFX and the adaptive music loop are synthesized with the
  Web Audio API; the tempo rises with each biome. No audio files.
- **Local top-5 high scores** (browser localStorage only — nothing leaves your device)

## Tech & security notes

- three.js r128 from CDN, **pinned with Subresource Integrity (SRI)** hashes and a
  fallback CDN; a strict `Content-Security-Policy` meta tag limits scripts to those hosts
- No analytics, no cookies, no network calls besides the CDN libraries and Google Fonts
- Unhandled startup errors are rendered with `textContent` (no HTML injection)
- WebGL with soft shadows, ACES tone mapping and an optional Unreal-style bloom pass
  (falls back to plain rendering automatically)

## Play online

Enable **GitHub Pages** (Settings → Pages → deploy from branch → `main`) and the game
becomes playable at `https://drtsunamy.github.io/Blue-run/`.
