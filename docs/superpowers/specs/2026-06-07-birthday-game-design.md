# NopeYep Birthday Game — Design Spec
**Date:** 2026-06-07  
**Status:** Approved  

---

## 1. Overview

A single-file HTML5 canvas birthday gift game for **NopeYep**. Classic Binding of Isaac feel: fixed rooms, twin-stick combat, mechanical skill pickups, and a final secret garden with birthday content (photo board, video board, throne + confetti). Ships as one `index.html` + `img/` + `vid/` — double-click to run, no build step.

---

## 2. Architecture

**File:** `index.html` — inline CSS + JS, no external dependencies.  
**Canvas:** 900×560px, fixed (no scroll, no zoom, no camera follow).  
**Font:** Press Start 2P embedded as base64 `@font-face` — works fully offline.  
**Assets:** `img/board*.png` and `vid/video*.mp4` loaded at runtime with graceful missing-file placeholders. All game art drawn procedurally on canvas (sprite functions are isolated so real sprites can be swapped in later).

### JS section order (top of `<script>`)

```
// ─── CONFIG ─────── friend name, captions, video list, tuning numbers
// ─── ASSETS ─────── load img/ and vid/ with fallback placeholders  
// ─── STATE ──────── player, rooms, tears, enemies, skills, gameState
// ─── AUDIO ──────── Web Audio API: playShoot, playHit, playDoorUnlock,
//                    playSkillPickup, playBossIntro, playConfetti
// ─── DRAWING ─────── drawPlayer, drawEnemy, drawRoom, drawHUD,
//                    drawMinimap, drawPopup, drawConfetti, drawBossHPBar
// ─── ENEMIES ─────── spawnEnemy, updateGrub, updateFly, updateTank
// ─── BOSS ────────── updateCakeGolem, drawCakeGolem
// ─── SKILLS ─────────applySkill, updateOrbitals, updateTears (skill effects)
// ─── ROOMS ──────────room graph data, loadRoom, checkDoors, transition
// ─── LOOP ───────────update() + render() via requestAnimationFrame
```

### State machine

```
TITLE → PLAYING → TRANSITION (room slide, 0.3s)
                → SKILL_PICKUP (post-room-clear card select)
                → BOSS_INTRO (flash + name card, 2s)
                → POPUP (board open)
                → CELEBRATION (throne)
       all non-PLAYING states → back to PLAYING on dismiss
```

---

## 3. Room System

### Room graph (linear, 6 rooms)

```
[START] → [FIGHT 1] → [FIGHT 2] → [FIGHT 3] → [BOSS] → [GARDEN 🎂]
```

| Room | Enemies | Special |
|------|---------|---------|
| Start | none | Tutorial hint text, sets scene |
| Fight 1 | 3 grubs + 1 fly | Skill pickup on clear |
| Fight 2 | 2 flies + 2 grubs | Skill pickup on clear |
| Fight 3 | 1 tank + 2 grubs | Skill pickup on clear |
| Boss | Cake Golem | Boss intro flash; skill pickup on death |
| Garden | none | Photos board, Videos board, Throne |

### Room structure

- **Tile grid:** 15×9 tiles @ 60px = 900×540px playfield + 20px HUD strip at top.
- **Obstacles:** Per-room tile data (rocks, bushes). Player and enemies collide with them.
- **Doors:** N/S/E/W as needed. Lock (barred, red tint) while enemies alive. Unlock with chime + flash on room clear.
- **Transitions:** 0.3s horizontal canvas slide. Current room slides out, next slides in.
- **Art:** Garden theme — grass-tile floor with subtle variation, border walls, decorative flowers/pebbles. Soft vignette overlay.

### Minimap (Isaac-style, top-right HUD)

- Room-grid map, not a dot-on-terrain map.
- States: current room (gold highlight), visited (grey), unvisited (dark), garden (green + 🎂 icon).
- Updates immediately on room transition.

---

## 4. Combat System

### Player

| Stat | Value |
|------|-------|
| Speed | 3.5 px/frame |
| Max HP | 6 half-hearts (3 full hearts displayed) |
| i-frames | 1.5s after hit, player blinks |
| Controls | WASD move · Arrow keys shoot · E interact · Esc close popup |

- Instant response — no acceleration lag.
- Facing direction flips sprite horizontally.
- Soft shadow drawn under player feet.
- Sprite: `img/character.png` with white-fringe cleaned. Falls back to a placeholder shape.

### Tears

| Stat | Value |
|------|-------|
| Radius | 4px |
| Speed | 10 px/frame |
| Fire rate | every 8 frames (held key) |
| Lifespan | 0.6s (fade + shrink) |
| Range | ~160px before disappearing |

