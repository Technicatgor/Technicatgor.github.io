---
layout: post
title: Obsidian LiveSync
date: 2025-08-20 16:00:00 +800
categories: [Tool]
tags: [Obsidian]
image:
  path: /assets/img/headers/obd-livesync-cover.png
  lqip: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA8gAAAH4CAIAAAAZ1VPRAALJIklEQVR4Aeyah5IbuxVE0aQ3h/eUv8L//1MOm/NumxaKp9TVZInO4e1d1QgDXNyMHgyG+vj59+Od
---

### Deploy locally

1. Prepare

```bash
# Creating the save data & configuration directories.
mkdir couchdb-data
mkdir couchdb-etc
```

2. Create a `compose.yml` file with the following added to it

#### without proxy

```yaml
services:
  couchdb:
    image: couchdb:latest
    container_name: couchdb-for-ols
    environment:
      - COUCHDB_USER=<INSERT USERNAME HERE> #Please change as you like.
      - COUCHDB_PASSWORD=<INSERT PASSWORD HERE> #Please change as you like.
    volumes:
      - ./couchdb-data:/opt/couchdb/data
      - ./couchdb-etc:/opt/couchdb/etc/local.d
    ports:
      - 5984:5984
    restart: unless-stopped
```

#### with Traefik

```yaml
services:
  couchdb:
    image: couchdb:latest
    container_name: obsidian-livesync
    environment:
      - COUCHDB_USER=username
      - COUCHDB_PASSWORD=password
    volumes:
      - ./data:/opt/couchdb/data
      - ./local.ini:/opt/couchdb/etc/local.ini
    # Ports not needed when already passed to Traefik
    #ports:
    #  - 5984:5984
    restart: unless-stopped
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      # The Traefik Network
      - "traefik.docker.network=proxy"
      # Don't forget to replace 'obsidian-livesync.example.org' with your own domain
      - "traefik.http.routers.obsidian-livesync.rule=Host(`obsidian-livesync.example.org`)"
      # The 'websecure' entryPoint is basically your HTTPS entrypoint. Check the next code snippet if you are encountering problems only; you probably have a working traefik configuration if this is not your first container you are reverse proxying.
      - "traefik.http.routers.obsidian-livesync.entrypoints=websecure"
      - "traefik.http.routers.obsidian-livesync.service=obsidian-livesync"
      - "traefik.http.services.obsidian-livesync.loadbalancer.server.port=5984"
      - "traefik.http.routers.obsidian-livesync.tls=true"
      # Replace the string 'letsencrypt' with your own certificate resolver
      - "traefik.http.routers.obsidian-livesync.tls.certresolver=letsencrypt"
      - "traefik.http.routers.obsidian-livesync.middlewares=obsidiancors"
      # The part needed for CORS to work on Traefik 2.x starts here
      - "traefik.http.middlewares.obsidiancors.headers.accesscontrolallowmethods=GET,PUT,POST,HEAD,DELETE"
      - "traefik.http.middlewares.obsidiancors.headers.accesscontrolallowheaders=accept,authorization,content-type,origin,referer"
      - "traefik.http.middlewares.obsidiancors.headers.accesscontrolalloworiginlist=app://obsidian.md,capacitor://localhost,http://localhost"
      - "traefik.http.middlewares.obsidiancors.headers.accesscontrolmaxage=3600"
      - "traefik.http.middlewares.obsidiancors.headers.addvaryheader=true"
      - "traefik.http.middlewares.obsidiancors.headers.accessControlAllowCredentials=true"

networks:
  proxy:
    external: true
```

3. Run the Docker Compose file to boot check

```bash
docker compose up -d
```

### Go to CouchDB admin page

- Go to your server ip eg. `http://192.168.1.0:5984/_utils`, login with your credentials created in compose file.
  ![/assets/img/obd-01.png](/assets/img/obd-01.png)

- You will see your db in CouchDB
- Open Obsidian apps and browse the LiveSync plugin
  ![/assets/img/obd-02.png](/assets/img/obd-02.png)

- Click Option and setup your connection.
- Server URI, Username, Password, Database Name
  ![/assets/img/obd-03.png](/assets/img/obd-03.png)

- Test Database Connection, Encryption
  ![/assets/img/obd-04.png](/assets/img/obd-04.png)

- Synchronization Method
  i. Change Live Sync method
  ![/assets/img/obd-05.png](/assets/img/obd-05.png)

### Install Obsidian on mobile

- Create a new vault and install the LiveSync plugin and configure the connection. The same setting in desktop side, then click **Fetch** button Fetch Settings.
  ![/assets/img/obd-06.png](/assets/img/obd-06.png)

Done! It will sync the notes in live.

ref link: <https://github.com/vrtmrz/obsidian-livesync>
