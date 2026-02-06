## **Category 1: Critical Actionable Alerts - Your Primary Response Targets**

These are alerts for components you directly manage and maintain. When these fire, your team owns the investigation and remediation. These should page your on-call engineer because they represent failures in systems under your operational control.

### **Alert 1: Gatekeeper Policy Enforcement Unavailable**

**Description**  
This alert fires when your Gatekeeper admission controller becomes unavailable, meaning the webhook endpoints are not responding to admission requests from the API server.

**Rationale**  
Let me explain why this is your most critical user-managed alert. In a banking environment, Gatekeeper serves as your automated compliance gatekeeper. When developers or automated systems try to create resources in your cluster, Gatekeeper intercepts those requests and validates them against your organizational policies before they ever get persisted to etcd. These policies might enforce things like mandatory resource limits, prohibited capabilities, required security contexts, or mandatory labels for cost allocation. When Gatekeeper is down, you face a critical decision point that's actually configured in the webhook's failure policy. If your webhook is configured with a failure policy of "Ignore," then policy violations can slip through silently, creating compliance debt and potential security vulnerabilities. If it's configured with "Fail," then all resource creation matching the webhook's scope gets blocked, which could halt all deployments across your cluster. Neither outcome is acceptable in production.

**Possible Impact**  
The impact depends on your webhook configuration, but both scenarios are severe. With a fail-open policy, you might allow privileged containers to run when your security standards prohibit them, or permit containers without resource limits that could monopolize node resources. In regulated banking, these violations could surface during audits and require expensive remediation. With a fail-closed policy, your entire deployment pipeline grinds to a halt. ArgoCD cannot sync applications, your developers cannot deploy their services, and you cannot even deploy a fix to Gatekeeper itself through normal channels. You would need to manually disable the webhook configuration to regain cluster functionality.

**Potential Thresholds and Rationale**  
I recommend alerting when Gatekeeper controller manager pods are unavailable for more than two minutes. This threshold accounts for brief disruptions during rolling updates while catching genuine failures quickly. The controller manager is the component that actually processes validation requests, so its unavailability directly translates to policy enforcement failure. You should also monitor the audit controller separately because it provides continuous validation of existing resources, helping you detect drift where resources that were compliant at creation time have been mutated into non-compliant states.

**PromQL Query**
```promql
# Alert when Gatekeeper controller is completely unavailable
kube_deployment_status_replicas_available{deployment="gatekeeper-controller-manager",namespace="gatekeeper-system"} == 0
```

**Alert Definition**
```yaml
groups:
  - name: user_managed_security
    interval: 30s
    rules:
      - alert: GatekeeperControllerUnavailable
        expr: kube_deployment_status_replicas_available{deployment="gatekeeper-controller-manager",namespace="gatekeeper-system"} == 0
        for: 2m
        labels:
          severity: critical
          component: security
          managed_by: user
        annotations:
          summary: "Gatekeeper policy enforcement is completely unavailable"
          description: "All Gatekeeper controller manager pods are unavailable. Admission webhook cannot enforce policies. Check if failurePolicy=Ignore (allowing violations) or failurePolicy=Fail (blocking all requests)."
          runbook_url: "https://your-runbook-url/gatekeeper-unavailable"
          action_required: "Check pod status, review recent changes to Gatekeeper configuration, verify webhook certificates are valid, ensure sufficient cluster resources"
```

---

### **Alert 2: ArgoCD Application Controller Unavailable**

**Description**  
This alert triggers when your ArgoCD application controller becomes unavailable, preventing GitOps synchronization and breaking your declarative deployment model.

**Rationale**  
Your ArgoCD application controller is the reconciliation engine that continuously compares the desired state in your Git repositories against the actual state in your cluster. Think of it as the conductor of your infrastructure orchestra, constantly ensuring every component plays the right notes. When this controller goes down, you lose the ability to deploy new applications or update existing ones through your GitOps workflow. More subtly, you also lose drift detection, which means someone could make manual changes to your cluster using kubectl and you would not get automatic correction back to the Git-defined state. In a mature GitOps practice, the application controller is your source of truth enforcement, and its failure means your cluster state can diverge from your approved configurations without detection or remediation.

