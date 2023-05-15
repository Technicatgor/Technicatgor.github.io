---
layout: post
title: "Self-Hosted Pihole on Kubernetes for a DNS server & Ads Blocker"
date: 2023-05-15 22:00:00 +800
categories: [Self-Hosted,Pi-Hole]
tags: [kubernetes,dns,pihole]
---

![pihole](https://community-assets.home-assistant.io/original/3X/e/8/e89b980ceaf3a94eeae7527ae9b64b6d9f478723.jpg)

The [Pi-hole®](https://pi-hole.net) is a DNS sinkhole that protects your devices from unwanted content without installing any client-side software.

## Preqre
- Helm 
- Kubernetes cluster, my homelab is using k3s-cluster.

## Installation
We will use mojo2600/pihole helm repo in [ArtifactHub](https://artifacthub.io/packages/helm/mojo2600/pihole)

helm add repo.
```
helm repo add mojo2600 https://mojo2600.github.io/pihole-kubernetes/
helm repo update
```

get the values.yml of mojo2600/pihole and configure it first.
```
helm show values mojo2600/pihole > values.yml
```
## Values.yml
I changed the DNS and https type to `LoadBalancer` and set the `loadBalancerIP: '10.0.50.77'`, cause I'm using metallb service.\
And configure the `storageClass: 'longhorn'` and the `adminPassword`
```
serviceDns:
  type: LoadBalancer 
  port: 53
  loadBalancerIP: "10.0.50.77"
  annotations:
    metallb.universe.tf/allow-shared-ip: pihole-svc

serviceWeb:
  http:
    enabled: true
    port: 80
  https:
    enabled: true
    port: 443
  type: LoadBalancer
  loadBalancerIP: 10.0.50.77
  annotations:
    metallb.universe.tf/allow-shared-ip: pihole-svc

persistentVolumeClaim:
  enabled: true
  accessModes:
    - ReadWriteOnce
  size: "2Gi"
  storageClass: "longhorn"

adminPassword: "P@ssw0rd"

extraEnvVars:
  TZ: Asia/Hong_Kong

DNS1: "1.1.1.1"
DNS2: "8.8.8.8"

podDnsConfig:
  enabled: true
  policy: "None"
  nameservers:
  - 127.0.0.1
  - 1.1.1.1
```

## Helm Install
```
helm install pihole mojo2600/pihole -n pihole --create-namespaces -f values.yml
```

## Web UI
access http://10.0.50.77/admin
![webui](/assets/img/pihole-1.png)

update the ads block list first
![webui](/assets/img/pihole-2.png)

you can setup your dns records as your local network dns server
![webui](/assets/img/pihole-3.png)

then bind your devices dns to pihole, you will see the traffice display on dashboard.
![webui](https://danielrampelt.com/images/blog/install-pihole-raspberry-pi-docker-ipv6/pihole-dashboard.png)

Also you can check the query log for the details
![webui](https://techwiser.com/wp-content/uploads/2019/09/query-log.jpg)
