---
layout: post
title: Deploying VMs With Terraform In Proxmox
date: 2023-07-11 15:00:00 +800
categories: [DevOps,Terraform]
tags: [terraform,proxmox,homelab]
---
## Terraform
![banner](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*JZV49LQUvk73CYqwcsqMGA.png)

## Prerequisites
1. Promxox Server

## Resources

1. [Debian Cloud Images](https://cloud.debian.org/images/cloud/)
2. [Terraform Proxmox Provider](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs)


## Install libguestfs-tools
```
ssh root@proxmox-server

apt update && apt install libguestfs-tools -y
```

## Download debian cloud image
```
wget https://cloud.debian.org/images/cloud/bullseye/latest/debian-12-generic-arm64.qcow2
```

## Install qemu-guest-agent in the cloud image
```
virt-customize -a debian-12-generic-amd64.qcow2 --install qemu-guest-agent
```

## Create VM Template
```
qm create 9001 --name "debian-12-cloudinit-template" --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0
qm importdisk 9001 debian-12-generic-amd64.qcow2 local-lvm
qm set 9001 --scsi0 local-lvm:9001/vm-9001-disk-0.raw
qm set 9001 --boot c --bootdisk scsi0
qm set 9001 --ide2 local-lvm:cloudinit
qm set 9001 --serial0 socket --vga serial0
qm set 9001 --agent enabled=1

qm template 9001
```

## Create a role and user for terraform
1. Create Role
```
pveum role add terraform_role -privs "Datastore.AllocateSpace Datastore.Audit Pool.Allocate Sys.Audit Sys.Console Sys.Modify VM.Allocate VM.Audit VM.Clone VM.Config.CDROM VM.Config.Cloudinit VM.Config.CPU VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Migrate VM.Monitor VM.PowerMgmt"
```

2. Create User
```
pveum user add terraform_user@pve --password terraform
```

3. Map Role to User
```
pveum aclmod / -user terraform_user@pve -role terraform_role
```

## Install Terraform
Go back your Jarvis PC (controller)
```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=$(dpkg --print-architecture)] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt update
sudo apt install terraform -y
```

## Structure
The file structure will like this.
```sh
.
├── terraform.tfvars
├── provider.tf
├── main.tf
├── README.md
└── terraform.tfstate
```

## Terraform init
1. Create a `provider.tf`

```tf
variable "pm_api_url" {
  type = string
}

variable "pm_user" {
  type = string
}

variable "pm_password" {
  type      = string
  sensitive = true
}

terraform {
  required_providers {
    proxmox = {
      source  = "telmate/proxmox"
      version = "2.9.14"
    }
  }
}

provider "proxmox" {
  pm_api_url      = var.pm_api_url
  pm_user         = var.pm_user
  pm_password     = var.pm_password
  pm_tls_insecure = true
}
```

2. Create `terraform.tfvars` to contain your credentials.

```
pm_api_url              = "https://192.168.1.100:8006/api2/json"
pm_user                 = "terraform_user@pve"
pm_password             = "terraform"
cloudinit_template_name = "debian-12-cloudinit-template"
proxmox_node            = "pve"
ssh_key                 = "ssh-rsa your-ssh-key-here"
```

3. Run `terraform init`
```
heston@mac:~/terraform# terraform init

Initializing the backend...

Initializing provider plugins...
- Finding telmate/proxmox versions matching "2.9.14"...
- Installing telmate/proxmox v2.9.14...
- Installed telmate/proxmox v2.9.14 (self-signed, key ID A9EBBE091B3A834E)

Partner and community providers are signed by their developers.
If you'd like to know more about provider signing, you can read about it here:
https://www.terraform.io/docs/cli/plugins/signing.html

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

## Terraform Plan
1. Create main.tf
```tf
{% raw %}
variable "cloudinit_template_name" {
  type = string
}

variable "proxmox_node" {
  type = string
}

variable "ssh_key" {
  type      = string
  sensitive = true
}

resource "proxmox_vm_qemu" "k8s-1" {
  count       = 1
  name        = "k8s-0${count.index + 1}"
  vmid        = "100${count.index + 1}"
  target_node = var.proxmox_node
  clone       = var.cloudinit_template_name
  agent       = 1
  os_type     = "cloud-init"
  cores       = 2
  sockets     = 1
  cpu         = "host"
  memory      = 2048
  balloon     = 1024
  scsihw      = "virtio-scsi-pci"
  bootdisk    = "scsi0"

  disk {
    slot    = 0
    size    = "20G"
    type    = "scsi"
    storage = "local"
  }

  network {
    model  = "virtio"
    bridge = "vmbr1"
  }

  lifecycle {
    ignore_changes = [
      network,
    ]
  }

  ipconfig0  = "ip=10.0.1.20${count.index + 1}/24,gw=10.0.1.1"
  nameserver = "10.0.1.53"

  sshkeys = <<EOF
  ${var.ssh_key}
  EOF

{% endraw %}
}
```
{: file='main.tf'}

## Terraform Plan
1. Run `terraform plan`

you will see `Plan: 1 to add, 0 to change, 0 to destroy.`

## Terraform Apply 

```sh
terraform apply
```

you will see `Apply complete! Resources: 1 added, 0 changed, 0 destroyed.`

## Terraform Destroy

```sh
terraform destroy 
```

you will see `Destroy complete! Resources: 1 destroyed.`

