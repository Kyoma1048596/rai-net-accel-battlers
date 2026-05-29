# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

雷ネット・アクセルバトラーズ (Rai-Net Access Battlers) is a STEINS;GATE-themed
digital board game — a hidden-piece strategy game inspired by the in-anime
"Rai-Net Access Battlers." It is a **single self-contained HTML file** with
inline CSS and JavaScript (no build system, no dependencies, no package
manager). The UI and all in-game text are in Japanese; the visual theme is a
green-on-black terminal/CRT aesthetic.

## Repository Layout

- **`rai-net-access-battlers.html`** — The actual game. This is the file you
  edit for any gameplay, UI, AI, or multiplayer change. Everything lives here:
  `<style>` (lines ~7–226), the HTML body / overlays (~228–783), and the
  `<script>` (~784–2787).
- **`index.html`** — A redirect stub (`<meta http-equiv="refresh">` to the main
  file) that also contains an **older, frozen copy** of the game
  (`GAME_VERSION='2.45'` vs the main file's `'4.0'`). Do **not** make gameplay
  changes here — it is not the source of truth. Only touch it if intentionally
  changing the landing/redirect behavior.
- **`bgm.mp3`, `Surviving_Cyber.mp3`** — Background music tracks loaded by the
  game (title vs battle BGM).

## Running & Testing

There is no build, lint, or test tooling. To run the game, open
`rai-net-access-battlers.html` in a browser, or serve the directory over HTTP
(needed for audio autoplay unlock and any same-WiFi relay testing):

```sh
python3 -m http.server 8000
# then open http://localhost:8000/rai-net-access-battlers.html
```

"Testing" is manual: play through CPU mode (each difficulty), the three board
variants, and online flows. There is no automated test suite — verify changes
by exercising the relevant overlay/flow in the browser. The `GAME_VERSION`
constant near the top of the script (`const GAME_VERSION='4.0'`) is the single
source of truth for the version shown in the UI; bump it when shipping
user-visible changes.

## Architecture

The whole game is procedural global-state JavaScript inside one `<script>`. Key
concepts span multiple functions, so understand these before editing.

### Game state (single global model)
The board and turn state are top-level `let` globals (see ~line 1152):
`board` (a `ROWS × COLS` 2D array of piece objects or `null`), `turn` (1 or 2),
`sel`/`validMoves` (current selection), `captured`, `over`, plus mode globals
`gameMode`, `myPN` (this client's player number), `difficulty`, and
`currentRoomKey`. A **piece** is a plain object `{type, player, value, revealed}`.
Piece `type` is one of `program` (with `value`), `firewall`, `virus`, `linker`,
`medic`, `worm`. `render()` rebuilds the board DOM from this state on every change.

### Board variants
Three modes are selected via `applyVariant(v)` / `setCpuMode` / `setOnlineMode`,
which mutate the geometry globals `ROWS`, `COLS`, `GOAL`, `HOME`:
- **`classic`** — 4×6, 8 pieces.
- **`extended`** — 5×6 (internally `ROWS=7,COLS=5`), 10 pieces, adds the MEDIC
  piece (transforms into a second LINKER on beating a VIRUS).
- **`worm`** — 6×6 (internally `ROWS=8,COLS=6`), 12 pieces, adds the WORM piece
  (chain-destroys adjacent enemy PROGRAMs). Piece placement per variant is in
  `placePieces` / `placePiecesExt` / `placePiecesWorm`.

### Combat resolution
`beats(a, b)` is the single source of truth for the type-matchup rules (VIRUS >
PROGRAM/LINKER, FIREWALL > VIRUS/low PROGRAM, PROGRAM≥3 pierces FIREWALL, LINKER
passes FIREWALL, MEDIC/WORM special cases, PROGRAM-vs-PROGRAM by `value`).
`resolveBattle(att, def)` calls `beats` both ways and returns `'att'`/`'def'`/
`'both'` (mutual destruction). All capture/movement, special MEDIC-transform and
WORM-chain logic, win checks, sound, animation, and the online-send hook live in
`doMove(fr,fc,tr,tc)` — this is the central mutation function. `checkWin()`
decides the result (LINKER reaches goal row, or all enemy LINKERs destroyed).

### AI
CPU opponent is `aiChooseMove()` + `scoreMove()`, gated by `difficulty` (2–5):
levels 2–3 add randomness over top candidates, 4 plays the top move, and 5 adds
a one-ply safety filter (avoid moves that hand the opponent an immediate win).
`scheduleCpu()` triggers the AI after the human's move when `gameMode==='cpu'`.

### Online multiplayer (relay, not P2P)
Players join a room by sharing an "あいことば" (passphrase), hashed to a room key
via `ppToKey`. There is **no WebSocket server** — state is exchanged by polling
three transports in parallel, all keyed off `currentRoomKey`:
1. **Firebase Realtime DB** REST relay (`FB_DB` / `fbPut`/`fbGet`) — the primary
   cross-network transport, polled on intervals.
2. **Local HTTP relay** (`lrPut`/`lrGet`) — only active on LAN hostnames
   (192.168.x etc.); expects a `/relay` endpoint on a same-WiFi PC server.
3. **`BroadcastChannel`** (`'dna-game'`) — same-device tab-to-tab sync.

Each transport carries the same message kinds (room handshake, `placement`,
`move`, `transform`, `sel`, `emote`), distinguished by suffixed keys
(`<room>m1`/`m2` moves, `p1`/`p2` placement, `tf1`/`tf2` transform, `sel1`/`sel2`,
`em1`/`em2`). The host/guest handshake is `startHost` / `startJoin`; incoming
moves run through `applyOppMove` → `doMove` with the `applyingRemote` guard to
avoid echo loops. `lastMoveTs` timestamps dedupe replayed messages.

### Auth, persistence & ratings
Optional Firebase Auth (`FB_API_KEY`, `_fbAuthFetch`, `authLogin`/`authRegister`/
`authGuest`) gates cloud save (`loadCloudData`/`saveCloudData`). Local profile,
stats, and Elo-style rating live in `localStorage` under `dna-*` keys
(`loadPlayerData`/`savePlayerData`, `dna-player-<name>`, `dna-name`,
`dna-auth-*`). Rating changes use `calcRatingDelta`; titles via `getTitle`.
`saveGameSnap`/`loadGameSnap`/`doRestore` persist an in-progress game.

### UI / screens
The app is a stack of fullscreen overlay `<div class="overlay">` elements toggled
by the tiny helpers `show(id)`/`hide(id)` (add/remove the `on` class). Flow:
splash → name → title → (difficulty select | online room) → placement phase →
game → win. Game-over and stats are in `showWin`. There is no router — each
`show*`/overlay function manages its own visibility.

### Audio
All sound effects are **synthesized at runtime** via the WebAudio helpers `tone`
and `noiseBlip` (see the `snd*` arrow functions, e.g. `sndBattle`, `sndWin`).
Browsers require a user gesture to start audio, handled by `_unlockAudio`
(listens for first touch/pointer). BGM uses the two `.mp3` files through
`startTitleBGM`/`startBattleBGM`; volumes persist in `localStorage`
(`dna-bgm-vol`, `dna-se-vol`).

## Conventions

- **Everything stays in the one file.** Match the existing dense, single-line
  style (compact globals, terse helpers, Japanese UI strings and comments). Do
  not introduce a build step, modules, or external dependencies.
- DOM event handlers are wired inline via `onclick="..."` attributes calling
  global functions — keep new handlers consistent with that pattern.
- After any state mutation, call `render()` (and `renderCaptured()` where
  relevant) rather than manually patching the board DOM.
- Route all combat decisions through `beats`/`resolveBattle` and all moves
  through `doMove`; do not duplicate matchup logic elsewhere.
- When adding an online message type, wire it into all three transports
  (Firebase poll, local relay, `BroadcastChannel`) and use the `applyingRemote`
  guard plus a timestamp to prevent echo/duplicate application.

## Git Workflow

Active development branch for this work: `claude/claude-md-docs-n4iiu`. Commit
and push there; do not push to `main` without explicit permission. The upstream
history is mostly "Add files via upload" commits (the game is authored
elsewhere and uploaded), so keep commits scoped and descriptive.
