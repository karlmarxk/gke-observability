# Part 4: OPTIONAL SLOs (Tier 2 - Operational Excellence)

## Executive Context for Banking

OPTIONAL SLOs are **not customer-impacting** but provide:
1. **Operational Efficiency**: Reduce toil, improve mean-time-to-resolution
2. **Cost Optimization**: Identify waste, right-size resources
3. **Proactive Capacity Planning**: Prevent issues before they become incidents
4. **Developer Experience**: Improve velocity without compromising safety

**Key Principle**: These SLOs don't trigger pages but inform:
- Capacity planning discussions
- Resource optimization efforts  
- Long-term platform improvements
- Developer productivity initiatives

---

## 3.1 Cluster Autoscaler Performance (OPTIONAL)

**SLO Definition**:
- Node scale-up time P95 < 5 minutes
- Node scale-down time P95 < 10 minutes
- Scale-up success rate: 95%
- Unschedulable pod resolution time P95 < 5 minutes

**Business Value**:
- **Cost Optimization**: Right-sizing cluster capacity saves ~20-30% on compute costs
- **Burst Capacity**: Handle month-end processing spikes without over-provisioning
- **Elasticity**: Scale down during nights/weekends (banking hours-specific)

**Why OPTIONAL**:
- Not customer-facing directly
- Manual intervention possible during scale failures
- SLO violations = operational inefficiency, not customer impact
- Can tolerate occasional failures with manual remediation

---

### PromQL Queries for Cluster Autoscaler

```promql
# ============================================
# AUTOSCALER PERFORMANCE METRICS
# ============================================

# Autoscaler Status (1 = healthy, 0 = unhealthy)
cluster_autoscaler_cluster_safe_to_autoscale

# Unschedulable Pods Count (triggers scale-up)
cluster_autoscaler_unschedulable_pods_count

# Nodes Currently Scaling Up
cluster_autoscaler_nodes_count{state="scaleUp"}

# Nodes Currently Scaling Down
cluster_autoscaler_nodes_count{state="scaleDown"}

# Scale-Up Events (successful)
sum(increase(cluster_autoscaler_scaled_up_nodes_total[1h]))

# Scale-Down Events (successful)
sum(increase(cluster_autoscaler_scaled_down_nodes_total[1h]))

# Failed Scale-Ups
sum(increase(cluster_autoscaler_failed_scale_ups_total[1h]))

# Scale-Up Duration P50
histogram_quantile(0.50,
  sum(rate(cluster_autoscaler_scale_up_duration_seconds_bucket[30m])) by (le)
)

# Scale-Up Duration P95
histogram_quantile(0.95,
  sum(rate(cluster_autoscaler_scale_up_duration_seconds_bucket[30m])) by (le)
)

# Scale-Up Duration P99
histogram_quantile(0.99,
  sum(rate(cluster_autoscaler_scale_up_duration_seconds_bucket[30m])) by (le)
)

# ============================================
# CAPACITY AND UTILIZATION
# ============================================

# Cluster CPU Capacity
sum(kube_node_status_allocatable{resource="cpu"})

# Cluster CPU Requests (how much allocated)
sum(kube_pod_container_resource_requests{resource="cpu"})

# Cluster CPU Utilization (allocated vs capacity)
(
  sum(kube_pod_container_resource_requests{resource="cpu"})
  /
  sum(kube_node_status_allocatable{resource="cpu"})
) * 100

# Cluster Memory Capacity
sum(kube_node_status_allocatable{resource="memory"})

# Cluster Memory Requests
sum(kube_pod_container_resource_requests{resource="memory"})

# Cluster Memory Utilization (allocated vs capacity)
(
  sum(kube_pod_container_resource_requests{resource="memory"})
  /
  sum(kube_node_status_allocatable{resource="memory"})
) * 100

# ============================================
# COST OPTIMIZATION METRICS
# ============================================

# Total Node Count
count(kube_node_info)

# Nodes by Instance Type (for cost analysis)
count(kube_node_info) by (instance_type)

# CPU Waste (requested but not used)
sum(kube_pod_container_resource_requests{resource="cpu"})
-
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m]))

# Memory Waste (requested but not used)
sum(kube_pod_container_resource_requests{resource="memory"})
-
sum(container_memory_working_set_bytes{container!=""})

# Pod Density per Node
count(kube_pod_info) by (node)

# Average Pod Density
avg(count(kube_pod_info) by (node))

# ============================================
# AUTOSCALER ERROR METRICS
# ============================================

# Autoscaler Errors
sum(rate(cluster_autoscaler_errors_total[5m])) by (type)

# Node Groups at Max Size (cannot scale up further)
cluster_autoscaler_node_groups_count{state="maxSize"}

# Node Groups Backoff (temporary scale-up prevention)
cluster_autoscaler_node_groups_count{state="backoff"}

# ============================================
# SCHEDULING EFFICIENCY
# ============================================

# Pods Pending Due to Insufficient Resources
count(kube_pod_status_scheduled_time == 0 
  and kube_pod_status_phase{phase="Pending"} == 1)

# Time Pods Spend Pending (rough estimate)
time() - kube_pod_created{phase="Pending"}
```

---

### Alert Definitions for Cluster Autoscaler (OPTIONAL - Non-Paging)

