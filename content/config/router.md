---
title: "The Router"
date: 2018-11-10T16:45:46-05:00
weight: 10
toc: "true"
---

The router is an ubuntu desktop with 3 nic cards. One card is used for the normal 'wide area network', and the other two cards are used to route between the two raspberry pi subnets. This is done using netplan on ubuntu, and editing the dhcpd.conf to set up static ip addresses. Also, the host files for each machine are edited to make tests more convenient. 

<!--more-->

# Netplan 

Netplan is a tool for configuring networks used by systemd linux machines. This is used to configure the *routes* that are advertised to nearby machines. So, when a machine trying to connect to an IP 192.168.1.2, the ubuntu router is able to redirect this request through the ethernet port enp3s0 because the `route` variable is set to handle all `192.168.1/24` addresses. This also is used to set the gateway ip for the interface to `192.168.1.100`

Note that netplan does *not* assign ip addresses to each of the machines. This must be done using a separate DHCPD configuration file (shown below). 
<details><summary>/etc/netplan/01-network-manager-all.yaml (click to show)</summary>
<p>

```yaml
# Let NetworkManager manage all devices on this system
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp3s0: # bay 1 - tarta
      dhcp4: no # do not assign an ip to this interface through dhcp. We will use a static IP in dhcpd.conf
      dhcp6: no
      match:
        macaddress: 00:0a:f7:0e:ff:c3 # Make sure this is the interface we really want
      addresses:
        - 192.168.1.100/24 # the ip this interface will be assigned
      routes:
        - to: 192.168.1.0/24 # advertise the routes for this interface
          via: 192.168.1.100 # the gateway ip for this subnet
          metric: 100 # how high the priority is for this route
    enp5s0: # bay 2 - churro
      dhcp4: no
      dhcp6: no
      match:
        macaddress: 00:0a:f7:16:80:7f
      addresses:
        - 192.168.2.100/24
      routes:
        - to: 192.168.2.0/24
          via: 192.168.2.100
          metric: 100
    enp4s0: # WAN
      match:
        macaddress: 5c:f9:dd:6a:e7:77
      dhcp4: true
      dhcp6: true
      routes:
        - to: 0.0.0.0/0
          via: 0.0.0.0/0
          metric: 50


```

</p>
</details>


# DHCPD configuration

DHCP handles the dynamic host configuration. The `/etc/dhcpd.conf` file configures the dchp daemon that is used to assign ip addresses to each host connecting to the subnet. 

<details><summary>/etc/dhcpd.conf (click to show)</summary>

```conf
# dhcpd.conf

# how long each dns configuration is kept before it must be reassigned
default-lease-time 3600;
max-lease-time 4800;

ddns-update-style none;

authoritative; # make dns authoritative

# declare the subnet for the tarta cluster
subnet 192.168.1.0 netmask 255.255.255.0 {
    option routers 192.168.1.100; # the router must be the ip for the interface configured in the netplan config
    option subnet-mask 255.255.255.0; # the mask for the network 
    range 192.168.1.1 192.168.1.99; # the possible ip addresses assigned. Note: the 192.168.1.0 addresses are usually reserved. 
    # Assigned the router to 192.168.1.0 causes many issues. Just use 192.168.1.100
}

# declare subnet for churro cluster
subnet 192.168.2.0 netmask 255.255.255.0 {
    option routers 192.168.2.100;
    option subnet-mask 255.255.255.0;
    range 192.168.2.1 192.168.2.99;
}

# declare static ip for each churro and tarta (optional)
host churro1 {
   hardware ethernet b8:27:eb:4d:07:f3;
   fixed-address 192.168.2.1;
}

host churro2 {
   hardware ethernet b8:27:eb:8e:ac:f8;
   fixed-address 192.168.2.2;
}

host churro3 {
   hardware ethernet b8:27:eb:0a:a7:cc;
   fixed-address 192.168.2.3;
}

host churro4 {
   hardware ethernet b8:27:eb:a9:e9:06;
   fixed-address 192.168.2.4;
}

host tarta1 {
   hardware ethernet b8:27:eb:4b:63:e9;
   fixed-address 192.168.1.1;
}

host tarta2 {
   hardware ethernet b8:27:eb:80:e5:b6;
   fixed-address 192.168.1.2;
}

host tarta3 {
   hardware ethernet b8:27:eb:2d:c6:54;
   fixed-address 192.168.1.3;
}

host tarta4 {
   hardware ethernet b8:27:eb:e2:43:b5;
   fixed-address 192.168.1.4;
}
```

</details>

The ip and mac address can be checked with the `ip -a` command.

# host configuration

For convenience, each raspberry pi is given the location of the name server and each other raspberry pi manually. This
is done by editing the `/etc/hosts` and the `/etc/resolv.conf` host file.

