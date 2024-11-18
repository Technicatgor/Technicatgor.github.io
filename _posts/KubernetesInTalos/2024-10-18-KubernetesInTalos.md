---
layout: post
title: Deploy Kubernetes Cluster in Talos
date: 2024-10-18 09:00:00 +800
categories: [Kubernetes]
tags: [Kubernetes, homelab]
image:
  path: /assets/img/headers/talos-cover.png
  lqip: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA8gAAAH4CAIAAAAZ1VPRAALJIklEQVR4Aeyah5IbuxVE0aQ3h/eUv8L//1MOm/NumxaKp9TVZInO4e1d1QgDXNyMHgyG+vj59+Od
---

## Getting Started

Go to release page to download the latest OS `.iso`

[https://github.com/siderolabs/talos](https://github.com/siderolabs/talos)

Docs:\
[https://www.talos.dev/v1.8/introduction/getting-started/](https://www.talos.dev/v1.8/introduction/getting-started/)

---

## Prerequisites

### Install Talosctl

```bash
curl -sL https://talos.dev/install | sh
```

Ref: https://www.talos.dev/v1.8/talos-guides/install/talosctl/

## Installation

talos kubernetes infra example
![talos-infra](/assets/img/talos-k8s-infra.png)

I am using my mac to be a commander.
taloscp and taloswk installed in my pve with 2 cores and 4GB ram.

```bash
taloscp ip:
	10.0.50.101
taloswk ip:
	10.0.50.102
	10.0.50.103

export CONTROL_PLANE_IP=10.0.50.101
export WORKER_NODE_1_IP=10.0.50.102
export WORKER_NODE_2_IP=10.0.50.103
```

---

## CNI

The default installation used flannel and kube-proxy in k8s cluster. \
If your want to control the ingress, egress network policies, \
you can deploy Cilium

### Default way

```bash
talosctl gen config talos-promox-cluster https://CONTROL_PLANE_IP:6443 --output-dir _out
```

### Install Cilium way

Create a `patch.yml`, disable kube-proxy

```yaml
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true
```

Run this command to gen config.

```bash
talosctl gen config talos-promox-cluster https://CONTROL_PLANE_IP:6443 --config-patch @patch.yaml --output-dir _out
```

### Applying config

```bash
root@commander:~# ls _out
controlplane.yaml  talosconfig  worker.yaml
```

Create the control plane from the output of the `gen config` command.

```bash
talosctl apply-config --insecure --nodes $CONTROL_PLANE_IP --file _out/controlplane.yaml
```

Create the worker node from the output of the `gen config` command.

```bash
talosctl apply-config --insecure ---nodes $WORKER_NODE_1_IP -nodes $WORKER_NODE_2_IP --file _out/worker.yaml
```

Set the API Server via a Config for Kubeconfig.

```bash
export TALOSCONFIG="_out/talosconfig"

talosctl config endpoint $CONTROL_PLANE_IP

talosctl config node $CONTROL_PLANE_IP

```

Bootstrap Etcd.

```bash
talosctl bootstrap
```

Download the Kubeconfig from my cluster.

```bash
talosctl kubeconfig .
```

Test to confirm that the Kubeconfig is valid.

```bash
kubectl get nodes --kubeconfig=kubeconfig
```

You can see the Pod output to confirm that the cluster is running as expected.

```bash
kubectl get po -n kube-system --kubeconfig=kubeconfig
```

Output should look similar to the below:

#### Flannel & kube-proxy

```bash
NAME                                    READY   STATUS    RESTARTS       AGE
coredns-68d75fd545-xhwrg                1/1     Running   0              3m41s
coredns-68d75fd545-zx6jn                1/1     Running   0              3m41s
kube-apiserver-talos-e3j-66i            1/1     Running   0              3m17s
kube-controller-manager-talos-e3j-66i   1/1     Running   2 (4m5s ago)   2m31s
kube-flannel-65tr9                      1/1     Running   0              3m37s
kube-flannel-sxv6t                      1/1     Running   0              3m28s
kube-proxy-b8b2f                        1/1     Running   0              3m28s
kube-proxy-tt8jp                        1/1     Running   0              3m37s
kube-scheduler-talos-e3j-66i            1/1     Running   2 (4m5s ago)   2m23s
```

#### Cilium

[cilium-cli](https://technicatgor.github.io/posts/KubernetesInTalos/#instal-cilium-cli)

```bash
cilium install \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=true \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup \
    --set k8sServiceHost=localhost \
    --set k8sServicePort=7445
```

Check cilium status:

```bash
cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    disabled (using embedded mode)
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

Deployment        cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet         cilium             Desired: 2, Ready: 2/2, Available: 2/2
Containers:       cilium-operator    Running: 1
                  cilium             Running: 2
Cluster Pods:     3/3 managed by Cilium
Image versions    cilium             quay.io/cilium/cilium:v1.13.3@sha256:77176464a1e11ea7e89e984ac7db365e7af39851507e94f137dcf56c87746314: 2
                  cilium-operator    quay.io/cilium/operator-generic:v1.13.3@sha256:fa7003cbfdf8358cb71786afebc711b26e5e44a2ed99bd4944930bba915b8910: 1
```

## Enable hubble ui

```bash
cilium hubble enable --ui
```

Run `kubectl get pod -n kube-system --kubeconfig=kubeconfig`

```bash
NAME                               READY   STATUS    RESTARTS        AGE
cilium-696nl                       1/1     Running   0               3m41s
cilium-c5lc2                       1/1     Running   0               3m41s
cilium-operator-777fccb4f8-z4c8b   1/1     Running   0               3m17s
coredns-68d75fd545-75wk4           1/1     Running   0               2m31s
coredns-68d75fd545-xqsst           1/1     Running   0               3m37s
hubble-relay-7bd98d9b74-hc6kd      1/1     Running   0               3m28s
hubble-ui-7868f68687-5k9rk         2/2     Running   0               3m28s
kube-apiserver-talos-cp            1/1     Running   0               3m37s
kube-controller-manager-talos-cp   1/1     Running   4 (4m5s ago)    2m23s
kube-scheduler-talos-cp            1/1     Running   2 (4m5s ago)    2m23s
```

---

## Provisioning

- Copy you kubeconfig
  `cp kubeconfig ~/.kube/config`
- alias kubectl `~/.bashrc` or `~/.zshrc` & `source ~/.bashrc` or `source ~/.zshrc`
  `alias k="kubectl"`

### Deploying LoadBalancer solution - Metallb

- Docs: https://metallb.universe.tf/installation/

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml`
```

### Configuration

- Docs: https://metallb.universe.tf/configuration/

1.  Create a simple addresses pool in `metallb.yml`

```yaml
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.0.50.80-10.0.50.90
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
    - first-pool
```

2. Run this.
   `k create -f metallb.yml`

### Testing Metallb

```bash
k create deploy nginx --image nginx
k expose deploy nginx --port 80 --type LoadBalancer
```

```bash
root@commander:~# k get svc
NAME         TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1     <none>        443/TCP        11h
nginx        LoadBalancer   10.98.9.134   10.0.50.80    80:32599/TCP   6s
```

`curl 10.0.50.80`

```html
<!doctype html>
<html>
  <head>
    <title>Welcome to nginx!</title>
    <style>
      html {
        color-scheme: light dark;
      }
      body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
      }
    </style>
  </head>
  <body>
    <h1>Welcome to nginx!</h1>
    <p>
      If you see this page, the nginx web server is successfully installed and
      working. Further configuration is required.
    </p>

    <p>
      For online documentation and support please refer to
      <a href="http://nginx.org/">nginx.org</a>.<br />
      Commercial support is available at
      <a href="http://nginx.com/">nginx.com</a>.
    </p>

    <p><em>Thank you for using nginx.</em></p>
  </body>
</html>
```

### Delete Nginx

```
k delete deploy nginx
```

---

## Longhorn

_Longhorn is a lightweight, reliable and easy-to-use distributed block storage system for Kubernetes._

### Upgrade Image

Go to `https://factory.talos.dev/`

Choose the iscsi-tools and util-linux-tools, then copy the #Talos Linux Upgrade provided image.

`factory.talos.dev/installer/613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245:v1.8.3`

```bash
talosctl upgrade -n 10.0.50.101 -n 10.0.50.102 -n 10.0.50.103 --image  factory.talos.dev/installer/613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245:v1.8.3 --preserve
```

> **Caution:** If you do not include the `--preserve` option, Talos wipes `/var/lib/longhorn`, destroying all replicas stored on that node.

When finish, check the extensions:

`talosctl get extensions -n 10.0.50.101 -n 10.0.50.102 -n 10.0.50.103`

```bash
NODE          NAMESPACE   TYPE              ID   VERSION   NAME               VERSION
10.0.50.101   runtime     ExtensionStatus   0    1         iscsi-tools        v0.1.6
10.0.50.101   runtime     ExtensionStatus   1    1         util-linux-tools   2.40.2
10.0.50.101   runtime     ExtensionStatus   2    1         schematic          613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245
10.0.50.102   runtime     ExtensionStatus   0    1         iscsi-tools        v0.1.6
10.0.50.102   runtime     ExtensionStatus   1    1         util-linux-tools   2.40.2
10.0.50.102   runtime     ExtensionStatus   2    1         schematic          613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245
10.0.50.103   runtime     ExtensionStatus   0    1         iscsi-tools        v0.1.6
10.0.50.103   runtime     ExtensionStatus   1    1         util-linux-tools   2.40.2
10.0.50.103   runtime     ExtensionStatus   2    1         schematic          613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245
```

### Check extensions

```bash
talosctl get extensions -n 10.0.50.101 -n 10.0.50.102 -n 10.0.50.103
```

### Patch the worker nodes

Create `patch.yml`

```bash
machine:
  kubelet:
    extraMounts:
      - destination: /var/lib/longhorn
        type: bind
        source: /var/lib/longhorn
        options:
          - bind
          - rshared
          - rw
```

Patch the worker nodes that create a mount dir for longhorn with each worker node.

```bash
talosctl -n 10.0.50.102 -n 10.0.50.103 patch machineconfig -p @patch.yml
```

### Install with helm

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm show values longhorn/longhorn >> values.yml
```

Editing the `values.yml` first:

```yml
longhornUI:
  # -- Replica count for Longhorn UI.
  replicas: 2 #your worker nodes number
defaultSettings:
  backupTarget: "nfs://nas:/volume1/backup/longhorn"
```

Installation:

```bash
helm install longhorn longhorn/longhorn --namespace longhorn-system --values values.yml --create-namespace --version 1.7.2
```

### Pod Security

```bash
kubectl label namespace longhorn-system pod-security.kubernetes.io/enforce=privileged
```

### Ingress

Create `ingress.yml`

```yml
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: longhorn-ingressroute
  namespace: longhorn-system
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`FQDN`) # <-- Replace with your FQDN
      kind: Rule
      services:
        - name: longhorn-frontend
          port: 80
  # tls:
  #     secretName: longhorn-certificate-secret
```

## Dynamic storage provisioning - NFS

### Deploy nfs-client-provisioner

_I recommend for single node use case. If using multi-node, please use longhorn_

https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master/deploy

```
k apply -f rbac.yml

k apply -f class.yml

k apply -f deployment.yml # change your nfs server and path in the file.

k get po
NAME                                      READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-6c9bfdf668-vblpx   1/1     Running   0          22s
```

### Test the pv, pvc

```bash
k apply -f test-claim.yml
```

It will create a folder in your nfs server.\
![/assets/img/nfs-01.png](/assets/img/nfs-01.png)
_Note: If you change the archiveOnDelete to true that will rename the folder to archived after the pvc deleted. And if using a helm chart to install that will set as true by default._

```yaml
parameters:
  archiveOnDelete: "true"
```

![/assets/img/nfs-02.png](/assets/img/nfs-02.png)

---

## Deploy Traefik

With using the helm chart to install.

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm repo list
helm search repo traefik
```

Get the `traefik-values.yml`

```bash
helm show values traefik/traefik > traefik-values.yml
```

Enable persistence.

```yml
persistence:
	enabled: true
	storageClass: "nfs-client"
```

Deploy traefik.

```bash
helm install traefik traefik/traefik --values traefik-values.yml -n traefik --create-namespace
```

`helm list -n traefik`

```bash
NAME   	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART         	APP VERSION
traefik	traefik  	2       	2024-10-14 15:55:32.192212254 +0800 HKT	deployed	traefik-32.1.1	v3.1.6
```

`k get all -n traefik`

```bash
NAME                          READY   STATUS    RESTARTS   AGE
pod/traefik-678ffbf4d-7ztvs   1/1     Running   0          3m38s

NAME              TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
service/traefik   LoadBalancer   10.97.2.176   10.0.50.80    80:31252/TCP,443:30560/TCP   3m38s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/traefik   1/1     1            1           3m38s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/traefik-678ffbf4d   1         1         1       3m38s
```

Enable dashboard ingressRoute in values.yml

```yml
ingressRoute:
	dashboard:
		enabled: true
```

Access traefik-dashboard

```bash
k -n traefik port-forward traefik-678ffbf4d-7ztvs 9000:9000 --address 0.0.0.0
```

Go to browser input the commander IP with url:

```
http://commander-ip:9000/dashboard/
```

### Dashboard view

![/assets/img/traefik-dashboard.png](/assets/img/traefik-dashboard.png)

---

## IngressRoute

### Blue Nginx

```yaml
{% raw %}
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx-deploy-blue
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx-blue
  template:
    metadata:
      labels:
        run: nginx-blue
    spec:
      volumes:
        - name: webdata
          emptyDir: {}
      initContainers:
        - name: web-content
          image: busybox
          volumeMounts:
            - name: webdata
              mountPath: "/webdata"
          command:
            [
              "/bin/sh",
              "-c",
              'echo "<h1>I am <font color=green>BLUE</font></h1>" > /webdata/index.html'
            ]
      containers:
        - image: nginx
          name: nginx
          volumeMounts:
            - name: webdata
              mountPath: "/usr/share/nginx/html"
{% endraw %}
```

### Green Nginx

```yaml
{% raw %}
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx-deploy-green
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx-green
  template:
    metadata:
      labels:
        run: nginx-green
    spec:
      volumes:
        - name: webdata
          emptyDir: {}
      initContainers:
        - name: web-content
          image: busybox
          volumeMounts:
            - name: webdata
              mountPath: "/webdata"
          command:
            [
              "/bin/sh",
              "-c",
              'echo "<h1>I am <font color=green>Green</font></h1>" > /webdata/index.html'
            ]
      containers:
        - image: nginx
          name: nginx
          volumeMounts:
            - name: webdata
              mountPath: "/usr/share/nginx/html"
{% endraw %}
```

### Main Nginx

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx-deploy-main
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx-main
  template:
    metadata:
      labels:
        run: nginx-main
    spec:
      containers:
        - image: nginx
          name: nginx
```

### Check pods and services

```bash
pod/nginx-deploy-blue-cf86857bd-spf4q     1/1     Running   0          17h
pod/nginx-deploy-green-6b4694bbbc-d49q6   1/1     Running   0          17h
pod/nginx-deploy-main-65df7cc4b7-p9ntv    1/1     Running   0          17h
service/nginx-deploy-blue    ClusterIP   10.106.85.163   <none>        80/TCP    17h
service/nginx-deploy-green   ClusterIP   10.105.218.32   <none>        80/TCP    17h
service/nginx-deploy-main    ClusterIP   10.102.30.4     <none>        80/TCP    17h
```

### IngressRoute

`ingressRoute.yml`

```yml
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nginx
  namespace: default
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`nginx.example.com`)
      kind: Rule
      services:
        - name: nginx-deploy-main
          port: 80
    - match: Host(`blue.nginx.example.com`)
      kind: Rule
      services:
        - name: nginx-deploy-blue
          port: 80
    - match: Host(`green.nginx.example.com`)
      kind: Rule
      services:
        - name: nginx-deploy-green
          port: 80
```

```bash
k get ingressroute
NAME    AGE
nginx   2m
```

### Setup local DNS

```bash
k get svc -n traefik
NAME      TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
traefik   LoadBalancer   10.97.2.176   10.0.50.80    80:31252/TCP,443:30560/TCP   18h
```

```bash
echo "<traefik-lb-ip> nginx.example.com blue.nginx.example.com green.nginx.example.com" >> /etc/hosts
```

### Testing Route

`curl http://nginx.example.com`\
`curl http://blue.nginx.example.com`\
`curl http://green.nginx.example.com`

---

## Pebble ACME

Using Pebble as ACME server for testing environments.

### Helm install

```bash
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
helm repo update
helm show values jupyterhub/pebble >> pebble-values.yml
```

### Edit pebble-values.yml

```yml
# change value to "1"
  env:
	...
    - name: PEBBLE_VA_ALWAYS_VALID
      value: "1"
...

...
# coredns change to disable
coredns:
  enabled: false

```

Deploy pebble

```bash
helm install pebble jupyterhub/pebble --values pebble-values.yml -n traefik
```

### Edit traefik-values.yml

Edit volumes, additionalAguments, env.

```yml
volumes:
  - name: pebble
    mountPath: "/certs"
    type: configMap
---
globalArguments:
---
additionalArguments:
  - --certificatesresolvers.pebble.acme.tlschallenge=true
  - --certificatesresolvers.pebble.acme.email=test@hello.com
  - --certificatesresolvers.pebble.acme.storage=/data/acme.json
  - --certificatesresolvers.pebble.acme.caserver=https://pebble/dir
env:
  - name: LEGO_CA_CERTIFICATES
    value: "/certs/root-cert.pem"
```

update helm traefik

```bash
helm upgrade --install traefik traefik/traefik --values talos-k8s/traefik/traefik-values.yml -n traefik
```

Look pods in traefik namespace

```bash
k get po -n traefik
NAME                       READY   STATUS    RESTARTS   AGE
pebble-57b4c95f84-lbmcx    1/1     Running   0          173m
traefik-84f68bb4cf-8vl6p   1/1     Running   0          162m
```

### Testing TLS IngressRoute

_Note: please delete the preview ingressRoute for testing_
`tls-ingessRoute.yml`

```yml
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nginx
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`nginx.example.com`)
      kind: Rule
      services:
        - name: nginx-deploy-main
          port: 80
    - match: Host(`blue.nginx.example.com`)
      kind: Rule
      services:
        - name: nginx-deploy-blue
          port: 80
    - match: Host(`green.nginx.example.com`)
      kind: Rule
      services:
        - name: nginx-deploy-green
          port: 80
  tls:
    certResolver: pebble
```

Go to browser https://nginx.example.com, or

```
curl https://nginx.example.com --insecure
```

Certificate:\
![/assets/img/cert-01.png](/assets/img/cert-01.png)

Traefik route page:\
![/assets/img/cert-02.png](/assets/img/cert-02.png)

---

## Middlewares

### Basic Auth

[https://doc.traefik.io/traefik/middlewares/http/basicauth/](https://doc.traefik.io/traefik/middlewares/http/basicauth/)

### Redirect Scheme

[https://doc.traefik.io/traefik/middlewares/http/redirectscheme/](https://doc.traefik.io/traefik/middlewares/http/redirectscheme/)

### Testing in Nginx

Now trying these two middlewares

```yml
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: basic-auth
spec:
  basicAuth:
    secret: authsecret
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: http-redirect-scheme
spec:
  redirectScheme:
    scheme: https
    permanent: true
    port: "443"

---
# Example:
#   htpasswd -nb user 1234 | base64
#   dmVua2F0OiRhcHIxJE52L0lPTDZlJDRqdFlwckpjUk1aWU5aeG45M0xCNi8KCg==

apiVersion: v1
kind: Secret
metadata:
  name: authsecret

data:
  users: |
	dXNlcjokYXByMSR0aFIwTE12OSQ1YlRPQy9TT2JGZGJhMnhKUFJaSDgvCgo=
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nginx-http
  namespace: default
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`nginx.example.com`)
      kind: Rule
      middlewares:
        - name: basic-auth
        - name: http-redirect-scheme
      services:
        - name: nginx-deploy-main
          port: 80
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nginx-https
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`nginx.example.com`)
      kind: Rule
      middlewares:
        - name: basic-auth
      services:
        - name: nginx-deploy-main
          port: 80
  tls:
    certResolver: pebble
```

Check ingressroute and middlewares

```bash
k get ingressroute
NAME                      AGE
nginx-http                47m
nginx-https               47m

k get middlewares
NAME                   AGE
basic-auth             47m
http-redirect-scheme   47m

k describe ingressroute nginx-http
Name:         nginx-http
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  traefik.io/v1alpha1
Kind:         IngressRoute
Metadata:
  Creation Timestamp:  2024-10-16T06:40:00Z
  Generation:          2
  Resource Version:    408533
  UID:                 18b421ee-9fd6-4684-88ff-2527b87fbbae
Spec:
  Entry Points:
    web
  Routes:
    Kind:   Rule
    Match:  Host(`nginx.example.com`)
    Middlewares:
      Name:  basic-auth
      Name:  http-redirect-scheme
    Services:
      Name:  nginx-deploy-main
      Port:  80
Events:      <none>
```

---

## Weight Round Robin

`weight_round_robin.yml`

```yml
apiVersion: traefik.io/v1alpha1
kind: TraefikService
metadata:
  name: nginx-wrr
  namespace: default
spec:
  weighted:
    services:
      - name: nginx-deploy-main
        port: 80
        weight: 1
      - name: nginx-deploy-blue
        port: 80
        weight: 1
      - name: nginx-deploy-green
        port: 80
        weight: 1
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nginx-http
  namespace: default
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`nginx.example.com`)
      kind: Rule
      services:
        - name: nginx-wrr
          kind: TraefikService
```

---

## Reference Tools

### Install Kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

[https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

---

### Install Talosctl

```bash
curl -sL https://talos.dev/install | sh
```

[https://www.talos.dev/v1.8/talos-guides/install/talosctl/](https://www.talos.dev/v1.8/talos-guides/install/talosctl/)

---

### Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

[https://v3-1-0.helm.sh/docs/intro/install/](https://v3-1-0.helm.sh/docs/intro/install/)

## Install cilium cli

[https://github.com/cilium/cilium-cli](https://github.com/cilium/cilium-cli)

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
GOOS=$(go env GOOS)
GOARCH=$(go env GOARCH)
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-${GOOS}-${GOARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-${GOOS}-${GOARCH}.tar.gz.sha256sum
sudo tar -C /usr/local/bin -xzvf cilium-${GOOS}-${GOARCH}.tar.gz
rm cilium-${GOOS}-${GOARCH}.tar.gz{,.sha256sum}
```

## Reference Links

### Longhorn Talos

[https://longhorn.io/docs/1.7.2/advanced-resources/os-distro-specific/talos-linux-support/#talos-linux-upgrades](https://longhorn.io/docs/1.7.2/advanced-resources/os-distro-specific/talos-linux-support/#talos-linux-upgrades)

### Editing machine configuration

[https://www.talos.dev/v1.8/talos-guides/configuration/editing-machine-configuration/](https://www.talos.dev/v1.8/talos-guides/configuration/editing-machine-configuration/)

### Talos Linux Image Factory

[https://factory.talos.dev](https://factory.talos.dev)
