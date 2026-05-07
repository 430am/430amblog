---
title: "Let's Deploy CycleCloud My Way - Required Infrastructure"
series: ["Azure Cyclecloud"]
series_order: 1
date: 2026-05-07
weight: 11
draft: false
description: "Demonstrates how to use Terraform to deploy and manage CycleCloud. Part 1 of series - this part covers the backend Azure infrastructure."
slug: "cyclecloud_terraform"
tags: ["terraform", "Azure", "Cyclecloud"]
categories: ["Infrastructure as Code", "Cloud", "HPC"]
---

I've been spending a lot of time lately standing up [Azure CycleCloud](https://learn.microsoft.com/azure/cyclecloud/) for HPC customers, and one of the things that always slowed me down was the *ceremony* of getting CycleCloud onto a clean, well-built Azure foundation before I could even start thinking about clusters. There's a lot of "click around in the portal, hope you remembered the private endpoint, did you wire up Bastion?" energy to it, and that doesn't scale when you're spinning up environments for multiple projects.

So I decided to do what I always end up doing — codify the whole thing in Terraform — and split it into a series of stages so each part of the build is small, opinionated, and easy to reason about. The repo lives on my GitHub {{< github repo="430am/iac_cyclecloud" showThumbnail=true >}}

This post is the first in a series. We're going to walk through stage one: **the Azure infrastructure that has to exist before CycleCloud itself shows up to the party.** Future posts will cover building the custom CycleCloud image with Packer and then actually deploying CycleCloud on top of all of this.

## What we're building

Everything for this stage lives under `1_infrastructure/`. The end state we're going to land in is roughly this:

```text
Subscription
└── Resource Group  (rg-<suffix>)
    ├── Virtual Network  (vnet-<suffix>)
    │   ├── AzureBastionSubnet          10.100.2.0/26
    │   ├── Storage (ANF, delegated)    10.100.2.64/26
    │   ├── PrivateEndpoints            10.100.2.128/27
    │   ├── CycleCloud                  10.100.2.192/27
    │   ├── SharedServices              10.100.2.224/27
    │   └── Cluster                     10.100.0.0/23
    ├── Azure Bastion (Standard, tunneling enabled)
    ├── NAT Gateway (cluster + cyclecloud + shared subnets)
    ├── Key Vault (RBAC, private endpoint)
    ├── Shared Image Gallery + Image Definition
    ├── Log Analytics Workspace + DCE + AMPLS
    ├── Storage Account (Log Analytics ingestion, key access disabled)
    ├── User-Assigned Managed Identity
    └── Custom RBAC Role (CycleCloud Orchestrator Role)
```

That's a lot of moving parts, but each `.tf` file in the repo owns exactly one slice of it. Here's the layout:

```shell
1_infrastructure/
├── providers.tf
├── variables.tf
├── locals.tf
├── main.tf
├── network.tf
├── keyvault.tf
├── imagegallery.tf
├── monitoring.tf
├── private_endpoints.tf
├── outputs.tf
└── environments/
    └── example.tfvars
```

Let's walk it.

## providers.tf

Same idea as the [last post](../terraform-azurekeyvault) — pin the providers we need, keep the surface small:

```hcl
terraform {
  required_version = "~> 1"

  required_providers {
    azuread = { source = "hashicorp/azuread", version = "~> 3" }
    azurerm = { source = "hashicorp/azurerm", version = "~> 4" }
    random  = { source = "hashicorp/random",  version = "~> 3" }
    time    = { source = "hashicorp/time",    version = "~> 0.12" }
    tls     = { source = "hashicorp/tls",     version = "~> 4" }
  }
}

provider "azuread" {}

provider "azurerm" {
  storage_use_azuread = true
  features {}
}
```

A couple of things worth calling out:

