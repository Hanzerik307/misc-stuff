---
title: Simple Bridge Using Networkd
author: Hanzerik307
date: May 26, 2026
---

# Intro
These are the simple instructions for setting up a network bridge using `systemd-networkd` on Debian 13 (Trixie)

# What you will need

* We're assuming you have installed Debian using the NetInstall image and very minimal (Standard System Utilities and SSH)
* Some Type of basic text editor: vim, nano, etc

I do everything via ssh terminal, so these will all be console command instructions from here on out.

# Getting Started

First we will create a few files for `systemd-networkd` to work with.

First up will be the `br0` device itself:
`sudo nano /etc/systemd/network/10-br0.netdev`
```
# Name of your desired Bridge device.
[NetDev]
Name=br0
Kind=bridge
```

Next will be the "network" for your physical network card to use, which in this case will be `br0`.
`sudo nano /etc/systemd/network/20-enp5s0.network`
```
# This is the name of your network card: eno1, enp1s0, enp5s0, etc, etc.
[Match]
Name=enp5s0
# This is the name of the Bridge device we created before.
[Network]
Bridge=br0
```

Next up is how the `br0` is going to get it's IP and other settings. This is typically static or via DHCP.
`sudo nano /etc/systemd/network/30-br0.network`
```
# Name of your Bridge device.
[Match]
Name=br0

[Network]
# Your desired static IP configuration that matches your local lan configuration. Comment out to use DHCP settings below.
Address=192.168.1.123/24
Gateway=192.168.1.1
DNS=192.168.1.1

# Or via simple DHCP. Un-comment line below, and comment out lines above to use DHCP.
# DHCP=yes
```

If you are reserving a specific IP for this machine on your router via MAC, then you can add this to the 10-br0.netdev file.
```
# Name of your desired Bridge device.
[NetDev]
Name=br0
Kind=bridge
# The actual MAC address of your physical card.
MACAddress=xx:xx:xx:xx:xx:xx
```

# Enabling systemd-networkd

By default on Debian 13, systemd-networkd is already installed, just not enabled. At this point we are still using `/etc/network/interfaces.

To enable systemd-networkd and disable ifupdown (interfaces):
`sudo systemctl enable systemd-networkd`

We're going to backup the files in `/etc/network`
`sudo mv /etc/network/interfaces /etc/network/interfaces.bak`
`sudo mv /etc/network/interfaces.d /etc/network/interfaces.d.bak`

We're now going to disable networking.service
`sudo systemctl disable networking.service`