Juice: tiny muzzle flash on fire, particle splash on impact, brief screen nudge on player hit.

### Enemies

#### 🐛 Hopping Grub
- **HP:** 2 · **Speed:** 1.2 px/frame · **Behavior:** chase player
- **Animation:** hop cycle — squash on land, stretch mid-air (bob every 30 frames)
- **Death:** green particle pop

#### 🪰 Dive-Bomb Fly
- **HP:** 1 · **Speed:** 2.5 orbit / 6 dash · **Behavior:** orbit at mid-range → telegraph → dash in straight line → reset
- **Animation:** wing flutter (y-oscillate ±3px, 20-frame cycle)
- **Death:** blue particle burst

#### 🐞 Tank Bug
- **HP:** 3 · **Speed:** 0.7 px/frame · **Behavior:** slow straight chase, ignores obstacles
- **Animation:** shell pulse (scale ±2%, 60-frame breathe cycle) + HP pips on back (go dark when hit)
- **Death:** large red explosion + 0.2s screen shake

### Hit feedback (all enemies)
- Red flash on hit (3 frames)
- Knockback impulse away from tear direction
- Screen shake on tank death and boss phase transition

---

## 5. Skill Pickup System

Triggered after each room clear (3 combat rooms + boss = 4 pickups total).

### Flow
1. Room clears → enemies dead → doors stay locked
2. Skill pickup overlay appears (dims background slightly)
3. Two skill cards shown, drawn from the pool of 6 **without replacement** across the run
4. Player navigates with ← → arrow keys, confirms with E
5. Skill activates immediately (particle flash) + sound effect
6. Doors unlock

### The 8 mechanical skills (pool of 8 — 2 shown per pickup, drawn without replacement)

| Skill | Effect |
|-------|--------|
| 🎯 Homing Tears | Tears curve toward nearest enemy (steering force 0.3) |
| 💥 Explosive Tears | On impact, burst into 4 cardinal-direction shards |
| 👻 Piercing Tears | Pass through all enemies (no limit) |
| 🛡️ Orbital Shield | 2 orbs orbit player at r=50px, 1 damage per enemy contact |
| ❄️ Freeze Tears | Hits apply 60% slow for 2s (blue tint); re-hit refreshes duration |
| 🔄 Boomerang Tears | Reverse direction at max range; can hit same enemy twice |
| 🌀 Triple Shot | Fires 3 tears in a cone spread simultaneously |
| ⚡ Speed Tears | Tears move 80% faster; pairs excellently with Homing |

**Skills stack** — e.g. homing + explosive = homing shards that burst on landing.  
4 pickups × 2 cards each = 8 card slots, drawn from pool of 8 without replacement. Every run shows all 8 skills exactly once.

---

## 6. Boss — The Cake Golem

**Room:** dedicated boss room between Fight 3 and Garden.  
**Intro:** 2s flash screen with boss name in large retro font + ominous audio sting.  
**Art:** placeholder = layered rect (tiered cake shape). Real sprite drops into `img/boss-cake.png`.

### Stats

| | Value |
|-|-------|
| HP | 24 hits |
| Phase 2 threshold | ≤12 HP |
| Size | ~80×90px |
| Speed | 0.9 px/frame |

### Phase 1 (24–13 HP)
- Slowly stomps toward player.
- Every 3s: candles fire a radial burst of 8 evenly-spaced flame-blobs outward (telegraphed 1s early by candles glowing white).
- Stomp causes 0.1s screen shake.

### Phase 2 (≤12 HP)
- Visual: loses top cake tier (damaged sprite / rect removed).
- Radial burst now fires + 1 aimed blob directly at player simultaneously.
- Spawns 2 tank-bugs ("frosting golems") on phase transition.
- Stomp shake doubles.

### Death
- Large explosion particles in pink/white/yellow.
- 0.5s screen shake.
- Boss HP bar flashes then fades.
- Skill pedestal drops in center of room.
- Doors to Garden unlock after pedestal is collected.

### Boss HUD
- HP bar across bottom of screen (Isaac-style).
- Bar label: "THE CAKE GOLEM".

---

## 7. Birthday Content

### Photos Board
- Prop in Garden room. Press E within 48px to open.
- Popup: wooden frame border, left/right nav arrows (◀ ▶ or arrow keys).
- Loads `img/board1.png` … `img/board6.png`. Missing files show a clean placeholder tile.
- Captions editable in CONFIG block.
- Close with E or Esc.

