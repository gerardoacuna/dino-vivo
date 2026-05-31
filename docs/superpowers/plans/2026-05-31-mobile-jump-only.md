# Mobile Jump-Only Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the ASCII Dino game comfortable on phones: one control (jump) via tap anywhere, cactus-only obstacles, and a canvas that scales to the screen width — keeping desktop keyboard play working.

**Architecture:** Targeted edits to the existing single file `index.html`. Remove the duck mechanic and all bird obstacles, add a `pointerdown` tap handler that reuses the existing `onJumpKey()` logic, and make the canvas responsive via CSS. Pure game logic (physics, speed, score, collision) is unchanged.

**Tech Stack:** Plain HTML5, Canvas 2D, vanilla JS, `localStorage`. Verification: `node --check` for syntax, the in-page `?test=1` harness for logic, plus a Node replication of the pure-logic suite. No dependencies, no build.

---

## File Structure

- `index.html` — the only file changed. All edits are localized to existing sections (Sprites, Game, Input, Boot) and the `<style>`/`<body>` shell.

No new files. The single-file structure is a deliberate project requirement.

---

## Task 1: Cactus-only obstacles (remove birds)

**Files:**
- Modify: `index.html` (sprites, `makeObstacle`, `spawnObstacle`, `drawObstacles`, `update` move loop; add a browser test)

- [ ] **Step 1: Add a failing browser test for cactus-only spawning**

In the `// ---- Pure logic tests ----` section, immediately AFTER the `getObstacleBox` test (the block ending at the line `});` right before `// ---- Game ----`), add:

```js
test('spawnObstacle: only spawns ground cacti', () => {
  game.obstacles = [];
  for (let i = 0; i < 20; i++) spawnObstacle();
  for (const o of game.obstacles){
    assert(o.h === CACTUS.length * OBST_FONT, 'every obstacle has cactus height');
    assert(Math.abs((o.y + o.h) - GROUND_Y) < 0.001, 'every obstacle sits on the ground');
  }
  game.obstacles = [];
});
```

- [ ] **Step 2: Verify it fails (conceptually) before the change**

This test runs in the browser via `index.html?test=1`. Before the Step 3–6 changes, `spawnObstacle` can produce `birdHigh`/`birdLow` (different height/`y`), so the assertions would fail. You cannot run a browser here; just confirm by inspection that current `spawnObstacle` (lines ~252-257) randomly picks bird kinds, which violates the test. Proceed to make it pass.

- [ ] **Step 3: Remove the BIRD sprites**

Delete these exact lines from the `// ---- Sprites (block ASCII) ----` section:

```js
const BIRD1 = [
  "▝▙   ▟▘",
  " ▜███▛ ",
];
const BIRD2 = [
  " ▟███▙ ",
  "▝▛   ▜▘",
];
```

- [ ] **Step 4: Simplify `makeObstacle` to cactus-only**

Replace the entire `makeObstacle` function:

```js
function makeObstacle(kind){
  const frames = kind === 'cactus' ? [CACTUS] : [BIRD1, BIRD2];
  const color  = kind === 'cactus' ? GREEN : BIRDC;
  const m = measureBlock(ctx, frames[0], OBST_FONT);
  let topY;
  if (kind === 'cactus')        topY = GROUND_Y - m.h;        // on the ground -> jump
  else if (kind === 'birdHigh') topY = 105;                   // body height -> duck
  else                          topY = GROUND_Y - m.h - 30;   // low -> jump
  return { kind, frames, color, x: W, y: topY, w: m.w, h: m.h, flip: 0, flipT: 0 };
}
```

with:

```js
function makeObstacle(){
  const m = measureBlock(ctx, CACTUS, OBST_FONT);
  return { x: W, y: GROUND_Y - m.h, w: m.w, h: m.h };
}
```

- [ ] **Step 5: Simplify `spawnObstacle`**

Replace the entire `spawnObstacle` function:

```js
function spawnObstacle(){
  const r = Math.random();
  const kind = r < 0.5 ? 'cactus' : (r < 0.75 ? 'birdHigh' : 'birdLow');
  game.obstacles.push(makeObstacle(kind));
  game.gap = randGap();
}
```

