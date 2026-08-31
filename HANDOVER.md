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
- **Roles**: whoever opens the page with no `?room=` query param is the *host*
  and gets an invite link (`?room=<peerId>`) to send. Whoever opens that link is
  the *guest* and auto-connects. A third form, `?host=<peerId>`, means "be the
  host **and claim this exact peer id**" — `new Peer(id)` rather than
  `new Peer()`. That is what lets a taken-over room keep the link that was
  already shared, and it also means reloading a host page with `?host=` re-claims
  the same id instead of minting a new one.
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
- **Live preview**: until the first flip locks things in, the coin shows the
  face *you* hold. `previewSide()` turns it on any side change (a shorter
  `.previewing` transition, so it doesn't read as a real flip), and it is a
  no-op once `flipping` or `sidesLocked` is true — `doFlip()` sets both before it
  calls `renderSideUI()`, which is what stops the preview fighting the flip
  animation. It also strips `.previewing` so the flip doesn't inherit that
  timing.
- **Side-switch gating**: the Heads/Tails buttons are gated on `calledSide`,
  **not** on `connected`. That distinction matters: the host draws a side at
  load and owns it, so they can change it while waiting for someone to join —
  gating on `connected` left them staring at greyed buttons with a side they
  never chose. A guest has no `calledSide` until the host's `call` arrives,
  which can only happen once connected, so the same condition keeps them
  correctly disabled while connecting. Sending a `call` before the connection
  opens is harmless: `send()` no-ops when the conn isn't open, and the host
  re-sends its current side on `open` regardless.
- **Flip gating**: the Flip button is separate and *does* require connected +
  both names set + sides assigned.
- **Recovery from a dead link**: a `?room=` link lives only as long as the
  host's page — peer ids regenerate on every load — so a link that has sat in a
  chat is usually stale. Opening one raises PeerJS `peer-unavailable`, which
  used to render as "Connection error — reload to retry". That was wrong advice:
  reloading retries the same dead room id, and there was no other way forward.
  `offerRestart(note, target, label, sub)` now reveals a panel whose button
  navigates to `?host=<the dead room id>`, so the page returns as a host that has
  **claimed that same id** — the link already sitting in someone's chat keeps
  working, and whoever else holds it can still join. Verified against the real
  broker: the regenerated invite link is byte-identical to the original.
  **The claim only succeeds if the broker has actually released the id.** It
  refuses with `unavailable-id` while the id is still registered, which is the
  case whenever the original host is still alive — so a *dropped connection* is
  NOT a takeover situation. That is why `conn.on('close')` offers **"Try to
  rejoin"** (reload the same `?room=` link) rather than a takeover: if the host
  is still there you reconnect, and only if they have truly gone does the empty-
  room path then offer the takeover. `peer-unavailable` means the id is already
  unregistered, so a takeover from *that* state is the case that reliably works.
  An id frees roughly a second after its peer goes away, so a claim refused by a
  narrow race is retried up to `MAX_CLAIM_RETRIES` times at 1.5s intervals before
  giving up and offering a fresh toss.
  The typed name rides across that reload in `sessionStorage` under
  `callit:name` and is prefilled on load (wrapped in try/catch — it throws in
  some privacy modes and is unavailable on `file://`, so test this over http,
  not a local file).
  If the id is still held after those retries, the panel offers a *fresh* toss
  at a bare pathname, so a takeover cannot loop forever on an id it will not get.
  **Testing note:** a stubbed broker that honours every requested id proves
  nothing about any of this — the first version of this feature passed such a
  stub and still failed in the browser. Exercise id claiming against the real
  broker, or not at all.
  It is offered on any fatal peer error, and to a **guest** whose host leaves.
  Deliberately not to a host whose guest leaves: that host's link is still live
  and someone can still join it, so re-hosting would needlessly invalidate a
  link they may already have sent.
- **Score**: tracked client-side on each browser independently (each side infers
  the same outcome from the same `result` + `calledSide`, so they should always
  agree — not verified in a live test yet).
- **Coin art**: two hand-built inline SVGs (not images), and both are puns on
  the side they represent. Obverse (heads) is a **guillotine** — uprights,
  lintel, oblique blade, and a beam with a semicircular **neck cradle** notched
  into its top edge. That notch replaced an enclosed circular hole: any beam
  thin enough to look right leaves only 2–3 unit slivers above and below a hole
  that size, and the result reads as an hourglass. A notch carved into the
  outline has no such constraint. The beam spans exactly upright-to-upright so
  it reads as connected — touching, never overlapping, since an overlap under
  evenodd would punch a hole.
  Reverse (tails) is a **man's head in profile wearing a rattail**, drawn as a
  plain silhouette. An ear was cut into it with evenodd and removed again — at
  coin scale it read as a crescent moon sitting in the middle of the head. The
  profile now carries no internal cuts at all, so its `fill-rule="evenodd"` is
  vestigial.
  The two faces are deliberately unalike, rectilinear against organic, so a
  glance tells them apart before you read the name.
- **Arc radii are not points.** The guillotine is authored at nominal size and
  scaled 0.9 in a build step. Scaling `A rx,ry` as though it were a coordinate
  pair — and then again as a radius — silently turns the neck cradle into a
  shallow ellipse instead of erroring. Oversized radii are legal SVG; the path
  just draws a lazier curve. It still renders, only wrong, which is how it
  shipped once. Same bug had ovalled the earlier circular neck hole.
- **Drawing the profile is the hard part.** It took ten rendered iterations, and
  every failure mode is worth knowing before you touch it: straight `L` segments
  between facial landmarks produce a beak — use `C` curves; the neck must be
  narrower than the skull with the occiput overhanging it, or head and neck
  merge into one ball; features need exaggeration, since a 3-unit step is under
  3px on the rendered coin; and the rattail must be rooted at the *nape* and
  hang clear of the bust, or it merges into the shoulder and reads as a hook.
  The tail itself is a separate path (not part of the evenodd outline), so it
  may overlap the head freely — that overlap is what attaches it. Its shape is
  offset from a centreline, and the thing that makes it read as *tied* hair is a
  **pinch**: narrow it sharply where the tie sits, then let it swell again
  below. A tie drawn as its own shape is invisible, being the same fill as the
  tail. Width matters — 9–13 units reads as a scarf, ~5 tapering to ~3 reads as
  a braid.
  A hairline groove was tried and removed — it read as a swim cap. Render it
  before you believe it.
  Relief is faked by drawing each emblem three times: a dark copy offset +1.5px,
  a light copy offset -1.2px, then the solid on top. A
  `feTurbulence`/`feDisplacementMap` filter roughens the strike, and three small
  verdigris radial gradients stain the disc. The lower rim carries a motto per
  face: "· CAPVT ·" (head) under the guillotine, "· FORTVNA ·" under the
  rattail.
- **The rim inscription**: the upper rim of each face is struck with the *name of
  whoever holds that face* — not the words "heads"/"tails", which appear nowhere
  on the coin. So the face that lands up shows the winner's name. Set by
  `inscribeCoin()`, which reads `localName`/`remoteName` against `mySide()`, so it
  must be re-run whenever a name OR a side assignment changes; it is called from
  `renderNames()`, `renderSideUI()`, the name `input` handler, and again on
  `document.fonts.ready` (the fit depends on the loaded face). An unclaimed face
  reads IGNOTVS.
  Because the inscription depends on the side assignment, switching sides also
  swaps which face carries which name.
  `inscribe()` shrinks font-size and tracking in a loop against
  `getComputedTextLength()` until the text fits `ARC_LIMIT` (150 user units along
  a 66-radius arc). Verified by rendering names of 0, 2, 4, 5, 8, 10, 12, 15 and
  19 characters — the 20-char `maxlength` is the ceiling. If you move the arc,
  re-derive ARC_LIMIT from its radius or long names will overrun the beading.

## Coin layout constraints
The rim furniture and the emblem share a small disc, and they must not touch:
- Name inscription: upper arc at r=66, fitted to `ARC_LIMIT` 150 (see above).
- Motto: lower arc at r=76. Its glyphs grow *inward* from that arc, so at
  font-size 10.5 they reach about r=65.5.
- **Emblems must therefore stay inside about r=57**, which leaves ~8 units of
  clear metal. The guillotine is drawn at a nominal size then scaled 0.9 and
  lifted 2 units to satisfy this; the profile is drawn within it directly.
  If you enlarge an emblem, check its extreme points against r=65.5 — the
  bottom corners of a bust or plinth are what break this first.

## Design direction — "Struck"
An engraver's proof sheet at night. The page splits into two mirrored
territories, yours and theirs, divided by an engraved seam that the coin sits
on; the winning territory takes a patina rule and a soft wash after each flip.

- **What darkens the coin's lower-right** — the thing that makes rim text hard
  to read at the end of a word. Two gradients contribute, and it is worth
  knowing which dominates:
  1. `metal`, the radial gradient. Centred off-axis at 38%/28%, so the
     lower-right is the farthest point and takes the darkest stop. **This is the
     dominant one.** Its outer stops were `#4a3616`/`#241a0b`, which buried the
     end of FORTVNA in near-black; they are now `#6d5326`/`#4a3818`, and the
     centre/radius moved from 33%/23% r=88% to 38%/28% r=96% for a gentler
     falloff.
  2. `sheen`, a linear overlay whose dark stop sits at 0.21 (was 0.42). Real but
     secondary — halving it alone did *not* fix the motto.
  Treat both as legibility settings. If rim text goes muddy again, reach for
  `metal`'s outer stops first.
- The motto deliberately has **no** light ghost copy, unlike the name. At 10.5px
  the 1.1px offset is proportionally far larger than it is on the 18px name, and
  it fuzzes the letters at the coin's real 186px size. Tested both ways.
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

## Testing
`index.html` has no test suite, but the interaction is coverable without a
second device: stub `Peer` with an in-page loopback that fires `open` then
`connection`, and echoes a `hello` back as the opponent. The real app code then
reaches `connected`, the side buttons enable, and `.click()` on them drives the
genuine handlers. Overriding `Math.random` first makes the drawn side
deterministic. This is how the side-switch preview, the lock-on-flip and the
winner highlight were verified; it is worth rebuilding if you touch that logic.

## Known gaps / suggested next steps
1. ~~**Live two-device test**~~ — **done.** Two real browser tabs against the
   public PeerJS broker: connection, name sync, random side assignment, a flip,
   and score agreement all verified. Both sides reported the same result and the
   same history, each score shown from its own perspective. Sides locked on both
   ends after the flip. The remaining untested dimension is two *separate
   networks* (NAT traversal), which one machine cannot exercise.
2. **Simultaneous toggle** — if both players click at the same instant the two
   `call` messages cross and the sides could briefly disagree. Not handled; a
   host-authoritative assignment would fix it if it ever matters.
3. **Reconnection handling** — there is still no automatic reconnect. A
   stranded guest now has a manual way out (see *Recovery* above), but neither
   side retries a dropped connection on its own, and an in-progress score is
   lost when either side re-hosts.
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
