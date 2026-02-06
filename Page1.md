# Comprehensive GKE Observability Framework for Banking Infrastructure

## Executive Summary

For a multinational bank running GKE, observability must balance **regulatory compliance**, **operational excellence**, and **customer experience**. This framework prioritizes:

1. **Customer-Facing SLOs** - Direct impact on banking services
2. **Platform Reliability SLOs** - Foundation for all services
3. **Security & Compliance Monitoring** - Non-negotiable for financial services
4. **Operational Efficiency** - Cost and performance optimization

---

## Part 1: SLO Framework

### Category A: MUST-HAVE SLOs (Tier 1 - Critical)

These SLOs directly impact customer experience, regulatory compliance, or platform stability.

#### 1.1 API Server Availability (Control Plane)

**SLO Definition**: 99.95% availability over 30-day window

**Business Rationale**:
- API server downtime = complete platform outage
- Banking services become unavailable
- Cannot deploy emergency fixes or rollbacks
- Regulatory reporting systems may fail
- Maximum acceptable downtime: ~22 minutes/month

**Measurement Strategy**:
```promql
# SLI: Ratio of successful API requests to total requests
# Success = HTTP 2xx responses
# Window: 30 days rolling

apiserver:availability:ratio_30d = 
  sum(rate(apiserver_request_total{code=~"2.."}[30d]))
  /
  sum(rate(apiserver_request_total[30d]))
```

**PromQL Queries**:

```promql
# Current SLO Compliance (30-day window)
(
  sum(rate(apiserver_request_total{code=~"2.."}[30d]))
  /
  sum(rate(apiserver_request_total[30d]))
) * 100

# Error Budget Remaining (%)
# Error budget = 1 - SLO = 0.0005 (0.05%)
(1 - (
  (1 - sum(rate(apiserver_request_total{code=~"2.."}[30d])) / sum(rate(apiserver_request_total[30d])))
  /
  (1 - 0.9995)
)) * 100

# Error Budget Consumption Rate
# How fast we're burning through monthly budget
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
# Triggers when burning 30-day budget in ~2 days
# Page immediately
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
    compliance_risk: high
  annotations:
    summary: "API Server burning error budget at 14.4x rate"
    description: |
      CRITICAL: API Server availability severely degraded
      
      Current Error Rate: {{ $value | humanizePercentage }}
      SLO Target: 99.95%
      Error Budget Burn Rate: 14.4x (consuming 30 days in ~2 days)
      
      IMMEDIATE IMPACT:
      - All Kubernetes operations affected
      - Unable to deploy or rollback services
      - Potential customer service disruption
      
      INVESTIGATION PRIORITY:
      1. Check API server pod status and logs
      2. Verify etcd cluster health
      3. Review recent control plane changes
      4. Check authentication/authorization systems
      5. Examine load balancer health
      
      ESCALATION: Page SRE on-call + Platform Lead
    runbook_url: "https://runbooks.bank.internal/sre/gke/api-server-fast-burn"
    dashboard_url: "https://monitoring.bank.internal/d/api-server-health"
    incident_severity: "P1"

# WARNING: Slow Burn (6x) - 6 hour window  
# Triggers when burning 30-day budget in ~5 days
# Alert but don't page
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
    impact: platform-degradation
  annotations:
    summary: "API Server burning error budget at 6x rate"
    description: |
      WARNING: Elevated API Server error rate
      
      Current Error Rate: {{ $value | humanizePercentage }}
      Error Budget Burn Rate: 6x (consuming 30 days in ~5 days)
      Time to Budget Exhaustion: ~5 days at current rate
      
      ACTION REQUIRED:
      - Review error patterns by verb and resource type
      - Check for client-side issues (retries, timeouts)
      - Monitor trends - may escalate to fast burn
      
      INVESTIGATION:
      1. Analyze error distribution by status code
      2. Identify problematic API clients
      3. Review recent application deployments
      4. Check for increased traffic patterns
    runbook_url: "https://runbooks.bank.internal/sre/gke/api-server-slow-burn"
```

**Rationale for Multi-Burn-Rate Approach**:
- **Fast burn (1h/5m windows)**: Catches sudden incidents requiring immediate response
- **Slow burn (6h/30m windows)**: Catches gradual degradation before it becomes critical
- **Two windows per burn rate**: Avoids false positives from transient spikes
- **Banking context**: Regulatory requirements demand quick incident response

---

#### 1.2 API Server Latency (Control Plane)

**SLO Definition**: 
- P95 latency < 1 second for mutating operations
- P99 latency < 5 seconds for mutating operations

**Business Rationale**:
- Slow API responses delay critical operations (deployments, scaling, incident response)
- Payment processing systems require fast Kubernetes operations
- Automated trading platforms need low-latency infrastructure changes
- Compliance: Must meet RTO (Recovery Time Objective) requirements

**PromQL Queries**:

