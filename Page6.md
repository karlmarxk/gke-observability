Below is a **platform-focused SLO + alerting blueprint for GKE** using metrics you already have in **Google Cloud Managed Service for Prometheus** (GMP). It’s written for a regulated environment (bank): **few SLOs that matter, tight paging, everything else tickets**.

Key implementation realities in Google Kubernetes Engine / Google Cloud:

* GKE control plane / kube-state / cAdvisor+kubelet metric packages write into Cloud Monitoring as Prometheus metrics using `prometheus_target` and metric names prefixed `prometheus.googleapis.com/` (Cloud Monitoring name), while **PromQL uses upstream metric names** (e.g., `apiserver_request_total`). ([Google Cloud Documentation][1])
* `apiserver_request_total` and `apiserver_request_duration_seconds` are **very high-cardinality**; you must filter/group intentionally or you will burn quota/cost and drown in noise. ([Google Cloud Documentation][1])
* cAdvisor/Kubelet curated metrics are explicitly a **subset** chosen to reduce ingestion volume/cost; and require specific GKE versions. ([Google Cloud Documentation][2])
* For GKE-based services in Cloud Monitoring, you typically create SLOs using **“Other” custom SLI** (time-series ratio / distribution cut); request-based vs windows-based evaluation are supported. ([Google Cloud Documentation][3])

---

## 1) Must-have SLOs (platform)

Each SLO below includes: **what to measure**, **how to measure (PromQL)**, and **why it’s must-have**. Targets are sane defaults—tune after 2–4 weeks of data.

### A. Control Plane (API server)

#### SLO A1 — API server “yield” (availability via success ratio)

**Target:** 99.95% over 30d (platform-level)
**SLI (PromQL):**

```promql
sum(rate(apiserver_request_total{cluster="CLUSTER", code=~"2..|3.."}[5m]))
/
sum(rate(apiserver_request_total{cluster="CLUSTER"}[5m]))
```

**Rationale:** The API server is the control plane “front door.” When yield drops, everything degrades (deployments, controllers, GitOps, admission, autoscalers). Also directly measurable and actionable. GKE positions API server metrics around latency/traffic/error/saturation signals. ([Google Cloud Documentation][1])

---

#### SLO A2 — API server latency compliance (percent of requests under threshold)

**Target:** 99% under 1s for single-resource calls (GET/POST/PATCH), 30s for LIST (start with 1s/30s split)
**SLI (example for 1s):**

```promql
sum(rate(apiserver_request_duration_seconds_bucket{cluster="CLUSTER", le="1"}[5m]))
/
sum(rate(apiserver_request_duration_seconds_count{cluster="CLUSTER"}[5m]))
```

**Rationale:** Control plane overload shows up first in latency. Kubernetes community SLO expectations are explicitly called out (e.g., 1s for single-resource calls). Also, request latency includes authn/z, admission webhooks, and datastore operations—so it’s a strong “platform health” proxy. ([Google Cloud Documentation][1])

> **Operational constraint:** this metric is *very high-cardinality*; filter by `verb`, `resource`, etc. ([Google Cloud Documentation][1])

---

#### SLO A3 — Admission webhook tail latency (security/compliance critical)

**Target:** p99 < 5s per webhook (or a % under 5s SLI); keep margin under ~10s timeout
**SLI (p99):**

```promql
histogram_quantile(
  0.99,
  sum(rate(apiserver_admission_webhook_admission_duration_seconds_bucket{cluster="CLUSTER"}[5m])) by (le, name)
)
```

**Rationale:** In a bank, admission controls (policy, security, image signing, etc.) are non-negotiable. Slow/unresponsive webhooks can **block deployments and scheduling**. GKE explicitly notes external webhooks have ~10s timeout and recommends alerting as you approach it. ([Google Cloud Documentation][1])

---

### B. Scheduler (workload placement)

#### SLO B1 — Pod scheduling latency (end-to-end)

**Target:** p99 < 5s (platform)
**SLI (newer clusters):**

```promql
histogram_quantile(
  0.99,
  sum(rate(scheduler_pod_scheduling_sli_duration_seconds_bucket{cluster="CLUSTER"}[5m])) by (le)
)
```

**Fallback (older clusters, deprecated metric):**

```promql
histogram_quantile(
  0.99,
  sum(rate(scheduler_pod_scheduling_duration_seconds_bucket{cluster="CLUSTER"}[5m])) by (le)
)
```

