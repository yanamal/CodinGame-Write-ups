# Code of Kutulu (contest) Post Mortem

## [Initial Bot](Initial-Bot.md)
Initial naive bot for experimenting with the rules - took me to Bronze.
(description in [separate file](Initial-Bot.md))

## Actual Bot

My final bot relied on calculating lots of different "maps" of important per-cell data (e.g. "quickest time an enemy might reach this cell") and then using them to pare down the viable move options to "good" ones until hopefully I decide on one best move.

Using abilities was sort of separate, but sort of intermixed with the move decision making.

### 1. Calculate "threat maps" for each minion
For each minion, and for each walkable square on the map, calculate how soon a minion could reach this cell (and deal damage to it).

#### Wanderers
For wanderers, this was fairly straightfowrard. I just used [Dijkstra's algorithm](https://en.wikipedia.org/wiki/Dijkstra%27s_algorithm) to calculate the "distance" from the wanderer to each cell on the map. 

> Near the end, I started precalculating this on the first turn for each starting cell, so I had the 'walking distance' from each cell to each other cell ready. It honestly didn't seem to affect performance that much, though.

After calculating the map, I made a couple of adjustments to the resulting values:
- if the wanderer was targeting someone other than me, I cut off the calculation for distances > `dist(wanderer, target)` (because I was safe from that wanderer targeting me if I didn't get closer than that)
- If the wanderer was still spawning, I added the time-remaining value to each cell.

#### Slashers
Slashers were, of course, much more complicated. I tried ignoring "wandering" slashers, and then tried pretending they were wanderers. Both worked OK, but far from perfect.

I ended up having to do Dijkstra's on a sort of "3D map" where the third dimension was the slasher state. So you can think of it as 6<sup>1</sup> layers of maps, each layer corresponding to a slasher state, with information on how soon you might see a slasher in a given cell, in that particular state. Then you can imagine a directed graph drawn through this 3D map, with an edge from A to B meaning that it's possible for the slasher to transition from location+state A to location+state B. and the weight of the edge is how long it would take the slasher to make that transition (e.g. 2 turns to go from stalking to rushing). This representation allowed me to precisely calculate when and where slashers can strike, without thinking too much about it - just wrote a "possible transitions and distance from state+location A" function, and then plugged it into a standard Dijkstra implementation.

Getting this "transition function" right was super fiddly to get all the slasher rules right.

<sup>1</sup> *Wait, Slashers have 5 states!* Yeah, I added a fake 6th state of "actually dealing damage now", which immediately transitioned into "Stunned". The "dealing damage" state is what I subsequently used as the actual threat map, and threw out the rest.

For transitions from the RUSH state, I also threw in some optimizations where if I was pretty sure that the current target is going to be in LoS by the time the Slasher strikes, I only marked the target's position as dangerous; otherwise, I considered everything within the LoS at time of rushing to be dangerous.

> I did not precalculate any of this, since there are dependencies on current state (mostly the target).