### Videos Board
- Same as Photos board but image area is a `<video controls>` element.
- Loads `vid/video1.mp4` … `vid/video3.mp4`. Missing files show placeholder.
- Captions editable in CONFIG block.

### Throne
- Prop in Garden room. Press E within 48px.
- Triggers: confetti (~80 colored rects falling from top), centered banner overlay.
- Banner: "🎉 HAPPY BIRTHDAY, NopeYep! 🎉" + subtitle (editable in CONFIG).
- Ascending fanfare via Web Audio.
- Dismiss with E or Esc → player can roam freely.

---

## 8. Audio (Procedural — Web Audio API)

All sounds synthesized in-code. No external audio files.

| Event | Sound |
|-------|-------|
| Shoot | Short high-pitched sine blip |
| Tear impact | Mid thud |
| Player hit | Low buzz + pitch drop |
| Enemy death | Pop + short decay |
| Tank death | Low boom |
| Door unlock | Ascending 3-note chime |
| Skill pickup | Sparkle arpeggio |
| Boss intro | Ominous low drone + hit |
| Boss death | Fanfare (ascending major arpeggio) |
| Confetti | Celebratory ascending run |

---

## 9. CONFIG Block (top of JS)

```js
const CONFIG = {
  friendName: "NopeYep",
  subtitle: "from all your friends ♥",

  photos: [
    { file: "img/board1.png", caption: "Caption 1" },
    { file: "img/board2.png", caption: "Caption 2" },
    { file: "img/board3.png", caption: "Caption 3" },
    { file: "img/board4.png", caption: "Caption 4" },
    { file: "img/board5.png", caption: "Caption 5" },
    { file: "img/board6.png", caption: "Caption 6" },
  ],

  videos: [
    { file: "vid/video1.mp4", caption: "Video 1" },
    { file: "vid/video2.mp4", caption: "Video 2" },
    { file: "vid/video3.mp4", caption: "Video 3" },
  ],

  player: {
    speed: 3.5,
    maxHP: 6,
    iframeDuration: 90,   // frames
    tearSpeed: 10,
    tearRadius: 4,
    tearLifespan: 36,     // frames (~0.6s @ 60fps)
    fireRate: 8,          // frames between shots
  },
};
```

---

## 10. Art & Polish

- **Palette:** Warm garden greens + earthy browns + gold accents. Dark room border/vignette.
- **Player:** `img/character.png` with white fringe cleaned (alpha-keyed). Fallback: simple circle+face shape. Flips horizontally by facing direction.
- **Enemies:** Canvas-drawn placeholder shapes. Isolated `drawGrub()`, `drawFly()`, `drawTank()` functions — swap in real sprites by replacing function body.
- **Boss:** Canvas-drawn layered rect placeholder. Swap via `drawCakeGolem()`.
- **Font:** Press Start 2P embedded as base64 `@font-face`. Used consistently for all UI text.
- **Rooms:** Grass tile floor (2 variants, checkerboard), stone border walls, flower/pebble decorations, soft radial vignette overlay.
- **Title screen:** Game title, "Happy Birthday, NopeYep!", controls list, Press E/Enter to start.

---

## 11. Definition of Done

- [ ] Runs by double-clicking `index.html` — no console errors, works offline
- [ ] Movement + shooting feel tight (per §4 tuning)
- [ ] All 3 enemy types animate and die with feedback
- [ ] 6-room progression with locked/unlocked doors
- [ ] Skill pickup overlay after each room clear and boss kill
- [ ] All 8 mechanical skills implemented and stackable
- [ ] Cake Golem boss: 2 phases, HP bar, intro flash, death explosion
- [ ] Photos + Videos boards open, navigate, handle missing files gracefully
- [ ] Throne → confetti + birthday banner; can dismiss and roam
- [ ] Isaac-style minimap updates correctly
- [ ] HUD: hearts, room label, boss HP bar
- [ ] Title screen with controls
- [ ] Press Start 2P font renders consistently
- [ ] Visually verified in browser before shipping

---

## 12. Milestones (implementation order)

1. **Core loop** — canvas setup, player movement + shooting, one room, one enemy type, hearts HUD
2. **Room system** — 6-room graph, doors lock/unlock, slide transitions, minimap
3. **Enemies** — all 3 types + death juice + room enemy configs
4. **Skill system** — pickup overlay UI + all 6 skills + stacking logic
5. **Boss** — Cake Golem, 2 phases, HP bar, boss intro, death
6. **Art polish** — garden rooms, vignette, player sprite cleanup, font pass, title screen
7. **Birthday content** — photo board, video board, throne + confetti + banner
8. **Audio** — all Web Audio sfx
9. **Final verification** — browser test, console check, packaging instructions
