---
layout: post
title:  Hexplode Game AI
date:   2017-08-19 09:00:00
excerpt_separator: <!--more-->
categories: [python, algorithms, games]
tags: [python, algorithms, games]
---
**The game hexplode lets two players fight one another using exploding stacks of
their tokens as means to take over enemy tokens. Over time, chain reactions become
more and more common and as a result the game becomes difficult to predict.
Here, I implement a 'minimax' algorithm to create a (very basic) computer
player for this game**

<!--more-->

### The Game
Hexplode is almost as old as space invaders. The game of was first created as a
BASIC program for the BBC Micro written by J. Ansell in 1982. I started working
on this game as part of the 2016 Man AHL coder prize. I am using their implementation
here, which is [openly available on GitHub](https://github.com/manahl/hexplode).
Their summary of the game is this:
> Hexplode is played by 2 players on a board of hexagonal tiles. The players take turns to place a counter on the board, they can only play a counter in an empty tile or a tile they already own. When the number of counters on a given tile equals the number of adjacent tiles the tile hexplodes and the contents of the tile are equally distributed to its neighbours changing the ownership of neighbouring counters over to the current player in the process. This process can often generate a chain reaction. The winner is the player that removes all of their opponents counters from the board.

Man AHL ships their version with a flask app that allows us to run the game
as a web server. This can be done either locally, or in the cloud. I have
a version running on [PythonAnywhere](https://www.pythonanywhere.com/),
which you can access at [http://janlauge.pythonanywhere.com](http://janlauge.pythonanywhere.com/).
Here is an example of what it looks like:
![Game example]({{ site.url }}/assets/hexplode_example.gif)

On the algorithm backend, the game is represented by a simple board state in the form of a
python dictionary. Each tile has a `tile_id`, a `score` (the number of counters on that tile),
an `owner` (the ID of the player that owns the token on this tile),
and a list of `neighbours` indicated by their respective `ID`s.  

```python
board = {
    tile_id = '3b',
    score = 1,
    owner = '96196ce4df35aa7ed838'
    neighbours = [
        '2a', '2b',
        '3a', '3c',
        '4b', '4c'
    ]
}
```

### The Algorithm
In a game-theoretical sense, hexplode is an adversarial, deterministic game with
perfect information. This means that the players are pursuing opposing goals,
that there is no element of chance or randomness, and that all information
about the state of the game world is always available to all players.

All three of these are attributes that hexplode shares with chess. I therefore decided
to start my work on the competition with a 'MiniMax' implementation. This algorithm
uses a tree approach. Each branch is a possible move, each leaf represents the
resulting board state. Board states in terminal leaves are evaluated one by one,
as "game over" for either player (+/- infinity), or, if that does not apply, using a heuristic
value function (see diagram below).
![MiniMax algorithm]({{ site.url }}/assets/hexplode_minimax.png)
*source: [Wikimedia commons](https://commons.wikimedia.org/wiki/File:Minimax.svg)*

Minimax is relatively straight-forward to implement. First, we need to be able to
calculate future board states as a function of our moves. I created a function
`AddCounter` for this, which uses a list of scores, the board shape, the id of the
tile that the counter should be placed on, along with the player_id (the algorithm)
and the current player (the one placing the counter, because we want to be able to
simulate enemy moves as well). Note that this is a **recursive** function,
because placing a marker on a tile can create an instable board state with
as many counters on a tile as that tile has neighbours. In this case,
the tile will "hexplode", distributing its counters to the neighbouring tiles.
This, in turn, could trigger subsequent hexplodes on those neighbouring tiles,
which would be handled by the respective recursions triggered by the function.

```python
def AddCounter(scores, BoardShape, tile_id, player_id, player_current):
    """Add a counter to the board and calculate
    the new resulting board configuration"""

    # Check if the game is still going (both player and enemy have remaining token)
    if (True in [score > 0 for score in scores.values()]) and (True in [score < 0 for score in scores.values()]):
        #If player, counters are positive
        if player_current == player_id:
            add = +1
        #If enemy, counters are negative
        else:
            add = -1
        #If placing counter on own tile, increase counters
        if (add > 0 and scores[tile_id] >= 0) or (add < 0 and scores[tile_id] <= 0):
            counter = scores[tile_id] + add
        #If placing counter on enemy tile, change and increase counters
        else:
            counter = (scores[tile_id] * -1) + add
        #If the tile is full, hexplode (recursive)
        if abs(counter) == len(BoardShape[tile_id]["neighbours"]):
            # Reset tile counter to zero
            scores[tile_id] = 0
            # Add one token to each neighbouring tile
            for neighbour in BoardShape[tile_id]["neighbours"]:
                scores = AddCounter(scores, BoardShape, neighbour, player_id, player_current)
        else:
            scores[tile_id] = counter
        return(scores)

    else:
        return(scores)
```

-----------------------

Now that we can calculate potential future board states, we can use this to
create a tree of move scenarios. A given board state allows for a number of moves
("place a marker on any tile that is not an enemy tile"). When it is our turn,
we can make `n_1` valid moves resulting in `n_1` possible board states. Using
another recursion call we can calculate `n_2` valid enemy moves with termination
conditions for a maximum depth (`n_i`) and/or for a game-over situation
(one player looses all counters).
