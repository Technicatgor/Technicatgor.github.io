---
layout: post
title: Monitor Your Proxmox with Grafana & Prometheus
date: 2023-10-25 10:00:00 +800
categories: [Hypervisors, Proxmox]
tags: [monitoring,prometheus,grafana,dashboard]
image:
  path: /assets/img/grafana_prometheus_proxmox.png
  lqip: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA8gAAAH4CAIAAAAZ1VPRAALJIklEQVR4Aeyah5IbuxVE0aQ3h/eUv8L//1MOm/NumxaKp9TVZInO4e1d1QgDXNyMHgyG+vj59+Od
---

## Prerequisites

1. Create a User on Proxmox and assign PVEAuditor role

  - Log into your pve, go to `DataCenter > Permissions > Groups`, `Create A group name as auditor`
  ![create_group](/assets/img/create_group.png)
  - Create a user name as prometheus@pve
  ![create_user](/assets/img/create_user.png)
  - Add group permissions
  ![add_group_permission](/assets/img/add_group_permission.png)

2. Docker
3. Prepare 1 node for deploy prompve/prometheus-pve-exporter

## Installation

### Docker node
Create a config file 
```
default:
    user: prometheus@pve
    password: password
    # Optional: set to false to skip SSL/TLS verification
    verify_ssl: true
```
or for token authenication
```
default:
    user: prometheus@pve
    token_name: "your-token-id"
    token_value: "..."
```
create a `docker-run.sh`
```
docker run --init --name prometheus-pve-exporter -d -p 127.0.0.1:9221:9221 -v ./pve.yml:/etc/pve.yml prompve/prometheus-pve-exporter
```
## Prometheus config
prometheus.yml
```yml
scrape_configs:
  - job_name: 'pve'
    static_configs:
      - targets:
        - 192.168.1.2  # Proxmox VE node.
        - 192.168.1.3  # Proxmox VE node.
    metrics_path: /pve
    params:
      module: [default]
      cluster:
        - 1
      node: 
        - 1
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9221  # PVE exporter.
```

## Grafana config.env & provisioning
`config.env`
```
GF_SECURITY_ADMIN_PASSWORD=password
GF_USERS_ALLOW_SIGN_UP=false
```

`./grafana/provisioning/datasources/datasource.yml`
```yml
# config file version
apiVersion: 1

# list of datasources that should be deleted from the database
deleteDatasources:
  - name: Prometheus
    orgId: 1

# list of datasources to insert/update depending
# whats available in the database
datasources:
  # <string, required> name of the datasource. Required
- name: Prometheus
  # <string, required> datasource type. Required
  type: prometheus
  # <string, required> access mode. direct or proxy. Required
  access: proxy
  # <int> org id. will default to orgId 1 if not specified
  orgId: 1
  # <string> url
  url: http://prometheus:9090
  # <string> database password, if used
  password:
  # <string> database user, if used
  user:
  # <string> database name, if used
  database:
  # <bool> enable/disable basic auth
  basicAuth: false
  # <string> basic auth username, if used
  basicAuthUser:
  # <string> basic auth password, if used
  basicAuthPassword:
  # <bool> enable/disable with credentials headers
  withCredentials:
  # <bool> mark as default datasource. Max one per org
  isDefault: true
  # <map> fields that will be converted to json and stored in json_data
  jsonData:
     graphiteVersion: "1.1"
     tlsAuth: false
     tlsAuthWithCACert: false
  # <string> json object of data that will be encrypted.
  secureJsonData:
    tlsCACert: "..."
    tlsClientCert: "..."
    tlsClientKey: "..."
  version: 1
  # <bool> allow users to edit datasources from the UI.
  editable: true
```

`grafana/provisioning/dashboards/dashboard.yml`

```
apiVersion: 1

providers:
- name: 'Prometheus'
  orgId: 1
  folder: ''
  type: file
  disableDeletion: false
  editable: true
  options:
    path: /etc/grafana/provisioning/dashboards
```
- provisioning your dashboard, your can import a json for your pve monitoring.
[pve_with_prom.json](/assets/file/pve_with_prom.json)

## Deploy Prometheus & Grafana with using Docker 

docker-compose.yml
```yml
version: '3.7'
services:

  prometheus:
    image: prom/prometheus:v2.40.0
    volumes:
      - ./prometheus/:/etc/prometheus/
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    ports:
      - 9090:9090
    networks:
      - monitor
    restart: always


  grafana:
    image: grafana/grafana
    depends_on:
      - prometheus
    ports:
      - 3000:3000
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning/:/etc/grafana/provisioning/

    env_file:
      - ./grafana/config.env
    networks:
      - monitor
    restart: always

volumes:
  prometheus_data:
    driver_opts:
      type: nfs
      o: addr=192.168.1.250,nfsvers=4 #your nas ip
      device: :/volume1/docker/prometheus-grafana/prometheus
  grafana_data:
    driver_opts:
      type: nfs
      o: addr=192.168.1.250,nfsvers=4 #your nas ip
      device: :/volume1/docker/prometheus-grafana/grafana
networks:
  monitor:
```

Run `docker-compose up -d`

## Dashboard
Import dashboard ID - 10347

## Additional
Monitor container metrics
### Cadvisor
Install cadvisor in docker-server
`docker-compose.yml`
```yml


```

