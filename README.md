# Mega Energy Color Lab

In Pokémon GO, every Mega Evolution's "Mega Energy" candy starts from the same gray template icon,
but shows up tinted a different color for every species. This project works out how that coloring
works and checks it against real in-game icons.

## Try it

**[Open the lab](https://aqi94.github.io/mega_energy_lab/)** — no install, no download, just click.
It renders each species' Mega Energy next to the real in-game icon and shows how far apart they are.

(Prefer to run it locally? Download [`mega_lab.html`](mega_lab.html) and double-click it — same tool,
works fully offline.)

## Where things stand

**The shader is known.** The game's Mega Energy shader (`NianticCustom/UI/MegaCandy`) was read out of
the game's files, so the lab renders with the game's own maths — nothing about how the color is
computed is guessed any more. [`research_notes/shader-source.md`](research_notes/shader-source.md)
restates it step by step.

**Each species' material settings are estimated.** Every species has its own material with nine numbers
that the shader reads (`_Stop1`–`_Stop4`, `_GlassColor`, `_GlassTint`, `_GlowIntensity`,
`_GlowColorIntensity`, `_BrightnessCurve`). Those live in a downloadable bundle nobody has dumped, and the
public data only has the colors. So the numbers were **fitted** to the in-game reference images:
[`raw/MegaCandyMaterialFloats.json`](raw/MegaCandyMaterialFloats.json) holds an estimate for all 96
entries (62 fitted, 34 guessed), and the lab starts every entry from it. They are estimates, not the game's real values.

| | species | error with the generic material | error with the fitted estimate |
|---|---|---|---|
| gold references | 7 | 35.4 | 12.5 |
| silver references | 47 | 23.4 | 9.5 |
| bronze references | 8 | 29.4 | 9.2 |

Error is RMSE in 0–255 color levels over the pixels both images cover; about 7 is the noise floor. 30
species fit within 10, 26 within 15, and 6 are still above 15: Aerodactyl, Pidgeot, Gallade, Ampharos,
Alakazam and Falinks. Neighbouring stops are kept at least 0.1 apart in every fit (that costs almost
nothing: 0.06 RMSE on average).

**Some numbers look like real constants.** Across the fitted species, `_BrightnessCurve` sits at about
0.15 and `_GlassTint` at about 0.1 almost everywhere (pinning them costs little); `_GlowIntensity`
(about 0.77) probably is too. `_Stop1`–`_Stop3` clearly vary per species, and `_Stop4` is barely
identified (it mostly sits just above `_Stop3`). `_GlowColorIntensity` can't be pinned down from images,
and `_GlassColor` is a per-species switch. The
JSON lists a verdict for each.

**The 34 entries with no reference image** get a guess: the constants above plus stop positions predicted
from the entry's own ramp colors. Tested by leaving each referenced species out in turn, that guess
averages an error of about 20, against 25 for the generic material. Treat it as a starting point.

**The references matter most.** Five of them (Blastoise, Steelix, Malamar, Staraptor, Garchomp) were replaced with
cleaned-up in-game screenshots — the earlier images did not match the in-game render — and all five then
fit at 4.5–6, right at the noise floor. The remaining poor fits are the first
candidates for the same treatment. The seven gold references also disagree with the rest of the set
(their fitted `_BrightnessCurve` is around 0.04 instead of 0.16), which is what an older or different
image source would look like.

## Using the lab

- **Entry / with reference:** pick a species; the checkbox limits the list to entries that have a
  reference image.
- **Material floats:** one slider per number, kept per entry and remembered in your browser. *Fitted
  estimate* re-reads the JSON and gives the entry its estimate; *Generic material*, *Even stops* and
  *Shader defaults* are the other starting points; *Apply to all*, *Reset*, *Undo* and *Copy* do what
  they say. Click a number to type a value instead (it must be a number within the slider's range, and stops
  must stay in order); double-click a slider to put that one value back to the entry's estimate.
- **Compare:** in-game, render, difference, wipe and flicker views, plus the error for this entry, the
  gold references and all references. Hover to probe a pixel, click to pin it, Shift-drag for a transect.
- **Shader components / Explore:** each stage of the shader drawn on its own, and colour-space, transect
  and feature-map tools. The *Fitting* tab runs the same descent that produced the JSON, keeping stops at least 0.1 apart.
- **Save PNG / Render all:** save the current render as a PNG, or all 96 in a single `.zip`.

## What's in the repo

- [`mega_lab.html`](mega_lab.html) — the whole lab in one file (works offline).
- [`raw/`](raw) — the source data: `pokemon_details_mega_candy.png` (the gray template),
  `PokemonMegaCandyAkaMegaEnergy.json` (each species' material colors, dumped by PokeMiners),
  `MegaCandyMaterialFloats.json` (the estimates), and `standards/{gold,silver,bronze}/` (the in-game
  reference images, by how much they are trusted).
- [`research_notes/`](research_notes) — what was tried, what failed, and what is still open.

## How the references are made

A screenshot is cut out of the dimmed screen, lined up with the template and warped slightly to
undo the candy's wiggle animation (the emblem edges are the guide), then shrunk to 128 × 128 with the
template's own outline. Those steps were done with local scripts that aren't part of this repo.
