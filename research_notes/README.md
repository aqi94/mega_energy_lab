# Research notes — index

This directory preserves what previous sessions learned trying to reverse-engineer the Mega Energy
recolor algorithm (see the mission statement in `../CLAUDE.md`), so future sessions don't redo the
same trial-and-error. **Read only what's relevant to what you're doing** — don't load every file into
context up front; each one below is a self-contained topic.

| File | Read this when you need... |
|---|---|
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

- `mega_lab.html` isn't just a viewer — its **Fitting** tab runs coordinate-descent fits (per-entry
  or gold-weighted) directly in the browser, with sliders for every knob in the current model. You
  can keep tuning the shipped model, or eyeball a new one, without any tooling beyond this repo.
- `raw/standards/{gold,silver,bronze}/` are the reference images every finding here was checked
  against, sorted by trust tier (see `reference-tiers.md`).
- There is no build step or fitting *script* in this repo (those are private tooling) — but there's
  nothing stopping you from writing your own throwaway script against `raw/` to test a hypothesis;
  just record what you learn here afterward.