**Rationale:** If scheduling is slow, incident impact is immediate (rollouts stall, node drains hurt more, autoscaling appears “broken”). Note GKE: `scheduler_pod_scheduling_duration_seconds` is deprecated/removed and replaced by `scheduler_pod_scheduling_sli_duration_seconds` for 1.30+. ([Google Cloud Documentation][1])

---

#### SLO B2 — Unschedulable backlog (good-window SLO)

**Target:** 99.9% of 5-minute windows with **no unschedulable pods** (platform namespaces)
**SLI helper (good-window fraction):**

```promql
avg_over_time(
  (
    sum(scheduler_pending_pods{cluster="CLUSTER", queue="unschedulable"}) == 0
  )[30d:5m]
)
```

**Rationale:** Backlogs indicate real capacity/constraints problems (resources, affinity/anti-affinity, taints/tolerations, PVC binding). GKE documentation explains the scheduler queues and why unschedulable backlog is a key signal. ([Google Cloud Documentation][1])

---

### C. Workload Management (platform components: security, Argo CD, detectors, exporters)

For platform services, focus on **availability and rollout health**. These are the things that keep the cluster governable and observable.

#### SLO C1 — Deployment availability (ready replicas == desired) for critical namespaces

**Target:** 99.9% of 5-minute windows “healthy” (per deployment)
**SLI helper:**

```promql
avg_over_time(
  (
    kube_deployment_status_replicas_available{cluster="CLUSTER", namespace="PLATFORM_NS", deployment="DEPLOY"}
    >=
    kube_deployment_spec_replicas{cluster="CLUSTER", namespace="PLATFORM_NS", deployment="DEPLOY"}
  )[30d:5m]
)
```

**Rationale:** This is your **platform availability** SLO (Argo CD controllers, policy controllers, exporters, etc.). GKE kube-state metrics explicitly provide these deployment replica metrics and example queries. ([Google Cloud Documentation][4])

---

#### SLO C2 — DaemonSet coverage (ready == desired) for node-level agents

**Target:** 99.9% of windows with `ready == desired`
**SLI helper:**

```promql
avg_over_time(
  (
    kube_daemonset_status_number_ready{cluster="CLUSTER", namespace="PLATFORM_NS", daemonset="DS"}
    ==
    kube_daemonset_status_desired_number_scheduled{cluster="CLUSTER", namespace="PLATFORM_NS", daemonset="DS"}
  )[30d:5m]
)
```

**Rationale:** Node Problem Detector, node exporters, CNI agents, security sensors—these fail “silently” unless you enforce coverage. GKE provides these DaemonSet metrics and alert patterns. ([Google Cloud Documentation][4])

---

#### SLO C3 — Platform pods not stuck in bad waiting states

**Target:** 99.9% windows with **0 pods** in `CrashLoopBackOff`, `ImagePullBackOff`, `ContainerCreating` in critical namespaces
**SLI helper (example for CrashLoopBackOff):**

```promql
avg_over_time(
  (
    sum by (namespace) (
      kube_pod_container_status_waiting_reason{
        cluster="CLUSTER",
        namespace="PLATFORM_NS",
        reason="CrashLoopBackOff"
      } == 1
    ) == 0
  )[30d:5m]
)
```

**Rationale:** For platform components, “not Ready” is often preceded by these waiting reasons; GKE calls out `kube_pod_container_status_waiting_reason` as a key metric for crashlooping and other stuck states. ([Google Cloud Documentation][4])

---

### D. Node / Kubelet / Storage health

#### SLO D1 — Kubelet runtime operations success ratio

**Target:** 99.9% (5m rolling), 99.99% (30d)
**SLI:**

```promql
1 -
(
  sum(rate(kubelet_runtime_operations_total{cluster="CLUSTER", status!="success"}[5m]))
  /
  sum(rate(kubelet_runtime_operations_total{cluster="CLUSTER"}[5m]))
)
```

**Rationale:** Kubelet runtime failures (create/start/stop) directly translate into pod instability. These kubelet metrics are part of the curated package. ([Google Cloud Documentation][2])

---

#### SLO D2 — Persistent volume capacity headroom (percent used)

**Target:** 99.9% windows where **used < 85%**
**SLI helper:**

```promql
avg_over_time(
  (
    (
      kubelet_volume_stats_used_bytes{cluster="CLUSTER"}
      /
      kubelet_volume_stats_capacity_bytes{cluster="CLUSTER"}
    ) < 0.85
  )[30d:5m]
)
```

**Rationale:** Volume-full events create high-severity incidents fast (databases, queues, controllers). Curated kubelet volume stats are explicitly available. ([Google Cloud Documentation][2])

---

