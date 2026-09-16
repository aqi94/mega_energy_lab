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