with:

```js
function spawnObstacle(){
  game.obstacles.push(makeObstacle());
  game.gap = randGap();
}
```

- [ ] **Step 6: Simplify `drawObstacles` and remove flap animation from `update`**

Replace the entire `drawObstacles` function:

```js
function drawObstacles(){
  for (const o of game.obstacles){
    const frame = o.frames.length > 1 ? o.frames[o.flip] : o.frames[0];
    drawBlock(ctx, frame, o.x, o.y, OBST_FONT, o.color);
  }
}
```

with:

```js
function drawObstacles(){
  for (const o of game.obstacles){
    drawBlock(ctx, CACTUS, o.x, o.y, OBST_FONT, GREEN);
  }
}
```

Then, in `update(dt)`, replace the move loop:

```js
  for (const o of game.obstacles){
    o.x -= speed * dt;
    if (o.frames.length > 1){
      o.flipT += dt;
      if (o.flipT > 0.15){ o.flip ^= 1; o.flipT = 0; }
    }
  }
```

with:

```js
  for (const o of game.obstacles){
    o.x -= speed * dt;
  }
```

- [ ] **Step 7: Verify**

Run:
```bash
cd /Users/gerardoacuna/Sites/dino
awk '/<script>/{f=1;next}/<\/script>/{f=0}f' index.html > /tmp/c.js && node --check /tmp/c.js && echo SYNTAX_OK && rm /tmp/c.js
grep -n "BIRD1\|BIRD2\|birdHigh\|birdLow\|BIRDC\|\.flip\|frames" index.html || echo "NO_BIRD_REFS"
```
Expected: `SYNTAX_OK`, and `NO_BIRD_REFS` (no remaining references to birds, flip, or frames). Note: the `BIRDC` constant declaration on line ~31 may still exist — that's fine to leave OR remove; if `grep` only shows the `const ... BIRDC` declaration line and nothing else, that is acceptable. (The browser test from Step 1 will pass when you open `index.html?test=1`.)

- [ ] **Step 8: Commit**

```bash
cd /Users/gerardoacuna/Sites/dino
git add index.html
git commit -m "feat: cactus-only obstacles (remove birds)"
```

---

## Task 2: Remove the duck mechanic

**Files:**
- Modify: `index.html` (`HERO_DUCK`, `game.ducking`, `resetGame`, `playerSprite`, ArrowDown listeners, `getPlayerBox` test)

- [ ] **Step 1: Remove the `HERO_DUCK` sprite**

Delete these exact lines from the Sprites section:

```js
const HERO_DUCK = [
  "▗▆█████▆▖",
  "▝▘▘  ▝▝▘ ",
];
```

- [ ] **Step 2: Remove `ducking` from the `game` object**

In the `game` object literal, delete this exact line:

```js
  ducking: false,
```

- [ ] **Step 3: Remove `ducking` reset from `resetGame`**

In `resetGame`, delete this exact line:

```js
  game.ducking = false;
```

- [ ] **Step 4: Simplify `playerSprite`**

Replace the entire `playerSprite` function:

```js
function playerSprite(){
  if (game.ducking && game.y === 0) return HERO_DUCK;
  return game.frame ? HERO_RUN2 : HERO_RUN1;
}
```

with:

```js
function playerSprite(){
  return game.frame ? HERO_RUN2 : HERO_RUN1;
}
```

- [ ] **Step 5: Remove ArrowDown handling from input**

Replace the keydown listener:

```js
window.addEventListener('keydown', (e) => {
  if (e.code === 'Space' || e.code === 'ArrowUp'){ e.preventDefault(); onJumpKey(); }
  else if (e.code === 'ArrowDown'){ e.preventDefault(); game.ducking = true; }
});
window.addEventListener('keyup', (e) => {
  if (e.code === 'ArrowDown'){ game.ducking = false; }
});
```

with:

```js
window.addEventListener('keydown', (e) => {
  if (e.code === 'Space' || e.code === 'ArrowUp'){ e.preventDefault(); onJumpKey(); }
});
```

- [ ] **Step 6: Remove the `ducking` reference in the `getPlayerBox` test**

