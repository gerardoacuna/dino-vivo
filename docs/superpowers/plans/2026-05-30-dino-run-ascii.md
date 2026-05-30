# Dino Run ASCII — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-file browser clone of Chrome's Dino Run where the player is a coral ASCII-block creature that runs, jumps over cacti, and ducks under birds, with rising speed and a persisted high score.

**Architecture:** One `index.html` with embedded `<style>` and `<script>`. Rendering on a 2D `<canvas>`; the character and obstacles are drawn with `ctx.fillText` using block characters in a monospace font. Pure logic (collision, speed curve, spawn decision) is extracted into testable functions, exercised by an in-page test harness opened via `index.html?test=1`. Visual/interactive behavior is verified manually in the browser.

**Tech Stack:** Plain HTML5, Canvas 2D API, vanilla JavaScript, `localStorage`. No dependencies, no build step, no server.

---

## File Structure

- `index.html` — the entire game and the in-page test harness. Logical sections inside the single `<script>`:
  - **Constants** — canvas size, physics, colors, sprite art.
  - **Sprites & drawing** — `drawBlock`, `measureBlock`, sprite matrices.
  - **Pure logic** — `aabbOverlap`, `currentSpeed`, `shouldSpawn`, box getters (testable).
  - **Game state & loop** — state machine, `update`, `render`, `requestAnimationFrame` loop.
  - **Input** — keyboard handlers.
  - **Test harness** — `test`/`assert`/`runTests`, run when `?test` is in the URL.

Everything lives in one file by design (project requirement: single portable HTML). The test harness ships in the same file but only activates with the `?test` query param.

### Shared constants (used across tasks — keep identical)

```js
const W = 660, H = 220, GROUND_Y = 174;
const MONO = "ui-monospace, 'SF Mono', Menlo, monospace";
const PLAYER_X = 48;
const GRAVITY = 2600, JUMP_V = 820;            // px/s^2, px/s
const BASE_SPEED = 240, SPEED_PER_SCORE = 0.7, MAX_SPEED = 560; // px/s
const SCORE_RATE = 12;                          // score points per second
const HERO_FONT = 22, OBST_FONT = 16;           // px
const SHRINK = 0.22;                            // collision forgiveness (player)
const OBST_SHRINK = 0.18;                       // collision forgiveness (obstacles)
const SKY_TOP = '#bfe3ff', SKY_BOT = '#eafaff';
const GROUND_C = '#d9b382', GROUND_LINE = '#a87f4f';
const CORAL = '#cf6a4c', GREEN = '#3f9e54', BIRDC = '#6b4fa3';
const TEXT_C = '#2c3e50';
const HI_KEY = 'dinoAsciiHiScore';
```

---

## Task 1: Scaffold `index.html` with scene background + test harness

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create `index.html` with the page shell, canvas, constants, and test harness**

