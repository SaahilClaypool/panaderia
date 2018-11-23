---
title: "Single flow"
date: 2018-11-23T10:01:37-05:00
toc: "true"
---

A single bbr flow should display a very regular cycle. Every 8 rtt's (round trip times) bbr should probe the bandwidth by increasing its sending rate, drain any queue that is built, and then return to a steady state. Every 10 seconds, the flow should completely drain the queue by throttling its speed, and then return to a normal sending rate. 

<!--more-->

# BBR 1.0 Behavior 

The typical behavior should look like this [^1]: 

![Typical Probe Bandwidth State](/bbr_typical_behavior.jpg)

We aimed to validate our raspberry pi testbed by recreating the experimental conditions defined in an ACM paper[^2] by Neal Cardwell and company. The scenario involves a single flow on a 10mbps link with a round trip time of 40 seconds. BBR's throughput and the amount of inflight data should look like this: 

![inflight data for changing bandwidth](/google_change_throughput.png)

# Increasing Bandwidth

In our testbed, we were able to generate this graph mirroring these results: 

![Our changing latency results](/single_changing_throughput.png)

The first plot depicts the amount of packets in flight as the bandwidth is increased from 10mbps to 40mbps at the 20 second mark. The steady state just before the 20 second mark does look similar to the typical bbr behavior; each .30 seconds, bbr seems to poll the max bandwidth by increasing its sending rate, and thus the amount of in flight packets. This in turn fills the queues and increases the net round trip time. BBR then drains the queues by decreasing the round trip time until the rtt returns to the previous level. 

But, at second 20 when the 10mbps link is lifted to a 40mbps link, the behavior deviates from the behavior previously described. Rather than increasing the inflight packets through a series of probe and drain cycles, the amount of inflight packets seems to exponentially increase until the queues begin to fill again. At first we considered this an oddity from our test bed (as token bucket filters are fiddly beasts), but after reading the slides [here][3], it seems that this is a change in how BBR functions. 


## BBR 2.0 Behavior

As discussed in the slides from the IETF 102, the BBR team is changing the 'probe' phase of the bbr cycle. Rather than increasing the 'gain' value at each cycle, the new behavior is to exponentially increase the inflight packets until the amount of in flight packets is too high (indicated by larger queue sizes, and thus fewer acked packets). This behavior is depicted in this rendered graph from that slide deck: 

![BBR 2.0 Ramp Up Behavior](/bbr2.0_ramp_up.png)

Because we are running a slightly modified Linux Kernel 4.19, we have a newer version of BBR than the previously mentioned behavior evaluates. It would seems that the behavior we see is much closer to BBR 2.0. 

# Decreasing Bandwidth

At 40 seconds, the bandwidth in our experiment is decreased from 40mbps to 20 mbps. The expected BBR 1.0 RTT and inflight packets are shown in the second second of 'figure 3' that is linked above. Until BBR hits the limit of inflight packets (determined by the cwnd_gain), the sender will continue to send packets at a rate faster than the bottleneck can handle. This will increase the inflight packets and the round trip times as the router queues begin to fill. After hitting this limit, BBR will decrease the sending rate and cycle down to return to the optimal RTT (at relatively empty queues). 

Our single measured bbr flow looks like this: 

![Decreasing Bandwidth](/decrease_throughput.png)

This *very* closely mirrors the bbr 1.0 and 2.0 behaviors and reaffirms that our setup is likely working as intended.


[^1]: [BBR: Congestion-Based Congestion Control](https://cacm.acm.org/magazines/2017/2/212428-bbr-congestion-based-congestion-control/fulltext)

[^2]: [BBR: Congestion-Based Congestion Control Measuring bottleneck bandwidth and round-trip propagation time](https://queue.acm.org/detail.cfm?id=3022184)

[3]: https://datatracker.ietf.org/meeting/102/materials/slides-102-iccrg-an-update-on-bbr-work-at-google-00 

