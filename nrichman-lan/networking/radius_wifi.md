---
title: Enterprise Wifi for fun
description: 
published: true
date: 2020-06-04T01:00:07.075Z
tags: networking
---

# Enterprise Wifi for fun
I wanted to set up an Enterprise Wifi network since I was so interested with how it worked.  My school (UW-Madison) used eduroam and a mac-caching open network.  When I was working as a network engineer, they gave me access to EVERYTHING since they had weren't used to having interns, so they didn't have a level between Help Desk access and full administrator access to anything really.  I spent hours outside of work trying to figure out how RADIUS was configured in Clearpass. 

So I wanted to try my hand at setting up my own network with a few things that I wanted:

1. Each user would have a username and password
2. Based on the user, they would be put onto a certain VLAN
3. User information and passwords would be stored encrypted in a MariaDB database

Assuming, I already had the VLANs created and allowed to the ports for the APs, the first step was to install RADIUS and the mariadb server.

## Installing and setting up FreeRADIUS and MariaDB
I ended up installing the RADIUS server on a Raspberry PI, but I might move it to a proxmox VM at some point.  On the pi, I was able to install the database and FreeRADIUS server using apt: `apt-get install freeradius freeradius-mysql mysql-server mysql-client`

### Setup MariaDB server
First I created a database for the radius server: `CREATE DATABASE radius;`

Then I used the schema supplied with freeRADIUS to initialize the database with the correct table formats.  For me, the schema was installed to `/etc/freeradius/3.0/mods-config/sql/main/[sql_dist]/schema.sql` where `[sql_dist]` was `mysql` for me: `sudo mysql radius < /etc/freeradius/3.0/mods-config/sql/main/mysql/schema.sql`

Then I created a user for the freeRADIUS server, and granted select privilege on all tables, as well as write to the accounting and postauth tables
```sql
CREATE USER 'freeradius'@'localhost' IDENTIFIED BY '[password]';

GRANT SELECT ON radius.* TO 'radius'@'localhost';

GRANT ALL on radius.radacct TO 'radius'@'localhost';
GRANT ALL on radius.radpostauth TO 'radius'@'localhost';
FLUSH PRIVILEGES;
```

### Configuring the sql module of FreeRADIUS
These are the steps I took to set up the sql module for freeRADIUS:

Soft link `mods-available/sql` to `mods-enabled/sql`: `ln -s mods-available/sql mods-enabled/sql`, then edit that file and set the following config items:
* set `driver = "rlm_sql_mysql"`
* set `dialect` = "mysql"`
* set `server = "localhost"` (for me this is true, but it can be an IP address too, and probably a domain name too)
* set `port = 3306` (default)
* set `login = "radius"`
* set `password = [password]`

### Configuring FreeRADIUS server
FreeRADIUS comes pretty well configured to do just about anything, I only had to add a few things.

At the beginning of `sites-enabled/default` I added a variable for my default vlan: `default_vlan = "100"`

I also commented out chap, mschap, digest, suffix, and files from the authrize section, since I really only want to use sql and my usernames won't have a domain (username would be nrichman, not nate@nrichman.dev).  I also uncommented the sql lines in the accounting section, since I wanted accounting.

Really the only significant thing I changed was in the post-auth section where the server sends the reply.  I need the server to send a reply that the user will go on a VLAN, as well as the VLAN id:
```
		update reply {
			Tunnel-Type := VLAN
			Tunnel-Medium-Type := IEEE-802
      Tunnel-Private-Group-Id = ${default_vlan}
	}
```

This section updates the reply to include a tunnel type of VLAN over an IEEE 802 medium.  Finally, it sets the VLAN id to the value of `default_vlan`, the opcode for the last assignment is just `=` instead if `:=` which means that if that attribute is already set from looking in the sql database it won't overwrite.

### Configuring EAP
The final step was to configure EAP.  The best solution I found to not have plaintext passwords was to use microsofts MS-CHAP which requires an NT-Password which is encrypted based on the MD4 hashing algorithm, definitely not the best, but doable for now.

In `mods-enabled/eap` I did the following
* set `default_eap_type = peap`
* set `ignore_unknown_eap_types = yes`
* Comment out MD5, PWD, LEAP, GTC, and TLS
* Under EAP-TTLS set `default_eap_type = mschapv2`
* Under EAP-PEAP set `default_eap_type = mschapv2`

## Configuring users and clients
The first thing I needed to do was to add clients which will relay the radius requests, I added one for localhost for testing, and one for each AP.  I also added one for the controller, but I don't think its needed, this is done in the `nas` table in the db:
```sql
MariaDB [radius]> DESCRIBE `nas`;
+-------------+--------------+------+-----+---------------+----------------+
| Field       | Type         | Null | Key | Default       | Extra          |
+-------------+--------------+------+-----+---------------+----------------+
| id          | int(10)      | NO   | PRI | NULL          | auto_increment |
| nasname     | varchar(128) | NO   | MUL | NULL          |                |
| shortname   | varchar(32)  | YES  |     | NULL          |                |
| type        | varchar(30)  | YES  |     | other         |                |
| ports       | int(5)       | YES  |     | NULL          |                |
| secret      | varchar(60)  | NO   |     | secret        |                |
| server      | varchar(64)  | YES  |     | NULL          |                |
| community   | varchar(50)  | YES  |     | NULL          |                |
| description | varchar(200) | YES  |     | RADIUS Client |                |
+-------------+--------------+------+-----+---------------+----------------+
MariaDB [radius]> INSERT INTO `nas` (`nasname`,`shortname`,`secret`,`description`)
VALUES('[ap1 IP]','ap1.nrichman.lan','[secret]','AP1 access point');

