# MEMORY

Durable self-knowledge, curated run by run; ephemeral state belongs in
`now.md`, not here.

## Tooling gotchas worth not re-discovering

- **`vite preview`'s default port isn't guaranteed, and `agent-browser open`
  won't tell you if you guessed wrong.** Started `vite preview --port 4321`
  (crit-5, run at 148h to cutoff) while something else on the host already
  had 4321 bound (an unrelated open tab serving the course website); vite
  logged "Port 4321 is in use, trying another one" and actually bound 4322,
  but `agent-browser open http://localhost:4321` still succeeded with no
  error, rendering that other site. The tell was the page title in the
  `open` command's own output ("Home — COMP4020 Agentic Coding Studio"
  instead of the game's title), not any error or empty screenshot. Read the
  preview server's own log line for the port it actually bound, or check the
  title/take a screenshot immediately after `open`, rather than assuming the
  `--port` you requested is the one you got.
- **`agent-browser` viewport**: it's `agent-browser set viewport <w> <h>` as
  its own command, not a `--viewport` flag on `open`. Passing it to `open`
  fails silently-ish (open still succeeds) and screenshots come back at the
  default desktop width — checked dimensions, not just eyeballed, is what
  caught this the one time it happened.
- **`mise`**: a fresh environment's global `~/.config/mise/config.local.toml`
  may need `mise trust <path>` before `pnpm`/other shims work. This is a
  trust operation on the user's own pre-existing config, not a content edit —
  safe to run without asking.
- **stylelint's `no-descending-specificity`**: attribute-selector compounds
  (`nav[aria-label="Primary"] a`) don't have a stable specificity order
  against plain class or tag selectors, so reordering CSS rules to fix a
  violation just moves it elsewhere. The real fix is structural: add an
  explicit class to the element and select on that instead of the attribute
  compound. Reconfirmed in a plainer shape (crit-4, `Chime`): a bare two-tag
  compound (`header h1`) before a later plain `h1` rule tripped the same
  check, no attribute selector involved — the trigger is any compound with a
  different specificity than a plain-selector rule touching the same element,
  not specifically attribute compounds. Same fix: an explicit class (`.brand`)
  on the element, selector on the class, drop the compound entirely.
- **Museum/gallery sites and bot-blocking**: metmuseum.org returns HTTP 429
  with a Vercel challenge header to *any* automated request, browser
  User-Agent or not, across its whole domain — not a per-URL quirk. If a
  future week links to museum collection pages, expect this class of
  problem and check with `curl -I` before assuming a reformatted URL will
  fix a links-check failure. Wikimedia Commons and Wikipedia have not shown
  this behaviour.
- **`agent-browser` in a fresh sandbox**: Chrome isn't preinstalled, and even
  after `agent-browser install` a bare `open` can fail with "Chrome exited
  before providing DevTools URL" / zygote sandbox errors. `--args` is a
  *global* option, not a per-subcommand one: `agent-browser open <url> --args
  "--no-sandbox"` (flag after the subcommand) fails silently back into the same
  sandbox error, whether quoted with a space or `=`. What actually works is
  `agent-browser --args "--no-sandbox" open <url>` (flag before the
  subcommand) — confirmed again this run (run 9, 52h to cutoff) after a prior
  run's note claimed the after-subcommand form worked "first try," which this
  run couldn't reproduce. Reconfirmed a third time (39h to cutoff): still the
  only form that launches Chrome in this sandbox, so treat it as settled
  rather than re-testing the after-subcommand form again.
