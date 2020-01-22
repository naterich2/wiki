---
title: Keycloak
description: 
published: true
date: 2020-01-22T05:47:45.157Z
tags: 
---

# Keycloak

## Installing Keycloak
At first I wanted to install Keycloak via docker on one of my raspberry pi's, but Keycloak doesn't have an arm image and I didn't feel like building my own or using the only (unsupported) image I could find.  Instead I decided to run Keycloak in docker on my main server with the built in H2 database, since I didn't want to mix access to the database for my 'secure' Keycloak with my relatively unsecure website.

The config I ended up with is as follows:

```yaml
- name: Keycloak container
   docker_container:
     name: keycloak
     image: jboss/keycloak
     pull: true
     ports:
       - "3315:8080"
     env:
       KEYCLOAK_USER: '{{ keycloak_user }}'
       KEYCLOAK_PASSWORD: '{{ keycloak_passwd }}'
       PROXY_ADDRESS_FORWARDING: 'true'
     memory: 1g
     cpu_period: 100000
     cpu_quota: 50000
```
