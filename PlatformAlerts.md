## **Consolidated Alert 1: Critical Controllers and Operators Unavailable**

This single alert replaces your individual alerts for Gatekeeper, ArgoCD Application Controller, External Secrets, and Vault Agent Injector. The magic happens through label-based filtering in the PromQL query.

**How This Consolidation Works**

Instead of writing four separate alert rules that each watch one deployment, you write one alert rule that watches all critical deployments using a regular expression to match their names. When any of these deployments goes to zero available replicas, the alert fires with labels identifying which specific deployment failed. Your alerting system treats each deployment as a separate alert instance, so you still get granular visibility into which component is down, but you maintain only one alert definition.

The practical benefit becomes clear during incident response. Your on-call engineer receives an alert that says "Deployment gatekeeper-controller-manager in namespace gatekeeper-system has zero available replicas" and they know immediately that this is a controller unavailability issue. They don't need different runbooks for different controllers because the troubleshooting steps are the same. Check the pod status with kubectl describe, review the logs, verify certificates if it's a webhook, check resource constraints, and examine recent changes.

**PromQL Query**
```promql
# Single query that monitors all critical user-managed controllers
# Fires separate alert instances for each affected deployment
kube_deployment_status_replicas_available{
  deployment=~"gatekeeper-controller-manager|gatekeeper-audit|argocd-application-controller|argocd-repo-server|external-secrets.*|vault-agent-injector.*|alertmanager.*",
  namespace!="kube-system"
} == 0
```

Let me explain what's happening in this query. The kube_deployment_status_replicas_available metric comes from kube-state-metrics and tracks how many replicas of each deployment are currently available and passing readiness checks. The regular expression in the deployment filter matches all your critical controllers and operators. The namespace filter excludes kube-system because those are GCP-managed components that belong in your informational alerts category. When this query evaluates to true (equals zero), it means that specific deployment has no healthy replicas running.

**Alert Definition**
```yaml
groups:
  - name: critical_platform_controllers
    interval: 30s
    rules:
      - alert: CriticalControllerUnavailable
        expr: |
          kube_deployment_status_replicas_available{
            deployment=~"gatekeeper-controller-manager|gatekeeper-audit|argocd-application-controller|argocd-repo-server|external-secrets.*|vault-agent-injector.*|alertmanager.*",
            namespace!="kube-system"
          } == 0
        for: 2m
        labels:
          severity: critical
          component: platform_controller
          managed_by: user
        annotations:
          summary: "Critical controller {{ $labels.deployment }} is completely unavailable"
          description: |
            Deployment {{ $labels.deployment }} in namespace {{ $labels.namespace }} has zero available replicas.
            
            Component-specific impact:
            - gatekeeper-controller: Policy enforcement stopped, admission control may fail
            - argocd-application-controller: GitOps synchronization halted, no drift detection
            - argocd-repo-server: Git operations failing, preventing syncs
            - external-secrets: Secret synchronization stopped, pod restarts will fail
            - vault-agent-injector: Secret injection webhook unavailable, new pods cannot start
            - alertmanager: Alert notifications not being sent
            
            Current status: {{ $value }} replicas available
          runbook_url: "https://your-runbook-url/controller-unavailable"
          action_required: "kubectl get pods -n {{ $labels.namespace }} | grep {{ $labels.deployment }}, kubectl describe pod -n {{ $labels.namespace }}, check logs for crash loops, verify resource limits, check webhook certificates for admission controllers"
```

Notice how the annotations template uses the label values to provide context-specific information. The double curly braces are Prometheus template syntax that gets replaced with actual values when the alert fires. So if Gatekeeper fails, you see "gatekeeper-controller-manager" in the summary. If External Secrets fails, you see the external-secrets deployment name. You get specific actionable information from a single consolidated alert rule.

---

## **Consolidated Alert 2: Critical DaemonSets Not Fully Deployed**

You already had this one consolidated in the original design, so let me just refine it slightly to make the consolidation pattern more explicit and add some components you might have missed.

**Why DaemonSets Deserve Their Own Consolidated Alert**

