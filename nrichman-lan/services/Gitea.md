---
title: Gitea
description: Description of my gitea instance
published: true
date: 2020-05-31T22:13:08.126Z
tags: git, homelab
---

# Gitea
I wanted my own git server and after trying Gitlab and having it just eat CPU power, I looked for a simpler instance and came across [Gitea](https://gitea.io/en-us/), which has worked amazingly for me.

Gitea not only works as a git server for me to host my server configuration and various coding projects, but it also works as the storage backend for this wiki.

## Setup
I set up Gitea using docker (again with ansible).  Gitea has a pretty simple setup, I just made directories for the data and started the image with the `docker_container` module:
```yaml
- name: Gitea docker image
   docker_container:
     name: gitea
     image: gitea/gitea
     pull: true
     hostname: git.nrichman.dev
     volumes:
       - "{{ gitea }}:/data:rw"
     ports:
       - "[gitea_port]:3000"
       - "[gitea_ssh_port]:22"
     networks:
       - name: main
         ipv4_address: "172.16.100.2"
         aliases:
           - "git"
     networks_cli_compatible: yes
     restart_policy: unless-stopped
     memory: 1g
``` 

I wanted this git instance to be outward facing so I made a subdomain for it on cloudflare and added a server for the subdoimain `git.nrichman.dev` in my nginx config:

```
##############################################################################################
# git.nrichman.dev main server
#  Redirects to localhost:[gitea_port] for gitea docker container
server {
    access_log /var/log/nginx/git.nrichman.dev.access.log main;
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    
    ssl_certificate /path/to/certs/site.pem;
    ssl_certificate_key /path/to/certs/site.key;

    server_name git.nrichman.dev;

    location / {
        proxy_set_header    Host              $host;
        proxy_set_header    X-Real-IP         $remote_addr;
        proxy_set_header    X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto $scheme;
        proxy_pass          http://[local_ip]:[gitea_port];
        proxy_read_timeout  90;
        proxy_redirect      off;
    } 
}

```

And that was pretty much it for basic config.

## OpenID-Connect through Keycloak
I set up openid-connect briefly using Keycloak (see Keycloak page) but ended up running into errors when I would update either Gitea or Keycloak due to the OIDC url being passed or parsed correctly (relating to [this](https://github.com/go-gitea/gitea/issues/8930) issue).

For now, I've gotten it working, but it may break again at some point.  Keycloak setup was fairly simple when I started to understand keycloak.  I created a client in keycloak and copied the client ID and the client secret.  Under Authentication Sources in Gitea, I added an OAuth2 Authentication source with the Provider OpenID Connect and put in the client id and secret.  I also copied the auto-discovery link from the Realm Settings->General->Endpoints->OpenID Endpoint Configuration and put it into the OpenID Connect Auto Discovery URL.  In general for Keycloak this seems to be https://login.nrichman.dev/auth/realms/{realm}/.well-known/openid-configuration

The redirect URI that I ended up using is the following: https://git.nrichman.dev/user/oauth2/{NAME}/callback*

I used this because it was the redirect that I was seeing in Keycloak logs, however Keycloak was saying it wasn't listed as a valid redirect URI, so I put this in and it seemed to work.

## Migrating to MariaDB
I then became interested in being able to access the database in which gitea stored most of its information.  Since I created the directory that maps to the `/data` folder on my local machine I was able to interact with the `{{ gitea }}/gitea/gitea.db` sqlite3 db, but I wanted to be able to access all my databases from the same location in my MariaDB instance.  

I used a free tool to [convert](https://www.rebasedata.com/convert-sqlite-to-mariadb-online) the sqlite3 database to mysql and created a database with the guidance of Gitea docs:

```sql
CREATE USER 'gitea' IDENTIFIED BY '[password]';

CREATE DATABASE giteadb CHARACTER SET 'utf8mb4' COLLATE 'utf8mb4_unicode_ci';
GRANT ALL PRIVILEGES ON giteadb.* TO 'gitea';
FLUSH PRIVILEGES;
```

Then I populated that DB with the converted sql file:

```
mysql --user=gitea --password=[password] -h 127.0.0.1 -P [mariadb_port] giteadb < gitea.sql
```

When I populated app.ini with those settings, gitea logs kept giving me an error on startup:

`You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near 'release MODIFY COLUMN tag_name`

It turns out that one of the tables created is called 'release' which is a special word in MariaDB, the actual table needed to be called ``release``, but I couldn't rename the table.  Luckily, if I just dropped the table and restarted gitea, it seemed to create that table for me, and since I had no releases yet, it wasn't an issue.

The second issue I had was trying to navigate to my user page which would return a 500 internal server error.  The logs would say `SearchRepository: Repo: arg template_id as int: strconv.ParseInt: parsing "": invalid syntax`.  This one took me a bit to figure out, as the structure looked the same in mysql and sqlite3:

sqlite3:
```sql
SELECT id,name,template_id FROM repository;
1|home-ansible|0
2|nginx-config|0
3|download-hub|
4|nrichman.dev|0
6|influxdata-docker|
7|guest_login|
9|wiki|0
10|OOPS_MEME|0
11|cov19_analysis|0
```

mysql:
```sql
SELECT id,name,template_id FROM repository;
+------+-------------------+-------------+
| id   | name              | template_id |
+------+-------------------+-------------+
|    1 | home-ansible      | 0           |
|    2 | nginx-config      | 0           |
|    3 | download-hub      |             |
|    4 | nrichman.dev      | 0           |
|    6 | influxdata-docker |             |
|    7 | guest_login       |             |
|    9 | wiki              | 0           |
|   10 | OOPS_MEME         | 0           |
|   11 | cov19_analysis    | 0           |
+------+-------------------+-------------+
```

I didn't actually know what the field `template_id` meant and I could have looked it up, but I assumed it wasn't important.  I figured the problem was coming from the blank values and how those were being parsed, but I couldn't figure out why it would have a problem with mysql but not sqlite, especially when I had set the character set just like the docs said.

I ended up just setting all the values to 0 (`UPDATE ``repository`` SET template_id = 0;`) which then worked for me.