- `azuread` is in here because we look up the deploying user later to tag resources with who deployed them. Nice when six people share a subscription.
- `time` is purely for `time_sleep` resources — Azure's control plane is *eventually* consistent, and a few of the private-endpoint and Log-Analytics-linked-storage flows need a beat to settle before downstream resources can wire into them. More on that below.
- `storage_use_azuread = true` on the `azurerm` provider tells it to use Azure AD auth instead of shared keys when it talks to Storage. We're going to disable shared-key access on the storage account, so this matters.

## variables.tf

The variables file is intentionally focused on the things you'd actually want to override per-environment, and it ships with sensible defaults for everything else:

```hcl
variable "location"           { default = "southcentralus" }
variable "admin_username"     { default = "cyclecloudadmin" }
variable "vnet_address_space" { default = ["10.100.0.0/16"] }

variable "subnets" {
  type = map(object({
    address_prefix = string
    name           = string
  }))
  default = {
    anf               = { address_prefix = "10.100.2.64/26",  name = "Storage" }
    bastion           = { address_prefix = "10.100.2.0/26",   name = "AzureBastionSubnet" }
    cluster           = { address_prefix = "10.100.0.0/23",   name = "Cluster" }
    cyclecloud        = { address_prefix = "10.100.2.192/27", name = "CycleCloud" }
    private_endpoints = { address_prefix = "10.100.2.128/27", name = "PrivateEndpoints" }
    shared            = { address_prefix = "10.100.2.224/27", name = "SharedServices" }
  }
}

variable "vm_skus" {
  type    = map(string)
  default = {
    cyclecloud = "Standard_D4ads_v6"
    imaging    = "Standard_D2ads_v6"
  }
}

variable "current_ip_address"   { default = "" }
variable "CURRENT_IP_ADDRESS"   { default = "" }   # uppercase compat with creds tfvars
```

The `subnets` map is the one I tweak most — every customer has slightly different opinions about address space — and using a `map(object(...))` means I can `for_each` over it later in `network.tf` and get a tidy set of named subnets without copy-pasting six resource blocks.

`vm_skus` lives here so the CycleCloud server VM and the Packer image-builder VM can be sized independently. The image build is short and CPU-bound, the CycleCloud server runs all the time, so they don't need to be the same shape.

The two `current_ip_address` variables (lowercase and uppercase) are a little quirky — they exist because the `creds.tfvars` file uses the uppercase shouty form to match the `ARM_*` environment variables, but I prefer lowercase elsewhere in the code. We pick one over the other in `locals.tf`.

## locals.tf

This is where I do the small bit of computation needed to keep the rest of the code clean:

```hcl
locals {
  configured_current_ip_address = trimspace(var.current_ip_address) != "" ? trimspace(var.current_ip_address) : trimspace(var.CURRENT_IP_ADDRESS)
  effective_ip_allowlist        = distinct(compact(concat(var.local_ip_address_prefixes, local.configured_current_ip_address != "" ? [local.configured_current_ip_address] : [])))

  common_tags = merge(var.tags, {
    component   = "backend-deployment"
    deployed_by = data.azuread_user.current_user.display_name
  })

  private_dns_names = [
    "privatelink.vaultcore.azure.net",
    "privatelink.blob.core.windows.net",
    "privatelink.file.core.windows.net",
    "privatelink.monitor.azure.com",
    "privatelink.ods.opinsights.azure.com",
    "privatelink.oms.opinsights.azure.com",
    "privatelink.agentsvc.azure-automation.net",
    # ...and several more
  ]

  ampls_private_dns_zones = [
    "privatelink.monitor.azure.com",
    "privatelink.ods.opinsights.azure.com",
    "privatelink.oms.opinsights.azure.com",
    "privatelink.agentsvc.azure-automation.net",
    "privatelink.blob.core.windows.net",
  ]
}
```

The interesting bits:

- `effective_ip_allowlist` collapses the lowercase/uppercase IP variables and any explicitly-passed prefixes into a single deduplicated list we can hand to NSG rules without worrying about which form the user supplied.
- `common_tags` merges the user-provided tags with two we always want — `component` (which stage of the build this is) and `deployed_by` (the human who ran apply, looked up via `data "azuread_user"`). That second one is gold during incident review.
- `private_dns_names` is the full set of `privatelink.*` zones we'll create. `ampls_private_dns_zones` is the subset that need to be linked to the Azure Monitor Private Link Scope. Keeping them as locals lets us `for_each` over them in `private_endpoints.tf`.

