---
title: "Terraform Key Vaults and Azure, Oh My!"
weight: 11
draft: false
description: "Demonstrates how to use the latest Terraform capabilities to deploy and manage SSH keypairs in Azure for VM authentication using Azure Key Vault"
slug: "terraform-azurekeyvault"
tags: ["terraform", "azure", "configuration"]
---

There are two "good" ways to manage authentication to Linux VMs in Azure:

1. SSH keys (which can be managed in a few ways, none of them super awesome.)
2. Entra authentication through the Entra SSH extension

I'll discuss the second option in a later post, as it's come a *LONG* way, but in this post I'm going to discuss SSH authentication. Most of my customers leverage SSH as their authentication of choice, as it is the form they are most used to in the high performance computing world. In HPC, you have a Linux cluster that you will use SSH to authenticate to to run your jobs. Generally, this means you're managing SSH keys in a variety of ways, none of them cloud-native.

I've been working with a number of customers who are building clusters in Azure for various scientific projects, and they often are building ephemeral clusters, meaning they want to spin it up and destroy it all when they have their results. Often, this means they spin up and down multiple clusters for different projects, and wish to do it in a repeatable manner.

I'm working on that, and through that process I found myself struggling with managing SSH keys while testing different parts of my builds. Noodling around in terraform, I figured out how to manage SSH keys in a repeatable fashion, using Azure Key Vault. This is what I'm going to show in this post.

## 