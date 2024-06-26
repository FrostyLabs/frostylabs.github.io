---
title: DEBIAN CONFIGURE IP ADDRESS AND VLANS
author: frosty
date: 2022-09-29 12:00:00 +0000
categories: [linux]
tags: [dev]     # TAG names should always be lowercase
---

Hello Everyone,

Recently at work, I needed to quickly configure IP addresses and VLANs. This is a quick page as a cheat-sheet on how you can apply the configuration on debian-based (perhaps more as well with the tools) systems using the terminal. This guide has both ad-hoc configurations, and configurations which are persistent. This guide will not use vconfig which is commonly used, however going to be depricated.

*Note that for this configuration you will need root access or at least permission to change interface configuration on the system.*

## Ad-Hoc Solution

Generally there are very little commands required but lets go through it:

```bash
# Create the virtual interface for eth0 with VLAN 1288
ip link add link eth0 name eth0.1288 type vlan id 1288

# Set the IP address for the virtual eth0.1288 interface
ip addr add 172.16.10.5/26 dev eth0.1288

# Bring the virtual eth0.1288 interface up
ip linkset dev eth0.1288 up

# Delete interface after use
# Will also be cleaned after a reboot
ip link delete eth0.1288
```

## Persistent Solution

For this you need to edit the `/etc/network/interfaces` file. Add this VLAN block to the end of the file.

```bash
$ vi /etc/network/interfaces

# auto eth0.1288
iface eth0.1288 inet static
  address 172.16.10.5
  netmask 255.255.255.192
  gateway 172.16.10.1
  dns-nameservers 172.16.10.2 8.8.8.8 8.8.4.4
```

You should restart the network interface or reboot the system:

```bash
ifup eth0.1288
```

## Challenge for you!

Create a script to configure the VLAN interface and addresses using a bash script with commandline options.