DaemonSets have a fundamentally different failure pattern than deployments. With a deployment, you care whether any replicas are available. With a DaemonSet, you care whether it's running on all the nodes it should be running on. A DaemonSet with ninety-nine percent coverage still represents a blind spot on that one node. This is especially critical for security and observability components where complete coverage is a compliance requirement, not just a best practice.

The troubleshooting approach for DaemonSet issues also differs from deployment issues. When a DaemonSet pod is missing from a node, you investigate node-level concerns like taints, resource pressure, and node selectors rather than deployment-level concerns like replica counts and pod disruption budgets.

**PromQL Query**
```promql
# Monitors all critical DaemonSets for incomplete deployment
# Alerts when any DaemonSet has nodes missing the required pod
(
  kube_daemonset_status_desired_number_scheduled{
    daemonset=~"fluentbit.*|fluent-bit.*|crowdstrike.*|qualys.*|aqua-enforcer.*|antrea-agent.*|konnectivity-agent.*",
    namespace!="kube-system"
  }
  - 
  kube_daemonset_status_number_ready{
    daemonset=~"fluentbit.*|fluent-bit.*|crowdstrike.*|qualys.*|aqua-enforcer.*|antrea-agent.*|konnectivity-agent.*",
    namespace!="kube-system"
  }
) > 0
```

This query calculates the gap between desired pods and ready pods for each DaemonSet. The desired number tells you how many nodes should have this DaemonSet pod based on node selectors and tolerations. The ready number tells you how many are actually running and healthy. The difference reveals your coverage gap. I've included your security agents, logging collectors, and network components in the filter. Notice I'm again excluding kube-system because components like konnectivity-agent in that namespace are managed by GCP and belong in your informational alerts.

**Alert Definition**
```yaml
groups:
  - name: critical_daemonsets
    interval: 30s
    rules:
      - alert: CriticalDaemonSetIncomplete
        expr: |
          (
            kube_daemonset_status_desired_number_scheduled{
              daemonset=~"fluentbit.*|fluent-bit.*|crowdstrike.*|qualys.*|aqua-enforcer.*|antrea-agent.*",
              namespace!="kube-system"
            }
            - 
            kube_daemonset_status_number_ready{
              daemonset=~"fluentbit.*|fluent-bit.*|crowdstrike.*|qualys.*|aqua-enforcer.*|antrea-agent.*",
              namespace!="kube-system"
            }
          ) > 0
        for: 5m
        labels:
          severity: warning
          component: daemonset
          managed_by: user
        annotations:
          summary: "DaemonSet {{ $labels.daemonset }} has incomplete node coverage"
          description: |
            DaemonSet {{ $labels.daemonset }} in namespace {{ $labels.namespace }} is missing {{ $value }} pods.
            
            Coverage gap impact:
            - fluentbit/fluent-bit: Log collection gaps, missing audit trail
            - crowdstrike: Security monitoring blind spot, potential compliance violation
            - qualys: Vulnerability scanning incomplete, unscanned attack surface
            - aqua-enforcer: Runtime security enforcement gaps
            - antrea-agent: Network policy enforcement incomplete
            
            This typically indicates node scheduling issues, resource constraints, or taint/toleration mismatches.
          runbook_url: "https://your-runbook-url/daemonset-incomplete"
          action_required: "kubectl get pods -n {{ $labels.namespace }} -l app={{ $labels.daemonset }} -o wide, kubectl get nodes --show-labels, check for node taints and resource pressure"
```

I've set this at warning severity rather than critical because while coverage gaps are serious, they don't typically cause immediate widespread service disruption the way a controller failure does. You need to investigate and remediate, but it's not usually a middle-of-the-night emergency.

---

## **Consolidated Alert 3: Pod Scheduling and Stability Issues**

Here's where we can make a significant consolidation that reduces noise while actually improving signal quality. Your original design had separate alerts for pods stuck pending and pods with high restart rates. These are both pod-level health issues, but they have different root causes and require different responses, so I recommend keeping them separate rather than consolidating them into a single alert. However, we can refine each one to reduce noise.

**Alert 3a: Pods Stuck in Pending State**

The key refinement here is adding more sophisticated filtering to exclude expected pending states and focus on genuine scheduling failures.