```promql
# P95 Latency by Operation Type (5-minute window)
histogram_quantile(0.95,
  sum(rate(apiserver_request_duration_seconds_bucket{
    verb=~"POST|PUT|PATCH|DELETE",
    subresource!~"proxy|attach|exec|portforward"
  }[5m])) by (le, verb, resource)
)

# P99 Latency (5-minute window)
histogram_quantile(0.99,
  sum(rate(apiserver_request_duration_seconds_bucket{
    verb=~"POST|PUT|PATCH|DELETE",
    subresource!~"proxy|attach|exec|portforward"
  }[5m])) by (le, verb, resource)
)

# SLO Compliance - Percentage of requests under 1s (30-day)
(
  sum(rate(apiserver_request_duration_seconds_bucket{
    le="1.0",
    verb=~"POST|PUT|PATCH|DELETE"
  }[30d]))
  /
  sum(rate(apiserver_request_duration_seconds_count{
    verb=~"POST|PUT|PATCH|DELETE"
  }[30d]))
) * 100

# Latency by Resource Type (identify slow operations)
histogram_quantile(0.95,
  sum(rate(apiserver_request_duration_seconds_bucket{
    verb=~"POST|PUT|PATCH|DELETE"
  }[5m])) by (le, resource)
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
    slo_type: latency
    impact: user-experience
  annotations:
    summary: "API Server P95 latency exceeds 1s SLO"
    description: |
      API Server response times degraded
      
      Current P95 Latency: {{ $value | humanizeDuration }}
      SLO Target: 1.0s
      Duration: 10 minutes
      
      IMPACT:
      - Slower deployments and rollbacks
      - Delayed autoscaling responses
      - Degraded CI/CD pipeline performance
      
      INVESTIGATION:
      1. Check etcd cluster performance (backend commit latency)
      2. Review API server CPU/memory utilization
      3. Identify slow resource types:
         kubectl top pods -n kube-system
      4. Check for large LIST operations or inefficient queries
      5. Examine watch cache efficiency
      
      TOP SLOW OPERATIONS:
      {{- range query "topk(5, histogram_quantile(0.95, sum(rate(apiserver_request_duration_seconds_bucket{verb=~'POST|PUT|PATCH|DELETE'}[10m])) by (le, resource)))" }}
      - {{ .Labels.resource }}: {{ .Value | humanizeDuration }}
      {{- end }}
    runbook_url: "https://runbooks.bank.internal/sre/gke/api-server-latency"

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
    slo_type: latency
    impact: severe-degradation
  annotations:
    summary: "API Server P99 latency critically high (>5s)"
    description: |
      CRITICAL: Severe API Server performance degradation
      
      Current P99 Latency: {{ $value | humanizeDuration }}
      Threshold: 5.0s
      
      SEVERE IMPACT:
      - Critical operations may timeout
      - Incident response severely hampered
      - Potential cascade failures
      
      IMMEDIATE ACTIONS:
      1. Check etcd health immediately
      2. Review API server resource exhaustion
      3. Check for stuck requests/deadlocks
      4. Consider API server restart if persistent
    runbook_url: "https://runbooks.bank.internal/sre/gke/api-server-latency-critical"
    incident_severity: "P1"
```

**Detailed Rationale**:

1. **Why separate P95 and P99?**
   - P95: Day-to-day operational quality
   - P99: Catches worst-case scenarios affecting critical operations
   - Banking: Even 1% of slow requests = thousands of affected transactions

2. **Why exclude WATCH/CONNECT?**
   - These are long-lived connections with different latency characteristics
   - Including them skews percentile calculations
   - Measured separately with different SLOs

3. **Why 10m for P95 vs 5m for P99?**
   - P95 violations need time to confirm trend
   - P99 violations are more severe, require faster response

---

#### 1.3 etcd Cluster Health (Control Plane)

**SLO Definition**:
- 99.99% availability
- P99 backend commit latency < 25ms
- P99 WAL fsync latency < 10ms
- Zero leader changes under normal operation

**Business Rationale**:
- etcd is single point of failure for entire Kubernetes cluster
- Stores all cluster state including secrets (encryption keys, credentials)
- Banking: PCI-DSS and regulatory compliance require high availability of audit logs
- etcd failure = complete platform failure, cannot be tolerated

**PromQL Queries**:

```promql
# Leader Stability - Should be 0 under normal conditions
rate(etcd_server_leader_changes_seen_total[10m])

# Backend Commit Duration P99 (disk write performance)
histogram_quantile(0.99,
  rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])
) * 1000  # Convert to milliseconds

# WAL Fsync Duration P99 (write-ahead log sync)
histogram_quantile(0.99,
  rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])
) * 1000

# Proposal Failure Rate (consensus failures)
rate(etcd_server_proposals_failed_total[5m])

# Database Size (approaching quota warnings)
etcd_mvcc_db_total_size_in_bytes / (8 * 1024 * 1024 * 1024) * 100  # % of 8GB quota

# Snapshot Duration (backup performance)
histogram_quantile(0.99,
  rate(etcd_snapshot_save_total_duration_seconds_bucket[30m])
)

# etcd Member Health
up{job="etcd"} == 1

# Compaction Duration (maintenance performance)
histogram_quantile(0.99,
  rate(etcd_debugging_mvcc_db_compaction_total_duration_milliseconds_bucket[30m])
)
```

