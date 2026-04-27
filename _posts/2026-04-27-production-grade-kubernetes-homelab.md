---
title: "How I Built a Production-Grade Kubernetes Cluster in My Homelab"
date: 2026-04-27 12:00:00 +0000
categories: [Homelab, Kubernetes]
tags: [kubernetes, talos, terraform, argocd, cilium, devops, gitops]
description: "What I learned building a 6-node, fully GitOps-managed Kubernetes cluster at home — and why it changed how I think about infrastructure."
pin: true
---

Most Kubernetes tutorials stop at `minikube start`. Real production environments don't. Here's what I learned building a 6-node, fully GitOps-managed Kubernetes cluster at home — and why it changed how I think about infrastructure.

## The Setup

My homelab runs on XCP-ng with XenOrchestra for virtualization management. On top of that sits a Talos Linux Kubernetes cluster: 3 control plane nodes and 3 workers, all provisioned through Terraform. No clicking around in dashboards, no snowflake configs — everything is declared in code.

The stack:

- **OS:** Talos Linux v1.34.1 (immutable, API-driven, no SSH)
- **CNI:** Cilium (eBPF-based networking and network policies)
- **Ingress:** Traefik
- **Storage:** Longhorn (replicated block storage)
- **GitOps:** ArgoCD syncing from a Git repository
- **Monitoring:** Prometheus, Grafana, Loki, Promtail
- **Registry:** Harbor v2.14.0 (self-hosted, TLS with custom CA)
- **IaC:** Terraform managing the full VM and cluster lifecycle

If that sounds like overkill for a homelab — it is. That's the point.

## Why Talos Linux?

I chose Talos because it removes an entire class of problems. There's no SSH. There's no shell. There's no package manager. You interact with the OS through an API, and the machine configuration is a YAML file that Terraform manages.

This sounds limiting until you realize it means:

- No configuration drift (there's nothing to drift)
- No "I SSH'd in and tweaked something" surprises
- Upgrades are a Terraform apply, not a prayer

The tradeoff is that debugging gets harder when you can't just jump into a shell. But it forces you to build proper observability from day one, which is a habit that pays off in production environments.

## GitOps or It Didn't Happen

Every application running on my cluster is managed through ArgoCD, syncing from a Git repository. Want to deploy something? Open a PR. Want to change a config? Open a PR. Want to roll back? Revert the commit.

```yaml
# Example ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://dev.azure.com/your-org/your-project/_git/k8s-homelab-Argocd
    targetRevision: main
    path: apps/monitoring
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

The repository structure uses Kustomize with base and overlay patterns, which keeps the file count manageable as the number of applications grows. ArgoCD watches the repo and automatically reconciles any drift — if someone (or something) changes the cluster state, ArgoCD puts it back.

This workflow has a learning curve, but once it clicks, you'll never want to go back to imperative deployments. The Git history becomes your audit log, your documentation, and your disaster recovery plan all in one.

## The Security Rabbit Hole

Running Kubescape against a default Kubernetes cluster is humbling. The number of findings on a fresh install is enough to make you question every deployment you've ever done.

Here's what I focused on:

### Network Policies

By default, every pod in Kubernetes can talk to every other pod. That's fine for a tutorial, terrifying for production. I implemented deny-all default policies and then explicitly allowed only the traffic that needed to flow.

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

It's tedious but it's how it should be done.

### Resource Limits

Every container gets CPU and memory limits. No exceptions. A runaway process shouldn't be able to starve the entire node.

### Service Account Tokens

Most pods don't need to talk to the Kubernetes API. Setting `automountServiceAccountToken: false` by default is a one-line change that removes an entire attack surface.

None of this is groundbreaking. It's just the boring security work that gets skipped in every "deploy Kubernetes in 5 minutes" tutorial and then causes incidents in production.

## Lessons That Translate to Production

**Self-hosted registries matter.** Running Harbor taught me more about container supply chains than any conference talk. Setting up TLS with self-signed CAs, injecting the CA trust into every node via Terraform machine config patches, and verifying the end-to-end pull works — these are skills that directly apply to enterprise environments.

**Storage is never simple.** Longhorn is excellent, but I've had incidents. A misconfigured `podSecurityContext` once took down my Grafana PVC. You learn more from recovering from storage failures than from any documentation.

**Observability isn't optional.** When something breaks at 10 PM, you need Prometheus metrics and Loki logs to tell you what happened, because you're not going to figure it out by staring at `kubectl get pods`. The monitoring stack isn't overhead — it's the thing that lets you sleep.

**Terraform state is sacred.** I learned the hard way about managing Terraform state when dealing with a two-phase IP workflow for Talos nodes. The plan now is to move to static IPs in machine configs to simplify the lifecycle.

## The Cost

The entire homelab runs on hardware I already owned. The software stack is 100% open source. The only ongoing cost is electricity and the occasional weekend afternoon when something breaks and I fall down a debugging rabbit hole.

But the real investment is time. Setting all of this up from scratch took weeks of evenings and weekends. The difference between my homelab and a tutorial environment is the same difference between reading about swimming and actually swimming — you learn things you didn't even know you needed to learn.

## What's Next

I'm planning a Kustomize restructuring to reduce file count using base/overlay patterns and an App of Apps bootstrap for ArgoCD. I'm also looking at replacing the current two-phase Terraform IP workflow with static IPs baked into the Talos machine configs.

If you're an infrastructure engineer thinking about building a homelab, my advice is simple: skip the tutorials that give you a working cluster in 10 minutes. Build it from scratch. Break it. Fix it. Break it again. That's where the learning happens.

---

*I occasionally take on consulting work for Kubernetes architecture, GitOps pipelines, and observability. Check out my [Consulting](/consulting/) page if your team needs a hand.*
