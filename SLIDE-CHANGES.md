# Slide changes in this branch (2026-11-15)

This branch is [julcnx/strava-heatmap-extension](https://github.com/julcnx/strava-heatmap-extension)'s v0.15.4 branch
(commit e2361f8, the v0.15.4 beta) plus the Slide changes that Chugach Mountain Bike Riders (CMBR) uses to fix trail
lines in Chugach State Park, Alaska. Only `src/clients/www.openstreetmap.org/slide/algorithm.js` and its tests differ
from the beta. It is a working copy for testing, not a release and not a pull request.

To try it, load the folder as an unpacked extension, the same way as the beta.

## The four parts, most important first

### 1. Switchback tips

The beta's smoothing blurs the line by about 5 m along its length, so it can cut switchback tips short. Here:

- **Tips are held still while the rest of the line is smoothed**, and smoothing runs up to each tip from both sides.
  A tip is a point where the traced line turns at least 95 degrees between chords 8 m behind and 8 m ahead (the beta:
  100). No tip is held within 20 m of a line end, where a sharp turn is usually a hook into a fork.
- **Tips are rounded more strongly.** Every tip the centring step finds (held or not) gets 10 averaging passes over
  6 samples either side, about 9 m (the beta: 2 passes over 2 samples), so a tip is not one sharp corner. A corner
  that smoothing held always counts as a tip here.
- **The walk out to the end of the band also stops at a dip.** The beta's walk stops only where the band ends, or at
  24 m. Here it also stops once the heat has fallen below 85% of the brightest heat seen on the way out and then rises
  again: that is another trail's heat beyond the tip.
- **Spike guard:** a tip is held only where the drawn line turns at least 80 degrees over 20 m either side, at a drawn
  point within 8 m of the tip. A false hairpin that the trace makes in braided or faint heat, where the mapper drew
  none, is then not held. It can still be pushed out by the walk above.

Effect, with the whole branch (all four parts): on the switchback stretch of South Fork Valley Trail (way 239468111),
the 90th-percentile distance from a hand-edited reference line drops from 3.7 m (the beta) to 1.4 m.

### 2. Junctions

- **Junction snap reach is 9 m** (the beta: 18 m). With this branch's finer smoothing step (part 4), an 18 m reach let
  a junction jump to a different intersection in the heat and fold the line (Denali View Trail, way 76158507, node 81:
  a 10.9 m fold). At 9 m, Flattop Sunnyside Trail (way 332538430) node 149 moves 2.2 m instead of 10 m.
- **A crossing stays where it is drawn if it sits on this line's own band** (the band's ridge within 2.5 m). Otherwise
  it slides as in the beta. Examples: Hunter Pass Bikers Trail (way 1192973305) node 89 moves 0 m instead of 2.9 m;
  Denali View node 28 moves 0 m instead of 5.8 m.

These junction figures come from two earlier builds of ours with one of these rules each (the 9 m reach; crossings
that always stay as drawn), against the beta. The 10.9 m fold is from our build before the 9 m reach. Our previous
build, which has both rules as here, gives the same lines as those builds on these runs. Node 149's 2.2 m is from a
live preview in iD; the beta's 10 m there is from an offline run.

Limits:
- 9 m is not a cap on every junction move. A movable junction that isn't snapping to an intersection can still move up
  to 25 m, as in the beta. On Hemlock Knob Trail (way 75370002) a junction slides 10.1 m onto a brighter trail's band;
  on Near Point Trail (way 525238470) one moves 13.7 m.
- julcnx raised two objections to these changes in #43: a 9 m reach misses real splits that are further off, and at
  crossings the other trail's heat can make "on this line's own band" true too often, which leaves a kink.

### 3. Curve centring

On curves, each point moves towards the middle of the band (halfway between the edges where the heat falls to half the
peak), by at most 2 m. It skips tips, merged switchback legs, line ends and bands wider than 14 m. The beta's line sits
a little inside curves; this brings it back. On its own the gain is small (about 0.1 m); it shows with part 4.

