---
layout: post
title: Enable TOTP on your Proxmox
date: 2023-08-18 10:00 +800
categories: [Hypervisors, Proxmox]
tags: [proxmox, homelab]
image:
  path: /assets/img/headers/totp-pve.png
  lqip: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA8gAAAH4CAIAAAAZ1VPRAALJIklEQVR4Aeyah5IbuxVE0aQ3h/eUv8L//1MOm/NumxaKp9TVZInO4e1d1QgDXNyMHgyG+vj59+Od
---

## Create your TOTP

Login your pve, and click Datacenter > Permissions > Two Factor
![TOTP-1](/assets/img/TOTP-1.png)

## Add your first TOTP

Click add then pop the QR code window. Enter the description and using eg.Google Authenicator by your phone.
![TOTP-2](/assets/img/TOTP-2.png)
After your scan it by your mobile then enter the verification code.

Now you can logout and re-login that will request your TOTP Authenication.

## Setup on SSH

We just setup 2FA in web-UI login. Now we setup TOTP Authenication on SSH also.

1. To install Google Authenticator on pve

- Debian/Ubuntu:

```
sudo apt-get install libpam-google-authenticator
```

- RHEL/CentOS:

```
sudo yum install google-authenticator
```

2. Configure Google Authenticator and synhronize it with your mobile phone

```
google-authenticator
```

After that will be prompted several questions. It suggested to answer "yes" on all questions.

3. Open Google Authenticator
   When you start this application, choose the  'Enter provided key'  option and write your secret key there.

4. Enable two-factor authentication for SSH protocol

- edit `/etc/pam.d/sshd file`, paste a command below common-auth section.

```
@include common-auth

auth required pam_google_authenticator.so
```

5. Open the `/etc/ssh/sshd_config`

```
ChallengeResponseAuthentication yes
PasswordAuthentication no
```

6. Restart SSH service

```
service sshd restart
```

## Reference

[https://kb.nomachine.com/AR12L00828](https://kb.nomachine.com/AR12L00828)
