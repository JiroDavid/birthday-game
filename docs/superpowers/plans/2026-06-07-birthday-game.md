# NopeYep Birthday Game — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a polished single-file HTML5 canvas birthday game for NopeYep — Isaac-style top-down shooter with 6 rooms, 3 enemy types, 8 mechanical skills, a Cake Golem boss, and a final garden with photo/video boards and throne celebration.

**Architecture:** Flat procedural JavaScript in one `index.html`. All game logic is global variables and functions in labeled sections (CONFIG → ASSETS → STATE → AUDIO → DRAWING → ENEMIES → BOSS → SKILLS → ROOMS → LOOP). No classes, no build step, no external dependencies.

**Tech Stack:** HTML5 Canvas 2D API, Web Audio API, vanilla JS ES6+, Press Start 2P font (base64 embedded for offline use)

---

## File Map

| File | Role |
|------|------|
| `index.html` | Entire game — HTML shell, CSS, all JS sections |
| `img/character.png` | Player sprite (user-supplied, placeholder if missing) |
| `img/board1-6.png` | Photo board images (user-supplied, placeholder if missing) |
| `img/boss-cake.png` | Boss sprite (user-supplied, placeholder if missing) |
| `vid/video1-3.mp4` | Video messages (user-supplied, placeholder if missing) |

All game logic lives in `index.html`. Tasks below add/expand sections within it.

---

## Task 1: HTML scaffold + CONFIG + canvas + game loop skeleton + title screen

**Files:**
- Create: `index.html`

- [ ] **Step 1.1 — Create the HTML shell with canvas and font**

Embed Press Start 2P font. Download the woff2 and encode it:
```bash
curl -sL "https://fonts.gstatic.com/s/pressstart2p/v15/e3t4euO8T-267oIAQAu6jDQyK3nYivN04w.woff2" | base64 | tr -d '\n' > /tmp/p2p_b64.txt
```
Then write `index.html` with `<BASE64_FONT>` replaced by the contents of `/tmp/p2p_b64.txt`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Happy Birthday, NopeYep!</title>
<style>
  @font-face {
    font-family: 'Press Start 2P';
    src: url('data:font/woff2;base64,<BASE64_FONT>') format('woff2');
  }
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    background: #0a0f0a;
    display: flex;
    align-items: center;
    justify-content: center;
    min-height: 100vh;
    overflow: hidden;
  }
  canvas {
    display: block;
    image-rendering: pixelated;
    border: 2px solid #2a4a2a;
  }
</style>
</head>
<body>
<canvas id="game" width="900" height="560"></canvas>
<script>
'use strict';
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const W = 900, H = 560;
const HUD_H = 20;          // top HUD strip height
const TILE = 60;           // tile size
const COLS = 15, ROWS = 9; // tile grid
// Playfield top offset (below HUD)
const PF_TOP = HUD_H;

// ─── CONFIG ───────────────────────────────────────────────────────────────
const CONFIG = {
  friendName: 'NopeYep',
  subtitle: 'from all your friends ♥',

  photos: [
    { file: 'img/board1.png', caption: 'Caption 1' },
    { file: 'img/board2.png', caption: 'Caption 2' },
    { file: 'img/board3.png', caption: 'Caption 3' },
    { file: 'img/board4.png', caption: 'Caption 4' },
    { file: 'img/board5.png', caption: 'Caption 5' },
    { file: 'img/board6.png', caption: 'Caption 6' },
  ],
  videos: [
    { file: 'vid/video1.mp4', caption: 'Video 1' },
    { file: 'vid/video2.mp4', caption: 'Video 2' },
    { file: 'vid/video3.mp4', caption: 'Video 3' },
  ],

  player: {
    speed: 3.5,
    maxHP: 6,
    iframeDuration: 90,  // frames (~1.5s @ 60fps)
    tearSpeed: 10,
    tearRadius: 4,
    tearLifespan: 36,    // frames (~0.6s @ 60fps)
    fireRate: 8,
  },
};
</script>
</body>
</html>
```

- [ ] **Step 1.2 — Add STATE section (all mutable globals)**

Inside `<script>`, after CONFIG:

```js
// ─── STATE ────────────────────────────────────────────────────────────────
let gameState = 'TITLE'; // TITLE | PLAYING | TRANSITION | SKILL_PICKUP | BOSS_INTRO | POPUP | CELEBRATION

const keys = {};
let frameCount = 0;
let screenShake = { x: 0, y: 0, timer: 0 };

const player = {
  x: 450, y: 300,
  vx: 0, vy: 0,
  hp: CONFIG.player.maxHP,
  maxHP: CONFIG.player.maxHP,
  iframes: 0,
  fireTimer: 0,
  facing: 1,           // 1 = right, -1 = left
  sprite: null,        // Image element, set in ASSETS
  skills: {
    homing: false, explosive: false, piercing: false,
    orbital: false, freeze: false, boomerang: false,
    triple: false, speed: false,
  },
  orbitals: [],        // [{angle, speed}] active orbital orbs
};

const tears = [];      // { x, y, vx, vy, life, radius, hitEnemies: Set, returning }
const enemies = [];    // see enemy types below
const particles = [];  // { x, y, vx, vy, life, maxLife, r, color }

let currentRoomIdx = 0;
let boss = null;       // Bloom object when active

// Transition state
const transition = { active: false, progress: 0, duration: 18, fromIdx: 0, toIdx: 0, dir: 1 };

// Skill pickup state
const skillPickupState = { choices: [], selected: 0 };
let skillPool = [];    // shuffled on game start, drawn without replacement

// Popup state
const popupState = { type: null, index: 0 }; // type: 'photos' | 'videos'

// Boss intro state
const bossIntro = { timer: 0, duration: 120 }; // 2s

// Celebration confetti
const confettiParticles = [];

// Pet companion (mini Bloom, befriended after boss death)
let pet = { x: 0, y: 0, active: false, befriended: false, bounceTimer: 0 };
```

- [ ] **Step 1.3 — Add the game loop skeleton and input handling**

```js
// ─── LOOP ─────────────────────────────────────────────────────────────────
function update() {
  frameCount++;
  if (gameState === 'TITLE') return;
  // (other states filled in later tasks)
}

function render() {
  ctx.save();
  if (screenShake.timer > 0) {
    ctx.translate(screenShake.x, screenShake.y);
    screenShake.timer--;
    screenShake.x = (Math.random() - 0.5) * screenShake.timer * 0.5;
    screenShake.y = (Math.random() - 0.5) * screenShake.timer * 0.5;
    if (screenShake.timer <= 0) { screenShake.x = 0; screenShake.y = 0; }
  }
  ctx.clearRect(-10, -10, W + 20, H + 20);

  if (gameState === 'TITLE') {
    drawTitle();
  }
  // (other states filled in later tasks)

  ctx.restore();
}

window.addEventListener('keydown', e => {
  keys[e.key] = true;
  e.preventDefault();
});
window.addEventListener('keyup', e => {
  keys[e.key] = false;
});

function startShake(intensity, duration) {
  screenShake.timer = duration;
  screenShake.x = (Math.random() - 0.5) * intensity;
  screenShake.y = (Math.random() - 0.5) * intensity;
}

(function loop() {
  update();
  render();
  requestAnimationFrame(loop);
})();
```

- [ ] **Step 1.4 — Add drawTitle() and DRAWING section stub**

```js
// ─── DRAWING ──────────────────────────────────────────────────────────────
function px(size) { return `${size}px 'Press Start 2P', monospace`; }

function drawTitle() {
  // Background
  ctx.fillStyle = '#0d1f0d';
  ctx.fillRect(0, 0, W, H);

  // Decorative border
  ctx.strokeStyle = '#3a7a3a';
  ctx.lineWidth = 4;
  ctx.strokeRect(20, 20, W - 40, H - 40);

  // Title
  ctx.fillStyle = '#e8c547';
  ctx.font = px(28);
  ctx.textAlign = 'center';
  ctx.fillText('NOPEYEP', W / 2, 160);
  ctx.fillStyle = '#a0e060';
  ctx.font = px(12);
  ctx.fillText('BIRTHDAY QUEST', W / 2, 200);

  // Subtitle
  ctx.fillStyle = '#7ab87a';
  ctx.font = px(8);
  ctx.fillText(`a birthday gift for ${CONFIG.friendName}`, W / 2, 260);

  // Controls
  ctx.fillStyle = '#4a7a4a';
  ctx.font = px(7);
  ctx.fillText('WASD move   ARROWS shoot   E interact', W / 2, 340);

  // Prompt
  if (Math.floor(frameCount / 30) % 2 === 0) {
    ctx.fillStyle = '#e8c547';
    ctx.font = px(8);
    ctx.fillText('PRESS ENTER TO START', W / 2, 420);
  }
  ctx.textAlign = 'left';
}
```

- [ ] **Step 1.5 — Wire Enter key to start the game**

In the `keydown` listener, add before `e.preventDefault()`:
```js
  if (e.key === 'Enter' && gameState === 'TITLE') {
    initGame();
  }
```

Add `initGame()` function (stub for now):
```js
function initGame() {
  // Reset player
  player.hp = player.maxHP;
  player.iframes = 0;
  player.x = 450; player.y = 300;
  Object.keys(player.skills).forEach(k => player.skills[k] = false);
  player.orbitals = [];

  // Shuffle skill pool
  skillPool = ['homing','explosive','piercing','orbital','freeze','boomerang','triple','speed'];
  for (let i = skillPool.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [skillPool[i], skillPool[j]] = [skillPool[j], skillPool[i]];
  }

  tears.length = 0;
  enemies.length = 0;
  particles.length = 0;
  currentRoomIdx = 0;
  boss = null;
  gameState = 'PLAYING';
  // (loadRoom called in Task 4)
}
```

- [ ] **Step 1.6 — Open in browser and verify**

Open `index.html` by double-clicking or `open index.html` (macOS) / `start index.html` (Windows).

Verify:
- Black background, canvas centered
- Title renders with Press Start 2P font (pixelated look)
- "PRESS ENTER TO START" blinks
- No console errors

- [ ] **Step 1.7 — Commit**

```bash
git add index.html
git commit -m "feat: add HTML scaffold, CONFIG, state skeleton, title screen"
```

---

## Task 2: Player movement, shooting, tears, and hearts HUD

**Files:**
- Modify: `index.html` (STATE, DRAWING, LOOP sections)

- [ ] **Step 2.1 — Implement player movement in update()**

Replace the `if (gameState === 'TITLE') return;` stub in `update()`:

```js
function update() {
  frameCount++;
  if (gameState === 'TITLE') return;

  if (gameState === 'PLAYING') {
    updatePlayer();
    updateTears();
    updateParticles();
  }
}

function updatePlayer() {
  // Movement
  let dx = 0, dy = 0;
  if (keys['w'] || keys['W']) dy -= 1;
  if (keys['s'] || keys['S']) dy += 1;
  if (keys['a'] || keys['A']) dx -= 1;
  if (keys['d'] || keys['D']) dx += 1;

  // Normalize diagonal
  if (dx !== 0 && dy !== 0) { dx *= 0.707; dy *= 0.707; }

  player.x = Math.max(TILE, Math.min(W - TILE, player.x + dx * CONFIG.player.speed));
  player.y = Math.max(PF_TOP + TILE, Math.min(H - TILE, player.y + dy * CONFIG.player.speed));

  if (dx > 0) player.facing = 1;
  if (dx < 0) player.facing = -1;

  // i-frames countdown
  if (player.iframes > 0) player.iframes--;

  // Shooting
  player.fireTimer = Math.max(0, player.fireTimer - 1);
  let sx = 0, sy = 0;
  if (keys['ArrowRight']) sx = 1;
  if (keys['ArrowLeft'])  sx = -1;
  if (keys['ArrowDown'])  sy = 1;
  if (keys['ArrowUp'])    sy = -1;

  if ((sx !== 0 || sy !== 0) && player.fireTimer === 0) {
    fireTear(sx, sy);
    player.fireTimer = CONFIG.player.fireRate;
  }
}
```

- [ ] **Step 2.2 — Implement fireTear() and updateTears()**

```js
function fireTear(sx, sy) {
  const speed = CONFIG.player.tearSpeed * (player.skills.speed ? 1.8 : 1);
  const len = Math.hypot(sx, sy);
  const baseTear = {
    x: player.x, y: player.y,
    vx: (sx / len) * speed,
    vy: (sy / len) * speed,
    life: CONFIG.player.tearLifespan,
    radius: CONFIG.player.tearRadius,
    hitEnemies: new Set(),
    returning: false,
  };

  if (player.skills.triple) {
    // 3-way spread: center + ±15 degrees
    [-15, 0, 15].forEach(deg => {
      const rad = deg * Math.PI / 180;
      const cos = Math.cos(rad), sin = Math.sin(rad);
      tears.push({
        ...baseTear,
        hitEnemies: new Set(),
        vx: baseTear.vx * cos - baseTear.vy * sin,
        vy: baseTear.vx * sin + baseTear.vy * cos,
      });
    });
  } else {
    tears.push({ ...baseTear });
  }
}

