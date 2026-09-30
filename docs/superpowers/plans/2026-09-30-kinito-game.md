# Kinito Game Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A single-file mobile web page that rolls, hides and reveals two dice for the Kinito pass-the-phone drinking game, tracking the pot.

**Architecture:** All game rules live in a pure reducer inside a `<script id="game">` block in `index.html`, exposed as `Kinito` on the global object. A thin UI layer in a second script block renders state to the DOM and dispatches actions. Tests extract the game block from the HTML and run it in `node:vm`.

**Tech Stack:** Vanilla HTML/CSS/JS, Node 20+ built-in test runner (`node --test`). No dependencies, no build.

**Spec:** `docs/superpowers/specs/2026-09-30-kinito-game-design.md`

## Global Constraints

- Single deliverable file `index.html`; no external scripts or stylesheets.
- Game logic script must not reference `document` or `window` directly (tests run it in a bare vm context).
- Dice values are always passed into the reducer as action payloads; the reducer never calls `Math.random`.
- Score = `max*10 + min`. Kinito = unordered pair in {1:2, 5:6, 6:6}.
- Pot starts at 1, never below 1. Persisted under localStorage key `kinito.pot`.
- Buttons full width, minimum 64px tall. Portrait mobile, no horizontal scroll.
- Screen names exactly: `START`, `HANDOFF`, `ROLLED`, `LIAR_REVEAL`, `KINITO`, `CHALLENGE`, `CHALLENGE_RESULT`.

## Review Focus

1. Double-tapping ROLL: a second `ROLL` while on `ROLLED` must be ignored, not re-roll. (Test in Task 2.)
2. Corrupt localStorage (`"abc"`, `"0"`, `"-3"`, `null`): pot must load as 1, not NaN or 0. (Test `parsePot` in Task 4.)
3. Challenge dice arriving in either order (1,2 vs 2,1) must both count as a hit. (Test in Task 3.)
4. `NEW_ROUND` from a screen other than `LIAR_REVEAL`/`CHALLENGE_RESULT` must be ignored. (Test in Task 3.)
5. A Kinito on the very first roll of a round: previous is null, must still go to `KINITO` and the reducer must not crash. (Test in Task 3.)

---

## File Structure

- `index.html` — the whole app. Three parts in order: `<style>`, markup for every screen, `<script id="game">` (pure logic), `<script id="ui">` (DOM).
- `test/load.js` — extracts the game block from `index.html`, runs it in a vm, exports `Kinito`.
- `test/game.test.js` — all unit tests.
- `package.json` — `"test": "node --test"` only. No dependencies.
- `README.md` — how to play and how to run tests.

---

### Task 1: Test harness, scoring and Kinito detection

**Files:**
- Create: `package.json`
- Create: `index.html` (skeleton with game block only)
- Create: `test/load.js`
- Create: `test/game.test.js`

**Interfaces:**
- Produces: `Kinito.score(a, b) -> number`, `Kinito.isKinito(a, b) -> boolean`, `Kinito.rollDice(rng?) -> [number, number]`.

- [ ] **Step 1: Create package.json**

```json
{
  "name": "kinito",
  "private": true,
  "version": "0.1.0",
  "scripts": {
    "test": "node --test"
  }
}
```

- [ ] **Step 2: Create the loader**

`test/load.js`:

```js
const fs = require('node:fs');
const path = require('node:path');
const vm = require('node:vm');

const html = fs.readFileSync(path.join(__dirname, '..', 'index.html'), 'utf8');
const match = html.match(/<script id="game">([\s\S]*?)<\/script>/);
if (!match) throw new Error('index.html has no <script id="game"> block');

const context = {};
vm.runInNewContext(match[1], context);
if (!context.Kinito) throw new Error('game script did not define Kinito');

module.exports = context.Kinito;
```

- [ ] **Step 3: Write the failing tests**

`test/game.test.js`:

