# Hand-off — crit-5 (One Stroke): found and fixed a real bug via a fresh angle

**Deliverable:** `comp4020-crit5-shitao`, source `crits/05-game` ("A game").
This run was 100h to cutoff — plan/build/deepen, not the final run. Brief
re-fetched, byte-identical to every prior run.

**State found at start of run:** `main` at `b3e2779` (a memory tick-snapshot;
no real commits since the prior run's `285e5e0`/`5d7d4ab`). Working tree
clean, `pnpm check` green (28 tests). Prior run (111h to cutoff) had already
re-verified both marking viewports and tried (inconclusively) a sustained-play
fairness question; five solid PROCESS.md moments already existed.

**What this run did:** rather than repeat the same viewport/resize
verification pass so soon after the last one, read `game-logic.ts`/`main.ts`
end to end looking for something genuinely new. Found it: `keydown`/`keyup`
toggle a single `keyDir` flag, but a key held when the window loses focus
(alt-tab, a notification, clicking another app) never gets its `keyup` — the
browser just stops delivering events. `keyDir` stayed stuck non-zero forever,
and the frame loop's keyboard branch ran every frame whenever `keyDir !== 0`,
silently overriding pointer control too (not just keyboard) for any player
who touched a key once and switched back to the mouse.

Confirmed with a proper A/B in an isolated `agent-browser --session
crit5-shitao-run`: dispatch a synthetic `keydown`, then a synthetic `blur`,
then move the mouse to the opposite edge of the canvas. Unfixed code: brush
stayed pinned wherever the stuck key had driven it, ignoring the mouse
entirely. Fixed code (a `window.addEventListener("blur", () => keyDir = 0)`):
brush followed the mouse immediately. Did the A/B properly — `git stash` to
get the pre-fix code, rebuilt, re-ran the identical repro, `git stash pop` to
restore — rather than trusting the fix by inspection alone. Also re-checked
both marking viewports (1920x1080, 390x844) after the change: pixel-fine,
console clean. Preview server shut down and confirmed free
(`lsof -ti:4715` empty) afterward.

**Commits this run:** `22122e0` (the fix), `cf5fb6c` (PROCESS.md, moment 6).
Not pushed — inside-24h gate hasn't opened yet (100h to cutoff).

**State at end of run:** `main` at `cf5fb6c` locally (2 commits ahead of
`origin/main`), working tree clean, `pnpm check` green.

**Most important next action:** nothing further needs building unless a
future run's own pass turns up something new. The game now has six solid
PROCESS.md moments — plenty for the finishing-run writeup. On the run the
prompt calls final: write `reflections/crit-5.md` (title "A game", 150–300
words, both standing prompts). Strongest breakthrough candidates, in order:
(1) the resize-feedback-loop bug (`285e5e0`) — the deepest, most surprising
CSS mechanism found all deliverable; (2) this run's stuck-`keyDir`-on-blur
bug — found by reading the source fresh rather than re-running the same
verification pass, and confirmed with a disciplined A/B rather than trusting
the fix by inspection. Either answers "what changed about the developer you
want to be" well: read the code for genuinely new angles rather than
re-verifying the same ground, and don't trust a one-sided "it looks fixed"
without reproducing the bug first.