```html
<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Dino Run ASCII</title>
<style>
  html,body{height:100%;margin:0;background:#0e2230;display:flex;
    align-items:center;justify-content:center;font-family:system-ui,sans-serif}
  #game{image-rendering:pixelated;border-radius:8px;box-shadow:0 10px 30px rgba(0,0,0,.4)}
  #hint{color:#cfe6ff;text-align:center;margin-top:10px;font-size:13px;opacity:.8}
</style>
</head>
<body>
<div>
  <canvas id="game" width="660" height="220"></canvas>
  <div id="hint">Espacio/↑ saltar · ↓ agacharse</div>
</div>
<script>
// ---- Constants ----
const W = 660, H = 220, GROUND_Y = 174;
const MONO = "ui-monospace, 'SF Mono', Menlo, monospace";
const PLAYER_X = 48;
const GRAVITY = 2600, JUMP_V = 820;
const BASE_SPEED = 240, SPEED_PER_SCORE = 0.7, MAX_SPEED = 560;
const SCORE_RATE = 12;
const HERO_FONT = 22, OBST_FONT = 16;
const SHRINK = 0.22, OBST_SHRINK = 0.18;
const SKY_TOP = '#bfe3ff', SKY_BOT = '#eafaff';
const GROUND_C = '#d9b382', GROUND_LINE = '#a87f4f';
const CORAL = '#cf6a4c', GREEN = '#3f9e54', BIRDC = '#6b4fa3';
const TEXT_C = '#2c3e50';
const HI_KEY = 'dinoAsciiHiScore';

const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');

// ---- Scene background ----
function drawScene(ctx){
  const g = ctx.createLinearGradient(0,0,0,H);
  g.addColorStop(0, SKY_TOP); g.addColorStop(1, SKY_BOT);
  ctx.fillStyle = g; ctx.fillRect(0,0,W,H);
  ctx.fillStyle = GROUND_C; ctx.fillRect(0, GROUND_Y, W, H-GROUND_Y);
  ctx.strokeStyle = GROUND_LINE; ctx.lineWidth = 2;
  ctx.beginPath(); ctx.moveTo(0,GROUND_Y); ctx.lineTo(W,GROUND_Y); ctx.stroke();
  ctx.fillStyle = GROUND_LINE;
  for(let i=10;i<W;i+=52) ctx.fillRect(i, GROUND_Y+14, 10, 3);
}

// ---- Test harness ----
const TESTS = [];
function test(name, fn){ TESTS.push({name, fn}); }
function assert(cond, msg){ if(!cond) throw new Error(msg || 'assertion failed'); }
function runTests(){
  const out = TESTS.map(t => {
    try { t.fn(); return {name:t.name, pass:true}; }
    catch(e){ return {name:t.name, pass:false, err:e.message}; }
  });
  const pre = document.createElement('pre');
  pre.style.cssText = "font:14px monospace;padding:16px;color:#cfe6ff;text-align:left";
  pre.textContent = out.map(r => (r.pass?'PASS ':'FAIL ') + r.name +
    (r.err ? ' — ' + r.err : '')).join('\n') +
    '\n\n' + out.filter(r=>r.pass).length + '/' + out.length + ' passed';
  document.body.innerHTML = ''; document.body.appendChild(pre);
}

// ---- Boot ----
function boot(){
  if (new URLSearchParams(location.search).has('test')) { runTests(); return; }
  drawScene(ctx); // placeholder until the loop is added
}
boot();
</script>
</body>
</html>
```

- [ ] **Step 2: Verify the scene renders**

Open `index.html` in a browser.
Expected: a 660×220 canvas with a blue→white sky gradient, a tan ground band with a brown line and dashed texture. No errors in the console.

- [ ] **Step 3: Verify the test harness runs (currently empty)**

Open `index.html?test=1`.
Expected: page shows `0/0 passed`. No errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: scaffold dino ascii with scene background and test harness"
```

---

## Task 2: Sprite data + block drawing; render a static character

**Files:**
- Modify: `index.html` (add sprites + `drawBlock`/`measureBlock`, draw character in `boot`)

- [ ] **Step 1: Add sprite matrices and drawing helpers** (place after Constants, before Scene)

```js
// ---- Sprites (block ASCII) ----
const HERO_RUN1 = [
  " ▐▛███▜▌ ",
  "▝▜█████▛▘",
  "  ▘▘ ▝▝  ",
];
const HERO_RUN2 = [
  " ▐▛███▜▌ ",
  "▝▜█████▛▘",
  "  ▖▖ ▗▗  ",
];
const HERO_DUCK = [
  "▗▆█████▆▖",
  "▝▘▘  ▝▝▘ ",
];
const CACTUS = [
  "  █  ",
  "█ █  ",
  "███ █",
  "  ███",
  "  █  ",
];
const BIRD1 = [
  "▝▙   ▟▘",
  " ▜███▛ ",
];
const BIRD2 = [
  " ▟███▙ ",
  "▝▛   ▜▘",
];