function updateTears() {
  for (let i = tears.length - 1; i >= 0; i--) {
    const t = tears[i];

    // Homing steering
    if (player.skills.homing && !t.returning) {
      let nearest = null, nearestDist = Infinity;
      enemies.forEach(e => {
        const d = Math.hypot(e.x - t.x, e.y - t.y);
        if (d < nearestDist) { nearestDist = d; nearest = e; }
      });
      if (boss) {
        const d = Math.hypot(boss.x - t.x, boss.y - t.y);
        if (d < nearestDist) nearest = boss;
      }
      if (nearest) {
        const dx = nearest.x - t.x, dy = nearest.y - t.y;
        const len = Math.hypot(dx, dy) || 1;
        const force = 0.3;
        t.vx += (dx / len) * force;
        t.vy += (dy / len) * force;
        // Clamp speed
        const spd = Math.hypot(t.vx, t.vy);
        const maxSpd = CONFIG.player.tearSpeed * (player.skills.speed ? 1.8 : 1);
        if (spd > maxSpd) { t.vx = t.vx / spd * maxSpd; t.vy = t.vy / spd * maxSpd; }
      }
    }

    // Boomerang: reverse at end of life midpoint
    if (player.skills.boomerang && !t.returning && t.life <= CONFIG.player.tearLifespan / 2) {
      t.returning = true;
      t.vx = -t.vx; t.vy = -t.vy;
      t.hitEnemies.clear(); // can hit again on return
    }

    t.x += t.vx;
    t.y += t.vy;
    t.life--;

    // Wall collision
    if (t.x < TILE || t.x > W - TILE || t.y < PF_TOP + TILE || t.y > H - TILE) {
      impactParticles(t.x, t.y, '#88ccff');
      tears.splice(i, 1);
    } else if (t.life <= 0) {
      tears.splice(i, 1);
    }
  }
}
```

- [ ] **Step 2.3 — Add particle system**

```js
function spawnParticles(x, y, color, count, speed, life) {
  for (let i = 0; i < count; i++) {
    const angle = Math.random() * Math.PI * 2;
    const spd = speed * (0.5 + Math.random() * 0.5);
    particles.push({
      x, y,
      vx: Math.cos(angle) * spd,
      vy: Math.sin(angle) * spd,
      life: life * (0.7 + Math.random() * 0.3),
      maxLife: life,
      r: 3 + Math.random() * 3,
      color,
    });
  }
}

function impactParticles(x, y, color) {
  spawnParticles(x, y, color, 5, 3, 12);
}