```promql
# ============================================
# AUTOSCALER PERFORMANCE ALERTS (INFO/WARNING ONLY)
# ============================================

# INFO: Autoscaler Not Safe to Scale
- alert: ClusterAutoscalerNotSafe
  expr: |
    cluster_autoscaler_cluster_safe_to_autoscale == 0
  for: 15m
  labels:
    severity: info
    component: cluster-autoscaler
    impact: autoscaling-disabled
    tier: optional
  annotations:
    summary: "Cluster autoscaler reports cluster not safe to scale"
    description: |
      INFO: Autoscaling temporarily disabled
      
      Duration: 15+ minutes
      
      AUTOSCALING DISABLED:
      - Cannot scale up for capacity
      - Cannot scale down for cost optimization
      - Manual capacity management required
      
      COMMON CAUSES:
      - Too many unready nodes
      - Recent cluster instability
      - Configuration issues
      - Metrics unavailability
      
      INVESTIGATION (Non-Urgent):
      1. Check autoscaler logs:
         kubectl logs -n kube-system deployment/cluster-autoscaler
      2. Review cluster health
      3. Check for recent disruptions
      4. Verify autoscaler configuration
      
      IMPACT:
      - May need manual node additions during peak hours
      - Cost optimization delayed (nodes won't scale down)
      - Monitor cluster capacity manually
      
      This is informational - not customer impacting
    runbook_url: "https://runbooks.bank.internal/sre/gke/autoscaler-not-safe"
    dashboard_url: "https://monitoring.bank.internal/d/autoscaler"
    no_page: "true"

# WARNING: High Scale-Up Failure Rate
- alert: ClusterAutoscalerHighScaleUpFailures
  expr: |
    (
      sum(rate(cluster_autoscaler_failed_scale_ups_total[30m]))
      /
      sum(rate(cluster_autoscaler_scaled_up_nodes_total[30m] + cluster_autoscaler_failed_scale_ups_total[30m]))
    ) > 0.10  # 10% failure rate
  for: 20m
  labels:
    severity: warning
    component: cluster-autoscaler
    impact: capacity-growth-limited
    tier: optional
  annotations:
    summary: "Cluster autoscaler scale-up failure rate at {{ $value | humanizePercentage }}"
    description: |
      WARNING: Frequent autoscaler scale-up failures
      
      Failure Rate: {{ $value | humanizePercentage }}
      Acceptable: <10%
      Duration: 20+ minutes
      
      CAPACITY IMPACT:
      - Cannot add capacity automatically
      - Pods may remain pending
      - Manual intervention may be needed
      
      FAILURE REASONS:
      {{- range query "topk(3, sum(rate(cluster_autoscaler_failed_scale_ups_total[30m])) by (reason))" }}
      - {{ .Labels.reason }}: {{ .Value | humanize }}/min
      {{- end }}
      
      INVESTIGATION (During Business Hours):
      1. Check autoscaler logs for specific errors
      2. Review GCP quotas (common cause):
         - CPU quota exhausted
         - IP address exhaustion
         - Disk quota limits
      3. Verify node pool configurations
      4. Check for node group at max size
      
      COMMON CAUSES IN BANKING:
      - GCP project quotas (need quota increase)
      - IP address exhaustion (CIDR planning)
      - Node pool max size reached (intentional limit)
      - IAM permission issues
      - Specific machine types unavailable in zone
      
      REMEDIATION:
      - Request GCP quota increases if needed
      - Review node pool max size settings
      - Consider expanding to additional zones
      - Manual node addition if urgent capacity needed
      
      BANKING CONTEXT:
      - Check timing vs. business hours
      - Month-end processing may need manual capacity
      - Not urgent outside peak hours
    runbook_url: "https://runbooks.bank.internal/sre/gke/autoscaler-scale-up-failures"
    no_page: "true"

# INFO: Slow Scale-Up Performance
- alert: ClusterAutoscalerSlowScaleUp
  expr: |
    histogram_quantile(0.95,
      sum(rate(cluster_autoscaler_scale_up_duration_seconds_bucket[1h])) by (le)
    ) > 300  # 5 minutes
  for: 30m
  labels:
    severity: info
    component: cluster-autoscaler
    impact: slow-capacity-growth
    tier: optional
  annotations:
    summary: "Cluster autoscaler P95 scale-up time: {{ $value | humanizeDuration }}"
    description: |
      INFO: Autoscaler scale-up slower than optimal
      
      P95 Scale-Up Time: {{ $value | humanizeDuration }}
      Target: <5 minutes
      
      OPERATIONAL IMPACT:
      - Pods wait longer for capacity
      - Slower response to demand spikes
      - Degraded elasticity
      
      NOT CUSTOMER IMPACTING:
      - Pods eventually scheduled
      - Applications designed to handle delays
      - Performance optimization opportunity
      
      INVESTIGATION (Low Priority):
      1. Review GCP API latency
      2. Check node boot time trends
      3. Review image pull times
      4. Examine node initialization duration
      
      OPTIMIZATION OPPORTUNITIES:
      - Pre-pull common images
      - Use node pools with faster boot times
      - Optimize DaemonSet startup
      - Consider node image optimization
      
      BANKING HOURS CONTEXT:
      - During peak hours: Monitor closely
      - Off-peak: Low priority
      - Month-end prep: May need faster scale-up
    runbook_url: "https://runbooks.bank.internal/sre/gke/autoscaler-slow-scale-up"
    no_page: "true"

# WARNING: Pods Unschedulable for Extended Period
- alert: PodsUnschedulableExtended
  expr: |
    cluster_autoscaler_unschedulable_pods_count > 5
  for: 15m
  labels:
    severity: warning
    component: cluster-autoscaler
    impact: delayed-scheduling
    tier: optional
  annotations:
    summary: "{{ $value }} pods unschedulable for 15+ minutes"
    description: |
      WARNING: Pods waiting for capacity
      
      Unschedulable Pods: {{ $value }}
      Duration: 15+ minutes
      
      SCHEDULING DELAY:
      - Autoscaler triggered but nodes not ready yet
      - May indicate scale-up in progress
      - Could indicate scale-up failures
      
      INVESTIGATION:
      1. Check if scale-up in progress:
         kubectl get nodes -w
      2. Review pending pod reasons:
         kubectl get pods --all-namespaces --field-selector=status.phase=Pending
      3. Check autoscaler events:
         kubectl describe pod -n kube-system cluster-autoscaler-*
      
      EXPECTED SCENARIOS:
      - Normal during scale-up (5-10 minutes)
      - Morning scale-up for business hours
      - Deployment surge capacity
      
      CONCERNING SCENARIOS:
      - Duration >20 minutes
      - Recurring daily at same time
      - Critical workloads affected
      
      ACTIONS:
      - If critical workloads: Escalate to scale-up failures investigation
      - If non-critical: Monitor for auto-resolution
      - If recurring: Capacity planning discussion
      
      BANKING CONTEXT:
      - Check workload priority
      - Batch jobs can tolerate delays
      - Real-time services should scale faster
    runbook_url: "https://runbooks.bank.internal/sre/gke/pods-unschedulable"
    no_page: "true"

# INFO: Cluster Capacity Utilization High
- alert: ClusterCapacityUtilizationHigh
  expr: |
    (
      sum(kube_pod_container_resource_requests{resource="cpu"})
      /
      sum(kube_node_status_allocatable{resource="cpu"})
    ) > 0.80
  for: 30m
  labels:
    severity: info
    component: capacity-planning
    impact: limited-headroom
    tier: optional
  annotations:
    summary: "Cluster CPU capacity at {{ $value | humanizePercentage }}"
    description: |
      INFO: Cluster capacity utilization high
      
      CPU Allocated: {{ $value | humanizePercentage }}
      Comfortable Range: 60-80%
      
      LIMITED HEADROOM:
      - Less buffer for burst traffic
      - Slower deployment rollouts
      - Reduced fault tolerance
      
      PROACTIVE CAPACITY PLANNING:
      
      Current Capacity Breakdown:
      - Total CPU: {{ with query "sum(kube_node_status_allocatable{resource='cpu'})" }}{{ . | first | value | humanize }}{{- end }} cores
      - Requested CPU: {{ with query "sum(kube_pod_container_resource_requests{resource='cpu'})" }}{{ . | first | value | humanize }}{{- end }} cores
      - Available CPU: {{ with query "sum(kube_node_status_allocatable{resource='cpu'}) - sum(kube_pod_container_resource_requests{resource='cpu'})" }}{{ . | first | value | humanize }}{{- end }} cores
      
      ANALYSIS RECOMMENDATIONS:
      1. Review upcoming deployments/rollouts
      2. Check for over-requested resources
      3. Plan capacity additions for peak periods
      4. Review autoscaler max node limits
      
      BANKING CALENDAR CONSIDERATIONS:
      - Month-end: Typically requires 20-30% more capacity
      - Quarter-end: May need 40-50% surge capacity
      - Tax season: Extended high utilization
      - Holiday weekends: Can scale down aggressively
      
      OPTIMIZATION OPPORTUNITIES:
      1. Review resource requests vs actual usage
      2. Implement Vertical Pod Autoscaler
      3. Right-size over-provisioned workloads
      4. Consider spot/preemptible nodes for non-critical workloads
      
      COST IMPACT:
      - Current monthly compute cost estimate: $X
      - Scaling to 100% would add ~$Y
      - Optimization could save ~$Z
      
      NO IMMEDIATE ACTION REQUIRED
      This is informational for capacity planning discussions
    runbook_url: "https://runbooks.bank.internal/sre/gke/capacity-utilization"
    no_page: "true"

# INFO: Inefficient Resource Utilization (Waste)
- alert: ClusterResourceWaste
  expr: |
    (
      (sum(kube_pod_container_resource_requests{resource="cpu"})
       -
       sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])))
      /
      sum(kube_pod_container_resource_requests{resource="cpu"})
    ) > 0.40  # 40% waste
  for: 2h
  labels:
    severity: info
    component: cost-optimization
    impact: cost-inefficiency
    tier: optional
  annotations:
    summary: "{{ $value | humanizePercentage }} of requested CPU unused"
    description: |
      INFO: Significant resource over-provisioning detected
      
      CPU Waste: {{ $value | humanizePercentage }}
      Acceptable Waste: <30%
      
      COST OPTIMIZATION OPPORTUNITY:
      
      Resource Breakdown:
      - Requested CPU: {{ with query "sum(kube_pod_container_resource_requests{resource='cpu'})" }}{{ . | first | value | humanize }}{{- end }} cores
      - Actual Usage: {{ with query "sum(rate(container_cpu_usage_seconds_total{container!=''}[5m]))" }}{{ . | first | value | humanize }}{{- end }} cores
      - Wasted CPU: {{ with query "sum(kube_pod_container_resource_requests{resource='cpu'}) - sum(rate(container_cpu_usage_seconds_total{container!=''}[5m]))" }}{{ . | first | value | humanize }}{{- end }} cores
      
      ESTIMATED COST IMPACT:
      - Wasted capacity costs approximately $X/month
      - Right-sizing could save 20-30% on compute
      
      OPTIMIZATION ACTIONS (Non-Urgent):
      1. Identify top over-provisioned workloads:
         Review resource requests vs actual usage
      2. Implement Vertical Pod Autoscaler (VPA)
      3. Work with app teams to right-size resources
      4. Use VPA recommendations for gradual adjustments
      
      TOP OVER-PROVISIONED NAMESPACES:
      (Analysis requires additional queries)
      
      BANKING CONTEXT:
      - Safety margins are intentional for critical services
      - Some over-provisioning acceptable for:
        * Payment processing (burst capacity)
        * Trading platforms (low latency priority)
        * Core banking services (reliability over cost)
      - Batch/analytics workloads good optimization targets
      
      RECOMMENDED APPROACH:
      - Start with non-critical workloads
      - Implement VPA in recommendation mode first
      - Gradual rollout to avoid performance regressions
      - Monthly review of optimization progress
      
      NO IMMEDIATE ACTION REQUIRED
      Schedule capacity optimization review
    runbook_url: "https://runbooks.bank.internal/sre/gke/resource-waste"
    dashboard_url: "https://monitoring.bank.internal/d/cost-optimization"
    no_page: "true"
```

