---
layout: post
title: Automatically Update Docker Container Images with Watchtower
date: 2023-07-04 14:00 +800
categories: [Docker,Watchtower]
tags: [docker, watchertower]
---
![banner](https://marc.tv/media/2020/02/docker-watchtower.jpg)
Watchtower is an application that will monitor your running Docker containers and watch for changes to the images that those containers were originally started from. \
If watchtower detects that an image has changed, it will automatically restart the container using the new image.

## Prerequisite
- docker
- discord(optional) #for push notification 

## Installation
Create a docker-compose.yml
```yml
version: '3'
services:
  watchtower:
    image: containrrr/watchtower
    container_name: watchtower
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      WATCHTOWER_SCHEDULE: "0 0 1 * * *"
      TZ: Asia/Hong_Kong
      WATCHTOWER_CLEANUP: "true"
      WATCHTOWER_DEBUG: "true"
      WATCHTOWER_NOTIFICATION_REPORT: "true"
      WATCHTOWER_NOTIFICATION_URL: "discord://${TOKEN}@${CHANNEL_ID}"
      WATCHTOWER_NOTIFICATION_TEMPLATE: |
        {{- if .Report -}}
          {{- with .Report -}}
        {{len .Scanned}} Scanned, {{len .Updated}} Updated, {{len .Failed}} Failed
              {{- range .Updated}}
        - {{.Name}} ({{.ImageName}}): {{.CurrentImageID.ShortID}} updated to {{.LatestImageID.ShortID}}
              {{- end -}}
              {{- range .Fresh}}
        - {{.Name}} ({{.ImageName}}): {{.State}}
            {{- end -}}
            {{- range .Skipped}}
        - {{.Name}} ({{.ImageName}}): {{.State}}: {{.Error}}
            {{- end -}}
            {{- range .Failed}}
        - {{.Name}} ({{.ImageName}}): {{.State}}: {{.Error}}
            {{- end -}}
          {{- end -}}
        {{- else -}}
          {{range .Entries -}}{{.Message}}{{"\n"}}{{- end -}}
        {{- end -}}
```
Add your Discord channel id & token into `.env` \
The link of webhook look like this `https://discord.com/api/webhooks/1125487910304870422/V5aQwFzeAy5zcxMNykSy8PZXMmfUSCLlh6Bcf0DHCQzFlfvaKTmNDorFosUjj-d-NZIQ`
```env
TOKEN=V5aQwFzeAy5zcxMNykSy8PZXMmfUSCLlh6Bcf0DHCQzFlfvaKTmNDorFosUjj-d-NZIQ
CHANNEL_ID=1125487910304870422
```
docker compose up 
```sh
docker-compose up -d
```
You can see all watchtower logs using command `docker-compose logs` \
or using dizzle to check the logs \

Here is the logs

```
watchtower    | time="2023-07-04T01:01:08+08:00" level=info msg="Creating /traefik"
watchtower    | time="2023-07-04T01:01:08+08:00" level=debug msg="Starting container /traefik (41ca12056e84)"
watchtower    | time="2023-07-04T01:01:08+08:00" level=info msg="Creating /bind9-dns"
watchtower    | time="2023-07-04T01:01:08+08:00" level=debug msg="Starting container /bind9-dns (897a2525f167)"
watchtower    | time="2023-07-04T01:01:08+08:00" level=info msg="Creating /Dashy"
watchtower    | time="2023-07-04T01:01:08+08:00" level=debug msg="Starting container /Dashy (c65c40a2c6ba)"
watchtower    | time="2023-07-04T01:01:08+08:00" level=info msg="Removing image ab8f641699b4"
watchtower    | time="2023-07-04T01:01:09+08:00" level=info msg="Removing image 63d7224eb30e"
watchtower    | time="2023-07-04T01:01:09+08:00" level=info msg="Removing image dd7f62e750f4"
watchtower    | time="2023-07-04T01:01:09+08:00" level=info msg="Removing image e29ed7e30b7d"
watchtower    | time="2023-07-04T01:01:10+08:00" level=info msg="Session done" Failed=0 Scanned=7 Updated=4 notify=no
watchtower    | time="2023-07-04T01:01:10+08:00" level=debug msg="Scheduled next run: 2023-07-05 01:00:00 +0800 HKT"
```
