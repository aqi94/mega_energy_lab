# Inputs: what the template and JSON actually mean

> **Superseded in part (2026-09-20).** The real shader was recovered from the APK — see
> `shader-source.md`. Still true below: R drives ramp position, G is the outline and goes to
> `_GlowColor`, the JSON is a Material dump, `_EmissionColor` is a stale leftover. **Wrong below:** B does
> not tint toward `_Color` (it lerps toward white; `_Color` multiplies the whole icon), `_EmissionColor`
> is not a shading term (the shader never reads it), and `_GlowColor` is not a directional light (the
> positional hue is a per-channel UV "glass" gradient). The APK's template is 256×256; the copy here is
> 128×128. The material also carries floats the JSON lacks: `_Stop1..4`, `_GlassColor`, `_GlassTint`,
> `_GlowIntensity`, `_GlowColorIntensity`, `_BrightnessCurve`.

## `raw/pokemon_details_mega_candy.png` (128×128 RGBA)

The template is not a plain color mask — each channel is doing a different job:

- **R (64–255, bright top-left → dark bottom-right):** drives shading / ramp position. This is by
  far the dominant coordinate — hue-based ramp-position recovery (fit a free per-pixel gain, then
  regress recovered position on R) gives r² = 0.93 across stops. Any new model should treat R as
  "where in the 4-color ramp this pixel sits" by default, and only reach for x/y or other geometry
  if R alone can't explain something.
- **B (helix emblem ~80, plus a top specular streak > ~115):** does **not** take ramp colors. It
  tints toward `_Color` (see below).
- **G (thin, ~1px outline, ~160):** does **not** take ramp colors either. It tints toward
  `_GlowColor`.

## `raw/PokemonMegaCandyAkaMegaEnergy.json`

96 keys (dex-padded like `0006`, plus `_MEGA_X`/`_MEGA_Y`/`_MEGA_Z`/`_Mega_Z` suffixes for
multi-form Megas — the underscore casing is inconsistent in the source data, e.g. `0448_Mega_Z` vs
`359_Mega_Z` unpadded). Each entry has 7 colors as floats 0–1: `_Color`, `_EmissionColor`,
`_GlowColor`, `_RampColor1..4`. Treat these as the ground-truth palette (they're ripped from the
game), not as something to refit.

**Confirmed color roles** (2026-09-14/15 findings, high confidence — see `findings-timeline.md` #16,
#20 for the evidence):

- `_RampColor1..4` are the 4-stop gradient keys placed along the R-driven position `t`. Layout by
  eye: Ramp1 near the bottom-right rim, Ramp2 bottom-left, Ramp3 mid/upper-right, Ramp4 top-left —
  but a plain "4 spatial poles at quadrant centers" model was explicitly rejected in favor of a
  1-D ramp over R once the R-correlation was found (see `models-tried.md`).
- `_EmissionColor` is **black on 96/96 entries**. It is the **dark end of the shading mix** —
  `mix(_EmissionColor, ramp_color, b·R)` — not a background canvas behind the alpha (an earlier
  "canvas" interpretation produced a jarring black outline and was retracted after user testing).
- `_Color` is white on 92/96 entries (the 4 exceptions: `0208`, `0384`, `0689`, `359_Mega_Z`). It
  tints the **B channel** (helix + streak) — `lerp(white_base, _Color, kb)`.
- `_GlowColor` tints the **G channel** (outline) the same way, *and* separately acts as a
  **directional light blended into the ramp itself** from one side of the icon (not a screen/additive
  glow) — see `findings-timeline.md` #20 (the Starmie discovery) for why this is a positional lerp,
  not a screen blend.

## Why this data can't just be composited literally

The `PokemonMegaCandyAkaMegaEnergy.json` values are very likely a dump of a Unity **Material**'s
`m_SavedProperties`. `_EmissionColor` black + `_Color` white-by-default across nearly all entries
matches the Standard-shader defaults — i.e. probably stale leftovers from a template material whose
shader got swapped to the actual Mega shader, not necessarily inputs the shader reads directly the way
their names suggest. *(Confirmed 2026-09-20: `_EmissionColor` is not a property of the real shader at
all. The shader — `NianticCustom/UI/MegaCandy`, which uses no `_Ramp2D` lookup texture — ships in the
APK and has been read; see `shader-source.md`. What is still missing is the per-species materials'
float values, which sit in a remote bundle.)*