**Possible Impact**  
The immediate impact is that deployments stop flowing through your pipeline. If you have a critical security patch that needs to be deployed, it will not sync to your cluster even if you merge the pull request and update your Git repository. Your deployment velocity drops to zero for GitOps-managed applications. The more insidious impact is drift accumulation. During the controller outage, well-meaning engineers might make emergency changes directly with kubectl to resolve incidents. Without the controller running, these changes persist and create configuration inconsistencies between your environments. When the controller finally recovers, it may attempt to revert these emergency changes, potentially causing new incidents if those changes were actually necessary fixes.

**Potential Thresholds and Rationale**  
I suggest alerting when the application controller has been unavailable for more than three minutes. ArgoCD deployments can take a moment to restart during upgrades, and you do not want alert fatigue from planned maintenance. However, three minutes is long enough to distinguish between a rolling update and a genuine failure. The application controller is stateless and should recover quickly if it is simply restarting. If it stays down beyond three minutes, something more serious is happening such as resource exhaustion, persistent crash loops, or infrastructure problems.

**PromQL Query**
```promql
# Alert when ArgoCD application controller has no available replicas
kube_deployment_status_replicas_available{deployment="argocd-application-controller",namespace=~"argocd|argo-cd"} == 0
```

**Alert Definition**
```yaml
groups:
  - name: user_managed_gitops
    interval: 30s
    rules:
      - alert: ArgoCDApplicationControllerUnavailable
        expr: kube_deployment_status_replicas_available{deployment="argocd-application-controller",namespace=~"argocd|argo-cd"} == 0
        for: 3m
        labels:
          severity: critical
          component: gitops
          managed_by: user
        annotations:
          summary: "ArgoCD application controller is unavailable"
          description: "ArgoCD cannot sync applications or detect configuration drift. GitOps deployments are blocked until controller recovers."
          runbook_url: "https://your-runbook-url/argocd-controller-unavailable"
          action_required: "Check controller pod logs, verify Redis connectivity, check for OOMKills, review recent ArgoCD configuration changes, ensure cluster has sufficient CPU and memory"
```

---

### **Alert 3: External Secrets Operator Unavailable**

**Description**  
This alert fires when your External Secrets operator becomes unavailable, preventing synchronization of secrets from external secret stores into Kubernetes secrets.

**Rationale**  
The External Secrets operator bridges the gap between your enterprise secret management system such as HashiCorp Vault, AWS Secrets Manager, or Google Secret Manager and your Kubernetes workloads. When this operator is running normally, it continuously syncs secrets from your external store into Kubernetes secret objects that your pods can consume. When it fails, the synchronization stops. Now, here is the critical nuance you need to understand. Existing secrets that were already synced into Kubernetes will continue to exist and your running pods can still access them. The problem emerges when pods restart or when you deploy new applications. Those pods expect the External Secrets operator to have created the necessary Kubernetes secrets, and if the operator is down, those secrets do not exist. The pods will fail to start because they cannot mount the required secret volumes or inject the required environment variables.

**Possible Impact**  
The impact unfolds gradually rather than immediately. Your currently running workloads continue operating normally because they already have their secrets mounted. However, any pod that restarts will fail to come back up if it depends on secrets managed by the External Secrets operator. This creates a dangerous situation during an incident. Imagine you have a memory leak in a service and the pod gets OOMKilled. Normally, Kubernetes would restart it automatically and you would have momentary downtime. With External Secrets down, that pod cannot restart and your momentary downtime becomes an extended outage. Similarly, any new deployments or scaling operations will fail. In your banking environment, this could prevent you from scaling up to handle increased transaction volume or from deploying urgent fixes.

**Potential Thresholds and Rationale**  
Alert when the External Secrets operator has zero available replicas for more than two minutes. This component is critical enough that even brief unavailability deserves attention, but two minutes allows for normal rolling updates without creating noise. The operator is typically lightweight and should restart quickly, so extended downtime suggests a real problem such as API authentication failures to your secret backend, network policy issues blocking access to the external secret store, or resource constraints preventing the operator from running.