### E. Observability pipeline itself

#### SLO E1 — Critical scrape target availability (`up`)

**Target:** 99.9% scrape success for critical jobs (kube-state-metrics, collectors, Argo CD metrics, security controllers, etc.)
**SLI (per job):**

```promql
avg_over_time(up{cluster="CLUSTER", job=~"kube-state-metrics|gmp-collector|argocd.*|gatekeeper.*"}[30d])
```

**Rationale:** If you can’t scrape, you can’t see. In regulated environments, “observability blindness” is an operational risk. Use `up` as your simplest, most dependable observability SLI.

---

## 2) Optional SLOs (good to have once must-haves are stable)

These are valuable, but don’t start here.

### O1 — API Priority & Fairness rejections (429 / APF rejected)

**SLI (429 fraction):**

```promql
sum(rate(apiserver_request_total{cluster="CLUSTER", code="429"}[5m]))
/
sum(rate(apiserver_request_total{cluster="CLUSTER"}[5m]))
```

**Rationale:** 429s mean the control plane is throttling. Useful for large org / heavy controllers / GitOps storms. GKE provides query examples for 429. ([Google Cloud Documentation][1])

### O2 — API server saturation (inflight)

**Indicator:**

```promql
max_over_time(apiserver_current_inflight_requests{cluster="CLUSTER"}[5m])
```

**Rationale:** Helps confirm overload, but thresholds depend on APF configuration (so not a clean SLO unless you standardize APF). ([Google Cloud Documentation][1])

### O3 — CPU throttling ratio (limits too tight / noisy neighbor risk)

**SLI:**

```promql
sum(rate(container_cpu_cfs_throttled_periods_total{cluster="CLUSTER"}[5m]))
/
sum(rate(container_cpu_cfs_periods_total{cluster="CLUSTER"}[5m]))
```

**Rationale:** Strong predictor of tail latency and unpredictable platform behavior. cAdvisor curated metrics include these. ([Google Cloud Documentation][2])

### O4 — Network packet drop ratio

```promql
sum(rate(container_network_transmit_packets_dropped_total{cluster="CLUSTER"}[5m]))
/
sum(rate(container_network_transmit_packets_total{cluster="CLUSTER"}[5m]))
```

**Rationale:** Detects node/kernel/CNI issues early. cAdvisor curated metrics include drops + totals. ([Google Cloud Documentation][2])

### O5 — Evictions rate (node controller)

```promql
sum(rate(node_collector_evictions_total{cluster="CLUSTER"}[5m]))
```

**Rationale:** A disruption SLO for “platform stability,” but evictions can be workload-driven; usually ticket-level unless it spikes. ([Google Cloud Documentation][1])

---

## 3) Alerts to monitor platform components (pager vs ticket)

Best practice: **page on SLO burn**, ticket on slower signals. Google’s SRE guidance is to turn SLOs into actionable alerts (burn-rate style). ([sre.google][5])

### 3.1 Paging alerts (high confidence, customer-impacting soon)

#### P1 — API server SLO burn-rate (Cloud Monitoring SLO module)

* Create the SLO in Cloud Monitoring (“Other” custom SLI for GKE-based services). ([Google Cloud Documentation][3])
* Alert on error budget burn using `select_slo_burn_rate`. ([Google Cloud Documentation][6])

**Condition idea (fast burn):** burn rate > 14 for 5m (tune)
**Condition idea (slow burn):** burn rate > 2 for 1h (tune)

> Why burn rate: it pages when you are on track to violate the SLO, not when a random metric twitches. ([Google Cloud Documentation][6])

---

#### P2 — API server latency p99 breach (symptom-based backstop)

```promql
histogram_quantile(
  0.99,
  sum(rate(apiserver_request_duration_seconds_bucket{cluster="CLUSTER"}[5m])) by (le)
) > 1
```

**for:** 10m
**Rationale:** Latency spikes can precede outright failures; high p99 is actionable and typically correlates with platform incidents. Also explicitly recommended to watch. ([Google Cloud Documentation][1])

---

#### P3 — Admission webhook near-timeout (security control degradation)

```promql
histogram_quantile(
  0.99,
  sum(rate(apiserver_admission_webhook_admission_duration_seconds_bucket{cluster="CLUSTER"}[5m])) by (le, name)
) > 8
```

**for:** 10m
**Rationale:** Webhook timeouts (~10s) can block scheduling/deployments. In a bank, this can also create emergency-change pressure (bad). ([Google Cloud Documentation][1])

---

#### P4 — Scheduler unschedulable backlog sustained

