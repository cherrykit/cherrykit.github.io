---
title: "Training Connect 4 Agents Against an Optimal Agent"
date: 2023-05-04
description: Training Connect 4 Agents Against an Optimal Agent
menu:
  sidebar:
    name: Connect 4 Agents
    identifier: connect4
    weight: 10
# tags: []
# categories: []
---

In a group of three people. we wrote Temporal Difference Learning (TDL) agents for Connect 4 as part of the “Introduction to Artificial Intelligence” course at the University of Waterloo (CS486). The code is built on top of an existing Java implementation of the Connect 4 game, including an optimal agent. In addition to training with self-play, we modified the TDL algorithm to allow training against this existing optimal agent.

The code for our project can be found [here](https://github.com/cherrykit/Connect-Four). We wrote a course paper on the project, which can be found [here](connect4.pdf).