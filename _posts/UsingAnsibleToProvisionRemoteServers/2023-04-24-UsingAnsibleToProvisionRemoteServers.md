---
layout: post
title: Using Ansible to provision remote servers
date: 2023-04-24 17:00:00 +800
categories: [Homelab,Proxmox]
tags: [proxmox,ansible]
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
NODE_VERSION: "v20.0.0"
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
- name: Download NodeJS
  ansible.builtin.get_url:
    url: "https://nodejs.org/dist/{{ NODE_VERSION }}/node-{{ NODE_VERSION }}-linux-x64.tar.xz"
    dest: /tmp
    
- name: Unarchive
  ansible.builtin.unarchive:
    src: /tmp/node-{{ NODE_VERSION }}-linux-x64.tar.xz
    dest: /tmp
    remote_src: true
      
- name: Copy to bin
  become: true
  ansible.builtin.copy:
    src: /tmp/node-{{ NODE_VERSION }}-linux-x64/{{ item }}
    dest: /usr
    remote_src: true
  with_items:
    - bin
    - include
    - share
    
- name: Check node version
  command: node -v

- name: Delete tmp file after installation
  ansible.builtin.file:
    state: absent
    path: "{{ item }}"
  with_items:
    - /tmp/node-{{ NODE_VERSION }}-linux-x64
    - /tmp/node-{{ NODE_VERSION }}-linux-x64.tar.xz
{% endraw %}
```
{: file="roles/download/tasks/main.yml" }

## Build
```
ansible-playbook playbook.yml -vv
```


