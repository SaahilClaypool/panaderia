---
title: "Config"
date: 2018-11-10T16:26:47-05:00
---

## The Setup

The test bed is comprised of 8 raspberry pi B+ computers, two switches and an ubuntu desktop machine acting as a router. Each switch is connected to a bay of 4 raspberry pi's in a small rack, and to a a NIC card on the router. The router is configured to setup these two racks as different subnets where the Pis on the first NIC card are assigned IP addresses 192.168.1.0/24 and the Pis on the second NIC card are assigned addresses 192.168.2.0/24. 

For convenience, the host names for the 192.168.1.0/24 subnet are 'tarta1' through 'tarta4' and the other subnet contains 'churro1' through 'churro4'.

```
tarta1 --+
         |
tarta2---+    +--------+
         +----| switch |
tarta3---+    +-+------+
         |      |      
tarta4---+      |      
                |      +---------------+
                +------+ NIC 1         |
                       |   Horno  NIC 3|--> WAN
                +------+ NIC 2         |
                |      +---------------+
churro1--+      |      
         |      |      
churro1--+    +-+------+ 
         +----| switch |
churro1--+    +--------+
         |
churro1--+

```

This setup allows us to imitate a real server-client experience. One subnet serves as the servers and host multiple connections to the clients in the other subnet. Every connection will be routed through the **Horno** machine, allowing us to apply arbitrary network conditions to simulate different network conditions. 