---

## 3.2 Horizontal Pod Autoscaler (HPA) Performance (OPTIONAL)

**SLO Definition**:
- HPA decision latency P95 < 30 seconds
- Scale-up decision latency P95 < 60 seconds
- Metric availability for HPA: 99%

**Business Value**:
- **Automatic Load Handling**: Services auto-scale during transaction spikes
- **Cost Efficiency**: Scale down during low-traffic periods
- **Developer Productivity**: Reduces manual intervention

**Why OPTIONAL**:
- Applications designed to handle slight over/under-provisioning
- Manual scaling still possible
- Not directly customer-impacting (assuming baseline capacity adequate)

---

### PromQL Queries for HPA Monitoring

```promql
# ============================================
# HPA STATUS AND BEHAVIOR
# ============================================

# HPAs at Max Replicas (cannot scale up further)
count(
  kube_horizontalpodautoscaler_status_current_replicas 
  == 
  kube_horizontalpodautoscaler_spec_max_replicas
) by (namespace)

# HPAs at Min Replicas (scaled down completely)
count(
  kube_horizontalpodautoscaler_status_current_replicas 
  == 
  kube_horizontalpodautoscaler_spec_min_replicas
) by (namespace)

# HPA Desired vs Current Replicas Gap
(
  kube_horizontalpodautoscaler_status_desired_replicas
  -
  kube_horizontalpodautoscaler_status_current_replicas
)

# HPA Current Metric Value vs Target
kube_horizontalpodautoscaler_status_current_metrics_average_value
/
kube_horizontalpodautoscaler_spec_target_metric_value

# ============================================
# HPA SCALING FREQUENCY
# ============================================

# HPA Scale-Up Events
sum(increase(kube_horizontalpodautoscaler_status_desired_replicas[5m]) > 0) 
by (namespace, horizontalpodautoscaler)

# HPA Scale-Down Events  
sum(increase(kube_horizontalpodautoscaler_status_desired_replicas[5m]) < 0) 
by (namespace, horizontalpodautoscaler)

# HPA Flapping Detection (rapid scale up/down)
# Multiple scale events in short period
count_over_time(
  (abs(delta(kube_horizontalpodautoscaler_status_desired_replicas[5m])) > 0)[30m:5m]
) > 3

# ============================================
# HPA METRIC SOURCE AVAILABILITY
# ============================================

# HPAs Unable to Get Metrics
kube_horizontalpodautoscaler_status_condition{
  condition="ScalingActive",
  status="false"
} == 1

# HPAs with Metric Problems
count(
  kube_horizontalpodautoscaler_status_condition{
    condition="AbleToScale",
    status="false"
  }
) by (namespace)
```

