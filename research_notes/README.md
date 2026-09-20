# Research notes — index

This directory preserves what previous sessions learned trying to reverse-engineer the Mega Energy
recolor algorithm (see the mission statement in `../CLAUDE.md`), so future sessions don't redo the
same trial-and-error. **Read only what's relevant to what you're doing** — don't load every file into
context up front; each one below is a self-contained topic.

| File | Read this when you need... |
|---|---|
| [`shader-source.md`](shader-source.md) | **Read first.** The real shader, recovered from the APK on 2026-09-20: its properties, the exact per-pixel maths, the generic icon's material values, what it overturns in the fitted notes below, and the one input still missing (per-species float values). Where it disagrees with another file, it wins. |
| [`inputs.md`](inputs.md) | What the template's R/G/B channels and each JSON color actually mean. Start here — these are the most load-bearing, highest-confidence facts. |
| [`reference-tiers.md`](reference-tiers.md) | How to judge whether a candidate model is actually improving, and why unweighted error metrics mislead you here. |
| [`models-tried.md`](models-tried.md) | The current best-known formula, its accuracy ceiling, and the lineage of approaches that got there — so you don't re-invent or re-reject the same model. |
| [`findings-timeline.md`](findings-timeline.md) | The numbered, dated log of specific discoveries (with numbers/evidence) behind the summaries in the other files. Skim for a topic; don't read start to finish unless you're doing a deep dive. |
| [`open-questions.md`](open-questions.md) | What's still unresolved and roughly what's been tried against each — the actual frontier of this work. |

## How this was produced

These notes were distilled from a much larger, private research log (hundreds of one-off fitting
scripts and experiment images) that lives outside this repo, in the PoGoRankIV monorepo this was
published from. That raw experiment trail is not included here — these files are the durable
conclusions, not the process. If you continue this work and reach a new durable conclusion, add it
here (append to `findings-timeline.md`, and update `models-tried.md` / `open-questions.md` /
`../CLAUDE.md`'s universal rules if it changes either) rather than only leaving it in your own
session's memory.

## What's actually available to keep working with

- `mega_lab.html` runs the **real shader** (since 2026-09-20). Its only controls are the nine unknown
  material floats, kept per entry; its **Fitting** tab runs coordinate descent over those floats (this
  entry, one shared set, or every reference separately) directly in the browser. Colours and shader
  constants are fixed on purpose. `models-tried.md` describes the *earlier fitted* models — history now.
- `raw/MegaCandyMaterialFloats.json` holds the **estimated** nine floats for all 96 entries (60 fitted, 36 inferred), a
  per-float constants verdict and the caveats (finding #24). `window.MegaLab` in the lab is a small automation API
  (`score` / `error` / `descend` — the lab's own shader); a throwaway headless driver can call it, but never copy the shader.
- `raw/standards/{gold,silver,bronze}/` are the reference images every finding here was checked
  against, sorted by trust tier (see `reference-tiers.md`).
- There is no build step or fitting *script* in this repo (those are private tooling) — but there's
  nothing stopping you from writing your own throwaway script against `raw/` to test a hypothesis;
  just record what you learn here afterward.
