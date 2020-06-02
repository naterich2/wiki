---
title: Enterprise WiFi network routed through VPN
description: How I created an enterprise wifi network that goes out through a TorGuard VPN
published: true
date: 2020-06-02T21:44:37.596Z
tags: networking, vpn, wifi
---

# Enterprise WiFi network routed through VPN

In progress...


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

### Creating VLAN to be routed out VPN
I went to Interfaces->VLANs and created a new vlan.  Under Interfaces->Interface Assignments, I assigned this to a new OPT interface and named it to VLAN_99_VPN_OUT.  I set its IPv4 Configuration type to Static IPv4 and made the IPv4 subnet a /28 network  since I didn't want many clients connected on this subnet.

Finally, I enabled the DHCP server on this Interface under Services->DHCP Server->VLAN_99_VPN_OUT, and set the range to what I wanted.  I also had to make sure I set the DNS servers not to my local server but to the ones provided by TorGuard, 10.9.0.1 and 10.8.0.1

### Gateway and NAT
The next step was to configure the gateway and NAT from internal subnets to this gateway.

I went to Interfaces->Interface Assignments, assign a new interface for the newly created network port ovpnc*(), and renamed it to VPN_OUT.

Then I went to System->Routing->Gateways and selected the gateway that had been created, made sure it was on the correct interface (for me VPN_OUT) and gave it a name and description.  For TorGuard, the gateway doesn't seem to respond to pings, so I checked "Disable Gateway Monitoring Action" so that the gateway will show as being up.

Finally, I created NAT rules for the gateway under Firewall->Nate->Outbound. I added a rule from the VPN_OUT interface from the subnet I made in my VLAN_99_VPN_OUT to be natted as the VPN_OUT interface address.

### Policy-based routing
Now that the VPN client, VLAN, gateway, and NAT were all set up, I just had to add Firewall rules to do the [policy-based routing](https://docs.netgate.com/pfsense/en/latest/routing/directing-traffic-with-policy-routing.html), under the VLAN_99_VPN_OUT interface, I added a pass rule allowing any IPv4 protocol from the VLAN_99_VPN_OUT network to any destination, but set the gateway to the one I had named above.  The rule should look like this:
![vpn_routing_rule_1.png](/vpn_routing_rule_1.png)

## Unifi controller setup
### Allow VLAN to APs
First I had to allow this VLAN to the ports of my APs, I have a Netgear FS728TP managed PoE switch that I use, and it has a decent web interface.  Under Switching->VLAN I just had to add a VLAN with the id 99 and whatever name I wanted, then under advanced->VLAN Membership, I added ports 13, 14 (my AP ports) and my trunk port (27) as tagged:
![vlan_tagged.png](/vlan_tagged.png).

### Create WiFi network
Under Settings->Wireless I chose Create New Network. I named it the same as my normal WiFi, but with "-vpn" on the end, and assigned the security to WPA Enterprise.  The Radius Profile I chose was the existing one I had made.

Then I provisioned the APs.
## Freeradius setup
The RADIUS setup on the Unifi controller is super easy, it was the freeRADIUS server that took more configuration.

I already had a Radius profile configured from my page [Configuring an RADIUS assigned VLAN enterprise network](/nrichman-lan/networking/radius_wifi)

So I only had to change some configuration from that.  All I ended up doing was adding a section to update the inner-tunnel reply to send the VPN VLAN id if the Called-Station-Id matched my criteria.  Probably not the best way of doing it, but it works for me easily and I'm not too concerned about security since anyone I've give access to my enterprise network will be trusted.

The config change I added was the following in `sites-enabled/inner-tunnel`:
```
if (&Called-Station-Id =~ /^[0-9A-F-]+:[\w.]+-([vpn]+)/) {
			update reply {
				&Tunnel-Private-Group-Id := '99'
			}
}
```
Basically, this just overrides the attribute from the database if the client is on the VPN SSID and drops them on the VPN VLAN.