// ---- Block drawing ----
function drawBlock(ctx, lines, x, topY, fontPx, color){
  ctx.font = fontPx + "px " + MONO;
  ctx.textBaseline = 'top';
  ctx.textAlign = 'left';
  ctx.fillStyle = color;
  let y = topY;
  for (const ln of lines){ ctx.fillText(ln, x, y); y += fontPx; }
}
function measureBlock(ctx, lines, fontPx){
  ctx.font = fontPx + "px " + MONO;
  let w = 0;
  for (const ln of lines) w = Math.max(w, ctx.measureText(ln).width);
  return { w, h: lines.length * fontPx };
}
```

- [ ] **Step 2: Draw a static character standing on the ground in `boot`**

Replace the placeholder line `drawScene(ctx); // placeholder until the loop is added` with:

```js
  drawScene(ctx);
  const m = measureBlock(ctx, HERO_RUN1, HERO_FONT);
  drawBlock(ctx, HERO_RUN1, PLAYER_X, GROUND_Y - m.h, HERO_FONT, CORAL);
```

- [ ] **Step 3: Verify the character renders on the ground**

Open `index.html`.
Expected: the coral block character (`▐▛███▜▌` / `▜█████▛` / `▘▘ ▝▝`) stands on the ground line at the left, roughly as in the approved mockup.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add sprite data and block drawing, render static character"
```

---

## Task 3: Pure logic — `aabbOverlap` (TDD)

**Files:**
- Modify: `index.html` (add pure-logic section + tests)

- [ ] **Step 1: Write the failing test** (add a `// ---- Pure logic tests ----` block before `// ---- Boot ----`)

```js
test('aabbOverlap: overlapping boxes return true', () => {
  assert(aabbOverlap({x:0,y:0,w:10,h:10}, {x:5,y:5,w:10,h:10}) === true);
});
test('aabbOverlap: separated on x return false', () => {
  assert(aabbOverlap({x:0,y:0,w:10,h:10}, {x:20,y:0,w:10,h:10}) === false);
});
test('aabbOverlap: separated on y return false', () => {
  assert(aabbOverlap({x:0,y:0,w:10,h:10}, {x:0,y:20,w:10,h:10}) === false);
});
test('aabbOverlap: edge-touching is not overlap', () => {
  assert(aabbOverlap({x:0,y:0,w:10,h:10}, {x:10,y:0,w:10,h:10}) === false);
});
```

- [ ] **Step 2: Run tests to verify failure**

Open `index.html?test=1`.
Expected: the four `aabbOverlap` tests show FAIL with `aabbOverlap is not defined`.

- [ ] **Step 3: Implement `aabbOverlap`** (add to a `// ---- Pure logic ----` section before the tests)

```js
function aabbOverlap(a, b){
  return a.x < b.x + b.w &&
         a.x + a.w > b.x &&
         a.y < b.y + b.h &&
         a.y + a.h > b.y;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Open `index.html?test=1`.
Expected: the four `aabbOverlap` tests show PASS.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add aabbOverlap collision helper with tests"
```

---

## Task 4: Pure logic — `currentSpeed` (TDD)

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Write the failing test** (add to the pure-logic tests block)

```js
test('currentSpeed: starts at BASE_SPEED when score is 0', () => {
  assert(currentSpeed(0) === BASE_SPEED);
});
test('currentSpeed: increases with score', () => {
  assert(currentSpeed(100) > currentSpeed(0));
});
test('currentSpeed: never exceeds MAX_SPEED', () => {
  assert(currentSpeed(100000) === MAX_SPEED);
});
```

- [ ] **Step 2: Run tests to verify failure**

Open `index.html?test=1`.
Expected: the three `currentSpeed` tests FAIL with `currentSpeed is not defined`.

- [ ] **Step 3: Implement `currentSpeed`** (add to pure-logic section)

```js
function currentSpeed(score){
  return Math.min(MAX_SPEED, BASE_SPEED + score * SPEED_PER_SCORE);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Open `index.html?test=1`.
Expected: the three `currentSpeed` tests PASS (alongside Task 3's).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add currentSpeed curve with tests"
```

---

## Task 5: Pure logic — `shouldSpawn` + box getters (TDD)

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Write the failing tests** (add to the pure-logic tests block)

