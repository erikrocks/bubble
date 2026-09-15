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
assets. Same rule as KUDR and the hub. Edit `index.html` directly; do not
introduce a bundler or split it into modules without a real reason.

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
  - **breathing** — a slow out-of-phase pulse, ±5.5% while blowing, ±2% in flight.
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
  study artifact), and a sprung wind squash in flight with the underside pinched.
