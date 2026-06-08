---
name: markgraf
description: Authoring rules + grammar reference for the markgraf animation language. Use when the user is writing or editing .markgraf files, talks about markgraf, asks about animations/animated diagrams of systems, or works on a graph diagram described as nodes/edges/tokens/frames flowing through a system. Covers syntax (frames, +node/-node, +edge/-edge, tokens with `->` / `<-`, par/seq blocks, seed, # comments), strict-mode validation rules, and authoring principles (C4 levels, short labels, par for simultaneity, one concept per frame, chained tokens for flows). Also explains how to preview and parse-check animations via the markgraf CLI.
---

# markgraf-cli — authoring animations

`markgraf` renders short animated graph diagrams from a tiny declarative source
language (`.markgraf` files). Frames describe **what the system looks like at
moments in time**; the engine animates the transitions between frames and
overlays data-flow tokens that travel along edges within a frame.

This skill is for writing **good** animations — pacing, narrative, layout
decisions — not just syntax. CLI flags and grammar appear at the bottom for
reference.

---

## Mental model

Think of a `.markgraf` file as a **slideshow script for a system diagram**.
There are two distinct kinds of beats:

1. **Structural beats** — the *shape* of the graph changes (a node or edge is
   added, removed, or relabelled). The renderer animates the topology change:
   nodes plop in, edges grow.
2. **Flow beats** — the shape stays put; data tokens travel along edges to
   tell a story (a request hits the API, the API queries the DB, …). Tokens
   are little circles that morph out of one node, slide along the edge, and
   morph into the next node.

A single `keyframe { … }` block can mix both — the engine sorts the structural
operations to the start of the frame and runs flow ops afterwards. But a clean
mental model is: **one beat per frame**. Mixing structural and flow inside one
frame works but is harder to read on the page.

### Principles (and where they come from)

The rules below aren't taste — they fall out of a small set of
results from graph drawing and perception research. Knowing the *why*
helps you judge edge cases.

- **A diagram is a claim, not a noun-pile.** Pick one question the
  diagram answers ("what calls what?", "what contains what?", "what
  flows where at runtime?") and commit. The default AI failure mode is
  conflating call-graph, dependency, and deployment edges into one
  ambiguous "→". Source: practitioner consensus (Brown's C4 model,
  ArchiMate); not measured, but load-bearing.

- **The y-axis means something.** In a layered (Sugiyama) layout,
  vertical position = causal/dependency depth. Putting `main` "in the
  middle because it looks balanced" silently lies about the structure.
  Source: Sugiyama, Tagawa, Toda (1981); this is what ELK implements
  and therefore what markgraf inherits.

- **Edge crossings dominate readability.** Of all the aesthetic
  criteria (crossings, bends, symmetry, angular resolution), crossings
  hurt comprehension by the largest measured margin; bends are second.
  Source: Helen Purchase, "Which Aesthetic has the Greatest Effect on
  Human Understanding?" (1997). Practical consequence: let the layout
  engine fight crossings — don't hand-place to make something "look
  tidy" if it adds a crossing.

- **Containment and adjacency are different channels.** Nesting (box
  inside box) means "is part of"; an edge means "talks to / depends
  on". Using both for the same relationship is incoherent. Source:
  Gestalt proximity + common region (Wertheimer 1923; applied to
  visualization in Colin Ware, *Information Visualization: Perception
  for Design*).

- **Object constancy is what makes animated transitions teach.**
  Animated transitions help comprehension *only* when objects preserve
  identity across the change and concurrent things are visibly
  concurrent; staged transitions beat "everything moves at once".
  Source: Heer & Robertson, "Animated Transitions in Statistical Data
  Graphics" (2007). This is the empirical basis for `par`/`seq` and
  the token chaining behaviour.

- **Common fate groups moving things.** Tokens travelling together
  read as one logical message; tokens travelling separately read as
  unrelated events. Source: Gestalt common fate (Wertheimer); validated
  for displays in Ware. Practical consequence: don't `par` two
  unrelated tokens just to save a frame — the eye will fuse them.

- **Tracking budget is small.** Viewers can reliably track ~3–8
  independently moving objects, fewer when they're fast or crowded.
  Source: Pylyshyn & Storm (1988); Alvarez & Franconeri (2007). For
  markgraf: more than ~4 simultaneous tokens in a `par` is a smell.

- **Hold time after a state change is real.** Motion-design folklore
  (not measured): every meaningful change deserves ~300–500ms of
  stillness so the eye can consolidate. This is why
  `minTokenDuration: 1.4s` exists and why "speeding up tokens" rarely
  fixes a crowded frame — splitting frames does.

- **"5–9 boxes per view" is folklore, not Miller.** Miller's 1956
  "7±2" was about working memory for unrelated items, not visual
  chunking. The diagramming convention is good advice anyway: at the
  top level, fewer than ~9 nodes lets a viewer hold the whole shape in
  their head. Treat it as a budget, not a law.

### Pick a C4 level before you start

Simon Brown's C4 model (practitioner convention, not measured, but
widely adopted) gives a vocabulary for *which* diagram you're drawing.
A markgraf animation should sit at exactly one level — mixing them is
the fastest way to produce a noun-pile:

| Level | Boxes are | Audience | Good markgraf use |
|---|---|---|---|
| **1. Context** | the system + external actors/systems | non-technical stakeholders | "user hits our product, which talks to Stripe and Postgres" |
| **2. Container** | deployable/runnable units (web app, API, DB, queue) | technical, on/near the team | most architecture animations live here |
| **3. Component** | major groupings inside one container | developers on that container | "inside the API: auth middleware → handler → repo" |
| **4. Code** | classes/functions | rarely worth drawing | skip — IDE does this better |

Concrete rules for markgraf:

- **State the level in the first frame name.** `keyframe "container view"`
  or `keyframe "component view: api"`. The viewer needs to know what
  altitude they're at before the first token moves.
- **Don't cross levels in one file.** If you need to "zoom in", make a
  *separate* `.markgraf` — the C4 insight is that each level is its
  own diagram, not a sub-region of a bigger one. Linking is editorial,
  not visual.
- **Every box has a type.** Container-level boxes are *not*
  interchangeable: a database behaves differently from a queue from a
  stateless service. Encode the type in the label (`"Postgres"`, not
  `"DB"`; `"SQS"`, not `"Queue"`) — markgraf has no shape vocabulary,
  so the noun has to carry it.
- **Every edge has a purpose + protocol.** C4's rule for static
  diagrams ("Reads from, [JDBC]") maps to markgraf as: the token
  label should name the *operation* (`"SELECT"`, `"publish"`,
  `"GET /user"`). If the protocol matters (sync vs async, RPC vs
  event), put it in the token label itself (`"publish (async)"`).

The deepest C4 idea worth internalising: **a diagram has an implicit
"above" and "below"**. If you can't say "this is the container view;
the component view would expand the API box", you're not at a level
— you're just drawing.

### Refuse pipeline-ism

The most common AI failure: every node has one in-edge and one
out-edge, the whole diagram is `A → B → C → D → E`. Real systems are
not pipelines. If your sketch looks like a pipeline, you are
documenting **one user journey** through the system, not the system
itself.

Before writing, list:

- **The hub.** What single thing does most of the work? It should
  have ≥3 edges, often more. (A pipeline has no hub.)
- **Fan-out points.** Where does one event become many — broadcasts,
  parallel calls, write-then-also-log, primary + cache + analytics?
- **Fan-in points.** Where do many sources funnel into one — every
  service writing to one log, every request landing on one ingress?
- **Back-edges and cycles.** Caches that invalidate, retries, queues
  that feed services that publish back. Real systems have them.
  Sugiyama will draw them with a distinct style; don't omit them just
  because they break the top-to-bottom story.
- **The thing that talks to everyone.** Auth, config, observability —
  often deliberately drawn off to one side with edges to many other
  nodes, so the topology *looks* the way it *is*.

If your diagram lacks all of these, you've drawn a sequence diagram in
disguise. Either pick a different claim (this *is* a user journey —
fine, but say so) or zoom out / sideways to find the real shape.

Concrete signs the diagram is too linear:

- Every frame contains exactly one token, going one hop.
- No `par` anywhere.
- No chained `seq` flow (the headline reason chaining exists is to
  render a multi-hop request as one continuous motion — if no frame
  uses it, you've made the diagram more rigid than the system).
- Every node has in-degree ≤ 1 and out-degree ≤ 1.
- The whole thing reads top-to-bottom with no sideways glance.

### Plan before you write a single frame

This is the step that separates correct diagrams from amazing ones. Do
this work *in your head or as a comment* before any `keyframe { … }` is
written. If you can't answer all six, you don't have a diagram yet.

1. **The claim.** One sentence: "this animation shows that X." If you
   can't finish that sentence, you're not ready.
2. **The level.** Context / Container / Component (rarely Code).
3. **The hub.** Which one node has ≥3 edges and would lose the most
   if removed? Name it. If you can't, you might be drawing a sequence
   not a system.
4. **The surprise.** What does the viewer not yet know that they
   *should* know after watching? The cache they didn't expect, the
   back-edge they didn't see coming, the fan-out hidden behind a
   single arrow on the static diagram. **No surprise → no
   animation.** A non-surprising diagram should be a still image.
5. **The shape of the story.** A pacing template that works:
   *setup* (one frame, all the boxes) → *naive case* (a flow that
   "obviously" works) → *the catch* (what breaks, what's slow, what's
   missing) → *the fix* (structural change introducing the new thing)
   → *rerun* (the same naive flow, now exercising the new shape).
   Five frames, one punchline. Other templates exist but this one is
   the workhorse.
6. **The frame budget.** Aim for **5–7 frames**. Eight is the upper
   limit. If you have ten, you've written a slide deck — go cut.
   Every frame you cut makes the survivors louder.

### Introduce nodes one beat at a time

Don't dump the whole topology in the setup frame. Frontloading all
the nodes overwhelms the viewer before anything has *happened* —
they're trying to parse five labels they have no reason to care
about yet. Instead, **introduce each node at the moment the story
needs it**, and ideally show it doing something in the same frame:

- Frame 1: the two ends of the simplest possible flow (e.g. `Client`
  + `API`). One token, one beat. The viewer locks in.
- Frame 2: introduce the next thing the story needs (e.g. `Database`
  for the slow source of truth) and immediately run a flow that uses
  it. Each new node *earns* its introduction by being used.
- Continue until all the actors are on stage.

This pattern composes naturally with the *setup → naive → catch →
fix → rerun* template: the "naive" stage usually only needs 2-3
nodes; the "catch" stage introduces the thing that creates the
problem; the "fix" stage adds the thing that solves it. Each
structural change is a *reveal*, not a precondition.

A useful rule: if a node never appears as the *star* of a frame
(either introduced there, or central to the action), it probably
shouldn't exist. Cut it.

### Dramatic structure (why animations work at all)

A static diagram answers "what is the shape?" An animation answers
"what *happens*?" — which is only worth doing when the answer has a
moment of *change*. Three pillars:

- **Setup → tension → release.** The setup frame establishes the
  shape. Mid-frames create tension (a slow path, a missing piece, a
  fan-out that overwhelms). The final frame releases it (the new
  shape handles the load, the new edge eliminates the round-trip).
  Without tension, the viewer has nothing to anticipate; without
  release, nothing to remember.
- **Show, then show again.** Run the *same flow* before and after a
  structural change. The viewer's eye locks onto the difference. This
  is why the cache example in the reference works: same `GET /user/42`
  before and after; the cache is the punchline.
- **One revelation per animation.** If you're trying to show three
  surprises, you have three animations to write. Pick the one the
  viewer needs most and cut the others — they'll dilute the impact of
  the one that matters.

### What looks good

- **One concept per frame.** "introduce cache" should *only* introduce the
  cache. Don't bolt a flow on the same beat — the viewer is busy parsing the
  shape change.
- **Always name your frames.** `keyframe "introduce cache" { … }` reads as a
  table of contents at the top of the file. Unnamed frames are anonymous in
  the player's keyframe scrubber.
- **Use `par { }` to make things happen at the same time.** Default sequencing
  is `seq` (one after another). `par` is what gives the animation its
  liveliness — two tokens leaving a node at the same instant read as a
  fan-out, not as two unrelated steps.
- **Let `seq` carry causation.** Inside a flow frame, `client -> api` followed
  by `api -> db` *automatically* chains: the second token leaves the API the
  instant the first arrives. This is what makes "request flows down through
  the stack" feel right; there's no explicit timing, the engine reads the
  topology of the block.

### Rules of thumb (the ones that matter most)

**Keep labels short.** Token and node labels render at a fixed text
size onto small graph elements. A token chip showing `"GET /user/42 with
session token"` either overflows or squishes the text to unreadable. Aim for:

- **Tokens (the chip on a flowing dot):** verb + small noun. `"GET /user"`,
  `"SELECT"`, `"publish"`. Roughly ≤ 16 chars; never more than 24.
- **Node labels:** noun. `"API"`, `"Cache"`, `"Order Service"`. Two words max.

**Use `par { }` for things that happen together.** Default sequencing is `seq`
(one after another). The animation only feels like a *system* if concurrent
things actually appear concurrent — two tokens leaving a node in the same
instant read as a real fan-out; the same two ops in series read as
"and then, separately, also...".

```
par {
  api -> cache "HIT"      # both legs leave the API
  api -> logger "trace"   # in the same instant
}
```

**Don't reach for `par` just for visual balance.** If "B happens *after*
A's response", that's `seq`, not `par`. Hedging ("in some deployments
this is parallel") is a sign you're at the wrong C4 level or drawing
two diagrams at once.

`par` and `seq` nest. The frame body is implicitly a `seq`, so you only
write `seq { … }` when you need an inner sequential block inside a `par`.

**Chain by default.** Most multi-hop flows should be a chain, not a
sequence of independent tokens. If the *to* of one token matches the
*from* of the next within a `seq` (the default), the engine renders
ONE continuously travelling dot — the literal animation of a request
flowing through the stack. This single mechanic is what makes
markgraf feel different from a sequence diagram. Use it.

```
keyframe "request" {
  client -> api "GET"            # chain: dot leaves client, passes through
  api -> db "SELECT"             # api, terminates at db — one motion
}
```

When you have `client -> api` then `api -> db`, **always** chain them
unless you mean "api processes the request, *then later* talks to
db." Inserting an unrelated op or an explicit `seq` block between
them breaks the chain visually for no reason. If you find your
animation has lots of one-hop frames, you've broken every chain.

**Prefer `<-` over a second edge for responses.** If `+edge a b`
exists, the reply flows back as `a <- b "reply"`, *not* by adding
`+edge b a` and writing `b -> a "reply"`. The latter says "there are
two parallel channels"; the former says "same channel, the reply
comes back." 99% of request/response architecture uses one channel.
Adding a second edge for the response visually doubles the topology
and lies about the system. **Reach for `<-` automatically.**

Break the chain (use `par`, or have the next leg start from a different
node) when the flow actually fans out or branches.

**Anti-patterns:**

| Anti-pattern | Why it's wrong | Fix |
|---|---|---|
| 8 frames showing one diagram appear piece by piece before any flow | Viewer is lost before the demo starts | One `setup` frame; start the story with the first flow |
| Long flow with no `par` | Reads as an itemised list, not a system | Group concurrent legs in `par` |
| `par` chosen for balance ("only used seq so far") | Misreads sequence as concurrency | Use `seq` unless two events genuinely happen in the same instant |
| Token going against an edge using `->` | Parser rejects: "no edge between X and Y" | Flip to `<-` (`a <- b "reply"`) — same edge, reverse motion |
| Structural + flow in one frame | Two concurrent beats, neither reads | Split: structural frame, then flow frame |
| Multiple identical-direction tokens between same pair | Looks like one flickering dot | Use one chained sequence, or `par` if they're truly concurrent |

### Strict-mode validation (what the parser will reject)

markgraf is strict — the CLI's `--check` flag fails fast on any of these,
so you can verify your file before declaring done.

- **Every token needs an edge.** `a -> b "msg"` requires an edge
  between `a` and `b` that is currently visible (added by `+edge`, not
  yet removed by `-edge`). No edge → `[ERROR] token a→b: no edge
  between a and b`.
- **Token direction must match or use `<-`.** If the edge is `+edge a b`
  and the message flows from `b` back to `a`, write `a <- b "reply"`,
  not `b -> a "reply"`. The reverse-arrow says "same edge, reverse
  motion" — it does not create a second edge.
- **Structural ops can't reference unknowns.** `+edge x y` requires
  both nodes to exist (added by an earlier `+node`, not yet removed).
  `-node x` and `-edge x y` require x (and the edge) to currently
  exist. `~edge a b -> c d` (repoint) requires the old edge and both
  new endpoints to exist.
- **Frame names must be unique** within a file.
- **No bubbles.** `+bubble`/`-bubble` syntax was removed; don't write
  it. The commentary that used to live in bubbles now belongs in the
  token label itself, in a node name, or — best — in the topology
  (introduce the queue node, don't bubble "async" onto a generic edge).

---

## Language

A **frame** is one beat in the animation. Statements inside a frame either
change the graph shape (structural) or move data along it (flow).

### Frames

```
keyframe setup { +node a "A" +node b "B" +edge a b }
keyframe "first request" { a -> b "hello" }
```

Names are unquoted identifiers (`setup`, `cache_hit`) or quoted strings
(`"first request"`). Use quotes for spaces.

### Adding and removing nodes

```
+node api "API"        # introduce node
-node api              # remove node
```

**Three label syntaxes — pick the right one:**

```
+node api "API"                          # quoted: short titles
+node api : Authorization API            # colon: rest-of-line, no quotes needed
+node api |API
                                          # pipe: multi-line, terse description
holds session tokens
proxies to all internal services|
```

**Node labels stay short — a keyword noun, two words max.** The
diagram has limited space inside each box; cramming sentences into
nodes makes the layout busy and steals attention from the moving
parts. Narration lives on the *tokens*, not the boxes. A node's job
is to be a stable landmark the eye can navigate around.

```
+node cache "Cache"           # good — short, recognisable
+node cache "Redis Cache"     # also fine — type-bearing
+node cache |Cache            # BAD — don't do this
LRU keyed by user id
evicted after 5min|
```

The `|...|` and `:` forms exist for nodes mostly so labels with
special characters (colons, commas, parens) don't need awkward
quoting — *not* so you can write paragraphs in boxes. If you find
yourself wanting to explain what a node *does*, that belongs on the
next token that touches it.

**Multi-label token carousels:** tokens accept multiple labels in
sequence; the chip cycles through them while the token travels.
Useful for showing a request's evolution (`"GET /user"` → `"check
ACL"` → `"200"`):

```
client -> api "GET /user" "check ACL" "200"
```

Don't overuse — three labels max, and only when each carries different
information.

**Token labels accept `|…|` multi-line blocks — and this is the
default, not the exception.** Multi-line tokens are how the chip
*narrates* instead of merely *labelling*. They render as a
Nintendo-style speech bubble: each `\n`-separated line cycles through
the chip in sequence while the dot travels along the edge, one line
visible at a time. The viewer reads them as a single thought
accreting.

```
api -> cache |LOOKUP user:42
checks the cache,
since reads should be fast|
```

The viewer sees three frames of chip text in order:
`LOOKUP user:42` → `checks the cache,` → `since reads should be fast`.

#### Five rules that make tokens read like English

These are the rules that separate good multi-line tokens from labels
that feel like disconnected bullet points.

**1. Line 1 is the operation. Lines 2-N are the present-tense
description of *that* operation.** Don't switch topic between lines.

```
# Good — one continuous sentence.
api -> cache |LOOKUP user:42
checks the cache,
since reads should be fast|

# Bad — three unrelated observations stapled together.
api -> cache |LOOKUP user:42
the simplest possible read
just the two of us so far|
```

**2. Stay in the moment. No timeline narration.** Frame names carry
chronology; tokens describe their own hop. Phrases like *"moments
later"*, *"the same read as before"*, *"now with the new cache"* are
narrator-voice — they belong in the frame name, not the chip.

```
# Good — describes what THIS token is doing.
client -> api |GET /user/42
asks for the row we
just overwrote|

# Bad — meta-commentary about timeline.
client -> api |moments later,
another GET for user 42|
```

**3. Break at clause boundaries, not topic boundaries.** Read the
block aloud as one sentence. If you naturally pause between two
lines, that's where the line break belongs. If the two lines could
swap order without breaking meaning, they aren't a sentence —
they're a bullet list, and you've written the wrong thing.

```
# Good — commas and dashes carry the eye into the next line.
client <- api |200 OK
but the row is stale --
the user sees old data|

# Bad — three independent statements.
client <- api |200 OK
cache returned old data
this is the bug|
```

**4. Don't mix registers within a block.** If line 1 is code
(`POST /user`, `SELECT`, `HIT`), lines 2-N describe *that code's
effect* in plain language — they don't switch to narrator commentary
about the system as a whole.

**5. Three lines, ≤ 30 chars each.** The chip cycles through your
lines roughly every ~0.5s. A line longer than ~30 chars rushes the
reader; more than 3 lines makes the cycle outlast the dot's flight,
leaving the viewer staring at trailing text after the action is
over. If you need more, you've packed two ideas into one token —
split into two consecutive tokens, each with its own short block.

**When a single word is enough:** glue tokens whose only job is to
move the dot along a chain you've already explained.  `"SELECT"`
after a multi-line "the cache missed, so we fall through" is fine —
the previous beat set up what's about to happen. The rule is **earn
brevity from context**, not from terseness for its own sake.

**Multi-label carousels vs. multi-line blocks.** Quoted multi-labels
(`a -> b "x" "y" "z"`) and a single `|…|` block both cycle the chip
through several texts. Use multi-labels when the texts are *discrete
beats* (e.g. `"GET /user" "200 OK" "cached"` — three independent
status snapshots). Use `|…|` when the texts are *one sentence in
pieces*. Don't mix the two in a single token unless you really need
both effects.

(`:` rest-of-line is *not* supported on tokens — it would eat the next
token in a chained sequence. Use `|…|` or quoted strings.)

### Adding and removing edges

```
+edge api db
-edge api db
```

Edges are directional — `+edge a b` ≠ `+edge b a`. The arrowhead points from
`from` to `to`.

### Tokens (data flow)

```
client -> api "GET /user"
api -> db "SELECT"
```

Each statement renders a circle that morphs out of the source, slides along
the edge, and morphs into the target. Consecutive tokens that chain (the
`to` of one matches the `from` of the next) render as ONE continuously
travelling dot — that's how requests feel like one motion through a stack.

### Reverse-direction tokens: `<-`

```
client -> api "GET"        # forward along +edge client api
client <- api "200 OK"     # reverse along the same edge
```

`<-` says "same edge, motion reversed". It does *not* create a second
edge. Use it for every response/reply that flows back along an edge
already pointing the request way.

### Concurrency: `par` and `seq`

```
keyframe "cache hit" {
  client -> api "GET"
  par {
    api -> cache "HIT"
    api -> logger "trace"   # both leave the API in the same instant
  }
  client <- api "value"
}
```

The frame body is implicitly `seq` — children run one after another. Wrap
children in `par { … }` to play them at the same time. `par` and `seq` nest:

```
par {
  seq { api -> svcA; svcA -> db }    # this leg is a chain
  seq { api -> svcB; svcB -> cache } # happening at the same time
}
```

### Top of file: `seed`

```
seed 1                  # optional, controls layout RNG (default 0)

keyframe setup { ... }
```

### Comments

```
# Line comments start with '#' and run to end of line.
+node a "A"     # trailing comments work too
```

### Dives (C4 drill-down)

A **dive** flies the camera *into* a node to reveal a sub-diagram hidden behind
it, then back out — the animated equivalent of zooming from a C4 Context diagram
into Containers, then into Components. Three pieces:

- **`inside <node> { … }`** — a top-level block (a sibling of `keyframe`, not nested
  in one) that holds the sub-diagram behind `<node>`. Its body is an ordinary
  document of frames. These blocks **nest to any depth**.
- **`enter <node>`** — a statement inside a frame. It flies the camera into
  `<node>` and then plays that node's `inside { … }` sub-film in full.
- **`exit`** — a statement that flies the camera back out to the parent level
  after the sub-film. Pair it with `enter` in a short transition frame.

```
# Level 1 — Context
keyframe "the system" {
  +node customer "Customer"
  +node bank "Internet Banking"
  +edge customer bank
}

keyframe "zoom in" {
  enter bank          # dive into `bank`, play its interior, …
  exit                # … then fly back out
}

# Level 2 — Containers, hidden behind `bank`
inside bank {
  keyframe "the moving parts" {
    +node spa "SPA"
    +node api "API"
    +edge spa api
  }

  keyframe "deeper" {
    enter api
    exit
  }

  # Level 3 — Components, hidden behind `api`
  inside api {
    keyframe "controllers" {
      +node signin "Sign In"
      +node security "Security"
      +edge signin security
    }
  }
}
```

Rules: the node you `enter` must exist in the parent level and must have a
matching `inside` block; dives nest by nesting `inside` blocks. `--check`
validates all of this.

---

## Install

```
brew install --cask i-am-the-slime/tap/markgraf
```

darwin-arm64 only as of v0.1.x. The cask handles quarantine stripping;
no Apple Developer signing yet. If `brew install` complains about a CLT
mismatch on macOS 26, the user needs to update Command Line Tools.

## CLI reference

```
markgraf [INPUT] [-o out.mp4] [--fps 60] [--scale 2.0] [--theme NAME]
         [--play] [--terminal] [--sequence PATH] [--check] [--version]
```

`INPUT` is optional: omit it to read from stdin (`pbpaste | markgraf --play`),
pass a path to read a file, or pass `-` for explicit stdin.

| Flag | Default | Notes |
|---|---|---|
| `INPUT.markgraf` | required | path to the source file |
| `-o, --output PATH` | `out.mp4` | output video path; container chosen by extension |
| `--fps INT` | `60` | render frame rate |
| `--scale NUMBER` | `2.0` | resolution multiplier; `2.0` is retina-quality |
| `--max-width PX` | — | cap output width in pixels; diagram aspect preserved (fits inside the max-width/max-height box) |
| `--max-height PX` | — | cap output height in pixels; diagram aspect preserved |
| `--min-text-px PX` | `12` | minimum on-screen text size; the camera zooms in further on small outputs to keep labels readable. `0` disables it (pure fit-all framing). |
| `--theme NAME` | `light` | look: `light \| dark \| blueprint \| whiteboard \| isometric` — see **Themes** below |
| `--play` | off | open the native player window instead of encoding. On darwin this is the Metal/AppKit "liquid glass" player by default; set `MARKGRAF_RENDERER=ebiten` to force the cross-platform ebiten player. |
| `--terminal` | off | play the animation in the terminal as ANSI (space: pause, ←/→: step, q: quit) |
| `--sequence PATH` | — | render as a static UML sequence-diagram PNG (no time axis, one image) |
| `--check` | off | parse + validate only; print `OK` (exit 0) or `[ERROR] …` (exit 1). No rendering. Use this to typecheck your file. |
| `-v, --version` | off | print version and exit |

### Themes

`--theme` picks the visual look; it does **not** change your source. The diagram
is identical across themes — only the rendering differs.

| Theme | Look |
|---|---|
| `light` | default — clean light background, solid node fills |
| `dark` | dark background variant of `light` |
| `blueprint` | technical blueprint overlay (blue grid, drafting look) |
| `whiteboard` | hand-drawn sketch — roughened strokes, marker fills, eraser sweeps on `-node` |
| `isometric` | extrudes every node into a 3D slab on a ground plane; edges run on the floor, labels sit on the slab faces |

The theme is orthogonal to the grammar — author your `.markgraf` once and re-render
with any `--theme` to change the aesthetic.

ffmpeg is embedded — no system dependency. CLI is darwin-arm64 only.

---

## Reference: a complete example

```
seed 1

keyframe setup {
  +node client "Client"
  +node api "API"
  +node db "Database"
  +edge client api
  +edge api db
}

keyframe "direct read" {
  client -> api "GET /user/42"
  api -> db "SELECT"
  api <- db "rows"
  client <- api "200 OK"
}

keyframe "introduce cache" {
  +node cache "Cache"
  -edge api db
  +edge api cache
}

keyframe "cache hit" {
  client -> api "GET /user/42"
  api -> cache "HIT"
  api <- cache "value"
  client <- api "200 OK"
}

keyframe "introduce queue" {
  +node queue "Queue"
  +edge api queue
}

keyframe publish {
  client -> api "POST /order"
  api -> queue "publish (async)"
}
```

Read top to bottom: structural setup → simple flow → structural change →
flow showing the *new* path → another structural change → flow with async
fan-out. Each frame earns its place by adding one idea.

This is the *setup → naive → catch → fix → rerun* template in concrete
form. The "catch" is implicit (the naive `direct read` goes to the
DB every time — visibly the slow path); the "fix" is `introduce
cache`; the "rerun" is `cache hit` showing the new shape exercised by
the same request.

---

## Good vs. great: the same diagram, two ways

Same task — "explain how our service handles a write": same actors,
same outcome. The first version passes `--check` and is informative.
The second teaches.

### Good (correct, dull)

```
keyframe setup {
  +node client "Client"
  +node api    "API"
  +node db     "Database"
  +node cache  "Cache"
  +edge client api
  +edge api db
  +edge api cache
}

keyframe "write request" {
  client -> api "POST /user"
  api -> db "INSERT"
}

keyframe "invalidate cache" {
  api -> cache "DEL user:42"
}

keyframe "respond" {
  client <- api "201"
}
```

Four frames. Each does one thing. Parses, renders, viewer comes away
with "API writes to DB and invalidates the cache, then responds."
Fine. Not memorable.

### Great (same content, structured, narrated)

The full source:

```
seed 1

keyframe "a simple read" {
  +node client "Client"
  +node api    "API"
  +edge client api

  client -> api |GET /user/42
asks the API
for one user record|
}

keyframe "DB joins the story" {
  +node db "Database"
  +edge api db

  client -> api |GET /user/42
arrives at the API|

  api -> db |the API falls through
to the source of truth|

  api <- db |the row comes back,
fresh from disk|

  client <- api |200 OK
correct, but every read
costs a DB round trip|
}

keyframe "add a cache to speed up reads" {
  +node cache "Cache"
  +edge api cache
}

keyframe "naive write: forget the cache" {
  client -> api |POST /user/42
updates the user's row|

  api -> db |INSERT
writes to the DB,
the source of truth|

  client <- api |201 Created
looks fine, but the cache
still holds the OLD row|
}

keyframe "the catch: a stale read" {
  client -> api |GET /user/42
asks for the row we
just overwrote|

  api -> cache |LOOKUP user:42
checks the cache,
since reads should be fast|

  api <- cache |HIT
the cache returns
the pre-write value|

  client <- api |200 OK
but the row is stale --
the user sees old data|
}

keyframe "fix: invalidate on write" {
  client -> api |POST /user/42
updates the user's row|

  api -> db |INSERT
writes to the DB|

  par {
    api -> cache |DEL user:42
drops the cache entry
so the next read refills|

    client <- api |201 Created
responds in parallel,
no waiting on the cache|
  }
}

keyframe "rerun: same GET, now correct" {
  client -> api |GET /user/42
asks for the row again|

  api -> cache |LOOKUP user:42
checks the cache first|

  api <- cache |MISS
the cache was just
invalidated|

  api -> db |SELECT
falls through to the DB|

  api <- db |the fresh row
comes back from disk|

  client <- api |200 OK
the user sees the
correct, current row|
}
```

The shape:

- **F1 "a simple read":** introduce `Client` + `API` only. One
  token: `client -> api |GET /user/42 / the simplest possible read
  / just the two of us so far|`. Two nodes, one beat, viewer locked
  in.
- **F2 "DB joins the story":** introduce `Database`. Show the full
  client→api→db→back round trip with multi-line tokens that
  narrate "slow but always correct". Now the viewer cares about
  performance.
- **F3 "add a cache to speed up reads":** introduce `Cache` as a
  structural change only. One frame, one reveal, no flow yet.
- **F4 "naive write: forget the cache":** the write happens.
  Multi-line tokens narrate "doesn't touch cache", "looks fine!",
  "but the cache is now stale". The setup for the catch.
- **F5 "the catch: a stale read":** the same `GET /user/42`. The
  cache returns stale. Multi-line `200 OK (stale!)` token tells the
  viewer: this is the bug.
- **F6 "fix: invalidate on write":** the same write, but with
  `par { DEL user:42, 201 Created }` — invalidation and response in
  parallel.
- **F7 "rerun: same GET, now correct":** chained miss→DB→fresh row,
  returning `200 OK (fresh)`. The viewer literally watches the system
  do the right thing.

**What's different from the "good" version:**

- **Nodes introduced one at a time.** Each new actor enters when the
  story needs it. The viewer's mental model accretes; nothing's
  dumped on them upfront.
- **Tokens narrate.** Every meaningful token is a `|…|` block of 2-4
  lines. Node labels stay one-word.
- **Naive → catch → fix → rerun structure.** F4-F5 set up the bug
  before the fix exists; F6-F7 show the fix working on the same flow.
- **`par` in the fix frame** makes "invalidate and respond at the
  same time" visually true.
- **Chained tokens in F7** (cache MISS → DB SELECT → fresh row →
  200): a single moving dot navigates the new shape.
- **Reverse tokens (`<-`)** throughout. One edge per channel.
- **Frame names narrate.** "the catch", "fix", "rerun" — the viewer
  always knows what each beat is *for*.

The difference between these is not skill at markgraf; it's whether
you did the planning step at the top before writing, and whether
you trusted multi-line tokens to carry the explanation.

---

## Iteration

You are writing this without seeing the rendered output. Use the CLI's
built-in checker — it's how you typecheck markgraf.

### Always run `--check` before declaring done

After writing or editing any `.markgraf` file, run:

```
markgraf path/to/foo.markgraf --check
```

- Exit 0 + `OK` → parse + strict validation + schedule build all passed.
  The file will at least *open* in the player.
- Exit 1 + an `[ERROR] …` message → fix the reported issue and re-run.
  Common errors and their meaning are in the **Strict-mode validation**
  section above. Don't ship a file that doesn't pass `--check`.

`--check` does not render; it's fast and safe to run repeatedly.

### Then read it as prose

`--check` catches mechanical errors. It does not catch *taste* errors:

- Read each `keyframe { … }` aloud as a sentence in plain English ("the
  API fans out: it queues a publish *and* logs a trace"). If you
  can't, the frame is doing too much — split it.
- Default timing is tuned for a "narrating-while-presenting" cadence
  (`tokenSpeed: 100`, `minTokenDuration: 1.4s`). If a flow has too many
  events, the fix is splitting frames, not speeding up tokens.
- Textual tells `--check` won't catch: long labels, missing `par` for
  genuinely concurrent things, `par` used for visual balance, deeply
  nested blocks, structural+flow in one frame, hedging in labels ("in
  some deployments…"). Spot these from the source alone.

### Kill your darlings (the cut pass)

After `--check` passes, do one mandatory pass where you **only cut**.
Adding more is forbidden in this pass. For each frame, ask:

- **"Could I delete this frame and lose nothing essential?"** If yes,
  delete it. Don't argue with yourself.
- **"Is this token a hop or an explanation?"** If a hop, can the
  surrounding chain absorb it? If an explanation, does it deserve a
  `|…|` block, or can it go on a node label instead?
- **"Does this node appear in the punchline?"** If a node is
  introduced but never participates in the surprise/reveal, it might
  be decoration. Cut, or move it to setup with no fanfare.
- **"Did I solve a problem the viewer didn't have?"** Frames added
  "for completeness" usually make the animation slower without making
  it clearer.

A useful target: cut at least one frame on every pass. If you can't,
your draft was already tight. If you keep cutting and the animation
still reads, your *next* draft will be tighter still. The cost of one
cut frame is essentially zero; the cost of one bloated animation is
that nobody watches it twice.

### Compare to the template

After cutting, check your animation against the workhorse template
from "Plan before you write": *setup → naive → catch → fix → rerun*.

- Did you do setup as one frame, or did you dribble nodes in?
- Did you actually show the naive case before introducing the fix?
- Is the catch *visible* (a slow path the viewer can see)?
- Did you re-run the naive flow against the new shape, so the viewer
  watches the fix work?
- Or did you ship a "here are the boxes, here are the arrows" tour?

If the answer is the tour, you have a diagram with an `--play` button,
not an animation. Restructure or accept that this should be a still
image instead.

### Open the preview yourself

After writing or editing a `.markgraf` file, **open the player for the user**
— don't just tell them the command. The player is a long-running window so
launch it with the Bash tool's `run_in_background: true`:

```
markgraf path/to/foo.markgraf --play
```

(use the Bash tool with `run_in_background: true` and a short description
like "preview foo.markgraf"; the window opens immediately, the user sees the
animation, and you don't block waiting for them to close it).

If the user has not saved a file (e.g. they pasted a snippet inline), pipe
the source into stdin instead — same `run_in_background: true`:

```
echo '<the source>' | markgraf --play
```

(`bash -c` so the pipe is interpreted; or write the snippet to a tempfile
first if quoting gets ugly).

You should still **mention** the same command in your text reply, so the
user can re-run it later without scrolling back. But don't make them copy-
paste to see what you just wrote.

In the player: `space` toggles play/pause, arrow keys scrub frames,
`r` restarts, `q` quits. Drag-and-drop a different `.markgraf` file onto
the window to reload from that file.

