# CLAUDE.md — bubble.eriksheridan.com

## What this is

A pixel-art bubble game, one of the addresses under `eriksheridan.com`. Read
`CLAUDE.md` in `erikrocks/eriksheridan.com` first — it documents all the sibling
addresses, the Squarespace DNS setup, and the GitHub Pages certificate trap.

Erik is not an engineer. Handle tooling, DNS and setup for him rather than handing
over commands, and explain changes by consequence ("players will see X") rather
than by mechanism.

| | |
|---|---|
| Address | `bubble.eriksheridan.com` |
| Repo | `erikrocks/bubble` |
| Host | GitHub Pages, branch `main`, path `/` |
| DNS | `CNAME bubble → erikrocks.github.io` at Squarespace |

## The one rule

**`index.html` is the entire game** — markup, CSS, and every sprite, in one file
with no dependencies but Google Fonts. No build step, no framework, no image
assets *in the game*. Same rule as KUDR and the hub. Edit `index.html` directly; do
not introduce a bundler or split it into modules without a real reason.

The only other files are Home Screen furniture: `manifest.webmanifest` and three PNG
icons, which were rendered from the game's own bubble sprite (see History) rather
than drawn by hand. Nothing in the game loads them.

## How the game is put together

Read top to bottom, `index.html`'s script is in these blocks:

- **pixel plumbing** — `painter()` wraps a raw `Uint8ClampedArray`. `px/rect/hl/vl/blob`
  draw into it. Three modifiers matter: `setA()` (alpha), `setFade()` (mixes a colour
  toward the sky at that row and writes it **opaque** — this is how background scenery
  recedes without the skyline showing through it), and `force`+`oy` (used to stamp a
  1px dark silhouette under each overhead obstacle so the deadly edge reads).
- **palette** — every colour in the game, named.
- **the bubble** — `drawBubbleE(d,W,H,cx,cy,rx,ry,t,pinchDown)`. A black keyline over an
  iridescent film band, with the keyline dropping to a half-mix with the film below
  radius 7 so small bubbles keep their shimmer. It takes separate x/y radii, plus
  `pinchDown`, which flattens the **underside** only. `drawBubble()` is a round wrapper.
  Squish comes from three places, all draw-time and none of them touching collision
  (the hitbox stays `0.82 * R`, so the visual is generous, never mean):
  - **breathing** — a slow out-of-phase pulse, ±3.5% while blowing, ±1.2% in flight.
  - **wind squash** — `G.sq`, a damped spring driven by what the air is doing: +1 while
    the wind is on (wide and flat), negative while falling free (tall). It overshoots to
    ~1.2 before settling, so starting and stopping the wind *rings*.
  - **pinchDown** — proportional to positive `G.sq`, so wind from below dents the bottom
    more than the top. This is the part that sells the force being uneven.
- **the kid, and the pigeon** — six kid sprites, one drawn at random per run. Each has
  an `anchor` = where the wand loop sits; the bubble spawns against it and the kid is
  drawn *after* the bubble so the loop stays in front.
- **sky + sidewalk**, **obstacles** — `GROUPS` holds every obstacle. Ground items anchor
  at the sidewalk; air items hang from the top and carry a `bg()` (the grounded thing
  they are attached to — trunk, shopfront, pole) plus `boxes[]`, which is what actually
  kills you and is deliberately smaller than the art.
- **the world / ramp / state / draw / loop** — the game itself.

## Making a ground piece read against the background

Big flat ground objects dissolve into hazed scenery. Two tools:

- `edge:true` on an item stamps a 1px dark silhouette *above and around* it (the air
  obstacles get the same treatment offset downward). On: bus shelter, car, phone box,
  bush, statue. **Careful with a full-width ground-shadow row on an `edge:true` item**:
  the keyline pass draws it one row up, and if the art does not cover that row it
  becomes a dark bar floating under the object. That is what happened to the car, which
  sits on wheels with a gap beneath it; its shadow is now just two contact points.
- Colour it against the sky, not in isolation. The statue was grey stone on a pale
  blue-grey horizon and vanished; verdigris bronze on near-black granite reads at a
  glance and still looks like a park statue.

Also: **never pair a `shelter` item with a `shop` item.** Paired obstacles sit ~8px
apart, and a bus shelter under a shopfront reads as the building's ground floor rather
than a separate thing to dodge. The rule lives in the pairing branch of `spawn()`.

## Shopfronts vary per instance

Obstacles are shared objects, so per-instance variety lives on the spawn record: each
gets `v`, a small random number, passed as the last argument to `draw()` and `bg()`.
`shopOf(v)` maps it to one of four schemes in `SHOPS` — brick, awning stripe and sign
board together. **Both** `draw` and `bg` need `v` forwarded; when only `draw` had it,
every awning was a different colour on an identical building and the variety looked
broken rather than absent.

Pick brick colours further apart than looks right in isolation: background scenery is
mixed 50% toward the sky, which halves every difference. The first set was subtle
enough to read as one building repeated.

## Park to city

A run walks out of a park and into the city. `cityMix(distance)` is 0 before 1000px,
1 after 2100px, and slides between — and it drives **both** the scenery and the
furniture, so the change of place and the change of difficulty are one event:

- Items carry `park:true` (hedge, statue) or `city:true` (hydrant, meter, mailbox,
  paper box, bus shelter, car, phone box, every shopfront, traffic signal). Untagged
  items — benches, wire bin, bollard, trees, streetlamp — belong to both and never
  leave. `pickFor()` weights the pool by the mix, so city things fade in as
  `mix²` (slow at first, then all at once) and park things fade out slightly faster
  than they arrive.
