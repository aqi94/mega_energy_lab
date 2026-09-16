# Mega Energy Color Lab

**Goal:** figure out the actual algorithm Niantic uses to tint the Mega Energy candy icon in
Pokémon GO — a single grayscale template whose R/G/B channels each drive a different piece of the
shading — into the distinct per-species color seen in-game, and do it well enough to **generate a
correct icon for a species that doesn't have official Mega Energy art yet** (i.e. an unreleased
Mega Evolution). Not "reproduce the handful of icons we already have" — that's just how the guess
gets checked.

## Why this is a lab, not a formula

No shader source for this effect is public. Niantic ships a grayscale template
(`raw/pokemon_details_mega_candy.png`) and, separately, a mined table of per-species colors
(`raw/PokemonMegaCandyAkaMegaEnergy.json` — `_Color`, `_GlowColor`, `_EmissionColor`, four
`_RampColor`s), but nothing that says how those colors combine with the template to produce the
finished art. The only ground truth is a handful of real reference icons
(`raw/standards/{gold,silver,bronze}/`, sorted by how much each can be trusted). So the only way to
find the algorithm is to guess a formula, render it, and hold it up against those references by
eye and by error — repeatedly, since a formula that nails one species routinely falls apart on the
next. `mega_lab.html` is built around that loop: it's a visual fitting bench (color-space views,
transects, per-pixel feature maps, side-by-side diffs against every reference, and a coordinate-
descent auto-fitter with a slider per knob), not a static answer, because a static answer isn't
what this problem currently has.

That also shapes what "done" looks like here. The model shipped in `mega_lab.html` today is fitted
to a *single* species (Venusaur) and is explicitly a starting point, not a validated general rule —
see `research_notes/models-tried.md` for why a shared, one-size-fits-all fit still scores far worse
than per-species fits, and `research_notes/open-questions.md` for what's still unexplained (e.g. why
some species want their highlight color applied as a multiply and others don't). Progress here
looks like ruling out wrong models and narrowing what's still possible, not declaring victory.

## Why one HTML file

`mega_lab.html` is a single self-contained page — the template image, every species' color data,
and the in-game reference images it checks against are all baked in as inline `data:` URIs, so it
has zero external references and works fully offline, just by opening the file. That's a deliberate
constraint, not an accident: it means anyone can pick this problem back up — poke at the model,
run a new fit, render a hypothesis — with nothing to install and no build step, years from now, on
any machine. The page shell is generated elsewhere and republished here as this one file; nothing
in this repo needs to be built to use it.

## What's in this repo

- **`mega_lab.html`** — the tool itself. Open it directly in a browser. The **Lab** tab is where the
  fitting work happens (color space / transect / feature-map views, per-entry and gold-weighted
  auto-fit, undo); the **Colour table** tab lists every species' fitted colors; the render toolbar
  exports the current model's output as real PNGs, named like the references.
- **`raw/`** — the source inputs: the grayscale template, the mined per-species color JSON, dex-number
  → name mapping, and the trust-tiered reference images everything is checked against.
- **`research_notes/`** — the durable, distilled conclusions from past sessions of this work: what
  the template's channels mean, which models were tried and rejected (and why), and what's still
  genuinely unresolved. Start at `research_notes/README.md`. This directory is the *record*, not the
  *process* — the many one-off fitting scripts and throwaway experiments that produced these
  conclusions aren't kept here; only what was learned from them is.

## Status

Far from solved. The current model gets close on some species and visibly wrong on others, and
there's open evidence the reference art itself may not all come from one consistent pipeline (see
`research_notes/open-questions.md`, question 2). Treat anything rendered here as a working
hypothesis to check by eye against `raw/standards/`, not a source of truth.
