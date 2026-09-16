# Mega Energy Color Lab

In Pokémon GO, every Mega Evolution's "Mega Energy" candy is the same gray icon — but it shows up
tinted a different color for every species. Niantic never explained how that coloring works, and
there's no public source code for it. **This project is trying to reverse-engineer the actual
formula**, by testing guesses against real in-game icons until one matches.


## Try it

**[Open the lab](https://aqi94.github.io/mega_energy_lab/)** — no install, no download, just click.
You'll see the current best guess rendered next to the real reference art, plus sliders to tweak the
color math yourself and watch it update live.

(Prefer to run it locally? Download [`mega_lab.html`](mega_lab.html) and double-click it — same tool, works fully offline.)

## Where things stand

Still very much a work in progress. The current model was fitted to make **Venusaur** look right,
and it does — but it doesn't generalize to every other species yet. Some render close, some render
visibly off. Nobody has found one set of math that nails all of them.

If you're curious about the deep-dive details (what's been tried, what failed, what's still an open
question), check the [`research_notes/`](research_notes) folder.
