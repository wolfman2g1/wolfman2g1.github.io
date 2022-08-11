---
title: Stand up Bookstack
date: 2022-07-10 12:00:00 -500
categories: [homelab,docker,wiki]
layout: post
tags: [bookstack,docker-compose] 
---

Make config directory
```console
mkdir   /data/bookstack_data
mkdir /data/bookstack_db_data
```
Copy the compose file to

```console
vi /data/bookstack_data/docker-compose.yml
```
```yaml
version: "3"
services:
  bookstack:
    image: lscr.io/linuxserver/bookstack
    container_name: bookstack
    environment:
      - PUID=1000
      - PGID=1000
      - DB_HOST=bookstack_db
      - DB_USER=bookstack
      - DB_PASS=DBPW
      - DB_DATABASE=bookstackapp
    volumes:
      - /data/bookstack_data:/config
    ports:
      - 6875:80
    restart: unless-stopped
    depends_on:
      - bookstack_db
  bookstack_db:
    image: lscr.io/linuxserver/mariadb
    container_name: bookstack_db
    environment:
      - PUID=1000
      - PGID=1000
      - MYSQL_ROOT_PASSWORD=ROOTPW
      - TZ=America/New_York
      - MYSQL_DATABASE=bookstackapp
      - MYSQL_USER=bookstack
      - MYSQL_PASSWORD=DBPW
    volumes:
      - /data/bookstack_db_data:/config
    restart: unless-stopped
```