---
layout: post
title:  Hexplode minimax
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
which you can access at [http://janlauge.pythonanywhere.com](http://janlauge.pythonanywhere.com/)

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

![MiniMax algorithm]({{ site.url }}/assets/minimax.png)
Source: [Wikimedia commons](https://commons.wikimedia.org/wiki/File:Minimax.svg)
