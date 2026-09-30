# Kinito Game Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A single-file mobile web page that rolls, hides and reveals two dice for the Kinito pass-the-phone drinking game, tracking the pot.

**Architecture:** All game rules live in a pure reducer inside a `<script id="game">` block in `index.html`, exposed as `Kinito` on the global object. A thin UI layer in a second script block renders state to the DOM and dispatches actions.

**Tech Stack:** Vanilla HTML/CSS/JS. No dependencies, no build, no automated tests (owner's choice). Each task ends with a browser smoke check instead.

**Spec:** `docs/superpowers/specs/2026-09-30-kinito-game-design.md`

## Global Constraints

- Single deliverable file `index.html`; no external scripts or stylesheets.
- Game logic script must not reference `document` or `window` directly.
- Dice values are always passed into the reducer as action payloads; the reducer never calls `Math.random`.
- Score = `max*10 + min`. Kinito = unordered pair in {1:2, 5:6, 6:6}.
- Pot starts at 1, never below 1. Persisted under localStorage key `kinito.pot`.
- Buttons full width, minimum 64px tall. Portrait mobile, no horizontal scroll.
- Screen names exactly: `START`, `HANDOFF`, `ROLLED`, `LIAR_REVEAL`, `KINITO`, `CHALLENGE`, `CHALLENGE_RESULT`.

## Review Focus

Behaviours to check by hand in the browser at the end (no unit tests by owner's choice):

1. Double-tapping ROLL must not re-roll: reducer ignores `ROLL` unless on `HANDOFF`.
2. Corrupt localStorage (`"abc"`, `"0"`, `"-3"`): pot loads as 1, never NaN or 0.
3. Challenge dice in either order (1,2 vs 2,1) both count as a hit.
4. `NEW_ROUND` from any screen other than `LIAR_REVEAL`/`CHALLENGE_RESULT` is ignored.
5. Kinito on the very first roll of a round (previous null) still reaches `KINITO`.

---

## File Structure

- `index.html` — the whole app. Three parts in order: `<style>`, markup for every screen, `<script id="game">` (pure logic), `<script id="ui">` (DOM).
- `README.md` — how to play.

---

### Task 1: Game logic block

**Files:**
- Create: `index.html` (skeleton + `<script id="game">`)

**Interfaces:**
- Produces: `Kinito.score(a, b)`, `Kinito.isKinito(a, b)`, `Kinito.rollDice(rng?)`, `Kinito.initialState(pot = 1)`, `Kinito.reduce(state, action)`, `Kinito.parsePot(raw)`.
  State: `{ screen, pot, current, previous, attempts, challengeResult, potDrunk }`.
  Actions: `START`, `ROLL {dice}`, `HIDE`, `LIAR`, `BEGIN_CHALLENGE`, `CHALLENGE_ROLL {dice}`, `NEW_ROUND`, `RESET_POT`.

- [ ] **Step 1: Create index.html with the game block**

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

  function parsePot(raw) {
    const n = Number(raw);
    return Number.isInteger(n) && n >= 1 ? n : 1;
  }

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

  root.Kinito = { score, isKinito, rollDice, parsePot, initialState, reduce };
})(typeof window !== 'undefined' ? window : globalThis);
</script>
</body>
</html>
```

- [ ] **Step 2: Sanity check in the browser console**

Run: `open index.html`, open devtools console, paste:

```js
Kinito.score(3, 6)            // 63
Kinito.isKinito(5, 6)         // true
Kinito.parsePot('abc')        // 1
Kinito.reduce(Kinito.initialState(), { type: 'START' }).screen   // 'HANDOFF'
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: kinito game logic"
```

---

### Task 2: UI — markup, styles, render and dispatch

**Files:**
- Modify: `index.html` (add `<style>`, body markup, `<script id="ui">`)

**Interfaces:**
- Consumes: everything on `Kinito` from Task 1.

Dice are rendered with the Unicode die faces U+2680–U+2685.

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

- [ ] **Step 4: Smoke test in a browser**

Run: `open index.html`

Check, in order:
1. START screen shows "Pot: 1 shot". START → HANDOFF, LIAR button hidden.
2. ROLL → two dice and score shown. HIDE & PASS → HANDOFF, LIAR now visible.
3. LIAR → previous dice shown. NEW ROUND → HANDOFF, LIAR hidden again.
4. Keep rolling until a Kinito appears (about 1 in 12 rolls) → flashing KINITO screen, CHALLENGE → "Attempt 1 of 3", ROLL up to three times → result screen, pot header updates.
5. Refresh the page: pot value survives. RESET POT on START sets it back to 1.
6. In devtools console: `localStorage.setItem('kinito.pot','abc')`, refresh → "Pot: 1 shot".
7. Toggle a phone viewport (iPhone 13 or similar): no horizontal scroll, all buttons reachable.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: mobile UI for kinito game"
```

---

### Task 3: README

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
4. Next player either taps ROLL and must claim higher, or taps LIAR! to
   reveal the last roll. Whoever was wrong drinks from their own glass.
5. Rolling 2:1, 6:5 or 6:6 is a KINITO. The player on the right gets three
   open rolls to hit any Kinito. Hit: add a shot to the pot. Miss: drink it.

The app tracks the pot and remembers it between refreshes.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README"
```
