# Initial Bot
My initial simple(ish) bot got me through to Bronze. 
I mostly used it to get a handle on the game rules and how various strategies pan out.
Many of its heuristics were purposefully extremely naive, just so I could see what I could get away with.

It tried to do the following things (in that order of priority):

## Evade imminent threats from wanderers
If there is a wanderer < 3 squares away (manhattan distance), try to avoid him. 
(Manhattan distance is a reasonably accurate proxy for actual walk distance when you're that close)
- If there's another explorer who's closer to the wanderer than me, and also is close enough for the YELL to work, then yell at him.
- Otherwise, try to move in a direction that is "safer". By the end (of me using this code), "safer" turned out to be:
  - At least as far away from the wanderer as I currently am
  - Closer to other explorers than I currently am (strictly closer)
  - Not jumping in the LoS of a slasher who is going to rush sooner than the wanderer would get me.
  
  I have no recollection as to why I'm using strict inequality to get closer to explorers, but <= to get away from wanderers. Guess it just worked better?..

## Try to not get rushed by a slasher
For each slasher who's either spawning, stalking, or stunned - if his "time to state change" parameter is 3 or less (meaning that he might rush in 3 turns), try to get out of LoS
> *Note*: This is all kinds of wrong, since stunned does not go directly into a rush, and rush actually takes one more turn, etc. Also, I didn't care about walls when calculating LoS, but that at least was a conscious simplification.

## Consider using abilities
If I haven't found a good move to make, then consider using PLAN or LIGHT.

- If I have plans left over, and also using plan wouldn't take me over max health (otherwise it would be wasteful), then use PLAN
  - the "how much health would PLAN add" calculation was just time_plan_lasts * base_health_gain_for_planning. I didn't bother taking into account other players **or** health loss over time
  - this heuristic actually made it all the way to my final bot.
- If there is a wanderer who would get caught in the light radius, and also an explorere who would **not**, then use LIGHT.

Light overrode plan if both were possible.

## Just get closer to other explorers
If none of the above gives you a move to do, just find the closest explorer and move toward him.
(actually, that was the very first calculation in my sequence, it just got overridden if I found something better to do)
