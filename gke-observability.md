# Comprehensive GKE Observability Framework for Banking Infrastructure

## Executive Summary

For a multinational bank running GKE, observability must balance **regulatory compliance**, **operational excellence**, and **customer experience**. This framework prioritizes:

1. **Customer-Facing SLOs** - Direct impact on banking services
2. **Platform Reliability SLOs** - Foundation for all services
3. **Security & Compliance Monitoring** - Non-negotiable for financial services
4. **Operational Efficiency** - Cost and performance optimization

Key implementation realities in Google Kubernetes Engine / Google Cloud:

* GKE control plane / kube-state / cAdvisor+kubelet metric packages write into Cloud Monitoring as Prometheus metrics using `prometheus_target` and metric names prefixed `prometheus.googleapis.com/`
* `apiserver_request_total` and `apiserver_request_duration_seconds` are **very high-cardinality**; you must filter/group intentionally
* cAdvisor/Kubelet curated metrics are explicitly a **subset** chosen to reduce ingestion volume/cost
* For GKE-based services in Cloud Monitoring, you typically create SLOs using **"Other" custom SLI** (time-series ratio / distribution cut)

---

## Table of Contents

1. [MUST-HAVE SLOs (Tier 1 - Critical)](#part-1-must-have-slos-tier-1---critical)
   - [Control Plane SLOs](#11-control-plane-slos)
   - [DNS/Network SLOs](#12-dnsnetwork-slos)
   - [Platform Component SLOs](#13-platform-component-slos)
   - [Workload Management SLOs](#14-workload-management-slos)
   - [Node & Kubelet SLOs](#15-node--kubelet-slos)
2. [OPTIONAL SLOs (Tier 2 - Operational Excellence)](#part-2-optional-slos-tier-2---operational-excellence)
   - [Cluster Autoscaler](#21-cluster-autoscaler-performance)
   - [HPA](#22-horizontal-pod-autoscaler-hpa-performance)
   - [Storage](#23-persistent-volume-performance)
   - [Container Registry](#24-image-pull-performance)
   - [Resource Quotas](#25-namespace-resource-quota-usage)
3. [Implementation Guide](#part-3-implementation-guide)
   - [GCP Managed Prometheus Setup](#31-gcp-managed-prometheus-setup)
   - [Alert Configuration](#32-alert-configuration)
   - [Dashboard Integration](#33-dashboard-integration)

---

# Part 1: MUST-HAVE SLOs (Tier 1 - Critical)

## 1.1 Control Plane SLOs

### 1.1.1 API Server Availability

**SLO Target**: 99.95% over 30-day window (allowing ~22 minutes downtime/month)

**Business Rationale**:
- API server downtime = complete platform outage
- Banking services become unavailable
- Cannot deploy emergency fixes or rollbacks
- Regulatory reporting systems may fail

**PromQL Queries**:

```promql
# Current SLO Compliance (30-day window)
(
  sum(rate(apiserver_request_total{code=~"2.."}[30d]))
  /
  sum(rate(apiserver_request_total[30d]))
) * 100

# Error Budget Remaining (%)
(1 - (
  (1 - sum(rate(apiserver_request_total{code=~"2.."}[30d])) / sum(rate(apiserver_request_total[30d])))
  /
  (1 - 0.9995)
)) * 100

# Error Budget Consumption Rate
(
  1 - (
    sum(rate(apiserver_request_total{code=~"2.."}[1h]))
    /
    sum(rate(apiserver_request_total[1h]))
  )
) / (1 - 0.9995)
```

**Multi-Window Multi-Burn-Rate Alerts**:

```promql
# CRITICAL: Fast Burn (14.4x) - 1 hour window
- alert: APIServerFastErrorBudgetBurn
  expr: |
    (
      1 - (
        sum(rate(apiserver_request_total{code=~"2.."}[1h]))
        /
        sum(rate(apiserver_request_total[1h]))
      )
    ) > (14.4 * 0.0005)
    and
    (
      1 - (
        sum(rate(apiserver_request_total{code=~"2.."}[5m]))
        /
        sum(rate(apiserver_request_total[5m]))
      )
    ) > (14.4 * 0.0005)
  for: 2m
  labels:
    severity: critical
    component: api-server
    burn_rate: fast
    impact: platform-outage
  annotations:
    summary: "API Server burning error budget at 14.4x rate"
    description: |
      CRITICAL: API Server availability severely degraded

      Current Error Rate: {{ $value | humanizePercentage }}
      Error Budget Burn Rate: 14.4x (consuming 30 days in ~2 days)

      IMMEDIATE INVESTIGATION:
      1. Check API server pod status and logs
      2. Verify etcd cluster health
      3. Review recent control plane changes
      4. Check authentication/authorization systems
      5. Examine load balancer health
    runbook_url: "https://runbooks.bank.internal/sre/gke/api-server-fast-burn"
    incident_severity: "P1"

# WARNING: Slow Burn (6x) - 6 hour window
- alert: APIServerSlowErrorBudgetBurn
  expr: |
    (
      1 - (
        sum(rate(apiserver_request_total{code=~"2.."}[6h]))
        /
        sum(rate(apiserver_request_total[6h]))
      )
    ) > (6 * 0.0005)
    and
    (
      1 - (
        sum(rate(apiserver_request_total{code=~"2.."}[30m]))
        /
        sum(rate(apiserver_request_total[30m]))
      )
    ) > (6 * 0.0005)
  for: 15m
  labels:
    severity: warning
    component: api-server
    burn_rate: slow
  annotations:
    summary: "API Server burning error budget at 6x rate"
    runbook_url: "https://runbooks.bank.internal/sre/gke/api-server-slow-burn"
```

---

### 1.1.2 API Server Latency

**SLO Target**:
- P95 < 1 second for mutating operations
- P99 < 5 seconds for mutating operations

**Business Rationale**:
- Slow API responses delay critical operations
- Payment processing systems require fast Kubernetes operations
- Automated trading platforms need low-latency infrastructure

**PromQL Queries**:

```promql
# P95 Latency
histogram_quantile(0.95,
  sum(rate(apiserver_request_duration_seconds_bucket{
    verb=~"POST|PUT|PATCH|DELETE",
    subresource!~"proxy|attach|exec|portforward"
  }[5m])) by (le, verb, resource)
)

# P99 Latency
histogram_quantile(0.99,
  sum(rate(apiserver_request_duration_seconds_bucket{
    verb=~"POST|PUT|PATCH|DELETE",
    subresource!~"proxy|attach|exec|portforward"
  }[5m])) by (le, verb, resource)
)
```

**Alert Definitions**:

```promql
# WARNING: P95 Latency Exceeds SLO
- alert: APIServerLatencyP95High
  expr: |
    histogram_quantile(0.95,
      sum(rate(apiserver_request_duration_seconds_bucket{
        verb=~"POST|PUT|PATCH|DELETE",
        subresource!~"proxy|attach|exec|portforward"
      }[10m])) by (le)
    ) > 1.0
  for: 10m
  labels:
    severity: warning
    component: api-server
  annotations:
    summary: "API Server P95 latency exceeds 1s SLO"

# CRITICAL: P99 Latency Critically High
- alert: APIServerLatencyP99Critical
  expr: |
    histogram_quantile(0.99,
      sum(rate(apiserver_request_duration_seconds_bucket{
        verb=~"POST|PUT|PATCH|DELETE",
        subresource!~"proxy|attach|exec|portforward"
      }[5m])) by (le)
    ) > 5.0
  for: 5m
  labels:
    severity: critical
    component: api-server
  annotations:
    summary: "API Server P99 latency critically high (>5s)"
    incident_severity: "P1"
```

---

### 1.1.3 etcd Cluster Health

**SLO Target**:
- 99.99% availability
- P99 backend commit latency < 25ms
- P99 WAL fsync latency < 10ms
- Zero leader changes under normal operation

**Business Rationale**:
- etcd is single point of failure for entire cluster
- Stores all cluster state including secrets
- Banking: PCI-DSS compliance requires high availability

**PromQL Queries**:

```promql
# Leader Stability
rate(etcd_server_leader_changes_seen_total[10m])

# Backend Commit Duration P99
histogram_quantile(0.99,
  rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])
) * 1000  # milliseconds

# WAL Fsync Duration P99
histogram_quantile(0.99,
  rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])
) * 1000

# Proposal Failure Rate
rate(etcd_server_proposals_failed_total[5m])
```

**Alert Definitions**:

```promql
# CRITICAL: etcd Leader Changes
- alert: EtcdLeaderChanges
  expr: |
    increase(etcd_server_leader_changes_seen_total[15m]) > 3
  labels:
    severity: critical
    component: etcd
    impact: platform-stability
  annotations:
    summary: "etcd cluster experiencing leadership instability"
    incident_severity: "P0"

# CRITICAL: etcd Backend Commit Latency High
- alert: EtcdHighBackendCommitLatency
  expr: |
    histogram_quantile(0.99,
      rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])
    ) > 0.025  # 25ms
  for: 5m
  labels:
    severity: critical
    component: etcd
  annotations:
    summary: "etcd backend commit latency exceeds 25ms"
    incident_severity: "P1"
```

---

### 1.1.4 Admission Controller Performance

**SLO Target**: 99.9% success rate, P99 < 5 seconds

**PromQL Queries**:

```promql
# Admission Webhook Success Rate
sum(rate(apiserver_admission_webhook_admission_duration_seconds_count{rejected="false"}[5m]))
/
sum(rate(apiserver_admission_webhook_admission_duration_seconds_count[5m])) * 100

# Admission Webhook Latency P99
histogram_quantile(0.99,
  sum(rate(apiserver_admission_webhook_admission_duration_seconds_bucket[5m]))
  by (le, name, operation)
)
```

---

## 1.2 DNS/Network SLOs

### 1.2.1 DNS Resolution Performance

**SLO Target**:
- 99.95% DNS query success rate
- P95 DNS query latency < 50ms
- P99 DNS query latency < 100ms
- CoreDNS pod availability: 99.9%

**Business Rationale**:
- Every microservice call requires DNS resolution
- Payment processing makes hundreds of DNS queries per transaction
- DNS failures cause cascading service failures
- Trading platforms: milliseconds matter

**PromQL Queries**:

```promql
# DNS Success Rate (5-minute window)
(
  sum(rate(coredns_dns_responses_total{rcode="NOERROR"}[5m]))
  /
  sum(rate(coredns_dns_responses_total[5m]))
) * 100

# DNS Query Latency P95
histogram_quantile(0.95,
  sum(rate(coredns_dns_request_duration_seconds_bucket[5m]))
  by (le, server, zone)
) * 1000  # milliseconds

# DNS Query Latency P99
histogram_quantile(0.99,
  sum(rate(coredns_dns_request_duration_seconds_bucket[5m]))
  by (le, server, zone)
) * 1000

# DNS Cache Hit Ratio
(
  sum(rate(coredns_cache_hits_total[5m]))
  /
  (sum(rate(coredns_cache_hits_total[5m])) + sum(rate(coredns_cache_misses_total[5m])))
) * 100

# CoreDNS Pod Availability
(
  count(kube_pod_status_phase{
    namespace="kube-system",
    pod=~"coredns.*",
    phase="Running"
  })
  /
  count(kube_pod_info{
    namespace="kube-system",
    pod=~"coredns.*"
  })
) * 100
```

**Alert Definitions**:

```promql
# CRITICAL: Fast DNS Error Budget Burn
- alert: CoreDNSFastErrorBudgetBurn
  expr: |
    (
      1 - (
        sum(rate(coredns_dns_responses_total{rcode="NOERROR"}[1h]))
        /
        sum(rate(coredns_dns_responses_total[1h]))
      )
    ) > (14.4 * 0.0005)
  for: 2m
  labels:
    severity: critical
    component: dns
    impact: platform-wide
  annotations:
    summary: "CoreDNS burning error budget at 14.4x rate"
    incident_severity: "P0"

# WARNING: DNS P95 Latency High
- alert: CoreDNSLatencyP95High
  expr: |
    histogram_quantile(0.95,
      sum(rate(coredns_dns_request_duration_seconds_bucket[10m])) by (le)
    ) > 0.050  # 50ms
  for: 10m
  labels:
    severity: warning
    component: dns
  annotations:
    summary: "CoreDNS P95 latency exceeds 50ms SLO"

# CRITICAL: CoreDNS Pod Availability Low
- alert: CoreDNSPodAvailabilityLow
  expr: |
    (
      count(kube_pod_status_phase{
        namespace="kube-system",
        pod=~"coredns.*",
        phase="Running"
      })
      /
      count(kube_pod_info{
        namespace="kube-system",
        pod=~"coredns.*"
      })
    ) < 0.5
  for: 2m
  labels:
    severity: critical
    component: dns
  annotations:
    summary: "Less than 50% of CoreDNS pods are running"
    incident_severity: "P0"
```

---

### 1.2.2 Service Mesh / Network Connectivity

**SLO Target**:
- 99.95% service endpoint availability
- Service-to-service connection success rate: 99.9%

**PromQL Queries**:

```promql
# Service Endpoint Availability
(
  kube_endpoint_address_available
  /
  (kube_endpoint_address_available + kube_endpoint_address_not_ready)
) * 100

# Services with No Endpoints
count(
  (kube_endpoint_address_available + kube_endpoint_address_not_ready) == 0
) by (namespace, endpoint)
```

**Alert Definitions**:

```promql
# CRITICAL: Service Has No Endpoints
- alert: ServiceHasNoEndpoints
  expr: |
    (
      sum(kube_endpoint_address_available) by (namespace, endpoint)
      +
      sum(kube_endpoint_address_not_ready) by (namespace, endpoint)
    ) == 0
  for: 3m
  labels:
    severity: critical
    component: networking
  annotations:
    summary: "Service {{ $labels.namespace }}/{{ $labels.endpoint }} has zero endpoints"
    incident_severity: "P1"
```

---

## 1.3 Platform Component SLOs

### 1.3.1 GitOps / ArgoCD SLOs

**SLO Target**:
- 99.9% ArgoCD API availability
- Application sync success rate: 99.5%
- Sync latency P95 < 60 seconds
- Webhook delivery success: 99.9%

**Business Rationale**:
- Deployment pipeline (SOX audit trail)
- Emergency rollbacks require ArgoCD
- Drift detection for security compliance
- GitOps integrity

**PromQL Queries**:

```promql
# ArgoCD API Success Rate
(
  sum(rate(argocd_server_api_requests_total{code=~"2.."}[5m]))
  /
  sum(rate(argocd_server_api_requests_total[5m]))
) * 100

# Application Sync Success Rate
(
  sum(rate(argocd_app_sync_total{phase="Succeeded"}[5m]))
  /
  sum(rate(argocd_app_sync_total[5m]))
) * 100

# Application Sync Duration P95
histogram_quantile(0.95,
  sum(rate(argocd_app_sync_duration_seconds_bucket[5m]))
  by (le, namespace, name)
)

# Applications Out of Sync
count(argocd_app_info{sync_status="OutOfSync"})

# Webhook Delivery Success Rate
(
  sum(rate(argocd_notifications_deliveries_total{succeeded="true"}[5m]))
  /
  sum(rate(argocd_notifications_deliveries_total[5m]))
) * 100
```

**Alert Definitions**:

```promql
# CRITICAL: ArgoCD API Server Down
- alert: ArgoCDAPIServerDown
  expr: |
    up{job="argocd-server"} == 0
  for: 2m
  labels:
    severity: critical
    component: argocd
    compliance_risk: critical
  annotations:
    summary: "ArgoCD API server is down"
    description: |
      CRITICAL: Deployment pipeline blocked
      - Cannot deploy applications
      - Cannot perform emergency rollbacks
      - SOX compliance: Audit trail broken
    incident_severity: "P1"

# CRITICAL: High Sync Failure Rate
- alert: ArgoCDSyncFailureRateHigh
  expr: |
    (
      sum(rate(argocd_app_sync_total{phase!="Succeeded"}[30m]))
      /
      sum(rate(argocd_app_sync_total[30m]))
    ) > 0.05
  for: 10m
  labels:
    severity: warning
    component: argocd
  annotations:
    summary: "ArgoCD sync failure rate at {{ $value | humanizePercentage }}"

# CRITICAL: Multiple Apps Out of Sync
- alert: ArgoCDMultipleAppsOutOfSync
  expr: |
    count(argocd_app_info{sync_status="OutOfSync"}) > 10
  for: 15m
  labels:
    severity: warning
    component: argocd
    security_risk: medium
  annotations:
    summary: "{{ $value }} ArgoCD applications out of sync"
```

---

### 1.3.2 Node Problem Detector SLOs

**SLO Target**:
- 99.9% NPD daemonset availability
- Problem detection latency < 60 seconds

**Business Rationale**:
- Early warning system for node issues
- Prevents cascade failures
- Detects hardware failures before data corruption

**PromQL Queries**:

```promql
# NPD DaemonSet Availability
(
  kube_daemonset_status_number_ready{
    namespace="kube-system",
    daemonset="node-problem-detector"
  }
  /
  kube_daemonset_status_desired_number_scheduled{
    namespace="kube-system",
    daemonset="node-problem-detector"
  }
) * 100

# Active Node Problems by Type
sum(node_problem_detector_problems{type!=""})
by (node, type, reason)
```

---

### 1.3.3 Observability Pipeline SLOs

**SLO Target**: 99.9% scrape target availability

**PromQL Queries**:

```promql
# Critical Scrape Target Availability
avg_over_time(up{
  cluster="CLUSTER",
  job=~"kube-state-metrics|gmp-collector|argocd.*|gatekeeper.*"
}[30d])
```

---

## 1.4 Workload Management SLOs

### 1.4.1 Pod Scheduling Latency

**SLO Target**:
- P95 < 5s for standard pods
- P99 < 15s

**PromQL Queries**:

```promql
# Scheduling Latency P95
histogram_quantile(0.95,
  sum(rate(scheduler_pod_scheduling_sli_duration_seconds_bucket[5m])) by (le)
)

# Scheduling Success Rate
sum(rate(scheduler_schedule_attempts_total{result="scheduled"}[5m]))
/
sum(rate(scheduler_schedule_attempts_total[5m])) * 100

# Unschedulable Backlog
sum(scheduler_pending_pods{queue="unschedulable"})
```

---

### 1.4.2 Workload Availability

**SLO Target**:
- 99.9% pod availability for customer-facing services
- 99.5% for internal services

**PromQL Queries**:

```promql
# Pod Availability by Namespace
(
  sum(kube_pod_status_phase{
    phase="Running",
    namespace=~"customer-.*|payments"
  }) by (namespace)
  /
  sum(kube_pod_status_phase{
    namespace=~"customer-.*|payments"
  }) by (namespace)
) * 100

# Deployment Replica Availability
(
  kube_deployment_status_replicas_available
  /
  kube_deployment_spec_replicas
) * 100
```

**Alert Definitions**:

```promql
# CRITICAL: Customer Service Degraded
- alert: CustomerServiceDegraded
  expr: |
    (
      sum(kube_pod_status_phase{
        phase="Running",
        namespace=~"customer-.*|payments"
      }) by (namespace)
      /
      sum(kube_pod_status_phase{
        namespace=~"customer-.*|payments"
      }) by (namespace)
    ) < 0.999
  for: 2m
  labels:
    severity: critical
    component: workload
    service_tier: customer-facing
  annotations:
    summary: "Customer service {{ $labels.namespace }} below 99.9% availability"
    incident_severity: "P1"
```

---

## 1.5 Node & Kubelet SLOs

### 1.5.1 Node Availability

**SLO Target**: 99.9% node uptime

**PromQL Queries**:

```promql
# Node Availability Ratio
(
  sum(kube_node_status_condition{condition="Ready",status="true"})
  /
  count(kube_node_info)
) * 100

# Nodes with Pressure Conditions
sum(kube_node_status_condition{
  condition=~"MemoryPressure|DiskPressure|PIDPressure",
  status="true"
}) by (node, condition)
```

**Alert Definitions**:

```promql
# CRITICAL: Node Not Ready
- alert: NodeNotReady
  expr: |
    kube_node_status_condition{condition="Ready",status!="true"} == 1
  for: 5m
  labels:
    severity: critical
    component: node
  annotations:
    summary: "Node {{ $labels.node }} is NotReady"
    incident_severity: "P1"

# CRITICAL: Node Memory Pressure
- alert: NodeMemoryPressure
  expr: |
    kube_node_status_condition{condition="MemoryPressure",status="true"} == 1
  for: 5m
  labels:
    severity: critical
    component: node
  annotations:
    summary: "Node {{ $labels.node }} under memory pressure"
    incident_severity: "P1"
```

---

### 1.5.2 Kubelet Performance

**SLO Target**:
- P95 PLEG relist latency < 1s
- Kubelet runtime operations success: 99.9%

**PromQL Queries**:

```promql
# PLEG Relist Latency P95
histogram_quantile(0.95,
  sum(rate(kubelet_pleg_relist_duration_seconds_bucket[5m])) by (le, node)
)

# Kubelet Runtime Success Ratio
1 -
(
  sum(rate(kubelet_runtime_operations_total{status!="success"}[5m]))
  /
  sum(rate(kubelet_runtime_operations_total[5m]))
)
```

---

# Part 2: OPTIONAL SLOs (Tier 2 - Operational Excellence)

## Key Principle

OPTIONAL SLOs inform optimization efforts, not operational incidents. These don't trigger pages but inform:
- Capacity planning discussions
- Resource optimization efforts
- Long-term platform improvements
- Developer productivity initiatives

---

## 2.1 Cluster Autoscaler Performance

**SLO Target**:
- Node scale-up time P95 < 5 minutes
- Scale-up success rate: 95%

**Business Value**:
- Cost optimization (20-30% savings)
- Handle month-end processing spikes
- Elasticity during off-hours

**Why OPTIONAL**:
- Not customer-facing directly
- Manual intervention possible
- Can tolerate occasional failures

**PromQL Queries**:

```promql
# Unschedulable Pods Count
cluster_autoscaler_unschedulable_pods_count

# Scale-Up Duration P95
histogram_quantile(0.95,
  sum(rate(cluster_autoscaler_scale_up_duration_seconds_bucket[30m])) by (le)
)

# Cluster CPU Utilization
(
  sum(kube_pod_container_resource_requests{resource="cpu"})
  /
  sum(kube_node_status_allocatable{resource="cpu"})
) * 100
```

**Alert Definitions** (Non-Paging):

```promql
# INFO: Autoscaler Not Safe to Scale
- alert: ClusterAutoscalerNotSafe
  expr: |
    cluster_autoscaler_cluster_safe_to_autoscale == 0
  for: 15m
  labels:
    severity: info
    tier: optional
  annotations:
    summary: "Cluster autoscaler reports cluster not safe to scale"
    no_page: "true"

# WARNING: High Scale-Up Failure Rate
- alert: ClusterAutoscalerHighScaleUpFailures
  expr: |
    (
      sum(rate(cluster_autoscaler_failed_scale_ups_total[30m]))
      /
      sum(rate(cluster_autoscaler_scaled_up_nodes_total[30m] + cluster_autoscaler_failed_scale_ups_total[30m]))
    ) > 0.10
  for: 20m
  labels:
    severity: warning
    tier: optional
  annotations:
    summary: "Cluster autoscaler scale-up failure rate at {{ $value | humanizePercentage }}"
    no_page: "true"
```

---

## 2.2 Horizontal Pod Autoscaler (HPA) Performance

**SLO Target**:
- HPA decision latency P95 < 30 seconds
- Metric availability for HPA: 99%

**Why OPTIONAL**:
- Applications designed to handle slight over/under-provisioning
- Manual scaling still possible

**PromQL Queries**:

```promql
# HPAs at Max Replicas
count(
  kube_horizontalpodautoscaler_status_current_replicas
  ==
  kube_horizontalpodautoscaler_spec_max_replicas
) by (namespace)

# HPA Flapping Detection
count_over_time(
  (abs(delta(kube_horizontalpodautoscaler_status_desired_replicas[5m])) > 0)[30m:5m]
) > 3
```

---

## 2.3 Persistent Volume Performance

**SLO Target**:
- PV provisioning time P95 < 60 seconds
- Volume usage < 85%

**PromQL Queries**:

```promql
# Volume Usage Percentage
(
  kubelet_volume_stats_used_bytes
  /
  kubelet_volume_stats_capacity_bytes
) * 100

# Storage Waste Percentage
(
  sum(kube_persistentvolumeclaim_resource_requests_storage_bytes - kubelet_volume_stats_used_bytes)
  /
  sum(kube_persistentvolumeclaim_resource_requests_storage_bytes)
) * 100
```

---

## 2.4 Image Pull Performance

**SLO Target**:
- Image pull success rate: 99%
- Image pull time P95 < 60 seconds

**PromQL Queries**:

```promql
# Pods with Image Pull Errors
count(kube_pod_container_status_waiting_reason{
  reason=~"ImagePullBackOff|ErrImagePull"
})
```

---

## 2.5 Namespace Resource Quota Usage

**SLO Target**:
- Namespace quota usage < 85%
- Zero hard quota rejections for production

**PromQL Queries**:

```promql
# CPU Quota Usage Percentage
(
  kube_resourcequota{resource="requests.cpu", type="used"}
  /
  kube_resourcequota{resource="requests.cpu", type="hard"}
) * 100

# Memory Quota Usage Percentage
(
  kube_resourcequota{resource="requests.memory", type="used"}
  /
  kube_resourcequota{resource="requests.memory", type="hard"}
) * 100
```

---

# Part 3: Implementation Guide

## 3.1 GCP Managed Prometheus Setup

### Recording Rules for Efficient SLO Calculations

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-recording-rules
  namespace: gmp-system
data:
  rules.yaml: |
    groups:
    - name: sli_recordings
      interval: 30s
      rules:
      # API Server SLI
      - record: apiserver:availability:4w
        expr: |
          sum(rate(apiserver_request_total{code=~"2.."}[4w]))
          /
          sum(rate(apiserver_request_total[4w]))

      - record: apiserver:latency:p95
        expr: |
          histogram_quantile(0.95,
            sum(rate(apiserver_request_duration_seconds_bucket{verb!~"WATCH|CONNECT"}[5m]))
            by (le)
          )

      # Node SLI
      - record: node:availability
        expr: |
          sum(kube_node_status_condition{condition="Ready",status="true"})
          /
          count(kube_node_info)
```

---

## 3.2 Alert Configuration

### GMP Rules Resource (Complete Example)

```yaml
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: platform-sre-alerts
  namespace: gmp-system
spec:
  groups:
  - name: gke-control-plane
    interval: 30s
    rules:
    - alert: GKEAPIServerErrorRateHigh
      expr: |
        (
          sum(rate(apiserver_request_total{cluster="CLUSTER", code=~"5..|429"}[5m]))
          /
          sum(rate(apiserver_request_total{cluster="CLUSTER"}[5m]))
        ) > 0.01
      for: 10m
      labels:
        severity: page
      annotations:
        summary: "Kubernetes API server errors high"

    - alert: GKEAPIServerP99LatencyHigh
      expr: |
        histogram_quantile(
          0.99,
          sum(rate(apiserver_request_duration_seconds_bucket{cluster="CLUSTER"}[5m])) by (le)
        ) > 1
      for: 10m
      labels:
        severity: page
      annotations:
        summary: "Kubernetes API server p99 latency high"

  - name: gke-scheduler
    interval: 30s
    rules:
    - alert: GKESchedulerUnschedulableBacklog
      expr: |
        sum(scheduler_pending_pods{cluster="CLUSTER", queue="unschedulable"}) > 0
      for: 15m
      labels:
        severity: page
      annotations:
        summary: "Unschedulable pod backlog"

  - name: platform-workloads
    interval: 30s
    rules:
    - alert: PlatformDeploymentUnavailable
      expr: |
        kube_deployment_status_replicas_available{cluster="CLUSTER", namespace="PLATFORM_NS"}
        <
        kube_deployment_spec_replicas{cluster="CLUSTER", namespace="PLATFORM_NS"}
      for: 10m
      labels:
        severity: page
      annotations:
        summary: "Platform deployment unavailable"

    - alert: PlatformPodsCrashLooping
      expr: |
        sum(
          kube_pod_container_status_waiting_reason{
            cluster="CLUSTER",
            namespace="PLATFORM_NS",
            reason="CrashLoopBackOff"
          } == 1
        ) > 0
      for: 10m
      labels:
        severity: ticket
      annotations:
        summary: "Platform pods crashlooping"
```

---

## 3.3 Dashboard Integration

### Recommended Dashboards

#### Dashboard 1: SLO Compliance Overview
- API Server availability gauge
- Error budget burn rate
- DNS success rate
- Platform component health
- **Audience**: Leadership, SRE
- **Update**: Real-time

#### Dashboard 2: Control Plane Health
- API server request rate & latency
- etcd performance metrics
- Scheduler throughput
- Admission webhook latency
- **Audience**: SRE, Platform team
- **Update**: Real-time

#### Dashboard 3: Workload Health
- Pod availability by namespace
- Deployment rollout status
- Container restart rates
- Application-level SLIs
- **Audience**: App teams, SRE
- **Update**: Real-time

#### Dashboard 4: Capacity Planning (OPTIONAL SLOs)
- Cluster utilization trends
- Resource waste metrics
- Autoscaler performance
- Storage utilization
- **Audience**: SRE, Finance, Leadership
- **Update**: Weekly review

---

## Summary Tables

### MUST-HAVE SLOs Priority Matrix

| Component | SLO Target | Business Impact | Page Threshold |
|-----------|-----------|----------------|----------------|
| **API Server Availability** | 99.95% | Platform outage | Fast burn: 14.4x |
| **API Server Latency** | P95 < 1s | Transaction delays | P99 > 5s |
| **etcd Health** | 99.99% | Data integrity | Leader changes > 3 |
| **DNS Success Rate** | 99.95% | Service discovery | Fast burn: 14.4x |
| **DNS Latency** | P95 < 50ms | Transaction latency | P99 > 100ms |
| **ArgoCD Availability** | 99.9% | Deployment blocked | API down > 2m |
| **Node Availability** | 99.9% | Workload evictions | NotReady > 5m |
| **Customer Services** | 99.9% | Revenue impact | < 99.9% for 2m |

### OPTIONAL SLOs Priority Matrix

| Component | Business Value | Alert Threshold | Why Non-Paging |
|-----------|---------------|-----------------|----------------|
| **Cluster Autoscaler** | Cost optimization | Scale failures >10% | Manual scaling possible |
| **HPA** | Auto-scaling | Flapping detected | Apps handle static capacity |
| **PV/PVC** | Storage capacity | Volume >85% full | Gradual degradation |
| **Image Pull** | Deployment velocity | >5 pods failing | Retries succeed |
| **Resource Quotas** | Cost control | Quota usage >85% | Detected early in dev/staging |

---

## Best Practices for Banking Infrastructure

### Alerting Philosophy

**Page on SLO burn**, ticket on slower signals:

✅ **Page for**:
- Fast error budget burn (14.4x rate)
- Slow error budget burn (6x rate)
- P0/P1 component failures (API server, etcd, DNS, ArgoCD)
- Customer-facing service degradation

❌ **Don't page for**:
- OPTIONAL SLO violations
- Gradual capacity degradation
- Cost optimization opportunities
- Developer experience issues

### Compliance Considerations

**Regulatory Requirements**:
- **SOX**: ArgoCD availability = deployment audit trail
- **PCI-DSS**: Network segmentation monitoring, secret management
- **GDPR**: Data locality, access logging
- **BCBS 239**: Data quality, aggregation capability

**Incident Documentation**:
- All P0/P1 incidents require post-mortem
- Compliance team notification for:
  - Extended ArgoCD outages
  - Data plane security issues
  - Audit log gaps

---

## Rationale for This Framework

### Why Multi-Window Multi-Burn-Rate Alerting?

Burn-rate alerting answers: *"Are we going to violate the SLO if this continues?"*

- **Fast burn (1h/5m windows)**: Catches sudden incidents requiring immediate response
- **Slow burn (6h/30m windows)**: Catches gradual degradation before critical
- **Two windows per burn rate**: Avoids false positives from transient spikes
- **Banking context**: Regulatory requirements demand quick incident response

### Why Separate MUST-HAVE vs OPTIONAL?

**MUST-HAVE SLOs**:
- Direct customer impact
- Platform availability
- Security/compliance requirements
- Merit immediate on-call response

**OPTIONAL SLOs**:
- Operational efficiency
- Cost optimization
- Developer experience
- Weekly/monthly review cycles

### Why Platform Components in Tier 1?

Platform components (ArgoCD, NPD, exporters) aren't "nice-to-have." They are the cluster's **operating system**:
- ArgoCD failure = cannot deploy (compliance violation)
- NPD failure = blind to node issues (extended MTTR)
- Exporter failure = blind to workload health (risk accumulation)

---

## Implementation Roadmap

### Phase 1: Foundation (Weeks 1-2)
1. Deploy GMP managed collection
2. Implement API Server SLOs
3. Implement DNS SLOs
4. Configure burn-rate alerting

### Phase 2: Platform Stability (Weeks 3-4)
1. Implement etcd monitoring
2. Deploy Node Problem Detector
3. Implement workload availability SLOs
4. Configure platform component alerts

### Phase 3: GitOps & Compliance (Weeks 5-6)
1. Implement ArgoCD SLOs
2. Configure drift detection alerts
3. Implement audit trail monitoring
4. Document compliance mapping

### Phase 4: Optimization (Weeks 7-8)
1. Implement OPTIONAL SLOs
2. Configure capacity planning dashboards
3. Establish weekly review cadence
4. Begin cost optimization initiatives

---

## Conclusion

This framework provides comprehensive observability for GKE banking infrastructure while maintaining focus on what matters most:

1. **Customer experience** (service availability, latency)
2. **Platform reliability** (control plane, DNS, nodes)
3. **Security & compliance** (ArgoCD, drift detection, audit)
4. **Operational efficiency** (autoscaling, capacity, cost)

By separating MUST-HAVE from OPTIONAL SLOs, teams can focus on preventing customer-impacting incidents while continuously improving operational excellence through data-driven capacity planning and cost optimization.
