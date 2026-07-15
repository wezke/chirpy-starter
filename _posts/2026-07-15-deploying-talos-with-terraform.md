---
title: "Deploying Talos Linux with Terraform"
date: 2026-07-15 12:00:00 +0000
categories: [Homelab, Kubernetes]
tags: [talos, terraform, kubernetes, iac, xcp-ng, homelab, devops]
description: "A deeper look at how I provision my Talos Linux Kubernetes cluster entirely with Terraform — from VM creation on XCP-ng to a bootstrapped, API-managed cluster with no clicking involved."
---

In my [homelab post]({% post_url 2026-04-27-kubernetes-homelab %}) I mentioned that the whole Talos cluster is provisioned through Terraform. A few people asked me to go deeper on that part specifically, so this is the how — from a blank hypervisor to a bootstrapped cluster, without ever touching a dashboard or an SSH session.

## Why Terraform for This?

Talos is API-driven and immutable by design. There's no SSH, no shell, no package manager — you hand it a YAML machine configuration and it becomes that machine. That pairs almost perfectly with Terraform, because both tools want the same thing: a declared end state that you can apply, version, and destroy.

The goal was simple: one `terraform apply` should take me from nothing to a running cluster, and one `terraform destroy` should give me my resources back. No manual steps in the middle.

## The Moving Parts

My homelab runs on XCP-ng with XenOrchestra for management. Once a base VM template exists, the Terraform stack has to do three things in order:

1. Create the VMs on XCP-ng (control plane + workers)
2. Generate the Talos machine configs and apply them to each node
3. Bootstrap the cluster and pull down a working `kubeconfig`

For that I lean on two providers: one for XCP-ng/XenOrchestra to build the VMs, and the [`siderolabs/talos`](https://registry.terraform.io/providers/siderolabs/talos/latest) provider to handle everything Talos-specific.

```hcl
terraform {
  required_providers {
    xenorchestra = {
      source = "vatesfr/xenorchestra"
    }
    talos = {
      source = "siderolabs/talos"
    }
  }
}
```

## Step 0: The Base Template

Before any of the Terraform steps below can run, XCP-ng needs a VM template to clone from. I built my own rather than reaching for something pre-made.

Talos ships several image variants depending on the platform, and the one that matters here is **nocloud**. It's meant for environments without a cloud-init-style metadata service — which is exactly the case on XCP-ng. There's no configdrive, no cloud-init user-data, none of that. The node boots, sits there with no configuration, and waits for a machine config to be pushed to it over the Talos API. That's it — no bootstrap scripts, no first-boot metadata service to reason about.

I downloaded the nocloud image, created a VM from it in XenOrchestra, and converted that into a template — `talos-1.11.6-nocloud-template` — so Terraform's `xenorchestra` provider just clones it per node:

```hcl
resource "xenorchestra_vm" "controlplane" {
  count     = 3
  name_label = "talos-cp-${count.index}"
  template  = data.xenorchestra_template.talos_nocloud.id
  # ...
}
```

Version-pinning the template name (rather than always pointing at "latest") means a Talos upgrade is a deliberate, visible change — build a new template, bump the reference in Terraform, apply. No node silently drifts onto a different image version underneath me.

## Step 1: Generating the Machine Secrets

Everything in a Talos cluster derives from a set of secrets — the PKI, tokens, and certificates that let nodes trust each other. The provider generates these once and Terraform holds onto them in state.

```hcl
resource "talos_machine_secrets" "this" {}
```

This is exactly why **Terraform state is sacred** (a lesson I bring up a lot). Lose this state without a backup and you've lost the keys to your cluster. I keep mine on a remote backend rather than a local file.

## Step 2: Rendering the Config for Each Node

Control plane nodes and workers get different machine configs. The provider builds them from the secrets above, and I layer my own patches on top for things like the CNI, the node's static IP, and the install disk.

```hcl
data "talos_machine_configuration" "controlplane" {
  cluster_name     = "homelab"
  cluster_endpoint = "https://10.0.0.10:6443"
  machine_type     = "controlplane"
  machine_secrets  = talos_machine_secrets.this.machine_secrets

  config_patches = [
    yamlencode({
      cluster = {
        network = {
          cni = { name = "none" }   # Cilium gets installed separately
        }
        proxy = { disabled = true }
      }
    })
  ]
}
```

Disabling the default CNI and kube-proxy here is deliberate — Cilium replaces both with its eBPF dataplane, so I don't want Talos installing the defaults only for me to rip them out later.

## Step 3: Applying and Bootstrapping

Once the VMs exist and have IPs, Terraform applies the config to each one and then bootstraps a single control plane node (only ever one — bootstrapping more than once is how you corrupt etcd).

```hcl
resource "talos_machine_configuration_apply" "controlplane" {
  count                       = length(var.controlplane_ips)
  client_configuration        = talos_machine_secrets.this.client_configuration
  machine_configuration_input = data.talos_machine_configuration.controlplane.machine_configuration
  node                        = var.controlplane_ips[count.index]
}

resource "talos_machine_bootstrap" "this" {
  depends_on           = [talos_machine_configuration_apply.controlplane]
  node                 = var.controlplane_ips[0]
  client_configuration = talos_machine_secrets.this.client_configuration
}
```

Those `depends_on` and ordering constraints matter more than they look. Terraform's dependency graph doesn't automatically know that a VM needs an IP before you can apply a config to it, so you end up being explicit about the sequence.

## Step 4: Getting a Kubeconfig Out

The nice payoff: once bootstrap finishes, Terraform can hand me a working `kubeconfig` as an output, so `kubectl` works the moment `apply` returns.

```hcl
resource "talos_cluster_kubeconfig" "this" {
  depends_on           = [talos_machine_bootstrap.this]
  node                 = var.controlplane_ips[0]
  client_configuration = talos_machine_secrets.this.client_configuration
}
```

## The Two-Phase IP Problem

This is the part that bit me, and it's the same thing I flagged as "what's next" in the homelab post. When VMs get their addresses from DHCP, Terraform has a chicken-and-egg problem: it needs the IPs to write the machine config, but the IPs don't exist until the VMs boot. That forces an awkward two-phase apply — build the VMs first, then feed their discovered IPs back in.

The fix I'm moving toward is **static IPs baked directly into the Talos machine config**. If the node's address is declared up front instead of discovered after boot, the whole thing collapses back into a single clean apply. It's a small change that removes a genuinely annoying part of the lifecycle.

## What I'd Tell Past Me

- **Back up your state before anything else.** `talos_machine_secrets` living only in a local state file is a disaster waiting to happen.
- **Bootstrap exactly one node, once.** Guard it, and never let a `count` or a loop touch it.
- **Be explicit about ordering.** Half my early failures were Terraform trying to configure a node that wasn't reachable yet.
- **Keep your patches small and readable.** The machine config is the source of truth for the whole OS — future-you will be reading it during an incident.

The end result is what I wanted from the start: the cluster is disposable in the best way. I can tear it down and rebuild it from code in minutes, and every decision about how those machines are configured lives in a Git repo instead of in my head.

---
