# Findings timeline

Dated, numbered discoveries with the evidence behind them. This is the detail layer behind
`inputs.md`, `reference-tiers.md`, and `models-tried.md` — skim for a topic rather than reading
straight through unless you're doing a deep dive on one issue.

1. **Ramp position is mostly template R.** Hue-based ramp-position recovery (fit a free per-pixel
   gain, then see what position that implies) gives a position map that regresses on R with r² 0.93.
   Spatial layout by eye: Ramp1 bottom-right rim, Ramp2 bottom-left, Ramp3 middle, Ramp4 top-left/
   top-right highlight.
2. **A 1-D ramp explains any single reference well** (per-reference residual 2–17), but contrast and
   lift vary per Mega (gain 0.4–1.5, lift 0–0.55; gain correlates −0.56 with palette mean luminance).
   Gold refs specifically want almost no white lift.
3. **Gold-weighting the loss function** dropped the global white-lift term from 0.16 to 0.05 and gold
   error from 28.8 → 15.6 — the main fix for the "washed out" look. See `reference-tiers.md`.
4. **At equal template R, color still varies left↔right** in gold refs (e.g. Gengar: right side red,
   lower-left saturated indigo; Venusaur/Malamar: Ramp4 green toward top-right). Linear x/y terms on
   top of the R-based gradient did not help — the true coordinate behind this is still unknown
   (see `open-questions.md`).
5. Colors are interpolated in **sRGB, not linear light** — linear mixing, overlay, and soft-light
   blends all push results toward gray.
6. **A universal per-pixel weight map** (shared across all species) reaches gold RMSE 8.9 in-fit /
   9.9 leave-one-out — see `models-tried.md` for the full writeup and its open problems.
7. **Tone is multiplicative, not additive.** In every gold reference, the exact chromaticity of the
   dark stop (Ramp2) appears at 1.1–1.7× its raw brightness (e.g. Gengar: in-game 104,89,175 ≈ Ramp2
   71,61,119 ÷ max × 0.69). An additive white lift can never reproduce that; dividing stops by
   (roughly) their max channel and multiplying by an R-driven gain can. This dropped gold error from
   29.6 → 25.5 and visibly de-washed the dark stops.
8. **No per-pixel function of the palette generalizes across gold refs**, even with very flexible
   fitting: free per-pixel linear weights over {Ramp1–4, glow, white} reach only 21.7 train / 30
   leave-one-out; adding product terms reaches 17.4 train / 40 leave-one-out (badly overfit); free
   bilinear position+gain per pixel plus per-reference exposure correction: 19–20; none of these
   beat the simpler two-gradient/ramp layout once you check generalization, not just training error.
   The unexplained residual sits in the **body** of the icon, not just the rim or emblem. Letting a
   reference's whole palette float freely under the ramp model still leaves 11–14 error and produces
   nonsense palettes. **Working conclusion: exact gold reproduction is not attainable from this data
   with a single deterministic function — aim for improvement, not exactness.**