## main.tf

`main.tf` carries the foundational stuff every other file leans on — the resource group, the random pet name, the managed identity, the custom role, and the ephemeral SSH keypair:

```hcl
data "azuread_user"      "current_user" { object_id = data.azurerm_client_config.current.object_id }
data "azurerm_client_config" "current"  {}
data "azurerm_subscription"  "current"  {}

resource "random_password" "vm_password" {
  length  = 16
  special = true
}

resource "random_pet" "naming" {
  length    = 2
  separator = ""
}

resource "azurerm_resource_group" "cyclecloud" {
  location = var.location
  name     = "rg-${random_pet.naming.id}"
  tags     = local.common_tags
}

resource "azurerm_user_assigned_identity" "cyclecloud" {
  location            = var.location
  name                = "uaid-${random_pet.naming.id}-cyclecloud"
  resource_group_name = azurerm_resource_group.cyclecloud.name
  tags                = local.common_tags
}

ephemeral "tls_private_key" "cyclecloud_ephemeral" { algorithm = "ED25519" }
ephemeral "tls_public_key"  "cyclecloud_ephemeral" {
  private_key_openssh = ephemeral.tls_private_key.cyclecloud_ephemeral.private_key_openssh
}
```

Two themes you'll recognise from the last post:

- **`random_pet`** is back, generating us a stable two-word suffix that gets reused across every resource name in the build. Because Key Vaults, storage accounts, etc. all have global-uniqueness requirements (and their own length and character constraints), having one place that generates the suffix and then `substr`-ing it where needed is much nicer than playing whack-a-mole with naming collisions.
- **Ephemeral TLS keys** — same trick from the [Key Vault SSH post](../terraform-azurekeyvault), generated at apply time and never persisted to state. We hand the public key to the CycleCloud VM and the private key to Key Vault (via write-only secret arguments — see `keyvault.tf`).

The chunky thing in `main.tf` is the **custom RBAC role**, defined in `azurerm_role_definition.cyclecloud`. CycleCloud orchestrates VMs, scale sets, disks, NICs, NSGs, public IPs and storage on your behalf, and the default Microsoft guidance is "give it Contributor on the subscription." That's *fine* for a lab, but for anything serious we want least-privilege. The role here grants exactly the actions CycleCloud needs across `Microsoft.Compute`, `Microsoft.Network`, `Microsoft.Storage`, `Microsoft.ManagedIdentity` and a handful of marketplace/authorisation actions — and nothing else. That role then gets assigned to the user-assigned managed identity, which is what the CycleCloud VM will run as.

## network.tf

The networking block stands up the VNet, the six subnets from our variable, the Bastion + its public IP, and the NAT Gateway + its public IP:

```hcl
resource "azurerm_virtual_network" "cyclecloud" {
  address_space       = var.vnet_address_space
  location            = var.location
  name                = "vnet-${random_pet.naming.id}"
  resource_group_name = azurerm_resource_group.cyclecloud.name
  tags                = local.common_tags
}

resource "azurerm_subnet" "cyclecloud" {
  for_each             = var.subnets
  address_prefixes     = [each.value.address_prefix]
  name                 = each.value.name
  resource_group_name  = azurerm_resource_group.cyclecloud.name
  virtual_network_name = azurerm_virtual_network.cyclecloud.name

  dynamic "delegation" {
    for_each = each.key == "anf" ? [1] : []
    content {
      name = "anf-delegation"
      service_delegation {
        name    = "Microsoft.Netapp/volumes"
        actions = [
          "Microsoft.Network/networkinterfaces/*",
          "Microsoft.Network/virtualNetworks/subnets/join/action",
        ]
      }
    }
  }
}
```