```js
test('shouldSpawn: true when rightmost obstacle is far enough left', () => {
  assert(shouldSpawn(300, 660, 280) === true);   // 300 < 660-280=380
});
test('shouldSpawn: false when rightmost obstacle still too close to edge', () => {
  assert(shouldSpawn(500, 660, 280) === false);   // 500 > 380
});
test('shouldSpawn: true when there are no obstacles (-Infinity)', () => {
  assert(shouldSpawn(-Infinity, 660, 280) === true);
});
test('rightmostX: returns -Infinity for empty list', () => {
  assert(rightmostX([]) === -Infinity);
});
test('rightmostX: returns the max x', () => {
  assert(rightmostX([{x:100},{x:540},{x:300}]) === 540);
});
```

- [ ] **Step 2: Run tests to verify failure**

Open `index.html?test=1`.
Expected: the new tests FAIL with `shouldSpawn is not defined` / `rightmostX is not defined`.

- [ ] **Step 3: Implement `shouldSpawn` and `rightmostX`** (add to pure-logic section)

```js
function shouldSpawn(rightmost, canvasW, gap){
  return rightmost < canvasW - gap;
}
function rightmostX(obstacles){
  return obstacles.length ? Math.max(...obstacles.map(o => o.x)) : -Infinity;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Open `index.html?test=1`.
Expected: all tests so far PASS.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add shouldSpawn and rightmostX with tests"
```

---

## Task 6: Game state machine, loop skeleton, input, READY/GAME OVER screens

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add the game state, reset, loop, render, and input** (add a `// ---- Game ----` section before Boot)

```js
const game = {
  state: 'ready',         // 'ready' | 'playing' | 'over'
  score: 0,
  hi: 0,
  y: 0,                   // player height above ground (px)
  vy: 0,                  // vertical velocity (px/s)
  ducking: false,
  frame: 0,               // run animation frame (0/1)
  frameTimer: 0,
  obstacles: [],
  gap: 320,               // distance before next spawn
  last: 0,
};

function resetGame(){
  game.state = 'playing';
  game.score = 0;
  game.y = 0; game.vy = 0;
  game.ducking = false;
  game.frame = 0; game.frameTimer = 0;
  game.obstacles = [];
  game.gap = 320;
}

function playerSprite(){
  if (game.ducking && game.y === 0) return HERO_DUCK;
  return game.frame ? HERO_RUN2 : HERO_RUN1;
}

function drawCenteredText(ctx, lines, color){
  ctx.fillStyle = color; ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
  ctx.font = "bold 22px " + MONO;
  let y = H/2 - (lines.length-1)*14;
  for (const ln of lines){ ctx.fillText(ln, W/2, y); y += 28; }
  ctx.textAlign = 'left'; ctx.textBaseline = 'top';
}

function render(){
  drawScene(ctx);
  // obstacles (added in Task 9) — drawObstacles() is a no-op until then
  if (typeof drawObstacles === 'function') drawObstacles();
  // player
  const lines = playerSprite();
  const m = measureBlock(ctx, lines, HERO_FONT);
  drawBlock(ctx, lines, PLAYER_X, GROUND_Y - m.h - game.y, HERO_FONT, CORAL);
  // score (added in Task 10) — drawScore() is a no-op until then
  if (typeof drawScore === 'function') drawScore();
  // overlays
  if (game.state === 'ready')
    drawCenteredText(ctx, ['PRESIONA ESPACIO', 'PARA EMPEZAR'], TEXT_C);
  if (game.state === 'over')
    drawCenteredText(ctx, ['GAME OVER', 'ESPACIO PARA REINTENTAR'], TEXT_C);
}

function update(dt){
  if (game.state !== 'playing') return;
  // run animation
  game.frameTimer += dt;
  if (game.frameTimer > 0.12){ game.frame ^= 1; game.frameTimer = 0; }
  // player physics, obstacles, collisions, score added in later tasks
}

function loop(t){
  if (!game.last) game.last = t;
  const dt = Math.min((t - game.last) / 1000, 0.05); // clamp for background tabs
  game.last = t;
  update(dt);
  render();
  requestAnimationFrame(loop);
}

// ---- Input ----
function onJumpKey(){
  if (game.state === 'ready' || game.state === 'over'){ resetGame(); return; }
  if (game.state === 'playing' && game.y === 0){ game.vy = JUMP_V; }
}
window.addEventListener('keydown', (e) => {
  if (e.code === 'Space' || e.code === 'ArrowUp'){ e.preventDefault(); onJumpKey(); }
  else if (e.code === 'ArrowDown'){ e.preventDefault(); game.ducking = true; }
});
window.addEventListener('keyup', (e) => {
  if (e.code === 'ArrowDown'){ game.ducking = false; }
});
```

