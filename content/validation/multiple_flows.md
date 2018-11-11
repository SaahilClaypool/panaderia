---
title: "Multiple Flow Convergence"
date: 2018-11-10T16:26:54-05:00
---


Given multiple bbr flows competing at the same bottleneck, all of the flows should sync. BBR has a mechanism to 
drain the queue and check for a lowest round trip time after some amount of seconds *since the last time the round trip time changed*. With multiple flows, they *all* should see the change in round trip time at the same time, so they should begin to drain the queue at the same time. 

The goodput (throughput seen by the receiver) should looks something like this result published by the BBR team: 

![Syncing of Multiple Flows](https://deliveryimages.acm.org/10.1145/3010000/3009824/figs/f8.jpg)

<!--more-->

## TC: Traffic Control

This was evaluated using a single 80 megabit per second link with a delay of 10 milliseconds. 

The tc (traffic control command line utility) settings for this are as follows: 
```sh
sudo tc qdisc del dev enp3s0 root
sudo tc qdisc add dev enp3s0 root handle 1:0 netem delay 10ms
# In the line below, rate is standard rate. burst is amount of tokens available, limit is the queue
sudo tc qdisc add dev enp3s0 parent 1:1 handle 10: tbf rate 80mbit buffer 1mbit limit 1000mbit 

tc -s qdisc ls dev enp3s0

# https://netbeez.net/blog/how-to-use-the-linux-traffic-control/
# https://wiki.linuxfoundation.org/networking/netem

tc -s qdisc ls dev enp3s0
```

This creates a new rule called **dev**, adds a delay of 10ms. Then, we create a token bucket filter (tbf) with for an 80mbit connection speed. The token buffer (amount of tokens available as a burst) is 1mbit, and the limit or queue size of the router is 1000mbit. 

These rules are *only* applied on the packets *leaving* the enp3s0 interface (outbound to the `tarta` subnet).

## Raspberry Pi 

Each client (churro 1 - 4) and server (tarta 1 - 4) is set to use BBR as their congest algorithm. This can be done with the commands:

```sh
sudo sysctl net.core.default_qdisc=fq;
sudo sysctl net.ipv4.tcp_congestion_control={cc};
sudo sysctl net.ipv4.tcp_congestion_control;
```

## Iperf3 

We create four iperf3 flows from the `churro` subnet to the `tarta` subnet. The bulk sender (the iperf3 client) connects from churro1 to tarta1 for each pair. 

Server: 
```sh
iperf3 -s
```

Client on each churro: 
```sh 
iperf3 -c tarta<n> -t 120
```


## Results

> Note: our raspberry pi testbed is running a slightly newer version of bbr than the one used to generate the graphs in this paper. 

We notice similar results, except that while most flows sync up, some flows do not. So, we three flows begin to drain the queue, the other flow will take the available buffer space. 

![Panaderia test bed results](/pana_bbr_compete.png)