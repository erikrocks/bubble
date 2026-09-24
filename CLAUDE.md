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

## The launch flow

`title -> ready -> blow -> fly -> dead`

**Three taps, and the first one is free.** Tap to clear the title, tap to start it
filling, tap to launch — and that launch tap is also the first gust of wind. **Nothing is
held** during the start, which is what makes it work in landscape on a phone. Leaving it
to fill too long pops it on the wand, so waiting for a big bubble is the risk you take.

The first tap doing nothing but clearing the title is deliberate, and `press()` **returns**
after `startRun()` so it cannot fall through into the `ready` branch within the same call.
It used to, and the effect was that the tap dismissing the instructions also started the
fill: you never saw the kid, the wand or the street before you were committed. `ready` is
therefore a bare prompt over a clean, undimmed scene — no scrim, no paragraph. The
explaining happens on the title screen, where nothing is ticking.

Chosen by playing four candidates against each other (hold/let-go/tap, hold/let-go with a
gravity hang, tap/tap, and a sweeping size meter). The others and the `armed` state they
needed are gone — don't reintroduce a hold-to-blow without testing it on a phone.

Death restarts into the `ready` prompt, not straight into blowing, with a 0.45s guard so
the press that killed you cannot restart you.

## Overlays anchor to the canvas, not the frame