---

### Alert Definitions for HPA (OPTIONAL - Non-Paging)

```promql
# INFO: HPA at Max Replicas
- alert: HPAAtMaxReplicas
  expr: |
    (
      kube_horizontalpodautoscaler_status_current_replicas 
      == 
      kube_horizontalpodautoscaler_spec_max_replicas
    ) == 1
  for: 15m
  labels:
    severity: info
    component: hpa
    impact: scaling-limited
    tier: optional
  annotations:
    summary: "HPA {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }} at max replicas"
    description: |
      INFO: HPA reached maximum replica count
      
      HPA: {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }}
      Current Replicas: {{ $value }}
      Max Replicas: {{ $value }}
      Duration: 15+ minutes
      
      CAPACITY CONSTRAINT:
      - Cannot scale up further
      - Load may be approaching limits
      - Manual intervention may be needed for burst traffic
      
      CURRENT LOAD:
      Metric Value: {{ with query "kube_horizontalpodautoscaler_status_current_metrics_average_value" }}{{ . | first | value | humanize }}{{- end }}
      Target Value: {{ with query "kube_horizontalpodautoscaler_spec_target_metric_value" }}{{ . | first | value | humanize }}{{- end }}
      
      INVESTIGATION (Low Priority):
      1. Check if load is still increasing
      2. Review if max replicas limit is appropriate
      3. Verify application can handle current load
      4. Check for performance bottlenecks
      
      POTENTIAL ACTIONS:
      - Increase max replicas if justified by load
      - Review application efficiency
      - Check for unusual traffic patterns
      - Consider vertical scaling if horizontal not sufficient
      
      BANKING CONTEXT:
      - Expected during:
        * Month-end processing
        * Batch job windows
        * Marketing campaign launches
      - Unexpected during off-peak hours
      
      NO IMMEDIATE ACTION REQUIRED
      Monitor application performance metrics
    runbook_url: "https://runbooks.bank.internal/sre/gke/hpa-max-replicas"
    no_page: "true"

# WARNING: HPA Flapping
- alert: HPAFlapping
  expr: |
    count_over_time(
      (abs(delta(kube_horizontalpodautoscaler_status_desired_replicas[5m])) > 0)[30m:5m]
    ) > 4
  for: 10m
  labels:
    severity: warning
    component: hpa
    impact: instability
    tier: optional
  annotations:
    summary: "HPA {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }} flapping"
    description: |
      WARNING: HPA rapidly scaling up and down
      
      HPA: {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }}
      Scale Events (30m): {{ $value }}
      
      INSTABILITY DETECTED:
      - Frequent scale up/down cycles
      - Inefficient resource usage
      - Potential application impact
      - Wasted cloud costs (churn)
      
      CAUSES:
      - Metric oscillating around threshold
      - Scale-down delay too short
      - Inappropriate metric choice
      - Application warmup time not considered
      
      INVESTIGATION:
      1. Review metric trend (oscillating?)
      2. Check HPA configuration:
         kubectl describe hpa {{ $labels.horizontalpodautoscaler }} -n {{ $labels.namespace }}
      3. Review application startup time
      4. Check for external load patterns
      
      RECOMMENDED FIXES:
      - Increase scale-down stabilization window
      - Adjust metric thresholds (add hysteresis)
      - Use better metric (avg vs current)
      - Consider custom metrics
      
      COST IMPACT:
      - Pod churn wastes resources
      - Startup costs repeated unnecessarily
      
      NO IMMEDIATE ACTION REQUIRED
      Tune HPA configuration during maintenance window
    runbook_url: "https://runbooks.bank.internal/sre/gke/hpa-flapping"
    no_page: "true"

# WARNING: HPA Cannot Get Metrics
- alert: HPAMetricsUnavailable
  expr: |
    kube_horizontalpodautoscaler_status_condition{
      condition="ScalingActive",
      status="false"
    } == 1
  for: 10m
  labels:
    severity: warning
    component: hpa
    impact: autoscaling-disabled
    tier: optional
  annotations:
    summary: "HPA {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }} cannot retrieve metrics"
    description: |
      WARNING: HPA unable to make scaling decisions
      
      HPA: {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }}
      Reason: {{ $labels.reason }}
      
      AUTOSCALING DISABLED:
      - HPA cannot read current metrics
      - Scaling decisions suspended
      - Replicas frozen at current count
      
      COMMON CAUSES:
      - Metrics server down/unavailable
      - Custom metrics provider failing
      - Invalid metric specification
      - RBAC permission issues
      
      INVESTIGATION:
      1. Check metrics-server health:
         kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
      2. Test metric availability:
         kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods
      3. Review HPA status:
         kubectl describe hpa {{ $labels.horizontalpodautoscaler }} -n {{ $labels.namespace }}
      
      BANKING IMPACT:
      - Application won't auto-scale
      - May need manual scaling during peaks
      - Monitor application performance closely
      
      MITIGATION:
      - Manual scaling if needed
      - Fix metrics provider
      - Verify HPA metric configuration
    runbook_url: "https://runbooks.bank.internal/sre/gke/hpa-metrics-unavailable"
    no_page: "true"
```

