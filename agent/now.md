# Hand-off — crit-5 (One Stroke) built and tested, not yet finished

**Deliverable:** `comp4020-crit5-shitao`, source `crits/05-game` ("A game").
This run was 165h to cutoff — plan/build/deepen, not the final run. No
reflection written, nothing pushed, per doctrine's finishing-steps gate.

**What this run built, from a placeholder template:**

- **One Stroke**: a single ink brushstroke (pointer, touch via unified
  `pointermove`, or arrow keys) that must keep threading a gap in the next
  approaching wall band while its ink runs dry unless it also catches drifting
  red drops. Either failure (wall or dry ink) ends the run; a fresh one starts
  1.3s later. Opening screen shows only the still brush and the first
  already-approaching wall — no on-screen or off-screen instructions anywhere,
  satisfying the brief's affordance requirement.
- Two interacting mechanics (dodge + resource management), per the brief's
  "harder, better move" framing — see the tuning fix below for why this
  needed a second pass to actually interact.
- `game-logic.ts`: pure simulation (no canvas/DOM/`Math.random`/`Date.now`),
  so `vitest`'s jsdom (no real canvas backend) can test the rules directly.
  Deterministic gap/drop patterns via sine functions keyed to spawn index.
- `spec/game.test.ts`: table-driven test on `checkWallCollision` (the one
  rule the brief requires a focused test for) plus lifecycle tests. 28 tests
  total across the suite, all passing.
- `main.ts`: full rewrite from the placeholder `export {}` — canvas rendering
  reading pure state, devicePixelRatio-aware, ink-meter/trail/score drawing.
- `index.html`/`styles.css`: full-bleed canvas game layout, title "One
  Stroke", meta description rewritten, single visually-hidden `h1` (keeps the
  invariants test green without putting visible instruction text on screen).
- `public/card.png`: replaced the still-unmodified template placeholder
  (confirmed via `identify` at session start) with a generated card matching
  the game's own PAPER/INK/SEAL palette and wall geometry — built via a
  scratch `/tmp/one-stroke-card.html` replicating the real `game-logic.ts`
  constants, rendered through `agent-browser` at 1200×630, iterated once to
  fix a title/wall-band text collision, installed and committed (`f50e08f`).
- `PROCESS.md`: two real, citation-checked moments — the pure-simulation
  architecture (`21de962`) and a genuine playtest-driven balance fix
  (`cd95873`, see below).

**The play-driven fix (satisfies "one change came from playing, not
reading"):** original `inkDecay`/`inkPerDrop` (0.045/0.3) mathematically
implied real pressure, but sustained dodge-focused play — read via sampling
the ink-meter's own canvas pixels through `agent-browser eval`, no debug
hooks added to source — showed ink sitting at 60-90% throughout, because the
drop and gap y-patterns wander through similar ranges, so tracking gaps well
incidentally collects most drops. The second mechanic never actually
threatened a competent player. Retuned to `inkDecay: 0.09`, `inkPerDrop: 0.2`;
replaying the identical sequence showed a real downward trend and a visibly
thinning brush. All 28 tests still pass (they test rules, not tuning
constants) — this is exactly the kind of finding static analysis alone
wouldn't have surfaced, and it's why the standing "screenshot before
believing the checks" habit ([[verify-real-browser-not-just-checks]] if that
memory exists, else see MEMORY.md's browser-verification section) extends to
"play before believing the config."

**Verification done this run:** `pnpm check` (typecheck+build+28 tests)
green; `pnpm check:evidence` fails only on the expected/correct missing
`reflections/crit-5.md` (a finishing-steps item, not due yet) — both commit
citations in `PROCESS.md` resolve. Full browser pass via `pnpm exec vite
preview` + `agent-browser`: desktop (1280×800) and phone portrait (390×844,
fresh reload), pointer-driven brush movement confirmed responding correctly
at the narrow viewport, wall/gap geometry scales correctly by aspect ratio
(gap and wall thickness are fractions of canvas height/width, not fixed
pixels — no Chime-style aspect-ratio tuning bug found this time), console
clean on both. Preview server confirmed killed via `lsof -ti:<port>` (not
just `pkill`'s exit code, per the standing gotcha).

**State at end of run:** 5 commits ahead of `origin/main`
(`21de962`..`f50e08f`), working tree clean, nothing pushed — correct for a
non-final run at 165h to cutoff.

**Most important next action:** this deliverable is content-complete but not
finished. On a future run of `comp4020-crit5-shitao` (especially the one the
prompt names as final): re-fetch the source in case the brief changed, then
do the finishing steps — write `reflections/crit-5.md` (title "A game", both
standing prompts; the playtest-fix story above is the natural breakthrough
answer), a final full-viewport browser pass, `git status` clean, push. Before
trusting "content-complete, nothing to fix" on a later run without
re-opening the browser, see MEMORY.md's now-five-times-confirmed rule: green
checks are not sufficient evidence the rendered/played game is fine at every
viewport or under real play.
