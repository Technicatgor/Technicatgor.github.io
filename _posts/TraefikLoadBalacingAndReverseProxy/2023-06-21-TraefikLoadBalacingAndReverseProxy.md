---
layout: post
title: "Traefik - Load Balancing and Reverse Proxy"
date: 2023-06-21 16:00:00 +800
categories: [Networking,ReverseProxy]
tags: [docker,traefik]
---

![banner](https://doc.traefik.io/traefik/assets/img/traefik-architecture.png)
This blog will set up a simple reverse proxy and load balancing with traefik. Traefik is good for containerize environment. We can use a label in `docker-compose.yml` for config.

## Prerequisite
- [DockerCompose](https://technicatgor.github.io/posts/DockerGuide/#installation)
- set up your domain name record in your dns / bind it in /etc/hosts

## Installation
create a `docker-compose.yml`
```
version: "3.8"

services:
  traefik:
    image: traefik:latest
    ports:
      - 80:80
      - 443:443
      - 8080:8080
    networks:
      - proxy
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data/traefik.yml:/traefik.yml:ro

    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.entrypoints=http"
      - "traefik.http.routers.traefik.rule=Host(`traefik.techcat.local`)"

networks:
  proxy:
    external: true
```
create a `./data/traefik.yml`
```
api:
  dashboard: true
  insecure: true

entryPoints:
  http:
    address: ":80"
  https:
    address: ":443"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
```
## Dashboard
go `http://traefik.local:8080`
![traefik-dashboard](/assets/img/traefik-01.png)

## Demo
![flow](/assets/img/traefik-04.png)
go back `docker-compose.yml` file, add whoami container \
whoami default is using 80 port for web server, so no need to use load balancing in traefik.
```
...

whoami:
    image: "traefik/whoami"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.techcat.local`)"
      - "traefik.http.routers.whoami.entrypoints=http"
...

```
go `http://whoami.techcat.local`
![traefik-whoami](/assets/img/traefik-02.png)

## Middleware
We can add a middleware for add prefix, redirect and basic auth. \
reference: [https://doc.traefik.io/traefik/middlewares/http/overview/](https://doc.traefik.io/traefik/middlewares/http/overview/) \
I use basic auth for example.\
docker-compose.yml
```
  traefik:
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data/traefik.yml:/traefik.yml:ro
      - ./data/usersfile:/usersfile:ro

...

whoami:
    image: "traefik/whoami"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.techcat.local`)"
      - "traefik.http.routers.whoami.entrypoints=http"
      - "traefik.http.routers.whoami-secure.middlewares=test-auth"
      - "traefik.http.middlewares.test-auth.basicauth.usersfile=/usersfile"

...
```
create a usersfile, use htpasswd to generate your id & password \ 
or go [https://hostingcanada.org/htpasswd-generator/](https://hostingcanada.org/htpasswd-generator/) to generate it.

`./data/usersfile`
```
test:$2y$10$FPU6eAgBlmAsH6gXC3DTguyW6uK0J8ID7UteimxZLqGGyDSfq8T/W
```
reload your traefik docker-compose
```
docker-compose up -d --force-recreate
```
Now go http://whoami.techcat.local, you will get the login popup.

## Listen Host & Path
```
version: "3.8"
services:
  traefik:
    image: "traefik:latest"
    ports:
      - "80:80"
      - "8089:8080"
      - "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./data/traefik.yml:/traefik.yml:ro"
      - "./usersfile:/usersfile:ro"

  whoami:
    image: "traefik/whoami"
    labels:
      # http section
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.techcat.local`)"
      - "traefik.http.routers.whoami.entrypoints=http" 

  whoami2:
    image: "nginxdemos/hello"
    deploy:
      replicas: 1
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami2.rule=Host(`whoami.techcat.local`) && Path(`/whoami2`)"
      - "traefik.http.routers.whoami2.entrypoints=http"
```
```
docker-compose up -d
```
Now go http://whoami.techcat.local/whoami2 will see below \
![traefik-whoami2](/assets/img/traefik-03.png)

## TLS
Now add a https section into whoami label. \
enable tls and set the true.
```
...

  whoami:
    image: "traefik/whoami"
    labels:
      # http section
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.techcat.local`)"
      - "traefik.http.routers.whoami.entrypoints=http" 
      # https section
      - "traefik.http.routers.whoami-secure.rule=Host(`whoami.techcat.local`)"
      - "traefik.http.routers.whoami-secure.entrypoints=https"
      - "traefik.http.routers.whoami-secure.tls=true"
...

```

## Dynamic config
Define TLS certificate add into `data/dynamic-config.yml` \
docs: [https://doc.traefik.io/traefik/https/tls/](https://doc.traefik.io/traefik/https/tls/)
```
tls:
  certificates:
    certFile = "/certs/cert.crt"
    keyFile = "/certs/cert.key"
```
map a cert and dynamic-config.yml volume into docker-compose.yml
```
...
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./data/traefik.yml:/traefik.yml:ro"
      - "./data/dynamic-config.yml:/dynamic-config.yml:ro"
      - "./certs:/certs"
...

```
docker-compose.yml
```
version: "3.8"

services:
  traefik:
    image: "traefik:latest"
    ports:
      - "80:80"
      - "8080:8080"
      - "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./data/traefik.yml:/traefik.yml:ro"
      - "./data/dynamic-config.yml:/dynamic-config.yml:ro"
      - "./certs:/certs"
      - "./usersfile:/usersfile:ro"

  whoami:
    image: "traefik/whoami"
    labels:
      - "traefik.enable=true"

      # http section
      - "traefik.http.routers.whoami.rule=Host(`whoami.techcat.local`)"
      - "traefik.http.routers.whoami.entrypoints=http"
      - "traefik.http.routers.whoami.middlewares=https-redirect"
      - "traefik.http.middlewares.https-redirect.redirectscheme.scheme=https"

      # https section
      - "traefik.http.routers.whoami-secure.rule=Host(`whoami.techcat.local`)"
      - "traefik.http.routers.whoami-secure.entrypoints=https"
      - "traefik.http.routers.whoami-secure.tls=true"
      - "traefik.http.routers.whoami-secure.middlewares=test-auth"
      - "traefik.http.middlewares.test-auth.basicauth.usersfile=/usersfile"
```