**Alert Definitions**:

```promql
# CRITICAL: etcd Leader Changes (Cluster Instability)
- alert: EtcdLeaderChanges
  expr: |
    increase(etcd_server_leader_changes_seen_total[15m]) > 3
  labels:
    severity: critical
    component: etcd
    impact: platform-stability
    compliance_risk: critical
  annotations:
    summary: "etcd cluster experiencing leadership instability"
    description: |
      CRITICAL: etcd cluster unstable - {{ $value }} leader changes in 15 minutes
      
      IMPACT:
      - All Kubernetes operations may be affected
      - Data consistency at risk
      - Potential split-brain scenarios
      - Compliance audit log integrity concerns
      
      ROOT CAUSE INVESTIGATION:
      1. Check network connectivity between etcd members
      2. Review etcd member resource utilization (CPU/memory)
      3. Check for clock skew between nodes
      4. Examine disk I/O performance and latency
      5. Review recent infrastructure changes
      
      ETCD HEALTH CHECK:
      - Check member list: etcdctl member list
      - Check endpoint health: etcdctl endpoint health
      - Check endpoint status: etcdctl endpoint status
      
      IMMEDIATE ACTION:
      - Do NOT restart etcd members unless directed by runbook
      - Engage GCP support if managed control plane
      - Prepare for potential cluster rebuild if persistent
    runbook_url: "https://runbooks.bank.internal/sre/gke/etcd-leader-changes"
    incident_severity: "P0"
    escalation: "immediate"

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
    slo_type: latency
    impact: platform-performance
  annotations:
    summary: "etcd backend commit latency exceeds 25ms"
    description: |
      CRITICAL: etcd disk performance degraded
      
      Current P99 Commit Latency: {{ $value | humanizeDuration }}
      SLO Threshold: 25ms
      
      IMPACT:
      - Slow API server responses
      - Delayed consensus operations
      - Potential proposal timeouts
      - Risk of leader election triggers
      
      INVESTIGATION (Priority Order):
      1. Check disk I/O metrics:
         - IOPS saturation
         - Disk queue depth
         - Read/write latency
      2. Verify disk type (must be SSD, preferably NVMe)
      3. Check for competing disk I/O (noisy neighbors)
      4. Review etcd database size and compaction status
      5. Check filesystem performance (ext4 vs xfs)
      
      BANKING CONTEXT:
      - Slow etcd = slow transaction processing
      - May impact real-time payment systems
      - Could affect fraud detection latency
      
      MITIGATION:
      - Consider database compaction if size > 2GB
      - Review defragmentation needs
      - Evaluate disk upgrade (higher IOPS)
    runbook_url: "https://runbooks.bank.internal/sre/gke/etcd-commit-latency"
    incident_severity: "P1"

# CRITICAL: etcd WAL Fsync Latency High
- alert: EtcdHighWALFsyncLatency
  expr: |
    histogram_quantile(0.99,
      rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])
    ) > 0.010  # 10ms
  for: 5m
  labels:
    severity: critical
    component: etcd
    slo_type: latency
    impact: data-durability
  annotations:
    summary: "etcd WAL fsync latency exceeds 10ms"
    description: |
      CRITICAL: etcd write-ahead log sync performance degraded
      
      Current P99 Fsync Latency: {{ $value | humanizeDuration }}
      SLO Threshold: 10ms
      
      DATA DURABILITY RISK:
      - Slower writes to WAL
      - Increased risk of data loss on crash
      - Compliance concern: Audit log durability
      
      IMMEDIATE INVESTIGATION:
      1. Check disk write performance and saturation
      2. Verify disk write cache is enabled appropriately
      3. Check for filesystem issues
      4. Review recent storage infrastructure changes
      
      BANKING COMPLIANCE NOTE:
      - WAL durability critical for audit trail integrity
      - May require incident report for compliance team
    runbook_url: "https://runbooks.bank.internal/sre/gke/etcd-wal-fsync"
    incident_severity: "P1"
    compliance_notification: "required"

# CRITICAL: etcd Proposal Failures
- alert: EtcdProposalFailures
  expr: |
    rate(etcd_server_proposals_failed_total[5m]) > 0.01
  for: 2m
  labels:
    severity: critical
    component: etcd
    impact: consensus-failure
  annotations:
    summary: "etcd experiencing proposal failures"
    description: |
      CRITICAL: etcd consensus mechanism failing
      
      Proposal Failure Rate: {{ $value }}/sec
      
      CRITICAL IMPACT:
      - Writes to etcd failing
      - Cluster state updates blocked
      - Kubernetes operations will fail
      
      IMMEDIATE ACTIONS:
      1. Check etcd cluster member health
      2. Verify quorum (need majority of members)
      3. Check network partitions
      4. Review etcd logs for specific errors
      
      DO NOT:
      - Restart etcd members without analysis
      - Make changes to etcd cluster during failures
    runbook_url: "https://runbooks.bank.internal/sre/gke/etcd-proposals"
    incident_severity: "P0"

# WARNING: etcd Database Size Approaching Quota
- alert: EtcdDatabaseSizeHigh
  expr: |
    etcd_mvcc_db_total_size_in_bytes / (8 * 1024 * 1024 * 1024) > 0.80
  for: 10m
  labels:
    severity: warning
    component: etcd
    impact: capacity
  annotations:
    summary: "etcd database size at {{ $value | humanizePercentage }} of 8GB quota"
    description: |
      WARNING: etcd database approaching size limit
      
      Current Size: {{ $value | humanizePercentage }} of quota
      Threshold: 80%
      
      CAPACITY PLANNING REQUIRED:
      1. Review object count growth trends
      2. Check for resource leaks (orphaned objects)
      3. Plan database compaction
      4. Consider defragmentation
      
      BANKING IMPACT:
      - Full database = no new writes
      - Cannot create new resources
      - May prevent emergency deployments
      
      ACTIONS:
      1. Identify largest resource types
      2. Clean up old completed jobs/pods
      3. Review secret rotation policies
      4. Plan maintenance window for compaction
    runbook_url: "https://runbooks.bank.internal/sre/gke/etcd-database-size"

# WARNING: etcd Member Down
- alert: EtcdMemberDown
  expr: |
    up{job="etcd"} == 0
  for: 3m
  labels:
    severity: critical
    component: etcd
    impact: redundancy-loss
  annotations:
    summary: "etcd member {{ $labels.instance }} is down"
    description: |
      CRITICAL: etcd cluster member unreachable
      
      Member: {{ $labels.instance }}
      
      QUORUM STATUS:
      - 3-member cluster: Can tolerate 1 failure
      - Currently: {{ $value }} members down
      - Risk: One more failure = cluster unavailable
      
      IMMEDIATE ACTIONS:
      1. Verify member is actually down (not network issue)
      2. Check remaining members are healthy
      3. Do NOT remove failed member unless instructed
      4. Engage GCP support for managed control planes
      
      ESCALATION:
      - If 2+ members down: P0 incident
      - Prepare for potential DR procedures
    runbook_url: "https://runbooks.bank.internal/sre/gke/etcd-member-down"
    incident_severity: "P1"
```

