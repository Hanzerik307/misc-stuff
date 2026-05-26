---
Title: Simple Bridge Using Networkd
Author: Hanzerik307
Date: May 26, 2026
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

By default on Debian 13, systemd-networkd is already installed, just not enabled. At this point we are still using `/etc/network/interfaces`.

To enable systemd-networkd and disable ifupdown (interfaces):

`sudo systemctl enable systemd-networkd`

We're going to backup the files in `/etc/network`

`sudo mv /etc/network/interfaces /etc/network/interfaces.bak`

`sudo mv /etc/network/interfaces.d /etc/network/interfaces.d.bak`

We're now going to disable networking.service

`sudo systemctl disable networking.service`

At this point, everything should be ready to go. I typically have easy access to my machines in my house, so if something was wrong, I can login locally and fix it. 

But when you reboot and log back in you will see something like this:
```
hanzerik@debian-testing:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp5s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br0 state UP group default qlen 1000
    link/ether 10:66:6a:bb:1d:d6 brd ff:ff:ff:ff:ff:ff
    altname enx10666abb1dd6
3: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 10:66:6a:bb:1d:d6 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.145/24 metric 1024 brd 192.168.1.255 scope global dynamic br0
       valid_lft 85379sec preferred_lft 85379sec
    inet6 fe80::1266:6aff:febb:1dd6/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
```

You can see that `enp5s0` is now slave to `br0`, and it has no IP. `br0` is now master doing all the work and has an IP. You may have noticed in this example that both MACs are the same, thats because I did this example within a Debian VM running on incus, and the VM is using a macvlan network instead of a br0 device to get it's IP on the lan. Without using the `MACAddress=` setting in the VM's netdev file, it wouldn't work.
But for a normal "Host" system, the above instructions should work fine for creating a `br0` device, and the network configuration files are easily copied over to a new system if needed to re-install...as long as you have backups of them.
