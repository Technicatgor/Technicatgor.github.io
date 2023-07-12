---
layout: post
title: Monitoring WatchTower In Grafana
date: 2023-07-12 16:00:00 +800
categories: [Kubernetes,Monitoring]
tags: [kubernetes,monitoring,prometheus,grafana,dashboard,watchtower]
---

## WatchTower 
![banner](https://marc.tv/media/2020/02/docker-watchtower.jpg)
A process for automating Docker container base image updates.\

## Prerequisite
1. Docker
2. Prometheus
> My Prometheus is host in my k3s cluster. Also it is a same way to configure with prometheus config.
3. Grafana
4. Discord(for notification)

## Installation
1. Create `docker-compose.yml` for deploy WatchTower.\

```yaml
{% raw %}
version: '3.8'
services:
  watchtower:
    image: containrrr/watchtower
    container_name: watchtower
    ports:
      - 8080:8080
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      WATCHTOWER_SCHEDULE: "0 0 1 * * *" # In everyday 01:00
      TZ: Asia/Hong_Kong
      WATCHTOWER_HTTP_API_TOKEN: "your-token"
      WATCHTOWER_HTTP_API_METRICS: "true"
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
{% endraw %}
```

> if you want to monitor remote server, add below line into environment.\
`DOCKER_HOST: "tcp://remote-docker-server:2375"`

2. Up the container `docker-compose up -d`

3. Check the process, it will show like this below

```
94345236d0d4   containrrr/watchtower           "/watchtower"            1 days ago      Up 1 days     0.0.0.0:8080->8080/tcp                                     watchtower
```

## Configure Prometheus 
1. Edit your prometheus.yml

```
scrape_configs:
 - job_name: watchtower
   scrape_interval: 5s
   metrics_path: /v1/metrics
   bearer_token: your-token
   static_configs:
     - targets:
       - 'dockersrv:8080'
```
2. Go to prometheus webui to check the target status.\
`http://prometheus:9090/targets?search=` \
something like below,
![watchtower-01](/assets/img/watchtower-01.png)

3. Then logon Grafana and import watchtower dashboard.
![watchtower-02](/assets/img/watchtower-02.png)

---
Kubernetes prometheus config
update soon...

###  In Addition
Enable TCP port 2375 for external connection:
---
1. Create `daemon.json` file in `/etc/docker`:

```
{"hosts": ["tcp://0.0.0.0:2375", "unix:///var/run/docker.sock"]}
```

2. Add `/etc/systemd/system/docker.service.d/override.conf`

```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd
```

3. Reload the systemd daemon:

```
systemctl daemon-reload
```

4. Restart docker:

```
systemctl restart docker.service
```
5. Test your port is open with another device, I use netcat for scan. you can use telnet also

```
nc -zv dockersrv 2375
```

```
Connection to dockersrv (10.0.50.11) 2375 port [tcp/*] succeeded!
```
