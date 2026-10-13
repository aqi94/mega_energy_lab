# Medals (badges) — the tier palette from the APK, the tint fitted (2026-09-26)

The third recolour case after Mega Energy and Candy. Medals ("badges" in the code) used to ship as three
pre-painted images per medal (`Badge_38_1_01.png` bronze · `_2_01` silver · `_3_01` gold); since Platinum
medals every medal and every tier ring exists only as an **RGB-mask template** tinted at runtime. Unlike the
two earlier cases the shader itself is **not in the APK** — only its inputs are — so this note has two parts:
what was read out of the game (settled) and what was fitted to in-game screenshots (an estimate).
`medal_lab.html` renders it.

## What the game ships (settled)

Pokémon GO 0.429.1 (Android, Unity 6000.0.53f1, URP, IL2CPP, Gamma colour space).

| Thing | Where | What |
|---|---|---|
| `Niantic.Holoholo.BadgeMaterialConfig` | `sharedassets1.assets` pathID **162909** (a MonoBehaviour; its typetree is unreadable, the bytes were decoded from the metadata's field order — `raw_data/pogo_apk/extract_badge_config.py`) | the tier palette + the per-medal accent colours, below |
| `BadgeRank` | enum in `global-metadata.dat` | NONE 0 · BRONZE 1 · SILVER 2 · GOLD 3 · PLATINUM 4 |
| `HoloBadgeType` | enum in `global-metadata.dat` (1374 values, C#-style names: `BadgeUnown` = 38, `BadgeCaptureSmallPokemon` = 80 …) | the number in `Badge_<n>.png` (sprite-name format `Badge_{0}` / `Badge_Inset_{0}` in the code) |
| `BadgeService` | pathID 110668 | asset addresses only: rings `badge_ring_1…4` (one per tier), frames `badge_frame_0…12`, pin `badge_pin_0`, `tx_badge_sparkles`, materials **`ma_Badge`** (normal frame) / **`ma_Badge_UseFrameColor`** (colourised frame) |
| The medal shader + `ma_Badge*` | **remote** (Niantic's own asset server; not in the APK, not in `aa/catalog.json`) | the code sets `_MetalColorPower`, `_MetalSaturation`, `_DetailColor`, `_DetailStrength`, `_DetailTex`, `_InsetTex` on it; there is no `_MetalColor` string, so MetalColor is most likely the Image's vertex colour (`GetBadgeMetalColor`) |
| Templates, rings, sheen, sparkles | remote too; PokeMiners' `Images/Badges/` has them | see the inventory below |
| `tx_badge_frame_shine` | `metalSheenTextureAddress`; PokeMiners `Misc/tx_badge_frame_shine.png` (32²) | a small diagonal glare sprite (the animated sheen), **not** a lookup ramp |

The four shaders in the APK with "badge" in the name are other things: `NianticCustom/UI/Badge/BadgeShader` /
`BadgeShaderLite` (gym badges: main texture × mask alpha, warp / blend / additive textures), `SpecialRankBadgeGem`
(GBL rank gems) and `RouteBadgeFrameTriColor` (route medals, below). `TriColorSplat` (Candy) is not used for medals.

### `metalSettingsPairs` — one entry per tier

| tier | MetalColor (0–1) | 0–255 | MetalPower | MetalSaturation |
|---|---|---|---|---|
| Bronze | 0.960784 · 0.811765 · 0.654902 | 245 207 167 `#F5CFA7` | 3.74 | 1.0 |
| Silver | 0.694118 · 0.847059 · 0.913725 | 177 216 233 `#B1D8E9` | 4.18 | 1.0 |
| Gold | 1.000000 · 0.874510 · 0.098039 | 255 223 25 `#FFDF19` | 4.42 | 1.0 |
| Platinum | 0.654902 · 0.756863 · 0.811765 | 167 193 207 `#A7C1CF` | 4.00 | 1.0 |

### `detailSettingPairs` — one `DetailColor` per medal (59 entries; the accent, template B)

The full list is `raw/medals/BadgeMaterialConfig.json`. Examples (0–255): Unown 38, Pikachu 37, Youngster 36,
Champion 40, the Mega medals 69/70 → white; Jogger 1 (7,132,246); the Pokédex-region medals (255,9,0); Berry
Master 42 (241,47,123); Gentleman 49 (235,0,255); Tiny / Jumbo Collector 80/81 (167,68,224); Life of the Party 83
(255,0,50); Showcase Star 1004 (216,222,105). Alpha is stored but 0 on all but one entry (unused). Gen5 (56) is
listed twice (first entry wins). A medal without an entry (Rising Star Duo 66, Paldea 82, …) is rendered with a
white accent by the lab — an assumption.

`badgePaddingOverridePairs` = [(1004, EventBadgeAnchorOverride 0.0, StandardBadgeAnchorOverride 0.18)] — Showcase
Star is drawn larger inside its disc (measured 0.80 of the frame vs 0.60 for every other medal).

## The templates (PokeMiners `Images/Badges/`)

- `Achievements/Badge_<n>.png` (57 RGB templates, ≤ 512 px, copied to `raw/medals/templates/`): **R unused**
  (0–36, anti-aliasing noise); **G = the tier metal, with the shading baked into G's intensity** (bulk 130–250,
  low G = outline / recess); **B = the medal's fixed accent** (laurel, Pikachu's purple triangle, party streaks),
  shaded the same way by B's intensity. G and B never overlap except on AA edges. `Badge_1002`, `Badge_5231`,
  `Events/*`, `Types/Badge_18–35`, `Frames/badge_frame_*` and `Inserts/*` are pre-painted, not templates
  (`Badge_Inset_70` is the rainbow `_InsetTex` overlay for medal 70).
- `Frames/badge_ring_1 #1241` (506²) · `badge_ring_2 #100` (512²) · `badge_ring_3 #1166` (504²) · `badge_ring_4`
  (504², byte-identical to `ring_3 #1166`): **pure-G ring templates, one per tier** (`raw/medals/rings/`). The
  `#133 / #1238 / #131` variants are the same rings at 512² with other padding; the 256-px `badge_ring_1/2/3.png`
  are the old pre-painted rings.
- The 44 older medals' `_1_01 / _2_01 / _3_01` art **was baked from these same templates** (resampled onto the
  template and binned by G, each bin's std is 2–10 levels) — but with an **older palette**: against the current
  in-game colours it scores RMSE 25 / 18 / 30 (bronze / silver / gold; Jogger's gold is a pale, desaturated yellow,
  Unown's is close). So it is a check, not a standard.

## The tint — fitted, not read (`raw/medals/MedalTintModel.json`, `scripts/fit_medals.py`)

Standards: four in-game screenshots (`raw/medals/standards/`, JPEG, 1290 px wide): the big medal of Unown (platinum),
Tiny Pokémon Collector (platinum), Life of the Party (platinum) and Showcase Star (silver), plus the row of
bronze / silver / gold / platinum thumbnails under each (≈ 40 px icons, the only current renders of every tier).
The big icon is aligned to its template by the alpha bounding box inside the white disc plus a ±6 px offset search,
eroded 4 px; thumbnails 1 px. Big icons weigh 1, thumbnails 0.5, per standard (not per pixel).

What the screenshots say, binned by template G (platinum, Unown): G 140 → (71,99,124) · 180 → (116,142,162) ·
200 → (139,162,180) · 220 → (169,187,201) · 240 → (210,221,228). A plain multiply (`G × colour`, Candy-style) does
not fit (the dark end is *more* saturated than MetalColor, the light end lighter than it); per-bin, `mix(M^k, white,
w)` fits every bin within 1.5 levels with `k ≈ MetalPower` at G ≈ 0.5 falling to ≈ 0.4·MetalPower at G ≈ 0.95 and
`w` rising ≈ 0.1 → 0.6. That, plus the config's own `MetalPower`, fixes the model family:

```
g = template G / 255,  b = template B / 255,  u, v = position across the sprite (0..1, v down)
k(g) = max(0, a0 + a1·g)              w(g) = sat(c0 + c1·g)
metal  = mix( MetalColor ^ (MetalPower · k(g)),  white,  w(g) )          (per channel, clamped)
accent = mix( DetailColor ^ k_d(b),            white,  w_d(b) )          (own constants, power 1)
c      = mix(metal, accent, b / (g + b))                                  (which channel owns the pixel)
c     += l0 + lu·u + lv·v                                                 (one directional light over the sprite)
alpha  = template alpha
```

| | a0 | a1 | c0 | c1 |
|---|---|---|---|---|
| metal (shared by all four tiers) | 2.1355 | −1.9514 | 0.0498 | 0.4185 |
| accent | 5.6342 | −4.8300 | 0.1673 | 0.5947 |

light: l0 0.0145 · lu −0.0395 · lv −0.0582 (lighter top-left, darker bottom-right; every reference shows it, and
the residual of a fit without it is a clean linear ramp in u and v).

**Scores (RMSE, 0–255, opaque template pixels):** big icons Unown 11.2 · Tiny Collector 14.6 · Life of the Party
11.9 · Showcase Star 11.3 (JPEG references — 8–10 is the floor); per tier over the fit set bronze 12.9 · silver 9.1
· gold 16.4 · platinum 12.3; accent 19.7. An unconstrained two-stop ramp per tier (dark and light free per tier,
18 numbers instead of 4) does only 0.5–1 better, so the constrained form loses almost nothing. The fitted light is
weaker than the residual suggested (the thumbnails pull it); with big icons only, gold thumbnails go to 26–35, so the
thumbnails are what make gold and bronze transferable at all.

**What this is not.** The real shader's k, w and light are unknown functions; these are the simplest ones that fit.
`MetalSaturation` is 1.0 on every tier, so whatever it does is invisible here. The ring's own template is tinted
with the same maths (no ring standard was aligned — the big medal's ring carries a frame glow and sparkles).
To replace the estimate with the truth: pull `ma_Badge`'s bundle from an Android device's download cache after
opening a medal and read the shader with UnityPy, as `shader-source.md` describes for the Mega materials.

## Composition (measured on nine screenshots, 2026-09-27 — `scripts/measure_layout.py`, `raw/medals/MedalLayouts.json`)

The first version of the lab guessed "icon = 0.60 of the frame"; the user suspected the ratio (and that some PokeMiners
templates are cropped to the art). Measured with the ring's inner edge fitted as a circle (720 rays, ±0.3 px) and the
template aligned to the icon (scale + offset search):

- **Every tier's ring lands at the same size on screen** (outer radius 332.3 px on all nine, silver 333.7 via its own
  inner/outer ratio 0.825 vs 0.809 for gold / platinum): the ring template fills the frame, and the **white disc is the
  ring's own inner radius** (207/253 bronze · 207/256 silver · 202/252 gold, platinum — per tier, not one constant).
- **The icon is drawn at one constant pixel scale**, not fitted to its art: 0.643 screen px per template px for every
  512-class template (Unown, Gym Leader, Pilot, Ultra League, Mega Guru, Tiny Collector) and exactly 2× that for the
  256-px ones (Life of the Party, Showcase Star). So the rule is the game's texture (512 px, or 256 for the newer
  assets) drawn with its **full rect centred on the ring** and its side = **(1 − 2·anchor) × ring outer diameter**,
  anchor 0.25 by default → 0.50 (measured 0.491–0.495), and 0.18 for the config's `StandardBadgeAnchorOverride`
  (Showcase Star) → 0.64 (measured 0.659). This is what makes cropped templates look wrong: Ultra League's PNG is
  462×452 (cropped to the triangle), so drawn at the rule it is smaller than a full 512 sprite and, in-game, its centre
  sits 0.029 of the frame *below* the ring centre (the crop lost the texture's padding). Tiny Collector's PNG keeps its
  512×512 padding and lands exactly.
- **Deviations** (kept as the lab's per-medal defaults, `MedalLayouts.json`): all 512-class medals within 1% of the rule
  and < 1 px off-centre; Ultra League dy +0.029; **Expert Navigator (77) is drawn 15% larger** (×1.147) and 9–12 px
  off-centre — either a smaller game texture or an override newer than 0.429.1's config; Showcase Star ×1.03.
- **Insets** (`Inserts/Badge_Inset_66.png`, `_70.png`): pre-painted overlays (`_InsetTex`) for Rising Star Duo and Mega
  Evolution Guru, drawn **on top of the tinted icon at the same pixel scale, centred** — measured on Mega Guru's
  screenshot at scale 0.985 and 2–3 px offset (colour RMSE 16.7), taken as 1 and 0. No screenshot for Rising Star Duo;
  the same rule is assumed.
- The big in-game medal adds a glow, the sheen and sparkles; the tier thumbnails are the same composition small.

## The route medal (a separate shader, read from the APK)

`NianticCustom/UI/RouteBadgeFrameTriColor` (file `594dbce4…`, pathID 1) with `route_medal_mask_tri_color.png`
(R = rim · B = header band · G = body) as `_FrameMaskTex` and `route_medal_tri_color.png` (plain white) as
`_MainTex`; material `Niantic_RouteBadgeFrame` (pathID 271): `_HeaderColor` (0.5647, 0.349, 0.9843),
`_BlurOutlineColor` (1, 1, 1, 0.502), `_HeaderOpacity` 1.0. Per pixel, m = mask, t = main texture, v = vertex colour:

```
c = lerp(m.a, t.rgb, m.g + m.b)                  -- body / header take the texture, the rest the mask alpha
c = lerp(c, _HeaderColor, m.b · _HeaderOpacity)  -- the header band
c = lerp(c, v.rgb, m.r)                          -- the rim takes the runtime colour (per-route variants)
a = m.a · v.a
```

then a 4-tap alpha blur outline (`_Kernel`, `_BlurOutlineColor`), a stepped-sine reflection band
(`_ReflectionIntensity`²) and an optional directional `_ShineColor` sine. Not in the lab (the per-route rim colours
sit in native code: `RouteBadgeColor`, `badgeFrameColors`, `badgeHeaderColors`).

## Reproducing this

- `raw_data/pogo_apk/extract_badge_config.py` (private checkout, beside `scan_candy.py`): reads pathID 162909, the two
  enums (IL2CPP v31: 88-byte type definitions, zigzag-compressed default values) and PokeMiners' `English.txt`, and
  writes `raw/medals/BadgeMaterialConfig.json` + `names.json`.
- `scripts/fit_medals.py`: the alignment, the fit, the scores, the aligned reference cut-outs
  (`raw/medals/standards/aligned/`, the lab's Compare view) and contact sheets in `mega_outputs/medals/`.
- `scripts/embed_medals.py`: packs templates, rings, config, model and names into `medal_lab.html`
  (`<script id="lab-medals">`; idempotent). `scripts/lab_driver.mjs --global MedalLab` drives the page headless;
  six sampled pixels equal the Python render exactly (2026-09-26).
