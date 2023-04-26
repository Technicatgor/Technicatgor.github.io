
---
layout: post
title: "Docker Guide"
date: 2023-04-26 10:00:00 +800
categories: [docker]
tags: [docker]

---
## Installation

### Ubuntu and Debian 
Run a one-line command
```sh
curl -fsSL https://get.docker.com | sudo sh -
```
## Uninstallation
```
sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras

# Images, containers, volumes, or custom configuration files on your host aren’t automatically removed. To delete all images, containers, and volumes: 
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```
