---
layout: post
title: Longhorn - Powerful distributed block storage for Kubernetes
date: 2023-05-11 12:00:00 +800
categories: [Kubernetes, Storage]
tags: [kubernetes,longhorn,helm]
---
## What is Longhorn? 
[Longhorn](https://longhorn.io/docs) is a lightweight, reliable and easy-to-use distributed block storage system for Kubernetes.

Longhorn is free, open source software. Originally developed by Rancher Labs, it is now being developed as a incubating project of the Cloud Native Computing Foundation.

## Prerequisites
- A container runtime compatible with Kubernetes (Docker v1.13+, containerd v1.3.7+, etc.)
- Kubernetes >= v1.21
- `open-iscsi` is installed
- RWX support requires that each node has a [NFSv4 client installed](https://longhorn.io/docs/1.4.1/deploy/install/#installing-nfsv4-client).
- filesystem supports:
  - ext4
  - XFS
- `bash, curl, findmnt, grep, awk, blkid, lsblk` must be installed.

## Installing Longhorn with Helm

1. Add the Longhorn Helm repository:
  ```
  helm repo add longhorn https://charts.longhorn.io
  ```

2. Fetch the latest charts from the repository:
  ```
  helm repo update
  ```

3. Install Longhorn in the longhorn-system namespace:
  ```
  helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --version 1.4.1
  ```

4. To confirm that the deployment succeeded, run:
  ```
  kubectl -n longhorn-system get pod
  ```
  The result should look like the following:
  ```
  NAME                                           READY   STATUS    RESTARTS   AGE
  longhorn-ui-b7c844b49-w25g5                    1/1     Running   0          2m41s
  longhorn-conversion-webhook-5dc58756b6-9d5w7   1/1     Running   0          2m41s
  longhorn-conversion-webhook-5dc58756b6-jp5fw   1/1     Running   0          2m41s
  longhorn-admission-webhook-8b7f74576-rbvft     1/1     Running   0          2m41s
  longhorn-admission-webhook-8b7f74576-pbxsv     1/1     Running   0          2m41s
  longhorn-manager-pzgsp                         1/1     Running   0          2m41s
  longhorn-driver-deployer-6bd59c9f76-lqczw      1/1     Running   0          2m41s
  longhorn-csi-plugin-mbwqz                      2/2     Running   0          100s
  csi-snapshotter-588457fcdf-22bqp               1/1     Running   0          100s
  csi-snapshotter-588457fcdf-2wd6g               1/1     Running   0          100s
  csi-provisioner-869bdc4b79-mzrwf               1/1     Running   0          101s
  csi-provisioner-869bdc4b79-klgfm               1/1     Running   0          101s
  csi-resizer-6d8cf5f99f-fd2ck                   1/1     Running   0          101s
  csi-provisioner-869bdc4b79-j46rx               1/1     Running   0          101s
  csi-snapshotter-588457fcdf-bvjdt               1/1     Running   0          100s
  csi-resizer-6d8cf5f99f-68cw7                   1/1     Running   0          101s
  csi-attacher-7bf4b7f996-df8v6                  1/1     Running   0          101s
  csi-attacher-7bf4b7f996-g9cwc                  1/1     Running   0          101s
  csi-attacher-7bf4b7f996-8l9sw                  1/1     Running   0          101s
  csi-resizer-6d8cf5f99f-smdjw                   1/1     Running   0          101s
  instance-manager-r-371b1b2e                    1/1     Running   0          114s
  instance-manager-e-7c5ac28d                    1/1     Running   0          114s
  engine-image-ei-df38d2e5-cv6nc                 1/1     Running   0          114s
  ```
  
## Web-UI

Change the Web-UI type to LoadBalancer:

```
kubectl -n longhorn-system get svc
```

```
For Longhorn v0.8.0, the output should look like this, and the CLUSTER-IP of the longhorn-frontend is used to access the Longhorn UI:

NAME                TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
longhorn-backend    ClusterIP      10.20.248.250   <none>           9500/TCP       58m
longhorn-frontend   ClusterIP      10.20.245.110   <none>           80/TCP         58m
```
![longhorn-ui](https://longhorn.io/img/screenshots/getting-started/longhorn-ui.png)

[Here](https://longhorn.io/docs/) for more setup