- **`agent-browser` has no keyboard-hold primitive.** `press <key>` fires a
  full keydown+keyup instantly — there's no `keydown`/`keyup` pair exposed
  separately, so you can't drive a "hold this key for N seconds" gesture
  through the normal command surface. Workaround that worked (crit-4, run at
  148h to cutoff, testing a hold-duration swell effect on the keyboard path):
  `agent-browser eval "window.dispatchEvent(new KeyboardEvent('keydown',
  {key: 'j', bubbles: true}))"`, then `sleep`, screenshot, then the matching
  `keyup` dispatch. This exercises the app's real `window.addEventListener`
  listeners (since the app itself doesn't distinguish a dispatched
  `KeyboardEvent` from a native one), so it's a legitimate test of the
  keyboard hold path, not a synthetic shortcut around it — just document
  which command produced it, since it's a step further from "real user
  input" than `mouse down`/`mouse up` are.
- **`agent-browser`/CDP cannot simulate multi-touch against code that calls
  `setPointerCapture`.** Tried dispatching two synthetic `PointerEvent`s with
  distinct `pointerId`s via `eval` (crit-4, `Chime`, 76h to cutoff) to verify
  a two-finger-touch gesture the app's real pointer handling never branches
  on `pointerType` for. No visible effect — traced it by monkey-patching
  `canvas.setPointerCapture` inside the same `eval` to log instead of throw:
  it throws `NotFoundError: No active pointer with the given id is found`
  for any JS-constructed `PointerEvent`, since Chrome's pointer-capture
  implementation checks its own hardware-backed active-pointer state, not
  properties on the event object. The throw happens *inside* the app's
  `pointerdown` listener, before any of the app's own logic runs — confirmed
  a plain listener with no capture call, added in the same `eval`, sees the
  synthetic event fine, so the block is specifically capture APIs, not event
  dispatch generally. This is not an app bug (real hardware pointerdown
  events always carry an id Chrome has live state for, so capture never
  throws for a genuine user) — it's a hard ceiling on what synthetic
  dispatch can test. Unlike the keyboard-hold trick above (keydown/keyup
  have no equivalent capture requirement), don't spend a future run trying
  variations of this for any app using `setPointerCapture`/similar capture
  APIs — fall back to the structural argument (code doesn't branch on
  `pointerType`, state keyed uniformly across input sources) plus whatever
  single-pointer/keyboard-chord gestures *are* directly testable.
- **After `pkill -f <process name>`, don't trust the exit code — check the
  port.** Killing a `vite preview` background server with `pkill -f "vite
  preview"` reported a non-zero/odd exit code, which looked like "no matching
  process, already dead" — but a `curl -sI` against the port it had been
  serving still returned `200 OK` immediately after (crit-4, 148h to
  cutoff). The reliable check is `lsof -ti:<port> | xargs -r kill` followed
  by a re-`curl` (or a bare `lsof -ti:<port>` returning nothing) — `pkill`'s
  reported outcome and the process's actual state can disagree, so verify
  the port is actually free before considering a preview/dev server shut
  down, per the doctrine's "shut down servers afterwards" step.
- **`vitest`'s jsdom has no real `<canvas>` backend**: `canvas.getContext("2d")`
  returns `null` there (not a stub with no-op methods), so any spec test that
  dispatches a pointer/draw event against canvas-based interactivity will
  throw unless the drawing code itself is guarded. Pattern that worked
  (assignment-1, the yihua ink-brush prototype): wrap every draw call in
  `if (ctx) { ... }` so the *behavioural* DOM state (a counter, a live status
  region) still updates and is testable in jsdom even though the ink itself
  never renders there. Keeps the canvas interaction from being untested by
  default just because the obvious jsdom check would crash.

- **`check-evidence.ts`'s commit-citation regex only matches hex-shaped link
  text.** `PROCESS.md` citations are parsed as
  `` [`sha`](url) `` where the link *text* must look like a SHA
  (`/[0-9a-f]{7,40}/`, or a `sha...sha` range) --- a citation written as
  `` [`spec/foo.test.ts`](...) `` silently doesn't count as a citation at all,
  and the whole check fails with "no commit citations found" even though a
  link is right there. Consequence: you cannot cite a commit's content before
  that commit exists. Commit first, then edit `PROCESS.md` to cite the real
  SHA in a follow-up commit --- cite-then-commit doesn't work, it has to be
  commit-then-cite.
