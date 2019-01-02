---
title: "Background"
date: 2018-11-10T16:27:33-05:00
---

## Congestion Control

*TCP Congestion Control* is responsible for regulating the rate and amount of packets that your computer sends. This is important for regulating the total amount of data in flight in a network  - if every host sends as much data as possible, the routers in the middle will not be able to handle the traffic and thus drop packets. 

Traditionally, TCP has used Loss based congestion control, such as [TCP Cubic](https://en.wikipedia.org/wiki/CUBIC_TCP). Generally, these schemes keep sending more data until they see loss in the network. This means that they have hit the maximum throughput of the network, and thus they back off slightly. While this is generally effective at maximizing throughput, this puts unnecessary strain on networks because there are more packets in flight than needed to fully utilize the network. 

Routers have queues or buffers to handle excess packets - when a router receives packets at a rate greater than it can retransmit, it will queue these packets into this buffer. When this buffer is filled, the router will simply drop this packet. Because loss based schemes wait until this dropping point to slow down their sending rate, they will *always* increase until the queue is completely full. If the queues are large, this can greatly increase the round trip time of packets, and reduce the networks' resilience to traffic bursts. 

## Delay Based Congestion Control

Delay based control aims to fix this problem by attempting to send at the maximum rate possible *without* filling the intermediate buffers. As the number of packets sent by an application increases, its throughput will increase up until the bottleneck routers operating point. Then, these packets will just be queued into the router's buffer or dropped. As the router queue becomes filled, the round trip time for each packet will be increased as they must now wait in this filled queue. This relationship is depicted below.

![BBR: Optimal Operating Point](/optimal_operating.png)
*[BBR: Congestion-Based Congestion Control. Neal Cardwell et. al](https://queue.acm.org/detail.cfm?id=3022184)*

# BBR

BBR (bottleneck bandwidth and round-trip propagation time) is *not* strictly a loss based or delay based congestion, but rather a congestion based protocol that attempts to *infer* the optimal sending rate for an application by estimating two parameters: the perceived bandwidth and the round trip time. Using these parameters, BBR should also operate at the optimal sending rate shown above.

In a perfect world, the optimal operating point is at the BDP (bandwidth delay product) which calculates the maximum packets that can be transmitted through a network without excess delay. But, operating at this point reliably in practice is still an active area of research. 

## Background Work