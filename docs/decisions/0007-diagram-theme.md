# 0007 — Diagram theme is a named key, neutral by default

- Status: Accepted
- Date: 2026-07-12

## Context

Mermaid blocks were first rendered with Mermaid's own `neutral` theme — grey
nodes, black edges — passed to `mmdc` as `-t neutral`. A later change made every
diagram inherit the document's accent instead: node fills, borders, and arrows
were all re-tinted, unconditionally, with no way to turn it off.

That looked cohesive on a document whose diagrams were drawn for it, and wrong
on everything else. A diagram is frequently a *borrowed or shared* artifact — a
block pasted from an RFC, or one that will be lifted back out into a wiki or an
issue. Splashing a house accent across it by default changes an author's drawing
without being asked, and the only escape was to not use imprint.

The accent tint is still worth having; it just isn't a default.

## Decision

1. **A named `diagram_theme` key**, not a boolean. It takes `accent` or one of
   Mermaid's built-ins (`neutral`, `default`, `forest`, `dark`), and follows the
   same precedence as every other setting: **CLI > front matter > config >
   default**. An unrecognized name is a hard error, not a silent fallback.
2. **`neutral` is the default** — byte-identical output to the pre-accent
   behavior, so a diagram renders the way its author meant it to unless someone
   asks otherwise.
3. **`accent` is opt-in** and is the only theme imprint colors itself. It derives
   node fill, node border, and edge color from the document's `accent`, reusing
   the same `lighten(90%)` tint the template gives callouts. Every other palette
   entry stays a fixed neutral: the accent is a highlight, not a repaint.

A boolean (`diagram_accent: true|false`) was the first shape considered and
rejected — it would have exposed the accent tint while leaving Mermaid's other
built-in themes unreachable, and it had no room to grow. A named key costs the
same to implement and hands the user the full set. The boolean never shipped.

## Consequences

- **The theme must travel in `mmdc`'s JSON config, never its `-t` flag.** `-t`
  whitelists only `default`/`forest`/`dark`/`neutral` and rejects `base` — and
  `base` is the one theme that honors `themeVariables`, which is how `accent`
  applies its palette at all. So *every* theme, built-in ones included, is
  written into the config file that `mermaid.initialize()` reads. Passing a theme
  on the command line looks like it should work and would quietly break `accent`.
- **`dark` gets its own canvas.** `mmdc -b` is white for every other theme, but
  Mermaid's `dark` assumes a dark background and strokes its edges in
  `lightgrey`, which all but vanishes on a white page. So `dark` keeps Mermaid's
  `#333` and the figure lands as a dark card inset in the page. This is the one
  place imprint overrides a built-in theme rather than passing it through, and it
  is a legibility fix, not a style preference.
- **`diagram_theme` is consumed by the preprocessor**, unlike every other config
  key, which becomes a pandoc `-V` variable for the template. It steers diagram
  rendering, not layout, so `PREPROCESS_KEYS` holds it back from the generic
  passthrough — otherwise it would leak into custom templates as a stray
  variable.
- The accent tint is computed in Python rather than Typst, because the SVG is
  rendered before Typst ever runs. The `lighten()` math is therefore duplicated;
  it is a handful of lines, and the alternative (teaching the preprocessor to
  call Typst) is far worse.
- **The greys are not duplicated.** Only three things in the palette follow the
  accent — node fill, node border, and edge color. Everything else (`ink`,
  `heading-ink`, `hairline`, `surface`) belongs to the *page*, and a tinted
  diagram has to sit on the same ink and surfaces as the prose around it. Those
  are read out of the bundled template's `#let <token> = rgb("#hex")` block at
  preprocess time, so the template stays the single source of theme truth and a
  changed token can't leave the diagram behind. A hardcoded fallback covers a
  token block reshaped past recognition. A custom template keeps the bundled
  greys either way — imprint cannot know what palette someone else's layout uses.
- Adding a future theme means adding a name to one tuple. Presets beyond
  Mermaid's own (a house dark, a print-safe monochrome) can be layered on without
  disturbing this default.
