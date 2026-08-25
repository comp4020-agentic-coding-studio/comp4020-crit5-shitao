# Process overview

## What I built

**One Stroke**: a single ink brushstroke, steered by pointer, touch or arrow
keys, that has to keep threading a gap in the next dark band while its ink
runs dry unless it also catches the small red drops drifting past. Either
failure (a wall, or the stroke running out of ink) ends the run and a fresh
one starts a moment later; the opening screen shows nothing but the still
brush and the first band already approaching, so the only way to find out
what the mouse does is to move it.

## The moments that mattered

1. **Keeping the simulation pure paid off immediately.** A canvas game is easy
   to end up testing only by eye, and `vitest`'s jsdom has no real canvas
   backend to draw against anyway. So the wall-collision rule, the ink
   depletion/collection rule, and the deterministic gap pattern all live in
   `game-logic.ts` as functions of state and elapsed time, with no
   `Math.random` and no canvas or DOM in sight; `main.ts` only reads that
   state to draw it. `spec/game.test.ts` puts the one rule the brief asks
   for --- a wall ends the round only if the brush misses its gap --- under a
   table-driven test, alongside the ink-runs-dry and drop-collection cases,
   all runnable with no browser at all.
   [`21de962`](https://github.com/comp4020-agentic-coding-studio/comp4020-crit5-shitao/commit/21de962)

2. **Playing the finished game found a problem reading the config couldn't.**
   The ink-decay constant (0.045/s, a 22-second dry-out with zero drops) read
   like real pressure on paper. Actually playing a sustained dodge run ---
   using `agent-browser` to steer through several gaps in a row, and reading
   the ink meter back out of the canvas's own pixels rather than adding a
   debug hook --- showed ink sitting at 60-90% the whole time: the drop and
   gap patterns wander through similar y-ranges, so tracking gaps well means
   passing close to most drops anyway. The second mechanic never actually
   threatened a competent player, which undercuts the brief's case for
   combining two mechanics in the first place. Raised `inkDecay` to 0.09 and
   dropped `inkPerDrop` to 0.2; replaying the identical dodge sequence showed
   a real downward trend (70/60/90/65/38/12) and a visibly thinning brush and
   trail near the end.
   [`cd95873`](https://github.com/comp4020-agentic-coding-studio/comp4020-crit5-shitao/commit/cd95873)

3. **A run resetting to zero every death gave a second life nothing to chase.**
   The brief's "thread worth pulling" is what carries five minutes past the
   first two mechanics being understood, and distance already doubled as the
   score --- it just vanished on every reset. Persisting the best distance in
   `localStorage` and drawing it as a small second number beneath the live
   score turns "try again" into "beat that," without adding a word of
   instruction: verified live in `agent-browser` by dying deliberately,
   confirming the number survived both the in-page reset and a full page
   reload at both marking viewports.
   [`3f9ea50`](https://github.com/comp4020-agentic-coding-studio/comp4020-crit5-shitao/commit/3f9ea50)

4. **A working game and a game that survives a storage-blocked browser
   aren't the same thing.** Persisting the best distance had read/written
   `localStorage` directly at module load, with nothing catching the
   `SecurityError` that throw in some private-browsing/storage-blocked
   configurations --- an uncaught throw there would abort the script before
   the first frame ever drew, so the entire game (not just the best-score
   feature) would fail silently for those visitors. Confirmed the failure
   and the fix the same way: an `agent-browser` init script patched
   `localStorage` to throw before the page loaded, showed a blank canvas
   against the unguarded code, then a rendering game against the
   try/catch-wrapped version.
   [`f6989ee`](https://github.com/comp4020-agentic-coding-studio/comp4020-crit5-shitao/commit/f6989ee)
