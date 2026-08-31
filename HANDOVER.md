# Handover: Shared Coin Flip (P2P, GitHub Pages)

## What this is
A single-file, static web app: two people on different computers call heads/tails
and watch the same coin flip together, synced peer-to-peer (no backend). Built to
be dropped straight onto GitHub Pages.

## Current state
Feature-complete and believed working, but **not yet tested live across two real
browsers/devices** — only previewed in isolation, since the sandbox this was built
in blocks the outbound PeerJS connection. That live two-device test is the first
thing to do.

## Files
- `index.html` — the entire app (HTML + CSS + JS, no build step, no dependencies
  to install). This is the only file that needs to exist in the repo for GitHub
  Pages to serve it.

## Stack / how it works
- Vanilla HTML/CSS/JS. No framework, no bundler.
- **PeerJS** (loaded from `unpkg.com/peerjs@1`) handles the WebRTC connection.
  It uses PeerJS's free public cloud broker only to introduce the two browsers to
  each other — the coin-flip data itself flows peer-to-peer after that. This is
  fine for casual use but has no uptime guarantee; if it becomes flaky, the fix is
  running a self-hosted PeerServer (npm package `peer`) instead of the public one.
- **Roles**: whoever opens the page with no `?room=` query param is the *host* and
  gets an invite link (`?room=<peerId>`) to send. Whoever opens that link is the
  *guest* and auto-connects.
- **Message protocol** (JSON objects sent over the PeerJS data connection):
  - `{type: 'hello', name}` — sent by each side whenever their name field changes.
  - `{type: 'call', side}` — `side` is always the *host's* side; the guest is
    always the opposite. The host draws one at random on load and sends it when
    the connection opens. After that either player may send it: clicking the side
    you don't currently hold swaps both players. Ignored once a flip has
    happened.
  - `{type: 'flip', result}` — sent by whichever side clicks Flip first; both
    sides animate the same `result`.
- **Sides**: drawn at random by the host (so neither player picks first), shown
  as two buttons on both cards with the local player's side highlighted. Either
  player can toggle the assignment until the first flip, after which the buttons
  are disabled and greyed for the rest of the session.
- **Flip gating**: the Flip button stays disabled until: connected + both names
  set + sides assigned.
- **Score**: tracked client-side on each browser independently (each side infers
  the same outcome from the same `result` + `calledSide`, so they should always
  agree — not verified in a live test yet).
- **Coin art**: two hand-built inline SVGs (not images) — an "obverse" laureate
  profile bust with "IMP · CAESAR" and a "reverse" spread-wing eagle with
  "S P Q R", aged-bronze radial gradient, beaded dotted border, and a
  `feTurbulence`/`feDisplacementMap` filter that gives the coin edge a slightly
  hand-struck, non-perfect-circle look.

## Design tokens (for consistency if extending)
- Background: deep felt green (`--felt-dark #0d3b2e`, `--felt #12503e`)
- Coin metal: antique bronze-gold (`#e8c98a` → `#5f4726` radial gradient)
- Accent/buttons: bronze (`--bronze #b9793f`)
- Text: cream (`--cream #f2e9d8`)
- Type: system sans for UI, Georgia/serif for the coin legends and headline

## Known gaps / suggested next steps
1. **Live two-device test** — deploy to GitHub Pages, open on two separate
   devices/networks, confirm connection, name sync, side-call, flip animation,
   and score all behave correctly. This hasn't happened yet.
2. **Simultaneous toggle** — if both players click at the same instant the two
   `call` messages cross and the sides could briefly disagree. Not handled; a
   host-authoritative assignment would fix it if it ever matters.
3. **Reconnection handling** — if the guest refreshes or the connection drops
   mid-session, there's currently no reconnect/retry flow; it just shows
   "Opponent disconnected."
4. **Mobile layout** — built responsive-ish (single column, no media queries
   added) but not checked on an actual small screen.
5. **Optional**: self-host a PeerServer instead of relying on the public broker,
   if this needs to be more reliable than "casual use."
6. **Deployment**: create a GitHub repo, add `index.html` at the root, enable
   GitHub Pages (Settings → Pages → Deploy from branch → main / root).

## Non-goals / things intentionally left out
- No backend, no database, no build tooling — kept deliberately as a single
  static file for trivial GitHub Pages hosting.
- No persistence of past sessions — history/score reset on page reload.