```js
const test = require('node:test');
const assert = require('node:assert/strict');
const Kinito = require('./load');

test('score puts the higher die in the tens column', () => {
  assert.equal(Kinito.score(3, 6), 63);
  assert.equal(Kinito.score(6, 3), 63);
  assert.equal(Kinito.score(4, 4), 44);
  assert.equal(Kinito.score(1, 2), 21);
});

test('isKinito matches 2:1, 6:5, 6:6 in either order', () => {
  for (const [a, b] of [[2, 1], [1, 2], [6, 5], [5, 6], [6, 6]]) {
    assert.equal(Kinito.isKinito(a, b), true, `${a}:${b}`);
  }
  for (const [a, b] of [[6, 4], [1, 1], [3, 6], [5, 5]]) {
    assert.equal(Kinito.isKinito(a, b), false, `${a}:${b}`);
  }
});

test('rollDice uses the supplied rng and returns values 1..6', () => {
  assert.deepEqual(Kinito.rollDice(() => 0), [1, 1]);
  assert.deepEqual(Kinito.rollDice(() => 0.999), [6, 6]);
  const values = [0.1, 0.9];
  let i = 0;
  assert.deepEqual(Kinito.rollDice(() => values[i++]), [1, 6]);
});
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `npm test`
Expected: FAIL with "ENOENT ... index.html" (file does not exist yet).

- [ ] **Step 5: Create index.html skeleton with the game block**

`index.html`:

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Kinito</title>
</head>
<body>
<script id="game">
(function (root) {
  'use strict';

  const KINITO_PAIRS = [[1, 2], [5, 6], [6, 6]];

  function sorted(a, b) {
    return a <= b ? [a, b] : [b, a];
  }

  function score(a, b) {
    const [lo, hi] = sorted(a, b);
    return hi * 10 + lo;
  }

  function isKinito(a, b) {
    const [lo, hi] = sorted(a, b);
    return KINITO_PAIRS.some(([x, y]) => x === lo && y === hi);
  }

  function rollDie(rng) {
    return Math.floor(rng() * 6) + 1;
  }

  function rollDice(rng = Math.random) {
    return [rollDie(rng), rollDie(rng)];
  }

  root.Kinito = { score, isKinito, rollDice };
})(typeof window !== 'undefined' ? window : globalThis);
</script>
</body>
</html>
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `npm test`
Expected: 3 tests pass.

- [ ] **Step 7: Commit**

```bash
git add package.json index.html test/
git commit -m "feat: scoring, kinito detection and test harness"
```

---

### Task 2: Reducer — basic round flow

**Files:**
- Modify: `index.html` (game block)
- Modify: `test/game.test.js`

**Interfaces:**
- Consumes: `isKinito` from Task 1.
- Produces: `Kinito.initialState(pot = 1) -> State`, `Kinito.reduce(state, action) -> State`.
  State: `{ screen, pot, current, previous, attempts, challengeResult, potDrunk }`.
  Actions handled here: `START`, `ROLL {dice}`, `HIDE`, `LIAR`, `NEW_ROUND`, `RESET_POT`.

- [ ] **Step 1: Write the failing tests**

Append to `test/game.test.js`:

```js
const { initialState, reduce } = Kinito;

function run(actions, state = initialState()) {
  return actions.reduce(reduce, state);
}

test('initialState starts on START with pot 1 and nothing rolled', () => {
  assert.deepEqual(initialState(), {
    screen: 'START', pot: 1, current: null, previous: null,
    attempts: [], challengeResult: null, potDrunk: null,
  });
  assert.equal(initialState(3).pot, 3);
});

test('START moves to HANDOFF', () => {
  assert.equal(run([{ type: 'START' }]).screen, 'HANDOFF');
});

test('ROLL on HANDOFF shows the dice on ROLLED', () => {
  const s = run([{ type: 'START' }, { type: 'ROLL', dice: [3, 6] }]);
  assert.equal(s.screen, 'ROLLED');
  assert.deepEqual(s.current, [3, 6]);
});

test('ROLL is ignored when not on HANDOFF (double tap)', () => {
  const rolled = run([{ type: 'START' }, { type: 'ROLL', dice: [3, 6] }]);
  const again = reduce(rolled, { type: 'ROLL', dice: [1, 1] });
  assert.equal(again, rolled);
});

test('HIDE stores the roll as previous and returns to HANDOFF', () => {
  const s = run([{ type: 'START' }, { type: 'ROLL', dice: [3, 6] }, { type: 'HIDE' }]);
  assert.equal(s.screen, 'HANDOFF');
  assert.deepEqual(s.previous, [3, 6]);
  assert.equal(s.current, null);
});

