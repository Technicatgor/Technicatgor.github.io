---
layout: post
title: Kubernetes IDE - OpenLens
date: 2023-05-08 19:00:00 +800
categories: [Kubernetes]
tags: [kubernetes,software]
---
  
## Openlens
The Openlens Kubernetes IDE is the open-source code project under MIT license, from which Lens desktop originates as a branch. It is a development tool for Kubernetes environments and containerized projects.
github repo [here](https://github.com/MuhammedKalkan/OpenLens)

Openlens startup ui
![openlen-1](/assets/img/openlen-1.png)

Add your own k3s cluster, import your ~/.kube/config from your control plan
![openlen-2](/assets/img/openlen-2.png)

You can add to Hotbar, and click the Workloads > Overviews to look at your cluster status
![openlen-3](/assets/img/openlen-3.png)

After deploy the kube-prometheus-stack, you will see the chart. You can check the metrics of cluster in Lens.
And you can check the issue.
![openlen-4](/assets/img/openlen-4.png)

And you can check the Nodes tab, it seems to be `kubectl get nodes` in cli.
![openlen-5](/assets/img/openlen-5.png)

You can edit the cluster, such as deploy, rs, svc...etc. It just like `kubectl edit deploy my-deploy`
![openlen-6](/assets/img/openlen-6.png)