**PromQL Query**
```promql
# Alert when External Secrets operator has no available replicas
kube_deployment_status_replicas_available{deployment=~"external-secrets.*",namespace=~"external-secrets.*"} == 0
```

**Alert Definition**
```yaml
groups:
  - name: user_managed_secrets
    interval: 30s
    rules:
      - alert: ExternalSecretsOperatorUnavailable
        expr: kube_deployment_status_replicas_available{deployment=~"external-secrets.*",namespace=~"external-secrets.*"} == 0
        for: 2m
        labels:
          severity: critical
          component: secrets
          managed_by: user
        annotations:
          summary: "External Secrets operator is unavailable"
          description: "Secret synchronization from external stores has stopped. Existing secrets remain available but pod restarts and new deployments will fail if they depend on operator-managed secrets."
          runbook_url: "https://your-runbook-url/external-secrets-unavailable"
          action_required: "Verify operator pod status, check authentication to secret backend, review network policies, validate SecretStore configurations, check operator logs for API errors"
```

---

### **Alert 4: Vault Agent Injector Unavailable**

**Description**  
This alert detects when your Vault Agent Injector mutating webhook becomes unavailable, preventing automatic injection of Vault secrets into pods.

**Rationale**  
The Vault Agent Injector works through a mutating admission webhook that intercepts pod creation requests. When a pod has specific annotations requesting Vault secret injection, the webhook modifies the pod specification to add init containers and sidecar containers that handle the actual secret retrieval from Vault. This is a different pattern from the External Secrets operator. With External Secrets, the operator creates Kubernetes secret objects that pods consume normally. With Vault injection, the secrets are fetched at pod startup time directly from Vault and never exist as Kubernetes secrets at all. When the Vault Agent Injector webhook is unavailable, pods with injection annotations will fail to start because the webhook cannot modify them to add the injection machinery.

**Possible Impact**  
The impact is immediate for new pod deployments that request Vault injection. Those pods will either fail to be created if the webhook failure policy is set to Fail, or they will be created without secret injection if the policy is set to Ignore. In the first case, your deployments are blocked. In the second case, your applications start but lack their required credentials, which typically causes them to crash or fail health checks immediately. Unlike External Secrets where existing pods continue running, Vault injection affects every pod deployment because the injection happens at pod creation time. This makes scaling operations particularly dangerous. If you need to scale up to handle load but the injector is down, your new pods will not get their credentials and will not contribute to serving traffic.

**Potential Thresholds and Rationale**  
Alert when the Vault Agent Injector has been unavailable for more than one minute. This is a shorter threshold than some other components because the injector is a webhook that participates in every pod creation, and failures here have immediate blast radius. One minute is enough to distinguish between a brief pod restart and a sustained outage, while being aggressive enough to catch problems before they cause incident escalation.

**PromQL Query**
```promql
# Alert when Vault Agent Injector has no available replicas
kube_deployment_status_replicas_available{deployment=~"vault-agent-injector.*",namespace=~"vault.*"} == 0
```

**Alert Definition**
```yaml
groups:
  - name: user_managed_secrets
    interval: 30s
    rules:
      - alert: VaultAgentInjectorUnavailable
        expr: kube_deployment_status_replicas_available{deployment=~"vault-agent-injector.*",namespace=~"vault.*"} == 0
        for: 1m
        labels:
          severity: critical
          component: secrets
          managed_by: user
        annotations:
          summary: "Vault Agent Injector webhook is unavailable"
          description: "Vault secret injection into pods has failed. New pods requesting injection cannot start. Check webhook failurePolicy to determine if pods are blocked or starting without secrets."
          runbook_url: "https://your-runbook-url/vault-injector-unavailable"
          action_required: "Check injector pod status, verify webhook certificate validity, ensure Vault connectivity, review mutating webhook configuration, check for resource constraints"
```

---

