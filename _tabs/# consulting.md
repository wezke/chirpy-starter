---
layout: page
title: Consulting
icon: fas fa-terminal
order: 5
---

## Production-grade infrastructure, built from experience.

I design, deploy, and harden Kubernetes clusters, GitOps pipelines, and full observability stacks. Everything I offer clients, I run myself — battle-tested in a multi-node homelab that mirrors real production environments.

---

## Core Expertise

### Kubernetes Architecture
Multi-node cluster design with Talos Linux, including HA control planes, worker scaling, and security hardening via Kubescape, NetworkPolicies, and least-privilege configurations.

`Talos Linux` · `Kubescape` · `NetworkPolicies` · `RBAC`

### GitOps & Infrastructure as Code
Fully declarative infrastructure managed through Terraform and ArgoCD. Every change is version-controlled, reviewed, and automatically reconciled — no manual kubectl applies.

`Terraform` · `ArgoCD` · `Kustomize` · `Git workflows`

### Cloud-Native Networking
CNI deployment and management with Cilium, ingress routing via Traefik, TLS automation, and network segmentation for multi-tenant workloads.

`Cilium` · `Traefik` · `TLS` · `Ingress`

### Observability & Monitoring
Full-stack monitoring with Prometheus metrics, Grafana dashboards, Loki log aggregation, and Promtail collection. Alert pipelines that catch issues before users do.

`Prometheus` · `Grafana` · `Loki` · `Promtail`

### Storage & Registry
Persistent storage with Longhorn, including tuning, backup strategies, and PVC management. Private container registries with Harbor, including CA trust chains and auto-start configuration.

`Longhorn` · `Harbor` · `PVC management`

### Virtualization Platform
XCP-ng/XenOrchestra virtualization management, VM lifecycle automation, storage repository optimization, and platform maintenance.

`XCP-ng` · `XenOrchestra` · `Ansible`

---

## What I Run Daily

```terminal
wez@homelab:~$ kubectl get nodes
cp-01   Ready   control-plane   192.168.20.89
cp-02   Ready   control-plane   192.168.20.33
cp-03   Ready   control-plane   192.168.20.55
wk-01   Ready   worker          192.168.20.14
wk-02   Ready   worker          192.168.20.10
wk-03   Ready   worker          192.168.20.9

# 3 control planes + 3 workers | Talos v1.34.1 | Cilium CNI | ArgoCD GitOps
```

---

## Services

### Cluster Setup & Hardening
Go from zero to a production-ready Kubernetes cluster with proper security baselines, network policies, RBAC, and GitOps-managed deployments.

> Typical engagement: 2-4 weeks

### GitOps Pipeline Design
Implement ArgoCD with structured Kustomize overlays, automated sync policies, and a clean repo structure that scales with your team.

> Typical engagement: 1-2 weeks

### Observability Stack
Deploy and configure Prometheus, Grafana, and Loki with meaningful dashboards and alerts — not just default templates, but monitoring that actually helps you.

> Typical engagement: 1-2 weeks

### Infrastructure Audit
Review your existing Kubernetes setup for security gaps, resource waste, reliability risks, and missed best practices. Walk away with an actionable improvement plan.

> Typical engagement: 2-3 days

---

## Let's build something solid.

I take on a limited number of consulting engagements alongside my day job. If your infrastructure needs expert hands, let's talk.

📧 [wesley.billiaert2@telenet.be](mailto:wesley.billiaert2@telenet.be) — Usually respond within 24 hours
