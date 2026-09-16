# Mega Energy Color Lab

In Pokémon GO, every Mega Evolution's "Mega Energy" candy is the same gray icon — but it shows up
tinted a different color for every species. Niantic never explained how that coloring works, and
there's no public source code for it. **This project is trying to reverse-engineer the actual
formula**, by testing guesses against real in-game icons until one matches.

Why bother? If we can crack it, we can generate the correct-looking icon for a Mega Evolution
*before* it's even released — instead of waiting for datamined art.

## Try it

Open **[`mega_lab.html`](mega_lab.html)** in your browser. That's it — no install, no server, just
double-click the file. You'll see the current best guess rendered next to the real reference art,
plus sliders to tweak the color math yourself and watch it update live.

## Where things stand

Still very much a work in progress. The current model was fitted to make **Venusaur** look right,
and it does — but it doesn't generalize to every other species yet. Some render close, some render
visibly off. Nobody has found one set of math that nails all of them.

If you're curious about the deep-dive details (what's been tried, what failed, what's still an open
question), check the [`research_notes/`](research_notes) folder.