In the `// ---- Pure logic tests ----` section, replace this line:

```js
  game.y = 0; game.ducking = false; game.frame = 0;
```

with:

```js
  game.y = 0; game.frame = 0;
```

- [ ] **Step 7: Verify**

Run:
```bash
cd /Users/gerardoacuna/Sites/dino
awk '/<script>/{f=1;next}/<\/script>/{f=0}f' index.html > /tmp/c.js && node --check /tmp/c.js && echo SYNTAX_OK && rm /tmp/c.js
grep -n "ducking\|HERO_DUCK\|ArrowDown\|keyup" index.html || echo "NO_DUCK_REFS"
```
Expected: `SYNTAX_OK` and `NO_DUCK_REFS`.

- [ ] **Step 8: Commit**

```bash
cd /Users/gerardoacuna/Sites/dino
git add index.html
git commit -m "feat: remove duck mechanic"
```

---

## Task 3: Tap-anywhere control

**Files:**
- Modify: `index.html` (`// ---- Input ----` section)

- [ ] **Step 1: Add a `pointerdown` listener that reuses `onJumpKey`**

In the `// ---- Input ----` section, immediately AFTER the `keydown` listener (the one added in Task 2), add:

```js
window.addEventListener('pointerdown', (e) => {
  e.preventDefault();
  onJumpKey();
});
```

- [ ] **Step 2: Verify**

