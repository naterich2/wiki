---
title: Keycloak
description: 
published: true
date: 2020-05-31T22:01:32.883Z
tags: authentication, homelab
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

## Setup
I created a realm for my website `nrichman.dev` with openid-connect and saml support

Then I created clients for all the services I use, namely gitea, nextcloud, and wikijs. The openid-connect setup is much easier than the saml setup.  In general for openid, all it takes in keycloak is finding the root url, and valid redirect uri's which is usually on the clients documentation. I always set the access type to confidential.  Then in the client, I just need the client id, which is whatever I create and then I just set client authenticator to 'Client Id and Secret' and put the secret into the client.

## FreeIPA Integration
coming...
