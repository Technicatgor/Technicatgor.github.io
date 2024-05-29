---
layout: post
title: Azure with using Terraform
date: 2024-05-29 10:00 +800
categories: [Cloud]
tags: [azure, terraform]
image:
  path: /assets/img/headers/az-cloud-cover.jpeg
  lqip: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA8gAAAH4CAIAAAAZ1VPRAALJIklEQVR4Aeyah5IbuxVE0aQ3h/eUv8L//1MOm/NumxaKp9TVZInO4e1d1QgDXNyMHgyG+vj59+Od
---

## IPSec VPN Setup

### Prerequisite

- Terraform - [installation](https://developer.hashicorp.com/terraform/install)
- Azure-cli - [installation](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- Azure Subscription Account
- Already create subscription and Resource Group

### Setup

`provider.tf`

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>3.0"
    }
  }
}

provider "azurerm" {
  features {}
  skip_provider_registration = "true"
}
```

`variables.tf`

```hcl
variable location{
  type = string
}
```

`terraform.tfvars`

```hcl
location = "East Asia"
```

### Terraform init

`terraform init`

`main.tf`

```hcl
data "azurerm_resource_group" "existing" {
  name = "DEMO"
}

resource "azurerm_public_ip" "demo-ip" {
  name                = "demo-ip-${random_string.main.result}"
  resource_group_name = data.azurerm_resource_group.existing.name
  location            = data.azurerm_resource_group.existing.location
  sku                 = "Standard"
  allocation_method   = "Static"
}

resource "random_string" "main" {
  length  = 6
  upper   = false
  special = false
}

resource "azurerm_virtual_network" "demo-vnet" {
  name                = "demo-vnet-${random_string.main.result}"
  location            = data.azurerm_resource_group.existing.location
  resource_group_name = data.azurerm_resource_group.existing.name
  address_space       = ["10.20.0.0/16"]

  subnet {
    name           = "default"
    address_prefix = "10.20.0.0/24"
  }


}

resource "azurerm_subnet" "demo-gw-subnet" {
  name                 = "GatewaySubnet"
  resource_group_name  = data.azurerm_resource_group.existing.name
  virtual_network_name = azurerm_virtual_network.demo-vnet.name
  address_prefixes     = ["10.20.1.0/24"]
}

resource "azurerm_local_network_gateway" "demo-hq" {
  name                = "demo-hq-local-network-gw-${random_string.main.result}"
  resource_group_name = data.azurerm_resource_group.existing.name
  location            = data.azurerm_resource_group.existing.location
  gateway_address     = "<remote-wan-ip>"
  address_space       = ["<remote-local-subnet>"]
}

resource "azurerm_virtual_network_gateway" "demo-vnet-gw-tf" {
  name                = "demo-vnet-gw-tf"
  resource_group_name = data.azurerm_resource_group.existing.name
  location            = data.azurerm_resource_group.existing.location

  type     = "Vpn"
  vpn_type = "RouteBased"

  active_active = false
  enable_bgp    = false
  sku           = "VpnGw1"

  ip_configuration {
    name                 = "vnetGatewayConfig"
    public_ip_address_id = azurerm_public_ip.demo-ip.id
    subnet_id            = azurerm_subnet.demo-gw-subnet.id
  }
}

resource "azurerm_virtual_network_gateway_connection" "demo-hq-to-az" {
  name                = "demo-hq-to-az"
  resource_group_name = data.azurerm_resource_group.existing.name
  location            = data.azurerm_resource_group.existing.location


  type                       = "IPsec"
  virtual_network_gateway_id = azurerm_virtual_network_gateway.demo-vnet-gw-tf.id
  local_network_gateway_id   = azurerm_local_network_gateway.demo-hq.id

  shared_key          = "<preshared-key>"
  connection_protocol = "IKEv2"
  dpd_timeout_seconds = "45"
  ipsec_policy {
    ike_encryption = "AES256"
    ike_integrity  = "SHA256"
    dh_group       = "DHGroup2"

    ipsec_encryption = "AES256"
    ipsec_integrity  = "SHA256"
    pfs_group        = "None"
  }
}
```

### Run terraform plan and apply

```hcl
# validate .tf file
terraform validate

# check the plan
terraform plan

# apply your config without approve
terraform apply --auto-approve
```

Now you can setup the remote site VPN

## VM

```bash
# editing...
```