Run:
```bash
cd /Users/gerardoacuna/Sites/dino
awk '/<script>/{f=1;next}/<\/script>/{f=0}f' index.html > /tmp/c.js && node --check /tmp/c.js && echo SYNTAX_OK && rm /tmp/c.js
grep -n "pointerdown" index.html
```
Expected: `SYNTAX_OK` and one line showing the `pointerdown` listener. (Behavior — a tap starts/jumps/restarts — is verified on a real phone in Task 5's manual checklist.)

- [ ] **Step 3: Commit**

```bash
cd /Users/gerardoacuna/Sites/dino
git add index.html
git commit -m "feat: tap anywhere to jump (pointerdown)"
```

---

## Task 4: Responsive canvas + updated hint

**Files:**
- Modify: `index.html` (`<style>`, `<body>` wrapper div, hint text)

- [ ] **Step 1: Make the canvas scale to the screen width**

In `<style>`, replace this exact line:

```css
  #game{image-rendering:pixelated;border-radius:8px;box-shadow:0 10px 30px rgba(0,0,0,.4)}
```

with these two rules:

```css
  #wrap{width:min(96vw,720px)}
  #game{image-rendering:pixelated;border-radius:8px;box-shadow:0 10px 30px rgba(0,0,0,.4);width:100%;height:auto;touch-action:none;user-select:none;-webkit-user-select:none}
```

- [ ] **Step 2: Give the wrapper div the `wrap` id**

Replace this exact line:

```html
<div>
  <canvas id="game" width="660" height="220"></canvas>
```

with:

```html
<div id="wrap">
  <canvas id="game" width="660" height="220"></canvas>
```

- [ ] **Step 3: Update the hint text**

Replace this exact line:

```html
  <div id="hint">Espacio/↑ saltar · ↓ agacharse</div>
```

with:

```html
  <div id="hint">Toca la pantalla o Espacio para saltar</div>
```

- [ ] **Step 4: Verify**

Run:
```bash
cd /Users/gerardoacuna/Sites/dino
grep -n "id=\"wrap\"\|min(96vw,720px)\|touch-action:none\|Toca la pantalla" index.html
```
Expected: lines showing the `#wrap` rule, the wrapper div id, `touch-action:none`, and the new hint text. (Visual scaling on a phone is checked in Task 5.)

- [ ] **Step 5: Commit**

```bash
cd /Users/gerardoacuna/Sites/dino
git add index.html
git commit -m "feat: responsive canvas scaling and mobile hint"
```

---

## Task 5: Final verification

**Files:**
- None (verification only)

- [ ] **Step 1: Confirm syntax and that the pure-logic suite still passes in Node**

Run:
```bash
cd /Users/gerardoacuna/Sites/dino
awk '/<script>/{f=1;next}/<\/script>/{f=0}f' index.html > /tmp/c.js && node --check /tmp/c.js && echo SYNTAX_OK && rm /tmp/c.js
node <<'EOF'
const BASE_SPEED=240,SPEED_PER_SCORE=0.7,MAX_SPEED=560,OBST_SHRINK=0.18;
function aabbOverlap(a,b){return a.x<b.x+b.w&&a.x+a.w>b.x&&a.y<b.y+b.h&&a.y+a.h>b.y;}
function currentSpeed(s){return Math.min(MAX_SPEED,BASE_SPEED+s*SPEED_PER_SCORE);}
function shouldSpawn(r,w,g){return r<w-g;}
function rightmostX(o){return o.length?Math.max(...o.map(x=>x.x)):-Infinity;}
function getObstacleBox(o){const px=o.w*OBST_SHRINK,py=o.h*OBST_SHRINK;return{x:o.x+px,y:o.y+py,w:o.w-2*px,h:o.h-2*py};}
let n=0,p=0;const t=(nm,f)=>{n++;try{f();p++;}catch(e){console.log('FAIL',nm,e.message);}};const ok=(c,m)=>{if(!c)throw new Error(m||'a');};
t('aabb',()=>ok(aabbOverlap({x:0,y:0,w:10,h:10},{x:5,y:5,w:10,h:10})));
t('aabb-x',()=>ok(!aabbOverlap({x:0,y:0,w:10,h:10},{x:20,y:0,w:10,h:10})));
t('spd-base',()=>ok(currentSpeed(0)===BASE_SPEED));
t('spd-cap',()=>ok(currentSpeed(1e6)===MAX_SPEED));
t('spawn',()=>ok(shouldSpawn(300,660,280)&&!shouldSpawn(500,660,280)));
t('rm',()=>ok(rightmostX([])===-Infinity&&rightmostX([{x:1},{x:9}])===9));
t('obox',()=>{const b=getObstacleBox({x:100,y:50,w:40,h:30});ok(b.x>100&&b.w<40&&b.h<30);});
console.log(p+'/'+n+' pure-logic tests passed');
EOF
```
Expected: `SYNTAX_OK` and `7/7 pure-logic tests passed`.

- [ ] **Step 2: Manual acceptance (done by the user on a phone)**

Open `https://gerardoacuna.github.io/dino-vivo/` after deploy and confirm:
- [ ] The ↓ key does nothing; there is no way to duck.
- [ ] Only cacti appear (no birds).
- [ ] A tap anywhere starts, jumps, and restarts.
- [ ] Space/↑ still jumps on desktop.
- [ ] On the phone the game fills the screen width and is fully visible.
- [ ] Tapping does not zoom or scroll the page.
- [ ] The hint reads "Toca la pantalla o Espacio para saltar".
- [ ] `index.html?test=1` shows all tests passing (now includes the cactus-only test).

There is no commit in this task (verification only). Deployment (merge to main + push, which triggers GitHub Pages) is handled in the finishing step after the plan completes.

---

## Self-Review Notes

- **Spec coverage:** remove duck (Task 2), cactus-only / remove birds (Task 1), tap-anywhere control (Task 3), scale-to-width responsive + touch-action + hint text (Task 4), tests updated and re-verified (Tasks 1, 2, 5), deploy noted (Task 5 + finishing step). Every spec section maps to a task.
- **Placeholder scan:** no TBD/TODO; all steps contain exact code and exact commands.
- **Type/name consistency:** after Task 1, an obstacle is `{x, y, w, h}` — `getObstacleBox(o)` (uses `o.x/o.y/o.w/o.h`), the move loop, and the cull filter (`o.x + o.w > 0`) all still match. `makeObstacle()` is now zero-arg and called zero-arg from `spawnObstacle`. `drawObstacles` no longer reads `o.color`/`o.frames`/`o.flip` (all removed). `playerSprite()` no longer reads `game.ducking` (removed everywhere). `onJumpKey()` is reused unchanged by both keydown and pointerdown.
- **YAGNI:** `BIRDC` constant may remain unused after Task 1; harmless to leave (removing it is optional and not required for correctness). No other dead code introduced.
