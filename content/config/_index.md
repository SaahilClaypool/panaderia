---
title: "Config"
date: 2018-11-10T16:26:47-05:00
---

## The Setup

The test bed is comprised of 8 raspberry pi B+ computers, two switches and an ubuntu desktop machine acting as a router. Each switch is connected to a bay of 4 raspberry pi's in a small rack, and to a a NIC card on the router. The router is configured to setup these two racks as different subnets where the Pis on the first NIC card are assigned IP addresses 192.168.1.0/24 and the Pis on the second NIC card are assigned addresses 192.168.2.0/24. 
