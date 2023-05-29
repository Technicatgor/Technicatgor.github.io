---
title: Running A DNS Server - Bind9
date: 2023-05-27 21:00:00 +800
categories: [Docker,Bind9]
tags: [docker,dns,bind9]
---

![bind9](/assets/img/bind9.png)
BIND 9 is a very flexible, full-featured DNS system \
You can deploy BIND for your own local network. \
I will using ubuntu/bind9 docker image to deploy.

## Prerequisite
- Docker / Podman (alternative)
- disable local dns service 
- Prepare your domain
> you can use the fake domain that means not publicly resolvable, but will not allow you to issue trusted SSL certificates.
> So I recommend using a real public domain 
- Basic DNS knowledge 
> you can read the [server-world](https://www.server-world.info/en/note?os=Ubuntu_23.04&p=dns&f=1) for learning

## Disable local DNS service
Edit resolved.conf
```
sudo vim /etc/systemd/resolved.conf
```
Uncomment and change to no

```
[Resolve]
...

DNSStubListener=no

...
```

Restart the service
```
sudo systemctl restart systemd-resolved
```

## Setup
Create a bind9 dir
```
mkdir bind9-dns && cd bind9-dns
vim docker-compose.yml
```
docker-compose.yml
```yml
version: "3"

services:
  bind9:
    container_name: bind9-dns
    image: ubuntu/bind9:latest
    environment:
      - BIND9_USER=root
      - TZ=Asia/Hong_Kong
    ports:
      - "53:53/tcp"
      - "53:53/udp"
    volumes:
      - ./config:/etc/bind
      - ./cache:/var/cache/bind
      - ./records:/var/lib/bind
    restart: unless-stopped
```
{: file="bind9-dns/docker-compose.yml"}

## Config 
Create the main config file, `sudo vim ./config/named.conf`
```conf
acl internal {

options {
  forwarders {
    1.1.1.1;
    1.0.0.1;
  };
  allow-query { internal; }; using acl
};

zone "local.example.com" IN {
  type master;
  file "/etc/bind/local-example-com.zone"; must be same as the zone name
};
```

## Prepare the zone file
Create the zone file, `sudo vim ./config/local-example-com.zone`
docs in [bind9](https://bind9.readthedocs.io/en/v9.18.15/chapter3.html#example-com-base-zone-file)
```conf
$TTL 2d

$ORIGIN local.example.com.

@             IN     SOA    ns.local.example.com. (
                            2023052700     ; serial
                            12h            ; refresh
                            15m            ; retry
                            3w             ; expire
                            2h )           ; minimum TTL

              IN     NS     ns.local.example.com.

ns            IN     A      192.168.0.53

dev-srv       IN     A      192.168.0.10
*.dev-srv     IN     A      192.168.0.10

prod-srv      IN     A      192.168.0.210

```
## Add your DNS Records
According to the following examples, you can add additional DNS Records, defined in the [IANA's DNS Resource Records TYPEs](https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml#dns-parameters-4).

## Start the container
```
docker-compose up -d
```

## Test
Using dig
```sh
dig @192.168.0.53 dev-srv.local.example.com

; <<>> DiG 9.18.12-0ubuntu0.22.04.1-Ubuntu <<>> @192.168.0.53 dev-srv.local.example.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 31161
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: f8406044c1fcc44601000000647208c475d03a9d7a83ebf1 (good)
;; QUESTION SECTION:
;dev-srv.local.example.com.    IN      A

;; ANSWER SECTION:
dev-srv.local.example.com. 172800 IN   A       192.168.0.10

;; Query time: 0 msec
;; SERVER: 192.168.0.53#53(192.168.0.53) (UDP)
;; WHEN: Sat May 27 21:42:28 HKT 2023
;; MSG SIZE  rcvd: 99
{}
```

using nslookup
```sh
nslookup dev-srv.local.example.com 192.168.0.53

Server:  192.168.0.10
Address:  192.168.0.10#53

Name:    dev-srv.local.example.com
Address:  192.168.0.10
```
## References
- [Server-world](https://www.server-world.info/en/note?os=Ubuntu_23.04&p=dns&f=1)
- [Bind9 Configuration and Zone Files](https://bind9.readthedocs.io/en/v9_18_10/chapter3.html)
- [IANA's DNS Resource Records TYPEs](https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml#dns-parameters-4)

