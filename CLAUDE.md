# CLAUDE.md

Guidance for AI assistants (and humans) working in this repository.

## What this project is

**雷ネット・アクセルバトラーズ / RAI-NET ACCEL BATTLERS** — a browser
implementation of the fictional strategy board game from the visual novel
*STEINS;GATE*. It is a turn-based, Stratego-like game of hidden pieces played
on a grid, with single-player vs. CPU and real-time online multiplayer.

The entire game — markup, styling, game logic, AI, audio synthesis, and
online networking — lives in **one self-contained HTML file**. There is no
build step, no framework, no package manager, and no test suite. It is pure
vanilla HTML/CSS/JavaScript that runs by opening the file in a browser.

The UI text, in-game messages, and most code comments are in **Japanese**.
Preserve that language when editing user-facing strings and comments.

## Repository layout

| File                          | Purpose                                                        |
| ----------------------------- | ------------------------------------------------------------- |
| `index.html`                  | The game. Everything is here.                                 |
| `rai-net-accel-battlers.html` | **Byte-identical duplicate** of `index.html` (see below).     |
| `bgm.mp3`                     | Title-screen background music.                                |
| `Surviving_Cyber.mp3`         | Battle background music.                                       |

### ⚠️ The two HTML files must stay identical

`index.html` and `rai-net-accel-battlers.html` are kept byte-for-byte the
same (one is the canonical filename, the other is the deploy/share name).
**Any change to one must be mirrored to the other in the same commit.** After
editing, verify with:

```bash
diff -q index.html rai-net-accel-battlers.html   # must print nothing
```

A practical workflow: edit `index.html`, then `cp index.html
rai-net-accel-battlers.html`.

## Anatomy of the HTML file

Roughly (line numbers drift as the file changes — search rather than trust
them):

- **`<head>` / `<style>`** — all CSS. CRT/terminal aesthetic: dark green
  phosphor palette (`#00ff41`), scanline overlay, monospace font. Board and
  piece styling, overlays (`.overlay`), responsive `--cell` sizing.
- **`<body>`** — static markup: board container, side panel (status,
  difficulty, captured pieces, rules), and a stack of fullscreen `.overlay`
  modals (splash, name entry, auth, online lobby, tutorial, my-page,
  leaderboard, settings, win/lose, etc.). Buttons wire up via inline
  `onclick="..."` handlers calling global functions.
- **`<script>`** — all game logic as global functions (no modules, no
  classes). Major sections are marked with `// ── Section ──` comment banners.

### Script sections (search for the banner comments)

- **Version** — `GAME_VERSION`. Bump this when shipping notable changes; it is
  rendered into the title/splash automatically.
- **Audio** — Web Audio API synthesized SFX (`tone`, `noiseBlip`, and `snd*`
  functions like `sndWin`, `sndBattle`). Includes iOS/Android audio-unlock and
  AudioContext suspend logic so the game yields the audio session to apps like
  Spotify. BGM is the two `.mp3` files via `<audio>` elements.
- **Firebase Realtime Database relay** — `FB_DB`, `fbPut`/`fbGet`. Online play
  relays moves through Firebase RTDB by polling JSON endpoints (no SDK; plain
  `fetch`).
- **Local relay** — `lrPut`/`lrGet` for same-WiFi play via an optional local
  HTTP relay server (detected by private-IP hostname).
- **BroadcastChannel** — same-device, cross-tab sync for local testing.
- **Firebase Auth** — email/password + guest, via the Identity Toolkit REST
  API (`_fbAuthFetch`). Cloud save/load of player data and leaderboard.
- **State** — top-level `let` globals: `board`, `turn`, `sel`, `validMoves`,
  `over`, `captured`, `gameMode`, `myRole`/`myPN`, ratings, stats, etc.
- **Constants / variants** — `ROWS`, `COLS`, `GOAL`, `HOME` are reassigned by
  `applyVariant(v)` per game mode.
- **Placement / Battle / Move / Win** — core rules (see below).
- **AI** — `scheduleCpu`, `aiChooseMove`, `scoreMove` (see below).
- **Rendering** — `render()` rebuilds the board DOM each turn; `clickCell`
  handles input; many `show*`/overlay functions drive UI flow.

## Game rules (so changes stay correct)

- **Board:** grid of `ROWS × COLS`. Player 1 (you) starts at the bottom and
  aims for the top goal row (`GOAL[1]`); Player 2 (opponent/CPU) is the mirror.
  `HOME` defines each player's two starting rows.
- **Variants** (`gameVariant`, set via `applyVariant`):
  - `classic` — 6×4, piece set without medic/worm.
  - `extended` — 7×5, adds **MEDIC**.
  - `worm` — 8×6, adds **MEDIC** and **WORM**.