**Detailed Rationale**:

1. **Why such strict etcd SLOs?**
   - etcd is non-redundant from application perspective (cannot have multiple independent clusters)
   - Banking: Single source of truth for all cluster state including security credentials
   - Regulatory: Audit logs must be durable and highly available

2. **Why measure both commit and fsync latency?**
   - Commit: Overall backend performance (includes B-tree operations)
   - Fsync: Disk durability guarantee (critical for compliance)
   - Different issues require different remediation

3. **Why zero leader changes?**
   - Leader changes indicate cluster instability
   - Can cause brief write unavailability
   - Multiple changes = serious problem requiring immediate attention

---

#### 1.4 Workload Availability (Application Layer)

**SLO Definition**:
- 99.9% pod availability for customer-facing services
- 99.5% pod availability for internal services
- Deployment rollouts complete successfully 99% of the time

**Business Rationale**:
- Customer-facing banking services must be highly available
- Internal services (fraud detection, reporting) still critical but slightly more tolerance
- Failed deployments delay features and fixes, increase operational toil

**PromQL Queries**:

```promql
# Pod Availability by Namespace (Customer-Facing Services)
(
  sum(kube_pod_status_phase{
    phase="Running",
    namespace=~"customer-.*|payments|cards|loans"
  }) by (namespace)
  /
  sum(kube_pod_status_phase{
    namespace=~"customer-.*|payments|cards|loans"
  }) by (namespace)
) * 100

# Deployment Replica Availability
(
  kube_deployment_status_replicas_available
  /
  kube_deployment_spec_replicas
) * 100

# Deployment Rollout Success Rate (30-day window)
(
  count(kube_deployment_status_condition{
    condition="Progressing",
    status="true"
  } == 1) by (namespace)
  /
  count(kube_deployment_spec_replicas) by (namespace)
) * 100

# Pod Ready Time (startup latency)
# This requires tracking creation time to ready time
(time() - kube_pod_created{phase="Running"}) 
  and on(pod, namespace) 
kube_pod_status_ready{condition="true"} == 0

# Container Restart Rate by Service
rate(kube_pod_container_status_restarts_total[15m])

# Pods Stuck in Non-Running States
count(kube_pod_status_phase{
  phase!="Running",
  phase!="Succeeded"
}) by (namespace, phase)
```

