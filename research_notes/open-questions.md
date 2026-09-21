# Open questions — the actual frontier

These are unresolved as of the last recorded session. If you make progress on one, update it here
and cross-reference the new finding in `findings-timeline.md`.

## 0. What are the per-species materials' float values? (the frontier as of 2026-09-20)

The shader itself is known (`shader-source.md`, finding #22). Each of the 96 species materials
(`MegaCandy_<key>.mat`, remote bundle `megacandymaterials_assets_all_*.bundle`) can set its own
`_Stop1..4`, `_GlassColor`, `_GlassTint`, `_GlowIntensity`, `_GlowColorIntensity` and
`_BrightnessCurve`; the PokeMiners JSON holds only the colours. With the generic icon's floats the gold
refs score 34.8 mean RMSE and different species prefer different stops, so these floats are real
per-species data. Getting them (a PokeMiners re-dump including `m_Floats`, or the on-device bundle cache
read with UnityPy) should settle most of what follows.
**Status 2026-09-20 (finding #24):** all 60 referenced species are fitted and stored in `raw/MegaCandyMaterialFloats.json` (estimates, with per-float intervals); `_BrightnessCurve` ~0.15 and `_GlassTint` ~0.1 look like real constants, `_GlowIntensity` ~0.77 / `_Stop4` ~0.84 probably, `_Stop1-3` vary per species, `_GlowColorIntensity` cannot be identified from images. Still open: 7 species (Garchomp, Aerodactyl, Pidgeot, Gallade, Alakazam, Ampharos, Falinks) stay above RMSE 15 - not a search failure. Replacing three blog-sourced references (Blastoise, Steelix, Malamar) with cleaned in-game screenshots took each of them to the noise floor (finding #25), so the likely fix is better references, not a better model; the gold references also want a much lower curve, which fits an older or different image source. Several fits push `_GlowColorIntensity` to its bound (1) and want more (1.03-1.5 when the range is widened), which hints the outline term saturates or something in alpha / the G channel is mis-modelled.  Until then, **fit these nine floats per species
inside the real shader** rather than inventing new model terms — and if a fit lands on the same values
for many species, that is probably the artists' template material.

Questions 1, 3 and 5 below were asked of fitted models that lacked the glass layer and the stop
positions; read them as symptoms of those missing inputs, not as separate mysteries. Question 2 stands,
and becomes directly testable once the floats are known (a species that still misses badly with its true
material was rendered by an older pipeline). Question 4 is moot — it is about a removed model.

## 1. What coordinate explains the left↔right hue variance at equal R?

At the same template-R value, color still differs left vs. right in gold references (finding #4).
Tried and rejected: linear x/y terms added on top of the R-based gradient (no improvement — the
optimizer zeroes them out). The directional glow-light coordinate (`ga`/`gc`/`gw`, finding #17–18)
explains *some* of this for the glow-hue corner specifically, but not the rest of the body. Untried
candidates: the template's edge-normal direction, a rotated UV independent of the glow angle, or a
second texture channel/asset not currently in `raw/` (would need sourcing new reference material to
test).

## 2. Is the gold reference art even the deterministic output of one shader over today's JSON?

Multiple independent findings point toward "probably not, exactly":
- Per-reference exposure needed to match each gold ref varies 0.5–1.5×, with no correlation to
  palette luminance (finding #8).
- The two art "generations" (`reference-tiers.md`) suggest at least some gold refs were rendered from
  an older template or pipeline version.
- The universal per-pixel weight map's fitted colors for gold references sit far from the JSON
  values, while its fitted colors for silver references land close to the JSON (`models-tried.md`).

If true, chasing gold RMSE below ~10-12 per species may be chasing noise rather than a real,
recoverable function. Worth testing directly: fit the universal weight-map approach's colors
per-tier and see if the gold-vs-silver gap is consistent across many more references, or is
particular to the handful of gold refs currently sorted (only ~10 exist).

## 3. The top-right multiplicative glow tint doesn't generalize

Grid sampling and the systematic theory round (findings #10, #19) agree that Ramp3-dominated regions
(typically top-right) need a **multiplicative** boost toward the glow hue that a lerp/mix pipeline
structurally can't produce — but fitting this tint's strength globally collapses it to zero, meaning
whatever triggers it isn't uniform across species (Venusaur/Charizard/Manectric/Gengar show it;
Houndoom doesn't). No per-species trigger condition has been found. This is the same open slot as
question #1 in spirit — a missing per-species or per-region signal — but is worth tracking separately
because the *shape* of the fix (multiply, not lerp) is already known; only *when to apply it* isn't.

## 4. Why does Ramp3 never win a pixel under the universal weight map?

The learned per-pixel weight map (`models-tried.md`) puts real weight on Ramp1, Ramp2, and Ramp4 but
never selects Ramp3 as the dominant basis color anywhere, despite Ramp3 clearly appearing in several
species' standards by eye. Untested hypothesis: Ramp3 might mostly show up as a *secondary* mix
component rather than ever dominating a pixel outright — worth checking the raw per-pixel weight
distributions (not just the argmax) before concluding Ramp3 is redundant with the other three.

## 5. Per-species light direction and per-species pole layout: is there a pattern?

Different species want the glow light from different sides (Venusaur ~111°, Starmie clusters toward
upper-right ~45°; finding #17–18) and, in the earlier free-pole experiments, different species wanted
different spatial pole layouts entirely (finding #16 in `models-tried.md`'s "rejected alternatives").
No one has yet checked whether either of these correlates with something about the species — its
primary type, its base color, its dex-order "art generation" (`reference-tiers.md`), or anything else
sourceable from `raw/names.json` or elsewhere. If a correlate exists, it could replace "fit per
species" with "look up per species", which is the difference between a formula and a lookup table.