9. **Two art "generations" among gold refs** — see `reference-tiers.md`.
10. Grid-point color sampling shows in-game colors are often **more saturated than any single ramp
    stop** (e.g. Venusaur top-right 106,228,192 vs. Ramp3 141,225,185 ≈ Ramp3 × a lerp toward the
    glow hue at strength 0.4) — evidence for a *multiplicative* glow tint specifically in
    Ramp3-dominated regions, distinct from the directional glow-light finding (#20 below). Fitting
    this tint globally collapsed to zero — it appears to be local to the top-right, not global.
11. Coordinate-descent optimizers reliably fail to move the gain-slope parameter off its starting
    value in every model tried — treat any reported value for a "gain slope" knob as hand-set/
    unverified unless refit with a different optimizer.
12. **`_GlowColor` under-expression.** An early shared-ramp model was judged "slightly worse… the
    glow color is very under-expressed" by eye despite similar RMSE — a reminder that RMSE alone
    misses whether a model captures the right *structure*, not just the right average color.
13. **Channel roles confirmed** (2026-09-14): B (helix + streak) → tints toward `_Color`; G (outline)
    → tints toward `_GlowColor`. Documented fully in `inputs.md`.
14. **Simplest model preferred by the user** (2026-09-14): plain quadrant color poles, no
    normalization/tint/rim-band/cut/desaturation/glow/lift — deliberately unfitted (gold RMSE 38.9),
    evaluated and adjusted by eye rather than by optimizer. This is a working-style note as much as a
    model note: don't add a transform without specific evidence it's needed.
15. **Data provenance investigation** (2026-09-14): the JSON is very likely a dump of a Unity
    Material's saved properties, with `_EmissionColor` black and `_Color` white on nearly every entry
    matching Standard-shader defaults (i.e., probably stale leftovers, not necessarily direct shader
    inputs in the way their names imply). The actual shader and its ramp lookup textures live in an
    unpublished asset bundle — not obtainable from public data. See `inputs.md`.
16. **Venusaur deep-dive** (2026-09-14, "get the Venusaur standard near-perfect first"): single-
    reference fits across several model families all land at RMSE 11.7–12.5 and look visually
    identical (a uniform pale mint) — RMSE alone is blind to missing structure. Per-pixel analysis
    showed: the lower-left is genuinely a ~50/50 mix of Ramp1/Ramp2 at reduced brightness where every
    model puts pure Ramp2 near full brightness; the upper-right needs a *multiplicative* saturation
    boost no scalar lerp of the stops can reach; the bottom-right rim is Ramp1 mixed with white. A
    single linear gain can't fix the lower-left without also darkening the mid-body.
17. **`_GlowColor` is a directional light, not a screen glow** (2026-09-15, discovered via Starmie,
    a silver reference with clearly separable hues): the glow hue occupies the *entire* top-right
    quadrant at every value of R — no fixed template channel predicts this, but a rotated diagonal
    position coordinate does (sharp transition around one threshold of that coordinate). The
    corner's in-game color ≈ the glow color darkened by the normal shading, meaning this lerp happens
    *before* shading, not as a post-process screen blend. This became the shipped "glow light" step
    in `models-tried.md`.
18. **Per-species light direction** (2026-09-15, Venusaur): the same directional-light idea needed a
    configurable angle, not a fixed upper-right default, because Venusaur's light comes from the
    upper-left/top while Starmie's comes from the upper-right. Best Venusaur-only fit: RMSE 12.6.
    Residual analysis after this fit found three regions the model's knobs structurally cannot reach
    at once (top-right needs more saturation than any lerp gives; left-middle needs to be bluer than
    the ramp allows; the rim needs less blue than pure Ramp1) — treat ~12.6 as this model family's
    accuracy floor for Venusaur specifically.
19. **Systematic theory round** (2026-09-15, 8 parallel agents testing representability ceilings):
    the raw 4-color ramp alone represents only ~3% of Venusaur's body pixels within reasonable error;
    adding the glow lerp and shade-to-black steps raises that to 61% within a small error band; adding
    the wash-toward-`_Color` step before shading nearly doubles species-level accuracy (e.g. Manectric
    23.6 → 9.4 RMSE) and was the biggest single win of this round. The remaining errors are
    overwhelmingly **hue** misses (93% of bad pixels), concentrated at the two ends of the ramp: the
    rim (standard is warmer/less blue than pure Ramp1) and the top-right (needs Ramp3 multiplied by a
    partial lerp toward glow — a multiply the additive/lerp pipeline can't reach). Confirmed
    separately: the JSON palette itself is not the problem (letting it float freely only improves fit
    from 12.9 → 10.5); R alone explains almost all of the spatial geometry (≤2% left on the table);
    and R-to-position is actually two-regime — a flat plateau below roughly R ≈ 0.56, then a jump,
    then a shallower climb (this became the "rim plateau" knob `rs` in the shipped model).
20. **Wash + rim-plateau + interpolation-space additions shipped** (2026-09-15): adding the
    wash-toward-`_Color` step and the rim plateau to the shared model brought the joint gold-wide fit
    from 30.2 to 24.8 RMSE and Venusaur's own fit to 11.7. Interpolating the ramp in OKLab
    instead of sRGB scored identically on Venusaur (11.7 either way); HSV interpolation was worse
    (14.1) and dropped. This is the "current best shared model" described in `models-tried.md`.
21. **Generic and universal-weight-map models removed from the lab; Venusaur fit kept as the only
    default** (user judgment call, by eye): of everything tried, only the Venusaur-specific fit
    (RMSE 11.7) looked anywhere close to correct — the gold-wide shared defaults and the universal
    per-pixel weight map (finding #6) were both judged failed experiments and removed from
    `mega_lab.html`'s Shader menu. Their write-ups stay in `models-tried.md` as history, not as
    active options. This is a judgment call about what's worth keeping in front of a user, not a new
    numerical result — treat it as reinforcing finding #8/universal rule #9 (no shared formula found
    yet generalizes), not as evidence the Venusaur fit itself generalizes to other species.
22. **The real shader recovered from the APK** (2026-09-20; supersedes #15's "not obtainable"). Pokémon GO
    0.429.1 ships `NianticCustom/UI/MegaCandy` in `sharedassets1.assets`, compiled to GLES3 = readable
    GLSL. Full write-up in `shader-source.md`. Headlines: a plain 4-stop ramp over R with **stop
    positions as material floats** (`_Stop1..4`); a per-RGB-channel UV "glass" gradient blended in by
    `_GlassTint`; B → white by `_GlowIntensity`; G → `_GlowColor` by `_GlowColorIntensity`; an additive
    mid-tone `_BrightnessCurve`; `_Color` is a whole-icon multiply; `_EmissionColor` is never read; the
    project is Gamma colour space. Verified with nothing fitted on the generic icon (`MegaCandyDefault`
    material vs `silver/GO_Mega_Energy.png`: RMSE 9.3, same icon by eye). Overturns #13's B → `_Color`,
    the `_EmissionColor` shading mix, and #17–18's directional glow light; explains #19's plateau below
    R ≈ 0.56 (`_Stop2 = 0.594`). **Not yet solved:** the 96 per-species materials live in a remote bundle
    (`megacandymaterials_assets_all_*.bundle`) and PokeMiners dumped only their colours — with the default
    material's floats the gold refs score 34.8 mean RMSE and no one float set fits every species, so each
    species material evidently has its own stops / glass / glow values.
23. **Lab rebuilt on the real shader** (2026-09-20, user: "regarding those unknown float values, modify the
    mega_lab such that I can interact and tinker with it; as for the knowns, make them constant and remove
    options to change them"). `mega_lab.html` now has one model, the recovered shader; the fitted model,
    colour editors + eyedropper, stage toggles and shader menu are gone; the nine unknown floats are sliders
    **per entry**, remembered in the browser, with presets, apply-to-all, undo, JSON copy, and a Fitting tab
    limited to those floats. The lab's numbers agree with an independent script (Venusaur 30.1 both ways).
    First result from fitting inside the real shader: **Altaria 34.3 → 11.3** by moving five floats
    (`_Stop2` 0.53, `_Stop3` 0.61, `_Stop4` 0.84, `_GlassTint` 0.07, `_GlowIntensity` 1.0 — the last one
    pinned at the slider's ceiling) — as good as the old model's best single-species fit, with no invented
    terms. Treat fitted floats as estimates of the true material values, to be replaced when a dump exists.
24. **The nine floats fitted for all 60 referenced species; which are constants** (numbers as of the first fit; see #25 for the updated ones) (2026-09-20, user: "fan out up to 5
    Sonnet sub-agents to help determine the Unknown Material Floats ... determine if any of these floats happen to be
    *almost* the same across all species"). Five agents fitted 12 species each through `window.MegaLab` (new headless
    API, driven by `scripts/lab_driver.mjs`; every number is the lab's own shader): 3 preset + 8 random starts x
    `_GlassColor` 0/1 per species, stops kept ordered, then profile intervals and pin tests; the poor fits got 40-64
    extra starts and did not move (<= 0.05 RMSE), so their misfit is model/reference, not search. Result in
    `raw/MegaCandyMaterialFloats.json` (the lab loads it; embedded copy for `file://`). Mean RMSE generic -> fitted:
    gold 34.8 -> 12.7, silver 24.5 -> 10.3, bronze 30.7 -> 11.0 (24 good <= 10, 28 fair, 8 poor > 15; noise floor ~7).
    **Constants, by lock test (pin the float, re-optimise the rest; median cost over the 52 fits <= 15 RMSE):**
    `_BrightnessCurve` **0.15 - near-constant** (median cost 0.17, silver/bronze fits 0.13-0.18); `_GlassTint`
    **~0.1 - near-constant** (adds 0.5 more); `_GlowIntensity` **~0.77 probable** (not the generic 0.56; loose);
    `_Stop4` **~0.84 probable, weak**; `_Stop1` **not constant** (0 or 0.3-0.5); `_Stop2` (0.42-0.65, tightly
    identified) and `_Stop3` (0.60-0.94) **vary per species**; `_GlowColorIntensity` **unidentifiable** (flat 0-1);
    `_GlassColor` **per-species** (mode 0 wins ~25, mode 1 ~12, rest tie because tint fits ~0). Pinning all the
    look constants at once costs a median 2.3 RMSE, so they are constant-ish, not exactly constant. **The 8 gold
    references split from the rest:** `_BrightnessCurve` median 0.04 (gold) vs 0.16 (silver/bronze) and they fit worst -
    supports open question #2 (older pipeline) but cannot prove it. Stop3 correlates with the Ramp3-Ramp2 lightness gap
    (r = 0.79), so the 36 species with no reference get the constants + a colour regression for Stop1-3: leave-one-out
    mean RMSE **20.7** vs 26.8 for the generic material (beats generic for 49 of 60) - a guess, not a fit.
25. **Blog images replaced by cleaned in-game screenshots** (2026-09-20, user: Blastoise and Steelix looked off; both
    references came from official blogs, and the in-game render was visibly different - Blastoise has more blue on the
    right edge, Steelix was "jarring"; Malamar was also poor). Three in-game screenshots (not kept in the repo) were cut out of the dimmed
    UI, registered to the template (affine on the silhouette, then a smooth mesh warp guided by the emblem edges to absorb
    the wiggle animation; max ~2 template px), sampled with 4x supersampling and written as 128 px silver standards with
    the template's own alpha (rim pixels that blended with the background are refilled from the nearest interior colour).
    Against the generic material they score Steelix 36.4 -> 14.2, Blastoise 30.7 -> 23.7, Malamar ~43 -> 20.1; **refitted,
    all three land at the noise floor** (Steelix 4.2, Blastoise 5.2, Malamar 5.5, down from 21.8 / 14.5 / 13.2) with floats
    inside the common cluster (stops ~0.4 / 0.6 / 0.78, tint ~0.06, glow ~0.8, curve ~0.16). So the bad fits were bad
    references, not the model. Blastoise moved from gold to silver (gold is now 7 refs); Malamar's best set wants glass mode 1
    and `_GlowColorIntensity` at 1 (identified, at the bound). Updated numbers: mean RMSE generic -> fitted gold(7) 35.4 -> 12.5,
    silver(44) 23.5 -> 9.6, bronze(9) 30.7 -> 11.0; 27 fits <= 10, 26 <= 15, 7 above 15 (Garchomp, Aerodactyl, Pidgeot,
    Gallade, Alakazam, Ampharos, Falinks). `_BrightnessCurve` median 0.155 (MAD 0.012, 53 fits) - pinning it at 0.15 costs a
    median 0.10 RMSE; the 7 gold refs still fit ~0.04. The colour-regression guess for unreferenced entries averages 19.8
    (leave-one-out) vs 25.9 for the generic material. Any remaining poor reference is a candidate for the same treatment.
26. **Minimum stop spacing, and Staraptor + Garchomp redone** (2026-09-20, user: Malamar's stops were too close to each
    other; avoid that; also redo Staraptor and Garchomp from in-game screenshots). Fits had drifted to stops only 0.02-0.09
    apart (33 of 60 fits had a gap < 0.1; `_Stop4` mostly sat on top of `_Stop3` because it is barely identified). Rule now:
    neighbouring stops >= 0.1 apart (`MIN_STOP_GAP` in the lab's descent; fits also use joint shifts of adjacent stops,
    since coordinate descent stalls on the boundary). Cost measured over all 60 species: gap 0.05 = +0.00 mean RMSE, **0.10 =
    +0.06 (worst Heracross +0.56)**, 0.15 = +0.69 (14 species lose > 1). Malamar at 0.10: 5.52 -> 5.59, stops 0.30 / 0.594 / 0.694 /
    0.90. Consequences: `_Stop4` is no longer a near-constant (median 0.87, rides at `_Stop3` + gap) - verdict "weakly
    identified"; pinning it now costs more. Staraptor and Garchomp (bronze, blog images) replaced by cleaned screenshots
    -> silver: Staraptor 15.4 -> 4.5, Garchomp 24.2 -> 5.9 (were 11.3 / 21.9). Now 7 gold / 46 silver / 7 bronze; mean RMSE generic
    -> fitted 35.4 -> 12.5 / 23.3 -> 9.5 / 30.9 -> 9.5; 29 fits <= 10, 25 <= 15, 6 above 15 (Aerodactyl, Pidgeot, Gallade,
    Ampharos, Alakazam, Falinks). `_BrightnessCurve` median 0.156 (MAD 0.013); pinning it costs a median 0.13. Guess for the
    36 unreferenced entries: LOO mean 19.8 vs 25.9 generic.
27. **Legacy Charizard and Mewtwo added as references** (2026-09-20, user: they are still legit). The base entries `0006` Charizard
    (silver) and `0150` Mewtwo (bronze) - the original single icons, distinct from the X / Y entries - had no reference. With the
    new images: Mewtwo 18.5 -> 7.7 (float set inside the main cluster: stops 0.33 / 0.55 / 0.77 / 0.90, tint 0.11, glow 0.79, curve
    0.16); Charizard 27.2 -> 12.6 but as an outlier (`_Stop1` = 0, curve 0.02), like Charizard X / Y (12.3 / 12.8) - all three Charizard
    references may share a source that is off. Now 62 fitted (7 gold / 47 silver / 8 bronze) and 34 inferred; mean RMSE generic ->
    fitted 35.4 -> 12.5 / 23.4 -> 9.5 / 29.4 -> 9.2; 30 fits <= 10, 26 <= 15, 6 above 15 (unchanged list). Constants unchanged
    (`_BrightnessCurve` median 0.156, MAD 0.013; pin costs a median 0.13). Guess for unreferenced entries: LOO mean 19.9 vs 25.5 generic.
28. **Reference provenance from PNG metadata** (2026-09-20, user: the blog images carry imagemagick.org in their metadata and
    were used for the gold standard). Parsing the PNG text chunks: all 7 gold and 5 silver references (legacy Charizard,
    Abomasnow, Ampharos, Gyarados, Lopunny) carry `software: https://imagemagick.org` and `Thumb::URI ... /tmp/thumblr/...`
    (a wiki thumbnailer), created 2020-10-15 .. 2021-05-03; the current `GO_Mega_Energy.png` (2021-06-08) too. Other groups:
    Adobe XMP (13), sRGB-tagged (6), EXIF (5), no metadata (21), cleaned in-game screenshots (5). The 12 ImageMagick references
    fit worst on average (12.3 vs 5-10) and 9 of 12 fit `_BrightnessCurve` < 0.1 (0 of the other 49; median 0.082 vs 0.153-0.174),
    5 of 12 fit `_Stop1` = 0. It is NOT a simple tone or blur artifact: for the generic icon the true material is known, and the
    old generic image (no metadata) matches it at RMSE 7.2 with the curve minimum exactly at the true 0.15, while the ImageMagick
    generic image scores 26.6 and no curve, blur (sigma 0-2.5), gamma or per-channel affine correction brings it below ~22. More
    likely these are older-client renders (Oct 2020 - May 2021) of a material / shader that has since changed (the shader was
    read from build 0.429.1): fitted curve rises with the thumbnail date (Oct 2020 refs 0.00-0.09 with `_Stop1` = 0 in 4 of 5;
    Spearman 0.55, p = 0.06, n = 12 - suggestive, not proven). Consequence: "gold" is a misnomer for the current game;
    those references should be replaced by current in-game screenshots (worst first: Pidgeot, Ampharos, Venusaur, Lopunny,
    Charizard, Beedrill, Gengar) or kept out of the constants statistics; the old generic image, not the ImageMagick one, is
    the valid check of the recovered default material.
    **Direct test (Venusaur, in-game screenshot vs the gold thumbnail, 2020-10-15):** the cleaned screenshot differs from the
    gold image by RMSE 38 (gold is a saturated teal-green with a purple base, mean luma 163 / std 36; the screenshot is a pale
    pastel with a pink rim, luma 194 / std 25). The screenshot fits at RMSE 5.97 (noise floor) with `_BrightnessCurve` 0.169
    (interval 0.167-0.169), `_GlassTint` 0.15, `_GlowIntensity` 0.84, stops 0.417 / 0.632 / 0.732 / 0.838 - a main-cohort
    material. The gold fit (14.9) had curve 0, `_Stop1` 0.225, glow 0.59, `_GlowColorIntensity` 0. The gold-fitted floats score
    35.9 against the screenshot; changing only their curve to 0.15 gives 14.7. So the gold thumbnail is a render of an older
    build (no mid-tone brightening), not a bad crop; every gold-derived float for it was wrong.
    **Replacement (2026-09-20, later):** screenshots of Venusaur, Beedrill, Gengar, Lopunny and Pidgeot were cleaned and
    installed as silver (Beedrill, Gengar, Pidgeot, Venusaur were gold), and all five refit at the noise floor: Venusaur
    5.97 (was 14.9), Beedrill 5.70 (12.0), Pidgeot 5.30 (17.3), Gengar 4.01 (12.0), Lopunny 4.97 (14.3). Their fitted
    `_BrightnessCurve` is 0.144-0.169 (was 0.00-0.08 for the thumbnails) and every one lands in the main cohort. Now 62 fitted
    (3 gold / 51 silver / 8 bronze); mean RMSE generic -> fitted 31.0 -> 10.3 / 23.6 -> 9.0 / 29.4 -> 9.2; 35 fits <= 10, 57 <= 15,
    5 above 15 (Aerodactyl, Gallade, Ampharos, Alakazam, Falinks). Curve median 0.156 (MAD 0.011) over the 57 fits <= 15, pin
    cost median 0.08 (cumulative curve+tint+glow pins: 0.55); tint 0.089, glow 0.781 - verdicts unchanged. Seven ImageMagick
    thumbnails remain (Altaria, Houndoom, Manectric gold; Charizard, Abomasnow, Ampharos, Gyarados silver): curve 0.02-0.16,
    median 0.09. Guess for unreferenced entries: LOO mean 18.7 (was 19.9) vs 24.7 for the generic material. Lesson: the
    "gold" tier was the least trustworthy one, and the constants were already visible through it; cleaner references only
    sharpen them.