---

## 3.3 Persistent Volume Performance (OPTIONAL)

**SLO Definition**:
- PV provisioning time P95 < 60 seconds
- PVC binding success rate: 99%
- Volume usage < 85% (prevent full disk)

**Business Value**:
- **Database Performance**: Fast storage critical for transactional databases
- **Cost Optimization**: Identify over-provisioned storage
- **Capacity Planning**: Prevent disk full incidents

**Why OPTIONAL**:
- PV issues typically don't cause immediate outages
- Manual provisioning possible
- Storage full usually has gradual warning period

---

### PromQL Queries for PV/PVC Monitoring

```promql
# ============================================
# PVC STATUS AND BINDING
# ============================================

# PVC Binding Status
count(kube_persistentvolumeclaim_status_phase{phase="Bound"}) by (namespace)

# PVCs Pending (not bound)
count(kube_persistentvolumeclaim_status_phase{phase="Pending"}) by (namespace, persistentvolumeclaim)

# PVCs Lost
count(kube_persistentvolumeclaim_status_phase{phase="Lost"}) by (namespace)

# ============================================
# VOLUME USAGE AND CAPACITY
# ============================================

# Volume Usage Percentage
(
  kubelet_volume_stats_used_bytes
  /
  kubelet_volume_stats_capacity_bytes
) * 100

# Volumes Over 85% Full
count(
  (kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) > 0.85
) by (namespace)

# Volume Inodes Usage Percentage
(
  kubelet_volume_stats_inodes_used
  /
  kubelet_volume_stats_inodes
) * 100

# ============================================
# STORAGE CLASS METRICS
# ============================================

# PVCs by Storage Class
count(kube_persistentvolumeclaim_info) by (storageclass)

# PV Count by Storage Class
count(kube_persistentvolume_info) by (storageclass)

# ============================================
# COST OPTIMIZATION
# ============================================

# Total Provisioned Storage (GB)
sum(kube_persistentvolumeclaim_resource_requests_storage_bytes) / (1024^3)

# Unused Storage (provisioned but not used)
sum(
  kube_persistentvolumeclaim_resource_requests_storage_bytes
  -
  kubelet_volume_stats_used_bytes
) / (1024^3)

# Storage Waste Percentage
(
  sum(kube_persistentvolumeclaim_resource_requests_storage_bytes - kubelet_volume_stats_used_bytes)
  /
  sum(kube_persistentvolumeclaim_resource_requests_storage_bytes)
) * 100
```

