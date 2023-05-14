---
layout: post
title: Monitor Your Kubernetes with Grafana & Prometheus
date: 2023-05-14 19:00:00 +800
categories: [Kubernetes,Monitoring]
tags: [kubernetes,monitoring,prometheus,grafana,dashboard]
---

Prometheus is a monitoring solution for storing time series data like metrics. Grafana allows to visualize the data stored in Prometheus (and other sources). This sample demonstrates how to capture NServiceBus metrics, storing these in Prometheus and visualizing these metrics using Grafana.

## Prerequisites

Install helm
```sh
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

helm will be using to install [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)

## Helm
Add helm repo
```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

Create a secret file for login info
```sh
echo -n 'admin' > ./admin-user
echo -n 'P@ssw0rd' > ./admin-password
```

Create a monitoring namespace
```sh
kubectl create ns monitoring
```

Create a secret for grafana-admin-credentials
```sh
kubectl create secret generic grafana-admin-credentials --from-file=./admin-user --from-file=admin-password -n monitoring
```

Remove the secret files
```sh
rm admin-user && rm admin-password
```
## Values file
Create a values.yml
```yml
{% raw %}
fullnameOverride: prometheus

defaultRules:
  create: true
  rules:
    alertmanager: true
    etcd: true
    configReloaders: true
    general: true
    k8s: true
    kubeApiserverAvailability: true
    kubeApiserverBurnrate: true
    kubeApiserverHistogram: true
    kubeApiserverSlos: true
    kubelet: true
    kubeProxy: true
    kubePrometheusGeneral: true
    kubePrometheusNodeRecording: true
    kubernetesApps: true
    kubernetesResources: true
    kubernetesStorage: true
    kubernetesSystem: true
    kubeScheduler: true
    kubeStateMetrics: true
    network: true
    node: true
    nodeExporterAlerting: true
    nodeExporterRecording: true
    prometheus: true
    prometheusOperator: true

alertmanager:
  fullnameOverride: alertmanager
  enabled: true
  ingress:
    enabled: false

grafana:
  enabled: true
  fullnameOverride: grafana
  forceDeployDatasources: false
  forceDeployDashboards: false
  defaultDashboardsEnabled: true
  defaultDashboardsTimezone: Asia/Hong_Kong
  service:
    type: LoadBalancer
  serviceMonitor:
    enabled: true
  admin:
    existingSecret: grafana-admin-credentials
    userKey: admin-user
    passwordKey: admin-password

kubeApiServer:
  enabled: true

kubelet:
  enabled: true
  serviceMonitor:
    metricRelabelings:
      - action: replace
        sourceLabels:
          - node
        targetLabel: instance

kubeControllerManager:
  enabled: true
  endpoints: # ips of servers
    - 10.0.50.101
    - 10.0.50.102
    - 10.0.50.103
    - 10.0.50.104
    - 10.0.50.105

coreDns:
  enabled: true

kubeDns:
  enabled: false

kubeEtcd:
  enabled: true
  endpoints: # ips of servers
    - 10.0.50.101
    - 10.0.50.102
    - 10.0.50.103
    - 10.0.50.104
    - 10.0.50.105
  service:
    enabled: true
    port: 2381
    targetPort: 2381

kubeScheduler:
  enabled: true
  endpoints: # ips of servers
    - 10.0.50.101
    - 10.0.50.102
    - 10.0.50.103
    - 10.0.50.104
    - 10.0.50.105

kubeProxy:
  enabled: true
  endpoints: # ips of servers
    - 10.0.50.101
    - 10.0.50.102
    - 10.0.50.103
    - 10.0.50.104
    - 10.0.50.105

kubeStateMetrics:
  enabled: true

kube-state-metrics:
  fullnameOverride: kube-state-metrics
  selfMonitor:
    enabled: true
  prometheus:
    monitor:
      enabled: true
      relabelings:
        - action: replace
          regex: (.*)
          replacement: $1
          sourceLabels:
            - __meta_kubernetes_pod_node_name
          targetLabel: kubernetes_node

nodeExporter:
  enabled: true
  serviceMonitor:
    relabelings:
      - action: replace
        regex: (.*)
        replacement: $1
        sourceLabels:
          - __meta_kubernetes_pod_node_name
        targetLabel: kubernetes_node

prometheus-node-exporter:
  fullnameOverride: node-exporter
  podLabels:
    jobLabel: node-exporter
  extraArgs:
    - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/.+)($|/)
    - --collector.filesystem.fs-types-exclude=^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs)$
  service:
    portName: http-metrics
  prometheus:
    monitor:
      enabled: true
      relabelings:
        - action: replace
          regex: (.*)
          replacement: $1
          sourceLabels:
            - __meta_kubernetes_pod_node_name
          targetLabel: kubernetes_node
  resources:
    requests:
      memory: 512Mi
      cpu: 250m
    limits:
      memory: 2048Mi

prometheusOperator:
  enabled: true
  prometheusConfigReloader:
    resources:
      requests:
        cpu: 200m
        memory: 50Mi
      limits:
        memory: 100Mi

prometheus:
  enabled: true
  prometheusSpec:
    replicas: 1
    replicaExternalLabelName: "replica"
    ruleSelectorNilUsesHelmValues: false
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    probeSelectorNilUsesHelmValues: false
    retention: 1d
    enableAdminAPI: true
    walCompression: true
    additionalScrapeConfigs: |

      - job_name: 'my-server'
        metrics_path: /metrics
        static_configs:
          - targets: ['10.0.50.100:9100']


    storageSpec:
     volumeClaimTemplate:
       metadata:
         name: prometheus-prometheus-prometheus-db
       spec:
         storageClassName: longhorn
         accessModes: ["ReadWriteOnce"]
         resources:
           requests:
             storage: 5Gi

thanosRuler:
  enabled: false
{% endraw %}
```
## Install
Use helm install kube-prometheus-stack
```sh
helm install -n monitoring prometheus prometheus-community/kube-prometheus-stack -f values.yaml
```

## Grafana
Access Grafana web-ui
```sh
k get svc -n monitoring
```
```sh
prometheus-prometheus     ClusterIP      10.43.13.18     <none>        9090/TCP                     6d5h
node-exporter             ClusterIP      10.43.187.199   <none>        9100/TCP                     6d5h
kube-state-metrics        ClusterIP      10.43.66.196    <none>        8080/TCP,8081/TCP            6d5h
prometheus-operator       ClusterIP      10.43.194.4     <none>        443/TCP                      6d5h
grafana                   LoadBalancer   10.43.189.189   10.0.50.201   80:31748/TCP                 6d5h
prometheus-alertmanager   ClusterIP      10.43.251.198   <none>        9093/TCP                     6d5h
alertmanager-operated     ClusterIP      None            <none>        9093/TCP,9094/TCP,9094/UDP   6d5h
prometheus-operated       ClusterIP      None            <none>        9090/TCP                     6d5h
```
![grafana](https://assets.digitalocean.com/articles/67242/grafana_login.png)

## Dashboard
Import your dashboard\
Go to [Link](https://grafana.com/grafana/dashboards/) to search\
I find a pretty good dashboard for k8s cluster - [Link](https://grafana.com/grafana/dashboards/13105-1-k8s-for-prometheus-dashboard-20211010/)

![grafana](/assets/img/grafana-1.png)

## Update
You can update and change your values.yml
```sh
helm upgrade -n monitoring prometheus prometheus-community/kube-prometheus-stack -f values.yaml
```


