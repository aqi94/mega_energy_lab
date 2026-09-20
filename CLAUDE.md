# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Mission
**Determine the actual algorithm Niantic uses to tint the Mega Energy candy icon**
(`raw/pokemon_details_mega_candy.png`, a grayscale template whose R/G/B channels each play a
different structural role) into the per-species icon seen in-game, using
`raw/PokemonMegaCandyAkaMegaEnergy.json` (color data mined from the game by PokeMiners) as the only
other input. Until 2026-09-20 everything here was inferred by fitting rendered output against real
reference images (`raw/standards/`).

**The shader itself is now known** — it was read out of the game's APK (`NianticCustom/UI/MegaCandy`;
see `research_notes/shader-source.md`, which overrides any fitted note it contradicts) and reproduces
the generic Mega Energy icon with nothing fitted. **What is still missing is data, not maths:** each
species' material sets its own float values (`_Stop1..4`, `_GlassColor`, `_GlassTint`, `_GlowIntensity`,
`_GlowColorIntensity`, `_BrightnessCurve`), those materials live in a remote bundle, and the PokeMiners
JSON dumped only their colours. `mega_lab.html` now runs this shader: everything known (the maths, its
constants, the JSON colours) is fixed and read-only, and the only controls are those nine unknown floats,
kept **per entry**. **Estimated values now ship** in `raw/MegaCandyMaterialFloats.json` (2026-09-20, finding #24:
fitted per species against the references, or inferred where there is no reference) — estimates, not the game's numbers.

**The fit is currently far from perfect and inconsistent from species to species** — no single
formula tried so far reproduces the reference art exactly, and some species fit far better than
others under the same model (see `research_notes/`). The end goal is not to reproduce a handful of
known icons, but to find a transformation trustworthy enough to **generate correct icons for
species that don't have official Mega Energy art yet.** Keep this mission in view even when a
session's task is narrow (e.g. a UI tweak to `mega_lab.html`) — it's why the tool exists.

**Before doing new discovery work, read `research_notes/README.md` first.** A lot of trial-and-error
has already happened; the goal is to build on it, not repeat it.

## Universal rules (high confidence — treat as settled unless you find contradicting evidence)
Rules 1–7 were revised on 2026-09-20 against the recovered shader (`research_notes/shader-source.md`);
the struck-through claims came from image fitting and are wrong. Rules 8–9 are still fitting-era
judgement (see `research_notes/findings-timeline.md` for the evidence behind each):

1. **Template R drives ramp position — exactly.** The ramp is a plain 4-stop gradient over R with each
   segment clamped; the stop positions are material floats `_Stop1..4` (generic icon: 0 · 0.594 · 0.78 ·
   0.806; shader default 0 · 0.3 · 0.6 · 1). There is no x/y term in the ramp.
2. **Template channel roles are fixed:** R = ramp position; B (helix emblem + specular streak) lerps
   toward **white** by `_GlowIntensity` (~~toward `_Color`~~); G (thin outline) lerps toward `_GlowColor`
   by `_GlowColorIntensity` and also adds to alpha. `_Color` multiplies the whole icon.
3. **`_EmissionColor` is never read** — it is not a property of the shader (~~the dark end of a shading
   mix~~). The dark body colour is `_RampColor2` itself.
4. **Position-dependent hue comes from the "glass" layer**, a fixed per-RGB-channel UV gradient (red
   rises left→right, green bottom→top, blue falls off from the bottom-left corner) blended into the ramp
   colour by `_GlassTint` (~~`_GlowColor` as a directional light whose direction varies per species~~).
   `_GlassColor` = 1 rotates it; that is the only direction switch.
5. **Colors composite in sRGB, not linear light** — confirmed: the project's colour space is Gamma.
6. **Tone = the glass blend + an additive mid-tone parabola** `_BrightnessCurve · (1 − (2c − 1)²)`
   (~~a multiplicative R-driven gain~~). There is no gain or exposure term.
7. **The shader ships in the APK and has been read** (~~no source exists~~). It uses no lookup texture.
   The decompiled listing is Niantic's code: keep it out of this public repo — notes restate the maths.
   The per-species *material floats* are the part still unobtained (remote bundle).
