# Kinito — pass-the-phone dice drinking game

## Purpose

A mobile web game for a group sitting around a table with one phone. The app
rolls two dice, hides them, and reveals them on demand. All bluffing and
"who drinks" decisions are made by the humans; the app never knows who is
holding it.

## Rules the app enforces

- **Score**: two dice, higher die is the tens digit. 3 and 6 = 63. Ties (e.g. 4:4 = 44).
- **Kinito**: any of 2:1, 6:5, 6:6. Checked on the unordered pair.
- **Turn**: player receives phone, chooses ROLL or LIAR!. After rolling, sees
  dice + score, presses HIDE & PASS. Spoken claim is not entered.
- **LIAR!**: available only when a previous hidden roll exists in this round.
  Reveals the previous roll's dice and score. Humans decide who drinks.
  Round then resets.
- **Kinito roll**: dice shown to all immediately (no hide step). The player
  to the right of the roller (i.e. the previous player) gets 3 open rolls.
  Any Kinito combo counts as a hit.
  - Hit: pot +1 shot. The challenge passes back to the other player for
    three rolls of their own, and bounces back and forth until someone
    misses. Someone always drinks the pot.
  - Three misses: "Drink the pot!". Pot resets to 0. Round resets.
- **Pot**: starts empty (0). The first Kinito rolled pours in 1 shot. After
  someone drinks it, it is empty (0) again until the next Kinito. Displayed always. Persisted in localStorage.
  Reset button on start screen.

## Non-goals

- No player names or turn tracking.
- No entering spoken claims.
- No networking, accounts, or multi-device play.
- No animation beyond a simple dice "shake" if cheap.

## Architecture

Single `index.html`, vanilla JS and CSS, no build step, no dependencies.
Portrait mobile layout, buttons full-width and at least 64px tall.

### Game logic (pure, testable)

A pure module inside the page, exposed as `window.Kinito`:

```
score(a, b)            -> int            max*10 + min
isKinito(a, b)         -> bool
rollDice(rng)          -> [a, b]         rng defaults to Math.random
reduce(state, action)  -> newState       pure state transition
initialState(pot=1)    -> state
```

State shape:

```
{
  screen: 'START'|'HANDOFF'|'ROLLED'|'LIAR_REVEAL'|'KINITO'|'CHALLENGE'|'CHALLENGE_RESULT',
  pot: int,
  current: [a,b] | null,     // roll just made, shown on ROLLED/KINITO
  previous: [a,b] | null,    // last hidden roll this round; null => LIAR hidden
  attempts: [[a,b], ...],    // challenge rolls so far (max 3)
  challengeResult: 'HIT'|'MISS'|null,
  potDrunk: int | null       // shots drunk on a MISS, so the result screen can say so after pot resets
}
```

Actions:

```
START            START -> HANDOFF
ROLL {dice}      HANDOFF -> ROLLED (or KINITO if isKinito)
HIDE             ROLLED -> HANDOFF, previous = current
LIAR             HANDOFF -> LIAR_REVEAL (requires previous != null)
BEGIN_CHALLENGE  KINITO -> CHALLENGE, attempts = []
CHALLENGE_ROLL {dice}
                 CHALLENGE -> CHALLENGE (miss, <3) |
                              CHALLENGE_RESULT HIT (pot+1) |
                              CHALLENGE_RESULT MISS after 3 (potDrunk=pot, pot=1)
NEW_ROUND        LIAR_REVEAL|CHALLENGE_RESULT -> HANDOFF, previous=null
RESET_POT        pot = 1 (START only)
```

Dice values are passed in as action payload so the reducer stays pure.

### UI layer

`render(state)` maps state to DOM: sets a `data-screen` attribute on
`<body>`; CSS shows the matching section. Buttons dispatch actions; ROLL
and CHALLENGE_ROLL generate dice via `rollDice()` then dispatch. After
every dispatch, `pot` is written to localStorage.

Screens and copy:

| screen | shows | buttons |
|---|---|---|
| START | title, "Pot: N shot(s)" | START, RESET POT |
| HANDOFF | "Pass the phone", pot | ROLL, LIAR! (hidden if previous null) |
| ROLLED | two dice, score large | HIDE & PASS |
| LIAR_REVEAL | previous dice + score, "Who lied? Loser drinks." | NEW ROUND |
| KINITO | flashing "KINITO!!!", dice, "Player on the right: 3 rolls to hit a Kinito" | CHALLENGE |
| CHALLENGE | dice of last attempt, "Attempt k/3" | ROLL |
| CHALLENGE_RESULT | HIT: "Pot is now N shots!" / MISS: "Drink the pot!" | NEW ROUND |

Pot shown in a persistent header on every screen.

## Error handling

- localStorage unavailable: fall back to in-memory pot, no error shown.
- Invalid action for current screen: reducer returns state unchanged.

## Testing

No automated tests. Manual check in iOS Safari: every screen reachable,
buttons one-handed, pot survives refresh, no horizontal scroll.

## Deployment

Open `index.html` directly, or host on GitHub Pages / Cloudflare Pages
later. Out of scope for the first build.