`.frame` holds the canvas *and* the HUD, the player row and the tool bar, so an overlay at
`inset:0` covers all of it: the dim swallowed the stats and the character picker, and
anything positioned from the bottom (`.launch`, an overlay's prompt) landed on the chrome
below the game instead of in the scene. Overlays, the zone banner and the launch hint
instead pin to `top:0` with `aspect-ratio:226/104` — exactly the canvas box, since the
canvas is always the frame's full width.

Two fallbacks revert them to `inset:0`, both being cases where the canvas is no longer the
tall part of the frame: **under 600px wide**, where the title copy outgrows a ~174px-high
canvas, and **in fullscreen**, where the canvas is centred vertically rather than sitting
at the top. `min-height:min-content` lets an overlay grow past the canvas rather than clip.

After a death the first tap only brings the next kid to the wand; the tap after that
starts filling. Restarting and committing to a size stay separate actions.

## Birds have to scale with the bubble

A 40px bubble falls at 15 px/s. In the ~1.6s a bird takes to cross the screen it can
move 18px — less than its own radius — so a bird aimed at its height is not a dodge,
it is a death. Three rules keep that honest, all in the `RULES.birds` block:

- **Ambient** birds use `clearLane(want)`: a height at least `R*0.82 + CLEAR` from the
  player, trying the far side if the near one clamps, and returning null rather than
  spawning an unfair bird when neither side has room. They are traffic, not an attack.
- **Camp** birds are aimed straight at `G.y` — that is the entire point of them. They are
  made fair by **time, not by missing**: the lead is derived from how long a bubble that
  size needs to shift its own radius, so R=20 gets 2.9s and R=5 gets 1.3s. Making them
  pass wide instead was the obvious fix and the wrong one: camping stopped being punished
  at all. Measured: a parked bubble is hit essentially every time, one that reacts never is.
- The camp threshold is `termFall(R) * 0.35` — a third of a second of that bubble's own
  free-fall — not a flat pixel count. Patience stretches with size too. A flat 14px
  asked a big bubble for a third of its whole manoeuvring budget.
- Bird flight bobs as `y0 + sin(x)*2.2`, **not** `y += sin(x)*0.28`. The second form
  integrates, so birds random-walked several pixels off their lane and ate the
  clearance; that alone was killing hovering players.

Verified by simulation: holding station takes zero hits at every radius from 5 to 20,
while doing nothing still drives you into the bird placed below you.

## The kid turns up in every zone

The kid you are playing is drawn **into obstacle sprites**, one per zone: sitting on the
fountain rim in the park, standing in the shop window under the striped awning in the
city, serving at the fry stand's counter on the boardwalk, sitting under the beach
umbrella, and out in the moored dinghy at sea.

The first attempt put these in the parallax background as free-floating scenes - a bench,
a shopfront, a Ferris wheel, a rubber ring, a rowboat - and that was wrong. They belong to
the furniture you are already dodging, so they read as part of the world instead of
wallpaper. **The kid does not have to sit inside the collider.** The awning is the hitbox;
the window below it is the same obstacle's art, and that is where the kid goes.

`hasKid(v)` gates it on `v`, the per-instance random the spawner already attaches, so
about a third of each obstacle's instances are occupied. Every instance would be wallpaper
again.

The playable sprite is 24px tall; a second one at that size reads as a second player, so
cameos use `miniKid`/`sitKid` or a few inline rows, built from the four colours each kid
carries in its `mini` block (hair, skin, top, leg). Enough to tell Pigtails from Ball Cap,
not enough to compete with the bubble.

## Zones, and the route

The run is a **round trip that repeats**: out to the sea, then back through the same
places in reverse, then out again. Endless without endless content, and it gives the
crossing a job — the turnaround — rather than being a dead end.

```
ROUTE = park, city, boardwalk, beach, sea, beach, boardwalk, city   (then repeat)
```

The first park leg is `FIRST` long (the tutorial); every leg after is `LEG`. `legW(i,d)`
ramps a leg in over `FADE` before its start and out over `FADE` before the next, so two
legs overlap during a handover. A place appears more than once on the route, so
`zw(id,d)` takes the strongest active leg with that id.

| Leg | Place | Reached |
|---|---|---|
| 0 | Park | 0 m |
| 1 | City | 45 m |
| 2 | Boardwalk | 78 m |
| 3 | Beach | 112 m |
| 4 | The crossing | 145 m |
| 5-7 | Beach, Boardwalk, City | back the other way |
| 8 | Park | 278 m, then the cycle repeats |

The same weights drive **both** the furniture and the horizon, so the change of place and
the change of difficulty are one event rather than two that coincide.

- Items carry `zones:["park"]` etc. Untagged items — benches, bins, bollards, trees,
  streetlamps — belong everywhere on land, and `itemW` returns `1 - zw("sea")` for them,
  so nothing from the land is left floating during the crossing. `itemW` squares a zoned
  item's weight and multiplies by 1.9, so a place's own furniture arrives late and then
  dominates rather than trickling in.
- Horizon: `drawTreeline` / `drawSkyline` / `drawSea`, each at its zone's `zw` weight —
  the long fade, so a place appears before you reach it.
- **Ground is separate and much sharper** (`groundMix`, `GROUND_FADE` 280px). Water
  painted at 35% over sand for eighteen seconds reads as a flooded beach, not as an
  approaching sea. The ground fade starts AT the leg boundary, so sand stays sand right
  up to the crossing and turns over in about four seconds.
  Two rules that cost a bug each: paint the place you are **leaving at full alpha** and
  fade the new one in on top — cross-fading both at partial alpha lets the concrete base
  show through the middle — and **stop painting the old one once the fade completes**, or
  the boardwalk's rail (drawn above the ground line, where sand cannot cover it) rides
  onto the beach.
- Birds: bluejay in the park, pigeon downtown, gull on the boardwalk and the coast.
- **Every zone needs its own tall ground piece** or that stretch becomes hoverable — park
  has the boxwood and statue, city the shelter/car/phone box, boardwalk the fry stand,
  beach the lifeguard chair, the crossing the lighthouse rock. All live in the `shelter`
  group, which is what the `tall` gate reads.

`zoneAt(d)` returns the leg you have entered, not whichever weighs most: the scenery
fades in early on purpose so a place appears before you reach it. A banner names it one
second after that start.

The later legs sit past most runs, so the tuning drawer has a **Start in** control that
begins a run at a place's first appearance purely to look at it. Those runs set `PREVIEW`
and record no score, medal or run count.

## Collision is a circle, and hitboxes can only ever shrink toward the art

**The bubble is tested as a circle.** `hitsAt` used to test a *square* of half-width
0.82R against every box. On a flat face that is the intended 18% forgiveness, but a
square's corner reaches 0.82*sqrt(2) = 1.16R - further than the bubble you can see - so
every corner of every obstacle popped a big bubble through up to 3px of clear air.
`circleRect` is a closest-point test with the same 0.82R: faces behave exactly as before,
corners only ever got more forgiving.

**How fairness was measured.** For every obstacle, every reachable bubble centre (soft
ceiling pops above R/3, the pavement below WALKY-0.82R), every radius and every art
variant: if `hitsAt` says hit, how far was the bubble's edge from the nearest pixel of
art, across all animation frames unioned? Anything over ~2px is a pop through clear air.
Before this pass every obstacle had one; the worst were the streetlamp (a box over an arm
that rises to the right), the palm (a box over a V of drooping fronds), the dinghy (a
32px wall around a 2px mast) and the clouds and boughs (boxes under rounded bottoms).

**Refit rule.** Twelve obstacles got bands fitted to their art, column by column, then
**clipped to the old collider** - a refit may remove collider, never add it. That was
verified pixel by pixel against the previous version: zero collider added anywhere, and
the only art no longer solid is the kid in the dinghy (the kid is scenery everywhere).
Tallest/deepest bands are unchanged, so the pairing-gap check and difficulty are too.
Art that deliberately sits outside a box - the kite's tail, frond tips, rain streaks -
stays pass-through.

`vboxes` holds one box set per art variant, for an obstacle whose SHAPE depends on the
instance's `v`. Only the squall needs it (three cloud banks). `boxes` stays as the
envelope, which is what `depthOf` reads.

**Still deliberately solid:** the space between a hanging obstacle and the ceiling. A
small bubble hugging the top can be popped above the outer fronds of a palm or over the
end of an awning with sky visibly between them. Hanging things are treated as solid up
to the top edge so that there is no sneaking over trees; opening that would need boxes
with a top as well as a bottom.

## Night must not hide the hazards

One tint for everything made the night atmospheric and the furniture invisible: a park
bench's brightness contrast against the verge fell from ~110 to ~11 out of 255, a cooler
was 60% camouflaged. Two things fix it, and both are night-only - daytime frames were
checked pixel-identical before and after:

- **`HAZ`**: the painter can record the pixels it touches (`gp.mk`). Obstacles, birds
  and the kid draw with it on, and `tintNight` gives those pixels `NIGHT_HAZ` (0.30)
  instead of the scenery's `NIGHT_SCENE` (0.62). Scenery stays properly dark.
- **Moonlight rim**: dark things - iron benches, hydrants, bins - read by day *because*
  they are dark on a light street. At night that is gone, so the keyline shifts from
  `C.edge` toward `RIM_NIGHT` as it gets dark, and ground furniture with no keyline gets
  one along its top edge, faded in with the dusk.

Worst night contrast went from 11 to ~20-27, and most obstacles now read at least as
well at night as by day.

## The frame is not the play area

`.frame` wraps the canvas **and** the HUD, the player row, the tool bar and the panels,
but the game's `pointerdown` listener is on the frame. Everything inside it therefore has
to be filtered out, and `e.target.tagName==="BUTTON"` is not enough to do it:

- a character portrait is a `<canvas>` **inside** a `<button>`, so a finger landing on the
  picture picked the kid *and* started the run;
- the HUD is plain `<dd>`s, so tapping the distance readout - directly under the play area
  on a phone - blew and launched a bubble.

The filter is `e.target.closest("button,.hud,.chars,.tools,.board,.hiscore")`. It matches
the nearest chrome ancestor rather than the target's own tag, and it leaves taps on the
fullscreen letterbox working, because in stripped fullscreen the HUD is `pointer-events:
none` and the event lands on the canvas anyway.

## Naming the zone at the right moment

This has been wrong in both directions, so the reasoning matters.

The horizon cross-fades over `FADE` (1100px), which means a place is visible long before
its leg starts and the two zones swap dominance at the **halfway point**, `FADE/2` before
the boundary. Three candidate moments:

| Trigger | City announced at | Verdict |
|---|---|---|
| Any weight at all | 1600px | Too early - a smudge on the horizon |
| Leg start (`legIndex`) | 2700px | Too late - a full skyline while it still says PARK |
| **Horizon handover (max weight)** | **2150px** | Right |

The banner now fires 0.6s after the handover, so on the city it lands at ~2186px instead
of ~2760px - about nine seconds earlier at the speed you are going there. `zoneAt` drives
the banner, the bird species and the HUD readout together, so all three turn over on the
same event.

## Day and night

The run is a round trip, so the light is too. `nightAt(d)` returns 0 for daylight and 1
for full dark:

| Where | Light |
|---|---|
| Park, city, boardwalk, beach (outbound) | day |
| The crossing | dusk falls - starts as the sea takes the horizon, complete two thirds across |
| Beach, boardwalk, city (homeward) | night |
| Park, at the top of the next lap | sunrise across the leg |

**The first lap opens in daylight** - `lap===0` is special-cased, because starting a new
player in the dark makes the game look broken rather than atmospheric. You have to earn
the night. It begins around 190m; a lap is 18,100px (302m).

The crossing is `SEA_LEG` = 3400px rather than the usual 2000. A sunset needs room to
happen in, and it gives the sea a second job. Legs are therefore **no longer evenly
spaced** - `legStart`/`legIndex` go through the `LEGCUM` table, and anything that does
arithmetic on `LEG` directly is a bug waiting to happen.

### How the dark is actually applied

Three parts, and the order matters:

1. **The sky palette** is mixed from `SKY_DAY` to `SKY_NIGHT` and the whole sky base is
   rebuilt - but only when the light has moved 0.02, so it is about fifty rebuilds across
   a whole sunset rather than one per frame. Stars and the moon are **baked into the sky
   base**, not drawn over it.
2. **`tintNight`** mixes every pixel toward a deep blue at the end of `drawWorld`, with two
   exemptions: any pixel still exactly matching the sky base (which is how stars and the
   moon survive at full brightness), and a small set of `LIT` colours.
3. **Light sources** must be painted at their exact palette colour to land in that `LIT`
   set, so lit windows and lamp glass go through `lit(p,fn)`, which drops the atmospheric
   fade for the moment it takes to stamp them. Miss that and the lamp is just another grey
   square. Lit skyline windows and lit shopfronts only appear above `NIGHT>0.25`.

The title screen is pinned to day (`nightAt` is passed 0 while the phase is `title`), or
the attract loop's distance would put the menu in the dark.

### Seeing it without a 300m run

The `/admin` jump list carries the homeward legs and the sunrise as separate entries, since
they are the same places in different light and no one is going to run there on purpose.

## High scores

Copied from KUDR. Supabase REST called with plain `fetch` — **no Supabase JS library**, so
the no-dependencies rule holds.

```
project  kudr-leaderboard (shared)   table  public.bubble_scores
columns  id, initials, score, created_at
RLS      SELECT + INSERT only - no UPDATE or DELETE policy exists
```

With RLS on, an operation with no policy matches zero rows, so PATCH and DELETE return
**204 having changed nothing** — the board can be added to but never edited or wiped with
the public key. Verified against the live table before any UI was written. Constraints
reject anything but `^[A-Z]{3}$` and a score outside 0-1,000,000.

The publishable key sits in the HTML on purpose; that is how Supabase is designed.
**Scores are forgeable** by anyone with devtools — unavoidable without accounts, accepted.

The column is an integer and Bubble measures metres with a decimal, so **distance is
stored as metres x 10**: 1287 is 128.7 m. `toScore` / `fromScore` are the only places
that know.

**Every call returns `null`/`false` on any failure and times out at 6s.** `loadScores`,
`submitScore` and `qualifies` all swallow errors. `qualifies` returning false when the
board cannot be read is deliberate: the game must never ask for initials it would then
fail to save. Nothing in the frame loop awaits or throws.

Periods are **calendar** periods (week starts Monday), not rolling windows, so everyone's
board resets together. Initials render through `textContent`, never `innerHTML` — that is
other people's text.

Shares KUDR's project rather than having its own. Free-tier Supabase **pauses after ~7
days with no activity**; one project serving two games halves what can go quietly to
sleep. The trade, which KUDR's notes made the other way: the two games' keys can now read
and insert into each other's score tables. Given scores are public and forgeable anyway,
that buys little.

## The tuning drawer lives at /admin

The physics sliders, rules, Start-in preview and obstacle rotation are hidden unless
`ADMIN` is true: the path is `/admin`, or the hash is `#admin`, or there is an `admin`
query parameter. `admin/index.html` is a five-line redirect to `/#admin`, so the tidy URL
works while the game stays one file.

**This is obscurity, not security, and that is the right amount.** Everything the drawer
changes is client-side and already sitting in `localStorage`; there is nothing to protect,
only clutter to hide from players.

"Reset to system default" restores the tune, rules, preview and obstacle rotation. It
deliberately keeps best distance, runs, medals and the chosen kid — those are the
player's, not settings.

## Who's blowing

A row of kid portraits under the game picks the character, or "Anyone" for a random kid
each run. It persists as `kid` in `localStorage`. Vivian asked for this, and only ever
plays as Pigtails.

## Two lamps, and neither is allowed everywhere

`lamp` (the modern cast-arm streetlamp) is `zones:["city","boardwalk"]`. It used to be
untagged, which meant it stood on the beach and in the park, and it read as municipal kit
someone had dumped there.

`parklamp` is the park's own: Victorian cast iron, dark grey, and **straight up and down**
— fluted post, tapered lantern sitting on top of it, finial above that. It is not on a
bracket and must not be put back on one. Because the post is `bg` it is drawn at half fade
toward the sky, so it starts darker than the streetlamp on purpose: it has to survive the
mix and still read as cast iron. The collider is the lantern and its neck only; the post is
scenery you pass in front of, like every other pole.

`leafy` (the overhanging bough) is `zones:["park","city"]` for the same reason - untagged,
it hung over the boardwalk, the beach and the open sea.

Measured across a run: streetlamp weight is 0 for the whole beach and the whole crossing,
and 0 in the park core; the park lamp is 0 everywhere except the park; the leafy bough is
0 on the boardwalk, the beach and the sea.

Benches (`slat`, `scroll`, `stone`), the wire bin and the bollard are
`["park","city","boardwalk"]` - right on a promenade, wrong on sand. The two boughs that
ship switched off (`bare`, `blossom`) are tagged park/city as well, so enabling them in
/admin cannot hang a tree over the sea. **Nothing is untagged any more, and nothing new
should be.**

The rule this keeps arriving at: an untagged item is weighted `1-zw("sea")`, which means
*everywhere on land*. That was fine when the game was one street. With five zones, leaving
a piece of furniture untagged is a decision to put it on the beach.

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

## Fullscreen is the only way to lose the phone's URL bar

Safari keeps the address bar in landscape and no web API can hide it, so the on-screen
Fullscreen button is the whole story on a phone. It has two failure modes worth knowing:
iPhone Safari has no Fullscreen API for non-video elements at all (the button says
"Not allowed here" rather than doing nothing), and some hosts allow only the document
root, which is why there is a `documentElement` fallback and a `body.fs-root` class to
hide the page around the game when that path is taken.

`.frame` is the fullscreen element, and it carries the HUD, the player row and the tool
bar as well as the canvas. On a phone in landscape that chrome leaves a canvas too small
to play, so `@media (pointer:coarse),(max-height:560px)` strips it: the player row and
tool bar go, the HUD collapses to Distance and Best floating over the top-left of the
canvas, and `.fsbar` — a corner pause button and a ✕ — appears. **Neither is optional.** Stripping
the tool bar takes both the Pause and the Exit fullscreen buttons with it, and a phone has
no P or Esc key. Both call `stopPropagation` on `pointerdown` because the frame turns any
pointerdown into a gust of wind.

The pause glyph flips ❙❙ / ▶ through `syncFsPause()`, which is called from `togglePause`,
from `startRun`, from `pop` and from `hudTick` - the first three so it cannot lag a phase
change, the last as a backstop. Its `disabled` test is **exactly** `togglePause`'s own
guard (title and dead), or the button lies about whether pressing it will do anything.
Tapping the screen resumes as well, which is why the pause overlay says "Tap, P or Esc".

Desktop fullscreen is deliberately left alone: there is room for the HUD there and it
already worked.

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
- **15 Sep 2026** — Added the `armed` state: blow, let go, the bubble waits on the wand,
  tap to launch. The launch press doubles as the first wind input.
- **15 Sep 2026** — Birds made size-aware (see above) after big bubbles turned out to be
  unable to dodge them at all, and the camping rule stopped punishing a size that cannot
  physically move fast.
- **15 Sep 2026** — Boardwalk added as a third zone, which meant generalising the single
  park/city slider into a `ZONES` list. New furniture: deck railing, ice cream cart, coin
  telescope, fry stand (the walk's tall piece) and an arcade sign. Sea horizon,
  plank decking, gulls instead of pigeons. Banners now name the zone a second after you
  enter it rather than announcing difficulty stages.
- **15 Sep 2026** — Title screen got a flock of eight bubbles that pop on the scenery for
  real, using the player's own collision test.
- **15 Sep 2026** — The crossing's water was washing 12 m into the beach: ground now has its
  own short fade, separate from the horizon's. Camp birds aim at the player again — passing
  wide had made camping free.
- **15 Sep 2026** — Sprites animate: boats and buoys ride the swell, the lighthouse flashes,
  traffic signals cycle. Gulls at sea instead of pigeons. "Who's blowing" became "Who's playing".
- **15 Sep 2026** — The squall's rain animates (air sprites now get the clock) and comes in
  three shapes. The white cloud moved from the crossing to the boardwalk — mixing fair and
  stormy weather in one place read as a mistake, and the boardwalk's ceiling was thin.
- **15 Sep 2026** — Tuning drawer moved behind `/admin`, with a reset-to-defaults button.
  Pigtails wears pink at Vivian's request (it was blue only because the original yellow
  merged with her blonde hair; pink has no such problem).
- **15 Sep 2026** — High-score board added, copied from KUDR: same Supabase pattern, own
  table in the same project. Verified the RLS behaviour against the live table first.
- **15 Sep 2026** — The route became a repeating round trip (Erik's idea): out to the sea
  and back through every place in reverse. Added the crossing itself — open water underfoot,
  buoy, moored dinghy, lighthouse rock, low cloud and a rain squall. Added a character
  select row, because Vivian only wants to play as the pigtails girl.
- **15 Sep 2026** — Fixed hitboxes that did not match their art. Ground items can now be
  described as stacked bands; the fry stand, car and umbrella were killing players in empty
  sky beside their narrow upper halves.
- **15 Sep 2026** — Shops with something out front: the greengrocer (city) and souvenir
  stand (boardwalk). Needed `up:true` collision boxes so a single air obstacle can also
  own a box standing on the ground.
- **15 Sep 2026** — Four fixes: the palm trunk was drawn bottom-up while leaning, so its
  top landed 6px off the fronds (it now draws from the crown down); the boardwalk railing
  stopped being a spawnable 40px obstacle and became a continuous rail in the deck scenery;
  cars pick one of five colours per instance; and zone naming moved from weight-based to
  nominal starts, which was announcing the city nine seconds early.
- **15 Sep 2026** — Beach added as a fourth zone: lifeguard chair (its tall piece), umbrella,
  volleyball net, cooler, palm fronds, kite and a pier you fly under, over sand with a taller
  sea behind. Gulls carry over from the boardwalk. Added the Start-in preview control, since
  the beach begins at 107 m and nobody is going to reach it while iterating on it.
- **15 Sep 2026** — Trees were allowed in the park but could never appear there: overhead
  obstacles started at 1500px and the park stopped being dominant at 1000px. Air now starts
  at 1250px and the city at 2700px, so a tree shows up in the park in 80% of runs.
- **15 Sep 2026** — A bird per zone: bluejay, pigeon, gull. The boardwalk bunting was cut —
  a full-width string of pennants read as a giant banner across the screen rather than
  scenery, the same failure mode as the power lines it replaced. Overhead on the boardwalk
  is now the arcade sign plus the trees and lamps that carry through every zone.
- **15 Sep 2026** — The park bush is a clipped boxwood ball on a trunk, picked by Erik from
  five candidates (round shrub, boxwood, flowering, grass tuft, trimmed hedge) after two
  earlier attempts were rejected. Its item id is still `hedge` so saved rotations keep working.
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
- **15 Sep 2026** — The start sequence stopped eating its own instructions. One press used
  to clear the title *and* begin the fill, so the how-to was dismissed by the very tap that
  committed you — and the scrim over it hid the kid and the street you were about to play.
  Now: tap clears the title, tap fills, tap launches. The how-to moved to the title screen,
  `ready` became a bare "Tap to blow" over an undimmed scene, and overlays were re-anchored
  to the canvas so no prompt lands on the HUD (see above). Also cut ~160 lines of stale
  duplicate sections this file had been carrying, which still documented the `armed` state
  and the four-way style switch, both long removed.
- **15 Sep 2026** — Fullscreen on a phone became actually playable: the HUD, player row
  and tool bar are stripped below 560px of height (or on a coarse pointer), the distance
  readout moves into the corner of the canvas, and a ✕ appears, since stripping the tool
  bar removes the only way back out and phones have no Esc.
- **15 Sep 2026** — The bubble wand was lying on its side, loop beside the fist. Kids hold
  them upright, so `blowArm` now draws the handle running down from the loop into the
  hand, with the arm reaching out and down to meet it. The loop stays at mouth height —
  raising it to sit above the handle puts it over the kid's head, which Erik rejected
  the first time round. Every kid's `anchor.dx` went 19 -> 15 to follow the loop.
- **15 Sep 2026** — The kid now appears in the background of every zone (see above), the
  park got its own Victorian lamppost and the modern streetlamp was confined to the city
  and boardwalk, and the high-score dials open on the last initials used on that device
  rather than back at AAA every time.
- **15 Sep 2026** — The zone cameos were rebuilt. They started as free-floating background
  scenes; they are now drawn into the obstacles themselves (fountain, shop window, fry
  stand counter, beach umbrella, dinghy) on about a third of instances. The Ferris wheel
  that came with the first attempt went with it. Leafy bough tagged park/city so it stops
  hanging over the sea.
- **15 Sep 2026** — Benches and the wire bin tagged park/city/boardwalk. They had been
  untagged, so roughly 40% of beach ground spawns were park benches and city bins standing
  on the sand.
- **15 Sep 2026** — Bollard and the two off-by-default boughs tagged, so no obstacle is
  untagged any more. The zone banner moved from the leg start to the horizon handover: it
  was naming the city about nine seconds after the skyline had taken over.
- **16 Sep 2026** — Night. The crossing was extended to 3400px and the light now falls
  across it, holds through the whole way home, and comes back up over the park at the top
  of the next lap. Lamps, lit skyline windows and lit shopfronts are exempted from the
  tint so they read as the only warm thing on the street.
- **17 Sep 2026** — QA pass. Found and fixed: tapping a character portrait or the HUD
  started the game (see above); pausing from `ready` stacked two overlays; the board threw
  if the server answered with a string or a list of nulls; a corrupt save could set
  `RULES.ceiling` to any string. Confirmed *not* broken: corrupt localStorage in seventeen
  shapes, a rotation with everything switched off, leaderboard HTML injection, tab-away
  timestep blowups, and 4000 simulated seconds of random input without a crash, a NaN or
  an illegal state transition.
- **17 Sep 2026** — Fullscreen got a pause button beside the ✕. Without it, phone
  fullscreen had no way to pause at all: no tool bar, no keyboard.
- **24 Sep 2026** — Second QA pass. Collision became a true circle test; twelve hitboxes
  were refitted to their art (never growing); the squall got per-variant boxes; night got
  a hazard mask and a moonlight rim so obstacles stay visible after dark.