---

### Alert Definitions for PV/PVC (OPTIONAL - Non-Paging)

```promql
# WARNING: PVC Pending for Extended Time
- alert: PVCPendingExtended
  expr: |
    kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1
  for: 10m
  labels:
    severity: warning
    component: storage
    impact: volume-unavailable
    tier: optional
  annotations:
    summary: "PVC {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} pending for 10+ minutes"
    description: |
      WARNING: Persistent volume claim not binding
      
      PVC: {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }}
      Duration: 10+ minutes
      
      STORAGE PROVISIONING ISSUE:
      - Volume not binding to claim
      - Pod may be stuck waiting for storage
      - Database/stateful app may not start
      
      INVESTIGATION:
      1. Check PVC status:
         kubectl describe pvc {{ $labels.persistentvolumeclaim }} -n {{ $labels.namespace }}
      2. Check available PVs:
         kubectl get pv
      3. Verify storage class exists and working:
         kubectl get storageclass
      4. Check for provisioner errors:
         kubectl logs -n kube-system -l app=csi-provisioner
      
      COMMON CAUSES:
      - No PVs available matching requirements
      - Storage class misconfiguration
      - Cloud provider quota exhausted
      - CSI driver issues
      - Zone/region mismatch
      
      BANKING IMPACT:
      - Check if database pod affected
      - StatefulSet may not start
      - Data persistence blocked
      
      REMEDIATION:
      - Verify storage class configuration
      - Check cloud provider storage quotas
      - Manual PV creation if needed
      - Review pod/PVC zone affinity
    runbook_url: "https://runbooks.bank.internal/sre/gke/pvc-pending"
    no_page: "true"

# INFO: Volume Approaching Full
- alert: PersistentVolumeFillingUp
  expr: |
    (
      kubelet_volume_stats_used_bytes
      /
      kubelet_volume_stats_capacity_bytes
    ) > 0.85
  for: 30m
  labels:
    severity: info
    component: storage
    impact: capacity-warning
    tier: optional
  annotations:
    summary: "PV {{ $labels.persistentvolumeclaim }} at {{ $value | humanizePercentage }} capacity"
    description: |
      INFO: Persistent volume approaching full
      
      PVC: {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }}
      Usage: {{ $value | humanizePercentage }}
      Threshold: 85%
      
      CAPACITY WARNING:
      - Volume filling up
      - May impact application if full
      - Time to plan expansion
      
      VOLUME DETAILS:
      - Used: {{ with query "kubelet_volume_stats_used_bytes" }}{{ . | first | value | humanize }}B{{- end }}
      - Total: {{ with query "kubelet_volume_stats_capacity_bytes" }}{{ . | first | value | humanize }}B{{- end }}
      - Available: {{ with query "kubelet_volume_stats_available_bytes" }}{{ . | first | value | humanize }}B{{- end }}
      
      INVESTIGATION (Non-Urgent):
      1. Identify what's consuming space
      2. Check for log files or temp data
      3. Review data retention policies
      4. Plan volume expansion if needed
      
      BANKING CONTEXT:
      - Database volumes: Critical to expand proactively
      - Log volumes: Consider log rotation
      - Backup volumes: Review retention policies
      
      EXPANSION OPTIONS:
      - Resize PVC if storage class supports it
      - Migrate to larger volume
      - Clean up unnecessary data
      
      NO IMMEDIATE ACTION REQUIRED
      Plan capacity expansion within next week
    runbook_url: "https://runbooks.bank.internal/sre/gke/volume-filling-up"
    no_page: "true"

# INFO: High Storage Waste
- alert: StorageOverProvisioned
  expr: |
    (
      sum(kube_persistentvolumeclaim_resource_requests_storage_bytes - kubelet_volume_stats_used_bytes)
      /
      sum(kube_persistentvolumeclaim_resource_requests_storage_bytes)
    ) > 0.50  # 50% waste
  for: 24h
  labels:
    severity: info
    component: cost-optimization
    impact: storage-waste
    tier: optional
  annotations:
    summary: "{{ $value | humanizePercentage }} of provisioned storage unused"
    description: |
      INFO: Significant storage over-provisioning
      
      Wasted Storage: {{ $value | humanizePercentage }}
      
      COST OPTIMIZATION OPPORTUNITY:
      - Total Provisioned: {{ with query "sum(kube_persistentvolumeclaim_resource_requests_storage_bytes) / (1024^3)" }}{{ . | first | value | humanize }}GB{{- end }}
      - Actually Used: {{ with query "sum(kubelet_volume_stats_used_bytes) / (1024^3)" }}{{ . | first | value | humanize }}GB{{- end }}
      - Wasted: {{ with query "sum(kube_persistentvolumeclaim_resource_requests_storage_bytes - kubelet_volume_stats_used_bytes) / (1024^3)" }}{{ . | first | value | humanize }}GB{{- end }}
      
      ESTIMATED COST IMPACT:
      - Storage costs approximately $0.10-0.20/GB/month
      - Potential savings: $X/month by right-sizing
      
      OPTIMIZATION ACTIONS (Low Priority):
      1. Identify over-provisioned PVCs
      2. Work with app teams to right-size
      3. Resize PVCs if storage class supports shrinking
      4. Plan gradual optimization
      
      BANKING CONSIDERATIONS:
      - Some over-provisioning intentional (growth headroom)
      - Database volumes often provisioned for peak capacity
      - Archive/backup storage good optimization targets
      
      NO IMMEDIATE ACTION REQUIRED
      Schedule storage optimization review
    runbook_url: "https://runbooks.bank.internal/sre/gke/storage-waste"
    no_page: "true"
```

