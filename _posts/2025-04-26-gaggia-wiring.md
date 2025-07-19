---
layout: post
title:  "Nodal Analysis of Gaggia Classic Pro"
---

I see a lot of reddit posts on /r/gaggiaclassic about electrical and wiring issues.

Figure 1, the manufacturer's wiring diagram.

The manufacturer's wiring diagram is an essential resource for electrical troubleshooting and modifications, but it can be difficult to understand.

Figure 2, the image from the great blog post.

To understand how the system functions, there's a great diagram (pictured above) in a [blog post](https://comoricoffee.com/en/gaggia-classic-pro-circuit-diagram-en/) by "naru" over at comoricoffee.com.
I find this a lot easier to understand from a functional perspective.

Figure 3, my diagram.

I created this diagram after trying to do [nodal analysis](https://en.wikipedia.org/wiki/Nodal_analysis) in my head several times and often losing track halfway through.
The image is based off the official wiring diagram, so one can identify particular wires and connections during troubleshooting.
This also means that even if I've somehow botched the annotations, any errors should be discoverable.
It's not as easy to understand as naru's diagram, but it has more information about the physical wiring of the machine.

For reader's not familiar with nodal analysis, it's a really powerful concept for understanding (and troubleshooting) electrical circuits.
Nodes can be defined as continuous sections of a circuit that are not separated by a resistor.
To define them, start highlighting a wire, and stop when you hit a resistor or switch.
Continue until all wires in the circuit are assigned to a node.

No heat?
That means there's (close to) zero difference in voltage across the terminals of the heating element.
Steam temperature lamp on?
That means there is a substantial difference in voltage across the terminals of the lamp.


### Examining the system

Let's walk through the diagram briefly.

Closing the on/off switch (S1) connects the red node (AC live) to the pink node.

It also connects the dark gray node (AC neutral) to the dark brown node.
If the thermal fuse (S6) is intact, the light brown node is also tied to neutral.

If we trace the neutral node, we quickly see there are three parallel paths between neutral and other nodes: the on/off lamp, the brew switch, and the boiler.

### The Lamp

The on/off lamp (RL1) is pretty simple.
It connects the pink node, which we've already established is live, to the dark brown node (neutral).
Note that this difference in potential is not contingent on the integrity of the thermal fuse (S6).

### The Brew Switch

We see that when the brew switch (S2) is closed, light gray node will be connected to neutral (if the thermal fuse is intact).
The light gray node is in turn connected to terminals of both the solenoid valve (RS) and the pump (RP).

The opposite terminal of the pump is connected to the pink node, meaning current will flow between live and neutral and energize the pump.

The opposite terminal of the solenoid is connected to the purple node.
The purple node is connected to the pink node when the steam switch (S3) is closed.
If the steam switch is not activated and the brew switch is activated, the solenoid will energize and allow water to flow from the group head.

### The Boiler



Since the brew thermostat (S4) is normally closed at ambient temperatures, the orange node will be connected to the pink node, and in turn AC live.

The orange node can connect to the yellow node 






You may be interested to see that when the system is on, the path from live to neutral *always* runs through the heating element. So what causes the boiler to decrease power when the system reaches temperature? The lamps (RL3

When the brew thermostat (S4) is 

The steam thermostat (S5) is closed at ambient and brew temperatures. When it is closed, it shorts the yellow node to the orange node, bypassing the steam temp lamp (RL3). When the system reaches steam temperature and S5 opens, the short is broken and current must flow through 

 As an example, imagine your boiler isn't heating water. What you've observed is that there is not a substantial difference in voltage between the yellow net, 



Closing the on/off switch connects the red node (AC live) to the pink node, as well as connecting the dark gray node (AC neutral) to the dark brown node. Assuming the thermal switch is intact, the light brown node is also connected to neutral.


