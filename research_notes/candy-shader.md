# The Candy / Candy XL shader — recovered from the APK (2026-09-26)

The same exercise as [`shader-source.md`](shader-source.md), for the ordinary striped Candy and Candy XL icons.
Where Mega Energy needed nine unknown per-species floats, the candy turned out to be the simplest possible case:
a linear three-channel mask with nothing to fit. The decompiled listing is Niantic's code and is **not** kept in
this repo; the maths is restated below.

## Where things live (Pokémon GO 0.429.1, Android, Unity 6000.0.53f1, URP, Gamma colour space)

| Thing | Location |
|---|---|
| Material `GeneralCandyMat` | APK `assets/bin/Data/sharedassets1.assets`, pathID 210. Colours: `_ColorR` (1,0,0) · `_ColorG` (0,1,0) · `_ColorB` (1,1,1) · `_Color` white (not read) · a stale `_EmissionColor` and Standard-shader floats (leftovers, not read). |
| Shader `NianticCustom/UI/TriColorSplat` | referenced through the file's externals table (entry 288 → file `dbea9233fafdc404cb980513cf22c122`, pathID 1). One pass, `Blend SrcAlpha OneMinusSrcAlpha`, ZWrite off, Transparent queue; keywords only the UI clip-rect / alpha-clip variants (6 GLSL programs, 20,708 bytes decompressed). |
| Template `candy_rgb` | same assets file, pathID 620 — 256×256, ASTC 6×6, no mips; its sprite (pathID 2517) covers the full texture, not atlased. PokeMiners' `candy_rgb.png` is the same mask: inside the icon the two differ by < 1 level per channel; the only differences sit on the anti-aliased alpha edge (ASTC). |
| `candy_vector_rgb` | pathID 961 / sprite 2782 — a 64×64 three-channel list icon of the same kind (not in the PokeMiners clone, not used by the lab). |
| Candy XL template | **not in the APK** (remote Addressables bundle). `LV40_XLCandy_RGB_PSD.png` (220×232) follows the same R/G/B-mask convention, and the C# has a `PokemonXlCandyWidget`, so it is assumed to run the same material with the sprite swapped. **It is the cut-out art, not the icon frame:** in the game's 256×256 icon its opaque box sits 14 px right and 19 px down (`raw/candy/standards/GO_Nymble_Candy_XL.png`, an in-game icon: opaque box x 17–230 / y 22–247 vs the template's x 3–216 / y 3–228; alpha RMSE 1.9 at that offset, ≥ 5 anywhere else). `candy_rgb.png` already is the frame (box centred at x 21–234 / y 21–234). The lab places the XL template at (14, 19) in a 256×256 canvas. |
| Colour data `PokemonCandyColorData` | a ScriptableObject in a remote bundle; PokeMiners' dump is `raw/candy/PokemonCandyColorData.json` (549 families, `FamilyId` = base species dex, `PrimaryColor` / `SecondaryColor` as 0–1 floats, alpha always 1; family 0 = white / magenta placeholder). |
| C# side | `Niantic.Holoholo.UI.PokemonCandySpritesService` (`GeneralCandyMaterial`, `_candyColorData`, `FamilyColor`, `CandyMaterial`) and `PokemonCandyWidget.ColorizeCandy` / `SetCandyColor`. IL2CPP: the code is native (`libil2cpp.so`); only the names and the literal strings `_ColorR` / `_ColorG` / `_ColorB` are readable in `global-metadata.dat`. |

## Shader properties

`_MainTex` ("Sprite Texture"), **`_ColorR`** ("R Channel Tint"), **`_ColorG`** ("G Channel Tint"), **`_ColorB`**
("B Channel Tint"), plus the standard UI plumbing (`_StencilComp`, `_Stencil`, `_StencilOp`, `_StencilWriteMask`,
`_StencilReadMask`, `_ColorMask`, `_UseUIAlphaClip`, `_DepthOffset`). Nothing else — no stops, no glass, no curve.

## The maths

Per pixel, with template sample `(R, G, B, A)`:

```
t   = (R, G, B, A) + _TextureSampleAdd          // 0 for an RGBA texture
rgb = t.R · _ColorR + t.G · _ColorG + t.B · _ColorB
a   = t.A                                       // × the UI clip-rect test; the ALPHACLIP variant discards a < 0.001
```

- **No clamp** in the program — the 8-bit render target clamps on write, which is what the lab does too.
- **No vertex colour.** The vertex stage passes only position and UV, so a UI Image's colour / alpha never reaches the
  candy (unlike Mega Energy, where `_Color` multiplies the whole icon).
- **sRGB throughout** (project colour space is Gamma), so the JSON floats are used as they are.

Template channels: R = the stripes, G = the body, B = the specular flecks (they are G + B, cyan, in the template, so a
fleck is body colour + white → clamped white). The ~430 near-black outline pixels stay dark because every channel is
near zero there. Stripes and body never overlap (no pixel has R and G both above 128).

## What this settles

- **`src/lib/candyColor.js` in the main app has been the game's exact maths all along**: `out = R·Secondary +
  G·Primary + B·white`, alpha preserved. The lab's Candy tab renders the same formula on the same two templates; a
  headless check of five pixels (body, stripe, centre, and the XL equivalents) against a Python evaluation matched
  exactly (2026-09-26).
- The "asset colour mask" recipe in the parent repo's `asset-color-mask` skill is therefore verified, not inferred.

## The channel mapping — confirmed against an in-game icon (2026-09-26, later)

`GO_Nymble_Candy_XL.png` (user-supplied in-game Candy XL, family 919: Primary (96, 112, 137), Secondary
(227, 226, 229)) rendered through the formula with `_ColorG ← PrimaryColor`, `_ColorR ← SecondaryColor`, `_ColorB`
white lands at **RMSE 7.6** over the 33,933 pixels opaque in both (mean offset −2.7 per channel — the icon is a
touch darker, within compression / resampling noise); the swapped mapping scores **99.3**. So the mapping is no
longer an assumption. The earlier reasoning is kept below for the record.

### What was assumed before that

**Which JSON colour the game writes into which channel.** `GeneralCandyMat` ships `_ColorR` red, `_ColorG` green,
`_ColorB` white; `PokemonCandySpritesService` then sets colours per family, and that code is native. The mapping used
everywhere here — `_ColorG ← PrimaryColor` (body), `_ColorR ← SecondaryColor` (stripes), `_ColorB` left white — is the
main app's long-standing one and matches in-game candy from memory (Charmander: Primary orange body, Secondary cream
stripes; Bulbasaur: teal body, light-green stripes). The pre-painted candy art in the PokeMiners clone
(`pokemon_details_candy.png`, the Today-view icons, `trade_tut_candy_bonus.png`) is generic pink / tutorial art and
cannot confirm it. The lab's **Swap base ↔ stripe** button shows the other reading. To settle it for good: one
in-game screenshot with a candy in it, or Il2CppDumper + Ghidra on `PokemonCandySpritesService.FamilyColor`.
(User decision 2026-09-26: assume the current mapping.)

## Reproducing this

Local tooling in the gitignored `apk_research/` of the private checkout: `scan_candy.py` (finds the material,
resolves its shader through the externals table, lists the candy textures / sprites and the C# names, exports the APK's
`candy_rgb` texture → `scan_candy.json`, `candy_rgb_from_apk.png`) and
`extract_shader.py "NianticCustom/UI/TriColorSplat" tricolorsplat` (LZ4-decompresses the shader blob and cuts the
GLSL at each `#version` → `tricolorsplat_glsl.txt`). Python `UnityPy` + `lz4`, nothing else.
