# Part 3: Platform Component SLOs (MUST-HAVE - Tier 1 Critical)

## Executive Context for Banking Infrastructure

Platform components (ArgoCD, monitoring exporters, node problem detectors, security tools) are **mission-critical infrastructure** in banking environments because:

1. **Regulatory Compliance**: Audit trails, security controls, and change management are non-negotiable
2. **GitOps = Audit Trail**: ArgoCD provides immutable deployment history required by SOX/PCI-DSS
3. **Early Warning Systems**: Node problem detectors and exporters prevent incidents before they impact customers
4. **Security Enforcement**: Security controllers are the last line of defense before production

**Key Principle**: Platform component failure = **delayed incident detection/response** = **extended customer impact** = **regulatory violations**

---

## 2.1 GitOps / ArgoCD SLOs

**SLO Definition**:
- 99.9% ArgoCD API availability
- Application sync success rate: 99.5%
- Sync latency P95 < 60 seconds
- Drift detection latency < 5 minutes
- Webhook delivery success: 99.9%

**Business Rationale**:
- **Deployment Pipeline**: ArgoCD is the ONLY approved deployment mechanism (regulatory requirement)
- **Emergency Rollbacks**: Failed ArgoCD = cannot deploy fixes during incidents
- **Audit Trail**: Every deployment must be tracked via ArgoCD for SOX compliance
- **Drift Detection**: Unauthorized changes must be detected within 5 minutes (security requirement)
- **Change Freeze**: During month-end/quarter-end, ArgoCD stability is critical for emergency patches

**Banking-Specific Context**:
- Month-end processing windows require guaranteed deployment capability
- Security patches must be deployable within 24 hours (compliance SLA)
- Failed deployments during trading hours = millions in potential losses
- Audit trail gaps = regulatory findings

---

### PromQL Queries for ArgoCD Monitoring

