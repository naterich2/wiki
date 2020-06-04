---
title: Enterprise Wifi for fun
description: 
published: true
date: 2020-06-04T00:04:33.661Z
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

Then Ised the schema supplied with freeRADIUS to initialize the database with the correct table formats.  For me, the schema was installed to `/etc/freeradius/3.0/mods-config/sql/main/[sql_dist]/schema.sql` where `[sql_dist]` was `mysql` for me:

Then I set up the mysql server by creating a database for radius, a user for the freeRADIUS server, and granting select privilege on all tables, as well as write to the accounting and postauth tables
```sql
CREATE USER 'freeradius'@'localhost' IDENTIFIED BY '[password]';

GRANT SELECT ON radius.* TO 'radius'@'localhost';

GRANT ALL on radius.radacct TO 'radius'@'localhost';
GRANT ALL on radius.radpostauth TO 'radius'@'localhost';
FLUSH PRIVILEGES;
```

I used the schema supplied with freeRADIUS to initialize the database with the correct table formats.  For me, the schema was installed to `/etc/freeradius/3.0/mods-config/sql/main/[sql_dist]