- **Pieces** (`piece.type`, some carry a `value`):
  - `program` (value 2–4), `firewall`, `virus`, `linker` (☁ — the king-piece),
    `medic`, `worm`.
  - Pieces start `revealed:false` (hidden, Stratego-style) and are revealed on
    contact/battle.
- **Win conditions** (`checkWin`): move your `linker` onto the opponent's goal
  row, **or** capture all of the opponent's linkers. No linkers left on either
  side is a draw.
- **Movement** (`getMoves`): one orthogonal step into an empty cell or onto an
  enemy piece (which triggers a battle).
- **Combat** (`beats(a, b)` / `resolveBattle`): the single source of truth for
  who wins a clash. Edit `beats` carefully — the AI and win logic both depend on
  it. Summary of current matchups:
  - `worm` never "beats" via `beats()` but consumes adjacent enemy `program`s
    in a chain (special-cased in `doMove`).
  - `medic` beats `virus` only; loses to nothing except virus (and can
    transform after winning — `showTransformPicker`).
  - `virus` beats `program` and `linker`.
  - `firewall` beats `virus` and weak programs (value ≤ 2).
  - `program` beats lower-value programs, firewalls if value ≥ 3, and linkers.
  - `linker` beats `firewall`.

## AI

- `difficulty` ranges 2–5 (中級 / 上級 / 超上級 / スーパーハカー). Set via
  `setDiff`.
- `aiChooseMove` enumerates all legal Player-2 moves, scores each with
  `scoreMove`, then picks based on difficulty:
  - **2:** weighted-random among top 3.
  - **3:** mostly the best, occasionally 2nd.
  - **4:** always the best.
  - **5:** takes an immediate win if available, otherwise filters out moves
    that would hand Player 1 an immediate win (one-ply safety check).
- `scoreMove` is a hand-tuned heuristic: rewards advancing the linker toward
  goal, winning favorable battles, defending the linker, and (at higher
  difficulty) positional/threat awareness. Sentinel scores: `1e6` = winning
  linker move, `9e5` = capturing the enemy linker.
- A `noise[]` table adds randomness inversely scaled to difficulty.

## Online multiplayer notes

- Three transports, tried/used depending on context: **Firebase RTDB**
  (internet), **local HTTP relay** (same WiFi), **BroadcastChannel** (same
  device). All exchange small JSON payloads; there is no game server with
  authoritative state — clients trust each other and replay moves.
- A room is keyed by a passphrase (`ppToKey`). Host/guest roles assign
  `myPN` (1 or 2). Remote moves are applied with `applyingRemote=true` to avoid
  echo loops. Disconnect recovery snapshots game state to `localStorage`.
- **Firebase config (API key, DB URL, RTDB instance) is committed in the
  HTML on purpose** — it is client-side Firebase config, not a secret. Don't
  "fix" it by removing it, but never add real server secrets to this repo.

## Persistence

- `localStorage` keys are namespaced with the **`dna-`** prefix (e.g.
  `dna-name`, `dna-bgm-on`, `dna-auth-token`, `dna-player-*`). Keep new keys on
  that prefix.
- Logged-in users additionally sync data to Firebase under
  `users/<uid>/data` and `leaderboard/<uid>`.

## Conventions

- **No build, no deps, no tests.** Don't add a bundler, framework, or
  `package.json` unless explicitly asked — the single-file, zero-dependency
  nature is the point (it must run from a plain file:// open and be easy to
  host as a static page).
- Code style is terse, single-file, globals-and-functions. Match the existing
  density and the `// ── ... ──` section-banner style; don't reformat
  wholesale.
- Keep UI strings and comments in **Japanese**.
- When changing rules, update *all* of: `beats`/`resolveBattle`, the AI
  (`scoreMove`), `checkWin`, the placement functions, and the in-game rules/
  piece-info modals so the displayed rules match the code.
- Bump `GAME_VERSION` for player-visible changes.

## Verifying changes

There is no automated test harness. To check a change:

1. Keep the two HTML files in sync (`diff -q`, above).
2. Open `index.html` in a browser (or `python3 -m http.server` and visit it)
   and play through the affected flow — placement, a few moves, a battle, and
   a win/loss.
3. For online flows, opening two tabs exercises the BroadcastChannel transport
   without needing Firebase.

## Git workflow

- The upstream history shows the file is often edited externally and
  re-uploaded ("Add files via upload" commits). Regardless, when committing
  from here: make focused commits, **keep the two HTML files identical in the
  same commit**, and write clear messages.
- Do not create pull requests unless explicitly asked.