```promql
# ============================================
# ARGOCD API SERVER HEALTH
# ============================================

# ArgoCD API Server Availability
(
  sum(up{job="argocd-server"})
  /
  count(up{job="argocd-server"})
) * 100

# ArgoCD API Request Success Rate (5m window)
(
  sum(rate(argocd_server_api_requests_total{code=~"2.."}[5m]))
  /
  sum(rate(argocd_server_api_requests_total[5m]))
) * 100

# ArgoCD API Request Success Rate (30d SLO window)
(
  sum(rate(argocd_server_api_requests_total{code=~"2.."}[30d]))
  /
  sum(rate(argocd_server_api_requests_total[30d]))
) * 100

# ArgoCD API Error Rate by Status Code
sum(rate(argocd_server_api_requests_total{code!~"2.."}[5m])) 
by (code) 

# ArgoCD API Request Rate (QPS)
sum(rate(argocd_server_api_requests_total[5m]))

# ArgoCD API Latency P50
histogram_quantile(0.50,
  sum(rate(argocd_server_api_request_duration_seconds_bucket[5m])) 
  by (le, method)
)

# ArgoCD API Latency P95
histogram_quantile(0.95,
  sum(rate(argocd_server_api_request_duration_seconds_bucket[5m])) 
  by (le, method)
)

# ArgoCD API Latency P99
histogram_quantile(0.99,
  sum(rate(argocd_server_api_request_duration_seconds_bucket[5m])) 
  by (le, method)
)

# ============================================
# APPLICATION SYNC METRICS
# ============================================

# Application Sync Success Rate (5m window)
(
  sum(rate(argocd_app_sync_total{phase="Succeeded"}[5m]))
  /
  sum(rate(argocd_app_sync_total[5m]))
) * 100

# Application Sync Success Rate (7d rolling for weekly reports)
(
  sum(rate(argocd_app_sync_total{phase="Succeeded"}[7d]))
  /
  sum(rate(argocd_app_sync_total[7d]))
) * 100

# Application Sync Failures by Phase
sum(rate(argocd_app_sync_total{phase!="Succeeded"}[5m])) 
by (phase, namespace)

# Currently Syncing Applications
count(argocd_app_info{sync_status="Syncing"})

# Applications Out of Sync (Drift Detection)
count(argocd_app_info{sync_status="OutOfSync"})

# Applications in Degraded State
count(argocd_app_info{health_status="Degraded"})

# Applications in Progressing State
count(argocd_app_info{health_status="Progressing"})

# Applications in Unknown Health State
count(argocd_app_info{health_status="Unknown"})

# Application Sync Duration P50
histogram_quantile(0.50,
  sum(rate(argocd_app_sync_duration_seconds_bucket[5m])) 
  by (le, namespace, name)
)

# Application Sync Duration P95
histogram_quantile(0.95,
  sum(rate(argocd_app_sync_duration_seconds_bucket[5m])) 
  by (le, namespace, name)
)

# Application Sync Duration P99
histogram_quantile(0.99,
  sum(rate(argocd_app_sync_duration_seconds_bucket[5m])) 
  by (le, namespace, name)
)

# ============================================
# GIT REPOSITORY OPERATIONS
# ============================================

# Git Repository Request Success Rate
(
  sum(rate(argocd_git_request_total{request_type="fetch"}[5m])) 
  - 
  sum(rate(argocd_git_request_total{request_type="fetch",error!=""}[5m]))
)
/
sum(rate(argocd_git_request_total{request_type="fetch"}[5m])) * 100

# Git Repository Fetch Latency P95
histogram_quantile(0.95,
  sum(rate(argocd_git_request_duration_seconds_bucket{request_type="fetch"}[5m])) 
  by (le, repo)
)

# Git Repository Errors
sum(rate(argocd_git_request_total{error!=""}[5m])) 
by (repo, error)

# ============================================
# ARGOCD APPLICATION CONTROLLER METRICS
# ============================================

# Application Controller Queue Depth
argocd_app_reconcile_queue_depth

# Application Reconciliation Rate
sum(rate(argocd_app_reconcile_total[5m]))

# Application Reconciliation Duration P95
histogram_quantile(0.95,
  sum(rate(argocd_app_reconcile_duration_seconds_bucket[5m])) 
  by (le, namespace, name)
)

# Pending Application Operations
argocd_app_pending_request_total

# ============================================
# ARGOCD REPOSITORY SERVER METRICS
# ============================================

# Repository Server Up
up{job="argocd-repo-server"}

# Repository Server CPU Usage
rate(process_cpu_seconds_total{job="argocd-repo-server"}[5m])

# Repository Server Memory Usage
process_resident_memory_bytes{job="argocd-repo-server"}

# ============================================
# ARGOCD NOTIFICATION METRICS
# ============================================

# Webhook Delivery Success Rate
(
  sum(rate(argocd_notifications_deliveries_total{succeeded="true"}[5m]))
  /
  sum(rate(argocd_notifications_deliveries_total[5m]))
) * 100

# Failed Webhook Deliveries
sum(rate(argocd_notifications_deliveries_total{succeeded="false"}[5m])) 
by (service, destination)

# Notification Trigger Rate
sum(rate(argocd_notifications_trigger_eval_total[5m])) 
by (name)

# ============================================
# ARGOCD RESOURCE UTILIZATION
# ============================================

# ArgoCD Server Pod Count
count(kube_pod_info{
  namespace="argocd",
  pod=~"argocd-server-.*"
})

# ArgoCD Server CPU Usage
sum(rate(container_cpu_usage_seconds_total{
  namespace="argocd",
  pod=~"argocd-server-.*",
  container="argocd-server"
}[5m]))

# ArgoCD Server Memory Usage
sum(container_memory_working_set_bytes{
  namespace="argocd",
  pod=~"argocd-server-.*",
  container="argocd-server"
}) / (1024 * 1024 * 1024)  # GB

# ArgoCD Application Controller CPU Usage
sum(rate(container_cpu_usage_seconds_total{
  namespace="argocd",
  pod=~"argocd-application-controller-.*",
  container="argocd-application-controller"
}[5m]))

# ArgoCD Application Controller Memory Usage
sum(container_memory_working_set_bytes{
  namespace="argocd",
  pod=~"argocd-application-controller-.*",
  container="argocd-application-controller"
}) / (1024 * 1024 * 1024)  # GB

# ============================================
# ARGOCD AUDIT AND COMPLIANCE
# ============================================

# Failed Login Attempts (Security)
sum(rate(argocd_server_login_attempts_total{status="failed"}[5m]))

# Successful Deployments by User (Audit Trail)
sum(rate(argocd_app_sync_total{phase="Succeeded"}[1h])) 
by (user) 

# Manual Sync Operations (Should be rare in production)
sum(rate(argocd_app_sync_total{initiator=~".*manual.*"}[1h]))

# Auto-Sync Disabled Applications (Policy Violation)
count(argocd_app_info{autosync_enabled="false"})
```

---

### Alert Definitions for ArgoCD

