---
title: "Experimental Results"
date: 2019-01-24
toc: "true"
---

# Experimental Results

Our focus was on determining exactly why BBR behaves the way it does in shallow buffers,
and when competing with CUBIC. 

## Shallow Buffers

BBR has had a notorious problem dealing with shallow buffers. Namely, this is because 
BBR does not use loss as a congestion metric. So, when it sends too much data, it does
does not slow down its sending rate accordingly. 

> Note: in more recent proposals, there is talk of "creating headroom" when there is loss. But, we do not
> think that is the root of the issue.

### Standing Queue

In theory, BBR operates at Kleinrock's optimal operating point. That is, BBR *should* cap the inflight data 
to exactly the BDP. That, combined with the fact that BBR *paces* its sending rate to match the estimating
throughput, should ensure that the bottleneck router never builds up any queue. 

But, in practice, this does not happen. BBR slightly over estimates the throughput when there is more than a
single flow in the network. Additionally, BBR dooes *not* cap the inflight data to just 1BDP. Rather, 
BBR will allow up to 2 * BDP to be inflight at any point. This is to ensure that when ACKs are aggregated 
(such as on most mobile networks), BBR is still able to send enough packets to fully utilize the network. 

This 2 BDP CWND means that, if one BDP is 'on the wire' then the other BDP's worth of packets must queued 
at the bottleneck router. This means that BBR *should* create a standing queue of 1 BDP. To confirm this, 
we ran a trial on our testbed. 

![4 BBR flows competing with a large buffer](/results/large_bbr_4_result_queue_(bytes).svg)

We ran this trial with 4 BBR flows, a 200,000 byte router queue, 25ms RTT, and and 80mbit throughput. 
As with all our tests, we recorded the performance of the flows tracked the router queue sizes. 

Sure enough, the queue size is *constantly* at around 28,000 or more bytes. Given our throughput
and RTT settings, this corresponds to **1.12** times the BDP. And, because the router 
can send 1 Bandwidth's worth of packets per second (by definition of bandwidth), this 
constant router queue corresponds to 1 RTT_Prop, where RTT prop is the minimum 25ms delay. 
This increases the minimum delay by 100%. 

In this test, the RTT is fairly low, so a one BDP router buffer adds *only* 25ms of extra delay. But, 
imagine a satellite link with a 500ms delay - this one BDP buffer would add an *additional* 500ms delay!

As one BDP scales with both throughput and latency, high throughput - high latency connections
have the largest BDP. This means that BBR will create larger router queues on these large BDP networks.
So, what happens when the router queue is *smaller* than the 1 BDP mark? 

### 'Shallow' Buffers: Less than one BDP 

For BBR, a shallow buffer is *proportional to the bdp* as the standing queue created at the router is itself
proportional to the BDP. When the maximum router queue is *less* than the BDP, then BBR creates enormous amounts
of loss. 

We found this by doing the following: We ran different numbers of BBR flows competing at the same bottleneck
router at different throughputs and for different buffer maximum queue lengths. For each of these, we recorded
the loss rate of the *steady state* part of the BBR connection (we ignore startup, as BBR tends to create even more
loss during its startup). The results of 4 BBR flows competing for 80mbit throughput on a 25ms latency 
with different max router queues is shown below. 

![Drop Rate](/results/bbr_4_80_drop_rate.svg)

This shows that shallow buffers, any amount smaller than 1.5 BDP, BBR creates a *huge* (up to 12%) amount of loss. 
The fact that the queue must be at least 1.5 * BDP is due to the fact that BBR always over estimates the 
throughput when competing with other flows. This causes the inflight CWND to be slightly too large. 

For reference, here is the loss rate of 4 cubic flows in the same conditions: 

TODO: maybe rerun with just cubic to show it never gets much loss...

We ran this same test with 40mbit, 80mbit, and 120mbit. And, the shape of the curve is always the same - the
loss rate is extremely high until the queue is at least 1.5 BDP. 


## Competing with Cubic

Because CUBIC is a loss-based congestion control protocol (and the most common congestion control protocol), 
BBR's high loss rate in certain conditions can be problematic. 

### Shallow Buffers: Loss Ignorance