### **Alert 5: Critical DaemonSet Not Running on All Nodes**

**Description**  
This alert triggers when critical DaemonSets like Fluentbit, CrowdStrike, or Qualys are not running on all schedulable nodes, creating gaps in logging or security coverage.

**Rationale**  
DaemonSets are Kubernetes' mechanism for ensuring that specific pods run on every node in your cluster. This is perfect for infrastructure concerns that need node-level coverage, such as log collection, security monitoring, and vulnerability scanning. Your Fluentbit DaemonSet collects logs from all containers on each node and forwards them to your centralized logging system. Your CrowdStrike and Qualys DaemonSets provide endpoint detection and response as well as vulnerability scanning on each node. When a DaemonSet is not running on all nodes, you have blind spots. Logs from those nodes are not being collected, security events are not being detected, and vulnerabilities are not being scanned. In a banking environment with regulatory requirements for comprehensive audit logging and security monitoring, these gaps can become compliance violations.

**Possible Impact**  
The impact depends on which DaemonSet is affected and which nodes are missing coverage. If Fluentbit is not running on a node, any security incident or application error on that node will not appear in your logs, making forensic investigation difficult or impossible. If CrowdStrike is not running, malicious activity on that node may go undetected until it propagates to other systems. If Qualys is not running, you may be operating with vulnerable software on that node without knowing it. The problem compounds over time because these are continuous monitoring systems. A node without Fluentbit for six hours means six hours of missing logs. A node without CrowdStrike for a day means a full day where an attacker could have established persistence without being detected.

**Potential Thresholds and Rationale**  
Alert when a critical DaemonSet has desired pods that exceed available pods by more than zero for five minutes. This means at least one node is missing the DaemonSet pod. The five-minute threshold accounts for normal situations like new nodes being added to the cluster where it takes a moment for the DaemonSet pod to be scheduled and become ready. You should customize this alert to target your specific critical DaemonSets rather than alerting on all DaemonSets, since some may be less critical or expected to run only on subsets of nodes using node selectors.

**PromQL Query**
```promql
# Alert when critical DaemonSets are not running on all desired nodes
(
  kube_daemonset_status_desired_number_scheduled{daemonset=~"fluentbit|crowdstrike|qualys-scanner"}
  - 
  kube_daemonset_status_number_ready{daemonset=~"fluentbit|crowdstrike|qualys-scanner"}
) > 0
```

**Alert Definition**
```yaml
groups:
  - name: user_managed_infrastructure
    interval: 30s
    rules:
      - alert: CriticalDaemonSetNotFullyDeployed
        expr: |
          (
            kube_daemonset_status_desired_number_scheduled{daemonset=~"fluentbit|crowdstrike|qualys-scanner"}
            - 
            kube_daemonset_status_number_ready{daemonset=~"fluentbit|crowdstrike|qualys-scanner"}
          ) > 0
        for: 5m
        labels:
          severity: warning
          component: infrastructure
          managed_by: user
        annotations:
          summary: "DaemonSet {{ $labels.daemonset }} is not running on all nodes"
          description: "{{ $value }} nodes are missing this critical DaemonSet pod. This creates gaps in {{ if eq $labels.daemonset \"fluentbit\" }}log collection{{ else if eq $labels.daemonset \"crowdstrike\" }}security monitoring{{ else }}vulnerability scanning{{ end }}."
          runbook_url: "https://your-runbook-url/daemonset-incomplete"
          action_required: "Identify which nodes are missing pods, check for node taints or tolerations issues, verify resource availability on affected nodes, review DaemonSet pod logs for scheduling failures"
```

---

## **Category 2: Important Actionable Alerts - Degradation and Performance**

These alerts indicate problems that require investigation and remediation but may not immediately impact production services. They typically give you early warning of issues that could escalate into critical failures if left unaddressed.

### **Alert 6: Pod Stuck in Pending State**

**Description**  
This alert fires when user-managed pods remain in Pending state for an extended period, indicating they cannot be scheduled onto nodes.

