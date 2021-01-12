---
title: "Cellula Automata - Creating Wire World"
date: 2020-09-02T21:01:55+01:00
draft: false
description: "Creating a Wire World Cellula Automaton for fun"
slug: "" 
tags: []
categories: []
externalLink: ""
series: []
---

You might have heard of cellular automata. They consist of a grid of cells, where each cell has a finite number of states. It is the rules that govern these states that create a variety of interesting effects, from Conway's Game of Life, [giving rise to civilisations, wars and factories](https://en.wikipedia.org/wiki/Cellular_automaton), to [Wire World](https://en.wikipedia.org/wiki/Wireworld), a cellular automaton designed to simulate, at least in some respect, transistors.

So when my internet went out unexpectedly for an entire day, and having wanted to do something for a fun ["code jam"](https://itch.io/jam/olc-codejam-2020), I decided to make my own version of Wire World, to see what all the fuss was about. I'm quite pleased with the results, and it certainly killed the time I was hoping for. You can view it [on GitHub](https://github.com/lukawarren/PixelWire) if you'd like.

![Wire world screenshot](/img/wireworld/Logo.png)

# A computer within a computer
Wire World is really quite simple. There are four cells, and they act accordingly:
* Empty cells do nothing
* Electron heads (which are blue) turn into tails
* Electron tails (red) turn into conductors
* Conductors (yellow) turn into heads if exactly one or two neighbouring cells are heads, else stay the same

I used C++ and a tiny little framework called the [Pixel Game Engine](https://github.com/OneLoneCoder/olcPixelGameEngine) for graphics, and quickly had something in about half an hour. In fact the code for the actual simulation was really rather simple - this is all it is:

```
// Copy array to output
for (int x = 0; x < nWorldWidth; ++x)
	for (int y = 0; y < nWorldHeight; ++y)
		m_WorldLogic[x][y] = m_World[x][y];

// Iterate through each cell
for (int x = 0; x < nWorldWidth; ++x)
	for (int y = 0; y < nWorldHeight; ++y)
	{
		auto cellState = m_World[x][y].state;
		if (cellState == Cell::CONDUCTOR)
		{
			// Determine amount of neighbours that are heads
			int nHeads = 0;
			for (int headX = x - 1; headX <= x + 1; ++headX)
			{
				for (int headY = y - 1; headY <= y + 1; ++headY)
				{
					if (headX < 0 || headY < 0 || headX >= nWorldWidth || headY >= nWorldHeight) continue;
					else if (m_World[headX][headY].state == Cell::HEAD) nHeads++;
				}
			}

			// Switch state if nessecary
			if (nHeads == 1 || nHeads == 2) cellState = Cell::HEAD;
		}
		else if (cellState == Cell::HEAD) cellState = Cell::TAIL;
		else if (cellState == Cell::TAIL) cellState = Cell::CONDUCTOR;

		m_WorldLogic[x][y].state = cellState;
	}

// Copy output to array
for(int x = 0; x < nWorldWidth; ++x)
	for (int y = 0; y < nWorldHeight; ++y)
		m_World[x][y] = m_WorldLogic[x][y];
```

I added some basic controls, kept track of performance somewhat and added the ability to save and load simulations next. Finally, this was my result:

![Another Wire world screenshot](/img/wireworld/screenshot.png)

# Layers of complexity
I read a bit more about Wire World. The above screenshot was an example of an AND gate, and OR gates and XOR gates are trivial too.

![A Wire world OR gate](/img/wireworld/or.gif)
![A Wire world XOR gate](/img/wireworld/xor.gif)
![A Wire world AND gate](/img/wireworld/and.gif)

Just like how some people make mini CPUs inside Minecraft, often with fully capable APUs and sometimes even multiple cores, it turned out there was [an entire guide](https://www.quinapalus.com/wi-index.html) on creating a Turing-complete computer inside of Wire World! I dread to think about the amount of time it'd take to make such a thing, but it really is a testament to how a simple ruleset can create astonishingly complex things.

![A Wire world computer](/img/wireworld/computer.gif)

Hopefully, you've been inspired to make your own cellular automata. There's plenty of interesting ones to implement, or perhaps you could experiment with your own rules. Some sort of population on a world map sounds like an interesting idea, or perhaps a liquid in a cave, complete with foam and spray. It's always a fun thing to do as a proof of concept when working with a new language or machine.