When the buffers are smaller than 1.5 BDP, BBR will create a large amount of ambient loss by itself. 
This high loss will cause CUBIC to back off and thus BBR will dominate the throughput. We tested this in a similar 
manner: we ran 2 CUBIC flows against 2 BBR flows on 40, 80, and 120mbit links with varying queue sizes
proportional to the BDP. 

![Shallow BBR vs Cubic](/results/80m_cubic_bbr_shallow_throughput.svg)


As shown in this graph of a buffer that is 125000 bytes (roughly 0.5 BDP), BBR will dominate. This makes sense when 
we look at the drop rate percent per second, which is very high: 

![Shallow BBR vs Cubic Droprate](/results/80m_cubic_bbr_shallow_droprate_125000.svg)

### Medium Buffers: Cyclic Performance


While unfairness bad, a more problematic result is the competition between CUBIC and BBR 
in *meidum sized* buffers. Namely, because CUBIC operates with a full router queue, BBRs
transition between probing the bandwidth (which causes loss) and draining the queue (which 
allows CUBIC to recover lost throughput), cause a extremely cyclic performance
between the throughput shares for BBR and Cubic. This is unstable network condition is very bad
for network applications. 

This plot throughput for BBR and CUBIC competing for a 1.75 BDP buffer. 

![Cyclic Throughput](/results/80m_bbr_cubic_cyclic_throughput_437500.svg)

So, why does this happen? 

To better understand the behavior, we can look at the loss rate over the same time frame: 

![Cyclic Droprate](/results/80m_bbr_cubic_cyclic_droprate_437500.svg)

The cycle has a length of 20 seconds. This is not a coincidence - BBR drains
the que every *10* seconds. The 'valley' in drop rate where it is relatively
low correspond to CUBIC flows (blue and orange) being relatively 'on top'.
This can be seen for the 10 seconds 80 - 90 seconds into the trace.

After 10 seconds, BBR will drain *its queue* for the *minimum RTT*. Because BBR is 'squashed', it does not 
have many packets in the queue, so removing its packets does not change the RTT. BBR will thus measure
a relatively high RTT, and thus predict the BDP to be much too large. This causes BBR to ramp up 
quickly and create a large standing queue, as the queue BBR creates is proportional to the BDP it estimates. 

This causes a large spike in loss, seen around second 100 in the drop rate plot. This loss in turn causes
CUBIC to decrease its CWND to almost nothing as CUBIC responds very quickly to loss. BBR will continue to 
dominate the throughput for 10 seconds, where it will drain the queue again to estimate the minimum RTT. BBR
will have the majority of packets in the queue as CUBIC still has a small CWND, so draining the queue will 
find the *TRUE* rtt prop value. This will be much smaller than the previous estimated rtt prop value.

Because the estimated rtt is small again, BBR will decrease its CWND accordingly. This will allow CUBIC to 
ramp up again as there is now free space in the router queue that can be taken without causing loss. 
Thus, the cycle will repeat, as the next time BBR drains the queue, it will be unable to drain CUBIC's traffic 
to estimate the RTT. 

### Large Buffers: CWND Strangling

When the buffer is very large, the cyclic performance continues, but BBR will 
obtain less of the network share. This is because BBR will still cap its CWND
proportionally to the BDP, and the queue size is much larger than the BDP. 
This will cause slightly lower drop rates, and thus slightly higher cubic performance. 

The plot below shows this behavior when CUBIC and BBR are competing for a 
2 BDP router queue. 

![bbr vs cubic cyclic throughput](/results/80m_bbr_vs_cubic_cyclic_throughput_200000.svg)


### Summary

The proportion of throughput for BBR and CUBIC in terms of the router queue length can be seen here: 

![bbr vs cubic quartile throughput](/results/q_throughput.svg)

This plot shows the 25% percentile to 75% throughput for the total BBR throughput and the total CUBIC 
throughput as a function of the queue size. As seen here, the *range* in throughput is huge. 
This is caused by the cyclic performance between CUBIC and BBR. At any queue size less than 1.5 BDP, 
BBR tends to dominate cubic as there is no cyclic performance. This can be seen by the relatively
tight quartiles. 

At around 5 BDP, the large buffer causes CUBIC to dominate the performance, but the huge range
in performance indicates that BBR and CUBIC still have a cyclic behavior where one will dominate for some
period of time before the other takes the majority of the bandwidth.


## What can BBR do Better?