### 4. More points

The point tolerance is 0.3 m (the beta: 1.2 m) and the smoothing step is 0.75 m (the beta: 1.5 m). Together with
part 3, on South Fork Valley Trail's curves the 90th-percentile distance from the hand-edited reference line drops by
0.3 to 0.6 m. The line has about twice as many points (2.05x on 681 park sections). The cost: in messy heat, a line
with more points is more work to fix by hand.

## How it was tested

All offline, on saved heat tiles. The park sections and the wobbled inputs were run the way the extension runs Slide
in iD, with the result cleaned twice by Slide's own clean-up (`cleanSlideResult`). The slide lab cleans once. The
other Alaska runs (including the offset and cut-short inputs), the 86 ways and the made-up shapes use the line before
clean-up.

- `npm test`: 27 pass, 0 fail, 1 todo (the faint-trail case, the same todo as the beta). No test exercises the dip
  stop or the spike guard yet; made-up heat in the tests' style doesn't reproduce the real cases.
- 15 test runs on Alaska trails, 86 OSM ways around them (13 with switchbacks), and the slide lab's 33 ways plus
  Catamount Trail (way 239840702).
- 681 sections of official trails in Chugach State Park. Every section where this branch and a tips-only version differ
  by more than 5 m was checked by eye. Most of those differences are junctions (part 2).
- The same trails with deliberately rough drawn lines: offset, wobbled, cut short.
- A live check in iD on a few trails.

The testing was done with AI help, and the results were checked by eye.

## Known limits

- **Tight, clean hairpins are cut more than in the beta.** On made-up hairpins with legs 9 m and 6 m apart, the tip ends
  3.8 m and 3.3 m short (the beta: 1.3 m and 1.3 m). On Hunter Pass Bikers Trail it is slightly worse than the beta: the
  90th percentile is 1.20 m against 1.06 m, and the maximum from a rough line is 3.6 m against 2.7 m.
- **Some corners end up sharper than the beta's, close to the 150 degree limit.** Slide's clean-up
  (`cleanSlideResult`) removes only points over 150 degrees. On Ptarmigan Pass Trail (way 882990922), in faint heat,
  a 149.9 degree point stays (the beta's sharpest there: 107 degrees). On Crow Pass Trail (way 1082255007) the sharpest
  point is 148.8 degrees at a junction node (the beta: 137 degrees); others reach 145-146 degrees. Near Point Trail
  (way 525238470) gets a 167 degree turn at the junction where a side loop rejoins: the clean-up doesn't drop junction
  nodes.
- **Wobbly drawn lines.** On Flattop Trail (Uphill) (way 983120521), a drawn line that wobbles 6 m every 60 m gives a
  loop about 10 m out into pale heat that crosses itself. A wobble of 8 m every 100 m gives a 13.7 m needle into the
  band. On Little O'Malley Peak Trail (way 866425307) with a 7 m wobble, the line crosses itself and stops about 6 m
  short of a tip (the beta stops about 7 m short there, without crossing).
- **Tips drawn cut short.** The spike guard's margin is thin: real tips in our tests turned 92 degrees or more on the
  drawn line, false ones 65 degrees or less. A real tip drawn cut short can lose its hold, and then it doesn't get the
  gain. In our tests that happened at cuts between 10 and 18 m, depending on the tip (its drawn turn falls below 80
  degrees, to between 62 and 77 degrees, or to 0 where the drawing is more than 8 m from the tip). Where
  a refused tip sits in braided heat the line can end further off: on Flattop Trail (Uphill) with its tip drawn 10 m
  short, 14.4 m from the reference against the beta's 9.3 m.
- **Some tips aren't recognised.** On Flattop Sunnyside Trail (way 332538430) one tip stays a single sharp corner.
- **A tip at the end of a selection stays sharp.** Slide doesn't move a selection's end nodes, so select across a tip,
  not up to it.
- Junction moves can exceed 9 m (see part 2).