**Rationale**  
When a pod is Pending, it means the Kubernetes scheduler cannot find a suitable node to run it on. This happens for several reasons, and understanding the distinction is important for your troubleshooting. The pod might have resource requests that exceed available capacity on any node, meaning you need larger nodes or more nodes in your cluster. It might have node selectors or node affinity rules that match no nodes, indicating a configuration mismatch. It might require a persistent volume that cannot be provisioned or attached. Or it might have tolerations that do not match the taints on available nodes. Each root cause requires a different remediation approach, but they all manifest as the pod sitting in Pending state indefinitely.

**Possible Impact**  
Pods stuck Pending cannot serve traffic or perform their intended function. If this affects your ArgoCD repo server, then Git repository operations slow down or fail. If it affects Alertmanager, you may not receive notifications for other alerts. The impact extends beyond the specific pod because other components may depend on it. For instance, if your CINS Kubernetes Operator pod is Pending, then the custom resources it manages will not be reconciled, creating a cascading effect where seemingly unrelated applications fail to deploy or update.

**Potential Thresholds and Rationale**  
Alert when pods have been Pending for more than ten minutes. Normal pod startup typically completes in under a minute, even accounting for image pulls. Ten minutes is conservative enough to avoid alerting on transient scheduling delays during cluster scaling events, but aggressive enough to catch real problems before they cause visible service impact. You should exclude certain namespaces like kube-system from this alert since GCP-managed components in that namespace are outside your control.

**PromQL Query**
```promql
# Alert when user pods are stuck Pending
# Exclude kube-system since those are GCP-managed
kube_pod_status_phase{phase="Pending",namespace!="kube-system"} == 1
```

**Alert Definition**
```yaml
groups:
  - name: user_managed_workloads
    interval: 30s
    rules:
      - alert: PodStuckPending
        expr: kube_pod_status_phase{phase="Pending",namespace!="kube-system"} == 1
        for: 10m
        labels:
          severity: warning
          component: workload
          managed_by: user
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} stuck in Pending state"
          description: "Pod cannot be scheduled. Common causes: insufficient resources, unsatisfied node selectors, volume provisioning failures, or taint/toleration mismatches."
          runbook_url: "https://your-runbook-url/pod-pending"
          action_required: "Run 'kubectl describe pod' to see scheduling events, check cluster capacity with 'kubectl top nodes', verify PVC status if pod uses volumes, review pod resource requests and node selectors"
```

---

### **Alert 7: High Pod Restart Rate for Critical Components**

**Description**  
This alert detects when critical user-managed pods are experiencing frequent restarts, indicating application instability or resource issues.

**Rationale**  
Healthy pods should run continuously without restarts. When you see frequent restarts, it indicates one of several underlying problems. The application might have a memory leak causing OOMKills where the process consumes more memory than its limit and the kernel kills it. The liveness probe might be misconfigured, failing even though the application is healthy, causing Kubernetes to restart the pod unnecessarily. The application might be crashing due to bugs, misconfigurations, or dependency failures. Or the pod might be experiencing resource contention where CPU throttling or disk I/O issues cause it to fail health checks. Each restart causes brief unavailability, and continuous restart loops can prevent the application from ever becoming stable enough to serve traffic.

**Possible Impact**  
The immediate impact is reduced availability as the pod cycles through crash loops. Each restart takes time for the container to initialize, for the application to start up, and for health checks to pass. If this affects your ArgoCD Redis instance, you might see intermittent failures in Git operations as the cache becomes unavailable. If it affects your Aqua Enforcer, you get temporary gaps in runtime security enforcement. The secondary impact is cluster resource waste as the scheduler repeatedly tries to restart the pod, consuming CPU cycles and potentially triggering autoscaling unnecessarily.

**Potential Thresholds and Rationale**  
Alert when a pod has restarted more than five times in thirty minutes. This threshold distinguishes between a single failure that was resolved by restart versus chronic instability. Five restarts in thirty minutes means the pod is stuck in a restart loop that will not self-resolve, requiring human intervention to diagnose and fix the underlying cause.

