---
title: "Adding congestion control protocols"
date: 2018-11-25
weight: 15
toc: "true"
---

The linux kernel comes with a number of different congestion control algorithms. To test new congestion control methods we to be able to:

1. Dynamically select new protocols
2. Recompile and install new protocols

<!--more-->

# Selecting an protocol

For any protocol that has already been installed as a kernel module, it should be possible to set that protocol as the current default using the following: 

```sh
sudo net.ipv4.tcp_congestion_control=cubic
sudo sysctl -p
sudo sysctl net.ipv4.tcp_congestion_control # check the active protocol
```

# Adding or Modifying Source code additions

The goal here is to A) create new protcols or B) install multiple versions of the same protocols with different names. This will allow user to switch between protocols without rebooting the client machines. 

The general process is as follows:

1. Add or modify a new file int `/net/ipv4/` for the congestion control protocol

    This file should contain the main code for the congestion control protocol. 

    The important parts are defined in the `struct tcp_congestion_ops`. This is defined in `include/net/tcp.h`. This specifies the init function and the main function and event handlers for this protocols.

2. Create an 'info' struct to hold the state in `include/uapi/linux/inet_diag.h`

    This must be added to the `union tcp_cc_info` in `include/uapi/linux/inet_diag.h`.

    note: if you are modifying the bbr stack, you can probably just reuse the existing bbr struct

3. Modify the `net/ipv4/Kconfig` 

    I *believe* this is used to to specify which modules should be built into the image. So, this is force the new modules you create to be compiled.

4. Add the .o file to `net/ipv4/Makefile`

    the line should look like: `obj-$(CONFIG_TCP_CONG_BBR2) += tcp_bbr2.o`


> As kernel version changes, the old bbr code has to be modified to work with the newer kernel api. So, running multiple different versions of BBR in the same kernel is difficult.

# Installing a kernel module

> Note: it should be possible to *just* install the kernel module for the new congestion control protocol

# User space congestion control

This [SIGCOMM Paper](https://akshayn.xyz/res/ccp-sigcomm18.pdf) published by Akshay Narayan et. al. at MIT discusses using a user space agent to implement congestion control protocols. This has interesting implications on the development of these protocols.