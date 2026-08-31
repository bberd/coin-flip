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
- **Coin art**: two hand-built inline SVGs (not images). Obverse is a 24-ray
  radiate crown around a faceted boss; reverse is a laurel wreath opening at the
  top around a compass star. Every emblem is generated from polar math (rays at
  2pi*i/N, wreath leaves swept along an arc), so symmetry is exact rather than
  hand-tuned — if you edit them, regenerate rather than nudging coordinates.
  Relief is faked by drawing each emblem three times: a dark copy offset +1.5px,
  a light copy offset -1.2px, then the solid on top. A
  `feTurbulence`/`feDisplacementMap` filter roughens the strike, and three small
  verdigris radial gradients stain the disc. Both faces carry "· FORTVNA ·" on
  the lower rim.
- **The rim inscription**: the upper rim of each face is struck with the *name of
  whoever holds that face* — not the words "heads"/"tails", which appear nowhere
  on the coin. So the face that lands up shows the winner's name. Set by
  `inscribeCoin()`, which reads `localName`/`remoteName` against `mySide()`, so it
  must be re-run whenever a name OR a side assignment changes; it is called from
  `renderNames()`, `renderSideUI()`, the name `input` handler, and again on
  `document.fonts.ready` (the fit depends on the loaded face). An unclaimed face
  reads IGNOTVS.
  `inscribe()` shrinks font-size and tracking in a loop against
  `getComputedTextLength()` until the text fits `ARC_LIMIT` (150 user units along
  a 66-radius arc). Verified by rendering names of 0, 2, 4, 5, 8, 10, 12, 15 and
  19 characters — the 20-char `maxlength` is the ceiling. If you move the arc,
  re-derive ARC_LIMIT from its radius or long names will overrun the beading.

## Design direction — "Struck"
An engraver's proof sheet at night. The page splits into two mirrored
territories, yours and theirs, divided by an engraved seam that the coin sits
on; the winning territory takes a patina rule and a soft wash after each flip.

- Ground: indigo-slate (`--ink #0f1420`), vignetted via a radial gradient
- Metal: brass (`--brass-hi #f3dca4`, `--brass #c9a24d`, `--brass-deep #7a5d24`)
- Win state: verdigris (`--patina #5fa892`) — the colour brass actually oxidises
  to, used *only* to mark the winner. Don't spend it elsewhere.
- Text: chalk (`--chalk #ece8dd`, `--chalk-dim #8b8877`)
- Type: Cinzel for the wordmark, verdict line and coin inscriptions — it is
  drawn from Roman monumental capitals, which is why the rim lettering reads as
  struck rather than set. IBM Plex Mono for every label, input and ledger figure.
  Both from Google Fonts, with local fallbacks — the page is no longer
  dependency-free at render time, though it degrades to the fallbacks.
  Cinzel Decorative was tried for the wordmark and rejected: its swashes merge
  the LL of "CALL" into an illegible mass.
- **Signature**: the guilloche rosette behind the coin is a real hypotrochoid,
  the curve family used for banknote and coin-die security engraving, generated
  offline and inlined as two `<path>`s (~15KB of the file). It rotates 120
  degrees while the coin is in the air. It is the one loud element — keep
  everything around it quiet.

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
4. **Mobile layout** — the two-territory split holds down to 320px via a
   `max-width: 380px` media query that shrinks the coin and rosette. Verified by
   rendering at that width, not on a physical handset.
5. **Optional**: self-host a PeerServer instead of relying on the public broker,
   if this needs to be more reliable than "casual use."
6. **Deployment**: create a GitHub repo, add `index.html` at the root, enable
   GitHub Pages (Settings → Pages → Deploy from branch → main / root).

## Non-goals / things intentionally left out
- No backend, no database, no build tooling — kept deliberately as a single
  static file for trivial GitHub Pages hosting.
- No persistence of past sessions — history/score reset on page reload.
