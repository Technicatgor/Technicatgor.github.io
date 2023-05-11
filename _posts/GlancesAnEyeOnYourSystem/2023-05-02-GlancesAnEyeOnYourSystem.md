---
layout: post
title: Glances - An Eye on your system
date: 2023-05-02 17:00:00 +800
categories: [Monitoring,Glances]
tags: [glances,dashy]
---

Glances is an open-source system cross-platform monitoring tool

## Installation
```
apt install python3 python3-pip -y
pip3 install bottle
pip3 install glances
```
## Create a service
`vim /etc/systemd/system/glances.service`

```
[Unit]
Description = Glances in Web Server Mode
After = network.target

[Service]
ExecStart = /usr/local/bin/glances  -w  -t  5

[Install]
WantedBy = multi-user.target
```
{! file=/etc/systemd/system/glances.service}
```
systemctl enable glances.service  # startup on boot
systemctl start glances.service  # start glances
systemctl status glances.service  # check glances status
systemctl restart glances.service  # restart glances
```


## Addition
Glances with Dashy
```yml
pageInfo:
  title: My Dashboard
  description: Welcome to My dashboard!
  navLinks:
    - title: GitHub
      path: https://github.com/Lissy93/dashy
    - title: Documentation
      path: https://dashy.to/docs
appConfig:
  theme: color-block
  layout: auto
  iconSize: medium
  language: en
sections:
  - name: Getting Started
    icon: fas fa-rocket
    items:
      - title: ESXi
        description: VMware ESXi
        icon: hl-vmwareesxi
        url: 'https://192.168.1.1'
        target: newtab
        statusCheckAllowInsecure: true
        id: 0_1481_esxi
      - title: Proxmox
        description: Proxmox
        icon: hl-proxmox
        url: https://192.168.1.2:8006
        target: newtab
        id: 1_1481_proxmox
      - title: Docs
        description: Configuring & Usage Documentation
        provider: Dashy.to
        icon: far fa-book
        url: https://dashy.to/docs
        id: 2_1481_docs
      - title: Showcase
        description: See how others are using Dashy
        url: https://github.com/Lissy93/dashy/blob/master/docs/showcase.md
        icon: far fa-grin-hearts
        id: 3_1481_showcase
      - title: Config Guide
        description: See full list of configuration options
        url: https://github.com/Lissy93/dashy/blob/master/docs/configuring.md
        icon: fas fa-wrench
        id: 4_1481_configguide
      - title: Support
        description: Get help with Dashy, raise a bug, or get in contact
        url: https://github.com/Lissy93/dashy/blob/master/.github/SUPPORT.md
        icon: far fa-hands-helping
        id: 5_1481_support
  - name: Today
    icon: far fa-smile-beam
    displayData:
      collapsed: false
      hideForGuests: false
    widgets:
      - type: clock
        options:
          timeZone: Asia/Hong_Kong
          format: en-GB
        id: 0_513_clock
      - type: weather
        options:
          apiKey: efdbade728b37086877a5e83442004db
          city: hongkong
        id: 1_513_weather
  - name: CPU History
    icon: fas fa-microchip
    displayData:
      cols: 2
    widgets:
      - type: gl-cpu-history
        options:
          hostname: http://192.168.1.2:61208
          limit: 60
        id: 0_1018_glcpuhistory
  - name: CPU & Memory
    icon: fas fa-tachometer
    widgets:
      - type: gl-current-cpu
        options:
          hostname: http://192.168.1.2:61208
        id: 0_967_glcurrentcpu
      - type: gl-current-mem
        options:
          hostname: http://192.168.1.2:61208
        id: 1_967_glcurrentmem
  - name: Disk Space
    icon: fas fa-hdd
    widgets:
      - type: gl-disk-space
        options:
          hostname: http://192.168.1.2:61208
        id: 0_919_gldiskspace
  - name: Network Interfaces
    icon: fas fa-ethernet
    widgets:
      - type: gl-network-interfaces
        options:
          hostname: http://192.168.1.2:61208
          limit: 500
        id: 0_1806_glnetworkinterfaces
  - name: System Load
    icon: fas fa-tasks-alt
    widgets:
      - type: gl-system-load
        options:
          hostname: http://192.168.1.2:61208
        id: 0_1061_glsystemload
```
{! file=conf.yml}
