# Code of Kutulu (contest) Post Mortem

## [Initial Bot](Initial-Bot.md)
Initial naive bot for experimenting with the rules - took me to Bronze.
(description in [separate file](Initial-Bot.md))

## Actual Bot

My final bot relied on calculating lots of different "maps" of important per-cell data (e.g. "quickest time an enemy might reach this cell") and then using them to pare down the viable move options to "good" ones until hopefully I decide on one best move.

Using abilities was sort of separate, but sort of intermixed with the move decision making.

### 1. "Threat maps" for each minion
For each minion, and for each walkable square on the map, calculate how soon a minion could reach this cell (and deal damage to it).

#### Wanderers
For wanderers, this was fairly straightfowrard. I just used [Dijkstra's algorithm](https://en.wikipedia.org/wiki/Dijkstra%27s_algorithm) to calculate the "distance" from the wanderer to each cell on the map. 

> Near the end, I started precalculating this on the first turn for each starting cell, so I had the 'walking distance' from each cell to each other cell ready. It honestly didn't seem to affect performance that much, though.

After calculating the map, I made a couple of adjustments to the resulting values:
- if the wanderer is targeting someone other than me, I cut off the calculation for distances >`dist(wanderer, target)` (because I was safe from that wanderer targeting me if I didn't get closer than that)
- If the wanderer is still spawning, I added the time-remaining value to each cell.
- If the wanderer is targeting me, then assume that he will go directly toward me; so for turns 0 through `dist(wanderer, me)`, only mark the cells on a direct path to me as dangerous. Then, past that, use the Dijkstra's as usual.
  - I wrote this toward the end, when I noticed that my bot was "afraid" of running around in a circle if the wanderer who was following him was halfway through that circle.
  
> I thought about this for a while, and ended up not using light in these calculations at all - since this is supposed to be how long it will actually take them to get there if they tried, not how long they think it might take.

#### Slashers
Slashers were, of course, much more complicated. I tried ignoring "wandering" slashers, and then tried pretending they were wanderers. Both worked OK, but far from perfect.

I ended up having to do Dijkstra's on a sort of "3D map" where the third dimension was the slasher state. So you can think of it as 6<sup>1</sup> layers of maps, each layer corresponding to a slasher state, with information on how soon you might see a slasher in a given cell, in that particular state. Then you can imagine a directed graph drawn through this 3D map, with an edge from A to B meaning that it's possible for the slasher to transition from location+state A to location+state B. and the weight of the edge is how long it would take the slasher to make that transition (e.g. 2 turns to go from stalking to rushing). This representation allowed me to precisely calculate when and where slashers can strike, without thinking too much about it - just wrote a "possible transitions and distance from state+location A" function, and then plugged it into a standard Dijkstra implementation.

Getting this "transition function" right was super fiddly, hard to get all the slasher rules right.

<sup>1</sup> *Wait, Slashers only have 5 states!* Yeah, I added a fake 6th state of "actually dealing damage now", which immediately transitioned into "Stunned". The "dealing damage" state is what I subsequently used as the actual threat map, and threw out the rest.

For transitions from the RUSH state, if I was pretty sure that the current target is going to be in LoS by the time the Slasher strikes, I only marked the target's position as dangerous; otherwise, I considered everything within the LoS at time of rushing to be dangerous.

> I did not precalculate any of this on turn 0, since there are dependencies on current state (mostly the target).

### 2. Timing Map
Combine all the threat maps from all the enemies into a "timing map" by taking the min. time across all maps for each cell. So for each cell, the timing map tells me how soon I think an enemy would be able to get there. Or in other words, how long I think that cell will remain "safe" for. Obviously, this is somewhat naive, as it does not account for various other game events that change the state. But it's good enough for the heuristics I was going to use it for.

### 3. Walkable Paths
Using the timing map, calculate all the places I could reach before any minion would reach them. This also used a variation on Dijkstra's algorithm, where you could go from cell A to its neighbor B if and only if your time-to-get-there was lower than cell B's rating on the Timing Map. And since Dijkstra's starts at your location and expands out, it naturally made sure that only nodes with an actual safe path to them were actually calculated. Plus, as a side effect, Dijkstra's also can generate the actual tree of shortest paths, so if I knew I wanted to get to somewhere that was considered "safe", I already had the information on how I'd actually go about getting there.

### 4. Sanity Bonus Map
Separately from all that threat stuff, calculate a map of "Sanity Bonuses": the places on the map that are better for my sanity. This just counted the benefit I would get **on this turn** if I were at this cell **right now**. Also pretty naive, but it was close enough to make decisions about which direction it'd be beneficial to head in.
- **Shelters** were the easiest - for each active shelter, add the shelter bonus to that cell on the map.
- **Plans** were also pretty straightforward - for each plan, for each cell that the plan reaches, add the plan bonus.
- I also modeled **Group bonuses** in the same map: for each cell that was close enough to at least one other explorer, add the difference between `sanity_loss_lonely` and `sanity_loss_group`

### 5. Give up and minimax
The threat/walkable paths data was reasonably good for reasoning about things in the middle-term, but for very short-term predictions, I ended up needing something more precise once I got to Legend. So I did a minimax calculation to predict (and avoid) worst-case damage to me, on just this turn:
- For each of my possible moves:
  - consider all combinations of possible other-explorer moves (not counting any abilities; which is at most 125, and in practice usually more like 30-80; multiplied by the number of my possible moves, of course)
    - for each combination, "simulate" the damage that all the minions will do to everybody on that turn. Not, by any means, a full simulation - I didn't even bother calculating where the minions would actually end up. Just whether they'd be able to deal damage, and to whom.
  - pick the combination(s) that are the worst case: (this is the **max** part of minimax)
    - all combinations with the the most damage to me
      - out of those, all combinations with the least cumulative damage to others.
- Now we have the worst-case outcome for each move I can make; pick the move(s) with the best worst-case outcome.

> Note: For my purposes, it was important to collect **all** moves with the same outcome, since I could then filter them by other heuristics.

### 6. Finally, the heuristics!
Well, they do say that 80% of AI is representation. I forget who said that though. And whether that's actually what they said.

Just to be clear, though, I did not write all of the above in one go before writing heuristics. It all evolved over time together. Though I was able to add a massive number of heuristics - quite a few more than I ended up using - and iterate on them very quickly once I had the full representation in place.

### 7. Sprinkle in some abilities