MariaDB [radius]> INSERT INTO `nas` (`nasname`,`shortname`,`secret`,`description`)
VALUES('127.0.0.1','localhost','[secret]','localhost');

MariaDB [radius]> INSERT INTO `nas` (`nasname`,`shortname`,`secret`,`description`)
VALUES('[controller IP]','data.nrichman.lan','[secret]','Unifi Controller');

MariaDB [radius]> INSERT INTO `nas` (`nasname`,`shortname`,`secret`,`description`)
VALUES('[ap2 IP]','ap2.nrichman.lan','[secret]','AP2 access point');

MariaDB [radius]> SELECT * FROM `nas`;
+----+-----------+-------------------+-------+-------+----------+--------+-----------+------------------+
| id | nasname   | shortname         | type  | ports | secret   | server | community | description      |
+----+-----------+-------------------+-------+-------+----------+--------+-----------+------------------+
|  1 | [ap1 ip]  | ap1.nrichman.lan  | other |  NULL | [secret] | NULL   | NULL      | AP1 access point |
|  2 | 127.0.0.1 | localhost         | other |  NULL | [secret] | NULL   | NULL      | localhost        |
|  3 | [data ip] | data.nrichman.lan | other |  NULL | [secret] | NULL   | NULL      | Unifi Controller |
|  4 | [ap2 ip]  | ap2.nrichman.lan  | other |  NULL | [secret] | NULL   | NULL      | AP2 access point |
+----+-----------+-------------------+-------+-------+----------+--------+-----------+------------------+
```

Now that I have the clients in there, I can add users.  For the `radcheck` table, I need a username, and NT-Password, the NT-Password can be calculated from the utility `smbencrypt` installed by freeRADIUS:
```bash
$smbencrypt "test"
LM Hash			 										NT Hash
--------------------------------	--------------------------------
01FC5A6BE7BC6929AAD3B435B51404EE	0CB6948805F797BF2A82807973B89537
```

I can insert a client into the `radcheck` table:
```sql
MariaDB [radius]> INSERT INTO `radcheck` (`username`,`attribute`,`value`) 
VALUES ('test','NT-Password','0CB6948805F797BF2A82807973B89537');
```
This would be enough to allow the user to login, but additionally, I wanted to be able to set VLAN by group membership, so I added the user to a group membership in `radusergroup`, as well as made sure the user is login in over wifi via `radgroupcheck`, finally in `radgroupreply` is where the VLAN can be sent:
```sql
MariaDB [radius]> INSERT INTO `radusergroup` (`username`,`groupname`)
VALUES(`test`,`client`);

MariaDB [radius]> INSERT INTO `radgroupcheck` (`groupname`,`attribute`,`value`)
VALUES(`client`,`NAS-Port-Type`,`Wireless-802.11`);

MariaDB [radius]> INSERT INTO `radgroupreply` (`groupname`,`attribute`,`value`)
VALUES(`client`,`Tunnel-Private-Group-Id`,`100`);

MariaDB [radius]> SELECT * FROM `radcheck`;
+----+-------------+-------------+----+----------------------------------+
| id | username    | attribute   | op | value                            |
+----+-------------+-------------+----+----------------------------------+
|  1 | test        | NT-Password | := | 0CB6948805F797BF2A82807973B89537 |
+----+-------------+-------------+----+----------------------------------+

MariaDB [radius]> SELECT * FROM `radusergroup`;
+-----------+-----------+----------+
| username  | groupname | priority |
+-----------+-----------+----------+
| test      | client    |        1 |
+-----------+-----------+----------+

MariaDB [radius]> SELECT * FROM `radgroupcheck`;
+----+-----------+---------------+----+-----------------+
| id | groupname | attribute     | op | value           |
+----+-----------+---------------+----+-----------------+
|  1 | client    | NAS-Port-Type | == | Wireless-802.11 |
+----+-----------+---------------+----+-----------------+

MariaDB [radius]> SELECT * FROM `radgroupreply`;
+----+-----------+-------------------------+----+-------+
| id | groupname | attribute               | op | value |
+----+-----------+-------------------------+----+-------+
|  1 | client    | Tunnel-Private-Group-Id | := | 100   |
+----+-----------+-------------------------+----+-------+
```

## Configuring Unifi
First thing to do is create a radius profile:
Under Profiles->RADIUS.  Enter a name, check Enable RADIUS assigned VLAN for wireless network, enter ip and port for auth server as well as the secret from before.
![unifi_radius_profile.png](/unifi_radius_profile.png)

Then under wireless networks, its as simple as creating a normal network, choosing WPA Enterprise Security, and choosing the radius profile that was just created.




