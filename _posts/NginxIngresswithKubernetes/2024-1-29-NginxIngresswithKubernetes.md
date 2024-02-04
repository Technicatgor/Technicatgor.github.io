---
layout: post
title: Simple Nginx Ingress In Kubernetes
date: 2024-01-29 09:00:00 +800
categories: [Kubernetes]
tags: [network, homelab]
image:
  path: https://matthewpalmer.net/kubernetes-app-developer/articles/ingress.png
  lqip: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA8gAAAH4CAIAAAAZ1VPRAALJIklEQVR4Aeyah5IbuxVE0aQ3h/eUv8L//1MOm/NumxaKp9TVZInO4e1d1QgDXNyMHgyG+vj59+Od
---

## Prerequisite

Kubernetes Cluster with metallb (self-hosted)
ref: https://github.com/Technicatgor/k3s-ansible

## Installation

1. Create a Namespace for your app:

```bash
kubectl create namespace my-web-app
```

2. Deploy Nginx Ingress Controller:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```

1.  Deploy your web app:
    There are three parts in the deployment.yaml file.

```yaml
{% raw %}
---
apiVersion: v1
kind: Service
metadata:
  name: my-web-app
  namespace: my-web-app
  labels:
    app: my-web-app
spec:
  type: ClusterIP
  selector:
    app: my-web-app
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-web-app
  namespace: my-web-app
  labels:
    app: my-web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-web-app
  template:
    metadata:
      labels:
        app: my-web-app
    spec:
      containers:
        - name: my-web-app
          image: nginx
          ports:
            - containerPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-web-app-ingress
  namespace: my-web-app
spec:
  ingressClassName: nginx
  rules:
    - host: web-app.homelab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-web-app
                port:
                  number: 80

{% endraw %}
```

## Test ingress

check your app ingress IP
`kubectl get ingress -n my-web-app`

```bash
NAME                 CLASS   HOSTS                   ADDRESS       PORTS   AGE
my-web-app-ingress   nginx   web-app.homelab.local   192.168.30.221   80      117m
```

```bash
Add your domain into `/etc/host`
192.168.30.221 web-app.homelab.local
```

`curl web-app.homelab.local`

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