**Alert Definitions**:

```promql
# CRITICAL: Customer-Facing Service Degraded
- alert: CustomerServiceDegraded
  expr: |
    (
      sum(kube_pod_status_phase{
        phase="Running",
        namespace=~"customer-.*|payments|cards|loans|mobile-banking"
      }) by (namespace)
      /
      sum(kube_pod_status_phase{
        namespace=~"customer-.*|payments|cards|loans|mobile-banking"
      }) by (namespace)
    ) < 0.999
  for: 2m
  labels:
    severity: critical
    component: workload
    service_tier: customer-facing
    impact: revenue
    compliance_risk: medium
  annotations:
    summary: "Customer-facing service {{ $labels.namespace }} below 99.9% availability"
    description: |
      CRITICAL: Customer service availability degraded
      
      Service: {{ $labels.namespace }}
      Current Availability: {{ $value | humanizePercentage }}
      SLO Target: 99.9%
      
      CUSTOMER IMPACT:
      - Potential transaction failures
      - Degraded user experience
      - Possible revenue loss
      - Brand reputation risk
      
      INVESTIGATION PRIORITY:
      1. Identify failing pods:
         kubectl get pods -n {{ $labels.namespace }} | grep -v Running
      2. Check pod events and logs
      3. Verify deployment health
      4. Check for resource exhaustion
      5. Review recent changes
      
      ESCALATION:
      - Notify application team immediately
      - Engage SRE if infrastructure-related
      - Prepare customer communication if prolonged
    runbook_url: "https://runbooks.bank.internal/sre/gke/customer-service-degraded"
    incident_severity: "P1"
    notify_teams: "app-owners,sre,customer-support"

# WARNING: Deployment Replicas Mismatch
- alert: DeploymentReplicasMismatch
  expr: |
    (
      kube_deployment_spec_replicas
      -
      kube_deployment_status_replicas_available
    ) > 0
  for: 15m
  labels:
    severity: warning
    component: workload
    impact: capacity
  annotations:
    summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} missing {{ $value }} replicas"
    description: |
      WARNING: Deployment not at desired replica count
      
      Deployment: {{ $labels.namespace }}/{{ $labels.deployment }}
      Desired Replicas: {{ $labels.spec_replicas }}
      Available Replicas: {{ $labels.available_replicas }}
      Missing: {{ $value }}
      Duration: 15 minutes
      
      CAPACITY IMPACT:
      - Reduced redundancy
      - Less capacity to handle traffic spikes
      - Slower response to load increases
      
      INVESTIGATION:
      1. Check pending pods:
         kubectl get pods -n {{ $labels.namespace }} -l app={{ $labels.deployment }}
      2. Review pod events
      3. Check for:
         - Resource quota exhaustion
         - Insufficient cluster capacity
         - Image pull errors
         - Failed health checks
      
      BANKING CONTEXT:
      - May affect ability to handle peak loads (month-end, payroll)
      - Could impact failover capacity during incidents
    runbook_url: "https://runbooks.bank.internal/sre/gke/deployment-replicas"

# CRITICAL: High Pod Restart Rate
- alert: HighPodRestartRate
  expr: |
    rate(kube_pod_container_status_restarts_total[15m]) > 0.05
  for: 10m
  labels:
    severity: warning
    component: workload
    impact: stability
  annotations:
    summary: "High restart rate for {{ $labels.namespace }}/{{ $labels.pod }}"
    description: |
      WARNING: Pod restarting frequently
      
      Pod: {{ $labels.namespace }}/{{ $labels.pod }}/{{ $labels.container }}
      Restart Rate: {{ $value }}/sec
      Threshold: 0.05/sec (3/min)
      
      STABILITY CONCERNS:
      - CrashLoopBackOff potential
      - Resource thrashing
      - Application instability
      - Possible memory leaks or OOM kills
      
      INVESTIGATION:
      1. Check container logs:
         kubectl logs -n {{ $labels.namespace }} {{ $labels.pod }} -c {{ $labels.container }} --previous
      2. Check restart reason:
         kubectl describe pod -n {{ $labels.namespace }} {{ $labels.pod }}
      3. Review resource limits vs actual usage
      4. Check for OOMKilled events
      5. Examine application health check configuration
      
      COMMON CAUSES IN BANKING APPS:
      - Database connection pool exhaustion
      - Memory leaks in Java applications
      - Misconfigured health checks (too aggressive)
      - External dependency timeouts
    runbook_url: "https://runbooks.bank.internal/sre/gke/high-restart-rate"

# CRITICAL: Deployment Rollout Stuck
- alert: DeploymentRolloutStuck
  expr: |
    kube_deployment_status_condition{
      condition="Progressing",
      status="false"
    } == 1
  for: 10m
  labels:
    severity: critical
    component: workload
    impact: deployment-blocked
  annotations:
    summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} rollout not progressing"
    description: |
      CRITICAL: Deployment rollout stuck
      
      Deployment: {{ $labels.namespace }}/{{ $labels.deployment }}
      Status: Not Progressing
      Duration: 10+ minutes
      
      DEPLOYMENT BLOCKED:
      - New version not deploying
      - Cannot rollout fixes or features
      - May need manual intervention
      
      INVESTIGATION:
      1. Check rollout status:
         kubectl rollout status deployment/{{ $labels.deployment }} -n {{ $labels.namespace }}
      2. Review ReplicaSet status:
         kubectl get rs -n {{ $labels.namespace }} -l app={{ $labels.deployment }}
      3. Check pod events for new pods
      4. Review deployment strategy (RollingUpdate settings)
      
      COMMON ISSUES:
      - Insufficient resources for new pods
      - Failed health checks (readiness/liveness)
      - Image pull errors
      - Init container failures
      - PVC mount issues
      
      ROLLBACK PROCEDURE:
      If critical service and rollout consistently failing:
      kubectl rollout undo deployment/{{ $labels.deployment }} -n {{ $labels.namespace }}
    runbook_url: "https://runbooks.bank.internal/sre/gke/rollout-stuck"
    incident_severity: "P2"

# WARNING: Pods Pending Too Long
- alert: PodsPendingTooLong
  expr: |
    count(
      kube_pod_status_phase{phase="Pending"} 
      and 
      (time() - kube_pod_created) > 300
    ) by (namespace) > 5
  for: 5m
  labels:
    severity: warning
    component: workload
    impact: capacity
  annotations:
    summary: "{{ $value }} pods stuck Pending in {{ $labels.namespace }}"
    description: |
      WARNING: Multiple pods unable to schedule
      
      Namespace: {{ $labels.namespace }}
      Pending Pods: {{ $value }}
      Duration: 5+ minutes
      
      SCHEDULING ISSUES:
      - Insufficient cluster capacity
      - Unschedulable pod constraints
      - Resource quotas exceeded
      - Node selector mismatches
      
      INVESTIGATION:
      1. List pending pods:
         kubectl get pods -n {{ $labels.namespace }} --field-selector=status.phase=Pending
      2. Check pod events for scheduling failures:
         kubectl describe pod <pod-name> -n {{ $labels.namespace }}
      3. Check cluster capacity:
         kubectl top nodes
      4. Review node conditions and taints
      
      CAPACITY PLANNING:
      - May need cluster autoscaling
      - Review resource requests vs actual usage
      - Consider node pool expansion
    runbook_url: "https://runbooks.bank.internal/sre/gke/pods-pending"
```

