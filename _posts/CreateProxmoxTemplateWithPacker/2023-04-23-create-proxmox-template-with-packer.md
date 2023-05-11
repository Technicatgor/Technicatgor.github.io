---
layout: post
title: Create Proxmox Template With Packer
date: 2023-04-24 10:00 +800
categories: [Hypervisors,Proxmox]
tags: [proxmox,packer,homelab]
---
## Installing Packer
Add the HashiCorp GPG key.
```sh
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
```
Add the official HashiCorp Linux repository.
```sh
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
```
Update and install.
```sh
sudo apt-get update && sudo apt-get install packer
```
check packer is installed `packer --version`

## Structure
```sh
.
├── credentials.pkr.hcl
└── ubuntu-2204
    ├── files
    │   └── 99-pve.cfg
    ├── http
    │   ├── meta-data
    │   └── user-data
    └── ubuntu-2204-server.pkr.hcl
```
## Files
ubuntu-2204/ubuntu-2204-server.pkr.hcl
```hcl
{% raw %}
# Ubuntu Server 2204
# ---
# Packer Template to create an Ubuntu Server on Proxmox

# Variable Definitions
variable "proxmox_api_url" {
    type = string
}

variable "proxmox_api_token_id" {
    type = string
}

variable "proxmox_node" {
    type = string
}

variable "proxmox_api_token_secret" {
    type = string
    sensitive = true
}

# Resource Definiation for the VM Template
source "proxmox" "ubuntu-server-2204" {

    # Proxmox Connection Settings
    proxmox_url = "${var.proxmox_api_url}"
    username = "${var.proxmox_api_token_id}"
    token = "${var.proxmox_api_token_secret}"
    # (Optional) Skip TLS Verification
    # insecure_skip_tls_verify = true

    # VM General Settings
    node = "${var.proxmox_node}"
    vm_id = "901"
    vm_name = "ubuntu-server-2204"
    template_description = "Ubuntu Server template"

    # VM OS Settings
    # (Option 1) Local ISO File
    iso_file = "local:iso/ubuntu-22.04.2-live-server-amd64.iso"
    # - or -
    # (Option 2) Download ISO
    # iso_url = "https://releases.ubuntu.com/22.04/ubuntu-22.04-live-server-amd64.iso"
    # iso_checksum = "84aeaf7823c8c61baa0ae862d0a06b03409394800000b3235854a6b38eb4856f"
    iso_storage_pool = "local"
    unmount_iso = true

    # VM System Settings
    qemu_agent = true

    # VM Hard Disk Settings
    scsi_controller = "virtio-scsi-pci"

    disks {
        disk_size = "20G"
        format = "raw"
        storage_pool = "local-lvm"
        storage_pool_type = "lvm"
        type = "virtio"
    }

    # VM CPU Settings
    cores = "2"

    # VM Memory Settings
    memory = "2048"

    # VM Network Settings
    network_adapters {
        model = "virtio"
        bridge = "vmbr0"
        firewall = "false"
    }

    # VM Cloud-Init Settings
    cloud_init = true
    cloud_init_storage_pool = "local-lvm"

    # PACKER Boot Commands
    boot_command = [
        "<esc><wait>",
        "e<wait>",
        "<down><down><down><end>",
        "<bs><bs><bs><bs><wait>",
        "autoinstall ds=nocloud-net\\;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/ ---<wait>",
        "<f10><wait>"
    ]
    boot = "c"
    boot_wait = "5s"

    # PACKER Autoinstall Settings
    http_directory = "./ubuntu-2204/http"
    # (Optional) Bind IP Address and Port
    http_bind_address = "192.168.50.250"
    http_port_min = 8802
    http_port_max = 8802

    ssh_username = "packer"

    # (Option 1) Add your Password here
    ssh_password = "P@ssw0rd"
    # - or -
    # (Option 2) Add your Private SSH KEY file here
    ssh_private_key_file = "~/.ssh/id_rsa"

    # Raise the timeout, when installation takes longer
    ssh_timeout = "20m"
}

# Build Definition to create the VM Template
build {

    name = "ubuntu-server-2204"
    sources = ["source.proxmox.ubuntu-server-2204"]

    # Provisioning the VM Template for Cloud-Init Integration in Proxmox #1
    provisioner "shell" {
        inline = [
            "while [ ! -f /var/lib/cloud/instance/boot-finished ]; do echo 'Waiting for cloud-init...'; sleep 1; done",
            "sudo rm /etc/ssh/ssh_host_*",
            "sudo truncate -s 0 /etc/machine-id",
            "sudo apt -y autoremove --purge",
            "sudo apt -y clean",
            "sudo apt -y autoclean",
            "sudo cloud-init clean",
            "sudo rm -f /etc/cloud/cloud.cfg.d/subiquity-disable-cloudinit-networking.cfg",
            "sudo sync"
        ]
    }

    # Provisioning the VM Template for Cloud-Init Integration in Proxmox #2
    provisioner "file" {
        source = "./ubuntu-2204/files/99-pve.cfg"
        destination = "/tmp/99-pve.cfg"
    }

    # Provisioning the VM Template for Cloud-Init Integration in Proxmox #3
    provisioner "shell" {
        inline = [ "sudo cp /tmp/99-pve.cfg /etc/cloud/cloud.cfg.d/99-pve.cfg" ]
    }
}

{% endraw %}
```
{: file='ubuntu-2204/ubuntu-2204-server.pkr.hcl'}

./credentials.pkr.hcl

```hcl
proxmox_api_url="https://192.168.50.250:8006/api2/json"  # Your Proxmox IP Address
proxmox_api_token_id="root@pam!packer"  # API Token ID
proxmox_api_token_secret="aaaaaaaa-bbbb-cccc-dddd-eeeeeeeee"
proxmox_node="pve"
```
{: file='credentials.pkr.hcl'}

ubuntu-2004/http/user-data
```yml
#cloud-config
autoinstall:
  version: 1
  locale: en_US
  keyboard:
    layout: us
  ssh:
    install-server: true
    allow-pw: true
    disable_root: true
    ssh_quiet_keygen: true
    allow_public_ssh_keys: true
  packages:
    - qemu-guest-agent
    - sudo
  storage:
    layout:
      name: direct
    swap:
      size: 0
  user-data:
    package_upgrade: false
    timezone: Asia/Hong_Kong
    users:
      - name: packer
        groups: [adm, sudo]
        lock-passwd: false
        sudo: ALL=(ALL) NOPASSWD:ALL
        shell: /bin/bash
        # passwd: your-password
        # - or -
        ssh_authorized_keys:
          - ssh-rsa <your-ssh-key> root@pve

```
{: file='ubuntu-2204/http/user-data'}

ubuntu-2204/http/meta-data
```
# leave a blank file for meta-data
```
{: file='ubuntu-2204/files/99-pve.cfg'}

ubuntu-2204/files/99-pve.cfg
```
datasource_list: [ConfigDrive, NoCloud]
```
{: file='ubuntu-2204/files/99-pve.cfg'}

## Build
```
$ packer init ubuntu-2204/ubuntu-2204-server.pkr.hcl
$ packer build -var-file=credentials.pkr.hcl ubuntu-2004/ubuntu-2204-server.pkr.hcl
```
