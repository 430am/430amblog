---
title: "Terraform Key Vaults and Azure, Oh My!"
date: 2026-05-07
weight: 11
draft: false
description: "Demonstrates how to use the latest Terraform capabilities to deploy and manage SSH keypairs in Azure for VM authentication using Azure Key Vault"
slug: "terraform-azurekeyvault"
tags: ["terraform", "Azure", "Security"]
categories: ["Infrastructure as Code", "Cloud", "Security"]
---

There are two "good" ways to manage authentication to Linux VMs in Azure:

1. SSH keys (which can be managed in a few ways, none of them super awesome.)
2. Entra authentication through the Entra SSH extension

I'll discuss the second option in a later post, as it's come a *LONG* way, but in this post I'm going to discuss SSH authentication. Most of my customers leverage SSH as their authentication of choice, as it is the form they are most used to in the high performance computing world. In HPC, you have a Linux cluster that you will use SSH to authenticate to to run your jobs. Generally, this means you're managing SSH keys in a variety of ways, none of them cloud-native.

I've been working with a number of customers who are building clusters in Azure for various scientific projects, and they often are building ephemeral clusters, meaning they want to spin it up and destroy it all when they have their results. Often, this means they spin up and down multiple clusters for different projects, and wish to do it in a repeatable manner.

I'm working on that, and through that process I found myself struggling with managing SSH keys while testing different parts of my builds. Noodling around in terraform, I figured out how to manage SSH keys in a repeatable fashion, using Azure Key Vault. This is what I'm going to show in this post.

## Ephemerals in Terraform

Terraform version 1.11 introduced `ephemeral` values in resources, which allows you to create and work with data that you would otherwise not wish to be saved to state or stored in plan files. Additionally, there are now `ephemeral` resource blocks and ephemeral write-only arguments on resources.

When creating an SSH keypair, using the ephemeral resource type, we can ensure that none of the data is stored in the state file or in any plan that we may create using `terraform plan`.

Additionally, as we create our public and private keys, we can leverage the new write-only arguments to save the secrets to the Azure Key Vault without saving it out to the plan or the state file. Additionally, if we want to make changes to these, we can leverage write-only versioning to ensure that we update the secret values if we need to make changes to our infrastructure!

## Terraform Structure

I've published a repo for this solution on my personal GitHub {{< github repo="430am/azurekeyvaultsshkeys" showThumbnail=true >}}

The repo includes the following files:

```shell

.
└── terraform
    └── main.tf
    ├── keyvault.tf
    ├── providers.tf
    ├── ssh.tf
    └── variables.tf
```

## providers.tf

In `providers.tf` we define the Terraform providers and versions we'll need for this solution:

```hcl
terraform {
    required_version = ">=1.14"

    required_providers = {
        azurerm = {
            source = "hashicorp/azurerm"
            version = ">= 4.68.0"
        }
        random = {
            source = "hashicorp/random"
            version = ">= 3.8.0"
        }
        tls = {
            source = "hashicorp/tls"
            version = ">= 4.2.1"
        }
    }
}

provider "azurerm" {
    features {
        key_vault {
            purge_soft_delete_on_destroy = true
        }
        resource_group {
            prevent_deletion_if_contains_resources = false
        }
    }
}
```

From top to bottom, we're leveraging the latest Terraform version (1.14 at the time of writing) so we can take advantage of the `ephemeral` and write-only features. We pull in `azurerm` to talk to Azure, `random` to generate a friendly name we can reuse across resources, and `tls` for the SSH keypair generation itself.

In the `azurerm` provider block, the `key_vault` and `resource_group` features are tuned for an *ephemeral* lab environment. Because we're going to be standing this up and tearing it down repeatedly, `purge_soft_delete_on_destroy = true` makes sure the Key Vault is actually gone when we run `terraform destroy` (rather than sitting in a soft-deleted state and blocking us from re-using the name). Similarly, `prevent_deletion_if_contains_resources = false` lets the resource group be removed even if it still has child resources hanging around.

> [!WARNING]
> **Be careful with these settings in production.** 
> Both options are great for short-lived lab and HPC clusters, but they're *not* what you want guarding the Key Vault that holds your production secrets.

## variables.tf

The variables file is intentionally tiny. The only thing we want to be able to tweak per-deployment is the Azure region:

```hcl
variable "region" {
  description = "The Azure region to deploy resources in"
  type        = string
  default     = "centralus"
}
```

Everything else is derived from this and from the random pet name we generate later, which keeps the inputs to the module dead simple.

## main.tf

`main.tf` is where we set up the supporting scaffolding — the things every other file is going to lean on:

```hcl
data "azurerm_client_config" "current" {}

resource "random_pet" "naming" {
    separator = ""
    length    = 2
}

resource "azurerm_resource_group" "rg" {
    name     = "${random_pet.naming.id}-rg"
    location = var.region
}
```

A few things going on here:

- `data "azurerm_client_config" "current"` pulls the current tenant and object IDs from whatever identity Terraform is running as. We need both later — the tenant ID to create the Key Vault, and the object ID to grant ourselves the RBAC role that lets us write secrets into it.
- `random_pet` gives us a short, human-friendly string (something like `selectfrog`) that we'll reuse for naming the resource group, the Key Vault, and the secrets. Using `separator = ""` keeps the result valid for resources like Key Vault that don't allow dashes everywhere.
- The resource group itself is named after that pet, in the region the user passed in via `var.region`.

## ssh.tf

This is where the new ephemeral resources really shine:

```hcl
ephemeral "tls_private_key" "ssh-priv-key" {
    algorithm = "ED25519"
}

ephemeral "tls_public_key" "ssh-pub-key" {
    private_key_openssh = ephemeral.tls_private_key.ssh-priv-key.private_key_openssh
}
```

