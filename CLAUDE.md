# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project shape

This repo is **a single self-contained HTML file**: `rai-net-access-battlers.html`. All HTML markup, CSS, and JavaScript live in that one file — there is no `package.json`, no build step, no test suite, no linter, and no other source files. The repository name (`rai-net-accel-battlers`) and the file name (`rai-net-access-battlers.html`) differ by one letter; do not "fix" one to match the other without an explicit request — links to the hosted file likely depend on the current name.

The game is a Steins;Gate-themed 2-player strategy game ("電ネット・アクセルバトラーズ" / "DEN-NET ACCEL BATTLERS") played on a 4×6 grid with hidden-info pieces (LINKER, PROGRAM P2/P3/P4, FIREWALL, VIRUS). UI text is Japanese.

## Running and testing

There is no dev server, bundler, or test runner. To try changes:

- Open `rai-net-access-battlers.html` directly in a browser (file:// works — Gun.js is loaded from a CDN, so an internet connection is required).
- For online-mode testing, open the file in **two browser windows** (or two browsers) and use HOST / JOIN with the same passphrase. The two clients rendezvous through public Gun.js relays (no local server needed).

Because there are no automated checks, when changing game logic, manually exercise: CPU mode at all three difficulties, online HOST flow, online JOIN flow, win/loss/draw overlays, and the "return to room" button (host-only post-game restart).

## Architecture

Everything below refers to `rai-net-access-battlers.html`.

### Game state (module-level globals, ~line 326)

The whole game is driven by a small set of mutable globals: `board` (6×4 array of piece objects or `null`), `turn` (1 or 2), `sel` (selected `[r,c]`), `validMoves`, `over`, `captured` (`{1:[],2:[]}`), `cpuThinking`, `difficulty` (1–3), `gameMode` (`'cpu'` | `'online'`), `myRole` (`'host'` | `'guest'`), `myPN` (this client's player number). Pieces are plain objects: `{player, type, value, revealed}` where `type ∈ {program, firewall, virus, linker}` and `value` is only meaningful for programs (2/3/4).

### Battle / win rules (`beats`, `resolveBattle`, `checkWin`)

`beats(a, b)` encodes the rock-paper-scissors-ish matrix: VIRUS beats PROGRAM & LINKER; FIREWALL beats VIRUS and P2 only; P3/P4 pierce FIREWALL; PROG-vs-PROG by value; LINKER passes through FIREWALL. `resolveBattle` returns `'att' | 'def' | 'both'` (both = mutual destruction). `checkWin` returns `1`, `2`, `'draw'`, or `0` — a player wins by either getting their LINKER to the opponent's goal row (`GOAL = {1:0, 2:5}`) **or** by destroying the opponent's LINKER.

### Move flow (`doMove`)

`doMove(fr,fc,tr,tc)` is the single chokepoint for all moves (player click, CPU, and remote network moves all funnel here). It mutates `board` and `captured`, reveals pieces on combat, checks for a win, flips `turn`, re-renders, and then either schedules the CPU or broadcasts to the network. **Key invariant:** `applyingRemote` is set to `true` while replaying a remote move so the move is not echoed back over the wire (the `gameMode==='online' && !applyingRemote` guards at the end of `doMove` and the `applyingRemote` check inside the Gun `.on` listener implement this). Preserve this guard when editing — without it, online play infinite-loops.

### CPU AI (`scheduleCpu`, `aiChooseMove`, `scoreMove`)

The AI enumerates every legal move for player 2, scores each via `scoreMove`, and picks by difficulty:
- **1 (初級):** 30% chance to pick the top move, otherwise uniform random.
- **2 (中級):** Weighted pick over the top 3 (0.65 / 0.25 / 0.10).
- **3 (上級):** Top-2 with 0.85 bias toward the best; adds linker-safety lookahead (penalty for moving adjacent to a revealed piece that beats it) and a bonus for getting close to a revealed enemy linker.

`scoreMove` is heuristic — forward movement, type-matchup bonuses, capture rewards weighted by piece value, and difficulty-scaled random noise (more noise at lower difficulty). Changing the scoring function is the main lever for tuning CPU behaviour.

### Online multiplayer (Gun.js relay)

There is **no peer-to-peer** in the current code despite a vestigial reference (`gameConn` at line 663 is undeclared — leftover from an earlier PeerJS-based version; safe to leave alone but be aware if touching `showWin`). All sync goes through public Gun.js relays declared at line 305.

The room layout, keyed off a lowercased passphrase via `ppToKey(pp)`:
- `roomKey` — host metadata `{hostName, board, ts, status}` where `status ∈ {waiting, playing, abandoned}`.
- `roomKey-g` — guest signal `{guestName, ts}`. Host's `.on` listener for this node is what transitions the game to `'playing'`.
- `roomKey-m1`, `roomKey-m2` — per-player move channels. Each client writes only to **its own** `myMvNode()` and subscribes only to `oppMvNode()`. Messages are `{fr, fc, tr, tc, ts}`; the receiver dedupes via `ts <= lastMoveTs`.

The host generates the initial board and pushes it as JSON; the guest deserializes and adopts it. Player numbering is fixed by role (host = P1, guest = P2). `returnToRoom()` lets the host start a fresh round in the same room after a game ends.

### Rendering

`render()` rebuilds the board DOM from scratch every move (no diffing — the board is tiny). Piece visibility is computed per-cell: `p.player === myPN || p.revealed` — opponents' un-revealed pieces render as the "hidden" style. When editing visibility logic, remember this is also what hides remote pieces in online mode (the comparison uses `myPN`, not a fixed `1`).

### UI / overlays

Multiple `.overlay` elements (`ov-name`, `ov-title`, `ov-difficulty`, `ov-rules`, `ov-online`, `ov-waiting`, `ov-win`) are toggled via `show(id)` / `hide(id)`, which add/remove the `on` class. `showTitle()` is the canonical "reset UI" path — it hides every overlay, cancels CPU timers, and calls `cleanupOnline()`. F2 is bound globally to `showTitle()`.

### Persistence

Only `localStorage` — keys are `dna-name` (player name) and `dna-uid` (per-browser ID, generated lazily). Gun's own `localStorage` is **disabled** (`localStorage: false` in the `Gun(...)` config) so room state is relay-side only and doesn't pollute the browser.

### Audio

Web Audio API, created lazily in `getAC()` (browsers require a user gesture before audio works — `getAC` resumes a suspended context). Sound helpers (`sndMove`, `sndBattle`, `sndVictory`, …) are short oscillator/noise blips; no external assets.

## Git workflow

The repository's main branch contains the file directly. When asked to make changes, follow the branching instructions in the task prompt — do not push to the default branch. Recent commit messages are short, imperative, and English ("Refactor online match handling and UI updates"); match that style.
