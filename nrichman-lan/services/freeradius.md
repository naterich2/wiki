---
title: Freeradius
description: 
published: true
date: 2020-01-22T04:58:08.959Z
tags: 
---

# Freeradius
## Installation

## Setup
I wanted to set up an enterprise network just for fun, so I looked into and installed Freeradius.

Then I followed this guide to set up my Freeradius to store data and authenticate against a MariaDB database, and added my clients into the `nas` table:

```SQL
MariaDB [radius]> select * from nas;
+----+-----------+-----------+-------+-------+--------+--------+-----------+------------------+
| id | nasname   | shortname | type  | ports | secret | server | community | description      |
+----+-----------+-----------+-------+-------+--------+--------+-----------+------------------+
|  1 | ip1       | host1     | other |  NULL | --     | NULL   | NULL      | AP1 access point |
|  3 | 127.0.0.1 | localhost | other |  NULL | --     | NULL   | NULL      | localhost        |
|  5 | ip2       | host2     | other |  NULL | --     | NULL   | NULL      | AP2 access point |
+----+-----------+-----------+-------+----------------+--------+-----------+------------------+
3 rows in set (0.000 sec)
```
The only change I made to the server was in the `inner-tunnel` site where I changed the following if statement to execute in order to copy the VLAN attributes to the outer reply:
```BASH
#
	#  Instead of "use_tunneled_reply", change this "if (0)" to an
	#  "if (1)".
	#
	if (1) {
		#
		#  These attributes are for the inner-tunnel only,
		#  and MUST NOT be copied to the outer reply.
		#
		update reply {
			User-Name !* ANY
			Message-Authenticator !* ANY
			EAP-Message !* ANY
			Proxy-State !* ANY
			MS-MPPE-Encryption-Types !* ANY
			MS-MPPE-Encryption-Policy !* ANY
			MS-MPPE-Send-Key !* ANY
			MS-MPPE-Recv-Key !* ANY
		}

		#
		#  Copy the inner reply attributes to the outer
		#  session-state list.  The post-auth policy will take
		#  care of copying the outer session-state list to the
		#  outer reply.
		#
		update {
			&outer.session-state: += &reply:
		}
	}
```
Now the server was configured to send the proper attributes, but I needed to set them in the sql tables.  I created the normal users that I wanted in the `radcheck` table, then in `radusergroup` I set the users to be in the correct group:

```SQL
MariaDB [radius]> select * from radcheck;
+----+----------+--------------------+----+-------------+
| id | username | attribute          | op | value       |
+----+----------+--------------------+----+-------------+
|  1 | test     | Cleartext-Password | := | testing     |
|  2 | user     | Cleartext-Password | := | temp        |
+----+----------+--------------------+----+-------------+
2 rows in set (0.000 sec)

MariaDB [radius]> select * from radusergroup;
+---------------+-----------+----------+
| username      | groupname | priority |
+---------------+-----------+----------+
| user          | client    |        1 |
| test          | mgmt      |        1 |
+---------------+-----------+----------+
2 rows in set (0.000 sec)
```
So when a client makes a request, the RADIUS server will verify their password and user membership.  Then in `radgroupcheck` the server checks that the user is connecting in through WiFi and looks to reply with the proper attributes from `radgroupreply`:

```SQL
MariaDB [radius]> select * from radgroupcheck;
+----+-----------+---------------+----+-----------------+
| id | groupname | attribute     | op | value           |
+----+-----------+---------------+----+-----------------+
|  1 | client    | NAS-Port-Type | == | Wireless-802.11 |
|  2 | mgmt      | NAS-Port-Type | == | Wireless-802.11 |
+----+-----------+---------------+----+-----------------+
2 rows in set (0.001 sec)

MariaDB [radius]> select * from radgroupreply;
+----+-----------+-------------------------+----+-------+
| id | groupname | attribute               | op | value |
+----+-----------+-------------------------+----+-------+
|  1 | mgmt      | Tunnel-Type             | := | 13    |
|  2 | mgmt      | Tunnel-Medium-Type      | := | 6     |
|  3 | mgmt      | Tunnel-Private-Group-Id | := | 1000  |
|  4 | client    | Tunnel-Type             | := | 13    |
|  5 | client    | Tunnel-Medium-Type      | := | 6     |
|  6 | client    | Tunnel-Private-Group-Id | := | 1001  |
+----+-----------+-------------------------+----+-------+
6 rows in set (0.001 sec)
```
Where a `Tunnel-Type := 1`3 means VLAN and `Tunnel-Medium-Type := 6` means IEEE-802, and the `Tunnel-Private-Group-Id` gives the actual VLAN number.

## Unifi Controller Setup
Once I had the RADIUS server configured, I needed my AP's to be able to forward and process requests as Network Access Servers (NASs).  Under Settings > Profiles I added a RADIUS profile:


## Switch VLAN passthrough

Not long after I set this up originally, I got a new access point.  When I plugged the new access point into my poe switch and it was adopted in the Unifi controller, I thought all was good...Until I started having intermittent DHCP timeouts.  I realized this was because I didn't tag the VLANs on the switch.

I have a Netgear FS728TP switch so I went into the Switching > VLAN > VLAN membership and made sure all the VLANs my users could end up on were set to 'tagged' on the port for each AP:

![vlan_switching.png](/vlan_switching.png)

Where my APs are on ports 14, and 16, and my uplink is port 27.