```promql
# ============================================
# CRITICAL: ARGOCD AVAILABILITY ALERTS
# ============================================

# CRITICAL: ArgoCD API Server Down
- alert: ArgoCDAPIServerDown
  expr: |
    up{job="argocd-server"} == 0
  for: 2m
  labels:
    severity: critical
    component: argocd
    impact: deployment-blocked
    compliance_risk: critical
  annotations:
    summary: "ArgoCD API server is down"
    description: |
      CRITICAL: ArgoCD API server unavailable
      
      Instance: {{ $labels.instance }}
      
      DEPLOYMENT PIPELINE BLOCKED:
      - Cannot deploy any applications
      - Cannot perform emergency rollbacks
      - Cannot view application status
      - GitOps workflow completely blocked
      
      BANKING IMPACT:
      - Emergency security patches cannot be deployed
      - Incident remediation delayed
      - Change freeze violations if forced manual deployments
      - SOX compliance: Deployment audit trail broken
      
      IMMEDIATE INVESTIGATION:
      1. Check ArgoCD server pod status:
         kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server
      2. Check pod logs:
         kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server --tail=100
      3. Check pod events:
         kubectl describe pods -n argocd -l app.kubernetes.io/name=argocd-server
      4. Verify service and ingress configuration
      
      COMMON CAUSES:
      - Pod CrashLoopBackOff (check resource limits)
      - Redis connection failures
      - RBAC/authentication misconfiguration
      - Certificate expiration
      - Database connectivity issues
      
      EMERGENCY PROCEDURES:
      - Do NOT bypass ArgoCD for deployments (audit violation)
      - Establish incident bridge
      - Prepare change control documentation if manual intervention needed
      - Notify compliance team if outage exceeds 30 minutes
      
      ESCALATION:
      - P1 incident during business hours
      - P0 incident if security patch deployment required
    runbook_url: "https://runbooks.bank.internal/sre/gke/argocd-api-down"
    incident_severity: "P1"
    compliance_notification: "required"

# CRITICAL: ArgoCD Application Controller Down
- alert: ArgoCDApplicationControllerDown
  expr: |
    up{job="argocd-application-controller"} == 0
  for: 2m
  labels:
    severity: critical
    component: argocd
    impact: sync-blocked
    compliance_risk: critical
  annotations:
    summary: "ArgoCD application controller is down"
    description: |
      CRITICAL: ArgoCD application controller unavailable
      
      GITOPS RECONCILIATION STOPPED:
      - Applications not syncing
      - Drift detection disabled
      - Auto-sync not functioning
      - Health checks not running
      
      SECURITY IMPACT:
      - Unauthorized changes NOT detected
      - Configuration drift accumulating
      - Security policy violations undetected
      
      COMPLIANCE RISK:
      - Cannot demonstrate continuous compliance
      - Drift detection is regulatory requirement
      - Audit finding likely if extended outage
      
      IMMEDIATE INVESTIGATION:
      1. Check controller pod:
         kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-application-controller
      2. Check controller logs
      3. Review resource utilization (often memory exhaustion)
      4. Check Redis connectivity
      
      COMMON CAUSES:
      - OOMKilled (controller is memory-intensive)
      - Too many applications for single controller
      - Git repository connectivity issues
      - Kubernetes API server connectivity
      
      MITIGATION:
      - Increase controller memory limits
      - Consider sharding (multiple controllers)
      - Reduce reconciliation frequency temporarily
    runbook_url: "https://runbooks.bank.internal/sre/gke/argocd-controller-down"
    incident_severity: "P1"
    compliance_notification: "required"

# CRITICAL: High ArgoCD API Error Rate
- alert: ArgoCDAPIHighErrorRate
  expr: |
    (
      sum(rate(argocd_server_api_requests_total{code!~"2.."}[10m]))
      /
      sum(rate(argocd_server_api_requests_total[10m]))
    ) > 0.05  # 5% error rate
  for: 5m
  labels:
    severity: warning
    component: argocd
    impact: degraded-experience
  annotations:
    summary: "ArgoCD API error rate at {{ $value | humanizePercentage }}"
    description: |
      WARNING: ArgoCD API experiencing errors
      
      Error Rate: {{ $value | humanizePercentage }}
      Threshold: 5%
      
      DEGRADED FUNCTIONALITY:
      - UI may be slow or failing
      - CLI operations may fail
      - Webhook deliveries may fail
      
      ERROR BREAKDOWN:
      {{- range query "topk(5, sum(rate(argocd_server_api_requests_total{code!~'2..'}[10m])) by (code))" }}
      - HTTP {{ .Labels.code }}: {{ .Value | humanize }}/sec
      {{- end }}
      
      INVESTIGATION:
      1. Check error types (4xx vs 5xx)
      2. Review API server logs
      3. Check authentication/authorization issues
      4. Verify backend connectivity (Git, K8s API)
      
      BANKING CONTEXT:
      - May affect deployment pipeline during critical windows
      - Check if coincides with deployment schedule
    runbook_url: "https://runbooks.bank.internal/sre/gke/argocd-api-errors"

# ============================================
# APPLICATION SYNC ALERTS
# ============================================

# CRITICAL: Application Sync Failure Rate High
- alert: ArgoCDSyncFailureRateHigh
  expr: |
    (
      sum(rate(argocd_app_sync_total{phase!="Succeeded"}[30m]))
      /
      sum(rate(argocd_app_sync_total[30m]))
    ) > 0.05  # 5% failure rate
  for: 10m
  labels:
    severity: warning
    component: argocd
    impact: deployment-failures
  annotations:
    summary: "ArgoCD sync failure rate at {{ $value | humanizePercentage }}"
    description: |
      WARNING: High application sync failure rate
      
      Failure Rate: {{ $value | humanizePercentage }}
      Threshold: 5%
      SLO Target: 99.5% success rate
      
      DEPLOYMENT PIPELINE DEGRADED:
      - Multiple applications failing to sync
      - Deployments being blocked
      - Potential configuration issues
      
      FAILURE BREAKDOWN:
      {{- range query "topk(5, sum(rate(argocd_app_sync_total{phase!='Succeeded'}[30m])) by (phase))" }}
      - {{ .Labels.phase }}: {{ .Value | humanize }}/min
      {{- end }}
      
      TOP FAILING APPLICATIONS:
      {{- range query "topk(5, sum(rate(argocd_app_sync_total{phase!='Succeeded'}[30m])) by (namespace, name))" }}
      - {{ .Labels.namespace }}/{{ .Labels.name }}
      {{- end }}
      
      INVESTIGATION:
      1. Review ArgoCD UI for sync errors
      2. Check application event logs
      3. Verify manifest syntax/validation
      4. Check resource quotas and limits
      5. Review RBAC permissions
      
      COMMON CAUSES:
      - Invalid Kubernetes manifests
      - Resource quota exhaustion
      - RBAC permission issues
      - Helm chart templating errors
      - Git repository access issues
      
      BANKING IMPACT:
      - Check if production apps affected
      - Assess blast radius of failed deployments
      - Coordinate with app teams
    runbook_url: "https://runbooks.bank.internal/sre/gke/argocd-sync-failures"

# WARNING: Application Sync Duration Slow
- alert: ArgoCDSyncDurationSlow
  expr: |
    histogram_quantile(0.95,
      sum(rate(argocd_app_sync_duration_seconds_bucket[15m])) 
      by (le)
    ) > 60  # 60 seconds
  for: 10m
  labels:
    severity: warning
    component: argocd
    impact: slow-deployments
  annotations:
    summary: "ArgoCD P95 sync duration exceeds 60s: {{ $value | humanizeDuration }}"
    description: |
      WARNING: Application syncs taking too long
      
      P95 Sync Duration: {{ $value | humanizeDuration }}
      SLO Target: <60s
      
      DEPLOYMENT VELOCITY IMPACT:
      - Slower rollouts
      - Extended deployment windows
      - Delayed incident response
      
      SLOWEST APPLICATIONS:
      {{- range query "topk(5, histogram_quantile(0.95, sum(rate(argocd_app_sync_duration_seconds_bucket[15m])) by (le, namespace, name)))" }}
      - {{ .Labels.namespace }}/{{ .Labels.name }}: {{ .Value | humanizeDuration }}
      {{- end }}
      
      INVESTIGATION:
      1. Identify slow applications (large manifests?)
      2. Check Git repository performance
      3. Review Kubernetes API server latency
      4. Check for resource generation overhead (Helm, Kustomize)
      
      OPTIMIZATION:
      - Consider breaking large applications into smaller ones
      - Optimize Helm chart complexity
      - Review sync wave configuration
      - Check for unnecessary pre-sync hooks
    runbook_url: "https://runbooks.bank.internal/sre/gke/argocd-sync-slow"

# CRITICAL: Multiple Applications Out of Sync
- alert: ArgoCDMultipleAppsOutOfSync
  expr: |
    count(argocd_app_info{sync_status="OutOfSync"}) > 10
  for: 15m
  labels:
    severity: warning
    component: argocd
    impact: drift-detected
    security_risk: medium
  annotations:
    summary: "{{ $value }} ArgoCD applications out of sync"
    description: |
      WARNING: Significant configuration drift detected
      
      Out of Sync Applications: {{ $value }}
      Threshold: 10
      Duration: 15+ minutes
      
      CONFIGURATION DRIFT:
      - Applications diverging from Git
      - Potential unauthorized changes
      - Auto-sync may be disabled or failing
      
      SECURITY IMPLICATIONS:
      - Unauthorized changes may be active
      - Security policies may be violated
      - Compliance drift accumulating
      
      INVESTIGATION:
      1. List out-of-sync apps:
         argocd app list --sync-status OutOfSync
      2. Check if auto-sync disabled:
         Look for manual sync policies
      3. Review recent cluster changes:
         kubectl get events --all-namespaces --sort-by='.lastTimestamp'
      4. Check for failed sync attempts
      
      OUT OF SYNC APPLICATIONS:
      {{- range query "argocd_app_info{sync_status='OutOfSync'}" }}
      - {{ .Labels.namespace }}/{{ .Labels.name }}
      {{- end }}
      
      REMEDIATION:
      - For legitimate drift: Update Git to match cluster
      - For unauthorized changes: Sync from Git (overwrites cluster)
      - For auto-sync failures: Investigate and fix sync errors
      
      BANKING COMPLIANCE:
      - Document all intentional drift
      - Investigate unauthorized changes immediately
      - Update change control records
    runbook_url: "https://runbooks.bank.internal/sre/gke/argocd-drift"
    security_review: "required"

# CRITICAL: Applications in Degraded Health
- alert: ArgoCDAppsDegraded
  expr: |
    count(argocd_app_info{health_status="Degraded"}) > 5
  for: 10m
  labels:
    severity: warning
    component: argocd
    impact: unhealthy-apps
  annotations:
    summary: "{{ $value }} ArgoCD applications in degraded health"
    description: |
      WARNING: Multiple applications unhealthy
      
      Degraded Applications: {{ $value }}
      Threshold: 5
      
      APPLICATION HEALTH ISSUES:
      - Resources not ready
      - Pods crashing or pending
      - Jobs failing
      - Health checks failing
      
      DEGRADED APPLICATIONS:
      {{- range query "argocd_app_info{health_status='Degraded'}" }}
      - {{ .Labels.namespace }}/{{ .Labels.name }}
      {{- end }}
      
      INVESTIGATION:
      1. Review application details in ArgoCD UI
      2. Check pod status for each app
      3. Review application logs
      4. Check for resource constraints
      
      BANKING IMPACT:
      - Determine if customer-facing services affected
      - Assess transaction processing impact
      - Coordinate with application owners
    runbook_url: "https://runbooks.bank.internal/sre/gke/argocd-apps-degraded"

# ============================================
# GIT REPOSITORY OPERATION ALERTS
# ============================================

# CRITICAL: Git Repository Fetch Failures
- alert: ArgoCDGitFetchFailures
  expr: |
    sum(rate(argocd_git_request_total{request_type="fetch",error!=""}[10m])) > 0.1
  for: 5m
  labels:
    severity: critical
    component: argocd
    impact: git-connectivity
  annotations:
    summary: "ArgoCD experiencing Git repository fetch failures"
    description: |
      CRITICAL: Cannot fetch from Git repositories
      
      Fetch Failure Rate: {{ $value }}/sec
      
      GITOPS PIPELINE BROKEN:
      - Cannot sync applications
      - Cannot detect configuration changes
      - Deployments blocked
      
      AFFECTED REPOSITORIES:
      {{- range query "sum(rate(argocd_git_request_total{request_type='fetch',error!=''}[10m])) by (repo)" }}
      - {{ .Labels.repo }}
      {{- end }}
      
      ERROR TYPES:
      {{- range query "sum(rate(argocd_git_request_total{request_type='fetch',error!=''}[10m])) by (error)" }}
      - {{ .Labels.error }}: {{ .Value | humanize }}/sec
      {{- end }}
      
      INVESTIGATION:
      1. Check Git provider status (GitHub, GitLab, etc.)
      2. Verify network connectivity to Git provider
      3. Check Git credentials/SSH keys validity
      4. Review firewall/proxy configuration
      5. Check rate limiting from Git provider
      
      COMMON CAUSES:
      - Expired credentials/tokens
      - Network connectivity issues
      - Git provider outage
      - Rate limiting
      - Certificate expiration
      
      IMMEDIATE MITIGATION:
      - Verify credential secrets in ArgoCD namespace
      - Check if manual git clone works from cluster
      - Review recent credential rotations
      
      BANKING IMPACT:
      - Deployment pipeline completely blocked
      - Cannot deploy emergency fixes
      - Escalate if security patch deployment required
    runbook_url: "https://runbooks.bank.internal/sre/gke/argocd-git-fetch-failures"
    incident_severity: "P1"

# WARNING: Git Repository Fetch Latency High
- alert: ArgoCDGitFetchLatencyHigh
  expr: |
    histogram_quantile(0.95,
      sum(rate(argocd_git_request_duration_seconds_bucket{request_type="fetch"}[10m])) 
      by (le)
    ) > 10  # 10 seconds
  for: 10m
  labels:
    severity: warning
    component: argocd
    impact: slow-git-operations
  annotations:
    summary: "ArgoCD Git fetch latency high: {{ $value | humanizeDuration }}"
    description: |
      WARNING: Slow Git repository operations
      
      P95 Fetch Latency: {{ $value | humanizeDuration }}
      Threshold: 10s
      
      PERFORMANCE IMPACT:
      - Slow application syncs
      - Delayed drift detection
      - Sluggish ArgoCD UI
      
      SLOWEST REPOSITORIES:
      {{- range query "topk(5, histogram_quantile(0.95, sum(rate(argocd_git_request_duration_seconds_bucket{request_type='fetch'}[10m])) by (le, repo)))" }}
      - {{ .Labels.repo }}: {{ .Value | humanizeDuration }}
      {{- end }}
      
      INVESTIGATION:
      1. Check Git provider performance
      2. Review repository sizes (large repos slow)
      3. Check network latency to Git provider
      4. Review ArgoCD repo server resources
      
      OPTIMIZATION:
      - Consider using shallow clones
      - Review repository structure (monorepo vs multi-repo)
      - Check if repository caching working properly
      - Consider Git LFS for large binaries
    runbook_url: "https://runbooks.bank.internal/sre/gke/argocd-git-latency"

# ============================================
# ARGOCD RESOURCE UTILIZATION ALERTS
# ============================================

# WARNING: ArgoCD Application Controller Memory High
- alert: ArgoCDControllerMemoryHigh
  expr: |
    (
      sum(container_memory_working_set_bytes{
        namespace="argocd",
        pod=~"argocd-application-controller-.*",
        container="argocd-application-controller"
      })
      /
      sum(kube_pod_container_resource_limits{
        namespace="argocd",
        pod=~"argocd-application-controller-.*",
        container="argocd-application-controller",
        resource="memory"
      })
    ) > 0.85
  for: 10m
  labels:
    severity: warning
    component: argocd
    impact: oom-risk
  annotations:
    summary: "ArgoCD controller memory at {{ $value | humanizePercentage }} of limit"
    description: |
      WARNING: ArgoCD controller approaching memory limit
      
      Memory Usage: {{ $value | humanizePercentage }}
      Threshold: 85%
      
      OOM RISK:
      - Controller may be OOMKilled
      - Would stop all application syncs
      - Would disable drift detection
      
      INVESTIGATION:
      1. Check number of managed applications:
         argocd app list | wc -l
      2. Review controller memory trends
      3. Check for memory leaks
      4. Review application complexity (large manifests)
      
      MITIGATION:
      - Increase controller memory limits
      - Consider application sharding
      - Reduce reconciliation frequency
      - Review caching configuration
      
      BANKING CONTEXT:
      - Controller manages {{  with query "count(argocd_app_info)" }}{{ . | first | value }}{{- end }} applications
      - OOM during deployment window would be critical
    runbook_url: "https://runbooks.bank.internal/sre/gke/argocd-controller-memory"

# WARNING: ArgoCD Reconciliation Queue Depth High
- alert: ArgoCDReconciliationQueueDepthHigh
  expr: |
    argocd_app_reconcile_queue_depth > 100
  for: 15m
  labels:
    severity: warning
    component: argocd
    impact: slow-reconciliation
  annotations:
    summary: "ArgoCD reconciliation queue depth at {{ $value }}"
    description: |
      WARNING: ArgoCD controller backlog building
      
      Queue Depth: {{ $value }}
      Threshold: 100
      
      PERFORMANCE DEGRADATION:
      - Slow application updates
      - Delayed drift detection
      - Increased sync latency
      
      INVESTIGATION:
      1. Check controller CPU/memory utilization
      2. Review recent application additions
      3. Check for stuck reconciliations
      4. Review Git repository performance
      
      CAPACITY PLANNING:
      - May need to scale controller resources
      - Consider application sharding
      - Review reconciliation timeout settings
      
      IMPACT:
      - Applications may take longer to sync
      - Drift detection delayed
      - Manual syncs may be queued
    runbook_url: "https://runbooks.bank.internal/sre/gke/argocd-queue-depth"

# ============================================
# WEBHOOK AND NOTIFICATION ALERTS
# ============================================

# WARNING: ArgoCD Webhook Delivery Failures
- alert: ArgoCDWebhookDeliveryFailures
  expr: |
    (
      sum(rate(argocd_notifications_deliveries_total{succeeded="false"}[15m]))
      /
      sum(rate(argocd_notifications_deliveries_total[15m]))
    ) > 0.05  # 5% failure rate
  for: 10m
  labels:
    severity: warning
    component: argocd
    impact: notification-failures
  annotations:
    summary: "ArgoCD webhook delivery failure rate at {{ $value | humanizePercentage }}"
    description: |
      WARNING: ArgoCD notifications failing
      
      Failure Rate: {{ $value | humanizePercentage }}
      SLO Target: 99.9% success
      
      NOTIFICATION IMPACT:
      - Teams not notified of sync events
      - Deployment notifications missing
      - Alert delivery failures
      
      AFFECTED DESTINATIONS:
      {{- range query "sum(rate(argocd_notifications_deliveries_total{succeeded='false'}[15m])) by (service, destination)" }}
      - {{ .Labels.service }}/{{ .Labels.destination }}
      {{- end }}
      
      INVESTIGATION:
      1. Check notification service availability (Slack, PagerDuty, etc.)
      2. Verify webhook credentials/tokens
      3. Check network connectivity
      4. Review notification controller logs
      
      BANKING IMPACT:
      - Deployment teams may miss critical updates
      - On-call not paged for sync failures
      - Compliance: Notification audit trail incomplete
    runbook_url: "https://runbooks.bank.internal/sre/gke/argocd-webhook-failures"

# ============================================
# SECURITY AND COMPLIANCE ALERTS
# ============================================

# WARNING: High Failed Login Attempts
- alert: ArgoCDHighFailedLogins
  expr: |
    sum(rate(argocd_server_login_attempts_total{status="failed"}[15m])) > 0.5
  for: 10m
  labels:
    severity: warning
    component: argocd
    security_risk: high
  annotations:
    summary: "High ArgoCD failed login attempts: {{ $value }}/sec"
    description: |
      WARNING: Potential brute force attack on ArgoCD
      
      Failed Login Rate: {{ $value }}/sec
      Threshold: 0.5/sec
      
      SECURITY CONCERN:
      - Potential credential stuffing attack
      - Possible compromised credentials
      - Unauthorized access attempts
      
      IMMEDIATE INVESTIGATION:
      1. Review ArgoCD audit logs
      2. Identify source IPs of failed attempts
      3. Check for credential leaks
      4. Review user accounts for compromise
      
      SECURITY ACTIONS:
      - Consider IP blocking if from specific sources
      - Review MFA configuration
      - Check for credential leaks in public repos
      - Enforce strong password policies
      
      BANKING SECURITY:
      - ArgoCD has deployment privileges
      - Compromise = potential production access
      - Notify security team immediately
      - Consider locking affected accounts
    runbook_url: "https://runbooks.bank.internal/sre/gke/argocd-failed-logins"
    security_team_notification: "required"
    incident_severity: "P2"

# WARNING: Manual Sync Operations Detected
- alert: ArgoCDManualSyncOperations
  expr: |
    sum(increase(argocd_app_sync_total{initiator=~".*manual.*"}[1h])) > 10
  for: 5m
  labels:
    severity: warning
    component: argocd
    compliance_risk: medium
  annotations:
    summary: "{{ $value }} manual ArgoCD sync operations in last hour"
    description: |
      WARNING: High manual sync activity detected
      
      Manual Syncs (1h): {{ $value }}
      Threshold: 10
      
      POLICY DEVIATION:
      - Auto-sync should be standard
      - Manual syncs indicate:
        * Auto-sync disabled
        * Sync failures requiring manual intervention
        * Bypass of automated controls
      
      COMPLIANCE CONCERN:
      - Manual syncs may bypass approval workflows
      - Audit trail may be incomplete
      - Change management process violated
      
      INVESTIGATION:
      1. Identify which applications manually synced
      2. Determine sync initiators (users)
      3. Review reasons for manual intervention
      4. Check if auto-sync disabled intentionally
      
      MANUAL SYNCS BY APPLICATION:
      {{- range query "topk(5, sum(increase(argocd_app_sync_total{initiator=~'.*manual.*'}[1h])) by (namespace, name))" }}
      - {{ .Labels.namespace }}/{{ .Labels.name }}: {{ .Value }}
      {{- end }}
      
      REMEDIATION:
      - Re-enable auto-sync where appropriate
      - Document approved manual sync scenarios
      - Review change control procedures
      
      BANKING COMPLIANCE:
      - May require change control exception documentation
      - Audit team may require justification
    runbook_url: "https://runbooks.bank.internal/sre/gke/argocd-manual-syncs"
    compliance_review: "recommended"

# WARNING: Auto-Sync Disabled on Production Apps
- alert: ArgoCDAutoSyncDisabled
  expr: |
    count(argocd_app_info{
      autosync_enabled="false",
      namespace=~"prod-.*|production"
    }) > 0
  for: 30m
  labels:
    severity: warning
    component: argocd
    compliance_risk: medium
  annotations:
    summary: "{{ $value }} production applications have auto-sync disabled"
    description: |
      WARNING: Production applications not auto-syncing
      
      Applications: {{ $value }}
      
      GITOPS VIOLATION:
      - Git is not source of truth
      - Configuration drift risk
      - Manual intervention required
      
      APPLICATIONS WITH AUTO-SYNC DISABLED:
      {{- range query "argocd_app_info{autosync_enabled='false', namespace=~'prod-.*|production'}" }}
      - {{ .Labels.namespace }}/{{ .Labels.name }}
      {{- end }}
      
      POLICY REQUIREMENT:
      - Production apps MUST auto-sync (per SOX compliance)
      - Exceptions require approval
      - Temporary disable must be documented
      
      INVESTIGATION:
      1. Check if temporary for deployment window
      2. Verify if approved exception
      3. Review with application teams
      4. Document justification if legitimate
      
      REMEDIATION:
      - Re-enable auto-sync after deployment windows
      - Document any permanent exceptions
      - Update change control records
    runbook_url: "https://runbooks.bank.internal/sre/gke/argocd-autosync-disabled"
    compliance_review: "required"
```