```promql
sum(scheduler_pending_pods{cluster="CLUSTER", queue="unschedulable"}) > 0
```

**for:** 15m
**Rationale:** Sustained unschedulable backlog means the cluster cannot place pods—this is platform availability degradation. GKE documents unschedulable queue meaning and causes. ([Google Cloud Documentation][1])

---

#### P5 — Critical platform component down (Argo CD / policy controller / exporters)

**Pattern (Deployment not available):**

```promql
kube_deployment_status_replicas_available{cluster="CLUSTER", namespace="PLATFORM_NS", deployment=~"argocd.*|gatekeeper.*|security.*"}
<
kube_deployment_spec_replicas{cluster="CLUSTER", namespace="PLATFORM_NS", deployment=~"argocd.*|gatekeeper.*|security.*"}
```

**for:** 10m
**Rationale:** If GitOps/policy/telemetry components are down, the cluster becomes unsafe or unmanageable. Deployment replica metrics are provided by GKE kube-state metrics. ([Google Cloud Documentation][4])

---

### 3.2 Ticket alerts (important, but not wake-you-up)

#### T1 — Stalled rollout (Deployment / StatefulSet / DaemonSet)

GKE provides canonical “stalled deployment” logic: ([Google Cloud Documentation][4])

```promql
(
  kube_deployment_spec_replicas{cluster="CLUSTER", namespace="NS", deployment="DEPLOY"}
    >
  kube_deployment_status_replicas_available{cluster="CLUSTER", namespace="NS", deployment="DEPLOY"}
)
and
(
  changes(kube_deployment_status_replicas_updated{cluster="CLUSTER", namespace="NS", deployment="DEPLOY"}[10m]) == 0
)
```

**Rationale:** This catches stuck upgrades early without paging for transient rollout delays.

---

#### T2 — Crashlooping / ImagePullBackOff / ContainerCreating in platform namespaces

```promql
sum by (namespace, reason) (
  kube_pod_container_status_waiting_reason{cluster="CLUSTER", namespace="PLATFORM_NS"} == 1
) > 0
```

**Rationale:** This is the single best “platform pods are unhappy” detector; GKE explicitly recommends it for crashlooping and waiting-state diagnosis. ([Google Cloud Documentation][4])

---

#### T3 — Kubelet runtime ops error ratio elevated

```promql
(
  sum(rate(kubelet_runtime_operations_total{cluster="CLUSTER", status!="success"}[5m]))
  /
  sum(rate(kubelet_runtime_operations_total{cluster="CLUSTER"}[5m]))
) > 0.01
```

**Rationale:** Often indicates container runtime instability; good early warning. Metrics exist in curated kubelet set. ([Google Cloud Documentation][2])

---

#### T4 — Volume usage high (capacity or inodes)

**Capacity:**

```promql
(
  kubelet_volume_stats_used_bytes{cluster="CLUSTER"}
  /
  kubelet_volume_stats_capacity_bytes{cluster="CLUSTER"}
) > 0.85
```

**Inodes:**

```promql
(
  kubelet_volume_stats_inodes_used{cluster="CLUSTER"}
  /
  kubelet_volume_stats_inodes{cluster="CLUSTER"}
) > 0.85
```

**Rationale:** Avoid preventable “disk full” outages. Metrics are available in curated kubelet set. ([Google Cloud Documentation][2])

---

#### T5 — CPU throttling ratio high (performance risk)

```promql
sum(rate(container_cpu_cfs_throttled_periods_total{cluster="CLUSTER"}[5m]))
/
sum(rate(container_cpu_cfs_periods_total{cluster="CLUSTER"}[5m]))
> 0.2
```

**Rationale:** This creates tail latency and intermittent failures. cAdvisor curated metrics include throttling counters. ([Google Cloud Documentation][2])

---

## 4) Alerting definitions (recommended implementation on GKE)

You have two “native GCP” choices:

1. **Cloud Monitoring SLO burn alerts** using `select_slo_burn_rate` (best for paging). ([Google Cloud Documentation][6])
2. **Prometheus-compatible alerts** using GMP managed rule evaluation (`Rules` CRD), which is deployed as part of managed collection. ([Google Cloud Documentation][7])

### Example: GMP `Rules` resource (starter pack)