- **A `cd` inside a compound/backgrounded Bash command changes cwd for every
  later command in the session**, since the Bash tool persists cwd across
  calls but has no per-command scoping. `cd dist && python3 -m http.server
  ... &` left the shell sitting in `dist/` afterward; subsequent `git status`
  silently showed `../PROCESS.md`-style relative paths instead of erroring.
  Caught by noticing the path shape look wrong, not by any command failing.
  Use a subshell (`(cd dist && ...)`) or `cd` back explicitly right after,
  never rely on a background job's `cd` staying scoped to that job.
- **`axe-core` snapshots `window`/`document` from `globalThis` at *import*
  time, not at call time** — and ESM hoists static `import` statements ahead
  of every other top-level statement in the module. So building your own
  jsdom instance and doing `Object.assign(globalThis, { window, document })`
  *after* a static `import axe from "axe-core"` is already too late: `axe.run()`
  fails claiming the globals aren't set, even though they plainly are by the
  time the call executes. Fix is a dynamic `await import("axe-core")` placed
  *after* the globals are assigned (confirmed the ordering at a bare Node
  REPL before trusting it: same globals, dynamic import after works, static
  import before doesn't). Found wiring axe-core into a vitest spec test that
  builds its own jsdom rather than using vitest's `environment: "jsdom"`
  (assignment-1, `spec/axe.test.ts`) — watch for the same shape if a future
  week reaches for axe-core again.
- **`agent-browser scroll`'s first argument is a direction keyword
  (`up`/`down`/`left`/`right`), not a pixel offset.** `agent-browser scroll 0
  500` parses `0` as an (invalid-but-silently-accepted?) direction and the
  page doesn't move — no error, just a screenshot identical to before the
  call, which is what gave it away. Correct form is `agent-browser scroll
  down 500` (direction first, then the pixel amount); `--help` on the
  subcommand spells this out and is worth checking before guessing
  positional-argument order on any `agent-browser` subcommand.
- **`agent-browser find`'s `--name` filter must come after the action, not
  between the locator value and the action.** `find role button --name "X"
  click` errors ("Unknown action '--name'"), because the parser reads
  positionally (`find <locator> <value> [action] [text]`) and treats
  anything after the value as the action slot until it sees a flag it
  recognises in that position; `find role button click --name "X"` is the
  form that works. Found while clicking a named "Clear canvas" button
  during an assignment-1 interaction pass (run 8).
- **`agent-browser` has no bandwidth-throttle command** — checked its full
  `--help` and the `skills get core --full` reference (assignment-1, run 5)
  looking for a way to test the artefact-criterion HD language ("holds up
  under... a slow connection"). It has `network route <url> --abort` (asset
  never arrives) and `set offline on` (always offline), but nothing between
  those and full speed — no CDP `emulateNetworkConditions` equivalent
  exposed. The honest substitute: route-abort `**/*.js` and `**/*.css`
  independently and combined against the built site, which bounds the worst
  case (assets that never arrive) even though it can't show a genuinely slow
  *trickle*. Don't spend a future run hunting for a throttle flag that isn't
  there — reach for route-abort combinations instead.
- **`agent-browser get box <sel>`'s coordinates aren't clipped to what's
  actually visible.** It returns the element's full bounding rect regardless
  of `window.innerHeight`, so a canvas whose box reports e.g. `y:470
  height:260` can have its bottom half (y > 577 in a 1280×577 headless
  window) sitting below the fold — `document.elementFromPoint` at a point
  inside the reported box returns `null` there, and `mouse move`/`down` to
  that point dispatch *zero* events, no error, nothing in `console`/`errors`.
  This produced a convincing false negative (assignment-1, 52h to cutoff):
  a real bug looked unreproducible in the live browser for several attempts
  before the actual cause (my test point, not the app) turned up by
  instrumenting the target element with a temporary listener that logged
  every event it received — an empty log at a point *inside* the reported
  box is the tell, not a `console`/`errors` check, since nothing throws.
  Pick interaction coordinates from `window.innerHeight`/`innerWidth`
  (checked via `eval`), not straight from `get box`, whenever the element is
  tall relative to the viewport.
- **Generating an `og:image` link-preview card from an app's own visual
  language, not a separately-designed banner.** When a course-checks update
  (crit-4, 135h to cutoff) added a presence-only `og:image` invariant and
  shipped the template's literal placeholder PNG, the fix was a scratch HTML
  file (`/tmp/...`, not committed) that re-implements the *actual* app's
  layout/draw formulas (copy the real constants/math from the source, not an
  approximation) inside a plain `<canvas>` at the exact card dimensions
  (1200×630), rendered via `agent-browser`: `agent-browser set viewport 1200
  630` then `agent-browser --args "--no-sandbox" open file:///tmp/.../card.html`
  (sandbox-launch and viewport gotchas both apply here too — see their own
  entries above), then `agent-browser screenshot <path>.png`. `identify
  <path>.png` confirms the exact pixel dimensions before installing it as
  `public/card.png`. Cheap and reusable for any future week whose brief wants
  a link-preview card: read the app's real rendering code first, replay it
  standalone, don't design a new image from scratch.

- **A multi-line Bash tool call with a nested `python3 -c` inside a shell
  function silently drops its computed values.** Tried computing pixel
  coordinates in a bash function (`mvy() { x=$(python3 -c "...")); agent-browser
  mouse move $x $y; }`) inside one multi-line Bash call (crit-5, "One Stroke"
  playtest); `agent-browser` errored "Missing arguments for: mouse move" as if
  `$x`/`$y` were empty, no python error surfaced. The `python3 -c` itself
  worked fine both in isolation (`bash -c`) and as a simpler body via the Bash
  tool directly — the failure was specific to nesting that quoting pattern
  inside a multi-line block passed as one tool-call string, not the shell or
  python. Workaround: skip the coordinate-computing subshell and hardcode the
  pixel integers directly in the `mouse move` call. If a future run wants
  computed coordinates, compute them in a separate prior Bash call and pass
  the resulting numbers as literals into the `agent-browser` call, rather than
  nesting the computation inline.
- **Reading game/app state off the canvas's own rendered pixels
  (`agent-browser eval` + `getImageData`), instead of adding debug hooks to
  source, is a reusable non-invasive verification technique.** Used to check
  an ink-meter's actual fill percentage during closed-loop playtesting
  (crit-5): the meter bar's fill fraction is directly readable as "how far
  right the fill colour extends before hitting the background colour," sampled
  via `ctx.getImageData` inside an `eval` string, with no change to `main.ts`.
  Generalises to any bar/meter/progress UI drawn to canvas — cheaper and more
  trustworthy than eyeballing a screenshot, and doesn't risk the debug hook
  itself being the thing that changes behaviour.
- **Closed-loop playtesting a scrolling/timed obstacle course needs "hold
  position until the obstacle has actually passed," not "move to the next
  target early."** Moved from one gap's x/y position to the next too early
  during a crit-5 playtest (before the first wall's x-column had reached the
  brush), causing an avoidable death that looked like a game bug at first.
  Diagnosed by working out the wall's actual arrival time from the sim's own
  constants (`scrollSpeed`, `wallSpacing`) rather than guessing from the
  screenshot cadence. When scripting a playtest sequence against a
  scroll-speed-based game, compute (or at least sanity-check) the time each
  obstacle needs to reach the player's fixed x-column before writing the
  `sleep`/`mouse move` sequence, rather than pacing moves by wall-clock feel.
- **"Content-complete... green checks" isn't sufficient evidence a *game's
  balance* is right, either — not just its layout.** Sixth confirmation of the
  standing rule below, but a new bug shape: crit-5's ink-mechanic constants
  read as real pressure from the numbers alone (22s to dry out at zero drops)
  and every automated test passed (they test the rules, not the tuning), but
  sustained real play showed the second mechanic never actually threatened a
  skilled player, because the drop and gap patterns correlate more than the
  numbers alone suggest. No screenshot or check catches this class of gap —
  only playing the actual game with a real strategy for a sustained stretch
  does. Worth budgeting one genuine playtest pass (not just an idle-screenshot
  pass) for any future week whose brief includes tunable difficulty/balance,
  same way a real-browser screenshot pass is already budgeted for layout.
- **`agent-browser set viewport` on an existing session doesn't reset page
  state.** Switching from a desktop viewport to a portrait one after already
  interacting with the page (crit-4, run at 100h to cutoff) carried over
  dismissed-invite/other in-page state from the desktop pass, briefly looking
  like a missing-invite bug at the new viewport before a proper `open` reload
  showed the correct fresh-load behaviour. When a browser pass is checking
  "what a stranger sees on first contact" at a second viewport, `open` the
  page again after setting the viewport, don't just resize an already-primed
  session.

## Working habits that paid off

- **For pointer/drag-driven interactions, simulate the real gesture, not
  just the resulting DOM state.** `agent-browser get box <sel>` returns
  element coordinates; `agent-browser mouse move <x> <y>`, `mouse down`,
  more `mouse move`s with a `sleep` between them to control simulated
  speed, then `mouse up` reproduces an actual slow or fast drag through
  real `pointermove` events — this is what caught (assignment-1) that a
  canvas ink-brush's speed-based width/opacity actually worked, not just
  that the canvas element existed. `agent-browser press ArrowLeft/Right`
  after `focus`ing a control does the equivalent for a keyboard-only path.
  A static screenshot of an untouched canvas proves nothing about whether
  the interaction itself works.
- **Screenshot before believing the checks.** All automated checks (build,
  lint, 51 tests) were green while a real rendering bug (unreadable banner
  text over a striped background) shipped anyway. Actually opening the page
  in `agent-browser` and looking at a screenshot at both required viewports
  (1920×1080, 390×844) is what caught it — this is not optional polish, it's
  the only check that catches this class of bug. Do this before considering
  a week "verified," not as an afterthought. Confirmed a second, different
  time in run 12 (28h to cutoff): after ten runs of "content-complete, nothing
  found," a phone-viewport screenshot of the about page caught the self-portrait
  `<img width="400">` overflowing its container horizontally — none of
  typecheck/build/lint/51 tests/evidence check saw it, because none of them
  render at a narrow viewport. Fixed with a global `img { max-width: 100%;
  height: auto; }` rule (styles.css), since only `.gallery img` had been made
  responsive and the about page's figure image hadn't. Lesson generalises:
  any raw `width="..."` HTML attribute on an `<img>` is a horizontal-overflow
  risk on mobile unless something constrains it — worth a quick eyeball at
  390×844 specifically, not just desktop, whenever a page adds an image.
- **Small, scoped commits over one big one.** Committed the spec test, the
  link fix, and the CSS fix as three separate commits rather than folding
  them into the original build commit — made each one legible on its own in
  `git log`, and made the `PROCESS.md` citations point at something a reader
  could actually verify in isolation.
- **"Resize mid-interaction" (the artefact-criterion HD language) means
  literally resizing while an interaction is in flight, not just checking
  both viewports separately.** Pattern that worked (assignment-1): `mouse
  down`, `mouse move`, `set viewport <w> <h>` *while the button is still
  down*, more `mouse move`, then `mouse up` — reproduces a real mid-drag
  resize through the same events a user dragging while rotating a device or
  resizing a window would generate. Caught (harmlessly, in this case) that a
  canvas resize handler using `getImageData`/`putImageData` does a raw pixel
  copy, not a proportional rescale — no crash either direction, but worth
  knowing before trusting the function's name. Checked via `agent-browser
  errors`/`console` (not just a screenshot) immediately after, since a
  silent JS exception wouldn't necessarily show up visually.
- **A "manual-only check" callout in your own `CLAUDE.md` is a punch-list
  item, not a permanent disclaimer.** This repo's `CLAUDE.md` said "nothing
  here measures... the contrast half of accessibility" ever since the
  axe-core moment disabled `color-contrast` for lack of a jsdom paint
  engine. But WCAG contrast is a pure function of two hex colours and
  doesn't need one: `spec/contrast.test.ts` (assignment-1) reads the real
  `:root` palette straight out of `styles.css` with a regex, rather than
  hardcoding a duplicate copy of the colours, so a future palette edit is
  caught automatically instead of the test silently going stale (the same
  failure mode as the reflection/README drift noted below). Verified it
  wasn't a rubber stamp the same way as the axe-core test: temporarily
  weakened one CSS variable, confirmed real failures naming the actual
  computed ratio, then restored the file. Worth reflexively rereading your
  own `CLAUDE.md`'s "not measured/not covered" language every so often —
  each one is a named gap, and closing it is exactly the kind of
  harness-level correction the process-legibility criterion rewards.
- **Grep every call site of a boolean/flag parameter before trusting that
  its "true" branch is reachable.** Found a real bug this way (assignment-1,
  52h to cutoff): `finishStroke(points, pooled)` had a whole message
  ("saturated and pooling") gated on `pooled`, and `grep -n "pooled"
  main.ts` showed every call site passing a hard-coded `false` — the branch
  was provably dead from the source alone, before touching a browser at
  all. This is a cheaper and more reliable first move than trying to drive
  the UI into the state and see if the message appears, which (per the
  `get box` viewport-clipping gotcha above) can give a false negative for
  reasons that have nothing to do with the bug. Static-analysis-first,
  live-browser-to-confirm-the-fix second — not the other order — when the
  question is "is this code path ever actually exercised."

- **When deepening a multi-input instrument, check an expressive signal
  exists meaningfully on every input modality before wiring it in — not just
  on the one that makes it easiest to reach for.** Deepening Chime
  (crit-4, run at 159h to cutoff) needed a velocity-like dimension.
  `PointerEvent.pressure` was the obvious first reach — it exists, it's a
  number — but it's unreliable across devices (0.5 default for a mouse,
  often 0 for touch without force support) and keyboard has no analogue
  signal at all, so using it would have deepened only the pointer path and
  quietly broken the instrument's own pattern of one voice-management path
  for mouse/touch/keyboard. The actual signal used instead — time between
  successive note-onsets — exists identically on both a pointer glissando
  and a fast keyboard run, so one shared clock (`strikeVelocity()`) drives
  loudness/brightness/attack-shape for every input type the same way.
  Generalises past this one instrument: any time a spec asks for multiple
  input modalities and a later deepening pass reaches for a per-modality
  hardware signal, ask whether a *timing*- or *sequence*-based signal
  (derived from events you already unify, not from raw hardware data) would
  cover all the modalities instead of just the fanciest one.

  A third deepening pass on the same instrument (crit-4, run at 141h to
  cutoff, chord density) found a further step past "prefer timing/sequence
  signals over hardware": some expressive dimensions aren't keyed to any
  single input *event* at all, and are better read off shared *state* instead.
  `chordDensity()` reads `activeVoices.size` — the live count in the map every
  input path already funnels into — rather than anything about how or when a
  particular note was triggered, so it's unified across mouse/touch/keyboard
  "for free," with no per-modality reasoning needed at all (unlike velocity or
  hold-time, which still had to be checked against each modality even though
  they passed). Worth trying this as the *first* question for a future
  dimension, before reaching for a timing signal: is there already a piece of
  shared state (a map's size, how long it's been non-empty, its contents)
  that this dimension could read directly, rather than deriving it from a
  per-input event stream at all.

## Publishing is the harness's job, not mine

Run 12's hand-off wrote "run the `/ship` skill" as a next action. There is no
such skill in the available-skills list, and doctrine says why it doesn't need
to exist: "the trusted harness scans, publishes, deploys and freezes the exact
commit you pushed; you never receive its GitHub credential." Confirmed run 13
(21h to cutoff): `gh auth status` is logged out, the repo API returns 404
unauthenticated (consistent with still-private), and there is no `gh`/API
token anywhere in env/netrc to change visibility even if I wanted to. My job
is to get a clean, pushed `main`; making the repo public and deploying Pages
happens on the harness's own schedule, outside my access entirely. Don't spend
a future run hunting for a way to flip repo visibility myself.

## Doctrine timing, reaffirmed

"Finishing steps" (including the push) are gated to inside 24h to cutoff;
before that, plan/build/deepen and commit locally without pushing.

**Out-of-band commits are normal, not a doctrine violation.** Across ten runs,
`origin/main` has repeatedly gained commits I didn't push myself, from three
distinct non-me sources: the harness's own `memory: tick snapshot ...` commits
(plain `git push` of whatever's sitting on local `main`, including any commit
I made but correctly left unpushed under the inside-24h gate — so "my commit
is already on origin" is never evidence a push rule was broken); and three
convenor-adjacent identities, `Ben Swift`, `COMP4020 teaching team
<comp4020@anu.edu.au>`, and `COMP4020 course automation <noreply@anu.edu.au>`,
pushing legitimate course-wide maintenance (CI hardening, reflection-naming/
prompt-order rule changes, `.gitignore` scope, and — new this run, crit-4,
135h to cutoff — a wholesale "bring the course-owned checks forward to the
template tip" commit that dropped `oxlint`/`stylelint` out of `pnpm check`,
rewrote `check-evidence.ts` to work offline from the repo name alone, and
added `meta description`/`og:image` invariants) directly to this student repo.
Signal for "this is convenor, not a violation":
a real person/team name (not "harness"/tick-snapshot) plus course-wide scope
rather than content specific to this site. Don't revert or fight these.

**But check content, not just the check, after one lands.** Twice now
(runs 5 and 6) a convenor commit changed a *rule* (reflection heading should
be the deliverable title not a week number; prompt order should be
breakthrough-first) by editing `reflections/README.md`/`CLAUDE.md` only —
leaving this repo's actual `reflections/crit-1.md` still following the old
rule, invisibly, because `pnpm check:evidence` only validates the reflection's
filename/word-count/citations, never its heading or content order. Standing
check: whenever a reflection- or evidence-adjacent convenor commit lands,
re-read `reflections/crit-1.md` itself against the current wording of
`reflections/README.md` and `CLAUDE.md`, not just re-run `check:evidence`.

The habit generalises past reflection-rule changes: the course-automation
commit above (crit-4, 135h to cutoff) was a *tooling* change, not a
reflection-rule change, and reading its full diff (not just its commit
message) is what surfaced that `public/card.png` had been dropped in as the
template's literal "Replace this card" placeholder — flagged in the commit's
own prose as "still to be replaced," but a fact that would never have shown
up as a failing check, since the new `og:image` invariant is presence-only
(any file at the path passes). Replaced it with a card generated from the
instrument's own `layout()`/`draw()` math rather than a generic banner — see
the card-generation technique below. General lesson: after *any* course-owned
commit, skim the full diff for prose that names something still needing
action ("replace this," "still to be filled in," "template default"), since
that class of task passes every automated check by construction.

**`now.md` is a hand-off, not a ledger.** It can go stale or skip a run (one
run's real commit, `2d18c08` adding the typecheck sensor, went unmentioned by
the next hand-off). "Take stock" (routine step 3) means reading `git log
--format='%h %an %ad %s'` since the last known state and reconciling it
yourself, not trusting the previous `now.md` prose at face value.

**"Content-complete" was true of the brief, not of every viewport.** Runs
3, 5–10 repeatedly found nothing to fix, but none since run 10 had actually
re-opened the browser — run 11 explicitly skipped it as redundant, and that's
exactly the run window in which the about-page image-overflow bug (see above)
sat unnoticed. Don't manufacture scope against a satisfied brief, but "checks
green + reflection matches README" is not sufficient evidence the rendered
page is fine — a real-browser pass at both viewports still needs to happen
periodically (not necessarily every run, but don't let it lapse for several
runs in a row on the assumption that nothing rendering-related could have
changed when nothing else changed either — this run had zero upstream
commits and still found a real bug that had presumably been there since
whenever the about page's image was added).

Third confirmation, this time in assignment-1 itself (run 6, 124h to
cutoff): five prior runs (2–5) had called the yīhuà build "content-complete,
nothing to fix" on the strength of 31/31 green checks alone, without a
fresh screenshot pass. A phone-viewport (390×844) screenshot caught a
dead ~150px gap between the speed slider and the "Clear canvas" button,
fixed in `ae3df16`. Same shape as the about-page bug: a pure CSS/layout
defect no automated check (typecheck/build/lint/axe/contrast/vitest) can
see, because none of them render at a real viewport. The specific CSS
trap: `.control` had `flex: 1 1 16rem` sized for the desktop row layout;
the mobile media query flipped the container to `flex-direction: column`
but left that basis in place, and a flex-basis always sizes along
whatever the *current* main axis is — under column layout, "16rem" became
a 256px minimum **height** around ~100px of real content, not a width.
Whenever a media query flips `flex-direction` between row and column,
check every `flex-basis` set for the other axis inside that same query;
don't assume it only affects the properties the query explicitly names.

Fourth confirmation, and a new shape of bug (assignment-1, run 63h-to-cutoff):
"content-complete, nothing to fix" had held for several runs on 31/31 green
checks plus periodic screenshot passes, but a screenshot alone wouldn't have
caught this one — it was a *label-vs-physics mismatch*, not a layout defect.
The keyboard demo stroke's status text ("swift and dry" / "even-handed" /
"measured and dark") looked plausible at every slider position in isolation,
and the drawn stroke looked like *a* stroke either way — nothing to eyeball
as wrong. What caught it was driving the interaction with `agent-browser
eval`, reading the actual slider value and computed classification, then
independently recomputing the same math (`node -e`) to check the two agreed.
They didn't: the demo path timed its points by x-only spacing while the path
also moved in y, so the real speed fed to the width/opacity functions was
inflated above what the slider implied, by a different ratio depending on
canvas width — same slider position, different label at each marking
viewport (`f5bb895`, assignment-1). Generalises past this one bug: for any
interaction whose displayed *label or classification* is derived from a
computed value (a speed, a score, a threshold band), a screenshot only
proves a stroke was drawn, not that the label matches the maths — cross-check
the number, don't just trust that the UI shows *something* plausible.

Fifth confirmation (crit-4, `Chime`, 124h to cutoff): a mobile-viewport
screenshot (390×844) of an already-shipped, 23/23-green instrument showed the
scattered notes bunched into a thin horizontal band with most of the screen
empty — `layout()`'s vertical spread was `height * 0.18`, tuned by eye against
a desktop window and never re-checked at a tall aspect ratio. Fixed
(`f7f2ad7`) by switching the amplitude to `height * 0.32` once `width <
height`. Same root cause as the yīhuà flex-basis bug: a layout constant tuned
for one aspect ratio silently wrong on the other, invisible to every
automated check because none of them render at a real viewport. Reconfirms
the standing rule ("content-complete... is not sufficient evidence the
rendered page is fine") on a fifth distinct codebase — worth treating as
settled rather than something to keep re-deriving evidence for.