- [ ] **Step 2: Start the loop in `boot`**

Replace the non-test branch of `boot` (the `drawScene` + static character lines) with:

```js
  requestAnimationFrame(loop);
```

- [ ] **Step 3: Verify states and transitions manually**

Open `index.html`.
Expected:
- READY screen: character standing + "PRESIONA ESPACIO / PARA EMPEZAR".
- Press Space → overlay disappears, character's legs animate (run frames alternate).
- Pressing ↓ shows the flatter ducked sprite; releasing returns to running.
- (No obstacles/jump yet — those come next.) No console errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add game state machine, loop, input, ready/gameover screens"
```

---

## Task 7: Player jump physics

**Files:**
- Modify: `index.html` (extend `update`)

- [ ] **Step 1: Add jump integration to `update`**

Inside `update`, after the run-animation block, add:

```js
  // jump physics
  if (game.y > 0 || game.vy !== 0){
    game.y += game.vy * dt;
    game.vy -= GRAVITY * dt;
    if (game.y <= 0){ game.y = 0; game.vy = 0; }
  }
```

- [ ] **Step 2: Verify jumping manually**

Open `index.html`, press Space to start, press Space/↑ to jump.
Expected: the character rises smoothly and falls back to the ground; cannot double-jump (a second Space mid-air does nothing); ducking is ignored while airborne.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add player jump physics"
```

---

## Task 8: Player & obstacle collision boxes (TDD) + dynamic player box

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Write failing tests** (add to pure-logic tests block)

```js
test('getPlayerBox: shrinks the sprite footprint and sits on the ground', () => {
  game.y = 0; game.ducking = false; game.frame = 0;
  const b = getPlayerBox();
  assert(b.x > PLAYER_X, 'box x padded inward');
  assert(Math.abs((b.y + b.h) - (GROUND_Y - 0)) < HERO_FONT, 'box bottom near ground');
});
test('getPlayerBox: raises with jump height', () => {
  game.y = 0; const ground = getPlayerBox();
  game.y = 100; const air = getPlayerBox();
  game.y = 0;
  assert(air.y < ground.y, 'jumping raises the box');
});
test('getObstacleBox: shrinks the obstacle footprint', () => {
  const o = {x:100, y:50, w:40, h:30};
  const b = getObstacleBox(o);
  assert(b.x > o.x && b.w < o.w && b.h < o.h);
});
```

- [ ] **Step 2: Run tests to verify failure**

Open `index.html?test=1`.
Expected: the three new tests FAIL with `getPlayerBox is not defined` / `getObstacleBox is not defined`.

- [ ] **Step 3: Implement the box getters** (add to pure-logic section)

```js
function getPlayerBox(){
  const lines = playerSprite();
  const m = measureBlock(ctx, lines, HERO_FONT);
  const topY = GROUND_Y - m.h - game.y;
  const padX = m.w * SHRINK, padY = m.h * SHRINK;
  return { x: PLAYER_X + padX, y: topY + padY, w: m.w - 2*padX, h: m.h - 2*padY };
}
function getObstacleBox(o){
  const padX = o.w * OBST_SHRINK, padY = o.h * OBST_SHRINK;
  return { x: o.x + padX, y: o.y + padY, w: o.w - 2*padX, h: o.h - 2*padY };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Open `index.html?test=1`.
Expected: all tests PASS.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add player and obstacle collision boxes with tests"
```

