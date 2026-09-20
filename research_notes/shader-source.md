# The real shader — recovered from the APK (2026-09-20)

Everything else in this directory was inferred by fitting images. This file is different: it is read
off the game's own files. **Where it disagrees with another note, this file wins.**

Pokémon GO 0.429.1 (Android, Unity 6000.0.53f1, URP, IL2CPP) ships the Mega Energy shader inside the
APK, compiled for GLES3 — which Unity stores as GLSL *text*, so the fragment program is directly
readable. The decompiled listing is Niantic's code and is **not** kept in this public repo; what
follows is the maths restated, plus where to find the original again.

## Where things live

| Thing | Location |
|---|---|
| Shader `NianticCustom/UI/MegaCandy` | APK `assets/bin/Data/sharedassets1.assets` (split files), pathID 1492. One pass, `Blend SrcAlpha OneMinusSrcAlpha`, Transparent queue, standard UI stencil / clip-rect properties. |
| Template texture `pokemon_details_mega_candy` | same file, pathID 462 — **256×256**, ASTC, no mips, bilinear, clamp. `raw/pokemon_details_mega_candy.png` in this repo is a 128×128 copy. Its sprite (pathID 2398) covers the full texture and is **not atlased**, so UVs run 0–1 across the icon. |
| Material `MegaCandyDefault` | same file, pathID 330 — the generic (species-less) Mega Energy icon. |
| 96 per-species materials `MegaCandy_<key>.mat` + `MegaCandyData.asset` | **remote** Addressables bundle `megacandymaterials_assets_all_<hash>.bundle` (downloaded at runtime; not in the APK, not in PokeMiners' repo or scratch folder). Asset paths `Assets/UI/GuiWindows/PokemonInfoGuis/MegaCandy/MegaCandy_0003.mat` … — the file-name suffixes are exactly the keys of `raw/PokemonMegaCandyAkaMegaEnergy.json`, which is a colours-only dump of these materials. |
| C# side | `Niantic.Holoholo.UI.PokemonMegaCandyData` (ScriptableObject: `MegaCandyMaterials`, `MegaCandyBranchMaterials`) and `PokemonMegaCandyWidget` (`MegaCandyImage`, `SetMegaCandyData`) — it swaps the whole **Material** on a UI Image; it does not push colours one by one. |
| Colour space | `PlayerSettings.m_ActiveColorSpace = 0` → **Gamma**. Texture and colour values are used raw (sRGB numbers), no linearisation anywhere. |

## Shader properties

`_MainTex`, `_Color`, `_RampColor1..4`, `_GlowColor`, **`_Stop1..4`**, **`_GlassColor`** ("Glass Mode",
0/1), **`_GlassTint`**, **`_GlowIntensity`**, **`_GlowColorIntensity`**, **`_BrightnessCurve`**, plus UI
stencil / colour-mask plumbing. **`_EmissionColor` is not a property of this shader** — it is a stale
Standard-shader leftover in the material and is never read.

| | `_Stop1..4` | `_GlassColor` | `_GlassTint` | `_GlowIntensity` | `_GlowColorIntensity` | `_BrightnessCurve` |
|---|---|---|---|---|---|---|
| Shader defaults (seen as leftovers on two unrelated fx materials) | 0 · 0.3 · 0.6 · 1.0 | 0 | 1.0 | 1.0 | 0.5 | 0.15 |
| `MegaCandyDefault` material | 0 · 0.594 · 0.78 · 0.806 | 0 | 0.1 | 0.56 | 0.5 | 0.15 |
| Per-species materials | **unknown — not dumped by anyone yet.** Estimates fitted against the references: `raw/MegaCandyMaterialFloats.json` (finding #24). Near-constants across species: curve ~0.15, glass tint ~0.1; glow intensity ~0.77 and Stop4 ~0.84 probably; stops 1-3 vary | | | | | |

`MegaCandyDefault` colours: Ramp1 (0.720, 0.476, 0.559) · Ramp2 (0.146, 0.173, 0.275) ·
Ramp3 (0.484, 0.535, 0.708) · Ramp4 white · Glow (0.544, 0.645, 0.741) · `_Color` white.

## The maths

Per pixel, with template sample `(R, G, B, A)` and UV `(u, v)` — `v = 0` at the **bottom** of the icon:

1. **Ramp over R** — a plain 4-stop gradient, each segment clamped:
   `s1 = sat((R−Stop1)/(Stop2−Stop1))`, `s2 = sat((R−Stop2)/(Stop3−Stop2))`, `s3 = sat((R−Stop3)/(Stop4−Stop3))`;
   `c = mix(Ramp1, mix(Ramp2, mix(Ramp3, Ramp4, s3), s2), s1)`.
2. **"Glass" layer — a position gradient that differs per RGB channel.** Mode 0:
   `g = ( sat(1.3·u), sat(1.3·v), sat(1.4 − |(u, v)|) )` — red rises left→right, green bottom→top, blue
   falls off with distance from the bottom-left corner. Mode 1 is the same thing rotated:
   `g = ( sat(1.3·(1−v)), sat(1.3·u), sat(1.4 − |(u, v−1)|) )`. `_GlassColor` lerps mode 0 → mode 1.
3. **Blend `c` with the glass layer, per channel:** where `g ≤ 0.5` → `1 − 2(1−g)(1−c)`; where `g > 0.5`
   → `2·g·c`. (An overlay blend with its two branches swapped — continuous at 0.5, steeper than a true
   overlay, and *not* clamped: it can leave 0–1.) Then `c = mix(c, blended, _GlassTint)`.
4. **White highlight from B:** `c = mix(c, white, B · _GlowIntensity)`. The helix emblem and the specular
   streak go toward **white**, not toward `_Color`.
5. **Glow colour from G:** `c = mix(c, _GlowColor, G · _GlowColorIntensity)`. Only the thin outline.
6. **Brightness curve:** `c += _BrightnessCurve · (1 − (2c − 1)²)` — a mid-tone lift, zero at black and
   white, +`_BrightnessCurve` at 0.5.
7. **Output:** `rgb = c · vertexColour · _Color` (a whole-icon multiplicative tint — the UI Image colour);
   `alpha = sat(A + G · _GlowColorIntensity) · A · vertexAlpha`.

The vertex stage only transforms the quad and passes `uv` and `vertexColour × _Color`.

## Verification

- **Generic icon: every parameter known, nothing fitted.** `MegaCandyDefault` rendered through the maths
  above vs `raw/standards/silver/GO_Mega_Energy.png` (a 128 px image upscaled to 256): **RMSE 9.3**, and
  by eye the same icon. Glass mode 1 scores 13.2 — mode 0 is right. This is the evidence that the decode
  is correct.
- **Gold species refs with `MegaCandyDefault`'s floats + the JSON colours: mean RMSE 34.8** (old evenly
  spaced 1-D ramp on the same metric: 39.5). Renders come out too pale: the references show a mid-tone
  body and a strongly coloured bottom-right rim. No single float set found in the APK fits every species
  — shader-default stops with the default material's glass values fit Altaria (16.3) and Beedrill (29.4)
  far better but Gengar / Blastoise far worse (98 / 87). **Conclusion: the per-species materials carry
  their own `_Stop*` / glass / glow floats**, and those are the missing input — not more shader maths.
  (RMSE here = opaque-interior pixels only, refs upscaled 128 → 256; not comparable digit-for-digit with
  the lab's numbers.)
- **All 60 references, 128 px template, nothing fitted** (contact sheet, 2026-09-20). Mean RMSE with the
  generic material's floats: gold 34.8 · **silver 24.5** · bronze 30.4; the generic icon itself 7.2. About
  twenty silver / bronze species already land at 11–18 and read as the same icon by eye (Latias 10.9,
  Pinsir 10.8, Latios 13.0, Diancie 13.5, Mawile 13.6, Metagross 13.6, Absol 15.2, Rayquaza 15.2,
  Heracross 15.4, Slowbro 15.4, Kangaskhan 16.1, Mewtwo Y 16.8, Swampert 16.9, Charizard X 17.2 …) — so
  many species materials evidently keep floats close to `MegaCandyDefault`. A second group comes out far
  too dark with those floats and is much closer with the shader-default stops 0 / 0.3 / 0.6 / 1 (same
  glass / glow values): Altaria 34 → 17, Audino 30 → 19, Camerupt 36 → 20, Chesnaught 38 → 21,
  Sharpedo 36 → 22, Victreebel 46 → 24, Delphox 51 → 26, Blaziken 70 → 27, Malamar 42 → 27,
  Raichu X 62 → 28, Beedrill 50 → 30. Two families of material (one cloned from the default, one made
  with other stops) would explain it; only the real floats can confirm. Gold is the worst-fitting tier
  under either set, which supports open question #2 (gold refs from an older pipeline).

## What this overturns

- "B tints toward `_Color`" → B lerps toward **white**, strength `_GlowIntensity`. `_Color` multiplies the
  whole icon.
- "`_EmissionColor` is the dark end of the shading mix" → **not used at all.** The dark body is `Ramp2`
  itself (the darkest stop on 80/96 entries) sitting at `_Stop2`.
- "`_GlowColor` is a directional light blended into the ramp" → `_GlowColor` touches only G-channel
  pixels. The position-dependent hue is the **glass layer** (step 2–3): a fixed per-channel UV gradient,
  same direction for every species (unless a material sets Glass Mode 1).
- "Tone is a multiplicative R-driven gain" → there is no gain term. Tone = the glass blend (partly
  multiplicative where `g > 0.5`) + the additive mid-tone parabola.
- Finding #19's "two-regime R→position, plateau below R ≈ 0.56" was the real `_Stop2 = 0.594` showing
  through: 24% of opaque template pixels sit below it, 70% between Stop2 and Stop3.

Confirmed unchanged: R drives ramp position; G = outline → `_GlowColor`; compositing is in sRGB.

## Next step

Get the floats of the 96 materials. In order of effort: ask PokeMiners to re-dump
`PokemonMegaCandyAkaMegaEnergy.json` with each material's `m_Floats` (they already hold the bundle);
or pull the game's downloaded bundle cache from an Android device that has opened a Mega screen and read
`megacandymaterials_assets_all_*.bundle` with UnityPy. No client modification is needed for either.

## Reproducing this

Unpack the APK (it is a zip), load `assets/bin/Data` with Python `UnityPy`, read the Shader object's
type tree, LZ4-decompress its `compressedBlob` per `offsets` / `compressedLengths` /
`decompressedLengths`, and take the text from each `#version` marker. Addressables locations:
`assets/aa/catalog.json` (`m_EntryDataString` → dependency bucket → bundle name).
