---
layout: post
title: Using Tailscale For Homelab VPN Connectivity
date: 2023-04-24 17:00:00 +800
categories: [Networking,VPN]
tags: [homelab,vpn,tailscale,pfsense]
---
## What is Tailscale?
Tailscale is a VPN service that makes the devices and applications you own accessible anywhere in the world, securely and effortlessly. It enables encrypted point-to-point connections using the open source WireGuard protocol, which means only devices on your private network can communicate with each other.

I am using Tailscale for VPN on my all devices. My pfsense is installed Tailscale package that is set up as an exit node and advertises my home network.
![subnets-router](https://tailscale.com/kb/1019/subnets/subnets.png)

## Set Up Tailscale on pfSense
1. Select System, then Package Manager.
![tailscale-01](/assets/img/tailscale-01.png)

2. Search for Tailscale, then install the Tailscale package.
![tailscale-02](/assets/img/tailscale-02.png)

3. Select VPN, then Tailscale to launch the Tailscale settings.
![tailscale-03](/assets/img/tailscale-03.png)

4. At this point, we need to configure the pre-authentication key. This can be created on the Tailscale website. If you don’t already have an account, create one, then log in and select Settings, then Keys.
![tailscale-04](/assets/img/tailscale-04.png)

5. Select generate auth key so that we can create the key for pfSense. Select Generate Key (the settings can stay as default).
![tailscale-05](/assets/img/tailscale-05.png)

6. After the key has been generated, copy it, then go back to the Authentication section of Tailscale on pfSense.
![tailscale-06](/assets/img/tailscale-06.png)

7. Paste the key that was just created, then select save.
![tailscale-07](/assets/img/tailscale-07.png)

8. After saving, select Settings, then enable Tailscale and Save.
![tailscale-08](/assets/img/tailscale-08.png)

## Exit Node
1. Inside the Tailscale settings on pfSense, enable the offer to be an exit node for outbound internet traffic from the Tailscale network option. Also, set the Advertised Routes as your local subnet, then save.
![tailscale-09](/assets/img/tailscale-09.png)

2. On the Tailscale website, select Machines, then Edit Route Settings.
![tailscale-10](/assets/img/tailscale-10.png)

3. Approve all subnet routes and Select use as exit node. The exit node functionality is now set up and can be used by client devices.
![tailscale-11](/assets/img/tailscale-11.png)

4. Select the DNS tab, you can set your local DNS server as Global namesever and override it.(if you have pihole, you can set the pihole's ip in there.)
![tailscale-12](/assets/img/tailscale-12.png)

5. Tailscale is now configured! You can now add other devices or simply connect to Tailscale from an external network to access all of your local devices.

## Cli command
[Advertise your subnet routes](https://tailscale.com/kb/1019/subnets/#step-2-connect-to-tailscale-as-a-subnet-router)
```
sudo tailscale up --advertise-routes=10.0.0.0/24,10.0.1.0/24
```
[Advertise the device as an exit node](https://tailscale.com/kb/1103/exit-nodes/#advertise-the-device-as-an-exit-node)
```
sudo tailscale up --advertise-exit-node
```
[For more details set up the ACL policy](https://tailscale.com/kb/1018/acls/#introduction)