---

## 2.2 Node Problem Detector SLOs

**SLO Definition**:
- 99.9% NPD daemonset availability (must run on all nodes)
- Problem detection latency < 60 seconds
- Alert delivery success: 99.5%

**Business Rationale**:
- **Early Warning System**: Detects node issues before they cascade (disk failures, kernel deadlocks, container runtime issues)
- **Proactive Remediation**: Alerts SREs to node problems before workload impact
- **Compliance**: Infrastructure health monitoring is regulatory requirement
- **Cost Optimization**: Identifies failing nodes early, preventing larger incidents

**Banking Context**:
- Node failures during trading hours = potential transaction losses
- Early detection prevents database pod disruptions (stateful workloads)
- Kernel issues can corrupt data - early detection critical

---

### PromQL Queries for Node Problem Detector

```promql
# ============================================
# NODE PROBLEM DETECTOR DAEMONSET HEALTH
# ============================================

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

# NPD Pods Not Ready
(
  kube_daemonset_status_desired_number_scheduled{
    namespace="kube-system",
    daemonset="node-problem-detector"
  }
  -
  kube_daemonset_status_number_ready{
    namespace="kube-system",
    daemonset="node-problem-detector"
  }
)

# NPD Pod Restarts
rate(kube_pod_container_status_restarts_total{
  namespace="kube-system",
  pod=~"node-problem-detector-.*"
}[30m])

# ============================================
# NODE PROBLEM DETECTION METRICS
# ============================================

# Active Node Problems by Type
sum(node_problem_detector_problems{type!=""}) 
by (node, type, reason)

# Problem Detection Rate
sum(rate(node_problem_detector_problems_total[5m])) 
by (type, reason)

# Nodes with Active Problems
count(node_problem_detector_problems > 0) 
by (type)

# ============================================
# SPECIFIC NODE PROBLEM TYPES
# ============================================

# Kernel Deadlock Detected
node_problem_detector_problems{type="KernelDeadlock"} > 0

# Memory Pressure Detected (from NPD)
node_problem_detector_problems{type="MemoryPressure"} > 0

# Disk Pressure Detected
node_problem_detector_problems{type="DiskPressure"} > 0

# PID Pressure Detected
node_problem_detector_problems{type="PIDPressure"} > 0

# Unregister Net Device Issues
node_problem_detector_problems{type="UnregisterNetDevice"} > 0

# Container Runtime Issues
node_problem_detector_problems{type="ContainerRuntimeUnhealthy"} > 0

# Kernel Panics
node_problem_detector_problems{type="KernelOops"} > 0

# Filesystem Corruption
node_problem_detector_problems{type="FilesystemCorruption"} > 0

# Docker Hang Detected
node_problem_detector_problems{type="DockerHang"} > 0

# NTP Service Down
node_problem_detector_problems{type="NTPProblem"} > 0

# ============================================
# NPD RESOURCE UTILIZATION
# ============================================

# NPD CPU Usage
sum(rate(container_cpu_usage_seconds_total{
  namespace="kube-system",
  pod=~"node-problem-detector-.*"
}[5m]))

# NPD Memory Usage
sum(container_memory_working_set_bytes{
  namespace="kube-system",
  pod=~"node-problem-detector-.*"
}) / (1024 * 1024)  # MB
```

