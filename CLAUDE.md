# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single self-contained HTML file (`ultimate-tic-tac-toe-multiplayer3.html`) implementing online multiplayer Ultimate Tic-Tac-Toe. There is no build system, no package.json, and no test suite — everything (markup, CSS, and JS) lives in this one file. It is not a git repository.

## Running it

Open `ultimate-tic-tac-toe-multiplayer3.html` directly in a browser (e.g. `open ultimate-tic-tac-toe-multiplayer3.html`), or serve it with any static file server. There is no build or compile step.

To test multiplayer behavior locally, open the file in two browser tabs/windows and log in with two different usernames.

## Architecture

The app is a vanilla-JS (ES module, no framework) single-page app with Firebase Firestore as the sole backend, used both as the database and as the realtime sync/transport layer (via `onSnapshot` listeners) — there is no separate server code.

**Firebase config is hardcoded inline** in the `<script type="module">` block (the `firebaseConfig` object). This is a client-side Firestore API key, expected to be restricted via Firestore security rules rather than kept secret.

### Screens

The UI is four `<div id="screen-*">` blocks (`login`, `lobby`, `game`, `history`) whose visibility is toggled by `showScreen(name)`. There's no router — `currentScreen` is a global variable and screens are shown/hidden by setting `display`.

### Firestore collections

- `presence` — doc ID is username; holds `lastSeen` timestamp, refreshed by a 10s heartbeat (`startPresenceHeartbeat`). A user is considered online if `lastSeen` is within the last 25s (checked client-side; `onlineExpiryInterval` re-renders every 5s to expire stale users even without a new snapshot).
- `invites` — `{ from, to, status, createdAt }`. Status is `pending` or `declined`; accepted invites are deleted after creating a game. Declined invites are cleaned up client-side by the sender's own `subscribeOutgoingInvites` listener.
- `games` — the full game document: `players` (array of 2 usernames), `symbols` (map username -> 'X'/'O'), `board` (flat array of 81 cells), `bigWinners` (array of 9, values `'X'`/`'O'`/`'draw'`/`null`), `currentTurn`, `activeBig` (index 0-8 of the sub-board the next move must be played in, or `null` for "any"), `status` (`active`/`finished`), `winner`, `forfeitedBy`, `rematchRequestedBy`, `lastMove`.

There is no backend validation — Firestore security rules (not present in this repo) are the only thing that could stop a client from writing an illegal move directly. All game logic (win checking, move legality, whose turn it is, computing `activeBig`) runs client-side in `makeMove()` and is trusted.

### Board representation

The 9x9 ultimate board is a flat 81-element array; big cell `bigIdx` (0-8) owns small cells at indices `bigIdx*9 .. bigIdx*9+8`. `LINES` (the 8 winning triples) is reused for both small-board win checks and the overall big-board win check via `checkWin()`.

### Realtime sync pattern

Each concern has its own Firestore listener + unsubscribe function, wired up on login and torn down on logout: `unsubPresence`, `unsubIncoming`, `unsubOutgoing`, `unsubGames`. `subscribeMyGames()` is the important one for gameplay — it listens to all games where the current user is a player, splits them into `activeGamesCache`/`historyCache`, and if the currently-open game changed, updates `currentGameData` and re-renders the board. There is no optimistic local move state; every move is written to Firestore and the board only updates when the snapshot listener fires.

### Rendering

All rendering is manual DOM construction (`document.createElement`, `innerHTML`) — no virtual DOM/templating library. `symbolSvg()` generates inline SVG for X/O marks at two sizes (small-cell vs. big-cell overlay for a won sub-board). User-provided strings (usernames) are passed through `escapeHtml()` before being inserted via `innerHTML`.
