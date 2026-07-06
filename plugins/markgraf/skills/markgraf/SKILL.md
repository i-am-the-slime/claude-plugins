---
name: markgraf
description: Authoring rules and grammar for the markgraf animation language. Use when writing or editing .markgraf files, talking about Markgraf diagrams/animations, sketching system graphs, sequence diagrams, still snapshots, nodes, edges, tokens, dives, or validating/previewing Markgraf with the CLI.
---

# Markgraf authoring

A `.markgraf` file is a short script for an animated system diagram. A bare top-level statement is a beat; Markgraf animates beats in source order. Use `par` when beats happen together and `seq` when a grouped block still needs ordered sub-beats.

## Modes

Animated graph: no header.

```markgraf
+ client: Client
+ api: API
+ db: Database
+ client -> api
+ api -> db
client ~> api: GET /user/42
api ~> db: SELECT
api <~ db: row
client <~ api: 200 OK
```

Still graph: `still`.

```markgraf
still

+ client: Client
+ api: API
+ db: Database
+ client -> api: calls
+ api -> db: reads from
```

Sequence diagram: `diagram sequence`.

```markgraf
diagram sequence

+ client: Client
+ api: API
+ db: Database
client ~> api: request
api ~> db: query
api <~ db: result
client <~ api: response
```

## Statements

- Nodes: `+ api: API`, `+ db "Database"`, `+ cache |Redis\nCache|`, `- api`.
- Graph edges: `+ api -> db`, `+ api -- cache`, `- api -> db`, `- api -- cache`.
- Runtime flow: `api ~> db: query`, `api <~ db: result`.
- Edge repointing: `~ api -> db => api -> cache`.
- Dives: `into api`; add `out` when returning is part of the story.
- Grouping: `par { ... }` for simultaneity, `seq { ... }` for ordered sub-beats.

Interiors describe what exists inside a node:

```markgraf
+ api: API

inside api {
  + handler: Handler
  + db: Database
  + handler -> db
}

into api
handler ~> db: load user
```

## Labels

- Node labels are short nouns: `API`, `Postgres`, `Auth Service`.
- Token labels are hop-local present-tense phrases: `checks cache`, `returns row`.
- Use `: label` for one-line labels.
- Use pipes for multiline labels; Markgraf trims and dedents the block.
- Escape a literal pipe inside a pipe label as `\|`.

```markgraf
api ~> cache |LOOKUP user:42
checks the cache first|
```

## Story rules

- One concept per beat.
- Build structure before sending tokens along it.
- Use graph edges for stable relationships.
- Use `~>` / `<~` for runtime flow.
- Use `par` only for true simultaneity.
- Let each dive stand alone; return only when the return matters.
- Keep names out of the source unless the diagram needs them.
- Run `markgraf --check` whenever the shape changes.

## CLI

```sh
markgraf --check file.markgraf
markgraf file.markgraf --play
markgraf file.markgraf -o out.mp4
markgraf file.markgraf --still '' -o still.svg
markgraf file.markgraf --sequence sequence.png
```
