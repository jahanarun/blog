---
date: "2022-08-04T22:32:03+05:30"
description: Simple way to view photos from all of your family's mobile devices in one place
featured_image: ""
tags:
- guide
- nextcloud
- photoprism
- docker
- ofelia
- unified
- photo
title: Connect Photoprism with Nextcloud
draft: false
---

[Nextcloud](https://github.com/nextcloud) is a great app to be used as your private cloud. Once you have setup Nextcloud and have created user accounts in it, you can then use the phone app to configure auto-sync of your phone's camera photos.
This works well for every user individually.

But now, what if you want all the photos taken by your family (with multiple devices) to be available in one place so that you can have a unified gallery look.

The ideal tool to use in this case would be [Photoprism](https://github.com/photoprism/photoprism) and configure it to use Nextcloud's data folder.

## Docker compose files
I use below docker compose files to setup Nextcloud and Photoprism. Use these as pointer and modify as necessary.

### nextcloud.yml

```dockerfile
version: "3.5"

services:
  nextcloud-db:
    image: mariadb
    container_name: "nextcloud-db"
    restart: unless-stopped
    networks:
      - backend
    expose:
      - "3306"
    command: --verbose --transaction-isolation=READ-COMMITTED --binlog-format=ROW --innodb-file-per-table=1 --skip-innodb-read-only-compressed
    volumes:
      - nextcloud-db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=${NEXTCLOUD_ROOT_PASSWORD}
      - MYSQL_PASSWORD=${NEXTCLOUD_PASSWORD}
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud-user

  nextcloud:
    image: jhnrn/nextcloud-linux
    container_name: "nextcloud"
    restart: unless-stopped
    ports:
      - 8080:80
    links:
      - nextcloud-db
    volumes:
      - /nextcloud/html:/var/www/html
      - /nextcloud/data:/var/www/html/data
    environment:
      - MYSQL_PASSWORD=${NEXTCLOUD_PASSWORD}
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud-user
      - MYSQL_HOST=nextcloud-db
      - NEXTCLOUD_TRUSTED_DOMAINS=${NEXTCLOUD_TRUSTED_DOMAINS}
      - REDIS_HOST=nextcloud-redis
      - NEXTCLOUD_HOSTNAME=${NEXTCLOUD_HOSTNAME}
      - NEXTCLOUD_ADMIN_USER=${NEXTCLOUD_ADMIN_USER}
      - NEXTCLOUD_ADMIN_PASSWORD=${NEXTCLOUD_ADMIN_PASSWORD}
    depends_on:
      - nextcloud-db

```
### photoprism.yml

```dockerfile
version: "3.5"

services:
  photoprism:
    image: photoprism/photoprism:latest
    container_name: "photoprism"
    depends_on:
      - photoprism-mariadb
    restart: unless-stopped
    ports:
      - "2342:2342" # HTTP port (host:container)
    environment:
      PHOTOPRISM_ADMIN_PASSWORD: ${PHOTOPRISM_ADMIN_PASSWORD} 
      PHOTOPRISM_SITE_URL: ${PHOTOPRISM_SITE_URL} 
      PHOTOPRISM_DATABASE_DRIVER: "mysql" 
      PHOTOPRISM_DATABASE_SERVER: "photoprism-mariadb:3306"
      PHOTOPRISM_DATABASE_NAME: "photoprism" 
      PHOTOPRISM_DATABASE_USER: "photoprism"
      PHOTOPRISM_DATABASE_PASSWORD: ${PHOTOPRISM_DATABASE_PASSWORD}
      PHOTOPRISM_SITE_TITLE: "PhotoPrism"
      PHOTOPRISM_SITE_CAPTION: "Browse Your Life"
      PHOTOPRISM_SITE_DESCRIPTION: ""
      PHOTOPRISM_SITE_AUTHOR: ""
      HOME: "/photoprism"
    working_dir: "/photoprism"
    volumes:
      - /nextcloud/data/alice/files/Photos/Camera:/photoprism/originals/alice:ro
      - /nextcloud/data/dennis/files/Photos/Camera:/photoprism/originals/dennis:ro
    labels:
      ofelia.enabled: "true"
      ofelia.job-exec.photoprism_index.schedule: "@every 1h"
      ofelia.job-exec.photoprism_index.command: "photoprism index --cleanup"

  photoprism-mariadb:
    restart: unless-stopped
    container_name: "photoprism-db"
    image: mariadb:10.6
    security_opt:
      - seccomp:unconfined
      - apparmor:unconfined
    networks:
      - backend
    command: mysqld --innodb-buffer-pool-size=256M --transaction-isolation=READ-COMMITTED --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --max-connections=512 --innodb-rollback-on-timeout=OFF --innodb-lock-wait-timeout=120
    volumes:
      - "photoprism-db:/var/lib/mysql"
    environment:
      MYSQL_ROOT_PASSWORD: ${PHOTOPRISM_ROOT_DATABASE_PASSWORD}
      MYSQL_DATABASE: photoprism
      MYSQL_USER: photoprism
      MYSQL_PASSWORD: ${PHOTOPRISM_DATABASE_PASSWORD}

  ofelia:
    restart: unless-stopped
    image: mcuadros/ofelia:latest
    container_name: ofelia
    depends_on:
      - photoprism
    command: daemon --docker
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
```
## Key mentions
Use the nextcloud.yml to setup Nextcloud first. You can also download the Nextcloud app from Andriod/Apple App store and setup Photo sync.

The key thing to note in the above docker-compose configuration is the `volume` section of Photoprism.
You can see the Photoprism uses bind mount to the folders created by Nextcloud app.
While using the photoprism.yml file, make sure the volume section is modified to point to the correct source path.

## Index refresh
This setup is fine and Photoprism would scan the folders and index the files on the first run. But any more photos synced to Nextcloud would not appear on Photoprism as there is no way for Photoprism to know about the photos added to Nextcloud.
So, I use [ofelia](https://github.com/mcuadros/ofelia) to run to schedule scan job on photoprism. This runs a scan job on Photoprism so that it reflects any new photos added, modified or deleted in Nextcloud.

That's it! Now you can go into the Photoprism app and find all your photos of your family in one place.