test('LIAR is ignored when there is no previous roll', () => {
  const s = run([{ type: 'START' }]);
  assert.equal(reduce(s, { type: 'LIAR' }), s);
});

test('LIAR reveals the previous roll', () => {
  const s = run([
    { type: 'START' }, { type: 'ROLL', dice: [3, 6] }, { type: 'HIDE' }, { type: 'LIAR' },
  ]);
  assert.equal(s.screen, 'LIAR_REVEAL');
  assert.deepEqual(s.previous, [3, 6]);
});

test('NEW_ROUND after LIAR_REVEAL clears previous and returns to HANDOFF', () => {
  const s = run([
    { type: 'START' }, { type: 'ROLL', dice: [3, 6] }, { type: 'HIDE' },
    { type: 'LIAR' }, { type: 'NEW_ROUND' },
  ]);
  assert.equal(s.screen, 'HANDOFF');
  assert.equal(s.previous, null);
  assert.equal(s.current, null);
});

test('RESET_POT only works on START', () => {
  assert.equal(reduce(initialState(4), { type: 'RESET_POT' }).pot, 1);
  const inGame = run([{ type: 'START' }], initialState(4));
  assert.equal(reduce(inGame, { type: 'RESET_POT' }).pot, 4);
});

test('unknown actions return the same state object', () => {
  const s = initialState();
  assert.equal(reduce(s, { type: 'NOPE' }), s);
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test`
Expected: new tests FAIL with "initialState is not a function" / "reduce is not a function".

- [ ] **Step 3: Implement initialState and reduce**

In the game block, after `rollDice`, add:

```js
  function initialState(pot = 1) {
    return {
      screen: 'START',
      pot,
      current: null,
      previous: null,
      attempts: [],
      challengeResult: null,
      potDrunk: null,
    };
  }

  function reduce(s, action) {
    switch (action.type) {
      case 'START':
        return s.screen === 'START' ? { ...s, screen: 'HANDOFF' } : s;

      case 'ROLL': {
        if (s.screen !== 'HANDOFF') return s;
        const [a, b] = action.dice;
        return { ...s, current: action.dice, screen: isKinito(a, b) ? 'KINITO' : 'ROLLED' };
      }

      case 'HIDE':
        if (s.screen !== 'ROLLED') return s;
        return { ...s, screen: 'HANDOFF', previous: s.current, current: null };

      case 'LIAR':
        if (s.screen !== 'HANDOFF' || s.previous === null) return s;
        return { ...s, screen: 'LIAR_REVEAL' };

      case 'NEW_ROUND':
        if (s.screen !== 'LIAR_REVEAL' && s.screen !== 'CHALLENGE_RESULT') return s;
        return {
          ...s, screen: 'HANDOFF', current: null, previous: null,
          attempts: [], challengeResult: null, potDrunk: null,
        };

      case 'RESET_POT':
        return s.screen === 'START' ? { ...s, pot: 1 } : s;

      default:
        return s;
    }
  }
```

Update the export line:

```js
  root.Kinito = { score, isKinito, rollDice, initialState, reduce };
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npm test`
Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add index.html test/game.test.js
git commit -m "feat: reducer for roll, hide, liar and new round"
```

---

### Task 3: Reducer — Kinito and the challenge

**Files:**
- Modify: `index.html` (game block, inside `reduce`)
- Modify: `test/game.test.js`

**Interfaces:**
- Consumes: `reduce`, `initialState`, `isKinito`.
- Produces: actions `BEGIN_CHALLENGE`, `CHALLENGE_ROLL {dice}`. State fields `attempts`, `challengeResult` (`'HIT' | 'MISS' | null`), `potDrunk` (number of shots drunk on a MISS, else null).

- [ ] **Step 1: Write the failing tests**

Append to `test/game.test.js`:

```js
test('rolling a Kinito skips ROLLED and goes straight to KINITO', () => {
  const s = run([{ type: 'START' }, { type: 'ROLL', dice: [6, 5] }]);
  assert.equal(s.screen, 'KINITO');
  assert.deepEqual(s.current, [6, 5]);
});

test('Kinito on the first roll of a round works with previous null', () => {
  const s = run([{ type: 'START' }, { type: 'ROLL', dice: [2, 1] }]);
  assert.equal(s.screen, 'KINITO');
  assert.equal(s.previous, null);
});

test('BEGIN_CHALLENGE moves to CHALLENGE with no attempts', () => {
  const s = run([{ type: 'START' }, { type: 'ROLL', dice: [6, 6] }, { type: 'BEGIN_CHALLENGE' }]);
  assert.equal(s.screen, 'CHALLENGE');
  assert.deepEqual(s.attempts, []);
});

test('BEGIN_CHALLENGE is ignored off the KINITO screen', () => {
  const s = run([{ type: 'START' }]);
  assert.equal(reduce(s, { type: 'BEGIN_CHALLENGE' }), s);
});

const toChallenge = [
  { type: 'START' }, { type: 'ROLL', dice: [6, 6] }, { type: 'BEGIN_CHALLENGE' },
];

test('a miss records the attempt and stays on CHALLENGE', () => {
  const s = run([...toChallenge, { type: 'CHALLENGE_ROLL', dice: [3, 4] }]);
  assert.equal(s.screen, 'CHALLENGE');
  assert.deepEqual(s.attempts, [[3, 4]]);
  assert.equal(s.pot, 1);
});

test('a hit on any attempt adds one shot to the pot', () => {
  for (const misses of [0, 1, 2]) {
    const actions = [...toChallenge];
    for (let i = 0; i < misses; i++) actions.push({ type: 'CHALLENGE_ROLL', dice: [1, 1] });
    actions.push({ type: 'CHALLENGE_ROLL', dice: [1, 2] });
    const s = run(actions, initialState(2));
    assert.equal(s.screen, 'CHALLENGE_RESULT', `after ${misses} misses`);
    assert.equal(s.challengeResult, 'HIT');
    assert.equal(s.pot, 3);
    assert.equal(s.attempts.length, misses + 1);
  }
});

test('challenge hit counts dice in either order', () => {
  const a = run([...toChallenge, { type: 'CHALLENGE_ROLL', dice: [2, 1] }]);
  const b = run([...toChallenge, { type: 'CHALLENGE_ROLL', dice: [1, 2] }]);
  assert.equal(a.challengeResult, 'HIT');
  assert.equal(b.challengeResult, 'HIT');
});

test('three misses drinks the pot and resets it to 1', () => {
  const s = run([
    ...toChallenge,
    { type: 'CHALLENGE_ROLL', dice: [1, 1] },
    { type: 'CHALLENGE_ROLL', dice: [3, 3] },
    { type: 'CHALLENGE_ROLL', dice: [4, 6] },
  ], initialState(3));
  assert.equal(s.screen, 'CHALLENGE_RESULT');
  assert.equal(s.challengeResult, 'MISS');
  assert.equal(s.potDrunk, 3);
  assert.equal(s.pot, 1);
});

test('CHALLENGE_ROLL is ignored once the challenge has resolved', () => {
  const done = run([...toChallenge, { type: 'CHALLENGE_ROLL', dice: [6, 6] }]);
  assert.equal(reduce(done, { type: 'CHALLENGE_ROLL', dice: [6, 6] }), done);
});

test('NEW_ROUND after a challenge clears attempts, result and potDrunk but keeps pot', () => {
  const s = run([...toChallenge, { type: 'CHALLENGE_ROLL', dice: [6, 5] }, { type: 'NEW_ROUND' }]);
  assert.equal(s.screen, 'HANDOFF');
  assert.equal(s.pot, 2);
  assert.deepEqual(s.attempts, []);
  assert.equal(s.challengeResult, null);
  assert.equal(s.potDrunk, null);
  assert.equal(s.previous, null);
});

test('NEW_ROUND is ignored from HANDOFF, ROLLED, KINITO and CHALLENGE', () => {
  const handoff = run([{ type: 'START' }]);
  assert.equal(reduce(handoff, { type: 'NEW_ROUND' }), handoff);
  const rolled = run([{ type: 'START' }, { type: 'ROLL', dice: [3, 4] }]);
  assert.equal(reduce(rolled, { type: 'NEW_ROUND' }), rolled);
  const kinito = run([{ type: 'START' }, { type: 'ROLL', dice: [6, 6] }]);
  assert.equal(reduce(kinito, { type: 'NEW_ROUND' }), kinito);
  const challenge = run(toChallenge);
  assert.equal(reduce(challenge, { type: 'NEW_ROUND' }), challenge);
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test`
Expected: Kinito/challenge tests FAIL (screen stays `KINITO` or `CHALLENGE` because the actions are unhandled).

- [ ] **Step 3: Add the two cases to `reduce`**

Insert before `case 'NEW_ROUND':`:

```js
      case 'BEGIN_CHALLENGE':
        if (s.screen !== 'KINITO') return s;
        return { ...s, screen: 'CHALLENGE', attempts: [] };

      case 'CHALLENGE_ROLL': {
        if (s.screen !== 'CHALLENGE') return s;
        const [a, b] = action.dice;
        const attempts = [...s.attempts, action.dice];
        if (isKinito(a, b)) {
          return { ...s, attempts, screen: 'CHALLENGE_RESULT', challengeResult: 'HIT', pot: s.pot + 1 };
        }
        if (attempts.length >= 3) {
          return {
            ...s, attempts, screen: 'CHALLENGE_RESULT', challengeResult: 'MISS',
            potDrunk: s.pot, pot: 1,
          };
        }
        return { ...s, attempts };
      }
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npm test`
Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add index.html test/game.test.js
git commit -m "feat: kinito challenge and pot handling in reducer"
```

---

### Task 4: Pot persistence helper

**Files:**
- Modify: `index.html` (game block)
- Modify: `test/game.test.js`

**Interfaces:**
- Produces: `Kinito.parsePot(raw) -> number` — accepts whatever came out of localStorage, returns an integer ≥ 1, defaulting to 1.

- [ ] **Step 1: Write the failing test**

Append to `test/game.test.js`:

```js
test('parsePot falls back to 1 for anything that is not a positive integer', () => {
  assert.equal(Kinito.parsePot('3'), 3);
  assert.equal(Kinito.parsePot('1'), 1);
  for (const bad of [null, undefined, '', 'abc', '0', '-3', '2.5', 'NaN']) {
    assert.equal(Kinito.parsePot(bad), 1, String(bad));
  }
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test`
Expected: FAIL with "Kinito.parsePot is not a function".

- [ ] **Step 3: Implement parsePot**

In the game block, before the export:

```js
  function parsePot(raw) {
    const n = Number(raw);
    return Number.isInteger(n) && n >= 1 ? n : 1;
  }
```

Export line becomes:

```js
  root.Kinito = { score, isKinito, rollDice, initialState, reduce, parsePot };
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npm test`
Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add index.html test/game.test.js
git commit -m "feat: parsePot guards localStorage input"
```

---

### Task 5: UI — markup, styles, render and dispatch

**Files:**
- Modify: `index.html` (add `<style>`, body markup, `<script id="ui">`)

**Interfaces:**
- Consumes: everything on `Kinito` from Tasks 1–4.
- Produces: the working page. No new exports.

Dice are rendered with the Unicode die faces U+2680–U+2685, coloured with CSS.

- [ ] **Step 1: Add styles inside `<head>` after `<title>`**

```html
<style>
  :root {
    --bg: #14121a;
    --panel: #221e2e;
    --text: #f4f1ff;
    --muted: #a79fc4;
    --accent: #ff4d6d;
    --accent2: #ffd166;
    --ok: #3ddc97;
  }
  * { box-sizing: border-box; }
  html, body { margin: 0; height: 100%; }
  body {
    background: var(--bg);
    color: var(--text);
    font-family: -apple-system, system-ui, "Segoe UI", Roboto, sans-serif;
    display: flex;
    flex-direction: column;
    min-height: 100dvh;
    padding: env(safe-area-inset-top) 16px env(safe-area-inset-bottom);
    overflow-x: hidden;
  }
  header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 16px 0;
    font-weight: 700;
    color: var(--muted);
  }
  header .pot { color: var(--accent2); }
  main { flex: 1; display: flex; flex-direction: column; }
  section { display: none; flex: 1; flex-direction: column; justify-content: center; gap: 24px; text-align: center; }
  body[data-screen="START"] #s-start,
  body[data-screen="HANDOFF"] #s-handoff,
  body[data-screen="ROLLED"] #s-rolled,
  body[data-screen="LIAR_REVEAL"] #s-liar,
  body[data-screen="KINITO"] #s-kinito,
  body[data-screen="CHALLENGE"] #s-challenge,
  body[data-screen="CHALLENGE_RESULT"] #s-result { display: flex; }
  h1 { font-size: 3rem; margin: 0; letter-spacing: 0.05em; }
  h2 { font-size: 1.6rem; margin: 0; }
  p { font-size: 1.2rem; color: var(--muted); margin: 0; }
  .dice { font-size: 7rem; line-height: 1; letter-spacing: 0.1em; }
  .score { font-size: 4rem; font-weight: 800; }
  .actions { display: flex; flex-direction: column; gap: 12px; padding-bottom: 16px; }
  button {
    width: 100%;
    min-height: 64px;
    border: 0;
    border-radius: 16px;
    font-size: 1.4rem;
    font-weight: 700;
    background: var(--panel);
    color: var(--text);
    touch-action: manipulation;
    -webkit-tap-highlight-color: transparent;
  }
  button:active { transform: scale(0.98); }
  button.primary { background: var(--accent); }
  button.warn { background: var(--accent2); color: #1a1400; }
  button.ok { background: var(--ok); color: #003321; }
  button[hidden] { display: none; }
  .flash { animation: flash 0.6s steps(2) infinite; color: var(--accent2); }
  @keyframes flash { to { opacity: 0.2; } }
</style>
```

- [ ] **Step 2: Add the markup at the top of `<body>`, before the game script**

```html
<header>
  <span>KINITO</span>
  <span class="pot" id="pot"></span>
</header>

<main>
  <section id="s-start">
    <h1>KINITO</h1>
    <p>Two dice. One pot. Don't get caught.</p>
    <div class="actions">
      <button class="primary" data-action="START">START</button>
      <button data-action="RESET_POT">RESET POT</button>
    </div>
  </section>

  <section id="s-handoff">
    <h2>Pass the phone</h2>
    <p>Roll, or call the last player out.</p>
    <div class="actions">
      <button class="primary" data-action="ROLL">ROLL</button>
      <button class="warn" data-action="LIAR" id="liar-btn">LIAR!</button>
    </div>
  </section>

  <section id="s-rolled">
    <div class="dice" id="rolled-dice"></div>
    <div class="score" id="rolled-score"></div>
    <p>Say a number. Then hide it.</p>
    <div class="actions">
      <button class="primary" data-action="HIDE">HIDE &amp; PASS</button>
    </div>
  </section>

  <section id="s-liar">
    <h2>They had…</h2>
    <div class="dice" id="liar-dice"></div>
    <div class="score" id="liar-score"></div>
    <p>Who lied? Loser drinks.</p>
    <div class="actions">
      <button class="primary" data-action="NEW_ROUND">NEW ROUND</button>
    </div>
  </section>

  <section id="s-kinito">
    <h1 class="flash">KINITO!!!</h1>
    <div class="dice" id="kinito-dice"></div>
    <p>Player on the right: 3 rolls to hit a Kinito.</p>
    <div class="actions">
      <button class="warn" data-action="BEGIN_CHALLENGE">CHALLENGE</button>
    </div>
  </section>

  <section id="s-challenge">
    <h2 id="attempt-label"></h2>
    <div class="dice" id="challenge-dice"></div>
    <p>Need 2:1, 6:5 or 6:6.</p>
    <div class="actions">
      <button class="primary" data-action="CHALLENGE_ROLL">ROLL</button>
    </div>
  </section>

  <section id="s-result">
    <h1 id="result-title"></h1>
    <div class="dice" id="result-dice"></div>
    <p id="result-text"></p>
    <div class="actions">
      <button class="primary" data-action="NEW_ROUND">NEW ROUND</button>
    </div>
  </section>
</main>
```

- [ ] **Step 3: Add the UI script after the game script, before `</body>`**

```html
<script id="ui">
(function () {
  'use strict';
  const { score, rollDice, initialState, reduce, parsePot } = window.Kinito;
  const POT_KEY = 'kinito.pot';
  const FACES = ['', '⚀', '⚁', '⚂', '⚃', '⚄', '⚅'];

  function loadPot() {
    try { return parsePot(localStorage.getItem(POT_KEY)); } catch (e) { return 1; }
  }
  function savePot(pot) {
    try { localStorage.setItem(POT_KEY, String(pot)); } catch (e) { /* in-memory only */ }
  }

  let state = initialState(loadPot());

  const $ = (id) => document.getElementById(id);
  const faces = (dice) => dice ? dice.map((d) => FACES[d]).join(' ') : '';
  const shots = (n) => `${n} shot${n === 1 ? '' : 's'}`;

  function render(s) {
    document.body.dataset.screen = s.screen;
    $('pot').textContent = `Pot: ${shots(s.pot)}`;

    $('liar-btn').hidden = s.previous === null;

    $('rolled-dice').textContent = faces(s.current);
    $('rolled-score').textContent = s.current ? score(...s.current) : '';

    $('liar-dice').textContent = faces(s.previous);
    $('liar-score').textContent = s.previous ? score(...s.previous) : '';

    $('kinito-dice').textContent = faces(s.current);

    const last = s.attempts[s.attempts.length - 1] || null;
    $('attempt-label').textContent = `Attempt ${Math.min(s.attempts.length + 1, 3)} of 3`;
    $('challenge-dice').textContent = faces(last);

    if (s.challengeResult === 'HIT') {
      $('result-title').textContent = 'KINITO!';
      $('result-text').textContent = `Pot is now ${shots(s.pot)}.`;
    } else if (s.challengeResult === 'MISS') {
      $('result-title').textContent = 'Drink the pot!';
      $('result-text').textContent = `${shots(s.potDrunk)} down the hatch. Pot resets to 1.`;
    }
    $('result-dice').textContent = faces(last);
  }

  function dispatch(action) {
    state = reduce(state, action);
    savePot(state.pot);
    render(state);
  }

  document.addEventListener('click', (e) => {
    const btn = e.target.closest('button[data-action]');
    if (!btn) return;
    const type = btn.dataset.action;
    if (type === 'ROLL' || type === 'CHALLENGE_ROLL') {
      dispatch({ type, dice: rollDice() });
    } else {
      dispatch({ type });
    }
  });

  render(state);
})();
</script>
```

- [ ] **Step 4: Run unit tests to confirm the game block is untouched**

Run: `npm test`
Expected: all tests pass.

- [ ] **Step 5: Smoke test in a browser**

Run: `open index.html`

Check, in order:
1. START screen shows "Pot: 1 shot". START → HANDOFF, LIAR button hidden.
2. ROLL → two dice and score shown. HIDE & PASS → HANDOFF, LIAR now visible.
3. LIAR → previous dice shown. NEW ROUND → HANDOFF, LIAR hidden again.
4. Keep rolling until a Kinito appears (about 1 in 12 rolls) → flashing KINITO screen, CHALLENGE → "Attempt 1 of 3", ROLL up to three times → result screen, pot header updates.
5. Refresh the page: pot value survives. RESET POT on START sets it back to 1.
6. In devtools, toggle a phone viewport (iPhone 13 or similar): no horizontal scroll, all buttons reachable.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: mobile UI for kinito game"
```

---

### Task 6: README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write the README**

```markdown
# Kinito

Pass-the-phone dice drinking game. One phone, two dice, one pot.

## Play

Open `index.html` on a phone (or host it anywhere static).

1. Put one shot in a glass. That's the pot.
2. Tap ROLL. Your score is the higher die then the lower: 3 and 6 is 63.
3. Say a number out loud (truth or bluff), tap HIDE & PASS, hand it left.
4. Next player either tap ROLL and must claim higher, or taps LIAR! to
   reveal the last roll. Whoever was wrong drinks from their own glass.
5. Rolling 2:1, 6:5 or 6:6 is a KINITO. The player on the right gets three
   open rolls to hit any Kinito. Hit: add a shot to the pot. Miss: drink it.

The app tracks the pot and remembers it between refreshes.

## Develop

No build, no dependencies. Requires Node 20+ for tests.

    npm test
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README"
```