---

## 3.4 Image Pull Performance (OPTIONAL)

**SLO Definition**:
- Image pull success rate: 99%
- Image pull time P95 < 60 seconds

**Business Value**:
- **Faster Deployments**: Slow image pulls delay rollouts
- **Cost Optimization**: Identify unnecessarily large images
- **Developer Experience**: Faster iteration cycles

**Why OPTIONAL**:
- Image pull failures typically retry successfully
- Slow pulls delay but don't prevent deployments
- Image caching reduces frequency of pulls

---

### PromQL Queries for Image Pull Monitoring

```promql
# ============================================
# IMAGE PULL METRICS
# ============================================

# Pods with Image Pull Errors
count(kube_pod_container_status_waiting_reason{reason="ImagePullBackOff"})

# Pods with ErrImagePull
count(kube_pod_container_status_waiting_reason{reason="ErrImagePull"})

# Image Pull Errors by Namespace
count(kube_pod_container_status_waiting_reason{
  reason=~"ImagePullBackOff|ErrImagePull"
}) by (namespace)

# ============================================
# REGISTRY PERFORMANCE (if metrics available)
# ============================================

# Image Pull Duration (requires custom metrics)
# This would come from kubelet metrics if exposed
# histogram_quantile(0.95, rate(kubelet_image_pull_duration_seconds_bucket[10m]))
```

### Alert Definition for Image Pull (OPTIONAL)

```promql
# WARNING: Image Pull Failures
- alert: ImagePullFailures
  expr: |
    count(kube_pod_container_status_waiting_reason{
      reason=~"ImagePullBackOff|ErrImagePull"
    }) > 5
  for: 15m
  labels:
    severity: warning
    component: container-registry
    impact: deployment-delays
    tier: optional
  annotations:
    summary: "{{ $value }} pods experiencing image pull failures"
    description: |
      WARNING: Multiple pods cannot pull images
      
      Affected Pods: {{ $value }}
      
      DEPLOYMENT IMPACT:
      - New pods cannot start
      - Rollouts delayed
      - Scaling operations blocked
      
      PODS WITH IMAGE PULL ERRORS:
      {{- range query "kube_pod_container_status_waiting_reason{reason=~'ImagePullBackOff|ErrImagePull'}" }}
      - {{ .Labels.namespace }}/{{ .Labels.pod }}
      {{- end }}
      
      COMMON CAUSES:
      - Image doesn't exist in registry
      - Registry authentication failed
      - Network connectivity to registry
      - Registry rate limiting
      - Image tag or name typo
      
      INVESTIGATION:
      1. Check image names in pod specs
      2. Verify registry credentials (imagePullSecrets)
      3. Test registry connectivity
      4. Check registry service status
      5. Review registry quotas/rate limits
      
      BANKING CONTEXT:
      - Check if production images affected
      - Verify internal registry health
      - Review recent image pushes
      
      NO IMMEDIATE ACTION REQUIRED (unless production)
      Monitor for self-resolution or investigate during business hours
    runbook_url: "https://runbooks.bank.internal/sre/gke/image-pull-failures"
    no_page: "true"
```

---

