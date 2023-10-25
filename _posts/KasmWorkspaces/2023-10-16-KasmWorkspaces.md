---
layout: post
title: Kasm - The Container Streaming Platform
date: 2023-10-16 09:00:00 +800
categories: [Container]
tags: [container, homelab]
image:
  path: /assets/img/headers/Kasm_cover.jpeg
  lqip: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA8gAAAH4CAIAAAAZ1VPRAALJIklEQVR4Aeyah5IbuxVE0aQ3h/eUv8L//1MOm/NumxaKp9TVZInO4e1d1QgDXNyMHgyG+vj59+Od
---

## Getting Started
Kasm Workspaces provides browser-based access to on-demand containerized desktops and applications.

General information about the Workspaces platform is available at https://kasmweb.com

## Requirements
Supported OS
- Ubuntu 18.04/20.04/22.04 (amd64/arm64)
- Debian 10/11/12 (amd64/arm64)
- CentOS 7/8/9 (amd64/arm64)
- Oracle Linux 7/8/9 (amd64/arm64)
- Raspberry Pi OS 10/11/12 (amd64/arm64)

Resource
- CPU (2cores)
- Memory (4GB)
- Storage (50GB SSD)

Docker (> 18.06)
Docker Compose (> 2.1.1)

## Installation
```
cd /tmp
curl -O https://kasm-static-content.s3.amazonaws.com/kasm_release_1.14.0.3a7abb.tar.gz
tar -xf kasm_release_1.14.0.3a7abb.tar.gz
sudo bash kasm_release/install.sh
```
- Log into the Web Console at https://server-ip
- The Default username are admin@kasm.local and user@kasm.local. And the password is randomly generate, you can find it on cli.
![login](https://kasmweb.com/docs/latest/_images/login5.webp)

## Upgrade
- [References](https://kasmweb.com/docs/latest/upgrade/single_server_upgrade.html#automated-upgrade)

## Admin Guide
- [References](https://kasmweb.com/docs/latest/admin_guide.html)






