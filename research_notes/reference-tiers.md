# Reference tiers — and why unweighted error is misleading

`raw/standards/{gold,silver,bronze}/` holds real in-game/official icons to check any model against.
They are **not equally trustworthy**, and fitting against them as if they were is the single biggest
trap in this project — it's what caused the early "washed out" models everyone rejected on sight.

## The tiers

- **gold/** — original announcement art, exact tone. The small set of currently-identified gold refs
  (Venusaur, Charizard `0006`, Blastoise, Beedrill, Pidgeot, Gengar, Houndoom, Manectric, Altaria,
  Lopunny) is the only set worth fitting *tone* (contrast, lift, blend math) against.
- **silver/** — clean art, tone mostly trustworthy. Includes Bulbapedia reference art and the
  user's own cropped in-game screenshots from the evolve dialog (true in-game tone, no halo).
- **bronze/** — in-game screenshots with hand-erased backgrounds; lossy, can have bad pixels. Use
  mainly for layout/hue sanity, not tone.

A reference's tier is just which subfolder it's sorted into — there's no algorithmic tier detection.

## Why this matters: gold-weighting fixed "washed out"

Fitting with plain unweighted RMSE across every tier hedges every color toward gray, because bronze
and silver outnumber gold and are individually less reliable. Gold refs consistently want **almost
no white lift** — bronze/silver artifacts make lift look necessary when it isn't. Switching to a
gold-weighted loss (`gold² + 0.15·silverRaw² + 0.5·silverTone² + 0.3·bronzeTone²`, where "raw" means
scored directly and "tone" means after a per-reference scalar gain+offset correction) dropped the
global white-lift parameter from 0.16 to 0.05 and gold error from 28.8 → 15.6 in one pass. **Any
new fitting approach should gold-weight its loss, or fit gold-only first and treat other tiers as a
sanity check**, not average them in unweighted.

## Two art "generations" among the gold refs

The older gold refs (Venusaur, Charizard, Blastoise, Beedrill, Pidgeot, Gengar, Houndoom — 7 of them)
have a blurrier, embossed emblem offset by roughly (+3, −2) px and scaled ~0.98 relative to the
current template. Manectric, Altaria, and Lopunny match the current template exactly. If you're
doing anything emblem/geometry-sensitive, treat these as two different renders and don't average
them naively — and if you add more gold refs later, check which generation they belong to first.

## Don't expect exact reproduction

Even fit per-reference (the most generous possible test — one set of knobs per single image, no
generalization required), gold RMSE floors out around 10–18 depending on the species; a shared model
across all species floors around 24–30 (see `models-tried.md`). Multiple independent lines of
evidence (per-reference exposure varying 0.5–1.5× with no correlation to palette luminance; free
per-pixel palette fits still leaving residual error) point to the same conclusion: **the gold
reference art may not be the deterministic output of one shader over today's JSON colors** — it
might be older/hand-touched marketing art rather than a straight render. Judge new models by
"is this an improvement", not "does this hit zero error".
