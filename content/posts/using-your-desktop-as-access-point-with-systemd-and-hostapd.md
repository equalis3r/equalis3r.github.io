+++
title = "Using your desktop as access point with Systemd and Hostapd"
date = "2022-01-26T21:16:12+01:00"
author = "equalis3r"
authorTwitter = "" #do not include @
cover = ""
tags = ["networking", "systemd", "hostapd"]
keywords = ["", ""]
description = "My self-documentation on how to configure my desktop as an access point"
showFullContent = false
readingTime = false
+++

## Situation

1. A desktop with Intel AX200
2. An old and slow access point.
3. Using `systemd-networkd`, `systemd-networkd`, and `nftables`
for network configuration.

## Preparation

Assuming that your external network is connected via **eth0** and
your internal network is connected via **wlan0**.

## Method 1: iwd AP mode

Make sure that your wireless adapter supports this operation mode
by checking with `iw phy`.
We first need to set up our interfaces with systemd-networkd.
For `eth0`, we want to receive the IP from our router, so DHCP
is set to yes.

    File: /etc/systemd/network/20-wired.network
    [Match]
    Name=eth0

    [Network]
    DHCP=yes

I don't think it matters to set up `wlan0` since we will designate
that task to iwd but we can use it later anyway.

    File: /etc/systemd/network/25-wireless.network
    [Match]
    Name=wlan0

    [Network]
    DHCP=yes

Then, we let iwd configure the network for us.

    File: /etc/iwd/main.conf
    [General]
    EnableNetworkConfiguration=true

Next, we need to create our wireless profile so iwd
can broadcast it. In my case, I call it **Void**.

    File: /var/lib/iwd/ap/Void.ap
    [Security]
    Passphrase=your passphrase
    
    [IPv4]
    Address=192.168.1.1
    Gateway=192.168.1.1
    Netmask=255.255.255.0
    DNSList=8.8.8.8

Then, we start iwd AP mode on `wlan0`.

    >>> iwctl
    >>> device wlan0 set-property Mode ap
    >>> ap wlan0 start-profile Void

Up and running? Nope. Turns out we need to enable 
NAT (Network Address Translation), otherwise our subnet cannot
access internet (or the externat network). Add the following rules
to your nftables

    File: /etc/nftables.conf
    ...
    table ip nat {
        chain prerouting {
             type nat hook prerouting priority 0; policy accept;
         }
     
        chain postrouting {
            type nat hook postrouting priority 100; policy accept;
            oifname "eth0" masquerade
        }
    }

Restart nftables.service and you should be able to connect to Void and
to whatever Void can connect to. Although this approach is very easy
and straightforward, the speed connection is terribly slow.
I can barely get a download speed of 4mbps. I got around 9mpbs when I
connected directly to the access point. I guess the reason is because
iwd AP mode is very basic that it does not take advantage of 
my wireless adapater features, and thus I search around for 
the second method.

## Method 2: Hostapd + Bridge wlan0 and eth0

First, we need to create a virtual bridge:

    File: /etc/systemd/network/02-br0.netdev
    [NetDev]
    Name=br0
    Kind=bridge

    File: /etc/systemd/network/16-br0_up.network
    [Match]
    Name=br0
    
    [Network]
    DHCP=ipv4

    File: /etc/systemd/network/04-eth0.network
    [Match]
    Name=eth0

    [Network]
    Bridge=br0

    File: /etc/systemd/network/25-wireless.network
    [Match]
    Name=wlan0

    [Network]
    IPMasquerade=yes
    Address=192.168.16.1/24
    DHCPServer=yes

    [DHCPServer]
    DNS=192.168.16.1
    DNS=8.8.8.8
    DNS=8.8.4.4 

Then configure hostapd. Depends on your use case,
you may want to change the settings such as country_code, channel,...

    File: /etc/hostapd/hostapd.conf
    interface=wlan0
    bridge=br0
    driver=nl80211
    ssid=Void
    country_code=NL
    ieee80211d=1
    hw_mode=g
    channel=12
    auth_algs=1
    wpa=2
    wpa_passphrase=verySecretPassword
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=CCMP
    ieee80211n=1

Enable hostapd.service and reboot. Using this method, my download 
speed when connected to Void is improved to around 85mpbs. Whoops!
However, I'm still looking into methods that can let me use 80211ac/ax 
on 5Ghz channels.
