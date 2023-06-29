---
layout: post
title: Installing Private S3 Storage - Minio
date: 2023-04-24 15:00:00 +800
categories: [Docker, Minio]
tags: [minio,docker,storage]
---

![banner](https://geekflare.com/wp-content/uploads/2020/10/minio-1200x385.jpg)

## Minio
MinIO is a high-performance, S3 compatible object store. It is built for \
large scale AI/ML, data lake and database workloads. It runs on-prem and \
on any cloud (public or private) and from the data center to the edge. MinIO \
is software-defined and open source under GNU AGPL v3.

In this blog, I will install minio using docker-compose. \
Then I create a bucket for my synology hyperbackup. \
Just simulate that NAS using Amazon S3 backup environment. \

## Prerequisite
- docker-compose
- dns name record for minio api 
- SSL/TLS cert
- nginx reverse proxy (optional)

## Installation
Create a directory `mkdir -p docker/minio` \
go to directory `cd docker/minio` \
create a docker-compose.yml `vim docker-compose.yml`
```
version: '3.9'
services:
    minio:
        container_name: minio
        image: quay.io/minio/minio
        environment:
            - MINIO_ROOT_USER=admin
            - MINIO_ROOT_PASSWORD=P@ssw0rd
        volumes:
            - './data:/data'
        ports:
            - '9090:9090'
            - '9000:9000'
        command: 'server /data --console-address ":9090"'
```
```
docker-compose up -d
```
## Configuration
1. Browse `http://127.0.0.1:9090`
![loginpage](/assets/img/minio-01.png)

2. Create Bucket
![create-bucket](/assets/img/minio-02.png)

3. Create User name is "backup", select readwrite policy
![create-user](/assets/img/minio-03.png)

4. Create access key for "backup"
![create-accesskey](/assets/img/minio-04.png)
![save-accesskey](/assets/img/minio-05.png)

5. Change the bucket - "test" access policy to private
![policy](/assets/img/minio-06.png)

## Connect Bucket
1. Create a name record minio server url, for my eg:`minio.itcatgor.local`
2. Insert your TLS cert into nginx and point to server-ip:9000 
3. Go to Synology Hyper Backup, select S3 storage
![backup-01](/assets/img/minio-07.png)
4. Choose Custom Server URL, then enter all information.
![backup-01](/assets/img/minio-08.png)
5. Run Backup Successfully
![backup-01](/assets/img/minio-09.png)

## And more...