---

## Task 9: Obstacle spawning, movement, drawing, bird flap

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add the obstacle factory and draw function** (add to the Game section)

```js
function randGap(){ return 260 + Math.random() * 260; }   // 260..520 px

function makeObstacle(kind){
  const frames = kind === 'cactus' ? [CACTUS] : [BIRD1, BIRD2];
  const color  = kind === 'cactus' ? GREEN : BIRDC;
  const m = measureBlock(ctx, frames[0], OBST_FONT);
  let topY;
  if (kind === 'cactus')        topY = GROUND_Y - m.h;        // on the ground -> jump
  else if (kind === 'birdHigh') topY = 96;                    // body height -> duck
  else                          topY = GROUND_Y - m.h - 30;   // low -> jump
  return { kind, frames, color, x: W, y: topY, w: m.w, h: m.h, flip: 0, flipT: 0 };
}

function spawnObstacle(){
  const r = Math.random();
  const kind = r < 0.5 ? 'cactus' : (r < 0.75 ? 'birdHigh' : 'birdLow');
  game.obstacles.push(makeObstacle(kind));
  game.gap = randGap();
}

function drawObstacles(){
  for (const o of game.obstacles){
    const frame = o.frames.length > 1 ? o.frames[o.flip] : o.frames[0];
    drawBlock(ctx, frame, o.x, o.y, OBST_FONT, o.color);
  }
}
```

- [ ] **Step 2: Add spawning + movement to `update`**

Inside `update`, after the jump-physics block, add:

```js
  // spawn
  if (shouldSpawn(rightmostX(game.obstacles), W, game.gap)) spawnObstacle();
  // move + animate + cull
  const speed = currentSpeed(game.score);
  for (const o of game.obstacles){
    o.x -= speed * dt;
    if (o.frames.length > 1){
      o.flipT += dt;
      if (o.flipT > 0.15){ o.flip ^= 1; o.flipT = 0; }
    }
  }
  game.obstacles = game.obstacles.filter(o => o.x + o.w > 0);
```

- [ ] **Step 3: Verify obstacles manually**

Open `index.html`, press Space.
Expected: green cacti (on the ground) and purple birds (some at body height, some low) scroll in from the right at a steady pace, spaced apart; birds flap (frames alternate); obstacles disappear off the left edge. Passing through them does nothing yet (collision is Task 10).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add obstacle spawning, movement, and bird flap animation"
```

---

## Task 10: Collision → game over, score increment + display, speed ramp

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add the score drawing function** (add to the Game section)

```js
function drawScore(){
  ctx.fillStyle = TEXT_C;
  ctx.font = "bold 17px " + MONO;
  ctx.textAlign = 'right'; ctx.textBaseline = 'top';
  const s = String(Math.floor(game.score)).padStart(5, '0');
  const hi = String(Math.floor(game.hi)).padStart(5, '0');
  ctx.fillText('HI ' + hi + '   ' + s, W - 16, 16);
  ctx.textAlign = 'left';
}
```

- [ ] **Step 2: Add score increment and collision to `update`**

Inside `update`, after the cull line (`game.obstacles = ...`), add:

```js
  // score
  game.score += dt * SCORE_RATE;
  // collision
  const pb = getPlayerBox();
  for (const o of game.obstacles){
    if (aabbOverlap(pb, getObstacleBox(o))){
      game.state = 'over';
      if (game.score > game.hi) game.hi = game.score;
      break;
    }
  }
```

- [ ] **Step 3: Verify gameplay manually**

Open `index.html`, press Space, and play.
Expected: the score counter rises; the speed visibly increases as the score grows; hitting a cactus or bird triggers the GAME OVER overlay; the high score (HI) updates to the best run; Space restarts cleanly. Confirm a well-timed jump clears cacti and the low bird, and ducking clears the body-height bird.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add collision game-over, score display, and speed ramp"
```

---

## Task 11: Persist high score in `localStorage`

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add safe load/save helpers** (add to the Game section, above `const game`)

