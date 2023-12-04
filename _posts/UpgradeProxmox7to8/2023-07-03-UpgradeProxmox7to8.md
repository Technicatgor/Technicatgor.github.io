---
layout: post
title: Upgrade Proxmox 7 to 8
date: 2023-07-03 10:00:00 +800
categories: [Hypervisors,Proxmox]
tags: [proxmox,homelab]
---

![banner](https://logovectorseek.com/wp-content/uploads/2021/10/proxmox-server-solutions-gmbh-logo-vector.png)

## New Features
- Integrated Ceph Enterprise Repository
- Enhanced LDAP & Active Directory Synchronization
- Software-Defined Networking (SDN) Control
- Flexible Resource Mappings
- Two-Factor Authentication (2FA)
- Text-Based User Interface for Installer ISO
- x86-64-v2-AES Default CPU Type

## Checklist Script
```sh
pve7to8 --full
```

```sh
= SUMMARY =

TOTAL:    35
PASSED:   26
SKIPPED:  5
WARNINGS: 4
FAILURES: 0
```

## Upgrade
1. Check your Proxmox version to ensure you’re running the latest Proxmox VE 7.4 packages. The output should show that you’re on v7.4-15 or newer.
```sh
apt update
apt dist-upgrade
pveversion
```
2. Update all Debian and Proxmox VE repository entries to Bookworm:
```sh
sed -i 's/bullseye/bookworm/g' /etc/apt/sources.list
```
3. Run the command below.
```sh
sed -i -e 's/bullseye/bookworm/g' /etc/apt/sources.list.d/pve-install-repo.list
```
4. Verify these files were updated with `bookworm`
```sh
cat /etc/apt/sources.list
cat /etc/apt/sources.list.d/pve-install-repo.list
```
```sh
deb [arch=amd64] http://download.proxmox.com/debian/pve bookworm pve-no-subscription
```
5. Finally, run a final update command and then upgrade the system
```sh
apt update
apt dist-upgrade
```
6.  When the installation finishes, run the command below to reboot Proxmox
```sh
reboot now
```

## Disable no pve-no-subscription pop-up
```
sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/\1/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js && systemctl restart pveproxy.service
```

