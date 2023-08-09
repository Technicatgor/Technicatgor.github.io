---
layout: post
title: Use iPerf3 To Test Network Bandwidth
date: 2023-08-09 10:00:00 +800
categories: [Network, iPerf3]
tags: [network, iperf3, docker]
---

![banner](https://www.redeszone.net/app/uploads-redeszone.net/2021/02/iperf-3-destacada.jpg)

## iPerf3
iPerf3 tests the bandwidth between any two networked computers to determine if the available bandwidth is large enough to support the transmission of an application.

iPerf3 is built on a client-server model and measures maximum User Datagram Protocol, TCP and Stream Control Transmission Protocol throughput between client and server stations. It can also be used to measure LAN and wireless LAN throughput.

## docker
I will use docker to run iPerf3. Here is the docker repo:\
[https://hub.docker.com/r/networkstatic/iperf3](https://hub.docker.com/r/networkstatic/iperf3)

## Server Side
run this command and listen service on port 5201
```
docker run  -it --rm --name=iperf3-server -p 5201:5201 networkstatic/iperf3 -s
```
Then will show:
```
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
```

## Client Side
Type your server's ip after the flag `-c`
```
docker run  -it --rm networkstatic/iperf3 -c 192.168.1.99
```
Then will show:
```
Connecting to host 192.168.1.99, port 5201
[  4] local 192.168.2.99 port 51148 connected to 192.168.1.99 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec  4.16 GBytes  35.7 Gbits/sec    0    468 KBytes
[  4]   1.00-2.00   sec  4.10 GBytes  35.2 Gbits/sec    0    632 KBytes
[  4]   2.00-3.00   sec  4.28 GBytes  36.8 Gbits/sec    0   1.02 MBytes
[  4]   3.00-4.00   sec  4.25 GBytes  36.5 Gbits/sec    0   1.28 MBytes
[  4]   4.00-5.00   sec  4.20 GBytes  36.0 Gbits/sec    0   1.37 MBytes
[  4]   5.00-6.00   sec  4.23 GBytes  36.3 Gbits/sec    0   1.40 MBytes
[  4]   6.00-7.00   sec  4.17 GBytes  35.8 Gbits/sec    0   1.40 MBytes
[  4]   7.00-8.00   sec  4.14 GBytes  35.6 Gbits/sec    0   1.40 MBytes
[  4]   8.00-9.00   sec  4.29 GBytes  36.8 Gbits/sec    0   1.64 MBytes
[  4]   9.00-10.00  sec  4.15 GBytes  35.7 Gbits/sec    0   1.68 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  42.0 GBytes  36.1 Gbits/sec    0             sender
[  4]   0.00-10.00  sec  42.0 GBytes  36.0 Gbits/sec                  receiver

iperf Done.
```
## Public iPerf3 Server
[https://iperf.fr/iperf-servers.php](https://iperf.fr/iperf-servers.php)
[https://github.com/R0GGER/public-iperf3-servers](https://github.com/R0GGER/public-iperf3-servers)