function updateParticles() {
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i];
    p.x += p.vx; p.y += p.vy;
    p.vx *= 0.92; p.vy *= 0.92;
    p.life--;
    if (p.life <= 0) particles.splice(i, 1);
  }
}
```

- [ ] **Step 2.4 — Add drawing functions for player, tears, particles, and HUD**

```js
function drawPlayer() {
  ctx.save();
  // Shadow
  ctx.fillStyle = 'rgba(0,0,0,0.3)';
  ctx.beginPath();
  ctx.ellipse(player.x, player.y + 22, 14, 6, 0, 0, Math.PI * 2);
  ctx.fill();

  // Blink during i-frames
  if (player.iframes > 0 && Math.floor(frameCount / 4) % 2 === 1) {
    ctx.restore(); return;
  }

  if (player.sprite) {
    ctx.translate(player.x, player.y);
    ctx.scale(player.facing, 1);
    ctx.drawImage(player.sprite, -20, -28, 40, 48);
  } else {
    // Placeholder: circle + face
    ctx.translate(player.x, player.y);
    ctx.scale(player.facing, 1);
    ctx.fillStyle = '#c8a87a';
    ctx.beginPath(); ctx.arc(0, -8, 18, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#222';
    ctx.beginPath(); ctx.arc(-6, -10, 3, 0, Math.PI * 2); ctx.fill();
    ctx.beginPath(); ctx.arc(6, -10, 3, 0, Math.PI * 2); ctx.fill();
    // Tears on cheeks
    ctx.fillStyle = '#88aaff';
    ctx.beginPath(); ctx.ellipse(-7, -4, 2, 4, 0, 0, Math.PI * 2); ctx.fill();
    ctx.beginPath(); ctx.ellipse(7, -4, 2, 4, 0, 0, Math.PI * 2); ctx.fill();
  }
  ctx.restore();
}

function drawTears() {
  tears.forEach(t => {
    const alpha = Math.max(0, t.life / CONFIG.player.tearLifespan);
    const r = t.radius * alpha;
    ctx.save();
    ctx.globalAlpha = alpha;
    ctx.fillStyle = player.skills.freeze ? '#88ccff' :
                    player.skills.explosive ? '#ff8844' : '#88aaff';
    ctx.beginPath(); ctx.arc(t.x, t.y, r, 0, Math.PI * 2); ctx.fill();
    ctx.restore();
  });
}

function drawParticles() {
  particles.forEach(p => {
    const alpha = p.life / p.maxLife;
    ctx.save();
    ctx.globalAlpha = alpha;
    ctx.fillStyle = p.color;
    ctx.beginPath(); ctx.arc(p.x, p.y, p.r * alpha, 0, Math.PI * 2); ctx.fill();
    ctx.restore();
  });
}

function drawHUD() {
  // HUD background strip
  ctx.fillStyle = '#0d1a0d';
  ctx.fillRect(0, 0, W, HUD_H);

  // Hearts (left side)
  const heartSize = 14;
  for (let i = 0; i < player.maxHP; i++) {
    const x = 8 + i * (heartSize + 2);
    const y = 3;
    ctx.fillStyle = i < player.hp ? '#e05252' : '#3a2a2a';
    // Simple heart using two arcs + triangle
    ctx.save();
    ctx.translate(x + heartSize / 2, y + heartSize / 2 + 1);
    ctx.scale(heartSize / 20, heartSize / 20);
    ctx.beginPath();
    ctx.arc(-5, -3, 5, Math.PI, 0);
    ctx.arc(5, -3, 5, Math.PI, 0);
    ctx.lineTo(0, 8);
    ctx.closePath();
    ctx.fill();
    ctx.restore();
  }

  // Room name (center) — placeholder, set in Task 4
  ctx.fillStyle = '#a0c87a';
  ctx.font = px(6);
  ctx.textAlign = 'center';
  ctx.fillText(currentRoom ? currentRoom.name : '', W / 2, 13);
  ctx.textAlign = 'left';
}
```

- [ ] **Step 2.5 — Wire drawing into render()**

Replace the render() body:
```js
function render() {
  ctx.save();
  if (screenShake.timer > 0) {
    ctx.translate(screenShake.x, screenShake.y);
    screenShake.timer--;
    screenShake.x = (Math.random() - 0.5) * screenShake.timer * 0.5;
    screenShake.y = (Math.random() - 0.5) * screenShake.timer * 0.5;
    if (screenShake.timer <= 0) { screenShake.x = 0; screenShake.y = 0; }
  }
  ctx.clearRect(-10, -10, W + 20, H + 20);

  if (gameState === 'TITLE') { drawTitle(); ctx.restore(); return; }

  // Playfield background (placeholder green until Task 4 adds room art)
  ctx.fillStyle = '#1e3a1e';
  ctx.fillRect(0, PF_TOP, W, H - PF_TOP);

  drawTears();
  drawPlayer();
  drawParticles();
  drawHUD();

  ctx.restore();
}
```

Also add `let currentRoom = null;` to STATE.

- [ ] **Step 2.6 — Verify in browser**

Press Enter on title. Verify:
- Player (placeholder circle or character.png) appears
- WASD moves player, player stays within wall boundary
- Arrow keys fire blue tear dots that travel and fade
- Hearts appear in HUD top-left
- No console errors

- [ ] **Step 2.7 — Commit**

```bash
git add index.html
git commit -m "feat: player movement, tear shooting, hearts HUD, particles"
```

---

## Task 3: Room data structure, wall collision, Grub enemy, combat loop

**Files:**
- Modify: `index.html` (ROOMS, ENEMIES, LOOP sections)

- [ ] **Step 3.1 — Define ROOM_GRAPH with 6 rooms**

Add the ROOMS section:

```js
// ─── ROOMS ────────────────────────────────────────────────────────────────

// Obstacle tiles: arrays of [tileCol, tileRow] that block movement
// Tile (0,0) is top-left of playfield. Usable area: cols 1–13, rows 1–7.
const ROOM_GRAPH = [
  {
    id: 0, name: 'Garden Path', type: 'start',
    doors: { north: null, south: null, east: 1, west: null },
    enemySpawns: [],
    obstacles: [[3,2],[3,3],[11,5],[11,6],[7,1],[7,7]],
    cleared: true, visited: true,
  },
  {
    id: 1, name: 'The Thicket', type: 'fight',
    doors: { north: null, south: null, east: 2, west: 0 },
    enemySpawns: [
      { type: 'grub', x: 200, y: 180 },
      { type: 'grub', x: 650, y: 350 },
      { type: 'grub', x: 420, y: 260 },
      { type: 'fly',  x: 700, y: 200 },
    ],
    obstacles: [[2,2],[2,3],[12,5],[12,6],[5,4],[9,4]],
    cleared: false, visited: false,
  },
  {
    id: 2, name: 'Mossy Clearing', type: 'fight',
    doors: { north: null, south: null, east: 3, west: 1 },
    enemySpawns: [
      { type: 'fly', x: 200, y: 200 },
      { type: 'fly', x: 650, y: 200 },
      { type: 'grub', x: 300, y: 380 },
      { type: 'grub', x: 550, y: 380 },
    ],
    obstacles: [[3,1],[3,7],[11,1],[11,7],[7,3],[7,5]],
    cleared: false, visited: false,
  },
  {
    id: 3, name: 'Stone Garden', type: 'fight',
    doors: { north: null, south: null, east: 4, west: 2 },
    enemySpawns: [
      { type: 'tank', x: 450, y: 280 },
      { type: 'grub', x: 200, y: 200 },
      { type: 'grub', x: 650, y: 380 },
    ],
    obstacles: [[2,2],[2,6],[12,2],[12,6],[4,4],[10,4]],
    cleared: false, visited: false,
  },
  {
    id: 4, name: 'Boss Chamber', type: 'boss',
    doors: { north: null, south: null, east: 5, west: 3 },
    enemySpawns: [], // boss spawned separately
    obstacles: [[2,2],[12,2],[2,6],[12,6]],
    cleared: false, visited: false,
  },
  {
    id: 5, name: 'Secret Garden', type: 'garden',
    doors: { north: null, south: null, east: null, west: 4 },
    enemySpawns: [],
    obstacles: [],
    cleared: true, visited: false,
    // Props: boards and throne positions
    photoBoard:  { x: 200, y: 280 },
    videoBoard:  { x: 450, y: 280 },
    throne:      { x: 700, y: 280 },
  },
];

function loadRoom(idx) {
  const room = ROOM_GRAPH[idx];
  currentRoom = room;
  currentRoomIdx = idx;
  room.visited = true;

  // Reset enemies
  enemies.length = 0;
  tears.length = 0;
  particles.length = 0;
  boss = null;

  // Place player at appropriate door entry
  // (direction-aware in Task 4; for now, center)
  player.x = W / 2;
  player.y = H / 2 + 60;

  // Spawn enemies
  room.enemySpawns.forEach(s => spawnEnemy(s.type, s.x, s.y));

  // Boss room
  if (room.type === 'boss') {
    initBoss();
  }
}
```

- [ ] **Step 3.2 — Implement tile-based wall collision helper**

```js
function tileRect(tx, ty) {
  return { x: tx * TILE, y: PF_TOP + ty * TILE, w: TILE, h: TILE };
}

function isObstacle(tx, ty) {
  if (!currentRoom) return false;
  return currentRoom.obstacles.some(([ox, oy]) => ox === tx && oy === ty);
}

function circleCollidesRect(cx, cy, cr, rx, ry, rw, rh) {
  const nearX = Math.max(rx, Math.min(cx, rx + rw));
  const nearY = Math.max(ry, Math.min(cy, ry + rh));
  return Math.hypot(cx - nearX, cy - nearY) < cr;
}

function resolveCircleVsObstacles(obj, radius) {
  // obj must have .x and .y
  // Check surrounding tiles
  const tx = Math.floor(obj.x / TILE);
  const ty = Math.floor((obj.y - PF_TOP) / TILE);
  for (let dy = -1; dy <= 1; dy++) {
    for (let dx = -1; dx <= 1; dx++) {
      const ntx = tx + dx, nty = ty + dy;
      if (ntx < 1 || ntx > COLS - 2 || nty < 1 || nty > ROWS - 2) continue;
      if (!isObstacle(ntx, nty)) continue;
      const r = tileRect(ntx, nty);
      if (circleCollidesRect(obj.x, obj.y, radius, r.x, r.y, r.w, r.h)) {
        // Push out (simple: find shortest axis)
        const overlapX = Math.min(obj.x + radius - r.x, r.x + r.w - obj.x + radius);
        const overlapY = Math.min(obj.y + radius - r.y, r.y + r.h - obj.y + radius);
        if (overlapX < overlapY) {
          obj.x += (obj.x < r.x + r.w / 2) ? -overlapX : overlapX;
        } else {
          obj.y += (obj.y < r.y + r.h / 2) ? -overlapY : overlapY;
        }
      }
    }
  }
}

function clampToRoom(obj, radius) {
  obj.x = Math.max(TILE + radius, Math.min(W - TILE - radius, obj.x));
  obj.y = Math.max(PF_TOP + TILE + radius, Math.min(H - TILE - radius, obj.y));
}
```

Apply obstacle resolution in `updatePlayer()`, after position update:
```js
  resolveCircleVsObstacles(player, 16);
  clampToRoom(player, 16);
```

- [ ] **Step 3.3 — Implement Grub enemy and spawnEnemy()**

Add the ENEMIES section:

```js
// ─── ENEMIES ──────────────────────────────────────────────────────────────
function spawnEnemy(type, x, y) {
  const base = { type, x, y, hitFlash: 0, slowTimer: 0, knockVx: 0, knockVy: 0 };
  if (type === 'grub') enemies.push({ ...base, hp: 2, hopTimer: 0 });
  if (type === 'fly')  enemies.push({ ...base, hp: 1, phase: 'orbit', orbitAngle: Math.random() * Math.PI * 2, dashVx: 0, dashVy: 0, dashTimer: 0, telegraphTimer: 0, orbitRadius: 120 + Math.random() * 60 });
  if (type === 'tank') enemies.push({ ...base, hp: 3, pulseTimer: 0 });
}

function updateGrub(e) {
  const speed = 1.2 * (e.slowTimer > 0 ? 0.4 : 1);
  const dx = player.x - e.x, dy = player.y - e.y;
  const len = Math.hypot(dx, dy) || 1;

  // Knockback decays
  e.knockVx *= 0.85; e.knockVy *= 0.85;

  e.x += (dx / len) * speed + e.knockVx;
  e.y += (dy / len) * speed + e.knockVy;

  resolveCircleVsObstacles(e, 14);
  clampToRoom(e, 14);

  e.hopTimer = (e.hopTimer + 1) % 30;
  if (e.slowTimer > 0) e.slowTimer--;
  if (e.hitFlash > 0) e.hitFlash--;
}
```

- [ ] **Step 3.4 — Add drawGrub() and enemy draw dispatcher**

```js
function drawGrub(e) {
  const bob = Math.sin(e.hopTimer / 30 * Math.PI * 2) * 4;
  const squash = 1 + Math.abs(Math.sin(e.hopTimer / 30 * Math.PI * 2)) * 0.2;

  ctx.save();
  ctx.translate(e.x, e.y + bob);
  ctx.scale(1 / squash, squash);

  // Blue tint if frozen
  const bodyColor = e.hitFlash > 0 ? '#ffffff' :
                    e.slowTimer > 0 ? '#88ccff' : '#4a8a2a';
  const highColor  = e.hitFlash > 0 ? '#ffffff' :
                     e.slowTimer > 0 ? '#aaddff' : '#5aaa3a';

  // Body
  ctx.fillStyle = bodyColor;
  ctx.beginPath(); ctx.ellipse(0, 0, 18, 13, 0, 0, Math.PI * 2); ctx.fill();
  ctx.fillStyle = highColor;
  ctx.beginPath(); ctx.ellipse(0, -2, 14, 10, 0, 0, Math.PI * 2); ctx.fill();

  // Eyes
  ctx.fillStyle = '#fff';
  ctx.beginPath(); ctx.arc(-7, -5, 4, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath(); ctx.arc(7, -5, 4, 0, Math.PI * 2); ctx.fill();
  ctx.fillStyle = '#222';
  ctx.beginPath(); ctx.arc(-6, -5, 2.5, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath(); ctx.arc(8, -5, 2.5, 0, Math.PI * 2); ctx.fill();

  ctx.restore();
}

function drawEnemy(e) {
  if (e.type === 'grub') drawGrub(e);
  else if (e.type === 'fly')  drawFly(e);
  else if (e.type === 'tank') drawTank(e);
}
```

- [ ] **Step 3.5 — Implement tear-enemy collision and enemy damage**

Add `checkTearEnemyCollisions()` and `damageEnemy()`:

```js
function damageEnemy(e, tearVx, tearVy, idx) {
  e.hp--;
  e.hitFlash = 6;
  // Knockback
  const len = Math.hypot(tearVx, tearVy) || 1;
  e.knockVx = (tearVx / len) * 4;
  e.knockVy = (tearVy / len) * 4;

  if (player.skills.freeze) {
    e.slowTimer = 120; // 2s
  }

  if (e.hp <= 0) killEnemy(e, idx);
}

function killEnemy(e, idx) {
  const colors = { grub: '#4aaa2a', fly: '#6666ff', tank: '#cc2222' };
  const color = colors[e.type] || '#ffffff';
  spawnParticles(e.x, e.y, color, e.type === 'tank' ? 20 : 10,
                 e.type === 'tank' ? 6 : 4,
                 e.type === 'tank' ? 40 : 20);
  if (e.type === 'tank') startShake(8, 12);
  enemies.splice(idx, 1);
  checkRoomClear();
}

function checkTearEnemyCollisions() {
  for (let ti = tears.length - 1; ti >= 0; ti--) {
    const t = tears[ti];
    let tearUsed = false;

    for (let ei = enemies.length - 1; ei >= 0; ei--) {
      const e = enemies[ei];
      if (t.hitEnemies.has(ei)) continue;
      if (Math.hypot(t.x - e.x, t.y - e.y) < t.radius + 14) {
        // Impact
        if (!player.skills.piercing) {
          // Explosive burst
          if (player.skills.explosive) {
            [[1,0],[-1,0],[0,1],[0,-1]].forEach(([sx, sy]) => {
              tears.push({
                x: t.x, y: t.y,
                vx: sx * CONFIG.player.tearSpeed * 0.7,
                vy: sy * CONFIG.player.tearSpeed * 0.7,
                life: 18, radius: t.radius,
                hitEnemies: new Set([ei]),
                returning: false,
              });
            });
          }
          impactParticles(t.x, t.y, '#88aaff');
          damageEnemy(e, t.vx, t.vy, ei);
          tears.splice(ti, 1);
          tearUsed = true;
          break;
        } else {
          // Piercing: mark hit, continue
          t.hitEnemies.add(ei);
          damageEnemy(e, t.vx, t.vy, ei);
          impactParticles(t.x, t.y, '#88aaff');
        }
      }
    }
    if (tearUsed) continue;

    // Boss collision handled in Task 7
  }
}
```

Add `checkPlayerEnemyCollisions()`:
```js
function checkPlayerEnemyCollisions() {
  if (player.iframes > 0) return;
  enemies.forEach(e => {
    if (Math.hypot(player.x - e.x, player.y - e.y) < 16 + 14) {
      player.hp--;
      player.iframes = CONFIG.player.iframeDuration;
      startShake(4, 8);
      if (player.hp <= 0) gameState = 'TITLE'; // Game over → title for now
    }
  });
}
```

- [ ] **Step 3.6 — Wire enemy update into main update()**

Add to the `if (gameState === 'PLAYING')` block:
```js
    for (let i = enemies.length - 1; i >= 0; i--) {
      const e = enemies[i];
      if (e.type === 'grub') updateGrub(e);
      // fly and tank added in Task 5
    }
    checkTearEnemyCollisions();
    checkPlayerEnemyCollisions();
```

Add enemies to render() before `drawParticles()`:
```js
  enemies.forEach(drawEnemy);
```

Add `checkRoomClear()` stub (full version in Task 4):
```js
function checkRoomClear() {
  if (enemies.length === 0 && boss === null) {
    currentRoom.cleared = true;
    // Skill pickup + door unlock in Task 4
  }
}
```

Call `loadRoom(0)` at the end of `initGame()`.

- [ ] **Step 3.7 — Verify in browser**

Press Enter, verify:
- Green grubs appear and hop toward player
- Arrow key tears hit and kill grubs (green particles)
- Player takes damage on contact, blinks, then regains control
- Hearts decrease correctly
- No console errors

- [ ] **Step 3.8 — Commit**

```bash
git add index.html
git commit -m "feat: room data structure, wall collision, grub enemy, combat loop"
```

---

## Task 4: Room system — 6 rooms, doors, transitions, minimap

**Files:**
- Modify: `index.html` (ROOMS, DRAWING, LOOP sections)

- [ ] **Step 4.1 — Draw room tiles and doors**

```js
function drawRoom(room) {
  // Grass floor — two tile variants in checkerboard
  for (let ty = 0; ty < ROWS; ty++) {
    for (let tx = 0; tx < COLS; tx++) {
      const rx = tx * TILE, ry = PF_TOP + ty * TILE;
      if (tx === 0 || tx === COLS - 1 || ty === 0 || ty === ROWS - 1) {
        // Border wall (stone)
        ctx.fillStyle = '#4a4a3a';
        ctx.fillRect(rx, ry, TILE, TILE);
        ctx.fillStyle = '#5a5a4a';
        ctx.fillRect(rx + 2, ry + 2, TILE - 4, TILE - 4);
        // Stone crack lines
        ctx.strokeStyle = '#3a3a2a';
        ctx.lineWidth = 1;
        ctx.beginPath();
        ctx.moveTo(rx + 8, ry + 4); ctx.lineTo(rx + 20, ry + 14);
        ctx.moveTo(rx + TILE - 10, ry + TILE - 8); ctx.lineTo(rx + TILE - 22, ry + TILE - 18);
        ctx.stroke();
      } else if (room.obstacles.some(([ox, oy]) => ox === tx && oy === ty)) {
        // Obstacle (rock/bush)
        ctx.fillStyle = '#2a4a2a';
        ctx.fillRect(rx, ry, TILE, TILE);
        ctx.fillStyle = '#3a6a3a';
        ctx.beginPath(); ctx.arc(rx + TILE/2, ry + TILE/2, 22, 0, Math.PI * 2); ctx.fill();
        ctx.fillStyle = '#4a8a4a';
        ctx.beginPath(); ctx.arc(rx + TILE/2, ry + TILE/2 - 4, 16, 0, Math.PI * 2); ctx.fill();
      } else {
        // Grass (two shades in checkerboard)
        ctx.fillStyle = (tx + ty) % 2 === 0 ? '#2a5a2a' : '#2e6430';
        ctx.fillRect(rx, ry, TILE, TILE);
      }
    }
  }

  // Doors
  const doorW = 60, doorH = TILE;
  const doorDefs = {
    north: { x: W/2 - doorW/2, y: PF_TOP },
    south: { x: W/2 - doorW/2, y: H - doorH },
    east:  { x: W - doorH,     y: H/2 - doorW/2 },
    west:  { x: 0,             y: H/2 - doorW/2  },
  };

  Object.entries(room.doors).forEach(([dir, targetIdx]) => {
    if (targetIdx === null) return;
    const d = doorDefs[dir];
    const locked = !room.cleared;
    ctx.fillStyle = locked ? '#5a1a1a' : '#4a7a2a';
    const isVertical = dir === 'north' || dir === 'south';
    const dw = isVertical ? doorW : doorH;
    const dh = isVertical ? doorH : doorW;
    ctx.fillRect(d.x, d.y, dw, dh);
    if (locked) {
      // Bars
      ctx.strokeStyle = '#883333';
      ctx.lineWidth = 3;
      for (let i = 0; i < 3; i++) {
        const offset = 12 + i * 14;
        ctx.beginPath();
        if (isVertical) { ctx.moveTo(d.x + offset, d.y); ctx.lineTo(d.x + offset, d.y + dh); }
        else             { ctx.moveTo(d.x, d.y + offset); ctx.lineTo(d.x + dw, d.y + offset); }
        ctx.stroke();
      }
    }
  });

  // Vignette overlay
  const grad = ctx.createRadialGradient(W/2, H/2, H*0.25, W/2, H/2, H*0.75);
  grad.addColorStop(0, 'rgba(0,0,0,0)');
  grad.addColorStop(1, 'rgba(0,0,0,0.45)');
  ctx.fillStyle = grad;
  ctx.fillRect(0, PF_TOP, W, H - PF_TOP);
}
```

- [ ] **Step 4.2 — Draw minimap**

```js
function drawMinimap() {
  // The 6 rooms in a linear chain: 0-1-2-3-4-5
  // Display as a 6×1 grid in top-right corner
  const cellW = 18, cellH = 12, gap = 2;
  const startX = W - 10 - (6 * (cellW + gap));
  const startY = 4;

  ROOM_GRAPH.forEach((room, i) => {
    const rx = startX + i * (cellW + gap);
    const ry = startY;

    if (!room.visited) {
      ctx.fillStyle = '#222';
    } else if (room.type === 'garden') {
      ctx.fillStyle = '#2a6a3a';
    } else if (room.cleared) {
      ctx.fillStyle = '#556655';
    } else {
      ctx.fillStyle = '#886644';
    }
    ctx.fillRect(rx, ry, cellW, cellH);

    // Current room highlight
    if (i === currentRoomIdx) {
      ctx.strokeStyle = '#e8c547';
      ctx.lineWidth = 2;
      ctx.strokeRect(rx, ry, cellW, cellH);
    }

    // Garden icon
    if (room.type === 'garden' && room.visited) {
      ctx.font = '9px serif';
      ctx.textAlign = 'center';
      ctx.fillText('🎂', rx + cellW / 2, ry + cellH - 1);
      ctx.textAlign = 'left';
    }
  });
}
```

Add `drawMinimap()` call at the end of `drawHUD()`.

- [ ] **Step 4.3 — Implement room door trigger and player entry positioning**

```js
function checkDoorTrigger() {
  if (!currentRoom || !currentRoom.cleared) return;
  const margin = 12;

  const dirs = { north: 0, south: 0, east: 0, west: 0 };
  if (player.y < PF_TOP + TILE + margin && player.x > W/2 - 30 && player.x < W/2 + 30)
    dirs.north = true;
  if (player.y > H - TILE - margin && player.x > W/2 - 30 && player.x < W/2 + 30)
    dirs.south = true;
  if (player.x > W - TILE - margin && player.y > H/2 - 30 && player.y < H/2 + 30)
    dirs.east = true;
  if (player.x < TILE + margin && player.y > H/2 - 30 && player.y < H/2 + 30)
    dirs.west = true;

  const dirMap = { north: 'south', south: 'north', east: 'west', west: 'east' };
  const entryPos = {
    north: { x: W/2, y: H - TILE * 2 },
    south: { x: W/2, y: PF_TOP + TILE * 2 },
    east:  { x: TILE * 2, y: H/2 },
    west:  { x: W - TILE * 2, y: H/2 },
  };

  for (const [dir, triggered] of Object.entries(dirs)) {
    if (!triggered) continue;
    const targetIdx = currentRoom.doors[dir];
    if (targetIdx === null) continue;

    // Store entry direction for player placement
    const entry = dirMap[dir];
    transition.entryDir = entry;
    transition.entryPos = entryPos[entry];
    startTransition(targetIdx);
    return;
  }
}

function startTransition(targetIdx) {
  transition.active = true;
  transition.progress = 0;
  transition.fromIdx = currentRoomIdx;
  transition.toIdx = targetIdx;
  // Slide direction: east door → slide left; west door → slide right
  transition.dir = targetIdx > currentRoomIdx ? -1 : 1;
  gameState = 'TRANSITION';
}
```

- [ ] **Step 4.4 — Implement slide transition animation**

```js
function updateTransition() {
  transition.progress++;
  if (transition.progress >= transition.duration) {
    transition.active = false;
    loadRoom(transition.toIdx);
    if (transition.entryPos) {
      player.x = transition.entryPos.x;
      player.y = transition.entryPos.y;
    }
    // Boss intro
    if (ROOM_GRAPH[transition.toIdx].type === 'boss') {
      gameState = 'BOSS_INTRO';
      bossIntro.timer = 0;
    } else {
      gameState = 'PLAYING';
    }
  }
}

function drawTransition() {
  const t = transition.progress / transition.duration;
  const offset = t * W * transition.dir;

  // From room (sliding out)
  ctx.save();
  ctx.translate(offset, 0);
  const fromRoom = ROOM_GRAPH[transition.fromIdx];
  if (fromRoom) drawRoom(fromRoom);
  ctx.restore();

  // To room (sliding in)
  ctx.save();
  ctx.translate(offset + W * -transition.dir, 0);
  const toRoom = ROOM_GRAPH[transition.toIdx];
  if (toRoom) drawRoom(toRoom);
  ctx.restore();
}
```

Wire into update() and render():
```js
// In update():
  if (gameState === 'TRANSITION') { updateTransition(); return; }

// In render(), replace the plain background fill with:
  if (gameState === 'TRANSITION') {
    drawTransition();
    ctx.restore(); return;
  }
  // ...existing PLAYING render...
  if (currentRoom) drawRoom(currentRoom);
```

Also add door trigger check at end of the PLAYING update block:
```js
    checkDoorTrigger();
```

- [ ] **Step 4.5 — Implement skill pickup trigger and door unlock on room clear**

Update `checkRoomClear()`:
```js
function checkRoomClear() {
  if (enemies.length > 0 || boss !== null) return;
  if (currentRoom.cleared) return;
  currentRoom.cleared = true;

  // Pull 2 skills from pool
  if (skillPool.length >= 2) {
    skillPickupState.choices = [skillPool.pop(), skillPool.pop()];
    skillPickupState.selected = 0;
    gameState = 'SKILL_PICKUP';
  }
  // If pool exhausted (shouldn't happen), just unlock
}
```

- [ ] **Step 4.6 — Implement SKILL_PICKUP state: update + draw**

```js
function updateSkillPickup() {
  // Navigation
  if (keysJustPressed['ArrowLeft'])  skillPickupState.selected = 0;
  if (keysJustPressed['ArrowRight']) skillPickupState.selected = 1;
  if (keysJustPressed['e'] || keysJustPressed['E'] || keysJustPressed['Enter']) {
    applySkill(skillPickupState.choices[skillPickupState.selected]);
    skillPickupState.choices = [];
    gameState = 'PLAYING';
  }
}

// keysJustPressed: reset each frame. Add this to STATE:
const keysJustPressed = {};
// In keydown listener, also set: keysJustPressed[e.key] = true;
// At start of update(), clear it: Object.keys(keysJustPressed).forEach(k => delete keysJustPressed[k]);

const SKILL_META = {
  homing:    { icon: '🎯', name: 'HOMING TEARS',    desc: 'Tears curve toward nearest enemy' },
  explosive: { icon: '💥', name: 'EXPLOSIVE TEARS', desc: 'Burst into 4 shards on impact' },
  piercing:  { icon: '👻', name: 'PIERCING TEARS',  desc: 'Pass through all enemies' },
  orbital:   { icon: '🛡️', name: 'ORBITAL SHIELD', desc: '2 orbs orbit you, blocking hits' },
  freeze:    { icon: '❄️', name: 'FREEZE TEARS',    desc: 'Hits slow enemies by 60%' },
  boomerang: { icon: '🔄', name: 'BOOMERANG TEARS', desc: 'Tears return and can hit twice' },
  triple:    { icon: '🌀', name: 'TRIPLE SHOT',     desc: 'Fire 3 tears in a spread' },
  speed:     { icon: '⚡',       name: 'SPEED TEARS',     desc: 'Tears move 80% faster' },
};

function drawSkillPickup() {
  // Dim background
  ctx.fillStyle = 'rgba(0,10,0,0.75)';
  ctx.fillRect(0, 0, W, H);

  // Header
  ctx.fillStyle = '#e8c547';
  ctx.font = px(10);
  ctx.textAlign = 'center';
  ctx.fillText('❖ UPGRADE ❖', W / 2, 140);
  ctx.fillStyle = '#888';
  ctx.font = px(6);
  ctx.fillText('Room cleared! Choose a power-up.', W / 2, 168);

  // Two cards
  skillPickupState.choices.forEach((skillId, i) => {
    const meta = SKILL_META[skillId];
    const cardW = 200, cardH = 160;
    const cx = W/2 + (i === 0 ? -cardW/2 - 16 : cardW/2 + 16);
    const cy = H/2;
    const selected = skillPickupState.selected === i;

    ctx.fillStyle = selected ? '#0a200a' : '#0d1a0d';
    ctx.strokeStyle = selected ? '#e8c547' : '#3a5a3a';
    ctx.lineWidth = selected ? 3 : 1;
    roundRect(ctx, cx - cardW/2, cy - cardH/2, cardW, cardH, 8);
    ctx.fill(); ctx.stroke();

    // Icon
    ctx.font = '32px serif';
    ctx.fillText(meta.icon, cx, cy - 40);

    // Name
    ctx.fillStyle = selected ? '#e8c547' : '#a0c87a';
    ctx.font = px(7);
    ctx.fillText(meta.name, cx, cy - 5);

    // Desc
    ctx.fillStyle = '#888';
    ctx.font = px(5);
    wrapText(ctx, meta.desc, cx, cy + 20, cardW - 20, 14);
  });

  ctx.font = px(5);
  ctx.fillStyle = '#555';
  ctx.fillText('[<- ->] navigate   [E] pick', W / 2, H - 80);
  ctx.textAlign = 'left';
}

// Helper: rounded rect path
function roundRect(ctx, x, y, w, h, r) {
  ctx.beginPath();
  ctx.moveTo(x + r, y);
  ctx.lineTo(x + w - r, y); ctx.arcTo(x + w, y, x + w, y + r, r);
  ctx.lineTo(x + w, y + h - r); ctx.arcTo(x + w, y + h, x + w - r, y + h, r);
  ctx.lineTo(x + r, y + h); ctx.arcTo(x, y + h, x, y + h - r, r);
  ctx.lineTo(x, y + r); ctx.arcTo(x, y, x + r, y, r);
  ctx.closePath();
}

// Helper: wrap text
function wrapText(ctx, text, cx, y, maxW, lineH) {
  const words = text.split(' ');
  let line = '';
  words.forEach((w, i) => {
    const test = line + w + ' ';
    if (ctx.measureText(test).width > maxW && i > 0) {
      ctx.fillText(line.trim(), cx, y); y += lineH; line = w + ' ';
    } else { line = test; }
  });
  ctx.fillText(line.trim(), cx, y);
}
```

Wire into update() and render():
```js
// update():
  if (gameState === 'SKILL_PICKUP') { updateSkillPickup(); return; }

// render():
  if (gameState === 'SKILL_PICKUP') {
    if (currentRoom) drawRoom(currentRoom);
    enemies.forEach(drawEnemy);
    drawPlayer();
    drawHUD();
    drawSkillPickup();
    ctx.restore(); return;
  }
```

- [ ] **Step 4.7 — Verify in browser**

Enter game. Verify:
- Start room renders with tiles, walls, vignette
- East door is open (start room is pre-cleared)
- Walking to east door slides to Fight Room 1
- Minimap shows 2 visited rooms
- Killing all grubs triggers skill pickup overlay
- ← → navigate cards, E selects skill
- Doors open after skill picked
- No console errors

- [ ] **Step 4.8 — Commit**

```bash
git add index.html
git commit -m "feat: room system, door transitions, minimap, skill pickup UI"
```

---

## Task 5: Bat (Fly) + Stone Golem (Tank) enemies, room configs, death juice

> **Display names:** The `e.type` values remain `'fly'` and `'tank'` in code (changing these would break room spawn configs in ROOM_GRAPH). Display/lore names are **Bat** (fly) and **Stone Golem** (tank).

**Files:**
- Modify: `index.html` (ENEMIES section)

- [ ] **Step 5.1 — Implement updateFly() with orbit → telegraph → dash AI**

```js
function updateFly(e) {
  const slowMult = e.slowTimer > 0 ? 0.4 : 1;
  if (e.slowTimer > 0) e.slowTimer--;
  if (e.hitFlash > 0) e.hitFlash--;

  if (e.phase === 'orbit') {
    e.orbitAngle += 0.025 * slowMult;
    const targetX = player.x + Math.cos(e.orbitAngle) * e.orbitRadius;
    const targetY = player.y + Math.sin(e.orbitAngle) * e.orbitRadius;
    const dx = targetX - e.x, dy = targetY - e.y;
    const len = Math.hypot(dx, dy) || 1;
    e.x += (dx / len) * 2.5 * slowMult;
    e.y += (dy / len) * 2.5 * slowMult;
    clampToRoom(e, 10);

    // Begin telegraph after 90–150 frames of orbit
    if (!e.orbitFrames) e.orbitFrames = 90 + Math.floor(Math.random() * 60);
    e.orbitFrames--;
    if (e.orbitFrames <= 0) {
      e.phase = 'telegraph';
      e.telegraphTimer = 40; // ~0.67s warning
    }
  } else if (e.phase === 'telegraph') {
    e.telegraphTimer--;
    if (e.telegraphTimer <= 0) {
      // Lock in dash direction toward player
      const dx = player.x - e.x, dy = player.y - e.y;
      const len = Math.hypot(dx, dy) || 1;
      e.dashVx = (dx / len) * 6;
      e.dashVy = (dy / len) * 6;
      e.dashTimer = 24;
      e.phase = 'dash';
    }
  } else if (e.phase === 'dash') {
    e.x += e.dashVx * slowMult;
    e.y += e.dashVy * slowMult;
    e.dashTimer--;
    clampToRoom(e, 10);
    if (e.dashTimer <= 0) {
      e.phase = 'orbit';
      e.orbitFrames = 90 + Math.floor(Math.random() * 60);
    }
  }
}
```

- [ ] **Step 5.2 — Implement drawFly()**

```js
function drawFly(e) {
  const flutter = Math.sin(frameCount * 0.3) * 3; // wing flutter
  const glow = e.phase === 'telegraph' ? Math.abs(Math.sin(frameCount * 0.2)) : 0;

  ctx.save();
  ctx.translate(e.x, e.y + flutter * 0.3);

  // Telegraph glow
  if (glow > 0) {
    ctx.shadowColor = '#ff4444';
    ctx.shadowBlur = 15 * glow;
  }

  // Wings
  const wingAlpha = e.hitFlash > 0 ? 1 : 0.5;
  ctx.globalAlpha = wingAlpha;
  ctx.fillStyle = e.slowTimer > 0 ? 'rgba(136,200,255,0.5)' : 'rgba(150,180,255,0.5)';
  ctx.beginPath(); ctx.ellipse(-14, -2 + flutter, 14, 7, -0.3, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath(); ctx.ellipse(14, -2 + flutter, 14, 7, 0.3, 0, Math.PI * 2); ctx.fill();
  ctx.globalAlpha = 1;

  // Body
  ctx.fillStyle = e.hitFlash > 0 ? '#fff' : (e.slowTimer > 0 ? '#88aacc' : '#3a3a6a');
  ctx.beginPath(); ctx.ellipse(0, 4, 8, 11, 0, 0, Math.PI * 2); ctx.fill();
  ctx.fillStyle = e.hitFlash > 0 ? '#fff' : (e.slowTimer > 0 ? '#aaccee' : '#5a5a9a');
  ctx.beginPath(); ctx.ellipse(0, -2, 10, 8, 0, 0, Math.PI * 2); ctx.fill();

  // Red compound eyes
  ctx.fillStyle = '#ff4444';
  ctx.beginPath(); ctx.arc(-5, -4, 4, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath(); ctx.arc(5, -4, 4, 0, Math.PI * 2); ctx.fill();
  ctx.fillStyle = '#cc0000';
  ctx.beginPath(); ctx.arc(-5, -4, 2.5, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath(); ctx.arc(5, -4, 2.5, 0, Math.PI * 2); ctx.fill();

  ctx.restore();
}
```

- [ ] **Step 5.3 — Implement updateTank() and drawTank()**

```js
function updateTank(e) {
  const speed = 0.7 * (e.slowTimer > 0 ? 0.4 : 1);
  if (e.slowTimer > 0) e.slowTimer--;
  if (e.hitFlash > 0) e.hitFlash--;

  const dx = player.x - e.x, dy = player.y - e.y;
  const len = Math.hypot(dx, dy) || 1;

  e.knockVx *= 0.8; e.knockVy *= 0.8;
  e.x += (dx / len) * speed + e.knockVx;
  e.y += (dy / len) * speed + e.knockVy;
  // Tank ignores obstacles per spec
  clampToRoom(e, 22);

  e.pulseTimer = (e.pulseTimer + 1) % 60;
}

function drawTank(e) {
  const pulse = 1 + Math.sin(e.pulseTimer / 60 * Math.PI * 2) * 0.04;
  ctx.save();
  ctx.translate(e.x, e.y);
  ctx.scale(pulse, pulse);

  const bodyColor = e.hitFlash > 0 ? '#fff' : (e.slowTimer > 0 ? '#88aacc' : '#6a1a1a');
  const highColor  = e.hitFlash > 0 ? '#fff' : (e.slowTimer > 0 ? '#aaccee' : '#8a2a2a');

  // Shell
  ctx.fillStyle = bodyColor;
  ctx.beginPath(); ctx.ellipse(0, 4, 24, 19, 0, 0, Math.PI * 2); ctx.fill();
  ctx.fillStyle = highColor;
  ctx.beginPath(); ctx.ellipse(0, 3, 20, 16, 0, 0, Math.PI * 2); ctx.fill();

  // Shell lines
  ctx.strokeStyle = bodyColor; ctx.lineWidth = 1.5;
  ctx.beginPath(); ctx.moveTo(0, -12); ctx.lineTo(0, 22); ctx.stroke();
  ctx.beginPath(); ctx.moveTo(-22, 4); ctx.bezierCurveTo(-10,-3, 10,-3, 22, 4); ctx.stroke();

  // HP pips on back
  for (let i = 0; i < 3; i++) {
    ctx.fillStyle = i < e.hp ? '#ff6060' : '#3a1010';
    ctx.beginPath(); ctx.arc(-8 + i * 8, 6, 4, 0, Math.PI * 2); ctx.fill();
  }

  // Head
  ctx.fillStyle = e.hitFlash > 0 ? '#fff' : '#7a2020';
  ctx.beginPath(); ctx.arc(0, -14, 10, 0, Math.PI * 2); ctx.fill();

  // Eyes
  ctx.fillStyle = '#fff';
  ctx.beginPath(); ctx.arc(-5, -16, 3.5, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath(); ctx.arc(5, -16, 3.5, 0, Math.PI * 2); ctx.fill();
  ctx.fillStyle = '#000';
  ctx.beginPath(); ctx.arc(-4, -16, 2, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath(); ctx.arc(6, -16, 2, 0, Math.PI * 2); ctx.fill();

  ctx.restore();
}
```

- [ ] **Step 5.4 — Wire fly and tank into the update loop**

In the enemy update block in `update()`:
```js
      if (e.type === 'grub') updateGrub(e);
      else if (e.type === 'fly')  updateFly(e);
      else if (e.type === 'tank') updateTank(e);
```

Update `killEnemy()` to handle fly collision radius (10px vs 14px for grub/tank):
```js
function enemyRadius(e) { return e.type === 'fly' ? 10 : e.type === 'tank' ? 22 : 14; }
```

Update all collision checks to use `enemyRadius(e)`.

- [ ] **Step 5.5 — Verify in browser**

Navigate to Fight Room 1 (east door from start). Verify:
- Grubs hop and chase
- Navigate to Fight Room 2: Bats (fly type) orbit then dash with red telegraph glow
- Fight Room 3: Stone Golem (tank type) lumbers with HP pips on back, pips go dark on hit, big explosion on death
- Correct enemy counts in each room
- No console errors

- [ ] **Step 5.6 — Commit**

```bash
git add index.html
git commit -m "feat: Bat (fly) and Stone Golem (tank) enemies, death juice, all 3 enemy types complete"
```

---

## Task 6: All 8 mechanical skills + orbital shield

**Files:**
- Modify: `index.html` (SKILLS section)

- [ ] **Step 6.1 — Add the SKILLS section with applySkill()**

```js
// ─── SKILLS ───────────────────────────────────────────────────────────────
function applySkill(skillId) {
  player.skills[skillId] = true;
  spawnParticles(player.x, player.y, '#e8c547', 20, 5, 30);

  if (skillId === 'orbital') {
    player.orbitals = [
      { angle: 0,        speed: 0.04 },
      { angle: Math.PI,  speed: 0.04 },
    ];
  }
}

function updateOrbitals() {
  if (!player.skills.orbital) return;
  const r = 50;
  player.orbitals.forEach(orb => {
    orb.angle += orb.speed;
    const ox = player.x + Math.cos(orb.angle) * r;
    const oy = player.y + Math.sin(orb.angle) * r;
    orb.x = ox; orb.y = oy;

    // Check vs enemies
    enemies.forEach((e, ei) => {
      if (Math.hypot(ox - e.x, oy - e.y) < enemyRadius(e) + 8) {
        if (!e.orbitalCooldown || e.orbitalCooldown <= 0) {
          damageEnemy(e, 0, 0, ei);
          e.orbitalCooldown = 30; // 0.5s cooldown per orb hit
        }
      }
      if (e.orbitalCooldown > 0) e.orbitalCooldown--;
    });
  });
}

function drawOrbitals() {
  if (!player.skills.orbital) return;
  player.orbitals.forEach(orb => {
    ctx.save();
    ctx.strokeStyle = '#e8c547';
    ctx.lineWidth = 2;
    ctx.shadowColor = '#e8c547';
    ctx.shadowBlur = 8;
    ctx.beginPath(); ctx.arc(orb.x, orb.y, 8, 0, Math.PI * 2); ctx.stroke();
    ctx.fillStyle = 'rgba(232,197,71,0.4)';
    ctx.fill();
    ctx.restore();
  });
}
```

Wire into update():
```js
// In PLAYING update block, after updatePlayer():
    updateOrbitals();
```

Wire into render(), after drawPlayer():
```js
    drawOrbitals();
```

- [ ] **Step 6.2 — Verify all 8 skills in browser**

Manually test each by temporarily hardcoding `applySkill('homing')` etc. in `initGame()`. Verify:
- **Homing:** tears curve toward enemies
- **Explosive:** tears burst into 4 on impact
- **Piercing:** tears pass through multiple enemies
- **Orbital:** two golden orbs orbit, damage enemies on contact
- **Freeze:** enemies slow down + turn blue on hit
- **Boomerang:** tears reverse at range end, can hit twice
- **Triple:** 3 tears fire in spread
- **Speed:** tears fly noticeably faster
- **Stacking:** enable homing + explosive together — shards should also home

Remove the debug `applySkill` calls after testing.

- [ ] **Step 6.3 — Commit**

```bash
git add index.html
git commit -m "feat: all 8 mechanical skills implemented and stackable"
```

---

## Task 7: Bloom boss (Giant Slime)

**Files:**
- Modify: `index.html` (BOSS section, LOOP)

- [ ] **Step 7.0 — Add Bloom sprite loading to ASSETS section**

Add to the ASSETS section (after player sprite load):

```js
const bloomSprites = {};
const bloomPaths = {
  crawl:  'sprites/Momo-Mama/Momo-Mama/momo mama/mm-crawl.png',
  attack: 'sprites/Momo-Mama/Momo-Mama/momo mama/mm-generic attack.png',
  jump:   'sprites/Momo-Mama/Momo-Mama/momo mama/mm-jump.png',
  hurt:   'sprites/Momo-Mama/Momo-Mama/momo mama/mm-hurt.png',
  happy:  'sprites/Momo-Mama/Momo-Mama/EXTRAS/mm-happy.png',
  heart:  'sprites/Momo-Mama/FX_Heart.png',
};
Object.entries(bloomPaths).forEach(([key, src]) => loadImage(src, img => { bloomSprites[key] = img; }));

// Frame counts per animation
const BLOOM_FRAMES = { crawl: 5, attack: 7, jump: 6, hurt: 6, happy: 5 };
const BLOOM_FRAME_SIZE = 64;
const BLOOM_DISPLAY_SIZE = 128; // 2× scale
```

Also add Dude Monster sprite loading:

```js
const dudeSprites = {};
const dudePaths = {
  idle: 'sprites/3 Dude_Monster/Dude_Monster_Idle_4.png',
  walk: 'sprites/3 Dude_Monster/Dude_Monster_Walk_6.png',
  hurt: 'sprites/3 Dude_Monster/Dude_Monster_Hurt_4.png',
};
const DUDE_FRAMES = { idle: 4, walk: 6, hurt: 4 };
const DUDE_FRAME_SIZE = 32;
const DUDE_DISPLAY_SIZE = 64; // 2× scale
Object.entries(dudePaths).forEach(([key, src]) => loadImage(src, img => { dudeSprites[key] = img; }));
```

- [ ] **Step 7.1 — Add BOSS section: initBoss() and boss state**

```js
// ─── BOSS ─────────────────────────────────────────────────────────────────
function initBoss() {
  boss = {
    x: W / 2, y: 220,
    hp: 24, maxHP: 24,
    phase: 1,
    speed: 0.9,
    jumpTimer: 180,       // 3s before first jump
    telegraphTimer: 0,
    telegraphing: false,
    shadowX: 0, shadowY: 0, // where jump will land
    jumpInFlight: false,
    jumpT: 0,             // 0→1 jump arc progress
    startX: 0, startY: 0, // jump start position
    hitFlash: 0,
    slowTimer: 0,
    spawnedMinions: false,
    knockVx: 0, knockVy: 0,
    projectiles: [],
    animFrame: 0,
    animTimer: 0,
  };
}
```

- [ ] **Step 7.2 — Implement updateBloom()**

```js
function updateBloom() {
  if (!boss) return;
  if (boss.hitFlash > 0) boss.hitFlash--;
  if (boss.slowTimer > 0) boss.slowTimer--;

  // Animate sprite frames
  boss.animTimer++;
  if (boss.animTimer >= 8) { boss.animTimer = 0; boss.animFrame++; }

  if (boss.jumpInFlight) {
    // Arc jump toward shadow position
    boss.jumpT += 0.06;
    boss.x = boss.startX + (boss.shadowX - boss.startX) * boss.jumpT;
    boss.y = boss.startY + (boss.shadowY - boss.startY) * boss.jumpT
             - Math.sin(boss.jumpT * Math.PI) * 80; // arc height
    if (boss.jumpT >= 1) {
      boss.jumpInFlight = false;
      boss.x = boss.shadowX; boss.y = boss.shadowY;
      startShake(boss.phase === 2 ? 12 : 8, boss.phase === 2 ? 20 : 14);
      spawnParticles(boss.x, boss.y, '#aa44ee', 20, 8, 40);
      spawnParticles(boss.x, boss.y, '#44ee88', 15, 6, 35);
      if (boss.phase === 2) {
        // Phase 2: aimed blob on landing
        const dx = player.x - boss.x, dy = player.y - boss.y;
        const len = Math.hypot(dx, dy) || 1;
        boss.projectiles.push({ x: boss.x, y: boss.y,
          vx: (dx/len)*3.5, vy: (dy/len)*3.5, life: 150, color: '#aa44ee' });
      }
    }
    return; // no other movement during jump
  }

  // Crawl toward player
  const speed = boss.speed * (boss.slowTimer > 0 ? 0.4 : 1) * (boss.phase === 2 ? 1.55 : 1);
  const dx = player.x - boss.x, dy = player.y - boss.y;
  const len = Math.hypot(dx, dy) || 1;
  boss.knockVx *= 0.85; boss.knockVy *= 0.85;
  boss.x += (dx/len)*speed + boss.knockVx;
  boss.y += (dy/len)*speed + boss.knockVy;
  clampToRoom(boss, 40);

  // Phase transition
  if (boss.phase === 1 && boss.hp <= 12) {
    boss.phase = 2;
    boss.animFrame = 0;
    startShake(12, 20);
    if (!boss.spawnedMinions) {
      boss.spawnedMinions = true;
      spawnEnemy('grub', boss.x - 120, boss.y + 80);
      spawnEnemy('grub', boss.x + 120, boss.y + 80);
      spawnEnemy('grub', boss.x, boss.y + 120);
    }
  }

  // Jump telegraph → jump
  boss.jumpTimer--;
  if (!boss.telegraphing && boss.jumpTimer <= 60) {
    boss.telegraphing = true;
    boss.telegraphTimer = 60;
    boss.shadowX = player.x; boss.shadowY = player.y;
  }
  if (boss.telegraphing) {
    boss.telegraphTimer--;
    if (boss.telegraphTimer <= 0) {
      boss.telegraphing = false;
      boss.jumpInFlight = true;
      boss.jumpT = 0;
      boss.startX = boss.x; boss.startY = boss.y;
      boss.jumpTimer = boss.phase === 2 ? 120 : 180;
      boss.animFrame = 0;
    }
  }

  // Update projectiles
  for (let i = boss.projectiles.length - 1; i >= 0; i--) {
    const p = boss.projectiles[i];
    p.x += p.vx; p.y += p.vy; p.life--;
    if (p.life <= 0 || p.x < 0 || p.x > W || p.y < PF_TOP || p.y > H) {
      boss.projectiles.splice(i, 1); continue;
    }
    if (player.iframes <= 0 && Math.hypot(p.x - player.x, p.y - player.y) < 12 + 16) {
      player.hp--; player.iframes = CONFIG.player.iframeDuration;
      startShake(4, 8); boss.projectiles.splice(i, 1);
      if (player.hp <= 0) gameState = 'TITLE';
    }
  }
}
```

- [ ] **Step 7.3 — Implement drawBloom()**

```js
function drawBloom() {
  if (!boss) return;

  // Telegraph shadow on ground
  if (boss.telegraphing) {
    const pulse = 0.5 + 0.5 * Math.abs(Math.sin(frameCount * 0.15));
    ctx.save();
    ctx.globalAlpha = 0.5 * pulse;
    ctx.fillStyle = '#220022';
    ctx.beginPath(); ctx.ellipse(boss.shadowX, boss.shadowY, 40, 20, 0, 0, Math.PI * 2); ctx.fill();
    ctx.restore();
  }

  ctx.save();
  ctx.translate(boss.x, boss.y);

  // Pick animation based on state
  let sheet = null, frames = 1;
  if (boss.hitFlash > 0 && bloomSprites.hurt) { sheet = bloomSprites.hurt; frames = BLOOM_FRAMES.hurt; }
  else if (boss.jumpInFlight && bloomSprites.jump) { sheet = bloomSprites.jump; frames = BLOOM_FRAMES.jump; }
  else if (boss.phase === 2 && bloomSprites.attack) { sheet = bloomSprites.attack; frames = BLOOM_FRAMES.attack; }
  else if (bloomSprites.crawl) { sheet = bloomSprites.crawl; frames = BLOOM_FRAMES.crawl; }

  const frame = Math.floor(boss.animFrame % frames);
  const half = BLOOM_DISPLAY_SIZE / 2;

  if (sheet && sheet.naturalWidth > 0) {
    ctx.drawImage(sheet, frame * BLOOM_FRAME_SIZE, 0, BLOOM_FRAME_SIZE, BLOOM_FRAME_SIZE,
                  -half, -half, BLOOM_DISPLAY_SIZE, BLOOM_DISPLAY_SIZE);
  } else {
    // Canvas fallback blob
    const wobble = Math.sin(frameCount * 0.1) * 4;
    ctx.fillStyle = boss.phase === 2 ? '#882299' : '#aa44ee';
    ctx.beginPath(); ctx.ellipse(0, wobble/2, 38+wobble, 35-wobble/2, 0, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = '#fff';
    ctx.beginPath(); ctx.arc(-13,-5,9,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(13,-5,9,0,Math.PI*2); ctx.fill();
    ctx.fillStyle = boss.phase===2 ? '#ff0000':'#222';
    ctx.beginPath(); ctx.arc(-11,-5,5,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(15,-5,5,0,Math.PI*2); ctx.fill();
  }
  ctx.restore();

  // Projectiles
  boss.projectiles.forEach(p => {
    ctx.save(); ctx.fillStyle = p.color||'#aa44ee';
    ctx.shadowColor = p.color; ctx.shadowBlur = 6;
    ctx.beginPath(); ctx.arc(p.x,p.y,7,0,Math.PI*2); ctx.fill(); ctx.restore();
  });
}

function drawBossHPBar() {
  if (!boss) return;
  const barW = 400, barH = 14;
  const bx = W / 2 - barW / 2, by = H - 30;
  ctx.fillStyle = '#1a0a0a';
  ctx.fillRect(bx - 2, by - 2, barW + 4, barH + 4);
  ctx.fillStyle = '#660000';
  ctx.fillRect(bx, by, barW, barH);
  const hpW = (boss.hp / boss.maxHP) * barW;
  ctx.fillStyle = boss.hp > 12 ? '#cc3333' : '#ff6666';
  ctx.fillRect(bx, by, hpW, barH);
  ctx.fillStyle = '#e8c547';
  ctx.font = px(5);
  ctx.textAlign = 'center';
  ctx.fillText('BLOOM', W / 2, by - 6);
  ctx.textAlign = 'left';
}
```

- [ ] **Step 7.4 — Tear-boss collision + boss death**

In `checkTearEnemyCollisions()`, after the enemy loop, add:
```js
  // Boss collision
  if (boss && boss.hp > 0) {
    for (let ti = tears.length - 1; ti >= 0; ti--) {
      const t = tears[ti];
      if (t.hitEnemies.has('boss')) continue;
      if (Math.hypot(t.x - boss.x, t.y - boss.y) < t.radius + 40) {
        if (player.skills.freeze) boss.slowTimer = 120;
        boss.hp--;
        boss.hitFlash = 6;
        const len = Math.hypot(t.vx, t.vy) || 1;
        boss.knockVx = (t.vx / len) * 2;
        boss.knockVy = (t.vy / len) * 2;
        if (player.skills.explosive) {
          [[1,0],[-1,0],[0,1],[0,-1]].forEach(([sx,sy]) => {
            tears.push({ x: t.x, y: t.y, vx: sx*5, vy: sy*5, life: 18,
                         radius: t.radius, hitEnemies: new Set(['boss']), returning: false });
          });
        }
        impactParticles(t.x, t.y, '#ffaaaa');
        if (!player.skills.piercing) {
          tears.splice(ti, 1);
        } else {
          t.hitEnemies.add('boss');
        }
        if (boss.hp <= 0) killBoss();
        break;
      }
    }
  }
```

Implement `killBoss()` — after setting `boss = null`, activate mini Bloom:
```js
function killBoss() {
  spawnParticles(boss.x, boss.y, '#aa44ee', 40, 8, 60);
  spawnParticles(boss.x, boss.y, '#44ee88', 20, 6, 50);
  spawnParticles(boss.x, boss.y, '#ffffff', 15, 5, 45);
  startShake(15, 30);
  pet.x = boss.x; pet.y = boss.y;
  pet.active = true; pet.befriended = false;
  boss = null;
  checkRoomClear();
}
```

Add `updatePet()` function:
```js
function updatePet() {
  if (!pet.active) return;
  pet.bounceTimer++;
  if (!pet.befriended) return; // stays at boss death position until befriended
  // Follow player with lag
  const dx = player.x - pet.x, dy = player.y - pet.y;
  const dist = Math.hypot(dx, dy);
  const targetDist = 60;
  if (dist > targetDist + 5) {
    const speed = Math.min((dist - targetDist) * 0.1, 4);
    pet.x += (dx / dist) * speed;
    pet.y += (dy / dist) * speed;
  }
}
```

Add `drawPet()` function:
```js
function drawPet() {
  if (!pet.active) return;
  const bounce = Math.sin(pet.bounceTimer * 0.08) * 3;
  ctx.save();
  ctx.translate(pet.x, pet.y + bounce);

  const sheet = bloomSprites.happy;
  const frame = Math.floor(pet.bounceTimer / 10) % BLOOM_FRAMES.happy;
  const petSize = 48; // 0.75× display size

  if (sheet && sheet.naturalWidth > 0) {
    ctx.drawImage(sheet, frame * BLOOM_FRAME_SIZE, 0, BLOOM_FRAME_SIZE, BLOOM_FRAME_SIZE,
                  -petSize/2, -petSize/2, petSize, petSize);
  } else {
    // Fallback tiny blob
    ctx.fillStyle = '#cc66ff';
    ctx.beginPath(); ctx.ellipse(0,0,16,13,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#fff'; ctx.beginPath(); ctx.arc(-6,-2,4,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(6,-2,4,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#220033'; ctx.beginPath(); ctx.arc(-5,-2,2.5,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(7,-2,2.5,0,Math.PI*2); ctx.fill();
  }

  if (!pet.befriended && Math.hypot(player.x-pet.x, player.y-pet.y) < 48) {
    ctx.fillStyle='#fff'; ctx.font='9px "Press Start 2P",monospace';
    ctx.textAlign='center'; ctx.fillText('[E] befriend',0,-32); ctx.textAlign='left';
  }
  ctx.restore();
}
```

Add E-key befriend trigger in `tryInteract()` (Task 9):
```js
// In tryInteract(), add:
if (pet.active && !pet.befriended && Math.hypot(player.x - pet.x, player.y - pet.y) < 48) {
  pet.befriended = true;
  spawnParticles(pet.x, pet.y, '#cc66ff', 15, 4, 30);
}
```

Wire `updatePet()` into the PLAYING update block (after `updateOrbitals()`), and `drawPet()` into render (after `drawPlayer()`).

Add `pet` reset in `initGame()`:
```js
pet.active = false; pet.befriended = false;
```

- [ ] **Step 7.5 — BOSS_INTRO state**

```js
function updateBossIntro() {
  bossIntro.timer++;
  if (bossIntro.timer >= bossIntro.duration) gameState = 'PLAYING';
}

function drawBossIntro() {
  ctx.fillStyle = 'rgba(0,0,0,0.8)';
  ctx.fillRect(0, 0, W, H);
  const alpha = Math.min(1, bossIntro.timer / 20);
  ctx.globalAlpha = alpha;
  ctx.fillStyle = '#cc3333';
  ctx.font = px(9);
  ctx.textAlign = 'center';
  ctx.fillText('BLOOM', W / 2, H / 2 - 20);
  ctx.fillStyle = '#888';
  ctx.font = px(6);
  ctx.fillText('She\'s here to protect her slimes...', W / 2, H / 2 + 20);
  ctx.globalAlpha = 1;
  ctx.textAlign = 'left';
}
```

Wire into update() and render():
```js
// update():
  if (gameState === 'BOSS_INTRO') { updateBossIntro(); return; }

// render():
  if (gameState === 'BOSS_INTRO') { drawRoom(currentRoom); drawBossIntro(); ctx.restore(); return; }
```

Wire boss into PLAYING update and render:
```js
// update() PLAYING block:
    if (boss) updateBloom();

// render() PLAYING block, after enemies:
    if (boss) drawBloom();
    if (boss) drawBossHPBar();
```

- [ ] **Step 7.6 — Verify in browser**

Navigate through 3 fight rooms and into boss room. Verify:
- Boss intro flash shows "BLOOM" then fades to gameplay
- Bloom crawls toward player with wobble animation
- Jump telegraph shadow appears on player position, then boss leaps and lands with AoE splash
- HP bar shows "BLOOM" across bottom
- Phase 2 triggers at ≤12 HP: speed increases, 3 grub minions spawn, aimed blob added on jump landing
- Killing boss spawns purple/green particles, triggers skill pickup
- No console errors

- [ ] **Step 7.7 — Commit**

```bash
git add index.html
git commit -m "feat: Bloom boss (giant slime), 2 phases, HP bar, jump attack, boss death"
```

---

## Task 8: Art polish — garden rooms, vignette, player sprite, font, title

> **Enemy display names and sprites:** `e.type === 'fly'` → display as **Bat**; `e.type === 'tank'` → display as **Stone Golem**. Type strings stay unchanged in code to avoid breaking room spawn configs.
>
> **Bat sprite files:** `sprites/fly/DarkFantasyEnemies_FREE/Bat/Bat with VFX/`
> - `Bat-IdleFly.png` (9 frames, 64×64px) — orbit phase animation
> - `Bat-Run.png` (8 frames, 64×64px) — dash phase animation
> - `Bat-Hurt.png` (5 frames) — hit flash animation
>
> **Stone Golem sprite files:** `sprites/mecha-stone-boss/Mecha-stone Golem 0.1/PNG sheet/Character_sheet.png`
> - 1000×1000px combined sheet, all animations packed, frames ~100×100px
> - Use row 0 (idle/moving, 4 frames) for normal movement

**Files:**
- Modify: `index.html` (DRAWING, ASSETS sections)

- [ ] **Step 8.1 — Add ASSETS section to load player sprite with fallback**

After CONFIG, before STATE:
```js
// ─── ASSETS ───────────────────────────────────────────────────────────────
function loadImage(src, onLoad) {
  const img = new Image();
  img.onload = () => onLoad(img);
  img.onerror = () => onLoad(null); // null = use placeholder
  img.src = src;
}

loadImage('img/character.png', img => { player.sprite = img; });
```

Player sprite sheets are in `sprites/3 Dude_Monster/`. Each frame is **32×32 pixels** in a horizontal sheet:
- `Dude_Monster_Idle_4.png` — 4 frames, used for idle
- `Dude_Monster_Walk_6.png` — 6 frames, used for movement
- `Dude_Monster_Hurt_4.png` — 4 frames, used during i-frames

Load via:
```js
loadImage('sprites/3 Dude_Monster/Dude_Monster_Idle_4.png', img => { player.spriteIdle = img; });
loadImage('sprites/3 Dude_Monster/Dude_Monster_Walk_6.png', img => { player.spriteWalk = img; });
loadImage('sprites/3 Dude_Monster/Dude_Monster_Hurt_4.png', img => { player.spriteHurt = img; });
```

Draw with 2× scale (32px source → 64px displayed):
```js
ctx.drawImage(sheet, frame * 32, 0, 32, 32, -32, -32, 64, 64)
```

The `drawPlayer()` function already checks `player.sprite` and falls back to the placeholder circle. When upgrading to sprite sheets, select the appropriate sheet based on movement state and i-frame status, then index into it by `animFrame % frameCount`.

- [ ] **Step 8.2 — Add flower and pebble decorations to drawRoom()**

After the tile loop in `drawRoom()`, before the door drawing:
```js
  // Decorations (deterministic positions based on room id)
  const seed = currentRoom.id * 17;
  const decorCount = 8;
  for (let i = 0; i < decorCount; i++) {
    const t = (seed + i * 37) % 100;
    const tx = 2 + (t * 11) % (COLS - 4);
    const ty = 2 + (t * 7)  % (ROWS - 4);
    if (currentRoom.obstacles.some(([ox, oy]) => ox === tx && oy === ty)) continue;
    const rx = tx * TILE + 20, ry = PF_TOP + ty * TILE + 20;

    if (i % 3 === 0) {
      // Flower
      ctx.fillStyle = i % 6 === 0 ? '#e8c547' : '#e05252';
      ctx.beginPath(); ctx.arc(rx, ry, 5, 0, Math.PI * 2); ctx.fill();
      ctx.fillStyle = '#4a9a4a';
      ctx.fillRect(rx - 1, ry + 4, 2, 8);
    } else {
      // Pebble
      ctx.fillStyle = '#5a5a4a';
      ctx.beginPath(); ctx.ellipse(rx, ry, 6, 4, 0.3, 0, Math.PI * 2); ctx.fill();
    }
  }
```

- [ ] **Step 8.3 — Garden room special art (no walls on inner area)**

In `drawRoom()`, add a special case for the garden type:
```js
  // Garden room: softer, brighter grass and flower carpet
  if (room.type === 'garden') {
    for (let ty = 1; ty < ROWS - 1; ty++) {
      for (let tx = 1; tx < COLS - 1; tx++) {
        const rx = tx * TILE, ry = PF_TOP + ty * TILE;
        const bright = (tx + ty) % 2 === 0 ? '#3a7a3a' : '#3e8440';
        ctx.fillStyle = bright;
        ctx.fillRect(rx, ry, TILE, TILE);
        // Grass tufts
        if ((tx * 3 + ty * 7) % 5 === 0) {
          ctx.strokeStyle = '#4a9a4a'; ctx.lineWidth = 1.5;
          [[6,8],[12,5],[18,9]].forEach(([gx, gy]) => {
            ctx.beginPath(); ctx.moveTo(rx + gx, ry + TILE - 2);
            ctx.lineTo(rx + gx - 2, ry + TILE - gy); ctx.stroke();
          });
        }
      }
    }
  }
```

Draw this early in `drawRoom()`, before the main tile loop, so normal tiles override it except in the garden section.

- [ ] **Step 8.4 — Draw garden props (boards + throne) in garden room**

```js
function drawGardenProps(room) {
  if (room.type !== 'garden') return;

  const drawBoard = (x, y, label, color) => {
    ctx.fillStyle = '#3a2010';
    ctx.fillRect(x - 30, y - 35, 60, 50);
    ctx.fillStyle = color;
    ctx.fillRect(x - 26, y - 31, 52, 42);
    ctx.fillStyle = '#e8c547';
    ctx.font = px(5);
    ctx.textAlign = 'center';
    ctx.fillText(label, x, y + 24);
    // Interact hint
    if (Math.hypot(player.x - x, player.y - y) < 50) {
      ctx.fillStyle = '#fff';
      ctx.font = px(5);
      ctx.fillText('[E]', x, y - 44);
    }
    ctx.textAlign = 'left';
  };

  drawBoard(room.photoBoard.x, room.photoBoard.y, 'PHOTOS', '#5a8a5a');
  drawBoard(room.videoBoard.x, room.videoBoard.y, 'VIDEOS', '#5a5a8a');

  // Throne
  const tx = room.throne.x, ty = room.throne.y;
  ctx.fillStyle = '#8b6914';
  ctx.fillRect(tx - 22, ty - 40, 44, 50);
  ctx.fillStyle = '#e8c547';
  ctx.fillRect(tx - 18, ty - 50, 36, 16);
  ctx.fillRect(tx - 18, ty - 38, 36, 36);
  ctx.fillStyle = '#aa8822';
  [[- 10, -46], [10, -46], [-10, -30], [10, -30]].forEach(([ox, oy]) => {
    ctx.beginPath(); ctx.arc(tx + ox, ty + oy, 4, 0, Math.PI * 2); ctx.fill();
  });
  if (Math.hypot(player.x - tx, player.y - ty) < 50) {
    ctx.fillStyle = '#fff'; ctx.font = px(5); ctx.textAlign = 'center';
    ctx.fillText('[E]', tx, ty - 60);
    ctx.textAlign = 'left';
  }
}
```

Call `drawGardenProps(currentRoom)` in render() after `drawRoom(currentRoom)`.

- [ ] **Step 8.5 — Polish title screen with background art**

Replace `drawTitle()`:
```js
function drawTitle() {
  // Animated grass background
  ctx.fillStyle = '#0d1f0d';
  ctx.fillRect(0, 0, W, H);
  for (let i = 0; i < 40; i++) {
    const x = ((i * 71) % W);
    const y = H - 60 - (i * 37) % 80;
    const sway = Math.sin(frameCount * 0.03 + i) * 3;
    ctx.strokeStyle = '#2a5a2a'; ctx.lineWidth = 2;
    ctx.beginPath(); ctx.moveTo(x + sway, y + 20); ctx.lineTo(x, y); ctx.stroke();
  }

  // Decorative frame
  ctx.strokeStyle = '#3a7a3a'; ctx.lineWidth = 3;
  ctx.strokeRect(30, 30, W - 60, H - 60);
  ctx.strokeStyle = '#2a5a2a'; ctx.lineWidth = 1;
  ctx.strokeRect(36, 36, W - 72, H - 72);

  // Title text
  ctx.fillStyle = '#e8c547';
  ctx.font = px(22);
  ctx.textAlign = 'center';
  ctx.shadowColor = '#e8c547'; ctx.shadowBlur = 15;
  ctx.fillText('NOPEYEP', W / 2, 170);
  ctx.shadowBlur = 0;
  ctx.fillStyle = '#a0e060';
  ctx.font = px(10);
  ctx.fillText('BIRTHDAY QUEST', W / 2, 210);

  ctx.fillStyle = '#7ab87a'; ctx.font = px(7);
  ctx.fillText(`a gift for ${CONFIG.friendName}`, W / 2, 258);

  ctx.fillStyle = '#4a7a4a'; ctx.font = px(6);
  ctx.fillText('WASD move   ARROWS shoot   E interact', W / 2, 330);
  ctx.fillText('clear rooms to earn power-ups', W / 2, 355);

  if (Math.floor(frameCount / 30) % 2 === 0) {
    ctx.fillStyle = '#e8c547'; ctx.font = px(8);
    ctx.shadowColor = '#e8c547'; ctx.shadowBlur = 10;
    ctx.fillText('PRESS ENTER', W / 2, 430);
    ctx.shadowBlur = 0;
  }
  ctx.textAlign = 'left';
}
```

- [ ] **Step 8.6 — Verify visual polish in browser**

Verify:
- Title screen has animated grass, glowing title
- Rooms have 2-tone grass, stone walls, decorations
- Garden room is brighter with boards and throne props
- [E] prompt shows when near interactables
- Player sprite loads from img/character.png if present, else shows placeholder

- [ ] **Step 8.7 — Commit**

```bash
git add index.html
git commit -m "feat: art polish, garden props, title screen, player sprite loading"
```

---

## Task 9: Birthday content — photo board, video board, throne + confetti

**Files:**
- Modify: `index.html` (ASSETS, DRAWING, LOOP sections)

- [ ] **Step 9.1 — Load board images and create video elements in ASSETS**

```js
// Board images
const boardImages = CONFIG.photos.map(p => {
  const img = new Image();
  img.src = p.file; // onerror leaves it unloaded; drawPopup checks naturalWidth
  return img;
});

// Video elements
const videoEls = CONFIG.videos.map(v => {
  const el = document.createElement('video');
  el.src = v.file;
  el.preload = 'metadata';
  el.controls = true;
  el.style.cssText = 'position:absolute;display:none;width:480px;height:270px;';
  document.body.appendChild(el);
  return el;
});

let activeVideoEl = null;
```

- [ ] **Step 9.2 — Implement E-key interact trigger for boards and throne**

In the `keydown` listener, before `e.preventDefault()`:
```js
  if ((e.key === 'e' || e.key === 'E') && gameState === 'PLAYING') {
    tryInteract();
  }
  if ((e.key === 'e' || e.key === 'E' || e.key === 'Escape') &&
      (gameState === 'POPUP' || gameState === 'CELEBRATION')) {
    closePopup();
  }
  if ((e.key === 'ArrowLeft') && gameState === 'POPUP') navigatePopup(-1);
  if ((e.key === 'ArrowRight') && gameState === 'POPUP') navigatePopup(1);
```

```js
function tryInteract() {
  if (!currentRoom || currentRoom.type !== 'garden') return;

  const pb = currentRoom.photoBoard, vb = currentRoom.videoBoard, th = currentRoom.throne;
  if (Math.hypot(player.x - pb.x, player.y - pb.y) < 50) {
    openPopup('photos');
  } else if (Math.hypot(player.x - vb.x, player.y - vb.y) < 50) {
    openPopup('videos');
  } else if (Math.hypot(player.x - th.x, player.y - th.y) < 50) {
    startCelebration();
  }
}

function openPopup(type) {
  popupState.type = type;
  popupState.index = 0;
  gameState = 'POPUP';
  if (type === 'videos') showVideo(0);
}

function closePopup() {
  if (activeVideoEl) { activeVideoEl.pause(); activeVideoEl.style.display = 'none'; activeVideoEl = null; }
  popupState.type = null;
  gameState = gameState === 'CELEBRATION' ? 'PLAYING' : 'PLAYING';
  confettiParticles.length = 0;
  gameState = 'PLAYING';
}

function navigatePopup(dir) {
  const list = popupState.type === 'photos' ? CONFIG.photos : CONFIG.videos;
  popupState.index = (popupState.index + dir + list.length) % list.length;
  if (popupState.type === 'videos') {
    if (activeVideoEl) { activeVideoEl.pause(); activeVideoEl.style.display = 'none'; }
    showVideo(popupState.index);
  }
}

function showVideo(idx) {
  activeVideoEl = videoEls[idx];
  const rect = canvas.getBoundingClientRect();
  activeVideoEl.style.left = (rect.left + W/2 - 240) + 'px';
  activeVideoEl.style.top  = (rect.top  + H/2 - 135) + 'px';
  activeVideoEl.style.display = 'block';
}
```

- [ ] **Step 9.3 — Implement drawPopup() for photo board**

```js
function drawPopup() {
  // Dim background
  ctx.fillStyle = 'rgba(0,0,0,0.75)';
  ctx.fillRect(0, 0, W, H);

  const pw = 600, ph = 360;
  const px = W/2 - pw/2, py = H/2 - ph/2;

  // Frame
  ctx.fillStyle = '#1a1209';
  roundRect(ctx, px, py, pw, ph, 8); ctx.fill();
  ctx.strokeStyle = '#8b6914'; ctx.lineWidth = 4;
  ctx.stroke();
  ctx.strokeStyle = '#5a4510'; ctx.lineWidth = 2;
  roundRect(ctx, px + 4, py + 4, pw - 8, ph - 8, 6); ctx.stroke();

  if (popupState.type === 'photos') {
    const cfg = CONFIG.photos[popupState.index];
    const img = boardImages[popupState.index];

    // Header
    ctx.fillStyle = '#e8c547'; ctx.font = px2(6);
    ctx.textAlign = 'center';
    ctx.fillText('📸 PHOTOS FROM FRIENDS', W/2, py + 24);
    ctx.fillStyle = '#666'; ctx.font = px2(5);
    ctx.fillText(`${popupState.index + 1} / ${CONFIG.photos.length}`, W/2, py + 40);

    // Image or placeholder
    const imgX = W/2 - 200, imgY = py + 55, imgW = 400, imgH = 220;
    if (img.naturalWidth > 0) {
      ctx.drawImage(img, imgX, imgY, imgW, imgH);
    } else {
      ctx.fillStyle = '#222'; ctx.fillRect(imgX, imgY, imgW, imgH);
      ctx.fillStyle = '#444'; ctx.font = px2(6); ctx.textAlign = 'center';
      ctx.fillText('[ drop ' + cfg.file + ' here ]', W/2, imgY + imgH/2);
    }

    // Caption
    ctx.fillStyle = '#c8a040'; ctx.font = px2(6);
    ctx.fillText(cfg.caption, W/2, imgY + imgH + 20);

    // Nav arrows
    ctx.fillStyle = '#e8c547'; ctx.font = '28px sans-serif';
    ctx.fillText('◄', px + 20, H/2 + 10);
    ctx.fillText('►', px + pw - 48, H/2 + 10);
  }

  if (popupState.type === 'videos') {
    ctx.fillStyle = '#a0a0e0'; ctx.font = px2(6); ctx.textAlign = 'center';
    ctx.fillText('🎥 VIDEO MESSAGES', W/2, py + 24);
    ctx.fillStyle = '#666'; ctx.font = px2(5);
    ctx.fillText(`${popupState.index + 1} / ${CONFIG.videos.length}`, W/2, py + 40);
    ctx.fillStyle = '#444'; ctx.font = px2(5);
    ctx.fillText(CONFIG.videos[popupState.index].caption, W/2, py + ph - 30);
    ctx.fillStyle = '#e8c547'; ctx.font = '28px sans-serif';
    ctx.fillText('◄', px + 20, H/2 + 10);
    ctx.fillText('►', px + pw - 48, H/2 + 10);
  }

  // Footer
  ctx.fillStyle = '#555'; ctx.font = px2(5);
  ctx.fillText('[< >] navigate   [E / Esc] close', W/2, py + ph - 12);
  ctx.textAlign = 'left';
}

function px2(size) { return `${size}px 'Press Start 2P', monospace`; }
// Note: px2 is same as px — unify them: rename px() → pxFont() or just use px() throughout.
// Actually just replace px2 with px in the above.
```

Wire popup into render():
```js
  if (gameState === 'POPUP') {
    if (currentRoom) drawRoom(currentRoom);
    drawGardenProps(currentRoom);
    drawPlayer();
    drawHUD();
    drawPopup();
    ctx.restore(); return;
  }
```

- [ ] **Step 9.4 — Implement throne + confetti celebration**

```js
function startCelebration() {
  gameState = 'CELEBRATION';
  // Spawn confetti
  const colors = ['#ff6b6b','#ffd93d','#6bcb77','#4d96ff','#ff922b','#cc5de8'];
  for (let i = 0; i < 80; i++) {
    confettiParticles.push({
      x: Math.random() * W,
      y: -10 - Math.random() * 60,
      vx: (Math.random() - 0.5) * 3,
      vy: 2 + Math.random() * 3,
      rot: Math.random() * Math.PI * 2,
      rotV: (Math.random() - 0.5) * 0.15,
      w: 8 + Math.random() * 8,
      h: 5 + Math.random() * 5,
      color: colors[Math.floor(Math.random() * colors.length)],
    });
  }
}

function updateCelebration() {
  for (let i = confettiParticles.length - 1; i >= 0; i--) {
    const c = confettiParticles[i];
    c.x += c.vx; c.y += c.vy; c.rot += c.rotV;
    c.vy += 0.05; // gravity
    if (c.y > H + 20) confettiParticles.splice(i, 1);
  }
}

function drawCelebration() {
  if (currentRoom) drawRoom(currentRoom);
  drawGardenProps(currentRoom);
  drawPlayer();
  drawHUD();

  // Confetti
  confettiParticles.forEach(c => {
    ctx.save();
    ctx.translate(c.x, c.y);
    ctx.rotate(c.rot);
    ctx.fillStyle = c.color;
    ctx.fillRect(-c.w/2, -c.h/2, c.w, c.h);
    ctx.restore();
  });

  // Banner
  const bw = 520, bh = 140;
  const bx = W/2 - bw/2, by = H/2 - bh/2;
  ctx.fillStyle = 'rgba(10,20,10,0.88)';
  roundRect(ctx, bx, by, bw, bh, 12); ctx.fill();
  ctx.strokeStyle = '#e8c547'; ctx.lineWidth = 3;
  ctx.stroke();

  ctx.textAlign = 'center';
  ctx.fillStyle = '#e8c547'; ctx.font = px(10);
  ctx.shadowColor = '#e8c547'; ctx.shadowBlur = 12;
  ctx.fillText('🎉 HAPPY BIRTHDAY 🎉', W/2, by + 48);
  ctx.shadowBlur = 0;
  ctx.fillStyle = '#ffffff'; ctx.font = px(18);
  ctx.shadowColor = '#fff'; ctx.shadowBlur = 15;
  ctx.fillText(CONFIG.friendName + '!', W/2, by + 90);
  ctx.shadowBlur = 0;
  ctx.fillStyle = '#a0c87a'; ctx.font = px(6);
  ctx.fillText(CONFIG.subtitle, W/2, by + 116);
  ctx.fillStyle = '#555'; ctx.font = px(5);
  ctx.fillText('[E / Esc] keep wandering', W/2, by + bh + 16);
  ctx.textAlign = 'left';
}
```

Wire into update() and render():
```js
// update():
  if (gameState === 'CELEBRATION') { updateCelebration(); return; }

// render():
  if (gameState === 'CELEBRATION') { drawCelebration(); ctx.restore(); return; }
```

Close celebration on E/Esc (already handled by `closePopup()` — update it to also handle CELEBRATION):
```js
function closePopup() {
  if (activeVideoEl) { activeVideoEl.pause(); activeVideoEl.style.display = 'none'; activeVideoEl = null; }
  confettiParticles.length = 0;
  gameState = 'PLAYING';
}
```

- [ ] **Step 9.5 — Verify birthday content in browser**

Navigate to the Garden room (can temporarily set `currentRoomIdx = 5` in `initGame()` for quick testing):
- Walk near PHOTOS board → [E] prompt → press E → popup opens with photo or placeholder
- ← → navigate photos
- E/Esc closes
- VIDEO board → video element appears over canvas
- Walk to THRONE → press E → confetti falls, banner appears
- E dismisses banner, player can roam
- No console errors

Remove any debug shortcuts.

- [ ] **Step 9.6 — Commit**

```bash
git add index.html
git commit -m "feat: birthday content — photo/video boards, throne confetti celebration"
```

---

## Task 10: Web Audio — procedural sound effects

**Files:**
- Modify: `index.html` (AUDIO section)

- [ ] **Step 10.1 — Add AUDIO section with context and helper**

```js
// ─── AUDIO ────────────────────────────────────────────────────────────────
let audioCtx = null;

function getAudioCtx() {
  if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  return audioCtx;
}

function playTone(freq, type, gainVal, duration, startTime, endFreq) {
  const ctx = getAudioCtx();
  const t = startTime ?? ctx.currentTime;
  const osc = ctx.createOscillator();
  const gain = ctx.createGain();
  osc.connect(gain); gain.connect(ctx.destination);
  osc.type = type || 'sine';
  osc.frequency.setValueAtTime(freq, t);
  if (endFreq !== undefined) osc.frequency.exponentialRampToValueAtTime(endFreq, t + duration);
  gain.gain.setValueAtTime(gainVal, t);
  gain.gain.exponentialRampToValueAtTime(0.001, t + duration);
  osc.start(t); osc.stop(t + duration);
}
```

- [ ] **Step 10.2 — Implement all sound functions**

```js
function playShoot() {
  playTone(880, 'sine', 0.15, 0.07, null, 660);
}

function playHit() {
  playTone(220, 'sawtooth', 0.3, 0.12, null, 110);
}

function playPlayerHit() {
  const ctx = getAudioCtx();
  const t = ctx.currentTime;
  playTone(150, 'sawtooth', 0.4, 0.08, t, 80);
  playTone(80, 'sine', 0.2, 0.2, t + 0.05, 40);
}

function playEnemyDeath() {
  playTone(440, 'square', 0.2, 0.15, null, 880);
}

function playTankDeath() {
  const ctx = getAudioCtx();
  const t = ctx.currentTime;
  playTone(80, 'sawtooth', 0.5, 0.3, t, 30);
  playTone(40, 'sine', 0.3, 0.5, t + 0.1, 20);
}

function playDoorUnlock() {
  const ctx = getAudioCtx();
  const t = ctx.currentTime;
  [523, 659, 784].forEach((freq, i) => playTone(freq, 'sine', 0.2, 0.15, t + i * 0.1));
}

function playSkillPickup() {
  const ctx = getAudioCtx();
  const t = ctx.currentTime;
  [523, 659, 784, 1047].forEach((freq, i) => playTone(freq, 'sine', 0.15, 0.12, t + i * 0.08));
}

function playBossIntro() {
  const ctx = getAudioCtx();
  const t = ctx.currentTime;
  playTone(80, 'sawtooth', 0.4, 1.5, t, 40);
  playTone(120, 'sine', 0.2, 0.3, t + 1.2);
}

function playBossDeath() {
  const ctx = getAudioCtx();
  const t = ctx.currentTime;
  [261, 329, 392, 523, 659, 784].forEach((freq, i) =>
    playTone(freq, 'sine', 0.25, 0.3, t + i * 0.1));
}

function playConfettiFanfare() {
  const ctx = getAudioCtx();
  const t = ctx.currentTime;
  [523, 659, 784, 1047, 1318].forEach((freq, i) =>
    playTone(freq, 'sine', 0.2, 0.4, t + i * 0.1));
}
```

- [ ] **Step 10.3 — Wire sounds to events**

| Location | Call |
|----------|------|
| `fireTear()` | `playShoot()` |
| `damageEnemy()` | `playHit()` |
| `killEnemy()` where type==='tank' | `playTankDeath()` else `playEnemyDeath()` |
| `checkPlayerEnemyCollisions()` on hit | `playPlayerHit()` |
| `checkRoomClear()` | `playDoorUnlock()` after skill card dismissed |
| `applySkill()` | `playSkillPickup()` |
| `updateBossIntro()` on first frame | `playBossIntro()` |
| `killBoss()` | `playBossDeath()` |
| `startCelebration()` | `playConfettiFanfare()` |

Note: `AudioContext` is suspended until user gesture. `getAudioCtx()` is called lazily — the first keypress in `initGame()` triggers context creation fine.

- [ ] **Step 10.4 — Verify audio in browser**

Play through the game and verify each sound triggers correctly. Browser DevTools → Console: no `AudioContext` errors.

- [ ] **Step 10.5 — Commit**

```bash
git add index.html
git commit -m "feat: Web Audio procedural sound effects for all game events"
```

---

## Task 11: Final verification + packaging

**Files:**
- No code changes — verification only, then packaging instructions

- [ ] **Step 11.1 — Full playthrough in browser**

Open `index.html` by double-clicking (not via dev server). Complete the full run:
- [ ] Title screen renders, Enter starts game
- [ ] Start room: no enemies, garden art, door east
- [ ] Fight Room 1: 3 grubs + 1 fly, doors lock, skill pickup after clear
- [ ] Fight Room 2: 2 flies + 2 grubs, skill pickup after clear
- [ ] Fight Room 3: 1 tank + 2 grubs, skill pickup after clear
- [ ] Boss room: intro flash, Cake Golem phase 1 → phase 2 → death → skill pickup
- [ ] Garden: boards + throne present, [E] prompts show
- [ ] Photos board: opens, ← → navigate, placeholder shows if file missing, Esc closes
- [ ] Videos board: video element overlays, Esc closes
- [ ] Throne: confetti falls, banner shows, Esc dismisses
- [ ] Minimap: all rooms update correctly
- [ ] All 8 skills selectable across the run (2 per pickup, 4 pickups = 8 total)
- [ ] Audio plays on all events
- [ ] No console errors (open DevTools → Console tab)

- [ ] **Step 11.2 — Test with real assets**

Drop in `img/character.png` and verify:
- Player sprite renders without white fringe
- Sprite flips horizontally when moving left

Drop in `img/board1.png` and verify:
- Photo shows in popup instead of placeholder

- [ ] **Step 11.3 — Push to GitHub**

```bash
git push origin main
```

- [ ] **Step 11.4 — Packaging instructions (for delivering the gift)**

To deliver to NopeYep:
1. Zip the folder: `zip -r birthday-game.zip index.html img/ vid/` (ensure real assets are in `img/` and `vid/`)
2. Send `birthday-game.zip`
3. Recipient unzips and double-clicks `index.html`
4. No internet required — the game runs fully offline

---

## Self-Review Notes

- All function names used across tasks are consistent: `drawGrub`, `drawFly`, `drawTank`, `drawBloom`, `updateGrub`, `updateFly`, `updateTank`, `updateBloom`, `killEnemy`, `killBoss`, `spawnEnemy`, `checkRoomClear`, `checkTearEnemyCollisions`, `resolveCircleVsObstacles`, `clampToRoom`, `enemyRadius`, `openPopup`, `closePopup`, `navigatePopup`, `tryInteract`, `startCelebration`, `roundRect`, `wrapText`, `px`, `startShake`, `spawnParticles`, `impactParticles`, `loadRoom`, `startTransition`, `checkDoorTrigger`, `fireTear`, `updateTears`, `updateParticles`, `updateOrbitals`, `applySkill`, `initBoss`, `updatePet`, `drawPet`
- `enemyRadius()` must be defined before `checkTearEnemyCollisions()` and `checkPlayerEnemyCollisions()`
- `keysJustPressed` must be cleared at start of `update()` before any state reads it
- `px()` and `px2()` are duplicates — consolidate to just `px()` throughout
- `damageEnemy()` receives enemy index `idx` — callers that call it from `forEach` need to switch to `for` loops (already done in `checkTearEnemyCollisions`)
- Orbital shield's `damageEnemy()` call: enemies may be spliced mid-iteration — use reverse index loop or `.filter()` copy
