# Models tried, and the current best-known formula

**Current lab state:** `mega_lab.html`'s Shader menu now holds only the Venusaur-fitted model
described below (RMSE 11.7 on Venusaur). The generic/gold-wide shared-defaults version and the
universal per-pixel weight map (both described further down, since the write-ups are still useful
history) were removed from the lab as failed experiments — by eye, only the Venusaur fit looked
anywhere close to right; nothing else did. **Don't mistake "the only model in the lab" for "a
validated general solution"** — it's a single-species fit kept as the best starting point, not
something known to generalize (see universal rule #9 in `../CLAUDE.md`).

Lower RMSE is better. "Gold RMSE" = mean error over the gold reference tier only (the only tier
worth chasing tone accuracy on — see `reference-tiers.md`); "shared" means one set of knob values
used across every species, vs. "per-reference"/"per-species" meaning the knobs were refit for each
image individually (an upper bound on how good *any* model in that family could look, not something
you can ship as-is).

## Current best shared model (as of the last recorded session, 2026-09-15)

A 1-D ramp positioned by template R, in this pipeline order:

1. **Position:** `t = (R − r0) / (r1 − r0)`, optionally with a low **rim plateau** (`t = 0` below
   `R ≈ rs`) before the climb starts — R alone predicts ramp position with r² 0.93, confirmed
   independently multiple times (see `findings-timeline.md` #1, #18).
2. **Ramp color:** 4-key gradient through `_RampColor1..4` at positions `0, p2, p3, 1` (p2 ≈ ⅓,
   p3 ≈ ⅔ by default, both fittable). Interpolation space (sRGB vs OKLab) is a toggle — both score
   about the same (11.7 either way on Venusaur); HSV interpolation is worse (14.1) and was dropped.
3. **Wash:** blend the ramp color toward `_Color` (white on 92/96 entries), strength driven by R
   (knobs `kw`, `w0`, `w1`), applied *before* shading. This alone improved the Venusaur fit and
   lifted the shared gold-wide fit from 30.2 → 24.8 RMSE.
4. **Glow light:** blend toward `_GlowColor` based on a directional coordinate `u`, rotated by an
   angle `ga` (default 45° = light from the upper-right; some species, e.g. Venusaur, want light
   from a different side — `ga ≈ 111°` fit best there), smoothstepped around a center `gc` and
   width `gw`, strength `kl`. This is `_GlowColor` acting as a **positional light on the ramp
   itself**, not a screen/additive glow overlay — see `findings-timeline.md` #20.
5. **Shade:** mix toward `_EmissionColor` (always black) by `b·R` — the dark end of the shading.
6. **Exposure:** global gain `g`.
7. **Channel tints:** B (helix + streak) → lerp toward `_Color` by `kb`; G (outline) → lerp toward
   `_GlowColor` by `kg`.

Accuracy: with **shared** default knobs, gold RMSE ≈ 39 (the per-reference knobs are genuinely where
the fitting work needs to happen — this is a starting point, not a tuned model). Per-reference fits
reach gold RMSE ≈ 17.8 shared-family / down to 10.9–12.6 for individually-tuned species like
Venusaur. A **gold-wide joint fit** (one set of knobs fit across all 10 gold refs at once, which is
the realistic "can we ship one formula" test) lands around 24.8–25.5.

## Why it looks like this (rejected alternatives, roughly newest-first)

- **Quadrant color poles** (Ramp1–4 planted at the four quadrant centers, Gaussian-weighted) was the
  very first model and is the simplest possible reading of "4 colors, 4 corners". The user explicitly
  preferred evaluating this by eye over any fitted alternative at one point ("assume the simplest
  solution", "the user rejected every transform they had not asked for") — treat "start simple, add
  a transform only when there's specific evidence for it" as the working style here, not just a
  historical note.
- **Free 4-pole spatial layout, fit per species** gets very good single-species scores (Venusaur/
  Manectric ≈ 9.6–10.9) but **the pole positions are different per species** (Manectric wants Ramp3
  top-left where Venusaur wants Ramp4 there) — a joint fit across species collapses back toward the
  center and loses most of the gain. No single spatial layout generalizes.
- **Two-gradient / split multiply-add / luma-from-template-chroma-from-ramp** models (an earlier
  research phase) all landed in the same gold ≈ 15–30 range depending on gold-weighting — they're
  numerically close enough to the ramp1d family that the deciding factor was visual judgment, not
  RMSE.
- **Additive white lift** (mix toward white by a constant) is what produced the "washed out" look
  early on and was replaced by the multiplicative wash — see `reference-tiers.md` for why unweighted
  fitting made this look necessary when it wasn't.
- **Screen-blend glow** was replaced by the positional-lerp glow light after the Starmie reference
  showed the glow hue occupies a specific corner *at every R value* — something a value-dependent
  screen blend can't produce, but a position-dependent lerp can.

## A different, more accurate but less-understood direction: universal per-pixel weights

Instead of a hand-designed formula, factorize every reference image as
`I_ref(pixel) ≈ Σ_k W(pixel, k) · C_ref,k` — one **shared** per-pixel simplex weight map `W` (learned
once, across all species) over 4 basis colors, plus 4 free colors per entry. This reached gold RMSE
**8.9 in-fit / 9.9 leave-one-out** — meaningfully better than any parametric ramp model above.

Open problems with it (see `open-questions.md` for more): Ramp3 never wins a pixel under the learned
weights (its basis slot goes unused), and the colors this approach *fits* for the gold references sit
far from today's JSON values (Ramp1/2 at ~0.75× luminance, Ramp2 +0.39 saturation, hues 12–21° off) —
while the fitted colors for silver references land close to the JSON. That gap is itself a finding:
it's more evidence that the gold art may predate or diverge from the current JSON palette, rather
than the model being wrong. This direction is promising but unexplained — a good place to pick the
work back up if you want to beat the parametric ceiling above.