This is a single, deployable ruleset. Trim/extend for your namespaces and naming.

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
        description: "5xx/429 ratio > 1% for 10m on cluster=CLUSTER."

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
        description: "p99 > 1s for 10m on cluster=CLUSTER."

    - alert: GKEAdmissionWebhookNearTimeout
      expr: |
        histogram_quantile(
          0.99,
          sum(rate(apiserver_admission_webhook_admission_duration_seconds_bucket{cluster="CLUSTER"}[5m])) by (le, name)
        ) > 8
      for: 10m
      labels:
        severity: page
      annotations:
        summary: "Admission webhook p99 near timeout"
        description: "Webhook p99 > 8s for 10m. Grouped by webhook name."

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
        description: "unschedulable queue > 0 for 15m."

    - alert: GKESchedulerP99SchedulingLatencyHigh
      expr: |
        histogram_quantile(
          0.99,
          sum(rate(scheduler_pod_scheduling_sli_duration_seconds_bucket{cluster="CLUSTER"}[5m])) by (le)
        ) > 5
      for: 15m
      labels:
        severity: ticket
      annotations:
        summary: "Scheduling p99 latency high"
        description: "p99 scheduling latency > 5s for 15m."

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
        description: "available replicas < desired for 10m in PLATFORM_NS."

    - alert: PlatformDeploymentStalledRollout
      expr: |
        (
          kube_deployment_spec_replicas{cluster="CLUSTER", namespace="PLATFORM_NS"}
            >
          kube_deployment_status_replicas_available{cluster="CLUSTER", namespace="PLATFORM_NS"}
        ) and (
          changes(kube_deployment_status_replicas_updated{cluster="CLUSTER", namespace="PLATFORM_NS"}[10m]) == 0
        )
      for: 10m
      labels:
        severity: ticket
      annotations:
        summary: "Platform deployment rollout stalled"
        description: "Deployment not progressing for 10m in PLATFORM_NS."

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
        description: "CrashLoopBackOff > 0 for 10m in PLATFORM_NS."

  - name: kubelet-and-storage
    interval: 30s
    rules:
    - alert: KubeletRuntimeErrorsHigh
      expr: |
        (
          sum(rate(kubelet_runtime_operations_total{cluster="CLUSTER", status!="success"}[5m]))
          /
          sum(rate(kubelet_runtime_operations_total{cluster="CLUSTER"}[5m]))
        ) > 0.01
      for: 15m
      labels:
        severity: ticket
      annotations:
        summary: "Kubelet runtime errors elevated"
        description: "Non-success runtime ops > 1% for 15m."

    - alert: PersistentVolumeUsageHigh
      expr: |
        (
          kubelet_volume_stats_used_bytes{cluster="CLUSTER"}
          /
          kubelet_volume_stats_capacity_bytes{cluster="CLUSTER"}
        ) > 0.85
      for: 30m
      labels:
        severity: ticket
      annotations:
        summary: "Persistent volume usage high"
        description: "PVC usage > 85% for 30m."
```

Why this approach works on GKE:

* GMP supports Prometheus-compatible rule evaluation and alerting and uses a Kubernetes `Rules` resource; the rule evaluator is deployed with managed collection. ([Google Cloud Documentation][7])
* You still use Cloud Monitoring for **SLO burn paging**, which is the most SRE-aligned way to page. ([Google Cloud Documentation][6])

---

## 5) Detailed rationale: why these SLOs and alerts (bank-grade)

### Why “must-have” is intentionally small

SRE best practice is to avoid metric sprawl: **SLOs are the contract**, not a dashboard wishlist. Too many SLOs = diluted ownership + alert fatigue. The must-have list covers:

* **Control plane reliability** (API yield + latency)
* **Policy and governance continuity** (admission webhook health)
* **Scheduling ability** (latency + backlog)
* **Platform service availability** (deployments/daemonsets + stuck pods)
* **Node/storage failure modes** that routinely cause severe incidents
* **Observability continuity** (scrape health)

### Why burn-rate alerting is your paging backbone

Burn-rate paging answers: *“Are we going to violate the SLO if this continues?”* That is exactly the kind of page that stays meaningful at scale. Cloud Monitoring explicitly supports burn-rate alerting with `select_slo_burn_rate`. ([Google Cloud Documentation][6])

### Why you must manage cardinality and ingestion cost up front

GKE docs explicitly warn that key API server metrics are very high-cardinality and must be filtered/grouped. If you ignore this in a multinational bank (many clusters, many tenants), you’ll hit ingestion quotas and cost, and your PromQL becomes unusable. ([Google Cloud Documentation][1])

### Why platform components get availability SLOs

Argo CD, security controllers, exporters, node-level agents aren’t “nice-to-have.” They are the cluster’s **operating system**. Their failure is operationally equivalent to losing the control plane’s ability to deploy, enforce policy, or see.

---