### Alert Definitions for Node Problem Detector

```promql
# CRITICAL: Node Problem Detector Not Running on Nodes
- alert: NodeProblemDetectorNotRunning
  expr: |
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
    ) < 0.95
  for: 10m
  labels:
    severity: warning
    component: node-problem-detector
    impact: monitoring-gap
  annotations:
    summary: "Node Problem Detector coverage at {{ $value | humanizePercentage }}"
    description: |
      WARNING: Node Problem Detector not running on all nodes
      
      Coverage: {{ $value | humanizePercentage }}
      Target: 100% (must run on all nodes)
      
      MONITORING GAP:
      - Some nodes not monitored for problems
      - Early warning system compromised
      - Node issues may go undetected
      
      INVESTIGATION:
      1. Check NPD daemonset status:
         kubectl get daemonset -n kube-system node-problem-detector
      2. Identify nodes without NPD:
         kubectl get pods -n kube-system -l app=node-problem-detector -o wide
      3. Check pod events and logs
      
      COMMON CAUSES:
      - Node taints preventing scheduling
      - Resource constraints
      - Image pull failures
      - Node selectors mismatch
    runbook_url: "https://runbooks.bank.internal/sre/gke/npd-coverage"

# CRITICAL: Kernel Deadlock Detected
- alert: NodeKernelDeadlock
  expr: |
    node_problem_detector_problems{type="KernelDeadlock"} > 0
  for: 1m
  labels:
    severity: critical
    component: node
    impact: node-failure-imminent
  annotations:
    summary: "Kernel deadlock detected on node {{ $labels.node }}"
    description: |
      CRITICAL: Node {{ $labels.node }} kernel deadlock
      
      IMMEDIATE NODE FAILURE RISK:
      - Kernel is hung
      - Node will become NotReady
      - All pods will be evicted
      - Potential data corruption
      
      IMMEDIATE ACTIONS:
      1. Cordon node immediately:
         kubectl cordon {{ $labels.node }}
      2. Drain node (graceful eviction):
         kubectl drain {{ $labels.node }} --ignore-daemonsets --delete-emptydir-data
      3. Investigate kernel logs:
         gcloud compute ssh {{ $labels.node }} -- sudo dmesg -T | tail -100
      
      BANKING IMPACT:
      - Check for database pods on this node
      - StatefulSets will need time to reschedule
      - Coordinate with app teams before draining
      
      RESOLUTION:
      - Node must be replaced (kernel issue unlikely to self-recover)
      - Submit node for analysis if recurring issue
    runbook_url: "https://runbooks.bank.internal/sre/gke/kernel-deadlock"
    incident_severity: "P1"

# CRITICAL: Container Runtime Unhealthy
- alert: NodeContainerRuntimeUnhealthy
  expr: |
    node_problem_detector_problems{type="ContainerRuntimeUnhealthy"} > 0
  for: 5m
  labels:
    severity: critical
    component: node
    impact: pod-failures
  annotations:
    summary: "Container runtime unhealthy on node {{ $labels.node }}"
    description: |
      CRITICAL: Container runtime (containerd/docker) failing
      
      Node: {{ $labels.node }}
      Reason: {{ $labels.reason }}
      
      POD IMPACT:
      - Cannot start new containers
      - Existing containers may fail
      - Health checks may fail
      - Pod restarts will fail
      
      INVESTIGATION:
      1. Check container runtime status:
         gcloud compute ssh {{ $labels.node }} -- sudo systemctl status containerd
      2. Review runtime logs:
         gcloud compute ssh {{ $labels.node }} -- sudo journalctl -u containerd -n 100
      3. Check disk space (runtime needs disk)
      4. Review for known bugs
      
      MITIGATION:
      - Restart container runtime if safe
      - Drain and replace node if persistent
      - Check for disk space issues
    runbook_url: "https://runbooks.bank.internal/sre/gke/container-runtime-unhealthy"
    incident_severity: "P1"
```

