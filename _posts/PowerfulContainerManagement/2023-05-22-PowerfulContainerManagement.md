---
layout: post
title: "Powerful Container Management - Portainter"
date: 2023-05-25 13:00:00 +800
categories: [Docker,Portainter]
tags: [docker,portainer]
---

![portainer-banner](https://www.portainer.io/hubfs/Edge%20Aug22/edge-mockup.png)

Containers are the way the world builds modern software applications, and Portainer is the way the world manages containers. With an intuitive UI, backed by codified best practices and cloud-native design patterns, Portainer reduces the operational burden of multi-cluster container management.

## Prerequisite
- [docker](https://technicatgor.github.io/posts/DockerGuide/)


## installation
you can go my [launcher](https://github.com/Technicatgor/launcher) repo.
```
version: '3'

services:
  portainer:
    image: portainer/portainer-ce
    container_name: portainer
    ports:
      - 9443:9443
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - proxy
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data:/data
networks:
  proxy:
    external: true

```
up the container
```
docker-compose up -d
```

## Managet Docker
![portainer-1](/assets/img/portainer-1.png)
This means the portainer is reading /var/run/docker.sock locally. Also we can add the existing enviroments, look at the end.

## Container Status
![portainer-2](/assets/img/portainer-2.png)
From the screen, you will be able to see the important information such as container ip, exposure port and the image. It likes `docker inspect <container>`

## Create Container
![portainer-3](/assets/img/portainer-3.png)
You can deploy a container with the page. But I like to do it in cli.

## Manage Volumes
![portainer-4](/assets/img/portainer-4.png)
You can see all the volumes here. In cli is `docker volume ls` \
if some volumes unused it will show the unused tag and you can remove that easily.
You can click that show the information. 

## Manage Networks
![portainer-5](/assets/img/portainer-5.png)
You will see all the networks here, if you are using docker-compose that will create automatically. Otherwise, you need to `docker network create <network>` first.
And you can click inside to inspect which container is using this network. In cli `docker network inspect <network>`

## View Logs
![portainer-6](/assets/img/portainer-6.png)
You can click the paper icon to view that container logs. 
In cli `docker logs <container>`

## Icons
![portainer-7](/assets/img/portainer-7.png)

## Add more Environments
On side menu, click the setting > Environments
![portainer-8](/assets/img/portainer-8.png)
You can add the existing enviroments here. And portainer is able to support K8s. Very cool. Alternatively, you can use lens as well.

![portainer-9](/assets/img/portainer-9.png)
Following the instruction, run the portainer agent on your existing host then input the information of your enviroments. Click connect is done.

## Summary 
Portainer is greate IDE of container management tool. And it supports LDAP auth function on your enviroments securely
