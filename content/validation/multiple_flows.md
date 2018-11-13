---
title: "Multiple Flow Convergence"
date: 2018-11-10T16:26:54-05:00
toc: "true"
---


Given multiple bbr flows competing at the same bottleneck, all of the flows should sync. BBR has a mechanism to 
drain the queue and check for a lowest round trip time after some amount of seconds *since the last time the round trip time changed*. With multiple flows, they *all* should see the change in round trip time at the same time, so they should begin to drain the queue at the same time. 

The goodput (throughput seen by the receiver) should looks something like this result published by the BBR team: 

![Syncing of Multiple Flows](https://deliveryimages.acm.org/10.1145/3010000/3009824/figs/f8.jpg)

<!--more-->

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

## Simulating a 80mbps Link

> Note: our raspberry pi testbed is running a slightly newer version of bbr than the one used to generate the graphs in this paper. 

### TC: Traffic Control

TC is a tool that allows us to control the traffic leaving the interface towards the `tarta` subnet. In `TC`, 
there are a number of different *queueing disciplines* (qdisc) that can be used to control this traffic. We looked
at 4 different qdiscs to recreate the BBR results above. 

### Token bucket filters with netem delay

The token bucket filter (tbf) for tc creates a bucket of tokens that generate at some rate. Packets that arrive 
can only be sent using a token. This allows us to effectively control the rate of transmission for the interface to 
the desired 80mbps. 

To create the 10ms delay, we used [Netem](http://man7.org/linux/man-pages/man8/tc-netem.8.html), 
a network emulator that can be used to add delay and simulate loss on an otherwise perfect 
wired connection. 

The setup for these two qdisc can be done as follows: 

```sh
# delete old rules
sudo tc qdisc del dev enp3s0 root

# Add the netem delay
sudo tc qdisc add dev enp3s0 root handle 1:0 netem delay 10ms
# Add the tbf as a child to the netem
sudo tc qdisc add dev enp3s0 parent 1:1 handle 10: tbf rate 80mbit buffer 1mbit limit 1000mbit 

tc -s qdisc ls dev enp3s0
```

And, here are the results: 

![Netem + TBF](/80m_tbf.png)

Unfortunately, there are 'spikes' in the graph after every phase where BBR is draining the queue. This could be because we are using a slightly different version of BBR than the original paper used, but we believe the real cause is the token bucket filter. 

Because the token bucket filter fills up tokens at a constant rate and BBR underutilizes the link during the draining phase, the token bucket filter will generate a pool of tokens. So, when the first flow leaves the draining phase, it will be able to send all of its packets through this token bucket immediately as there is a surplus of tokens. This could be the cause of the spikes. 

That being said, the convergence to a fair throughput is promising. But, we continued to explore other queue disciplines to remove these bursts. 

### Netem with Rate Control

After reading the netem documentation, it looks like netem natively supports rate control now. So, we tried to do a similar test with netem to see if this reduced the burst issues of the token bucket filter. 

This was the setup: 

```sh
sudo tc qdisc del dev enp3s0 root
sudo tc qdisc add dev enp3s0 root handle 1:0 netem rate 80mbit delay 10ms
tc -s qdisc ls dev enp3s0
```

And this was the result: 

![Netem Rate Control](/80m_netem.png)

While the flows did seem to sync, the throughput was all over the place. Additionally, despite the rate being capped at 80mbps and the first flow starting 2 seconds before the next flow, the maximum observed throughput is only around 30mbps. I *think* this is because netem is not provided consistent rate limiting or latency. the man page does provide this quote: 

> The main known limitation of Netem are related to timer granularity, since Linux  is  not  a  real-time
> operating system.

which may indicate that the rate features are not completely stable (at least not at the precision we need). As this is a relatively new feature in netem, there is not much documentation on why we may be having these results. 

### TBF with peak rate limits

As 'burstiness' is a common problem with token bucket filters, there are ways built in to mitigate these issues. Specifically, you can set a *peak rate* for the token bucket. This should limit the *maximum rate* at which tokens can be used from the bucket. So, if this is set to or close to the bucket-filling rate, then this should eliminate the burst. 

Our setup: 
```sh
sudo tc qdisc del dev enp3s0 root

sudo tc qdisc add dev enp3s0 root handle 1:0 netem delay 10ms
sudo tc qdisc add dev enp3s0 parent 1:1 handle 10: tbf rate 80mbit buffer 1mbit limit 1000mbit peakrate 81mbit mtu 1500

tc -s qdisc ls dev enp3s0
```

Note: we tried the mtu at a number of sizes larger than and smaller than the mtu of our target interface (1500). 

Our Results: 

![Peak Rate TBF](/80m_tbf_peak.png)

> Aborted early

This trial resulted in zero throughput after the bucket was drained. We are still unsure why. We will continue exploring this. 

### Cake

[Cake](https://www.bufferbloat.net/projects/codel/wiki/Cake/) is a more sophisticated queuing discipline for shaping traffic. While the results were initially promising (the graphs looked good), we believed that Cake was doing *too much* to shape the traffic, such as limiting the rate per flow (even with the settings off). 

As this would make the results of our experiments harder to reason about, and thus effectively worthless for validating our test bed, we decided to not pursue cake further. 
