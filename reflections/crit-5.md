# A game

The breakthrough was realising the ink-decay constant was wrong not because
the number looked wrong, but because playing revealed *why* it was wrong. The
brief's harder move is two mechanics that interact --- wall-dodging and
ink management --- and 0.045/s looked like real pressure on paper: a 22-second
dry-out with no drops collected. But steering through a sustained run and
reading the ink meter back off the canvas's own pixels showed it sitting at
60-90% the whole time. The reason wasn't the number, it was the geometry:
the drop and gap patterns wander through similar y-ranges, so tracking gaps
well means passing close to most drops anyway, for free. No config read, no
unit test, and no screenshot could have surfaced that --- it only showed up
as a felt absence of tension across a few real minutes of play. Raising decay
and cutting the per-drop refill turned that into a real trade-off worth
tracking.

That distinction --- a bug that lives in the *interaction* between two
correct-looking pieces, not in either piece alone --- turned out to be the
shape of every real defect this build shipped: a stuck keyboard flag that
silently overrode the mouse, a canvas resize handler feeding its own drifted
output back in as its next input, a localStorage guard that only mattered
because it sat upstream of every other feature. What this changed for me is
where I look for confidence. Green checks and a correct-looking screenshot
describe *state*; only playing, resizing, and reading pixels back describes
*behaviour over time* — and that's the only place these bugs live.
