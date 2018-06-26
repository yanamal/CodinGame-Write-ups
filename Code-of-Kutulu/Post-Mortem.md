# Code of Kutulu (contest) Post Mortem

## [Initial Bot](Initial-Bot.md)
Initial naive bot for experimenting with the rules - took me to Bronze.
(description in [separate file](Initial-Bot.md))

## Actual Bot

My final bot relied on calculating lots of different "maps" of important per-cell data (e.g. "quickest time an enemy might reach this cell") and then using them to pare down the viable move options to "good" ones until hopefully I decide on one best move.

Using abilities was sort of separate, but sort of intermixed with the move decision making.

It's a bit long, feel free to skim or [skip to the good part](#6-finally-the-heuristics)

### 1. "Threat maps" for each minion
For each minion, and for each walkable square on the map, calculate how soon a minion could reach this cell (and deal damage to it).

#### Wanderers
For wanderers, this was fairly straightfowrard. I just used [Dijkstra's algorithm](https://en.wikipedia.org/wiki/Dijkstra%27s_algorithm) to calculate the "distance" from the wanderer to each cell on the map. 

> Near the end of the competition, I started precalculating this on the first turn for each starting cell, so I had the 'walking distance' from each cell to each other cell ready. It honestly didn't seem to affect performance that much, though.

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

In practice, a lot of the time, the damage values are the same across most/all of the possibilities: the particular explorer is just not going to get damaged this turn, or my move is just not going to affect what happens to the other explorers in the best/worst case. This is fine, and just means that the minimax part offered no opinion.

> Note: For my purposes, it was important to collect **all** moves with the same outcome, since I could then filter them by other heuristics.

Later I also added "average-case" damage calculation: for each one of my possible moves, calculate the **average** damage to me and others across all possible opponent moves. Of course, it would be a much more accurate average if it took into account the likelihood of each combination; but I did not want to go down that rabbit hole at that point.

### 6. Finally, the heuristics!
Well, they do say that 80% of AI is representation. I forget who said that though. And whether that's actually what they said.

Just to be clear, though, I did not write all of the above in one go before writing heuristics. It all evolved over time together. Though I was able to add quite a few heuristics - quite a few more than I ended up using - and iterate on them very quickly once I had the full representation in place.

- First, use the minimax function above to filter the possible moves. Since minimax looks for the **best** worst-case move, it will always return at least one move.
- Then filter for moves that actually are part of the "safe walkable path" tree
  - for all of those, apply a bunch of "heuristic filters" based on some of the data above (in this order):
    1. Filter out any moves that place you on a cell that has <= 2 "danger" on the Timing Map (in other words, you are liable to get hit next turn if you stand on those squares.
    1. Filter for the moves that will eventually lead you to the biggest bonus (using the sanity bonus map, and the dikstra-based "safe paths", to decide what path can lead you where)
    1. Out of those, filter for the paths with the closest bonus
    1. Out of those, filter for the paths that have the "longest time to live" - the greatest value on the Timing Map, basically.
    1. filter out cells where you are in danger of a "yell" attack - not only is someone able to yell at you, but being frozen in that cell will actually result in you being damaged (i.e. the "danger" of that spot on the Timing Map is <= 3) (unless all cells have that danger, in which case, don't filter any out)
    1. Filter for the best (least) average expected damage to you, according to the averages from the minimax function
    1. Same as above, but for best(most) average expected damage to opponents
    1. Finally, filter for moves that will get you closest to other players.
- After all of that, there could still be a few equally-good moves left; or, conversely, there could be no good moves left (usually because there were no "safe paths" found at all - we are surrounded!)

  If the safe-path filtering didn't come up with any good safe moves, fall back on just the original minimax results.
- At this point, we have settled on one or more moves we could make. Just in case there are more than one, do a little more filtering:
  - filter out moves that lead into a "dead end" (e.g. Typhoon)
  - finally, filter by preferring moves that aren't going "backwards": they are not where you're standing now or (even worse) where you were the turn before this one. This is to mitigate "waffling, where my bot would go back and forth between two relatively-good cells because he didn't see a reason to pick one over the other.

Phew! now actually move.

#### Notes
- Most of the time, depending on the situation, only 1-3 filters actually filtered anything - the rest just had "no opinion" (e.g. "there's nobody close by, so none of your potential moves create a danger of being yelled at")
- The ordering and the specific filters are basically a result of lots of trial-and-error tweaking (mostly on the last and second-to-last day). The bot's performance was **super** sensitive to the ordering, which does make sense; but I couldn't always predict which filter would be more important before trying it.
- I tried pretty hard to get rid of that first "safety threshold" filter - it seemed unnecessarily conservative and arbitrary, and I kept thinking "now that I have minimax..." "now that I'm doing longest-ttl anyway..." - but the bot always ended up worse without it.
- I tried a bunch of heuristics that would average out the expected bonus/safety/etc. down a (branching) path, but none of them worked nearly as well as just "biggest bonus", "most safety", etc.
- the "dead end" and "don't go backwards" filters are at the very end because they need to apply whether or not we fell back on the minimax calculation.
- The "closer to other players" is actually not the only filter that prioritizes proximity, since the Sanity Bonus Map also marks spots close to other players as more beneficial. However, if the "safe path" does not reach any other players - either it's not particularly safe, or the other players are quite far away - then those cells on the map don't come into play through the sanity bonus filter.

### 7. Sprinkle in some abilities

I did also use abilities. If I had designed the moves/filtering data structures a bit better, I could have probably slotted them in as "moves" and written more filters for them. As it is, I slotted them into a couple of places in the decision sequeunce, so that they canceled any decisions that came afterwards (this was easy to do, since I could just change the action from MOVE to YELL/LIGHT/PLAN and even if I calculated a move and passed down the coordinates, the YELL/LIGHT/PLAN would take over. So my bot usually says something like "YELL 10 13").

The two places where I checked for abilit use were:
1. before any move heuristics at all, check whether yelling would make another player take damage. Namely:
  - There is a player within YELL distance whom you have not yelled at
  - That player's position has a danger of <= 2 on the Timing Map (the cell will get hit by a minion in the next 2 turns)
  - Your own position has a danger of >= 2 on the Timing Map (you will be able to get away after yelling)
1. After either the safe-path filtering or the minimax fallback, see if staying put is still one of the options you're considering. If so, check for whether it might be a good idea to use abilities (in this order of preference)
  - PLAN: do it if you think it will not take you over the max sanity (250). Same naive heuristic as in the [initial bot](Initial-Bot.md#consider-using-abilities). It's silly, but it seemed good enough (didn't spam plan too much, didn't use it in totally unreasonable situations)
  - YELL: same as the first YELL heuristic, except don't bother checking if it's safe to stand still for a turn while yelling (since you're already considering it as one of the best possible moves).
  - LIGHT: use light if all of the following are true:
    - there is a wanderer targeting me
    - he is > half the light radius away (otherwise light would no longer help)
    - there is an explorer further away from me than the wanderer is (so the light won't help him as much as it helps me)
    
The yell logic worked really well; the plan one seemed reasonable. I think empirically, I did slightly better after I implemented the light one, but it's hard to tell.
    
