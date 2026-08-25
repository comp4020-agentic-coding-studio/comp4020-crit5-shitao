# Hand-off — crit-5 (One Stroke): found and fixed a real resize bug

**Deliverable:** `comp4020-crit5-shitao`, source `crits/05-game` ("A game").
This run was 124h to cutoff — plan/build/deepen, not the final run. Brief
re-fetched, byte-identical to every prior run.

**State found at start of run:** `main` at `c4c7dab` (prior hand-off's work
plus harness tick-snapshot commits, already pushed out-of-band — normal).
Working tree clean, `pnpm check` green (28 tests). Prior hand-off had just
done an exhaustive dual-viewport playtest and found nothing — its own
framing was "a clean pass with no diff is a legitimate outcome," so this run
didn't repeat that same pass by default.

**What this run did:** ran `pnpm check` (green), then a lighter real-browser
spot check — but instead of stopping there, tried viewport sizes *between*
the two marking viewports (1280x577 default headless, 1280x720, 1024x600,
1366x768 — realistic laptop-with-browser-chrome sizes) and a *sequence* of
resizes in one session, not just isolated `open`s. That surfaced a genuine
bug neither marking viewport (1920x1080, 390x844) ever exposed: the
canvas's CSS height relied on flexbox stretch, and a canvas's own
`width`/`height` attributes give it an intrinsic aspect ratio that
overrides stretch once computed height goes "auto". `resizeCanvas()` read
the canvas's own rendered size back to decide its next bitmap size, so each
window `resize` event fed the previous, ratio-skewed height into the next
measurement — a feedback loop. A short sequence of `agent-browser` viewport
changes grew the canvas from 577px to 2976px tall, no console error, pushing
the ink meter and best-score readout off-screen. Root-caused via
`getComputedStyle`/`getBoundingClientRect` inspection (not guessing), tried
two intermediate fixes that each solved part of it (`height:100%` on the
flex item stopped divergence but left a ~100-160px residual; absolute
positioning with `inset` reintroduced the exact same divergent bug, because
CSS2.1 still derives auto height from the intrinsic ratio for absolutely
positioned *replaced* elements when top+bottom are both set and height is
auto — inset alone doesn't make height "definite"). Final fix: measure from
`main` (a plain block, no intrinsic ratio) and set the canvas's CSS box in
explicit pixels directly from JS, so the canvas's own attributes can never
re-enter the layout. Verified with 9 consecutive resizes in one session
(577→700→500→1080→844→720→600→768→577, all exact) plus fresh loads at 7
sizes (all exact), then a real play pass at both marking viewports (pointer
movement, a deliberate wall death, best-score persistence) with zero
console errors. `pnpm check` green after. Committed as `285e5e0` (the fix)
and `5d7d4ab` (PROCESS.md entry #5). Not pushed — inside-24h gate not yet
reached.

**State at end of run:** `main` at `5d7d4ab`, two commits ahead of
`origin/main`, working tree clean. Preview server (port 4711) shut down and
verified free (`lsof -ti:4711` empty) before finishing.

**Most important next action:** on a future run — especially the one the
prompt names as final — re-fetch the source once more in case the brief
changed, then do the finishing steps: write `reflections/crit-5.md` (title
"A game", both standing prompts). This run's resize-feedback-loop bug is now
the strongest breakthrough candidate for the reflection (better than the
localStorage-guard story two runs back): it's the clearest case yet of "a
check that passes at the declared marking viewports is not evidence the
layout is robust at every size a real user might resize to," found only by
testing *between* and *across* the declared viewports rather than just at
them. Final run: full-viewport browser pass (already fresh from this run,
but re-confirm if code changes further), `git status` clean, push.
