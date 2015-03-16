---
layout: post
title: "Conway's Game of Life"
---

# {{ page.title }}

Since it is the first post I've written in a longer while, I'll keep it short and simple. Lately, in my spare time, I've been exploring C\# and .NET framework. To this end, I've had a look through two great books: [C\# in a Nutshell](http://shop.oreilly.com/product/9780596001810.do) by P. Drayton, B. Albahari and T. Neward, and [Concurrency in C\# Cookbook](http://shop.oreilly.com/product/0636920030171.do) by S. Cleary. In order to further enhance my understanding of the concepts therein presented, I've decided to implement something simple and yet engaging. For that wee project of mine, I've decided to go for the Conway's Game of Life.

## The rules of the game
The game was invented in 1970 by a British mathematician, John Conway. In the simplest terms, the game constitutes an imitation (simulation) of life. Within the game, the world is represented by a 2D grid, while each element of the grid, referred to as cell, represents living being. Yes, you guessed correctly, each cell is either alive or dead.

The evolution of life is subject to certain grand rules; that is, whether the cell survives or dies from one stage of the simulation to the next one is determined by the following conditions. The cell survives if 1) it is alive and it has exactly 2 alive neighbours (cells within distance of 1 from the current cell); or 2) it has exactly 3 alive neighbours, regardless of whether it currently is alive or dead. In all other cases, sadly, the cell dies.

## Example
The beauty of the game comes from the fact that, depending on the initial state of the world (the distribution of live vs dead cells), the simulation may lead to a totally different (and sometimes very surprising) end state.

To illustrate the beauty of the game, I've enclosed a GIF of one particular instance of the game:

![Example run: gif](/images/gof.gif)

If I've managed to spark your interest and, like me, you find the simulation weirdly mesmerising, feel free to explore the beauty of the Conway's Game of Life yourself using my C\# app. The source code is available [here](https://github.com/kubkon/GameOfLifeApplication).