```js
function loadHi(){
  try { return parseInt(localStorage.getItem(HI_KEY), 10) || 0; }
  catch(e){ return 0; }
}
function saveHi(v){
  try { localStorage.setItem(HI_KEY, String(Math.floor(v))); }
  catch(e){ /* localStorage unavailable; keep playing without persistence */ }
}
```

- [ ] **Step 2: Initialize `game.hi` from storage and save on game over**

Change the `game` object's `hi: 0,` line to:

```js
  hi: loadHi(),
```

In `update`, inside the collision branch, replace:

```js
      if (game.score > game.hi) game.hi = game.score;
```

with:

```js
      if (game.score > game.hi){ game.hi = game.score; saveHi(game.hi); }
```

- [ ] **Step 3: Verify persistence manually**

Open `index.html`, play and lose to set a score, then reload the page.
Expected: the HI value persists across reload. Then open the browser in a mode where `localStorage` throws (or temporarily rename `HI_KEY` usage): the game still runs without errors — only persistence is lost.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: persist high score in localStorage with safe fallback"
```

---

## Task 12: Difficulty tuning + final acceptance pass

**Files:**
- Modify: `index.html` (constants only, if needed)

- [ ] **Step 1: Tune feel by adjusting constants**

Play several rounds. If jumps don't clear cacti, raise `JUMP_V` (e.g. 820 → 880) or lower the cactus height by removing a line from `CACTUS`. If the game ramps too fast, lower `SPEED_PER_SCORE` (e.g. 0.7 → 0.5) or `MAX_SPEED`. If body-height birds are unfair, nudge the `birdHigh` `topY` (96) up a few px; if low birds can't be jumped, reduce the `birdLow` offset (30). Change only constants/sprite line counts — no structural changes.

- [ ] **Step 2: Run the full acceptance checklist (from the spec)**

Open `index.html` and confirm each:
- [ ] READY screen shows the character + instruction.
- [ ] Space starts the game.
- [ ] Character runs with leg animation.
- [ ] Jump (Space/↑) clears cacti; no double-jump.
- [ ] Duck (↓) clears body-height birds.
- [ ] Score rises continuously and is shown.
- [ ] Speed increases with score.
- [ ] Collision → GAME OVER with score + record.
- [ ] HI persists after reload.
- [ ] Space restarts from GAME OVER.

- [ ] **Step 3: Run the test harness one final time**

Open `index.html?test=1`.
Expected: all pure-logic tests PASS.

- [ ] **Step 4: Commit any tuning changes**

```bash
git add index.html
git commit -m "chore: tune difficulty and finalize dino ascii"
```

---

## Self-Review Notes

- **Spec coverage:** single-file/no-deps (Task 1), colorful scene (Task 1), ASCII character + run animation (Tasks 2, 6), ASCII obstacles incl. bird flap (Task 9), jump (Task 7), duck + ducked sprite (Tasks 2, 6, 8), rising speed (Tasks 4, 9, 10), score + display (Task 10), high score persistence + fallback (Task 11), AABB collision with forgiveness (Tasks 3, 8, 10), state screens (Task 6), delta-time clamp for background tabs (Task 6 `loop`), fixed centered canvas (Task 1 CSS). Manual + pure-logic testing strategy (harness in Task 1, used throughout). All spec sections map to a task.
- **Out of scope honored:** no day/night, no backend, no sound, no touch controls.
- **Type consistency:** obstacle shape `{kind, frames, color, x, y, w, h, flip, flipT}` is created in `makeObstacle` (Task 9) and consumed identically by `drawObstacles` (Task 9) and `getObstacleBox` (Task 8). `game` fields set in Task 6 are read consistently in Tasks 7–11. `playerSprite`, `measureBlock`, `drawBlock`, `aabbOverlap`, `currentSpeed`, `shouldSpawn`, `rightmostX`, `getPlayerBox`, `getObstacleBox` names are used identically wherever referenced.
- **Note on `?test` mode:** the test harness replaces page contents and never starts the loop, so tests never race the game.
