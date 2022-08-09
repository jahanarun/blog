---
date: "2022-08-04T22:32:03+05:30"
description: Simple way to view photos from your family's mobile devices synced to nextcloud
featured_image: ""
tags:
- guide
- nextcloud
- photoprism
- docker
title: Unified gallery view of all photos from your family
draft: false
---

Let's say you want all the photos taken by your family (with multiple devices) to be available in one place so that you can have a unified gallery look.

The ideal tools to use in this case would be [Photoprism](https://github.com/photoprism/photoprism) and [Nextcloud](https://github.com/nextcloud).

## Docker compose files
I use below docker compose files to setup Nextcloud and Photoprism.

### nextcloud.yml

```dockerfile
version: "3.5"
volumes:
  nextcloud-html:
  nextcloud-db:
  nextcloud-data:
    driver: local
    driver_opts:
      type: "nfs4"
      o: addr=10.100.50.20,rw,noatime,rsize=1048576,wsize=1048576,tcp,timeo=14
      device: ":/nextcloud/data"

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
      - nextcloud-html:/var/www/html
      - nextcloud-data:/var/www/html/data
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
      - OVERWRITEPROTOCOL=https
    depends_on:
      - nextcloud-db


```
### photoprism.yml

```dockerfile
version: "3.5"
volumes:
  photoprism-db:
  photoprism-storage:
  photoprism-alice:
    driver: local
    driver_opts:
      type: "nfs4"
      o: addr=10.100.50.20,rw,noatime,rsize=1048576,wsize=1048576,tcp,timeo=14
      device: ":/nextcloud/data/alice/files/Photos/Camera"
  photoprism-dennis:
    driver: local
    driver_opts:
      type: "nfs4"
      o: addr=10.100.50.20,rw,noatime,rsize=1048576,wsize=1048576,tcp,timeo=14
      device: ":/nextcloud/data/dennis/files/Photos/Camera"

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
      - "photoprism-alice:/photoprism/originals/alice:ro"
      - "photoprism-dennis:/photoprism/originals/dennis:ro"
      - type: volume
        source: photoprism-storage
        target: /photoprism/storage
        volume:
          nocopy: true

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

```
## Key mentions
The key thing to note in the above docker-compose configuration is the `volume` section of Photoprism.
The two volumes `photoprism-<user>` would be mapped to the same location where the Nextcloud data of the users are. In this example, our nextcloud data is mapped to an NFS share. Photoprism also uses the same root path but drilled into it. You can also notice that the data volumes are mapped as readonly in Photoprism.

That's it! Now you can go into the Photoprism app and find the folders in `Originals` section.