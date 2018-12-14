---
title: "Improving the Experiment setup"
date: 2018-12-13
weight: 30
---

Being able to quickly run *and rerun* experiments is vital in ensuring that the tweaks we make to our testbed are having the effects that we expect. Specifically, we needed to be able to: 

1. Setup the router with the correct parameters

    Increasing or decreasing the bandwidth, latency, buffer size etc.

2. Orchestrate the raspberry pi startup

    This involves starting up `N` flows per raspberry pi, and measuring their performance over the course of the experiment session.

    And, for each raspberry pi, needed to start a packet capture to analyze the results post experiment.

3. Poll the router as the experiment progresses 

    This allows us to see the 'ground truth' for how many packets are in flight or dropped. (Currently, we only keep track of the router queue size).

4. Pull in all of the captures to the 'router'

5. Parse the captures into CSV files and create consistent plots


By wrapping all of these procedures in a single script, we are able to quickly change configurations and create plots in an *identical* manner. The reproducibility of this setup is key in allowing us to compare results across different configurations in a consistent manner. 

<!--more-->

Our experiment 