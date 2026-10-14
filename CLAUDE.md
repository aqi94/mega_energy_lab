# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Mission
**Determine the actual algorithm Niantic uses to tint the Mega Energy candy icon**
(`raw/pokemon_details_mega_candy_256.png`, the game's 256 px grayscale template whose R/G/B channels each play a
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

**The ordinary Candy / Candy XL is the solved contrast case (2026-09-26):** its shader (`NianticCustom/UI/TriColorSplat`,
also read out of the APK — `research_notes/candy-shader.md`) is a plain `R·_ColorR + G·_ColorG + B·_ColorB` mask with
nothing to fit, coloured per family from `raw/candy/PokemonCandyColorData.json` (`_ColorG ← PrimaryColor` body,
`_ColorR ← SecondaryColor` stripes — confirmed against an in-game Nymble Candy XL, RMSE 7.6 vs 99 swapped). The lab's
**Candy** tab renders it; the XL art (220×232) is placed at (14, 19) inside the game's 256×256 icon frame.

**Medals are the third case (2026-09-26), half-solved:** every medal (`Badge_<n>.png`, PokeMiners `Images/Badges/`)
and the four tier rings are RGB templates — G = the tier metal with the shading baked into G's intensity, B = a fixed
per-medal accent, R unused. The APK carries the **inputs** (`Niantic.Holoholo.BadgeMaterialConfig`: MetalColor /
MetalPower per tier, DetailColor per medal — `raw/medals/BadgeMaterialConfig.json`) but **not the shader** (`ma_Badge`,
remote bundle), so the tint is **fitted** to four in-game screenshots (`research_notes/medal-shader.md`,
`raw/medals/MedalTintModel.json`: `mix(MetalColor^(MetalPower·k(G)), white, w(G))` + one light ramp; RMSE ≈ 11–15
against JPEG references). The old pre-painted `_1_01/_2_01/_3_01` tiers were baked from the same templates but with an
older palette — a check, not a standard. **`medal_lab.html`** is the separate tool for it (same house rules: the
game's values are constants, only the fitted constants have sliders).

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
8. **Trust cleaned in-game screenshots most; treat wiki / blog thumbnails with caution.** The old "gold" tier turned out to
   be ImageMagick thumbnails of older game builds (finding #28: Venusaur's differed from its screenshot by RMSE 38 and
   wanted `_BrightnessCurve` 0 instead of 0.17). Check a reference's PNG metadata before weighting it, and do not let
   thumbnails drive constants (`research_notes/reference-tiers.md` predates this). On 2026-10-13 the last three gold
   thumbnails were replaced by screenshots (finding #35): the gold tier is now empty and the thumbnails sit in
   `raw/standards/legacy/`. A thumbnail can fit well on RMSE while pushing floats onto their bounds — check the floats too.
9. **A single shared formula across all species tops out far worse (RMSE ~24–30) than fitting each
   species individually (RMSE ~10–18).** Per-species variation is real signal, not noise — don't
   assume one fixed set of constants should work everywhere. (Now explained: every species has its own
   material, so its stops / glass / glow floats can differ. If you must fit, fit *those nine floats
   inside the real shader* — do not add new model terms.)

**What `mega_lab.html` ships (since 2026-09-20):** one model — the real shader — with every entry
starting at its **estimated** floats from `raw/MegaCandyMaterialFloats.json` (fetched over http, embedded snapshot on
`file://`; regenerate the snapshot with `scripts/embed_floats.py`; layering: generic material < estimate < the user's
slider edits; the header's "Δ vs generic" shows the gain; fits keep neighbouring stops >= 0.1 apart — `MIN_STOP_GAP` in the lab's descent; edits live in localStorage `megaLab.floats2` as differences from the estimate — the pre-estimate key `megaLab.floats` is set aside as `floatsLegacy`, since read as edits on top of the estimate it hid the fitted values). Before that every entry started at the generic material's floats (`MegaCandyDefault`: stops 0 / 0.594 / 0.78 / 0.806, glass tint
0.1, glow 0.56 / 0.5, curve 0.15). Those starting values are *known for the generic icon only*; for a
species they are a guess until its material is dumped. The earlier fitted model (Venusaur fit, RMSE 11.7)
and its colour editors, stage toggles and shader menu were removed: they modelled things the real shader
does not do. **Do not add controls for known quantities** (colours, glass constants, blend maths) — the
user asked for those to be constants. The **Candy** page (2026-09-26) is separate: everything about it is known, so
its only inputs are the two colours (family from the JSON, or custom).

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
- **Candy tab** — the striped Candy, Candy XL and the small icon-style candy (`candy_vector_rgb`, 64×64, shown at 2×)
  rendered together with the real candy shader at native size (Candy / XL in the game's 256×256 icon frame; the XL art
  sits at (14, 19), see `scripts/embed_candy.py`; the small icon is layered as in github.com/aqi94/pogo-candy — silhouette
  (alpha) in the base colour, stripes (R) in the stripe colour, glint (B) white on top, `compositeSmall`): pick a family (dropdown from `PokemonCandyColorData.json`, ←/→ step) or type / pick custom
  base and stripe colours (hex or colour input; `_ColorB` stays the material's white). Reset to family, Swap base ↔
  stripe (the two possible channel mappings), Copy JSON. Remembered in `localStorage megaLab.candy`.
- **Output folder… / Save PNG / Render all…** (top toolbar) — renders the current shader +
  parameters to a real 256 × 256 PNG (the texture's native size), named like the reference images
  (`GO_<Species>_Mega_Energy[_X|_Y|_Z].png`). "Save PNG" does the current entry; "Render all…" does
  all 96. On the Candy tab the same buttons save Bulbapedia-style names: `GO <Species> Candy.png` (the small icon),
  `GO <Species> Candy artwork.png` (the full-size art) and `GO <Species> Candy XL.png` (custom colours:
  `GO Custom <BASE> <STRIPE> …`), or every family into `candy_renders.zip`. In Chrome/Edge, "Output folder…" lets you pick a local folder (e.g. an `outputs/` you
  create) to write directly into; other browsers fall back to normal per-file downloads.

Nothing else typed into the page persists back to this repo — it's a scratchpad for eyeballing (and
now rendering) the model, not an editor that writes source files.

**`medal_lab.html`** (2026-09-26) — the medal tool, the same shape with no build step: pick a medal (←/→) and a
tier (Bronze · Silver · Gold · Platinum, keys 1–4); the **Inputs** card shows the game's values for that pick
(read-only), **What if** lets you substitute a custom MetalColor / MetalPower or accent, **Model** has one slider per
fitted constant (Reset to fitted, Copy JSON); the **Render** card draws the composed medal (ring + white disc + icon),
the icon and the ring at native size, and all four tiers; **Compare** (the four medals with a screenshot) shows the
aligned reference, the difference and the RMSE. **Layout** (user request 2026-09-26: some templates are cropped to the
art, so the default placement can sit off-centre): drag the icon inside the fixed ring with photo-editor guides
(frame centre crosshair, the icon's box and centre, snap to centre — Alt skips it, live Δx / Δy / × readout), wheel or
the Scale slider to scale with the aspect ratio always fixed, arrow keys nudge 1 px (Shift 10) while the render has
focus, a White-disc toggle, "Guides always"; kept per medal (`layouts` in `medalLab.state`), used by the tier strip and
by every export. **The default placement is the measured rule (2026-09-27, `raw/medals/MedalLayouts.json`,
`scripts/measure_layout.py`):** the ring fills the frame, the disc is the tier ring's own inner radius, the icon is the
game's 512-px texture (256 for PokeMiners' 256-px templates) with its full rect centred and side = half the ring's outer
diameter (anchor 0.25; 0.64 for Showcase Star's override) — one constant pixel scale, so a template cropped to its art
(Ultra League) is small and off-centre until the measured deviation or the user corrects it; the two insets
(`raw/medals/insets/`, Rising Star Duo / Mega Evolution Guru) are drawn on top at the same scale, centred, with a toggle.
Save PNG writes `GO_<Title>_Medal_<Tier>[_Icon|_Ring].png`; Render
all… zips every medal × tier into `medal_renders.zip`. Remembered in `localStorage medalLab.state`.
`window.MedalLab` is the automation API (`scripts/lab_driver.mjs --global MedalLab`). Rebuild the embedded data with
`python scripts/embed_medals.py` after anything in `raw/medals/` changes; refit with `python scripts/fit_medals.py`.

Estimating the unknown floats (refit, replace a reference, constants / cohorts, inference for unreferenced entries): use the
`mega-energy-floats` skill (`.claude/skills/`, local) — it drives the lab headless through `window.MegaLab`.

## Architecture
- `mega_lab.html` — the whole tool, one self-contained file (inline JSON + `data:` URIs, no external
  references, works offline). It was originally generated by build tooling that no longer exists, so it
  is now **edited by hand** — the script is one classic `<script>` IIFE after a single ~1.8 MB data line
  (never print that line; search by function name). The shader is `MODELS.megaCandy`; keep its maths
  identical to `research_notes/shader-source.md`.
- `raw/` — the source inputs the page was built from, kept for provenance and reproducibility:
  - `pokemon_details_mega_candy_256.png` — the grayscale RGB-channel template art, the APK's own 256 px texture (see
    "Universal rules" for what each channel means). `pokemon_details_mega_candy.png` is PokeMiners' older 128 px copy (unused).
  - `PokemonMegaCandyAkaMegaEnergy.json` — the game's own per-species target ramp colors, keyed by
    dex number (plus `_MEGA_X`/`_MEGA_Y`/`_MEGA_Z`/`_Mega_Z` suffixes for multi-form Megas).
  - `names.json` — dex number → species name, used only to label entries and to build export
    filenames.
  - `candy/` — the Candy tab's inputs: `candy_rgb.png` (256×256), `LV40_XLCandy_RGB_PSD.png` (220×232) and
    `candy_vector_rgb.png` (64×64, the small icon, extracted from the APK's `sharedassets1.assets` path ID 961), the
    R/G/B-mask templates, `PokemonCandyColorData.json` (549 families' Primary / Secondary colours) and
    `standards/GO_Nymble_Candy_XL.png` (an in-game icon: fixes the XL frame offset and the channel mapping). Embedded as
    `<script id="lab-candy">` by `scripts/embed_candy.py` (run it after any of them changes; idempotent).
  - `standards/{gold,silver,bronze}/` — in-game reference images the model is checked against, sorted
    by trust tier (see `research_notes/reference-tiers.md`).
  - `medals/` — the medal lab's inputs: `templates/Badge_<n>.png` (57 RGB templates + the route medal's mask and
    sprite), `rings/badge_ring_<1-4>.png` (one pure-G ring per tier), `BadgeMaterialConfig.json` (the tier palette and
    per-medal accents read out of the APK), `MedalTintModel.json` (the fitted tint constants + scores), `names.json`
    (enum + title per medal), `MedalLayouts.json` (the measured composition rule + per-medal deviations), `insets/`
    (the two pre-painted overlays) and `standards/` (nine in-game screenshots, the three baked Unown tiers, and
    `aligned/` — the screenshots' icons on the template grid, for the Compare view). Embedded as
    `<script id="lab-medals">` by `scripts/embed_medals.py` (idempotent).
- `research_notes/` — durable, distilled findings from past sessions (start at `README.md`). Add to
  this when you learn something new; don't just leave it in your own session's memory.

## Scope notes
- The shader maths is documented in `research_notes/shader-source.md`; the page's own UI (the "Render
  pipeline at this pixel" breakdown, the Shader components grid, slider tooltips) shows it per pixel.
- **The template is the game's own 256 px texture** (`raw/pokemon_details_mega_candy_256.png`, APK 0.429.1 pathID 462,
  ASTC 6×6; since 2026-10-13 — finding #34). It is embedded as raw RGBA bytes (`<script id="lab-template">`, written by
  `python scripts/embed_template.py`; a PNG would lose the outline's low-alpha texels to the canvas's premultiply). The
  lab shades every texel, averages each 2×2 block premultiplied onto the 128 px grid the references live on (scores,
  fits, panels), and exports at the native 256 px. Probes / feature maps read the 2×2-averaged texel. The old 128 px
  `raw/pokemon_details_mega_candy.png` (PokeMiners' copy, a differently drawn outline) is no longer used. The shader's
  own alpha is `sat(A + G·_GlowColorIntensity)·A`. `scripts/shade_mega_candy.py` does the same (`reduce2`, `template128`).
- 22 of the lab's embedded references are not byte-identical to their files in `raw/standards/` (six larger screenshots,
  sixteen warped onto the template when the lab was first built). The embedded copies are what every fit used; Python's
  cross-check differs on exactly those (by up to ~2 RMSE) and agrees within 0.1 elsewhere.
- To test a page change without the browser extension: serve the folder (`python -m http.server`) and drive
  headless Edge over its debugging port — file URLs and clicks both work that way.