**PromQL Query**
```promql
# Alert on pods stuck pending, excluding system namespaces and specific expected cases
kube_pod_status_phase{
  phase="Pending",
  namespace!~"kube-system|kube-public|kube-node-lease"
} == 1
```

I've excluded not just kube-system but also kube-public and kube-node-lease, which are other GCP-managed namespaces. You might also want to add any namespaces where you deliberately run pending pods for testing or validation purposes.

**Alert Definition**
```yaml
groups:
  - name: pod_health
    interval: 30s
    rules:
      - alert: PodStuckPending
        expr: |
          kube_pod_status_phase{
            phase="Pending",
            namespace!~"kube-system|kube-public|kube-node-lease"
          } == 1
        for: 10m
        labels:
          severity: warning
          component: workload
          managed_by: user
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} stuck in Pending state"
          description: |
            Pod has been Pending for over 10 minutes and cannot be scheduled.
            
            Common root causes:
            - Insufficient cluster resources (CPU, memory, or ephemeral storage)
            - Unsatisfied node selectors or affinity rules
            - Volume provisioning failure or PVC issues
            - Taint/toleration mismatches preventing scheduling
            - Pod security policies or admission webhook rejections
            
            Investigate with: kubectl describe pod -n {{ $labels.namespace }} {{ $labels.pod }}
          runbook_url: "https://your-runbook-url/pod-pending"
          action_required: "Check pod events for scheduling failures, verify cluster capacity, review resource requests, check PVC status if applicable"
```

**Alert 3b: Pods with High Restart Rates**

For restart rate monitoring, the refinement is to focus on recent restarts rather than all-time restart counts, which gives you better signal about current instability versus historical issues.

**PromQL Query**
```promql
# Alert on pods restarting frequently in the last 30 minutes
# Excludes system namespaces and calculates restart rate
rate(kube_pod_container_status_restarts_total{
  namespace!~"kube-system|kube-public|kube-node-lease"
}[30m]) * 1800 > 5
```

This calculates the restart rate over a thirty-minute window and multiplies by eighteen hundred (thirty minutes in seconds) to get the number of restarts in that window. When this exceeds five, you know the pod is in a restart loop that won't self-resolve.

**Alert Definition**
```yaml
groups:
  - name: pod_health
    interval: 60s
    rules:
      - alert: PodFrequentRestarts
        expr: |
          rate(kube_pod_container_status_restarts_total{
            namespace!~"kube-system|kube-public|kube-node-lease"
          }[30m]) * 1800 > 5
        for: 5m
        labels:
          severity: warning
          component: workload
          managed_by: user
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} restarting frequently"
          description: |
            Container {{ $labels.container }} has restarted {{ $value | humanize }} times in the last 30 minutes.
            
            Common root causes:
            - OOMKills from memory limit exceeded
            - Application crashes or panic conditions
            - Misconfigured liveness probes causing false failures
            - Dependency failures (database, external API, etc.)
            - Configuration errors in environment variables or mounted configs
            
            Check container status: kubectl get pod -n {{ $labels.namespace }} {{ $labels.pod }} -o jsonpath='{.status.containerStatuses[?(@.name=="{{ $labels.container }}")].lastState}'
          runbook_url: "https://your-runbook-url/pod-restarts"
          action_required: "Review pod logs, check for OOMKilled in describe output, examine resource limits, validate liveness probe configuration"
```

---

## **Consolidated Alert 4: ArgoCD Platform Health**

This is an interesting consolidation opportunity. Your original design had separate alerts for ArgoCD controller unavailability and application health degradation. These are related but serve different purposes, so let me show you how to handle this thoughtfully.

The controller unavailability alert is actually already covered by your consolidated controller alert, so you don't need a separate one. However, ArgoCD application health is conceptually different because it's monitoring the state of your deployed applications rather than the ArgoCD platform itself. I recommend keeping this as a separate focused alert.

**PromQL Query**
```promql
# Alert when ArgoCD applications are not healthy
# Combines both sync and health status issues
argocd_app_info{
  sync_status!="Synced"
} == 1
or
argocd_app_info{
  health_status!~"Healthy|Progressing"
} == 1
```