Due to length, I'll provide the remaining platform component SLOs (kube-state-metrics, monitoring exporters, security components) in a structured summary format:

---

## 2.3 Monitoring Exporters (kube-state-metrics, node-exporter) SLOs

**SLO Definition**:
- 99.9% exporter availability
- Metrics scrape success rate: 99.5%
- Metrics staleness < 60 seconds

**Key Metrics**:
```promql
# Exporter Up
up{job=~"kube-state-metrics|node-exporter"}

# Scrape Success Rate
(sum(up{job="kube-state-metrics"}) / count(up{job="kube-state-metrics"})) * 100

# Metrics Staleness
(time() - timestamp(kube_pod_info)) > 60
```

**Critical Alerts**:
- kube-state-metrics down → no workload metrics → blind to app failures
- node-exporter down → no node metrics → blind to capacity issues
- High metrics cardinality → Prometheus performance degradation

---

## Summary Table: Platform Component SLOs Priority

| Component | SLO Tier | Availability Target | Why Critical for Banking |
|-----------|----------|---------------------|-------------------------|
| **ArgoCD API** | MUST-HAVE | 99.9% | Deployment pipeline; SOX audit trail; emergency rollbacks |
| **ArgoCD Sync** | MUST-HAVE | 99.5% success | GitOps integrity; drift detection for security compliance |
| **Node Problem Detector** | MUST-HAVE | 99.9% coverage | Early warning system; prevents data corruption in stateful workloads |
| **kube-state-metrics** | MUST-HAVE | 99.9% | Foundation for all workload monitoring; blindness = risk |
| **node-exporter** | MUST-HAVE | 99.9% | Capacity planning; node health; compliance reporting |
