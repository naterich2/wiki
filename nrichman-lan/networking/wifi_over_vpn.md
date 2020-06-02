---
title: Enterprise WiFi network routed through VPN
description: How I created an enterprise wifi network that goes out through a TorGuard VPN
published: true
date: 2020-06-02T20:45:29.377Z
tags: networking, vpn, wifi
---

# Enterprise WiFi network routed through VPN

I have a private VPN through [TorGuard](https://torguard.net/) which has worked really well for me.  Right now, if I want to use it, I download the openvpn configuration and use it on each computer I want.  But Openvpn is a little touchy on mobile clients sometimes, and some clients don't have an openvpn client.  

I wanted a way to connect all the clients I want to my vpn tunnel easily, and I was inspired by a post by MonsterMuffin: [Tunneling specific traffic over a vpn with pfsense](https://blog.monstermuffin.org/tunneling-specific-traffic-over-a-vpn-with-pfsense/).  However, I didn't have nearly as fine grained rules as he had, so I wanted to make a whole vlan go out through the vpn gateway, and I figured I might as well attach that to a WiFi network as well.  Here's how I did it:

## pfSense config

### Creating vpn client
From the TorGuard account page I did tools>openvpn config generator and created one for the config I wanted, Chicago server, UDP port 192.

For pfSense, first in System->Cert. Manager-> CAs, I created a new certificate authority for Torguard.  I used the method "Import an existing Certificate Authority" and copied in the CA data from between the \<ca\> and \</ca\> tags.

Then I went to VPN->OpenVPN->Clients->Create new. Copy in the relevant information from the generated OpenVPN config, namely:
* Server host or address
* Server port
* Username
* Password
* TLS Key
* Peer Certificate Authority -> Select the newly created CA.
* Disable NCP and choose encryption algorithm on the config
* __Check Don't Pull Routes__
* __Check Don't add/remove routes__


The last two are important otherwise the routes will overwrite the system routing table and everything will go through the VPN gateway.

Finally, I added some options that were present in the downloaded config(`remote-cert-tls server;setenv CLIENT_CERT 0;tun-mtu 1500;mssfix 1550;`, checked Gateway creation IPv4 only, and saved it.

### Gateway and NAT
I wterfaces->Interface Assignments, assign a new interface for the newly created network port ovpnc*(),