This query catches both synchronization failures and health issues in a single alert. The "or" operator means the alert fires if either condition is true. The health_status check excludes both Healthy and Progressing because Progressing is a normal transient state during deployments. We only care about Degraded, Missing, Unknown, or Suspended states.

**Alert Definition**
```yaml
groups:
  - name: gitops_platform
    interval: 60s
    rules:
      - alert: ArgoCDApplicationIssue
        expr: |
          argocd_app_info{sync_status!="Synced"} == 1
          or
          argocd_app_info{health_status!~"Healthy|Progressing"} == 1
        for: 15m
        labels:
          severity: warning
          component: gitops
          managed_by: user
        annotations:
          summary: "ArgoCD application {{ $labels.name }} has issues"
          description: |
            Application {{ $labels.name }} in project {{ $labels.project }} is experiencing problems.
            
            Status: Sync={{ $labels.sync_status }}, Health={{ $labels.health_status }}
            
            Sync status issues indicate:
            - OutOfSync: Cluster state differs from Git, manual changes or sync failures
            - Unknown: ArgoCD cannot determine sync state
            
            Health status issues indicate:
            - Degraded: Resources deployed but not functioning (pods crashing, failing health checks)
            - Missing: Expected resources not found in cluster
            - Unknown: Cannot determine health status
            
            Review in ArgoCD UI for detailed resource status and sync differences.
          runbook_url: "https://your-runbook-url/argocd-application-issues"
          action_required: "Check ArgoCD UI for detailed status, review application logs, verify Git repository accessibility, check for resource quota issues"
```

The fifteen-minute threshold is important here. ArgoCD applications often show temporary out-of-sync or unhealthy states during deployments, and you don't want to alert on every rolling update. Fifteen minutes is long enough to let normal deployment processes complete while catching genuine problems that need investigation.

---

## **Informational Alerts: GCP-Managed Infrastructure**

These remain largely as they were because they're already monitoring at the right level of granularity. However, let me show you one consolidation opportunity.

**Consolidated Alert 5: GCP-Managed Component Issues**

**PromQL Query**
```promql
# Single alert for GCP-managed component degradation
# Covers both node health and critical system component availability
(
  kube_node_status_condition{condition="Ready",status="true"} == 0
)
or
(
  kube_deployment_status_replicas_available{
    deployment=~"kube-dns|metrics-server|event-exporter",
    namespace="kube-system"
  } < kube_deployment_spec_replicas{
    deployment=~"kube-dns|metrics-server|event-exporter",
    namespace="kube-system"
  }
)
```

This combines node health with critical system component availability in kube-system. The second part of the query checks if available replicas are less than desired replicas, which catches partial degradation as well as complete failures.

**Alert Definition**
```yaml
groups:
  - name: gcp_managed_infrastructure
    interval: 30s
    rules:
      - alert: GCPManagedComponentDegraded
        expr: |
          (
            kube_node_status_condition{condition="Ready",status="true"} == 0
          )
          or
          (
            kube_deployment_status_replicas_available{
              deployment=~"kube-dns|metrics-server|event-exporter",
              namespace="kube-system"
            } < kube_deployment_spec_replicas{
              deployment=~"kube-dns|metrics-server|event-exporter",
              namespace="kube-system"
            }
          )
        for: 3m
        labels:
          severity: info
          component: gcp_managed
          managed_by: gcp
        annotations:
          summary: "GCP-managed component {{ $labels.node }}{{ $labels.deployment }} is degraded"
          description: |
            A GCP-managed infrastructure component is experiencing issues.
            
            If this is a node: Node {{ $labels.node }} is NotReady. GKE auto-repair should handle this automatically.
            If this is a deployment: {{ $labels.deployment }} in kube-system has {{ $value }} replicas when expecting more.
            
            These components are managed by Google Cloud Platform. Monitor for auto-recovery.
            If issues persist beyond 15 minutes, escalate to GCP support.
          runbook_url: "https://your-runbook-url/gcp-component-degraded"
          action_required: "Monitor in GCP Console, check for ongoing maintenance or incidents in GCP Status Dashboard, verify no user changes affecting system namespace, escalate to GCP support if persistent"
```