**PromQL Query**
```promql
# Alert on high restart rate for user-managed pods
# Excludes kube-system namespace
rate(kube_pod_container_status_restarts_total{namespace!="kube-system"}[30m]) * 1800 > 5
```

**Alert Definition**
```yaml
groups:
  - name: user_managed_workloads
    interval: 60s
    rules:
      - alert: PodHighRestartRate
        expr: rate(kube_pod_container_status_restarts_total{namespace!="kube-system"}[30m]) * 1800 > 5
        for: 5m
        labels:
          severity: warning
          component: workload
          managed_by: user
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} restarting frequently"
          description: "Pod has restarted {{ $value | humanize }} times in last 30 minutes. Likely causes: OOMKills, application crashes, or failing health checks."
          runbook_url: "https://your-runbook-url/pod-restarts"
          action_required: "Check pod logs, review resource usage with 'kubectl top pod', examine liveness/readiness probe configuration, look for OOMKilled events in 'kubectl describe pod'"
```

---

### **Alert 8: ArgoCD Application Health Degraded**

**Description**  
This alert triggers when ArgoCD applications report degraded health status, indicating issues with the deployed resources even if they are in sync with Git.

**Rationale**  
ArgoCD tracks two separate dimensions of application state. Sync status tells you whether the cluster state matches your Git repository. Health status tells you whether the deployed resources are actually functioning correctly. An application can be fully synced but unhealthy if the resources are deployed correctly but experiencing runtime failures. For example, your deployment might match Git perfectly, but the pods could be in CrashLoopBackOff due to a configuration error in a ConfigMap. ArgoCD determines health by examining the status fields of various Kubernetes resources. Deployments are healthy when all replicas are ready. Services are healthy when they have endpoints. Ingresses are healthy when their backends are available. When ArgoCD reports degraded health, it means these runtime checks are failing.

**Possible Impact**  
Degraded health indicates that while your GitOps process completed successfully, the deployed application is not functioning correctly. This could mean users are experiencing errors, background jobs are not processing, or data synchronization is failing. The degraded state might be temporary as pods roll out and become ready, or it might be persistent if there is a fundamental problem with the application configuration. The key risk is that you might believe a deployment succeeded based on ArgoCD showing it as synced, when in reality the application is broken.

**Potential Thresholds and Rationale**  
Alert when applications have been in degraded health status for more than ten minutes. This allows time for normal deployment processes where pods temporarily show as unhealthy during rolling updates. Ten minutes is long enough to distinguish between transient unhealthy states during deployments and persistent problems requiring investigation.

**PromQL Query**
```promql
# Alert when ArgoCD applications are unhealthy
# Note: You may need to adjust this based on how ArgoCD exposes health metrics
argocd_app_info{health_status=~"Degraded|Missing"} == 1
```

**Alert Definition**
```yaml
groups:
  - name: user_managed_gitops
    interval: 60s
    rules:
      - alert: ArgoCDApplicationUnhealthy
        expr: argocd_app_info{health_status=~"Degraded|Missing"} == 1
        for: 10m
        labels:
          severity: warning
          component: gitops
          managed_by: user
        annotations:
          summary: "ArgoCD application {{ $labels.name }} is unhealthy"
          description: "Application health status is {{ $labels.health_status }}. Resources are deployed but not functioning correctly. Check pod status and application logs."
          runbook_url: "https://your-runbook-url/argocd-unhealthy"
          action_required: "Review application in ArgoCD UI, check resource events and pod logs, verify ConfigMaps and Secrets are correct, examine resource limits and health check configuration"
```

---

## **Category 3: Informational Alerts - GCP-Managed Components**

These alerts monitor GCP-managed infrastructure components. When they fire, you cannot directly fix the underlying issue, but you need to know about it for incident correlation and to inform your interactions with Google Cloud support. Set these at warning or informational severity rather than critical.

### **Alert 9: Node NotReady Status**

**Description**  
This alert fires when GKE nodes enter NotReady state, indicating Google's node management systems have detected a problem.

