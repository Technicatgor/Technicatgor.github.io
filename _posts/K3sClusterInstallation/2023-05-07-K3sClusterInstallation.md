---
layout: post
title: K3s Cluster Installation 
date: 2023-05-07 19:00:00 +800
categories: [Homelab, Automation]
tags: [kubernetes,ansible,helm,kubeapps]
---


![k8s](/assets/img/k8s-banner.jpg)
We will use Ansible to set up a High Availability K3s cluster and [etcd](https://etcd.io/), [MetalLB](https://metallb.universe.tf/), [kube-vip](https://kube-vip.io/).

## Prerequisites
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#pip-install)

clone this repo -[k3s-ansible](https://github.com/Technicatgor/k3s-ansible/)
```sh
git clone https://github.com/Technicatgor/k3s-ansible/ && cd k3s-ansible
cp ansible.example.cfg ansible.cfg
```
edit the inventory/my-cluster/hosts.ini 
```
# kubernetes HA need odd number of master
[master]
10.0.50.101
10.0.50.103
10.0.50.105

[node]
10.0.50.104
10.0.50.106

# only required if proxmox_lxc_configure: true
# must contain all proxmox instances that have a master or worker node
# [proxmox]
# 192.168.30.43

[k3s_cluster:children]
master
node
```

inventory/my-cluster/group_vars/all.yml
```yml
---
k3s_version: v1.25.9+k3s1
# this is the user that has ssh access to these machines
ansible_user: ansibleuser
systemd_dir: /etc/systemd/system

# Set your timezone
system_timezone: "Asia/Hong_Kong"

# interface which will be used for flannel
flannel_iface: "eth0"

# apiserver_endpoint is virtual ip-address which will be configured on each master
apiserver_endpoint: "10.0.50.100"

# k3s_token is required  masters can talk together securely
# this token should be alpha numeric only
k3s_token: "some-SUPER-DEDEUPER-secret-password"

# The IP on which the node is reachable in the cluster.
# Here, a sensible default is provided, you can still override
# it for each of your hosts, though.
k3s_node_ip: '{{ ansible_facts[flannel_iface]["ipv4"]["address"] }}'

# Disable the taint manually by setting: k3s_master_taint = false
k3s_master_taint: "{{ true if groups['node'] | default([]) | length >= 1 else false }}"

# these arguments are recommended for servers as well as agents:
extra_args: >-
  --flannel-iface={{ flannel_iface }}
  --node-ip={{ k3s_node_ip }}

# change these to your liking, the only required are: --disable servicelb, --tls-san {{ apiserver_endpoint }}
extra_server_args: >-
  {{ extra_args }}
  {{ '--node-taint node-role.kubernetes.io/master=true:NoSchedule' if k3s_master_taint else '' }}
  --tls-san {{ apiserver_endpoint }}
  --disable servicelb
  --disable traefik
extra_agent_args: >-
  {{ extra_args }}

# image tag for kube-vip
kube_vip_tag_version: "v0.5.12"

# metallb type frr or native
metal_lb_type: "native"

# metallb mode layer2 or bgp
metal_lb_mode: "layer2"


# image tag for metal lb
metal_lb_frr_tag_version: "v7.5.1"
metal_lb_speaker_tag_version: "v0.13.9"
metal_lb_controller_tag_version: "v0.13.9"

# metallb ip range for load balancer
metal_lb_ip_range: "10.0.50.90-10.0.50.99"
proxmox_lxc_configure: false

```
{: file="inventory/my-cluster/group_vars/all.yml"}

## Run Playbook
```sh
ansible-playbook ./site.yml
```

## Copy the kube config to your 'jarvis' host
```sh
scp ansible@10.0.50.103:~/.kube/confg ~/.kube/config
```

## Test your Cluster
```sh
kubectl get nodes

NAME       STATUS   ROLES                       AGE   VERSION
master-1   Ready    control-plane,etcd,master   1m   v1.25.9+k3s1
master-2   Ready    control-plane,etcd,master   1m   v1.25.9+k3s1
master-3   Ready    control-plane,etcd,master   1m   v1.25.9+k3s1
worker-1   Ready    <none>                      1m   v1.25.9+k3s1
worker-2   Ready    <none>                      1m   v1.25.9+k3s1
```
ping apiserver endpoint
```
ping 10.0.50.100
```
## Install Helm
install [helm](https://helm.sh/docs/intro/install/)
```sh
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## Deploy kubeapps
install [kubeapps](https://kubeapps.dev/docs/latest/tutorials/getting-started/#step-2-create-a-demo-credential-with-which-to-access-kubeapps-and-kubernetes)
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
kubectl create namespace kubeapps
helm install kubeapps --namespace kubeapps bitnami/kubeapps
```

Create a demo credential with which to access Kubeapps and Kubernetes
```sh
kubectl create --namespace default serviceaccount kubeapps-operator
kubectl create clusterrolebinding kubeapps-operator --clusterrole=cluster-admin --serviceaccount=default:kubeapps-operator
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: kubeapps-operator-token
  namespace: default
  annotations:
    kubernetes.io/service-account.name: kubeapps-operator
type: kubernetes.io/service-account-token
EOF
```
get token
```sh
kubectl get --namespace default secret kubeapps-operator-token -o go-template='{{.data.token | base64decode}}'
```

`kubectl get svc -n kubeapps`
```sh
NAME                             TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubeapps                         LoadBalancer   10.43.195.213   10.0.50.90    80:32012/TCP   23h
kubeapps-internal-dashboard      ClusterIP      10.43.82.202    <none>        8080/TCP       23h
kubeapps-internal-kubeappsapis   ClusterIP      10.43.173.13    <none>        8080/TCP       23h
kubeapps-postgresql              ClusterIP      10.43.11.237    <none>        5432/TCP       23h
kubeapps-postgresql-hl           ClusterIP      None            <none>        5432/TCP       23h
```
go to web console - `http://10.0.50.90` and access with token

## Reset your Cluster
```sh
ansible-playbook ./reset.yml
```

## What's Next?
I will show you how to deploy [longhorn](https://longhorn.io/) in next chapter.
