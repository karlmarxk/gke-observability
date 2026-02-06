# GKE Observability Framework

A comprehensive SLO-based observability framework for Google Kubernetes Engine (GKE) infrastructure, designed for regulated environments (banking/financial services).

## Overview

This framework provides PromQL queries, multi-window multi-burn-rate alerts, and implementation guidance for monitoring GKE clusters using **Google Cloud Managed Service for Prometheus (GMP)**. It balances regulatory compliance (SOX, PCI-DSS), operational excellence, and customer experience.

## Structure

| File | Description |
|------|-------------|
| `gke-observability.md` | Complete framework: Tier 1 & Tier 2 SLOs, implementation guide, recording rules, dashboards |
| `Page1.md` | Control Plane SLOs -- API Server, etcd, Workload Availability, Node & Kubelet Health |
| `Page2.md` | DNS/Network SLOs -- CoreDNS performance, upstream DNS, service endpoint availability |
| `Page3.md` | Platform Component SLOs -- ArgoCD, Node Problem Detector, monitoring exporters |
| `Page4.md` | Optional (Tier 2) SLOs -- Cluster Autoscaler, HPA, PV/PVC, Image Pull, Resource Quotas |
| `Page5.md` | Concise SLO reference -- compact PromQL + alert definitions across all layers |
| `Page6.md` | GMP-native alerting blueprint -- `Rules` CRD, burn-rate paging, ticket-level alerts |

## SLO Tiers

**Tier 1 -- MUST-HAVE (pages on-call):**
- API Server availability (99.95%) & latency (P95 < 1s)
- etcd health (99.99%, commit latency < 25ms)
- DNS resolution (99.95%, P95 < 50ms)
- ArgoCD availability (99.9%)
- Node availability (99.9%)
- Customer-facing workload availability (99.9%)

**Tier 2 -- OPTIONAL (tickets, weekly review):**
- Cluster Autoscaler, HPA, PV/PVC capacity, image pull, resource quotas

## Key Principles

- **Page on SLO burn-rate**, not raw metric thresholds
- **Few SLOs that matter** over metric sprawl
- **Platform components are Tier 1** -- ArgoCD, NPD, exporters are operational infrastructure
- **Filter high-cardinality metrics** (`apiserver_request_total`) to control cost
- Uses GMP `Rules` CRD + Cloud Monitoring `select_slo_burn_rate` for alerting

## Tech Stack

- Google Kubernetes Engine (GKE)
- Google Cloud Managed Service for Prometheus (GMP)
- PromQL / Cloud Monitoring SLOs
- ArgoCD (GitOps)
- CoreDNS, kube-state-metrics, Node Problem Detector