**Rationale**  
Even though GKE manages node health and will automatically repair or replace unhealthy nodes through node auto-repair, you need visibility into when this is happening. Node failures can cause temporary service disruptions as pods are rescheduled, and understanding the node failure pattern helps you make informed decisions about your cluster configuration. If you see frequent node failures in a particular node pool, it might indicate that your workload characteristics do not match the node type, or that you have reached capacity limits causing excessive pressure on nodes.

**Possible Impact**  
When a node goes NotReady, running pods on that node are eventually evicted and rescheduled to other nodes. For stateless workloads with multiple replicas, this might be transparent to users. For stateful workloads or single-replica services, you will see downtime during the migration. The auto-repair process takes several minutes to complete, during which your cluster has reduced capacity.

**What You Can Action**  
While you cannot fix the node itself, you can verify that auto-repair is functioning, check whether the node failure is affecting critical workloads, and potentially cordon other nodes if you see a pattern suggesting wider infrastructure issues. You can also adjust your pod disruption budgets or replica counts if node failures are causing too much impact.

**PromQL Query**
```promql
# Alert on nodes in NotReady state
kube_node_status_condition{condition="Ready",status="true"} == 0
```

**Alert Definition**
```yaml
groups:
  - name: gcp_managed_infrastructure
    interval: 30s
    rules:
      - alert: GKENodeNotReady
        expr: kube_node_status_condition{condition="Ready",status="true"} == 0
        for: 2m
        labels:
          severity: warning
          component: node
          managed_by: gcp
        annotations:
          summary: "Node {{ $labels.node }} is NotReady"
          description: "Node has been NotReady for >2 minutes. GKE auto-repair should replace it automatically. Monitor for pod evictions and workload impact."
          runbook_url: "https://your-runbook-url/node-not-ready"
          action_required: "Monitor auto-repair progress in GCP Console, check which pods were running on the node, verify workloads rescheduled successfully, consider scaling node pool if pattern emerges"
```

---

### **Alert 10: CoreDNS Low Replica Count**

**Description**  
This alert triggers when the number of available CoreDNS replicas drops below the expected count, potentially affecting cluster DNS resolution.

**Rationale**  
GCP manages CoreDNS for you, but you still need to know when it is degraded. DNS is so fundamental to cluster operations that even partial degradation can cause mysterious failures. Applications might experience intermittent connection timeouts, service discovery might fail sporadically, and troubleshooting becomes difficult because the symptoms appear in many different places while the root cause is centralized in DNS.

**Possible Impact**  
With reduced CoreDNS capacity, you might see increased DNS query latency or timeout errors during high load periods. New pod startup times might increase as they wait for DNS resolution during initialization. In extreme cases where all CoreDNS pods are unavailable, all service-to-service communication that relies on DNS names will fail, effectively halting all application traffic.

**What You Can Action**  
You can check if network policies you created are blocking CoreDNS, verify that your cluster has sufficient resources for the kube-system namespace, and review whether recent changes to your cluster might have affected it. You can also open a support case with Google Cloud if the issue persists beyond what auto-repair should handle.

**PromQL Query**
```promql
# Alert when CoreDNS availability is reduced
kube_deployment_status_replicas_available{deployment="kube-dns",namespace="kube-system"} < 2
```

**Alert Definition**
```yaml
groups:
  - name: gcp_managed_infrastructure
    interval: 30s
    rules:
      - alert: GKECoreDNSLowReplicas
        expr: kube_deployment_status_replicas_available{deployment="kube-dns",namespace="kube-system"} < 2
        for: 3m
        labels:
          severity: warning
          component: dns
          managed_by: gcp
        annotations:
          summary: "CoreDNS has fewer than 2 replicas available"
          description: "CoreDNS replica count is {{ $value }}, expected 2+. DNS resolution may be degraded. This is GCP-managed; monitor for auto-recovery."
          runbook_url: "https://your-runbook-url/coredns-low-replicas"
          action_required: "Monitor DNS query latency, check for recent cluster changes that might affect kube-system namespace, verify no user-created network policies block CoreDNS, escalate to Google Cloud support if persistent"
```