## 3.5 Namespace Resource Quota Usage (OPTIONAL)

**SLO Definition**:
- Namespace quota usage < 85%
- Zero hard quota limit rejections for production namespaces

**Business Value**:
- **Prevent Deployment Failures**: Quota exhaustion blocks new deployments
- **Fair Resource Sharing**: Prevents namespace resource hogging
- **Cost Control**: Enforces spending boundaries

**Why OPTIONAL**:
- Quota issues detected before production impact (staging/dev failures)
- Manual quota adjustments available
- Teams alerted before hard limits hit

---

### PromQL Queries for Resource Quota Monitoring

```promql
# ============================================
# RESOURCE QUOTA USAGE
# ============================================

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

# Pod Count Quota Usage
(
  kube_resourcequota{resource="pods", type="used"}
  /
  kube_resourcequota{resource="pods", type="hard"}
) * 100

# Namespaces Approaching Quota Limits
count(
  (kube_resourcequota{type="used"} / kube_resourcequota{type="hard"}) > 0.85
) by (namespace, resource)
```

### Alert Definition for Resource Quotas (OPTIONAL)

```promql
# WARNING: Namespace Approaching Quota Limit
- alert: NamespaceApproachingQuota
  expr: |
    (
      kube_resourcequota{type="used"}
      /
      kube_resourcequota{type="hard"}
    ) > 0.85
  for: 30m
  labels:
    severity: warning
    component: resource-quotas
    impact: deployment-risk
    tier: optional
  annotations:
    summary: "Namespace {{ $labels.namespace }} at {{ $value | humanizePercentage }} of {{ $labels.resource }} quota"
    description: |
      WARNING: Namespace approaching resource quota limit
      
      Namespace: {{ $labels.namespace }}
      Resource: {{ $labels.resource }}
      Usage: {{ $value | humanizePercentage }}
      
      DEPLOYMENT RISK:
      - New deployments may be rejected
      - Scaling operations may fail
      - Development/testing blocked
      
      CURRENT QUOTA STATUS:
      - Used: {{ with query "kube_resourcequota{type='used'}" }}{{ . | first | value | humanize }}{{- end }}
      - Hard Limit: {{ with query "kube_resourcequota{type='hard'}" }}{{ . | first | value | humanize }}{{- end }}
      - Available: {{ with query "kube_resourcequota{type='hard'} - kube_resourcequota{type='used'}" }}{{ . | first | value | humanize }}{{- end }}
      
      INVESTIGATION (Non-Urgent):
      1. Review namespace resource usage trends
      2. Identify largest resource consumers
      3. Check for resource leaks (orphaned resources)
      4. Plan quota adjustment if legitimate growth
      
      ACTIONS:
      - Contact namespace owners to review usage
      - Clean up unused resources
      - Request quota increase if justified
      - Review resource requests for optimization
      
      BANKING GOVERNANCE:
      - Quota changes require approval
      - Document justification for increases
      - Review against budget allocations
      
      NO IMMEDIATE ACTION REQUIRED
      Coordinate with namespace owners during business hours
    runbook_url: "https://runbooks.bank.internal/sre/gke/namespace-quota"
    no_page: "true"
```

---

## Summary Table: OPTIONAL SLOs Priority Matrix

| Component | Business Value | Alert Threshold | Why Non-Paging |
|-----------|---------------|-----------------|----------------|
| **Cluster Autoscaler** | Cost optimization, elasticity | Scale-up failures >10% | Manual scaling possible; not customer-facing |
| **HPA** | Auto-scaling, efficiency | Flapping, max replicas | Apps designed for static capacity; alerts informational |
| **PV/PVC** | Storage capacity, cost | Volume >85% full | Gradual degradation; time to remediate |
| **Image Pull** | Deployment velocity | >5 pods failing | Retries succeed; delays but doesn't block |
| **Resource Quotas** | Cost control, fairness | Quota usage >85% | Detected early in dev/staging; manual adjustments |

---

## Recommended Dashboard Layout for OPTIONAL SLOs

### Dashboard 1: "Capacity Planning & Cost Optimization"
- Cluster CPU/Memory utilization trends
- Resource waste metrics
- Autoscaler performance
- Storage utilization and waste
- **Update Frequency**: Weekly review
- **Audience**: SRE + Finance + Eng Leadership

### Dashboard 2: "Autoscaling Health"
- HPA status by namespace
- Cluster autoscaler events timeline
- Pods at min/max replicas
- Scaling latency trends
- **Update Frequency**: Daily check
- **Audience**: SRE + Platform team

### Dashboard 3: "Developer Experience"
- Image pull latency distribution
- Deployment success rates
- PVC provisioning times
- Resource quota headroom
- **Update Frequency**: On-demand
- **Audience**: Application teams

---

## Key Principle for OPTIONAL SLOs

**These SLOs inform optimization efforts, not operational incidents:**

✅ **Do**: 
- Review weekly in capacity planning meetings
- Set quarterly OKRs for improvement (e.g., "Reduce resource waste by 20%")
- Use as input for budget planning
- Track trends for proactive capacity additions

❌ **Don't**:
- Page on-call for OPTIONAL SLO violations
- Treat as production incidents
- Require immediate remediation
- Block deployments based on these metrics

---