**Detailed Rationale**:

1. **Why different SLOs for customer-facing vs internal?**
   - Customer services directly impact revenue and brand
   - Internal services still critical but can tolerate slightly more downtime
   - Allows more aggressive deployment strategies for internal services

2. **Why 15-minute window for replica mismatch?**
   - Allows time for normal scaling operations
   - Avoids alerting during legitimate scale-down
   - Still catches persistent issues before they escalate

3. **Why track deployment rollout success?**
   - Failed deployments create operational toil
   - Multiple failures indicate systemic issues
   - Banking: Need reliable deployment pipeline for security patches

---

### 1.5 Node Health and Availability

**SLO Definition**:
- 99.9% node availability (NotReady events)
- Zero node pressure conditions (Memory/Disk/PID)
- P95 kubelet PLEG relist latency < 1 second

**Business Rationale**:
- Node failures cause pod evictions and service disruptions
- Banking workloads are stateful (databases, caches) - node failures are expensive
- Node pressure triggers evictions affecting random services unpredictably
- PLEG issues cause cascading failures across the node

**PromQL Queries**:

```promql
# Node Availability Ratio
(
  sum(kube_node_status_condition{condition="Ready",status="true"})
  /
  count(kube_node_info)
) * 100

# Nodes in NotReady State
count(kube_node_status_condition{condition="Ready",status!="true"})

# Nodes with Pressure Conditions
sum(kube_node_status_condition{
  condition=~"MemoryPressure|DiskPressure|PIDPressure",
  status="true"
}) by (node, condition)

# Kubelet PLEG Relist Latency P95
histogram_quantile(0.95,
  sum(rate(kubelet_pleg_relist_duration_seconds_bucket[10m])) 
  by (le, node)
)

# Node CPU Utilization
(
  sum(rate(container_cpu_usage_seconds_total{id="/"}[5m])) by (node)
  /
  sum(kube_node_status_allocatable{resource="cpu"}) by (node)
) * 100

# Node Memory Utilization
(
  sum(container_memory_working_set_bytes{id="/"}) by (node)
  /
  sum(kube_node_status_allocatable{resource="memory"}) by (node)
) * 100

# Kubelet Runtime Operation Errors
rate(kubelet_runtime_operations_errors_total[5m]) > 0

# Node Filesystem Usage
(
  kubelet_volume_stats_used_bytes{persistentvolumeclaim=""}
  /
  kubelet_volume_stats_capacity_bytes{persistentvolumeclaim=""}
) * 100
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
    impact: capacity-loss
    compliance_risk: medium
  annotations:
    summary: "Node {{ $labels.node }} is NotReady"
    description: |
      CRITICAL: Node unreachable or unhealthy
      
      Node: {{ $labels.node }}
      Status: NotReady
      Duration: 5+ minutes
      
      IMMEDIATE IMPACT:
      - Pods on this node may be evicted
      - Reduced cluster capacity
      - Potential service disruptions
      - Stateful workloads at risk
      
      INVESTIGATION PRIORITY:
      1. Check node conditions:
         kubectl describe node {{ $labels.node }}
      2. Check kubelet status on node:
         gcloud compute ssh {{ $labels.node }} -- sudo systemctl status kubelet
      3. Review kubelet logs:
         gcloud compute ssh {{ $labels.node }} -- sudo journalctl -u kubelet -n 100
      4. Check node system resources (CPU, memory, disk)
      5. Review recent infrastructure changes
      
      BANKING CONTEXT:
      - May be running database pods (StatefulSets)
      - Check for critical workloads before draining
      - Coordinate with app teams if manual intervention needed
      
      REMEDIATION:
      - If persistent and no recovery: drain and replace node
      - For GKE: Node auto-repair should trigger
      - Manual: kubectl drain {{ $labels.node }} --ignore-daemonsets --delete-emptydir-data
    runbook_url: "https://runbooks.bank.internal/sre/gke/node-not-ready"
    incident_severity: "P1"

# CRITICAL: Node Memory Pressure
- alert: NodeMemoryPressure
  expr: |
    kube_node_status_condition{condition="MemoryPressure",status="true"} == 1
  for: 5m
  labels:
    severity: critical
    component: node
    impact: eviction-risk
  annotations:
    summary: "Node {{ $labels.node }} under memory pressure"
    description: |
      CRITICAL: Node experiencing memory pressure
      
      Node: {{ $labels.node }}
      Condition: MemoryPressure
      
      EVICTION RISK:
      - Kubelet will start evicting pods
      - Eviction order: BestEffort → Burstable → Guaranteed
      - Service disruptions likely
      
      IMMEDIATE INVESTIGATION:
      1. Check node memory usage:
         kubectl top node {{ $labels.node }}
      2. Identify memory-intensive pods:
         kubectl top pods --all-namespaces --sort-by=memory | grep {{ $labels.node }}
      3. Check for memory leaks
      4. Review pod memory requests vs limits
      
      BANKING IMPACT:
      - Database pods may be evicted (catastrophic)
      - Cache layers will be lost
      - Transaction processing may be disrupted
      
      IMMEDIATE MITIGATION:
      1. Identify and evict non-critical pods manually
      2. Scale down non-essential workloads
      3. Add nodes to cluster if general capacity issue
      
      LONG-TERM:
      - Review pod memory limits
      - Implement VPA (Vertical Pod Autoscaler)
      - Consider larger node types
    runbook_url: "https://runbooks.bank.internal/sre/gke/node-memory-pressure"
    incident_severity: "P1"

# CRITICAL: Node Disk Pressure
- alert: NodeDiskPressure
  expr: |
    kube_node_status_condition{condition="DiskPressure",status="true"} == 1
  for: 5m
  labels:
    severity: critical
    component: node
    impact: eviction-risk
  annotations:
    summary: "Node {{ $labels.node }} under disk pressure"
    description: |
      CRITICAL: Node running out of disk space
      
      Node: {{ $labels.node }}
      Condition: DiskPressure
      
      EVICTION RISK:
      - Kubelet will evict pods to free disk space
      - Image pulls may fail
      - Log collection may fail
      
      INVESTIGATION:
      1. Check disk usage on node:
         gcloud compute ssh {{ $labels.node }} -- df -h
      2. Check Docker/containerd disk usage:
         gcloud compute ssh {{ $labels.node }} -- sudo du -sh /var/lib/docker /var/lib/containerd
      3. Identify large directories:
         gcloud compute ssh {{ $labels.node }} -- sudo du -sh /var/* | sort -h
      
      COMMON CAUSES:
      - Excessive container logs
      - Unused Docker images
      - Orphaned volumes
      - Application writing to container filesystem
      
      IMMEDIATE REMEDIATION:
      1. Clean up container images:
         gcloud compute ssh {{ $labels.node }} -- sudo crictl rmi --prune
      2. Clean up unused volumes
      3. Rotate/compress logs
      
      BANKING COMPLIANCE:
      - Ensure audit logs are preserved before cleanup
      - May need to export logs to GCS before deletion
    runbook_url: "https://runbooks.bank.internal/sre/gke/node-disk-pressure"
    incident_severity: "P1"

# WARNING: Kubelet PLEG High Latency
- alert: KubeletPLEGHighLatency
  expr: |
    histogram_quantile(0.95,
      sum(rate(kubelet_pleg_relist_duration_seconds_bucket[10m])) 
      by (le, node)
    ) > 1.0
  for: 5m
  labels:
    severity: warning
    component: kubelet
    impact: performance-degradation
  annotations:
    summary: "Kubelet PLEG latency high on {{ $labels.node }}"
    description: |
      WARNING: Pod Lifecycle Event Generator (PLEG) slow
      
      Node: {{ $labels.node }}
      P95 PLEG Latency: {{ $value | humanizeDuration }}
      Threshold: 1.0s
      
      PERFORMANCE IMPACT:
      - Slow pod status updates
      - Delayed pod startup/termination
      - Increased kubelet CPU usage
      - Potential NotReady node condition
      
      INVESTIGATION:
      1. Check number of pods on node:
         kubectl get pods --all-namespaces -o wide | grep {{ $labels.node }} | wc -l
      2. Check container runtime performance:
         - High container churn
         - Container runtime issues
      3. Review node CPU/disk I/O
      4. Check for excessive container creation/deletion
      
      ROOT CAUSES:
      - Too many pods on single node
      - Slow container runtime operations
      - Disk I/O saturation
      - High pod churn rate
      
      MITIGATION:
      - Consider reducing pods per node
      - Improve container image layer caching
      - Ensure adequate disk I/O performance
    runbook_url: "https://runbooks.bank.internal/sre/gke/kubelet-pleg"

# WARNING: High Node CPU Utilization
- alert: NodeCPUUtilizationHigh
  expr: |
    (
      sum(rate(container_cpu_usage_seconds_total{id="/"}[5m])) by (node)
      /
      sum(kube_node_status_allocatable{resource="cpu"}) by (node)
    ) > 0.85
  for: 15m
  labels:
    severity: warning
    component: node
    impact: performance
  annotations:
    summary: "High CPU utilization on {{ $labels.node }}"
    description: |
      WARNING: Node CPU utilization high
      
      Node: {{ $labels.node }}
      CPU Utilization: {{ $value | humanizePercentage }}
      Threshold: 85%
      
      PERFORMANCE CONCERNS:
      - Reduced headroom for spikes
      - CPU throttling risk
      - Slower application response times
      
      INVESTIGATION:
      1. Identify CPU-intensive pods:
         kubectl top pods --all-namespaces | grep {{ $labels.node }} | sort -k3 -h
      2. Check for CPU throttling:
         Look at container_cpu_cfs_throttled_seconds_total
      3. Review pod CPU requests vs limits
      
      CAPACITY PLANNING:
      - May need additional nodes
      - Consider HPA for applicable workloads
      - Review CPU resource allocation
      
      BANKING CONTEXT:
      - High CPU during peak hours (month-end) is expected
      - Outside peak hours may indicate issues
    runbook_url: "https://runbooks.bank.internal/sre/gke/node-cpu-high"

# CRITICAL: Node Memory Utilization Critical
- alert: NodeMemoryUtilizationCritical
  expr: |
    (
      sum(container_memory_working_set_bytes{id="/"}) by (node)
      /
      sum(kube_node_status_allocatable{resource="memory"}) by (node)
    ) > 0.90
  for: 5m
  labels:
    severity: critical
    component: node
    impact: stability
  annotations:
    summary: "Critical memory utilization on {{ $labels.node }}"
    description: |
      CRITICAL: Node memory critically high
      
      Node: {{ $labels.node }}
      Memory Utilization: {{ $value | humanizePercentage }}
      Threshold: 90%
      
      IMMEDIATE RISK:
      - Memory pressure imminent
      - OOM kills likely
      - Pod evictions will start
      
      IMMEDIATE ACTIONS:
      1. Identify memory-intensive pods
      2. Check for memory leaks
      3. Consider emergency pod eviction of non-critical workloads
      4. Add nodes if cluster-wide issue
      
      See NodeMemoryPressure runbook for detailed steps
    runbook_url: "https://runbooks.bank.internal/sre/gke/node-memory-critical"
    incident_severity: "P1"
```

**Detailed Rationale**:

1. **Why 5-minute threshold for NotReady?**
   - Allows time for transient network issues to resolve
   - GKE node auto-repair typically takes longer
   - Prevents alert fatigue from brief disruptions

2. **Why separate alerts for pressure conditions?**
   - Each pressure type has different remediation
   - Memory pressure = immediate eviction risk
   - Disk pressure = gradual but predictable
   - Different on-call responses required

3. **Why track PLEG latency?**
   - PLEG issues are early warning sign of node problems
   - Often precedes NotReady condition
   - Indicates container runtime or disk performance issues

---