<details><summary>/etc/hosts and /etc/resolv.conf (click to show)</summary>
```conf
127.0.0.1       localhost
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters

127.0.1.1       churro1
192.168.1.1     tarta1
192.168.1.2     tarta2
192.168.1.3     tarta3
192.168.1.4     tarta4
192.168.2.1     churro1
192.168.2.2     churro2
192.168.2.3     churro3
192.168.2.4     churro4
```
and the resolv.conf (which sets the name server so the raspberry pi can reach outside sources)


```conf
nameserver 8.8.8.8
```
</details>


# systl.conf

Also, the sysctl.conf file must be changed to allow ipv4 forwarding. This can done by setting the variable `net.ipv4.ip_forward=1`. 

The file we use for the `/etc/sysctl.conf` is below. The modified variables are: 

- net.ipv4.ip_forward=1
- net.core.default_qdisc=fq
- net.ipv4.tcp_congestion_control=cubic 

    cubic is a safe default congestion control

- net.core.rmem_max=2097152

    The next all modify the memory allocated to the sockets to maximize throughput. This requires some manually tweaking per system.
    see [this site](https://wwwx.cs.unc.edu/~sparkst/howto/network_tuning.php) for more.

- net.core.wmem_max=2097152
- net.core.rmem_default=32768
- net.core.wmem_default=358400
- net.ipv4.tcp_rmem=2048 262144 4194304
- net.ipv4.tcp_wmem=10240 262144 4194304
- net.ipv4.tcp_mem=4194304 4194304 4194304
- net.ipv4.route.flush=1

<details><summary>/etc/sysctl.conf</summary>

```sh
#
# /etc/sysctl.conf - Configuration file for setting system variables
# See /etc/sysctl.d/ for additional system variables.
# See sysctl.conf (5) for information.
#

#kernel.domainname = example.com

# Uncomment the following to stop low-level messages on console
#kernel.printk = 3 4 1 3

##############################################################3
# Functions previously found in netbase
#

# Uncomment the next two lines to enable Spoof protection (reverse-path filter)
# Turn on Source Address Verification in all interfaces to
# prevent some spoofing attacks
#net.ipv4.conf.default.rp_filter=1
#net.ipv4.conf.all.rp_filter=1

# Uncomment the next line to enable TCP/IP SYN cookies
# See http://lwn.net/Articles/277146/
# Note: This may impact IPv6 TCP sessions too
#net.ipv4.tcp_syncookies=1

# Uncomment the next line to enable packet forwarding for IPv4
net.ipv4.ip_forward=1

# Uncomment the next line to enable packet forwarding for IPv6
#  Enabling this option disables Stateless Address Autoconfiguration
#  based on Router Advertisements for this host
#net.ipv6.conf.all.forwarding=1


###################################################################
# Additional settings - these settings can improve the network
# security of the host and prevent against some network attacks
# including spoofing attacks and man in the middle attacks through
# redirection. Some network environments, however, require that these
# settings are disabled so review and enable them as needed.
#
# Do not accept ICMP redirects (prevent MITM attacks)
#net.ipv4.conf.all.accept_redirects = 0
#net.ipv6.conf.all.accept_redirects = 0
# _or_
# Accept ICMP redirects only for gateways listed in our default
# gateway list (enabled by default)
# net.ipv4.conf.all.secure_redirects = 1
#
# Do not send ICMP redirects (we are not a router)
#net.ipv4.conf.all.send_redirects = 0
#
# Do not accept IP source route packets (we are not a router)
#net.ipv4.conf.all.accept_source_route = 0
#net.ipv6.conf.all.accept_source_route = 0
#
# Log Martian Packets
#net.ipv4.conf.all.log_martians = 1
#

###################################################################
# Magic system request Key
# 0=disable, 1=enable all
# Debian kernels have this set to 0 (disable the key)
# See https://www.kernel.org/doc/Documentation/sysrq.txt
# for what other values do
#kernel.sysrq=1

###################################################################
# Protected links
#
# Protects against creating or following links under certain conditions
# Debian kernels have both set to 1 (restricted) 
# See https://www.kernel.org/doc/Documentation/sysctl/fs.txt
#fs.protected_hardlinks=0
#fs.protected_symlinks=0
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=cubic
net.core.rmem_max=2097152
net.core.wmem_max=2097152
net.core.rmem_default=32768
net.core.wmem_default=358400
net.ipv4.tcp_rmem=2048 262144 4194304
net.ipv4.tcp_wmem=10240 262144 4194304
net.ipv4.tcp_mem=4194304 4194304 4194304
net.ipv4.route.flush=1
```

</details>