8. **Fit against gold-tier references, and weight the loss toward gold** (see
   `research_notes/reference-tiers.md`) — unweighted error across all reference tiers biases toward
   gray/washed-out results, because lower tiers outnumber and are less reliable than gold.
9. **A single shared formula across all species tops out far worse (RMSE ~24–30) than fitting each
   species individually (RMSE ~10–18).** Per-species variation is real signal, not noise — don't
   assume one fixed set of constants should work everywhere. (Now explained: every species has its own
   material, so its stops / glass / glow floats can differ. If you must fit, fit *those nine floats
   inside the real shader* — do not add new model terms.)

**What `mega_lab.html` ships (since 2026-09-20):** one model — the real shader — with every entry
starting at its **estimated** floats from `raw/MegaCandyMaterialFloats.json` (fetched over http, embedded snapshot on
`file://`; regenerate the snapshot with `scripts/embed_floats.py`; layering: generic material < estimate < the user's
slider edits; the header's "Δ vs generic" shows the gain; edits live in localStorage `megaLab.floats2` as differences from the estimate — the pre-estimate key `megaLab.floats` is set aside as `floatsLegacy`, since read as edits on top of the estimate it hid the fitted values). Before that every entry started at the generic material's floats (`MegaCandyDefault`: stops 0 / 0.594 / 0.78 / 0.806, glass tint
0.1, glow 0.56 / 0.5, curve 0.15). Those starting values are *known for the generic icon only*; for a
species they are a guess until its material is dumped. The earlier fitted model (Venusaur fit, RMSE 11.7)
and its colour editors, stage toggles and shader menu were removed: they modelled things the real shader
does not do. **Do not add controls for known quantities** (colours, glass constants, blend maths) — the
user asked for those to be constants.

## Commands
There is no build step. Open `mega_lab.html` directly in a browser — double-click it, or
`file://` it — and use it. All "development" on the model itself happens live on the page:
- **Lab tab → Color space / Transect / Feature map** — visualize the recolor model against the
  embedded references.
- **Lab tab → Unknown · material floats** (left column) — nine sliders named after the Unity properties,
  for the *current entry only*; each entry keeps its own values (remembered in `localStorage` under
  `megaLab.floats`). Presets: Generic material / Even stops / Shader defaults. Apply to all entries,
  Reset this / all, Undo, Copy this entry / Copy all changed (JSON keyed like the colour file).
- **Lab tab → Fitting** — coordinate descent over the checked floats: "This entry" stores the result in
  that entry; the shared targets fit one set and apply it to every entry; "Fit each reference separately"
  + "Apply each fit to its entry" fits all references from their own values. Undo restores.
- **Colour table tab** — every species' material colours (from the JSON) in one table.
- **Output folder… / Save PNG / Render all…** (top toolbar) — renders the current shader +
  parameters to a real PNG, named like the reference images
  (`GO_<Species>_Mega_Energy[_X|_Y|_Z].png`). "Save PNG" does the current entry; "Render all…" does
  all 96. In Chrome/Edge, "Output folder…" lets you pick a local folder (e.g. an `outputs/` you
  create) to write directly into; other browsers fall back to normal per-file downloads.

Nothing else typed into the page persists back to this repo — it's a scratchpad for eyeballing (and
now rendering) the model, not an editor that writes source files.

## Architecture
- `mega_lab.html` — the whole tool, one self-contained file (inline JSON + `data:` URIs, no external
  references, works offline). It was originally generated by build tooling that no longer exists, so it
  is now **edited by hand** — the script is one classic `<script>` IIFE after a single ~1.8 MB data line
  (never print that line; search by function name). The shader is `MODELS.megaCandy`; keep its maths
  identical to `research_notes/shader-source.md`.
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
- The shader maths is documented in `research_notes/shader-source.md`; the page's own UI (the "Render
  pipeline at this pixel" breakdown, the Shader components grid, slider tooltips) shows it per pixel.
- The lab renders at the repo's 128 px template and uses the shader's own alpha
  (`sat(A + G·_GlowColorIntensity)·A`). The APK's template is 256 px; it is not embedded.
- To test a page change without the browser extension: serve the folder (`python -m http.server`) and drive
  headless Edge over its debugging port — file URLs and clicks both work that way.
