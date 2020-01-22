---
title: Nextcloud
description: 
published: true
date: 2020-01-22T04:46:27.108Z
tags: 
---

# Nextcloud

## Installation

I originally installed the Nextcloud through docker and linked it to my MariaDB instance (also in docker).  I installed it on my main network and created some volumes so that I could tweak config while it was running.  Here is my ansible config for running nextcloud:

```yaml
- name: Create nextcloud directories
  file:
    path: "{{ item }}"
    state: directory
  with_items:
    - "{{ nextcloud }}/html"
    - "{{ nextcloud }}/config"
    - "{{ nextcloud }}/data"
  become: yes

#######################################################
##################### Nextcloud #######################
#######################################################
- name: Nextcloud Docker
  docker_container:
    name: nextcloud
    image: nextcloud:latest
    pull: true
    volumes:
	  - "{{ nextcloud }}/html:/var/www/html"
      - "{{ nextcloud }}/config:/var/www/html/config"
      - "{{ nextcloud }}/html:/var/www/html/data"
    env:
      MYSQL_DATABASE: "nextcloud"
      MYSQL_USER: '{{ mariadb_user }}'
      MYSQL_PASSWORD: "{{ mariadb_password }}"
      MYSQL_HOST: "{{ mariadb }}:3306"
      NEXTCLOUD_ADMIN_USER: "{{ nextcloud_admin }}"
      NEXTCLOUD_ADMIN_PASSWORD: "{{ nextcloud_admin_password }}"
      NEXTCLOUD_TRUSTED_DOMAINS: "https://cloud.nrichman.dev"
    ports:
      - "3005:80"
    networks:
      - name: main
        ipv4_address: "172.16.100.5"
        aliases:
          - "nextcloud"
    networks_cli_compatible: yes
    restart_policy: unless-stopped
    memory: 1g
    cpu_period: 100000
    cpu_quota: 50000
```

## Tweaking
This originally worked just fine, but since I have my Nextcloud instance behind a reverse proxy, I had to do some more configuration (see here).  I didn't need to do this configuration until I started trying to set up the SAML and SSO (more on that later) and I noticed that the redirect_uri and Client ID's were over http.  At first I thought that just adding `NEXTCLOUD_TRUSTED_DOMAINS: "https://cloud.nrichman.dev"` would work, but that didn't force over HTTPS.  So I eventually found the above documentation and added to my config/config.php the following lines:

```bash
  'overwritehost' => 'cloud.nrichman.dev',
  'trusted_proxies' => [--ip of my proxy--],
  'overwriteprotocol' => 'https'
```
So that fixed the HTTPS issue and now the client was importing as https://...  The final Issue I was having was an internal server error on login with keycloak.  The logs in Nextcloud were just saying:

> Error index OneLogin\Saml2\ValidationError: Found an Attribute element with duplicated Name at 
> .../vendor/onelogin/php-saml/src/Saml2/Response.php
{.is-warning}

I ended up doing some digging and found this closed issue on the nextcloud user_saml app, and it turns out I had to delete the Role mapper in my client, and change an overall Keycloak setting for SAML in the Client Scopes > role_list > Mapper Tab > Edit > Single Role Attribute (ON).

However I still get an internal server error on keycloak when I log out of nextcloud.