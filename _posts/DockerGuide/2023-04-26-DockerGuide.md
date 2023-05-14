---
layout: post
title: "Docker Guide"
date: 2023-04-26 10:00:00 +800
categories: [Docker,Guide]
tags: [docker]
---

![docker-banner](/assets/img/docker-banner.png)

Docker 是一種軟體平台，可讓您快速地建立、測試和部署應用程式。Docker 將軟體封裝到名為容器的標準化單位，其中包含程式庫、系統工具、程式碼和執行時間等執行軟體所需的所有項目。使用 Docker，您可以將應用程式快速地部署到各種環境並加以擴展，而且知道程式碼可以執行。

## Installation

### Ubuntu and Debian 
install-docker.sh
```sh
echo "   ################################"
echo "      Installing System Updates... "
echo "   ################################"
sudo apt update && sudo apt upgrade -y > ~/docker-script-install.log 2>&1 &

pid=$! # Process Id of the previous running command
spin='-\|/'
i=0
while kill -0 $pid 2>/dev/null
do
    i=$(( (i+1) %4 ))
    printf "\r${spin:$i:1}"
    sleep .1
done
printf "\r"

echo "   ################################"
echo "   Installing Preq Packages..."
echo "   ################################"
sleep 3s

sudo apt install curl wget git -y >> ~/docker-script-install.log 2>&1

sleep 3s
echo "   ################################"
echo "         Installing Docker"
echo "   ################################"
curl -fsSL https://get.docker.com | sh >> ~/docker-script-install.log 2>&1
sleep 2s
sudo usermod -aG docker "${USER}" >> ~/docker-script-install.log 2>&1
echo "   ################################"
echo "           Starting Docker"
echo "   ################################"
echo $(docker -v)
sudo systemctl enable docker >> ~/docker-script-install.log 2>&1
sudo systemctl start docker >> ~/docker-script-install.log 2>&1
sleep 3s

echo "   ################################"
echo "       Installing Docker-Compose"
echo "   ################################"
VERSION=$(curl --silent https://api.github.com/repos/docker/compose/releases/latest | grep -Po '"tag_name": "\K.*\d')

sudo curl -SL https://github.com/docker/compose/releases/download/$VERSION/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose

sleep 2
sudo chmod +x /usr/local/bin/docker-compose
echo $(docker-compose -v)

sleep 3s
```

## Create docker and add user to docker group
```sh
sudo groupadd docker
sudo usermod -aG docker ${USER}
```

## Deploy a service 

I deploy uptime-kuma for the example

### Create NFS docker volume

The simplest way to create and manage Docker volumes is using the docker volume command and its subcommands.

The syntax for creating an NFS Docker volume includes two options.

    The `--driver` option defines the local volume driver, which accepts options similar to the mount command in Linux.
    The `--opt` option is called multiple times to provide further details about the volume.

The details include:
- The volume type.
- The write mode.
- The IP or web address of the remote NFS server.
- The path to the shared directory on the server.

create the volume, type as nfs 
```sh
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.100.250,rw,nfsvers=4 \ # ip of my NAS
  --opt device=:volume1/docker/data/kuma \
  uptime-kuma
```

we can inspect the volume info
```sh 
docker inspect volume uptime-kuma
[
    {
        "CreatedAt": "2023-04-27T14:34:29+08:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/uptime-kuma/_data",
        "Name": "uptime-kuma",
        "Options": {
            "device": ":volume1/docker_data/uptime-kuma",
            "o": "addr=192.168.100.250,rw,nfsvers=4",
            "type": "nfs"
        },
        "Scope": "local"
    }
]
```

run uptime-kuma and bind volume we created before
```
docker run -d --restart=always -p 3001:3001 -v uptime-kuma:/app/data --name uptime-kuma louislam/uptime-kuma:1
```

## Docker-compose
we can also use docker-compose.yml to deploy 
```
version: '3.3'

services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    volumes:
      - uptime-kuma:/app/data
    ports:
      - 3001:3001
    restart: always

volumes:
  uptime-kuma:
    driver_opts:
      type: nfs
      o: addr=192.168.100.250,nfsvers=4
      device: :/volume1/docker_data/uptime-kuma
```

## Backup Image
ls the images 
```
docker images
REPOSITORY         TAG       IMAGE ID       CREATED             SIZE
hello              latest    a8f72c57ea36   About an hour ago   918MB
```
save hello:latest to .tar
```
docker save hello:latest > hello.tar
```
if the image is too big, we can use gzip compress it 
```
docker save hello:lateset | gzip -9 > hello.tar.gz

$ ls -la -h
-rw-------    1 root     root      897.0M Apr 27 12:03 hello.tar
-rw-r--r--    1 root     root      327.3M Apr 27 12:05 hello.tar.gz
```

##  Restore Image
```
docker load < hello.tar.gz
```



## Uninstallation
```
sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras

# Images, containers, volumes, or custom configuration files on your host aren’t automatically removed. To delete all images, containers, and volumes: 
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```

