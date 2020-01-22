---
title: Ansible Overview
description: 
published: true
date: 2020-01-22T04:41:49.319Z
tags: 
---

# Ansible Overview

## Ansible Organization
As far as I can tell, Ansible can be used in many different ways while following the guidelines suggested by the developers.

The overall scheme that I chose to use for managing my servers with Ansible was to create a role for each service or group of service that I run.

For example, I have these roles:
* common: common tasks that are on each server, namely Telegraf for collecting statistics
* nextcloud: Nextcloud and MariaDB instance for running Nextcloud
* web: nginx webserver, plex, gitea, and my website.
* wikijs: role for running this instance.

Within each role I have a set of unique variables in `vars/main.yml` file and often a private file `vars/credentials.yml` that I have universally in my gitignore for passwords and private information.  Additionally, I have a `main.yml` in my `tasks` folder as well as any other tasks that I import.  Finally, in my main Ansible directory under playbooks, I have small playbooks to apply each role.  So my Ansible directory looks like the following:

```yaml
ansible/
  tasks/
  playbooks/
    common.yml
    nextcloud.yml
    web.yml
    bookstack.yml
  roles/
    web/
      tasks/
        main.yml
        gitea.yml
        plex.yml
      vars/
        main.yml
        credentials.yml
```
I chose this format for Ansible since it seems like it will make it easier to eventually move services to different hosts as I expand hardware, since right now I only have one main server and three Raspberry Pi's.
