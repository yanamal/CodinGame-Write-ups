# Code of Kutulu (contest) Post Mortem

## Initial Bot
My initial simple(ish) bot got me through to Bronze. 
I mostly used it to get a handle on the game rules and how various strategies pan out.
Many of its heuristics were purposefully extremely naive, just so I could see what I could get away with.

It tried to do the following things (in that order of priority):

### Evade imminent threats from wanderers
If there is a wanderer < 3 squares away (manhattan distance), try to avoid him. 
(Manhattan distance is a reasonably accurate proxy for actual walk distance when you're that close)
- If there's another explorer who's closer to the wanderer than me, and also is close enough for the YELL to work, then yell at him.
- Otherwise, try to move in a direction that is "safer". By the end (of me using this code), "safer" turned out to be:
  - At least as far away from the wanderer as I currently am
  - Closer to other explorers than I currently am (strictly closer)
  - Not jumping in the LoS of a stalker who is going to rush sooner than the wanderer would get me.
  
  I have no recollection as to why I'm using strict inequality to get closer to explorers, but <= to get away from wanderers. Guess it just worked better?..

### Try to not get rushed by a stalker


### Consider using abilities

### Just get closer to other explorers
If none of the above gives you a move to do, just find the closest explorer and move toward him.
(actually, that was the very first calculation in my sequence, it just got overridden if I found something better to do)
