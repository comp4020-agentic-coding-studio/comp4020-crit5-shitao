# Hand-off — crit-5 (One Stroke): no code change, a tooling hazard found instead

**Deliverable:** `comp4020-crit5-shitao`, source `crits/05-game` ("A game").
This run was 111h to cutoff — plan/build/deepen, not the final run. Brief
re-fetched, byte-identical to every prior run.

**State found at start of run:** `main` at `cef7a70` (a memory tick-snapshot;
no real commits since the prior run's `32ee13d`). Working tree clean,
`pnpm check` green (28 tests). Prior run (117h to cutoff) had already done a
thorough verification pass (both marking viewports, a resize sequence, a real
play session) and found nothing — game is content-complete against the spec,
five solid cited PROCESS.md moments.

**What this run did:** since a repeat of the exact same verification pass so
soon after would be low value, tried a genuinely new angle: whether the
speed-ramp (`speedMultiplier` in `game-logic.ts`, maxing at 2.2x by distance
24) stays fair deep into a sustained run, not just in the opening seconds.
Two automated approaches both turned out unreliable, for different reasons —
full detail in `MEMORY.md`'s tooling-gotchas section:

1. An open-loop script (precomputed wall-arrival times from a standalone
   reimplementation of the sim's own math, scheduling ~105s of `mouse move`
   calls) died partway through — inconclusive, not a bug: real per-call
   latency drift compounds over 50+ scheduled events with no visual
   correction, unlike a real player.
2. A closed-loop script (live pixel-scanning the canvas via `getImageData`
   to steer reactively) did *worse* and never beat the same best distance —
   traced to a real infrastructure problem, not a script bug in the end:
   `agent-browser`'s "default" session on this host is a single shared tab,
   and something else (another concurrent agent, most likely) navigated it
   to two unrelated students' GitHub Pages sites mid-test. Confirmed via
   `agent-browser session info --json` (`pageCount: 1`) and `eval
   "location.href"` returning `comp4020-crit4-Gera1t-2001`/`comp4020-crit4-
   kyle-zjy` instead of the localhost preview.

Re-verified the game itself is fine with a short, low-risk check in an
explicitly isolated `--session crit5-shitao-run`: both marking viewports
(1920x1080, 390x844) render pixel-perfect, console clean, preview server
shut down and confirmed free (`lsof -ti:4715` empty) afterward. Did not
re-attempt the sustained-play fairness question this run — no trustworthy
answer either way, and chasing it further on a contended shared browser
felt like exactly the kind of over-engineered test scaffolding the doctrine
warns against. It's a nice-to-have depth question, not a spec requirement.

**State at end of run:** `main` unchanged at `cef7a70` (well, whatever the
tick-snapshot bot has pushed by the time this is read), working tree clean.
No commits this run — legitimate outcome, not a failure (third time this
framing has held for this repo).

**Most important next action:** nothing further needs building unless a
future run's own browser pass turns up something new. If a future run wants
to revisit the speed-ramp fairness question, use `--session <name>` on every
`agent-browser` call from the start (not just after something looks wrong),
and treat sustained (~100s+) automated play as inherently less trustworthy
than the short spot-checks already proven reliable here. On the run the
prompt calls final: write `reflections/crit-5.md` (title "A game", 150–300
words, both standing prompts) — the resize-feedback-loop bug from a few runs
back is still the strongest breakthrough candidate. This run's shared-session
discovery is a decent second-place candidate if a different angle is wanted:
it's a case of catching an automated check lying to you (screenshots looked
right, eval didn't) rather than the app being wrong.
