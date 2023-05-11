---
layout: post
title: Using Ansible to provision remote servers
date: 2023-04-24 17:00:00 +800
categories: [DevOps,Ansible]
tags: [proxmox,ansible,kubernetes]
---

## Install Ansible
```
python3 -m pip install --user ansible
ansible --version
```

## Structure
```sh
.
├── ansible.cfg
├── inventory
│   └── my-cluster
│       ├── group_vars
│       │   └── all.yml
│       └── hosts.ini
├── roles
│   ├── download
│   │   └── tasks
│   │       └── main.yml
│   └── reset
│       └── tasks
│           └── main.yml
├── reset.yml
└── playbook.yml
```
I write a playbook for install nodejs from binaries on my k3s-cluster

## Files
ansible.cfg
```
[defaults]
inventory = ./inventory/my-cluster/hosts.ini # setup your remote hosts information
```
{: file="ansible.cfg"}

inventory/my-cluster/hosts.ini
```ini
[master]
10.0.50.51
[node]
10.0.50.61
10.0.50.62

[k3s-cluster:children]
master
node
```
{: file="inventory/my-cluster/hosts.ini" }

inventory/my-cluster/group_vars/all.yml
```yml
---
ansible_user: heston
ansible_private_key_file: /root/.ssh/id_rsa # ssh key have been setup packer/terraform before
NODEJS_VERSION: "20"
```
{: file="inventory/my-cluster/group_vars/all.yml" }

playbook.yml
```yml
---
- hosts: k3s-cluster
  remote_user: heston
  become: yes
  roles:
    - role: download # you can set a role to run the specific tasks
      become: true
```
{: file="playbook.yml" }

roles/download/tasks/main.yml
```
{% raw %}
---
- name: Install the gpg key for nodejs LTS
  apt_key:
    url: "https://deb.nodesource.com/gpgkey/nodesource.gpg.key"
    state: present

- name: Install the nodejs LTS repos
  apt_repository:
    repo: "deb https://deb.nodesource.com/node_{{ NODEJS_VERSION }}.x {{ ansible_distribution_release }} main"
    state: present
    update_cache: yes

- name: Install the nodejs
  apt:
    name: nodejs
    state: present
{% endraw %}
```
{: file="roles/download/tasks/main.yml" }

## Build
```
ansible-playbook playbook.yml -vv
```


