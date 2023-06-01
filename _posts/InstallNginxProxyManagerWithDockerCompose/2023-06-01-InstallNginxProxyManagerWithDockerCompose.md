---
layout: post
title: Install Nginx Proxy Manager With Docker Compose
date: 2023-06-01 22:00:00 +800
categories: [Networking,ReverseProxy]
tags: [docker,nginx]
---

![banner](https://linuxhandbook.com/content/images/2020/09/deploy-multiple-services-with-nginx-reverse-proxy-container.png)
I am using nginx proxy manager for my work. It is easy to do maintenance with GUI. But I am using traefik for my homelab. And I love to use traefik that has more built-in features for managing microservices. Traefik is designed specifically for containerized environments. I will write a blog to talk about install traefik later.\
Nginx is the good choice also. Nginx is more focused on traditional web server functionality, such as load balancing, caching, and SSL termination.

## Prerequisite
- [DockerCompose](https://technicatgor.github.io/posts/DockerGuide/#installation)

## Installation
Create a docker-compose.yml from [official website](https://nginxproxymanager.com/setup/#running-the-app)
```
version: '3.8'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      # These ports are in format <host-port>:<container-port>
      - '80:80' # Public HTTP Port
      - '443:443' # Public HTTPS Port
      - '81:81' # Admin Web Port
      # Add any other Stream port you want to expose
      # - '21:21' # FTP
    environment:
      # Mysql/Maria connection parameters:
      DB_MYSQL_HOST: ${MYSQL_HOST}
      DB_MYSQL_PORT: ${MYSQL_PORT}
      DB_MYSQL_USER: ${MYSQL_USER}
      DB_MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      DB_MYSQL_NAME: ${MYSQL_NAME}
      # Uncomment this if IPv6 is not enabled on your host
      # DISABLE_IPV6: 'true'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    depends_on:
      - db

  db:
    image: 'jc21/mariadb-aria:latest'
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - ./mysql:/var/lib/mysql

```
Create a .env file contains the credential data
```
MYSQL_ROOT_PASSWORD=P@ssw0rd
MYSQL_DATABASE=npm
MYSQL_HOST=db
MYSQL_PORT=3306
MYSQL_USER=npm
MYSQL_PASSWORD=P@ssw0rd
MYSQL_NAME=npm
```

## Bring Up
```
docker-compose up -d
```

## Access Web UI
[http://localhost:81](http://localhost:81)
```
Email:    admin@example.com
Password: changeme
```
![login](https://linuxhint.com/wp-content/uploads/2021/04/How-to-use-Nginx-Proxy-Manager-1.png)

You will have to update the administrator details initially.
![update](https://linuxhint.com/wp-content/uploads/2021/04/How-to-use-Nginx-Proxy-Manager-2.png)

Main dashboard
![main](https://linuxhint.com/wp-content/uploads/2021/04/How-to-use-Nginx-Proxy-Manager-3.png)

## Add Proxy Host
Navigate to Hosts – Proxy Hosts and click on Add Proxy Host.\

Enter your domain name, scheme, forward hostname /IP and Port
You can also select Block common exploits for added security.
![newproxyhost](https://linuxhint.com/wp-content/uploads/2021/04/How-to-use-Nginx-Proxy-Manager-4.png)

Once you have exposed the service, try to access it using the specified hostname or IP and port. This service should be accessible. You can also manage the proxy in the proxy hosts list.
![proxyhost](https://linuxhint.com/wp-content/uploads/2021/04/How-to-use-Nginx-Proxy-Manager-5.png)

## Access List
In some instances, we may need to expose an application or service on the NPM proxy list to specific IP addresses. To configure this, you can use the NPM Access List.
![accesslist](https://linuxhint.com/wp-content/uploads/2021/04/How-to-use-Nginx-Proxy-Manager-6.png)

On the authorization tab, set the usernames and passwords you will use to log in to the service.
![authorization](https://linuxhint.com/wp-content/uploads/2021/04/How-to-use-Nginx-Proxy-Manager-7.png)

Navigate to the Access Tab and add the IP addresses you wish to allow connections from and deny all others.
![Access](https://linuxhint.com/wp-content/uploads/2021/04/How-to-use-Nginx-Proxy-Manager-8.png)

To attach the Access List to a specific web application, navigate to the Hosts – Proxy Host and select your host. Click on Edit and set the access list as defined above.
![editproxyhost](https://linuxhint.com/wp-content/uploads/2021/04/How-to-use-Nginx-Proxy-Manager-9.png)

## SSL Certs
NPM also allows you to provision SSL certificates on various domain names. Before adding a domain name to the SSL provision, ensure that the domain points to the Nginx proxy server.\
If you are using local domain, you should build a DNS server in your environment then point to NPM

Navigate to SSL certificates, and click on Add SSL certificate. Provide the domain names and the email address for Let’s Encrypt. Finally, Agree to the terms of service and save. You can also add a DNS challenge
![letsencrypt](https://linuxhint.com/wp-content/uploads/2021/04/How-to-use-Nginx-Proxy-Manager-10.png)

## Reference
[LinixHint](https://linuxhint.com/use-nginx-proxy-manager/)
