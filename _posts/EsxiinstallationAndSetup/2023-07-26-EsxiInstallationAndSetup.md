---
layout: post
title: ESXi Installation And Setup
date: 2023-07-26 10:00 +800
categories: [Hypervisor,ESXi]
tags: [vmware,homelab]
---

![banner](https://radiant.in/wp-content/uploads/2023/05/VMware-vSphere-banner.jpg)

VMware vSphere is a powerful and comprehensive virtualization platform that enables organizations to create and manage virtualized infrastructures. It provides a range of features and capabilities that optimize resource utilization, enhance scalability, and improve operational efficiency.

## Download
Download the VMware ESXi 7 ISO file for installation from the following site.

[https://customerconnect.vmware.com/evalcenter?p=free-esxi7](https://customerconnect.vmware.com/evalcenter?p=free-esxi7)

## Configure Networking
1. Login with root user account on ESXi console, select [Configure Management Network].
![network](/assets/img/esxi-01.png)

2. Configure IPv4, IPv6 networking, DNS & Suffix.
![console](/assets/img/esxi-02.png)

## ESXi Host Client
The ESXi host client is a web interface that allows you to manage your ESXi host. You can access the host client by entering the IP address of your ESXi host into a web browser. The new host client with ESXi 8.0 has had an enormous face lift and looks great with many features and capabilities.
![webui](/assets/img/esxi-03.png)

## SSH on ESXi Host
Secure Shell (SSH) enables secure remote management of your ESXi host.

- Enabling SSH: Host > Actions > Services > Enable Secure Shell (SSH).
![ssh](/assets/img/esxi-04.png)

## Patching the VMware ESXi Host
```
esxcli software profile get
esxcli network firewall ruleset set -e true -r httpClient
esxcli software profile update -d https://hostupdate.vmware.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml -p <Name>
```

## NTP Server Configuration
- Host > Manage > Time & date > Edit NTP settings.
![ntp](/assets/img/esxi-05.png)
- Start the NTP service under Manage > Services > ntpd
![ntp2](/assets/img/esxi-06.png)

## ESXi Logs to a Syslog Server 
- Navigate to Manage > System > Advanced System Settings in the ESXi host client, and modify the Syslog.global.logHost setting with the IP address of your syslog server.
![syslog](/assets/img/esxi-07.png)
In cli: 
```
esxcli system syslog config set --loghost=10.0.50.30
```

- Restarting the Syslog Service: Manage > Services > Syslog in the ESXi host client.
```
esxcli system syslog reload
```
## Exploring iSCSI Software Adapters
Software iSCSI adapters enable your host to connect to an iSCSI storage device.

- Adding the Software iSCSI Adapter: Storage > Storage Adapters > Add software iSCSI adapter in the ESXi host client.
- Configuring iSCSI Software Adapter:  Storage > Storage Adapters, select your iSCSI adapter, and click on Targets to add your iSCSI target server.
![iscsi](/assets/img/esxi-08.png)
After you add the iSCSI Software adapter, you will see the IQN that is generated for the iSCSI initiator.
![iscsi2](/assets/img/esxi-09.png)
## Adding a Standalone ESXi Host to a vCenter Server
vCenter Server allows centralized management of multiple ESXi hosts.

- In the vSphere Client, right-click on your datacenter or cluster and select Add Host.
- After adding, the hosts resources become part of the datacenter or cluster and can be managed centrally.
![vcenter](/assets/img/esxi-10.png)
The Add Host wizard will launch. Enter the host name or IP address.
![vcenter2](/assets/img/esxi-11.png)

## Licensing Your ESXi Host

- ESXi host client under Manage > Licensing > Assign license.
- In vCenter Server: navigate to Hosts and Clusters, select your host, then Manage > Settings > Licensing > Edit.


## Network Setting
- VMkernel NIC: Management > Network > VMkernel NICs > VMkernel NIC
![vmnic](/assets/img/esxi-12.png)
![vmnic2](/assets/img/esxi-13.png)

- Add Virtual Switch: Management > Network > Virtual Switches > Add standard virtual switch
![vswitch](/assets/img/esxi-14.png)
![vswitch2](/assets/img/esxi-15.png)

- Add Port Group: Management > Network > Port groups > Add port group button. Next, input new portgroup name and select virtual switch.
![portgp](/assets/img/esxi-16.png)
![portgp2](/assets/img/esxi-17.png)

- Add Uplink
Management > Network > Virtual Switches > Add uplink
![uplink](/assets/img/esxi-18.png)
![uplink2](/assets/img/esxi-19.png)


