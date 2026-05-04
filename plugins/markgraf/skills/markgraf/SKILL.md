---
name: markgraf
description: Authoring rules + grammar reference for the markgraf animation language. Use when the user is writing or editing .markgraf files, talks about markgraf, asks about animations/animated diagrams of systems, or works on a graph diagram described as nodes/edges/tokens/frames flowing through a system. Covers syntax (frames, +node/-node, +edge/-edge, tokens with `->`, +bubble/-bubble, par/seq blocks, seed, # comments) and authoring rules (short labels, par for simultaneity, one concept per frame, chained tokens for flows). Also explains how to preview animations via the markgraf CLI.
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

A single `frame { … }` block can mix both — the engine sorts the structural
operations to the start of the frame and runs flow ops afterwards. But a clean
mental model is: **one beat per frame**. Mixing structural and flow inside one
frame works but is harder to read on the page.

### What looks good

- **One concept per frame.** "introduce cache" should *only* introduce the
  cache. Don't bolt a flow on the same beat — the viewer is busy parsing the
  shape change.
- **Always name your frames.** `frame "introduce cache" { … }` reads as a
  table of contents at the top of the file. Unnamed frames are anonymous in
  the player's keyframe scrubber.
- **Use `par { }` to make things happen at the same time.** Default sequencing
  is `seq` (one after another). `par` is what gives the animation its
  liveliness — a token landing at a node *while* a bubble pops out beside it
  reads as cause-and-effect, not as two unrelated steps.
- **Let `seq` carry causation.** Inside a flow frame, `client -> api` followed
  by `api -> db` *automatically* chains: the second token leaves the API the
  instant the first arrives. This is what makes "request flows down through
  the stack" feel right; there's no explicit timing, the engine reads the
  topology of the block.

### Rules of thumb (the ones that matter most)

**Keep labels short.** Token, edge, and node labels render at a fixed text
size onto small graph elements. A token chip showing `"GET /user/42 with
session token"` either overflows or squishes the text to unreadable. Aim for:

- **Tokens (the chip on a flowing dot):** verb + small noun. `"GET /user"`,
  `"SELECT"`, `"publish"`. Roughly ≤ 16 chars; never more than 24.
- **Node labels:** noun. `"API"`, `"Cache"`, `"Order Service"`. Two words max.
- **Edge labels:** rare — only when the edge has a *type* the topology
  doesn't already say (`"http"` vs `"async"`). If unsure, omit.

**Use `par { }` for things that happen together.** Default sequencing is `seq`
(one after another). The animation only feels like a *system* if concurrent
things actually appear concurrent — a token landing at a node *while* a bubble
pops out beside it reads as cause-and-effect; the same two ops in series read
as "and then, separately, also...".

```
par {
  api -> cache "HIT"            -- happens at the same time as
  +bubble skip cache "no DB"    -- the bubble appearing
}
```

`par` and `seq` nest. The frame body is implicitly a `seq`, so you only
write `seq { … }` when you need an inner sequential block inside a `par`.

**Use `seq` chaining to model a single request flowing through a stack.** If
the *to* of one token matches the *from* of the next within a `seq` (the
default), the engine renders ONE continuously travelling dot rather than
three discrete tokens. This is what makes "request flows down through the
stack" feel like one motion:

```
frame "request" {
  client -> api "GET"            -- chain: dot leaves client, passes through
  api -> db "SELECT"             -- api, terminates at db — one motion
}
```

Break the chain (use `par`, or have the next leg start from a different
node) when the flow actually fans out or branches.

**Anti-patterns:**

| Anti-pattern | Why it's wrong | Fix |
|---|---|---|
| 8 frames showing one diagram appear piece by piece before any flow | Viewer is lost before the demo starts | One `setup` frame; start the story with the first flow |
| Long flow with no `par` | Reads as an itemised list, not a system | Group concurrent legs in `par` |
| `+bubble` without `-bubble` | Bubbles pile up across frames | Every `+bubble id` needs a matching `-bubble id` later (next frame, or same frame after the chain that motivates it) |
| Structural + flow in one frame | Two concurrent beats, neither reads | Split: structural frame, then flow frame |
| Multiple identical-direction tokens between same pair | Looks like one flickering dot | Use one chained sequence, or `par` if they're truly concurrent |

### When to use bubbles

Bubbles are the diagram's **commentary track**. A bubble appears anchored to a
node; use it when a viewer needs an *explanation* the topology can't carry on
its own:

```
+bubble skip cache "skipped DB!"   -- celebrate a cache hit
+bubble dispatch queue "async"     -- distinguish fire-and-forget from RPC
```

Bubbles are NOT for labels — that's what node/edge labels are. If the
information is intrinsic to the system (e.g. a node's name), use a label.
If it's intrinsic to *this moment in time*, use a bubble.

Bubbles persist across frames until you `-bubble id`. A bubble that lives for
two frames re-anchors itself if the underlying node moves due to a structural
change.

---

## Language

A **frame** is one beat in the animation. Statements inside a frame either
change the graph shape (structural) or move data along it (flow).

### Frames

```
frame setup { +node a "A" +node b "B" +edge a b }
frame "first request" { a -> b "hello" }
```

Names are unquoted identifiers (`setup`, `cache_hit`) or quoted strings
(`"first request"`). Use quotes for spaces.

### Adding and removing nodes

```
+node api "API"        # introduce node
-node api              # remove node
```

### Adding and removing edges

```
+edge api db "writes"  # label is optional
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

### Bubbles (commentary)

```
+bubble skip cache "skipped DB!"
-bubble skip
```

A bubble is anchored to a node and persists across frames until you remove
it. Use bubbles for things the topology can't say on its own — "async",
"cache hit", "retry". Use node/edge labels for everything else.

### Concurrency: `par` and `seq`

```
frame "cache hit" {
  client -> api "GET"
  par {
    api -> cache "HIT"
    +bubble skip cache "skipped DB!"
  }
  -bubble skip
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

frame setup { ... }
```

### Comments

```
# Line comments start with '#' and run to end of line.
+node a "A"     # trailing comments work too
```

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
markgraf [INPUT] [-o out.mp4] [--fps 60] [--scale 2.0] [--play]
```

`INPUT` is optional: omit it to read from stdin (`pbpaste | markgraf --play`),
pass a path to read a file, or pass `-` for explicit stdin.

| Flag | Default | Notes |
|---|---|---|
| `INPUT.markgraf` | required | path to the source file |
| `-o, --output PATH` | `out.mp4` | output video path; container chosen by extension |
| `--fps INT` | `60` | render frame rate |
| `--scale NUMBER` | `2.0` | resolution multiplier; `2.0` is retina-quality |
| `--play` | off | open the native macOS player instead of encoding |

ffmpeg is embedded — no system dependency. CLI is darwin-arm64 only as of
v0.1.0.

---

## Reference: a complete example

```
seed 1

frame setup {
  +node client "Client"
  +node api "API"
  +node db "Database"
  +edge client api
  +edge api db
}

frame "direct read" {
  client -> api "GET /user/42"
  api -> db "SELECT"
}

frame "introduce cache" {
  +node cache "Cache"
  -edge api db
  +edge api cache
}

frame "cache hit" {
  client -> api "GET /user/42"
  par {
    api -> cache "HIT"
    +bubble skip cache "skipped DB!"
  }
  -bubble skip
}

frame "introduce queue" {
  +node queue "Queue"
  +edge api queue
}

frame publish {
  client -> api "POST /order"
  par {
    api -> queue "publish"
    +bubble dispatch queue "async dispatch"
  }
  -bubble dispatch
}
```

Read top to bottom: structural setup → simple flow → structural change →
flow showing the *new* path → another structural change → flow with async
fan-out. Each frame earns its place by adding one idea.

---

## Iteration

You are writing this without seeing the rendered output. To stay grounded:

- Read each `frame { … }` aloud as a sentence in plain English ("the API
  fans out: it queues a publish *and* posts a notice"). If you can't, the
  frame is doing too much — split it.
- Default timing is tuned for a "narrating-while-presenting" cadence
  (`tokenSpeed: 100`, `minTokenDuration: 1.4s`). If a flow has too many
  events, the fix is splitting frames, not speeding up tokens.
- The user runs `markgraf <file> --play` to review — they see the result,
  you don't. So lean on these textual tells: long labels, missing `par`
  for things that happen together, deeply nested blocks, dangling
  bubbles, structural+flow in one frame. Each is a problem you can spot
  from the source alone.

### Tell the user how to preview

Whenever you write or edit a `.markgraf` file, end your reply with the
exact command the user can run to see the result. Don't assume they
remember the flags. Two patterns:

```
# Preview the file you just wrote / edited:
markgraf path/to/foo.markgraf --play

# Or render to mp4 (no window):
markgraf path/to/foo.markgraf -o foo.mp4
```

For quick experiments that don't need a saved file, suggest the stdin
form so the user doesn't have to manage a temp file:

```
# Pipe a heredoc directly into the player:
markgraf --play <<'EOF'
frame setup { +node a "A" +node b "B" +edge a b }
frame greet { a -> b "hello" }
EOF
```

In **fish**, heredocs aren't supported — use a single-quoted multi-line
string instead:

```
echo '
frame setup { +node a "A" +node b "B" +edge a b }
frame greet { a -> b "hello" }
' | markgraf --play
```

The player opens immediately; in it, `space` toggles play/pause, arrow
keys scrub frames, `r` restarts, `q` quits. Drag-and-drop a different
`.markgraf` file onto the window to reload.