- The horizon cross-fades from `drawTreeline()` to `drawSkyline()`, and
  `drawVerge()` lays grass along the path edge at `1 - mix`.
- **The park needs its own tall pieces** or the "forces you upward" rule breaks before
  the city arrives — that is what the hedge and the statue are for. If you ever make
  the shelter group city-only again, the park becomes hoverable.

## Adding an obstacle without it being invisible

`localStorage` holds the player's rotation as a list of item ids that are ON. A list
saved before your new obstacle existed does not mention it, and the naive read —
"replace the group with the stored list" — therefore switches it OFF for everyone who
has ever opened the tuning drawer. That is exactly what happened to the hedge and the
statue: shipped, tagged, weighted, and never seen.

The load path now saves a `known` roster alongside the rotation and treats **absent
from `known` as new, therefore on**. `LEGACY_ITEMS` is the roster from before `known`
was recorded; leave it alone. Add new obstacles freely — but if you change how the
rotation is stored, keep that rule or the next addition disappears too.

Worth knowing: a park-only piece also has a narrow window (tall pieces start at 800px,
the park ends at `PARK_END`), so park items carry a 1.8x weight. At 1.0 they showed up
in well under half of runs.

## Tuning

`P` holds the physics ("Twitchy": gravity 180, wind 420, drag 4.6, size effect 1.35,
blow rate 15). Response is `(10/radius)^1.35`, so size scales fall speed and lift
together — which means hover duty is the same at every size, and what size really
changes is tempo and how well you fit through gaps.

`STAGES` is the difficulty ramp by distance, and `speedAt()` ramps scroll speed from
60 to 98 px/s. **The ramp only works because some ground obstacles are tall** (bus
shelter 38px in a 104px view). Before those existed, hovering mid-screen survived
forever. If you add obstacles, keep something that forces the player upward.

The in-page "Tuning & rules" drawer writes to `localStorage` under `bubble.v1`, along
with best distance, runs and medals.

## View size

`226 x 104` — 2.17:1, a phone in landscape, because this is headed for an app. The
**height** is load-bearing: every obstacle's difficulty is a fraction of it. Changing
`GH` silently re-tunes the whole game. Widening is safe; making it taller is not,
without scaling the obstacles too.

## Fullscreen on a phone

Platform-split, and the button knows the difference:

- **Android / desktop** — the Fullscreen button calls `requestFullscreen()` then
  `screen.orientation.lock("landscape")`. Real fullscreen, locked sideways.
- **iPhone Safari** — there is no Fullscreen API for non-video elements, so the button
  hides itself. The route there is **Add to Home Screen**: `manifest.webmanifest` plus
  `apple-mobile-web-app-capable` gives a chrome-less launch. iOS ignores the manifest's
  `orientation`, so it cannot be *locked* — the player rotates, and their device
  rotation lock applies. A guaranteed landscape lock on iPhone needs the native wrapper.
- A rotate hint shows above the game on narrow portrait screens.

## Deploying

Push to `main`; Pages rebuilds in about a minute. Don't trust
`/pages/builds/latest` — it reports stale shas. Check the bytes:

```
curl -sS -o /tmp/live.html "https://bubble.eriksheridan.com/?cb=$RANDOM"; diff /tmp/live.html index.html
```

## History

- **Sep 2026** — Built from a design study (bubble sprite options, a physics tuning
  bench, and an obstacle sheet) and deployed here.
- **15 Sep 2026** — Squish added: breathing while blowing (chosen from six candidates in a
  study artifact), and a sprung wind squash in flight with the underside pinched. Dialled
  back on Erik's note — peak deformation is ~109% wide / 90% tall at R=15, which is the
  level he signed off; it was nearly twice that first. Pigeons also now cross the street
  on their own every 9-17s from stage 2, at their own height rather than aimed at the
  player (the loiter pigeon still comes for you). Page footer removed.
- **15 Sep 2026** — Stage 3 renamed "Let's go!". Added a Fullscreen button (Android/desktop
  only, hides itself on iPhone), a web app manifest for Add to Home Screen, and app icons
  rendered straight from `drawBubbleE` at 512px rather than drawn by hand.
- **15 Sep 2026** — Park-to-city progression (see above), with a hedge and a statue added so
  the park has its own things to climb over. Stage 2 renamed "Watch the ground", since
  "street" was wrong while you are still in a park. Pigeons now start in the park.
- **15 Sep 2026** — The hedge became a rounded bush (the rectangular block read as a wall),
  the statue went verdigris-on-granite to stop it blending into the sky, boxy ground pieces
  gained keylines, and shelters no longer pair with shopfronts. Death prompt is "Play again?".
- **15 Sep 2026** — Shopfront pass: doors were 64px tall (taller than the shop window)
  and now stand on the pavement at ~29px with the window beside them; four colour schemes
  per instance; the streetlamp lost a dithered "glow" below the head that read as a
  rendering artifact. Camping rule generalised — holding *any* height within 14px for
  2.6s sends a pigeon at you, not just the top third. Best distance promoted to its own
  line on the title screen.
- **15 Sep 2026** — Erik never saw the hedge or the statue. Two causes, both fixed: saved
  rotations switched them off (see "Adding an obstacle" above) and the park ended at 1000px
  while tall pieces only start at 800px, leaving a 200px window. Park now runs to 1500px,
  city from 2800px, park pieces weighted 1.8x — they now appear in 95% of runs to 60m.
  Fullscreen also got a root-element fallback and now reports "Not allowed here" instead of
  failing silently, because some embedders refuse the request and the button looked dead.
