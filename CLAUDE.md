# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Mission
**Determine the actual algorithm Niantic uses to tint the Mega Energy candy icon**
(`raw/pokemon_details_mega_candy.png`, a grayscale template whose R/G/B channels each play a
different structural role) into the per-species icon seen in-game, using
`raw/PokemonMegaCandyAkaMegaEnergy.json` (color data mined from the game by PokeMiners) as the only
other input. No shader source is public — everything here is inferred by fitting rendered output
against real reference images (`raw/standards/`).

**The fit is currently far from perfect and inconsistent from species to species** — no single
formula tried so far reproduces the reference art exactly, and some species fit far better than
others under the same model (see `research_notes/`). The end goal is not to reproduce a handful of
known icons, but to find a transformation trustworthy enough to **generate correct icons for
species that don't have official Mega Energy art yet.** Keep this mission in view even when a
session's task is narrow (e.g. a UI tweak to `mega_lab.html`) — it's why the tool exists.

**Before doing new discovery work, read `research_notes/README.md` first.** A lot of trial-and-error
has already happened; the goal is to build on it, not repeat it.

## Universal rules (high confidence — treat as settled unless you find contradicting evidence)
These held up across many independent findings (see `research_notes/findings-timeline.md` for the
evidence behind each). Model-specific parameters and layouts are *not* settled and change often —
these are the load-bearing facts underneath all of them:

1. **Template R drives ramp position**, dominantly. Recovered ramp position regresses on R with
   r² = 0.93, confirmed multiple independent ways. Reach for R first; only add spatial (x/y) or other
   geometry terms when there's specific evidence R alone can't explain something.
2. **Template channel roles are fixed:** R = shading/ramp position; B (helix emblem + specular
   streak) tints toward `_Color`; G (thin outline) tints toward `_GlowColor`.
3. **`_EmissionColor` (always black in the data) is the dark end of the shading mix**
   (`mix(_EmissionColor, ramp, b·R)`), not a background canvas behind the alpha.
4. **`_GlowColor` also acts as a positional light blended into the ramp itself** from one side of
   the icon (direction varies per species) — not an additive/screen glow overlay.
5. **Colors composite in sRGB, not linear light.** Linear mixing, overlay, and soft-light blends all
   push results toward gray.
6. **Tone must be multiplicative (a stop's chromaticity scaled by an R-driven gain), not additive
   white-lift.** Additive lift is what caused the early "washed out" look everyone rejected.
7. **No public source for the real shader exists** — it and its ramp lookup textures live in an
   unpublished asset bundle. Nothing here is derived from source; it's all fit-to-reference-images.
8. **Fit against gold-tier references, and weight the loss toward gold** (see
   `research_notes/reference-tiers.md`) — unweighted error across all reference tiers biases toward
   gray/washed-out results, because lower tiers outnumber and are less reliable than gold.
9. **A single shared formula across all species tops out far worse (RMSE ~24–30) than fitting each
   species individually (RMSE ~10–18).** Per-species variation is real signal, not noise — don't
   assume one fixed set of constants should work everywhere.

**Current default caveat:** `mega_lab.html` currently ships only one model, with knobs fitted to the
Venusaur standard (RMSE 11.7) — every other attempt tried so far (the gold-wide shared fit, the
universal per-pixel weight map) was judged a failed experiment by eye and removed. That means the
shipped default is a **single-species fit kept as a starting point**, not a validated general
solution — see rule #9. Don't extend or promote it to other species without checking by eye first.

## Commands
There is no build step. Open `mega_lab.html` directly in a browser — double-click it, or
`file://` it — and use it. All "development" on the model itself happens live on the page:
- **Lab tab → Color space / Transect / Feature map** — visualize the recolor model against the
  embedded references.
- **Lab tab → Fitting** — adjust ramp-color sliders per entry (or run coordinate-descent auto-fit
  against gold/silver/bronze references) and see the result immediately; Undo restores.
- **Colour table tab** — every species' fitted colors in one sortable table.
- **Output folder… / Save PNG / Render all…** (top toolbar) — renders the current shader +
  parameters to a real PNG, named like the reference images
  (`GO_<Species>_Mega_Energy[_X|_Y|_Z].png`). "Save PNG" does the current entry; "Render all…" does
  all 96. In Chrome/Edge, "Output folder…" lets you pick a local folder (e.g. an `outputs/` you
  create) to write directly into; other browsers fall back to normal per-file downloads.

Nothing else typed into the page persists back to this repo — it's a scratchpad for eyeballing (and
now rendering) the model, not an editor that writes source files.

## Architecture
- `mega_lab.html` — the whole tool. Generated (not hand-edited): the page shell's JS/CSS is authored
  elsewhere and the data below is baked in as inline JSON/`data:` URIs at build time, so this one
  file has zero external references and works fully offline.
- `raw/` — the source inputs the page was built from, kept for provenance and reproducibility:
  - `pokemon_details_mega_candy.png` — the grayscale RGB-channel template art (see "Universal rules"
    for what each channel means).
  - `PokemonMegaCandyAkaMegaEnergy.json` — the game's own per-species target ramp colors, keyed by
    dex number (plus `_MEGA_X`/`_MEGA_Y`/`_MEGA_Z`/`_Mega_Z` suffixes for multi-form Megas).
  - `names.json` — dex number → species name, used only to label entries and to build export
    filenames.
  - `standards/{gold,silver,bronze}/` — in-game reference images the model is checked against, sorted
    by trust tier (see `research_notes/reference-tiers.md`).
- `research_notes/` — durable, distilled findings from past sessions (start at `README.md`). Add to
  this when you learn something new; don't just leave it in your own session's memory.

## Scope notes
- The recolor model itself (ramp compositing, universal weight map, fitting math) is not documented
  field-by-field here because it isn't source in this repo — it's baked into `mega_lab.html`'s
  embedded JS. Read that file directly if you need the implementation (it's a single classic
  `<script>`, wrapped in one IIFE), or treat the page's own UI (hover states, the "Render pipeline
  at this pixel" breakdown, the Fitting tab's notes) as the documentation.
- If `mega_lab.html` looks stale relative to `raw/` or `research_notes/`, that means the private
  monorepo's copy moved on and this repo hasn't been re-published yet — regenerating it requires the
  build tooling that lives there, not anything in this repo.