The `for_each` over `var.subnets` is what makes the subnet variable carry its weight — one resource block, six subnets, named consistently. The `dynamic "delegation"` block is the slightly clever bit: only the `anf` subnet gets delegated to `Microsoft.Netapp/volumes` (because that's the one Azure NetApp Files will live in), and the rest are plain subnets. If you don't plan to use ANF you can drop the delegation entirely without changing the rest of the network.

```hcl
resource "azurerm_bastion_host" "cyclecloud" {
  location            = var.location
  name                = "bastion-${random_pet.naming.id}"
  resource_group_name = azurerm_resource_group.cyclecloud.name
  sku                 = "Standard"
  copy_paste_enabled  = true
  tunneling_enabled   = true

  ip_configuration {
    name                 = "bastion-ip-config"
    public_ip_address_id = azurerm_public_ip.bastion.id
    subnet_id            = azurerm_subnet.cyclecloud["bastion"].id
  }
}

resource "azurerm_nat_gateway"                       "cyclecloud" { ... }
resource "azurerm_nat_gateway_public_ip_association" "cyclecloud" { ... }
resource "azurerm_subnet_nat_gateway_association"    "cyclecloud" { subnet_id = azurerm_subnet.cyclecloud["cyclecloud"].id ... }
resource "azurerm_subnet_nat_gateway_association"    "cluster"    { subnet_id = azurerm_subnet.cyclecloud["cluster"].id ... }
resource "azurerm_subnet_nat_gateway_association"    "shared"     { subnet_id = azurerm_subnet.cyclecloud["shared"].id ... }
```

Bastion is **Standard SKU** with `tunneling_enabled = true`, which is the magic flag that lets you `az network bastion tunnel` to ssh through Bastion using your local OpenSSH client. That's how I get into the CycleCloud VM later — no public IPs on the workload itself.

The NAT Gateway is attached to the three subnets that need outbound internet (CycleCloud server, cluster nodes, shared services). The Bastion subnet doesn't need it, and the ANF and PrivateEndpoints subnets shouldn't have it. The result is a single, predictable egress IP for the cluster — useful if a downstream service wants to allowlist us.

## keyvault.tf

The Key Vault itself is straightforward — RBAC-enabled, standard SKU, soft-delete on with a 7-day retention:

```hcl
resource "azurerm_key_vault" "cyclecloud" {
  location                   = var.location
  name                       = substr("kv${random_pet.naming.id}", 0, 24)
  resource_group_name        = azurerm_resource_group.cyclecloud.name
  sku_name                   = "standard"
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  soft_delete_retention_days = 7
  purge_protection_enabled   = false
  enabled_for_deployment     = true
  rbac_authorization_enabled = true
  tags                       = local.common_tags
}

resource "azurerm_role_assignment" "kv_admin" {
  principal_id         = data.azurerm_client_config.current.object_id
  role_definition_name = "Key Vault Administrator"
  scope                = azurerm_key_vault.cyclecloud.id
}

resource "azurerm_key_vault_secret" "password" {
  depends_on   = [azurerm_role_assignment.kv_admin]
  key_vault_id = azurerm_key_vault.cyclecloud.id
  name         = "cc-${random_pet.naming.id}-password"
  value        = random_password.vm_password.result
}

resource "azurerm_key_vault_secret" "private_key" {
  depends_on       = [azurerm_role_assignment.kv_admin]
  key_vault_id     = azurerm_key_vault.cyclecloud.id
  name             = "cc-${random_pet.naming.id}-private-key"
  value_wo         = ephemeral.tls_private_key.cyclecloud_ephemeral.private_key_openssh
  value_wo_version = 1
}

resource "azurerm_key_vault_secret" "public_key" {
  depends_on       = [azurerm_role_assignment.kv_admin]
  key_vault_id     = azurerm_key_vault.cyclecloud.id
  name             = "cc-${random_pet.naming.id}-public-key"
  value_wo         = ephemeral.tls_public_key.cyclecloud_ephemeral.public_key_openssh
  value_wo_version = 1
}
```

This is the same pattern from the previous post: the deploying identity grants itself **Key Vault Administrator** scoped to this vault, and then we land three secrets in it:

- `cc-<suffix>-password` — the random VM admin password (a regular `value`, because it isn't ephemeral material).
- `cc-<suffix>-private-key` — the ephemeral SSH private key, written via `value_wo` so it never touches state.
- `cc-<suffix>-public-key` — the matching public key, also via `value_wo`.

The vault is reachable only over its private endpoint (we'll wire that up in `private_endpoints.tf`), so even though the secrets are accessible by RBAC, they're not accessible from the public internet at all.

## imagegallery.tf

The Shared Image Gallery is where the **stage 2** Packer build is going to publish its output. We pre-create the gallery and the image *definition* now so the Packer config has a stable place to publish image *versions* into:

```hcl
resource "azurerm_shared_image_gallery" "cyclecloud" {
  location            = var.location
  name                = substr("sig${random_pet.naming.id}", 0, 24)
  resource_group_name = azurerm_resource_group.cyclecloud.name
  description         = "Shared Image Gallery for CycleCloud"
  tags                = local.common_tags
}

resource "azurerm_shared_image" "cyclecloud" {
  gallery_name        = azurerm_shared_image_gallery.cyclecloud.name
  location            = var.location
  name                = "image-${random_pet.naming.id}"
  os_type             = "Linux"
  resource_group_name = azurerm_resource_group.cyclecloud.name

  identifier {
    publisher = "microsoft-dsvm"
    offer     = "ubuntu-hpc"
    sku       = "2404"
  }

  description                       = "Shared Image for CycleCloud installed on Ubuntu 24.04"
  hyper_v_generation                = "V2"
  min_recommended_vcpu_count        = 4
  min_recommended_memory_in_gb      = 16
  accelerated_network_support_enabled = true
  disk_controller_type_nvme_enabled   = true
}
```

The base image we'll be customising is `microsoft-dsvm/ubuntu-hpc/2404` — Microsoft's HPC-tuned Ubuntu 24.04, which already has things like NVIDIA drivers, the right kernel parameters, and the InfiniBand stack baked in. Hyper-V Gen V2, NVMe, and Accelerated Networking are all on, because we want to be able to use modern VM SKUs without surprises.

The fact that `sig_name` and `sig_image_name` are exposed as outputs (see `outputs.tf` below) is what stitches stage 1 and stage 2 together — Packer will read those values to know where to publish.

## monitoring.tf

Observability shows up in `monitoring.tf`. We create one Log Analytics workspace, a backing storage account, a Data Collection Endpoint, and a pile of diagnostic settings that fan everything we can into the workspace:

```hcl
resource "azurerm_log_analytics_workspace" "cyclecloud" {
  location            = var.location
  name                = "${random_pet.naming.id}logs"
  resource_group_name = azurerm_resource_group.cyclecloud.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
  tags                = local.common_tags

  identity { type = "SystemAssigned" }
}

resource "azurerm_storage_account" "monitoring" {
  name                            = substr("mon${random_pet.naming.id}", 0, 24)
  resource_group_name             = azurerm_resource_group.cyclecloud.name
  location                        = var.location
  account_tier                    = "Standard"
  account_replication_type        = "LRS"
  shared_access_key_enabled       = false
  allow_nested_items_to_be_public = false
  public_network_access_enabled   = true

  network_rules {
    default_action = "Deny"
    bypass         = ["AzureServices", "Logging", "Metrics"]
  }
}

resource "azurerm_role_assignment" "monitoring" {
  for_each             = toset(["Storage Table Data Contributor", "Storage Blob Data Contributor"])
  principal_id         = azurerm_log_analytics_workspace.cyclecloud.identity[0].principal_id
  role_definition_name = each.key
  scope                = azurerm_storage_account.monitoring.id
}

resource "time_sleep" "linked_storage_wait" {
  create_duration = "60s"
  depends_on      = [azurerm_role_assignment.monitoring]
}

resource "azurerm_log_analytics_linked_storage_account" "monitoring" {
  data_source_type    = "Ingestion"
  resource_group_name = azurerm_resource_group.cyclecloud.name
  storage_account_ids = [azurerm_storage_account.monitoring.id]
  workspace_id        = azurerm_log_analytics_workspace.cyclecloud.id

  depends_on = [azurerm_role_assignment.monitoring, time_sleep.linked_storage_wait]
}
```

A few things to unpack:

- The Log Analytics workspace gets a **system-assigned managed identity**, and we hand that identity `Storage Blob Data Contributor` and `Storage Table Data Contributor` on the monitoring storage account. That's how the workspace authenticates to its own backing store now that we've turned **shared key access off** (`shared_access_key_enabled = false`). No connection strings, no secrets in environment variables — RBAC only.
- The storage account itself denies traffic by default and only allows AzureServices/Logging/Metrics through — so even if a key did leak, you couldn't reach it from the internet.
- `time_sleep.linked_storage_wait` is the kind of wart that shows up when you're chaining resources whose IAM consistency doesn't catch up immediately. The role assignments report back as "done" instantly, but the actual data plane needs a beat to catch up before the linked storage account resource will succeed. Sixty seconds is not pretty, but it is *reliable*, and that's what I'm optimising for.

The rest of `monitoring.tf` is `azurerm_monitor_diagnostic_setting` blocks for Bastion, Key Vault, NAT Gateway, both public IPs, and the VNet — sending audit logs, DDoS notifications, flow logs, and metrics off to the workspace. This is the kind of thing you only set up *once*, in code, and then you have it forever for every environment you stand up.

## private_endpoints.tf

Everything that should be reachable privately gets a private endpoint here — Key Vault, the Azure Monitor Private Link Scope (AMPLS), and the monitoring storage account. We also create the full set of `privatelink.*` private DNS zones up front and link them all to the VNet, so DNS resolution Just Works for any service we add later:

```hcl
resource "azurerm_private_dns_zone" "zones" {
  for_each            = toset(local.private_dns_names)
  name                = each.key
  resource_group_name = azurerm_resource_group.cyclecloud.name
}

resource "azurerm_private_dns_zone_virtual_network_link" "zone_links" {
  for_each              = toset(local.private_dns_names)
  name                  = "${each.key}-link"
  private_dns_zone_name = each.value
  resource_group_name   = azurerm_resource_group.cyclecloud.name
  virtual_network_id    = azurerm_virtual_network.cyclecloud.id

  depends_on = [azurerm_private_dns_zone.zones]
}

resource "azurerm_monitor_private_link_scope" "ampls" {
  name                  = "ampls-${random_pet.naming.id}"
  resource_group_name   = azurerm_resource_group.cyclecloud.name
  ingestion_access_mode = "PrivateOnly"
}

resource "azurerm_monitor_private_link_scoped_service" "ampls" {
  linked_resource_id  = azurerm_log_analytics_workspace.cyclecloud.id
  name                = "link-${random_pet.naming.id}"
  resource_group_name = azurerm_resource_group.cyclecloud.name
  scope_name          = azurerm_monitor_private_link_scope.ampls.name
}

resource "time_sleep" "ampls_wait" {
  create_duration = "60s"
  depends_on      = [azurerm_monitor_private_link_scope.ampls]
}

resource "azurerm_private_endpoint" "ampls" {
  name                = "pe-ampls-${random_pet.naming.id}"
  resource_group_name = azurerm_resource_group.cyclecloud.name
  location            = var.location
  subnet_id           = azurerm_subnet.cyclecloud["private_endpoints"].id

  private_service_connection {
    name                           = "psc-${random_pet.naming.id}"
    private_connection_resource_id = azurerm_monitor_private_link_scope.ampls.id
    is_manual_connection           = false
    subresource_names              = ["azuremonitor"]
  }

  private_dns_zone_group {
    name                 = "pdzg-${random_pet.naming.id}"
    private_dns_zone_ids = [for zone in local.ampls_private_dns_zones : azurerm_private_dns_zone.zones[zone].id]
  }

  depends_on = [time_sleep.ampls_wait]
}
```

Two patterns repeat throughout this file:

- **`for_each` over a local list of zone names** for the DNS zones and zone links — one place to add a zone, two resources update.
- **A `time_sleep` before private endpoints that depend on freshly-created control-plane objects** (the AMPLS in particular). Same workaround as in `monitoring.tf`. Annoying, reliable, well worth the 60 seconds.

The Key Vault and monitoring storage account get their own `azurerm_private_endpoint` blocks that look very similar to the AMPLS one, each pointing at the appropriate `privatelink.*` DNS zone. After this stage, **none** of these services are reachable over the public internet from outside the VNet.

## outputs.tf

The outputs are the contract this stage exposes to the next ones (Packer image build, then CycleCloud deployment). The important ones:

```hcl
output "resource_group_name"      { value = azurerm_resource_group.cyclecloud.name }
output "resource_group_location"  { value = azurerm_resource_group.cyclecloud.location }
output "key_vault_name"           { value = azurerm_key_vault.cyclecloud.name }
output "key_vault_uri"            { value = azurerm_key_vault.cyclecloud.vault_uri }
output "key_vault_public_key_secret_name"  { value = azurerm_key_vault_secret.public_key.name }
output "key_vault_private_key_secret_name" { value = azurerm_key_vault_secret.private_key.name }
output "sig_name"                 { value = azurerm_shared_image_gallery.cyclecloud.name }
output "sig_image_name"           { value = azurerm_shared_image.cyclecloud.name }
output "virtual_network_name"     { value = azurerm_virtual_network.cyclecloud.name }
output "log_analytics_workspace_name" { value = azurerm_log_analytics_workspace.cyclecloud.name }
```

Crucially, **no secret material is ever output**. The Key Vault outputs are *names* and *URIs* — pointers to where the secrets live. To actually use them you go through Key Vault RBAC, which is exactly the boundary we want.

## Putting it all together

Once your credentials file is set up under `environments/`, deployment is the usual three-line dance:

```shell
cd 1_infrastructure
cp environments/example.tfvars environments/creds.tfvars
# ...edit creds.tfvars with your subscription / SP details and CURRENT_IP_ADDRESS

export ARM_SUBSCRIPTION_ID="<subscription-id>"
export ARM_CLIENT_ID="<client-id>"
export ARM_CLIENT_SECRET="<client-secret>"
export ARM_TENANT_ID="<tenant-id>"

terraform init
terraform plan  -var-file=environments/creds.tfvars
terraform apply -var-file=environments/creds.tfvars
```

When apply finishes, grab the values stage 2 will need:

```shell
RG_NAME=$(terraform output -raw resource_group_name)
SIG_NAME=$(terraform output -raw sig_name)
SIG_IMAGE_NAME=$(terraform output -raw sig_image_name)
```

And when you're done with the whole environment:

```shell
terraform destroy -var-file=environments/creds.tfvars
```

## Wrapping up

What I really like about laying it out this way is that **stage 1 is boring on purpose**. By the time we get to actually installing CycleCloud in a later post, we know the network is right, the secrets store is locked down behind a private endpoint, the VM identity has exactly the rights it needs (and nothing more), there's a Shared Image Gallery waiting for a custom image, and every diagnostic stream we care about is already pointing at Log Analytics. Nothing about CycleCloud's install gets to be "the part where I also had to fix the network."

In **part 2** we'll use Packer to build a custom Ubuntu 24.04 image with CycleCloud pre-installed, and publish it as a version into the Shared Image Gallery we just stood up. After that, **part 3** will deploy the CycleCloud VM itself, wire it to the managed identity, and pull its SSH credentials straight out of Key Vault. By the end of the series, "spin up a fresh CycleCloud environment" should be `terraform apply` away — three times.

