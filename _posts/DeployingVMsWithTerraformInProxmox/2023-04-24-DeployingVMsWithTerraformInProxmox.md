---
layout: post
title: Deploying VMs With Terraform In Proxmox
date: 2023-04-24 15:00:00 +800
categories: [DevOps,Terraform]
tags: [terraform,proxmox.homelab]
---

## Install Terraform
```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=$(dpkg --print-architecture)] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt update
sudo apt install terraform -y
```
## Create a API Token Key on Proxmox

## Structure
```sh
.
├── vars.tfvars
├── main.tf
├── README.md
└── terraform.tfstate
```

## Terraform init and provider install

create a `main.tf` 
```tf
terraform {
    required_providers {
      proxmox = {
        source  = "telmate/proxmox",
        version = "2.9.11" # 2.9.14 still has a bug, so using 2.9.11
      }
    }
  }
```
then run `terraform init`
```
root@pve:~/terraform# terraform init

Initializing the backend...

Initializing provider plugins...
- Finding telmate/proxmox versions matching "2.9.11"...
- Installing telmate/proxmox v2.9.11...
- Installed telmate/proxmox v2.9.11 (self-signed, key ID A9EBBE091B3A834E)

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

```tf
variable "ubuntu_user" {
    type = string
}

variable "ubuntu_password" {
    type = string
    sensitive = true
}

variable "pm_api_url" {
    type = string

}
variable "pm_api_token_id" {
    type = string

}
variable "pm_api_token_secret" {
    type = string
    sensitive = true
}

terraform {
    required_providers {
      proxmox = {
        source  = "telmate/proxmox",
        version = "2.9.11"
      }
    }
  }

provider "proxmox" {
    pm_api_url          = var.pm_api_url
    pm_api_token_id     = var.pm_api_token_id
    pm_api_token_secret = var.pm_api_token_secret
    pm_tls_insecure     = true
  }

resource "proxmox_vm_qemu" "ubuntu-server" {
  count       = 1
  vmid        = "50${count.index + 1}"
  name        = "server-${count.index + 1}"
  target_node = "pve"
  desc        = "ubuntu-server"
  os_type     = "cloud-init"
  ciuser      = var.ubuntu_user
  cipassword  = var.ubuntu_password
  clone       = "ubuntu-server-template"
  onboot      = true
  agent       = 1
  cores       = 2
  sockets     = 1
  cpu         = "kvm64"
  memory      = 4096
  balloon     = 1024
  network {
    bridge = "vmbr0"
    model  = "virtio"
  }
  ipconfig0  = "ip=192.168.50.5${count.index + 1}/24,gw=192.168.50.254"
  nameserver = "1.1.1.1"
}
```
{: file='main.tf'}

demo.tfvars
```
ubuntu_user="packer"
ubuntu_password="P@ssw0rd"
pm_api_url="https://192.168.50.250:8006/api2/json"
pm_api_token_id="root@pam!terraform"
pm_api_token_secret="your-api-token-here"
```
{: file='demo.tfvars'}

Then `terraform plan -var-file=demo.tfvars`
you will see `Plan: 1 to add, 0 to change, 0 to destroy.`

## Terraform Apply 
```sh
terraform apply -var-file=demo.tfvars --auto-approve
```
you will see `Apply complete! Resources: 1 added, 0 changed, 0 destroyed.`

## Terraform Destroy

```sh
terraform destroy -var-file=demo.tfvars --auto-approve
```
you will see `Destroy complete! Resources: 1 destroyed.`