The whole reason this works without leaking key material is the `ephemeral` block. Both the private key and the derived public key are generated _at apply time_ and then thrown away — they are **never** written to the state file and they don't show up in any plan output. We're using ED25519 here because it's modern, fast, and produces a much smaller key than RSA, which is nice for a key you're going to be passing around to a bunch of cluster nodes.

The second block takes the ephemeral private key and asks the `tls` provider to derive the matching OpenSSH-formatted public key from it. We'll hand that public key off to our VMs (or output it for the user) so they know what to trust.

## keyvault.tf

This is the heart of the solution. We create the vault, give ourselves permission to write to it, and then use the new write-only secret arguments to land the keypair inside the vault without ever putting it in state.

```hcl
resource "azurerm_key_vault" "kv" {
    location                   = azurerm_resource_group.rg.location
    name                       = "kv${random_pet.naming.id}"
    resource_group_name        = azurerm_resource_group.rg.name
    sku_name                   = "standard"
    tenant_id                  = data.azurerm_client_config.current.tenant_id
    soft_delete_retention_days = 7
    purge_protection_enabled   = false
    rbac_authorization_enabled = true
}

data "azurerm_key_vault" "kv_store" {
    name                = azurerm_key_vault.kv.name
    resource_group_name = azurerm_key_vault.kv.resource_group_name
}

resource "azurerm_role_assignment" "kv_admin" {
    principal_id         = data.azurerm_client_config.current.object_id
    scope                = azurerm_key_vault.kv.id
    role_definition_name = "Key Vault Administrator"

    depends_on = [ azurerm_key_vault.kv ]
}

resource "azurerm_key_vault_secret" "private_key" {
    key_vault_id     = azurerm_key_vault.kv.id
    name             = "ssh-${random_pet.naming.id}-private_key"
    value_wo         = ephemeral.tls_private_key.ssh-priv-key.private_key_openssh
    value_wo_version = "1"

    depends_on = [ azurerm_role_assignment.kv_admin ]
}

resource "azurerm_key_vault_secret" "public_key" {
    key_vault_id     = azurerm_key_vault.kv.id
    name             = "ssh-${random_pet.naming.id}-public_key"
    value_wo         = ephemeral.tls_private_key.ssh-priv-key.public_key_openssh
    value_wo_version = "1"

    depends_on = [ azurerm_role_assignment.kv_admin ]
}
```

Walking through this top-to-bottom:

- The Key Vault itself is named `kv<random_pet>` — Key Vault names have to be globally unique, and the random pet keeps us out of collision trouble. We're enabling `rbac_authorization_enabled` so we use Azure RBAC for data-plane permissions instead of the legacy access policies. That's the modern recommendation, and it makes the role assignment below all we need.
- The `data "azurerm_key_vault" "kv_store"` block re-reads the vault we just created. This is occasionally handy when downstream resources want a data source rather than a resource reference.
- The `azurerm_role_assignment` grants the principal that's running Terraform the **Key Vault Administrator** role, scoped to this vault. Without this, the very next step (writing secrets) would fail with an authorization error, which is why we have an explicit `depends_on` on the vault — we want the assignment to be in place before we try to use it.
- Finally, the two `azurerm_key_vault_secret` resources are where the magic happens. Notice that we're using `value_wo` instead of `value`. That's the **write-only** argument introduced alongside ephemerals: the value is sent to Azure, but it never gets stored in Terraform state. The `value_wo_version` is what tells Terraform whether the value needs to change — bump it from `"1"` to `"2"` (and re-supply a new ephemeral value) and Terraform will rotate the secret on the next apply. Leave it alone and Terraform won't even look at the value, which is exactly what we want for something it can't see anyway.

The end result: the secrets live safely in Key Vault, the state file has no idea what they actually are, and we can rotate them by versioning rather than by storing material we don't want to store.

## outputs.tf

The only thing we expose back to the operator is the public key, since that's the piece you'd typically need to hand to a VM, an `authorized_keys` file, or another module:

```hcl
output "public_key_openssh" {
  value       = ephemeral.tls_public_key.ssh-pub-key.public_key_openssh
  description = "The public key in OpenSSH format"
}
```

Because the source is an ephemeral resource, this output is itself ephemeral — it shows up in the apply output for that one run, but it isn't persisted anywhere Terraform manages. The private key is never output; the only place to retrieve it is from the Key Vault, where access is governed by Azure RBAC.

## Putting it all together

To deploy:

```shell
cd terraform
terraform init
terraform apply
```

After the apply finishes, you can pull the private key out of the vault when you actually need to authenticate to a VM:

```shell
az keyvault secret show \
  --vault-name <kv-name-from-apply> \
  --name ssh-<pet-name>-private_key \
  --query value -o tsv > ~/.ssh/cluster_key
chmod 600 ~/.ssh/cluster_key
```

And when the cluster is done doing whatever ephemeral science you stood it up for:

```shell
terraform destroy
```

Because of the `purge_soft_delete_on_destroy` and `prevent_deletion_if_contains_resources` settings from earlier, this tears everything down cleanly and frees up the Key Vault name for the next run.

## Wrapping up

What I really like about this pattern is that it leans on the newest Terraform features — `ephemeral` resources and write-only arguments — to solve a problem that has historically been pretty awkward: how do you generate sensitive material in Terraform without that material ending up sitting in your state file forever?

For ephemeral HPC clusters, this gives me a repeatable, low-ceremony way to stand up a vault, generate a fresh keypair, store it safely, and tear it all down again — without ever having a private key on disk that I have to remember to clean up afterwards. In a future post I'll cover the Entra-based SSH option, which removes the keypair from the equation entirely, but until that's a fit for every workload, this gets us most of the way there.