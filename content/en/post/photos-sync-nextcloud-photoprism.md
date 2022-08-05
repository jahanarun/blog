---
date: "2022-08-04T22:32:03+05:30"
description: Simple way to sync photos from your mobile devices to nextcloud
featured_image: ""
tags: []
title: Photos Sync Nextcloud Photoprism
---
{{<strike>}}We all want to have our photos taken on our mobile phones to be available in some other cloud/storage server/computer, so that we could free up storage on our phones.

You will need to purchase plans in iCloud and Google Photos after exhuasting the free storage plan.
{{</strike>}}

Let's say you want all the photos taken by your family (with multiple devices) to be available in one place so that you can have a unified gallery look.

The ideal tools to use in this case would be [Photoprism](https://github.com/photoprism/photoprism) and [Nextcloud](https://github.com/nextcloud).

## Outline of what we will do
1. Configure Nextcloud
1. Configure Photoprism
1. Mount Nextcloud directories to Photoprism

```
      device: ":/docker/lidarr"
  torrents:
    driver: local
    driver_opts:
      type: "nfs4"
    driver: local
    driver_opts:
      type: "nfs4"
  jackett-config:
    driver: local
    driver_opts:
      type: "nfs4"
  cloudflared-config:
    driver: local
    driver_opts:
      type: "nfs4"
      device: ":/docker/cloudflared"

networks:
  frontend_vlan60:
    driver: ipvlan
    driver_opts:
      parent: "enp6s18.60"
      name: ipvlan60
    ipam:
      config:
        - subnet: 10.100.60.0/24
          gateway: 10.100.60.1
  backend:
    driver: bridge
    internal: true

services:
  cloudflared-torrent:
```