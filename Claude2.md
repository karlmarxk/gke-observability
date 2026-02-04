# Part 2: DNS/Network SLOs (MUST-HAVE - Tier 1 Critical)

## 1.6 DNS Resolution Performance and Availability

**SLO Definition**:
- 99.95% DNS query success rate
- P95 DNS query latency < 50ms
- P99 DNS query latency < 100ms
- CoreDNS pod availability: 99.9%

**Business Rationale**:
- **Banking Critical**: Every microservice call requires DNS resolution
- Payment processing systems make hundreds of DNS queries per transaction
- DNS failures cause cascading service failures across the platform
- Slow DNS = slow transactions = customer experience degradation
- **Compliance**: Audit trails require reliable service discovery
- **Trading platforms**: Milliseconds matter for real-time systems

**Technical Context**:
- DNS is a shared infrastructure component affecting ALL workloads
- Single point of failure if not properly redundant
- Cache misses cause external queries (increased latency)
- NXDOMAIN responses can indicate security issues or misconfigurations

---

### PromQL Queries for DNS Monitoring

```promql
# ============================================
# DNS SUCCESS RATE METRICS
# ============================================

# Overall DNS Success Rate (5-minute window)
(
  sum(rate(coredns_dns_responses_total{rcode="NOERROR"}[5m]))
  /
  sum(rate(coredns_dns_responses_total[5m]))
) * 100

# DNS Success Rate (30-day SLO window)
(
  sum(rate(coredns_dns_responses_total{rcode="NOERROR"}[30d]))
  /
  sum(rate(coredns_dns_responses_total[30d]))
) * 100

# DNS Error Rate by Response Code
sum(rate(coredns_dns_responses_total{rcode!="NOERROR"}[5m])) 
by (rcode) * 100

# DNS Errors Breakdown
# - NXDOMAIN: Name doesn't exist (could be misconfig or security issue)
# - SERVFAIL: Server failure (upstream DNS issues)
# - REFUSED: Query refused (policy/security)
sum(rate(coredns_dns_responses_total{rcode=~"NXDOMAIN|SERVFAIL|REFUSED"}[5m])) 
by (rcode)

# DNS Error Budget Remaining (30-day window)
(1 - (
  (1 - sum(rate(coredns_dns_responses_total{rcode="NOERROR"}[30d])) 
       / sum(rate(coredns_dns_responses_total[30d])))
  /
  (1 - 0.9995)
)) * 100

# ============================================
# DNS LATENCY METRICS
# ============================================

# DNS Query Latency P50
histogram_quantile(0.50,
  sum(rate(coredns_dns_request_duration_seconds_bucket[5m])) 
  by (le, server, zone)
) * 1000  # Convert to milliseconds

# DNS Query Latency P95
histogram_quantile(0.95,
  sum(rate(coredns_dns_request_duration_seconds_bucket[5m])) 
  by (le, server, zone)
) * 1000

# DNS Query Latency P99
histogram_quantile(0.99,
  sum(rate(coredns_dns_request_duration_seconds_bucket[5m])) 
  by (le, server, zone)
) * 1000

# DNS Latency by Query Type (A, AAAA, SRV, PTR)
histogram_quantile(0.95,
  sum(rate(coredns_dns_request_duration_seconds_bucket[5m])) 
  by (le, type)
) * 1000

# DNS Cache Hit Ratio
# Higher is better - reduces latency and upstream load
(
  sum(rate(coredns_cache_hits_total[5m]))
  /
  (sum(rate(coredns_cache_hits_total[5m])) + sum(rate(coredns_cache_misses_total[5m])))
) * 100

# ============================================
# COREDNS POD HEALTH METRICS
# ============================================

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

# CoreDNS Pods Ready
count(kube_pod_status_ready{
  namespace="kube-system",
  pod=~"coredns.*",
  condition="true"
})

# CoreDNS Pod Restarts (instability indicator)
sum(rate(kube_pod_container_status_restarts_total{
  namespace="kube-system",
  pod=~"coredns.*"
}[30m]))

# ============================================
# COREDNS RESOURCE UTILIZATION
# ============================================

# CoreDNS CPU Usage by Pod
sum(rate(container_cpu_usage_seconds_total{
  namespace="kube-system",
  pod=~"coredns.*",
  container="coredns"
}[5m])) by (pod)

# CoreDNS Memory Usage by Pod
sum(container_memory_working_set_bytes{
  namespace="kube-system",
  pod=~"coredns.*",
  container="coredns"
}) by (pod) / (1024 * 1024)  # Convert to MB

# CoreDNS CPU Throttling
(
  sum(rate(container_cpu_cfs_throttled_seconds_total{
    namespace="kube-system",
    pod=~"coredns.*"
  }[5m])) by (pod)
  /
  sum(rate(container_cpu_usage_seconds_total{
    namespace="kube-system",
    pod=~"coredns.*"
  }[5m])) by (pod)
) * 100

# ============================================
# COREDNS FORWARDING & UPSTREAM METRICS
# ============================================

# Upstream DNS Server Errors
sum(rate(coredns_forward_responses_total{rcode!="NOERROR"}[5m])) 
by (to, rcode)

# Upstream DNS Healthcheck Failures
sum(rate(coredns_forward_healthcheck_failures_total[5m])) 
by (to)

# Upstream DNS Query Latency
histogram_quantile(0.95,
  sum(rate(coredns_forward_request_duration_seconds_bucket[5m])) 
  by (le, to)
) * 1000

# ============================================
# QUERY PATTERNS & ANOMALIES
# ============================================

# DNS Query Rate (QPS - queries per second)
sum(rate(coredns_dns_requests_total[5m]))

# DNS Query Rate by Type
sum(rate(coredns_dns_requests_total[5m])) by (type)

# DNS Query Rate by Zone
sum(rate(coredns_dns_requests_total[5m])) by (zone)

# Top Queried Domains (requires additional logging/metrics)
# This helps identify:
# - Misconfigured services causing excessive queries
# - Potential DNS amplification attacks
# - Services that should be cached longer
topk(10, 
  sum(rate(coredns_dns_requests_total[5m])) by (name)
)

# Unusual Query Spike Detection (comparing to baseline)
# Helps detect DNS-based attacks or misconfigurations
(
  sum(rate(coredns_dns_requests_total[5m]))
  /
  avg_over_time(sum(rate(coredns_dns_requests_total[5m]))[1h:5m])
) > 2  # 2x baseline

# ============================================
# CLUSTER DNS SERVICE ENDPOINT HEALTH
# ============================================

# kube-dns Service Endpoint Availability
(
  kube_endpoint_address_available{
    namespace="kube-system",
    endpoint="kube-dns"
  }
  /
  (kube_endpoint_address_available{
    namespace="kube-system",
    endpoint="kube-dns"
  } + kube_endpoint_address_not_ready{
    namespace="kube-system",
    endpoint="kube-dns"
  })
) * 100

# Number of Ready CoreDNS Endpoints
kube_endpoint_address_available{
  namespace="kube-system",
  endpoint="kube-dns"
}
```

---

### Alert Definitions for DNS/Network

```promql
# ============================================
# CRITICAL: DNS AVAILABILITY ALERTS
# ============================================

# CRITICAL: Fast DNS Error Budget Burn (14.4x rate)
- alert: CoreDNSFastErrorBudgetBurn
  expr: |
    (
      1 - (
        sum(rate(coredns_dns_responses_total{rcode="NOERROR"}[1h]))
        /
        sum(rate(coredns_dns_responses_total[1h]))
      )
    ) > (14.4 * 0.0005)
    and
    (
      1 - (
        sum(rate(coredns_dns_responses_total{rcode="NOERROR"}[5m]))
        /
        sum(rate(coredns_dns_responses_total[5m]))
      )
    ) > (14.4 * 0.0005)
  for: 2m
  labels:
    severity: critical
    component: dns
    impact: platform-wide
    burn_rate: fast
    compliance_risk: critical
  annotations:
    summary: "CoreDNS burning error budget at 14.4x rate - severe DNS failures"
    description: |
      CRITICAL: DNS service severely degraded
      
      Current DNS Error Rate: {{ $value | humanizePercentage }}
      SLO Target: 99.95% success rate
      Error Budget Burn: 14.4x (30 days consumed in ~2 days)
      
      PLATFORM-WIDE IMPACT:
      - ALL microservices affected
      - Service discovery failing
      - Inter-service communication breaking
      - Payment processing disrupted
      - Customer-facing services likely down
      
      CASCADING FAILURES:
      - Applications timing out on DNS lookups
      - Connection pool exhaustion
      - Circuit breakers tripping
      - Database connection failures
      
      IMMEDIATE INVESTIGATION (P0):
      1. Check CoreDNS pod health:
         kubectl get pods -n kube-system -l k8s-app=kube-dns
      2. Review CoreDNS logs:
         kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100
      3. Check upstream DNS health (if using forwarding)
      4. Verify CoreDNS service endpoints:
         kubectl get endpoints -n kube-system kube-dns
      5. Check for DDoS or DNS amplification attacks
      
      ERROR BREAKDOWN:
      {{- range query "sum(rate(coredns_dns_responses_total{rcode!='NOERROR'}[5m])) by (rcode)" }}
      - {{ .Labels.rcode }}: {{ .Value | humanize }}/sec
      {{- end }}
      
      EMERGENCY MITIGATION:
      - Scale CoreDNS replicas immediately:
        kubectl scale deployment coredns -n kube-system --replicas=6
      - Check for resource exhaustion (CPU/memory limits)
      - Consider emergency rollback if recent CoreDNS config changes
      
      ESCALATION:
      - Page: SRE On-Call + Platform Lead
      - Notify: All application teams
      - Bridge: Immediately establish incident bridge
      - Compliance: Prepare incident report for audit
    runbook_url: "https://runbooks.bank.internal/sre/gke/dns-fast-burn"
    incident_severity: "P0"
    escalation: "immediate"
    bridge_required: "yes"

# WARNING: Slow DNS Error Budget Burn (6x rate)
- alert: CoreDNSSlowErrorBudgetBurn
  expr: |
    (
      1 - (
        sum(rate(coredns_dns_responses_total{rcode="NOERROR"}[6h]))
        /
        sum(rate(coredns_dns_responses_total[6h]))
      )
    ) > (6 * 0.0005)
    and
    (
      1 - (
        sum(rate(coredns_dns_responses_total{rcode="NOERROR"}[30m]))
        /
        sum(rate(coredns_dns_responses_total[30m]))
      )
    ) > (6 * 0.0005)
  for: 15m
  labels:
    severity: warning
    component: dns
    impact: platform-degradation
    burn_rate: slow
  annotations:
    summary: "CoreDNS burning error budget at 6x rate - elevated DNS failures"
    description: |
      WARNING: Elevated DNS error rate
      
      Current DNS Error Rate: {{ $value | humanizePercentage }}
      Error Budget Burn: 6x (30 days consumed in ~5 days)
      Time to Budget Exhaustion: ~5 days at current rate
      
      GRADUAL DEGRADATION:
      - Intermittent service failures
      - Increased latency across services
      - May escalate to fast burn
      
      INVESTIGATION PRIORITY:
      1. Analyze error patterns:
         - Which domains are failing?
         - Which response codes are elevated?
      2. Check CoreDNS resource utilization
      3. Review recent DNS configuration changes
      4. Check upstream DNS provider health
      5. Examine query patterns for anomalies
      
      ERROR ANALYSIS:
      {{- range query "topk(5, sum(rate(coredns_dns_responses_total{rcode!='NOERROR'}[30m])) by (rcode))" }}
      - {{ .Labels.rcode }}: {{ .Value | humanize }}/sec
      {{- end }}
      
      BANKING CONTEXT:
      - May correlate with specific business flows
      - Check if errors align with transaction peaks
      - Review if specific services/databases affected
      
      PREVENTIVE ACTIONS:
      - Monitor closely for escalation to fast burn
      - Plan capacity increases if resource-constrained
      - Review DNS caching configuration
      - Consider NodeLocal DNSCache if not deployed
    runbook_url: "https://runbooks.bank.internal/sre/gke/dns-slow-burn"
    incident_severity: "P2"

# ============================================
# DNS LATENCY ALERTS
# ============================================

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
    slo_type: latency
    impact: performance
  annotations:
    summary: "CoreDNS P95 latency exceeds 50ms SLO"
    description: |
      WARNING: DNS query latency elevated
      
      Current P95 Latency: {{ $value | humanizeDuration }}
      SLO Target: 50ms
      Duration: 10+ minutes
      
      PERFORMANCE IMPACT:
      - Slower service-to-service calls
      - Increased transaction processing time
      - Degraded user experience
      - Timeout risk for latency-sensitive services
      
      BANKING IMPACT:
      - Payment processing slower
      - Real-time trading systems affected
      - Customer response times increased
      
      INVESTIGATION:
      1. Check DNS cache hit ratio:
         Cache Miss = Slow upstream query
      2. Review CoreDNS CPU/memory usage:
         kubectl top pods -n kube-system -l k8s-app=kube-dns
      3. Check upstream DNS latency:
         Examine coredns_forward_request_duration_seconds
      4. Verify CoreDNS not CPU throttled
      5. Check for query spikes or unusual patterns
      
      LATENCY BY QUERY TYPE:
      {{- range query "histogram_quantile(0.95, sum(rate(coredns_dns_request_duration_seconds_bucket[10m])) by (le, type)) * 1000 > 50" }}
      - {{ .Labels.type }}: {{ .Value | humanize }}ms
      {{- end }}
      
      CACHE EFFICIENCY:
      Current Cache Hit Ratio: 
      {{ with query "(sum(rate(coredns_cache_hits_total[10m])) / (sum(rate(coredns_cache_hits_total[10m])) + sum(rate(coredns_cache_misses_total[10m])))) * 100" }}
      {{ . | first | value | humanize }}%
      {{- end }}
      
      MITIGATION:
      - Increase CoreDNS cache size/TTL if low hit ratio
      - Scale CoreDNS replicas if CPU-bound
      - Deploy NodeLocal DNSCache for reduced latency
      - Review upstream DNS provider performance
    runbook_url: "https://runbooks.bank.internal/sre/gke/dns-latency-p95"

# CRITICAL: DNS P99 Latency Critical
- alert: CoreDNSLatencyP99Critical
  expr: |
    histogram_quantile(0.99,
      sum(rate(coredns_dns_request_duration_seconds_bucket[5m])) by (le)
    ) > 0.100  # 100ms
  for: 5m
  labels:
    severity: critical
    component: dns
    slo_type: latency
    impact: severe-performance
  annotations:
    summary: "CoreDNS P99 latency critically high (>100ms)"
    description: |
      CRITICAL: DNS tail latency unacceptable
      
      Current P99 Latency: {{ $value | humanizeDuration }}
      Threshold: 100ms
      
      SEVERE IMPACT:
      - 1% of queries experiencing severe delays
      - Application timeouts likely
      - Circuit breakers may trip
      - Cascading failures possible
      
      IMMEDIATE ACTIONS:
      1. Check for DNS amplification/DDoS attacks
      2. Review CoreDNS resource constraints
      3. Check upstream DNS health
      4. Consider emergency CoreDNS restart if hung
      
      This often indicates:
      - CoreDNS resource exhaustion
      - Upstream DNS provider issues
      - Network path problems
      - Excessive query load
    runbook_url: "https://runbooks.bank.internal/sre/gke/dns-latency-p99"
    incident_severity: "P1"

# ============================================
# COREDNS POD HEALTH ALERTS
# ============================================

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
    impact: availability
  annotations:
    summary: "Less than 50% of CoreDNS pods are running"
    description: |
      CRITICAL: CoreDNS pod availability severely degraded
      
      Running Pods: {{ $value | humanizePercentage }} of desired
      
      IMMEDIATE RISK:
      - DNS resolution capacity severely reduced
      - High query latency expected
      - Service failures imminent
      
      INVESTIGATION:
      1. Check pod status:
         kubectl get pods -n kube-system -l k8s-app=kube-dns
      2. Check pod events:
         kubectl describe pods -n kube-system -l k8s-app=kube-dns
      3. Check node capacity
      4. Review recent changes
      
      COMMON CAUSES:
      - Node failures
      - Resource constraints
      - Image pull failures
      - Configuration errors
      - Node anti-affinity preventing scheduling
    runbook_url: "https://runbooks.bank.internal/sre/gke/coredns-pod-availability"
    incident_severity: "P0"

# WARNING: CoreDNS Pods Restarting
- alert: CoreDNSPodsRestarting
  expr: |
    rate(kube_pod_container_status_restarts_total{
      namespace="kube-system",
      pod=~"coredns.*"
    }[30m]) > 0.02  # More than 1 restart per ~50 minutes
  for: 10m
  labels:
    severity: warning
    component: dns
    impact: stability
  annotations:
    summary: "CoreDNS pods restarting frequently"
    description: |
      WARNING: CoreDNS instability detected
      
      Pod: {{ $labels.pod }}
      Restart Rate: {{ $value }}/sec
      
      STABILITY CONCERNS:
      - OOMKilled events likely
      - Configuration issues possible
      - Memory leak potential
      
      INVESTIGATION:
      1. Check restart reason:
         kubectl describe pod {{ $labels.pod }} -n kube-system
      2. Review CoreDNS logs before restart:
         kubectl logs {{ $labels.pod }} -n kube-system --previous
      3. Check memory usage patterns
      4. Review CoreDNS configuration
      
      COMMON CAUSES:
      - Memory limits too low
      - Memory leak in plugins
      - Configuration syntax errors
      - Resource exhaustion under load
      
      BANKING CONTEXT:
      - Restarts during business hours = service disruption
      - Plan changes during maintenance windows
    runbook_url: "https://runbooks.bank.internal/sre/gke/coredns-restarts"

# WARNING: CoreDNS Service Endpoints Missing
- alert: CoreDNSNoEndpoints
  expr: |
    kube_endpoint_address_available{
      namespace="kube-system",
      endpoint="kube-dns"
    } == 0
  for: 2m
  labels:
    severity: critical
    component: dns
    impact: total-failure
  annotations:
    summary: "kube-dns service has no available endpoints"
    description: |
      CRITICAL: DNS service completely unavailable
      
      TOTAL PLATFORM FAILURE:
      - NO DNS resolution possible
      - All pod-to-pod communication failing
      - Cluster effectively non-functional
      
      IMMEDIATE ACTIONS:
      1. Check CoreDNS deployment:
         kubectl get deployment coredns -n kube-system
      2. Check pod status:
         kubectl get pods -n kube-system -l k8s-app=kube-dns
      3. Check service:
         kubectl get service kube-dns -n kube-system
      4. Check endpoints:
         kubectl get endpoints kube-dns -n kube-system
      
      This is a P0 incident - engage all hands
    runbook_url: "https://runbooks.bank.internal/sre/gke/coredns-no-endpoints"
    incident_severity: "P0"
    escalation: "all-hands"

# ============================================
# DNS RESOURCE UTILIZATION ALERTS
# ============================================

# WARNING: CoreDNS CPU Throttling
- alert: CoreDNSCPUThrottling
  expr: |
    (
      sum(rate(container_cpu_cfs_throttled_seconds_total{
        namespace="kube-system",
        pod=~"coredns.*"
      }[5m])) by (pod)
      /
      sum(rate(container_cpu_usage_seconds_total{
        namespace="kube-system",
        pod=~"coredns.*"
      }[5m])) by (pod)
    ) > 0.25
  for: 15m
  labels:
    severity: warning
    component: dns
    impact: performance
  annotations:
    summary: "CoreDNS pod {{ $labels.pod }} experiencing CPU throttling"
    description: |
      WARNING: CoreDNS CPU throttled
      
      Pod: {{ $labels.pod }}
      Throttling Rate: {{ $value | humanizePercentage }}
      Threshold: 25%
      
      PERFORMANCE IMPACT:
      - Increased DNS query latency
      - Reduced query throughput
      - Potential timeout issues
      
      INVESTIGATION:
      1. Review CPU requests vs limits:
         kubectl describe pod {{ $labels.pod }} -n kube-system
      2. Check current CPU usage:
         kubectl top pod {{ $labels.pod }} -n kube-system
      3. Review query load patterns
      
      MITIGATION:
      - Increase CPU limits for CoreDNS
      - Scale CoreDNS replicas
      - Review CoreDNS plugin configuration (some are CPU-intensive)
      
      BANKING CONSIDERATION:
      - Correlate with business transaction volumes
      - May need capacity increases during peak periods
    runbook_url: "https://runbooks.bank.internal/sre/gke/coredns-cpu-throttling"

# WARNING: CoreDNS Memory Usage High
- alert: CoreDNSMemoryUsageHigh
  expr: |
    (
      sum(container_memory_working_set_bytes{
        namespace="kube-system",
        pod=~"coredns.*",
        container="coredns"
      }) by (pod)
      /
      sum(kube_pod_container_resource_limits{
        namespace="kube-system",
        pod=~"coredns.*",
        container="coredns",
        resource="memory"
      }) by (pod)
    ) > 0.85
  for: 10m
  labels:
    severity: warning
    component: dns
    impact: stability
  annotations:
    summary: "CoreDNS pod {{ $labels.pod }} memory usage at {{ $value | humanizePercentage }}"
    description: |
      WARNING: CoreDNS approaching memory limit
      
      Pod: {{ $labels.pod }}
      Memory Usage: {{ $value | humanizePercentage }} of limit
      
      OOM RISK:
      - Approaching memory limit
      - OOMKill and restart likely
      - Service disruption imminent
      
      INVESTIGATION:
      1. Check memory usage trend
      2. Review cache size configuration
      3. Check for memory leaks
      4. Examine query patterns for abuse
      
      IMMEDIATE MITIGATION:
      - Increase memory limits if legitimate usage
      - Reduce cache size if over-configured
      - Investigate potential memory leaks
      
      PREVENTIVE:
      - Implement VPA for CoreDNS
      - Review cache TTL settings
    runbook_url: "https://runbooks.bank.internal/sre/gke/coredns-memory"

# ============================================
# DNS UPSTREAM & FORWARDING ALERTS
# ============================================

# CRITICAL: Upstream DNS Server Failures
- alert: UpstreamDNSServerFailures
  expr: |
    rate(coredns_forward_healthcheck_failures_total[5m]) > 0.1
  for: 5m
  labels:
    severity: critical
    component: dns
    impact: external-dependency
  annotations:
    summary: "Upstream DNS server {{ $labels.to }} healthcheck failing"
    description: |
      CRITICAL: External DNS dependency failure
      
      Upstream Server: {{ $labels.to }}
      Failure Rate: {{ $value }}/sec
      
      IMPACT:
      - External domain resolution failing
      - Internet-facing services affected
      - Cloud API calls may fail (*.googleapis.com)
      
      BANKING SERVICES AFFECTED:
      - External payment gateways
      - Credit bureau integrations
      - Third-party KYC services
      - Cloud provider APIs
      
      INVESTIGATION:
      1. Verify upstream DNS server reachability
      2. Check network connectivity/firewall rules
      3. Review upstream DNS provider status
      4. Check for DNS configuration errors
      
      FAILOVER OPTIONS:
      - Switch to backup upstream DNS servers
      - Use alternative DNS providers (8.8.8.8, 1.1.1.1)
      - Review fallback configuration
      
      COMPLIANCE NOTE:
      - External integrations failing may affect regulatory reporting
    runbook_url: "https://runbooks.bank.internal/sre/gke/upstream-dns-failures"
    incident_severity: "P1"

# WARNING: High Upstream DNS Latency
- alert: UpstreamDNSLatencyHigh
  expr: |
    histogram_quantile(0.95,
      sum(rate(coredns_forward_request_duration_seconds_bucket[10m])) 
      by (le, to)
    ) > 0.200  # 200ms
  for: 10m
  labels:
    severity: warning
    component: dns
    impact: external-performance
  annotations:
    summary: "High latency to upstream DNS server {{ $labels.to }}"
    description: |
      WARNING: Slow upstream DNS resolution
      
      Upstream Server: {{ $labels.to }}
      P95 Latency: {{ $value | humanizeDuration }}
      Threshold: 200ms
      
      IMPACT:
      - Slow resolution for external domains
      - Increased overall DNS latency
      - Cache misses more expensive
      
      INVESTIGATION:
      1. Check network path to upstream DNS
      2. Verify upstream DNS provider status
      3. Consider geographic proximity
      4. Review firewall/proxy latency
      
      MITIGATION:
      - Use geographically closer DNS servers
      - Increase cache TTLs where appropriate
      - Consider multiple upstream providers
    runbook_url: "https://runbooks.bank.internal/sre/gke/upstream-dns-latency"

# ============================================
# DNS QUERY PATTERN ALERTS
# ============================================

# WARNING: DNS Query Spike
- alert: DNSQuerySpike
  expr: |
    (
      sum(rate(coredns_dns_requests_total[5m]))
      /
      avg_over_time(sum(rate(coredns_dns_requests_total[5m]))[1h:5m])
    ) > 3  # 3x baseline
  for: 10m
  labels:
    severity: warning
    component: dns
    impact: capacity
    security_risk: medium
  annotations:
    summary: "DNS query rate {{ $value }}x higher than baseline"
    description: |
      WARNING: Unusual DNS query spike detected
      
      Current QPS: {{ with query "sum(rate(coredns_dns_requests_total[5m]))" }}{{ . | first | value | humanize }}{{- end }}
      Baseline QPS: {{ with query "avg_over_time(sum(rate(coredns_dns_requests_total[5m]))[1h:5m])" }}{{ . | first | value | humanize }}{{- end }}
      Multiplier: {{ $value }}x
      
      POSSIBLE CAUSES:
      1. Legitimate traffic spike (business event)
      2. Application misconfiguration (query loop)
      3. DNS amplification attack
      4. New service deployment
      
      INVESTIGATION:
      1. Identify top queried domains
      2. Check query type distribution
      3. Review recent deployments
      4. Check for security incidents
      
      QUERY TYPE BREAKDOWN:
      {{- range query "topk(5, sum(rate(coredns_dns_requests_total[5m])) by (type))" }}
      - {{ .Labels.type }}: {{ .Value | humanize }}/sec
      {{- end }}
      
      BANKING SECURITY:
      - Could indicate data exfiltration attempt
      - Review for signs of compromise
      - Check correlation with other security alerts
      
      CAPACITY PLANNING:
      - May need to scale CoreDNS
      - Review rate limiting configuration
    runbook_url: "https://runbooks.bank.internal/sre/gke/dns-query-spike"
    security_review: "recommended"

# WARNING: High NXDOMAIN Rate
- alert: HighNXDOMAINRate
  expr: |
    (
      sum(rate(coredns_dns_responses_total{rcode="NXDOMAIN"}[10m]))
      /
      sum(rate(coredns_dns_responses_total[10m]))
    ) > 0.10  # 10% of queries
  for: 10m
  labels:
    severity: warning
    component: dns
    impact: configuration
    security_risk: medium
  annotations:
    summary: "High rate of NXDOMAIN responses ({{ $value | humanizePercentage }})"
    description: |
      WARNING: Many DNS queries for non-existent domains
      
      NXDOMAIN Rate: {{ $value | humanizePercentage }}
      Threshold: 10%
      
      POSSIBLE CAUSES:
      1. Application misconfiguration
      2. Hard-coded service names (old/removed services)
      3. DNS typos in configuration
      4. DNS tunneling/exfiltration attempt
      5. Malware C2 communication attempts
      
      INVESTIGATION PRIORITY:
      1. Identify which domains are being queried:
         Review CoreDNS query logs
      2. Identify source pods/services
      3. Check for security incidents
      4. Review recent service removals
      
      BANKING SECURITY CONTEXT:
      - High NXDOMAIN can indicate:
        * Compromised workload trying to reach C2 servers
        * Data exfiltration via DNS tunneling
        * Malware attempting to spread
      
      IMMEDIATE ACTIONS:
      1. Enable detailed DNS query logging
      2. Identify top NXDOMAIN queries
      3. Correlate with security monitoring
      4. Review pod network policies
      
      REMEDIATION:
      - Fix application configurations
      - Update service discovery configs
      - Implement DNS firewall rules if malicious
      - Consider egress filtering
    runbook_url: "https://runbooks.bank.internal/sre/gke/high-nxdomain"
    security_review: "required"

# CRITICAL: DNS Cache Disabled/Ineffective
- alert: DNSCacheHitRatioLow
  expr: |
    (
      sum(rate(coredns_cache_hits_total[30m]))
      /
      (sum(rate(coredns_cache_hits_total[30m])) + sum(rate(coredns_cache_misses_total[30m])))
    ) < 0.50
  for: 15m
  labels:
    severity: warning
    component: dns
    impact: performance
  annotations:
    summary: "DNS cache hit ratio critically low ({{ $value | humanizePercentage }})"
    description: |
      WARNING: DNS cache ineffective
      
      Cache Hit Ratio: {{ $value | humanizePercentage }}
      Expected: >50% (typically 80-95%)
      
      PERFORMANCE IMPACT:
      - Most queries hitting upstream DNS
      - Increased latency for all queries
      - Higher load on upstream servers
      - Wasted network bandwidth
      
      INVESTIGATION:
      1. Check if cache is enabled in CoreDNS config
      2. Review cache size configuration
      3. Check TTL settings
      4. Examine query patterns:
         - Are queries unique (no repeats)?
         - Are TTLs too short?
      
      CONFIGURATION REVIEW:
      1. Check CoreDNS ConfigMap:
         kubectl get configmap coredns -n kube-system -o yaml
      2. Verify cache plugin configuration
      3. Review cache size vs query volume
      
      COMMON CAUSES:
      - Cache disabled in configuration
      - Cache size too small
      - TTLs set too low
      - High rate of unique queries
      
      OPTIMIZATION:
      - Increase cache size if capacity allows
      - Review upstream TTL settings
      - Enable negative caching
      - Consider NodeLocal DNSCache deployment
    runbook_url: "https://runbooks.bank.internal/sre/gke/dns-cache-low"
```

---

## 1.7 Service Mesh / Network Connectivity SLOs

**SLO Definition**:
- 99.95% service endpoint availability
- Service-to-service connection success rate: 99.9%
- Network policy enforcement: 100% (security requirement)

**Business Rationale**:
- **Banking Critical**: Service mesh is the backbone of microservices communication
- Payment flows involve 10-15 service hops - any failure cascades
- Network policies enforce PCI-DSS segmentation requirements
- Service discovery failures = transaction failures

---

### PromQL Queries for Service/Network Monitoring

```promql
# ============================================
# SERVICE ENDPOINT HEALTH
# ============================================

# Service Endpoint Availability
(
  kube_endpoint_address_available
  /
  (kube_endpoint_address_available + kube_endpoint_address_not_ready)
) * 100

# Services with No Ready Endpoints
count(
  (kube_endpoint_address_available + kube_endpoint_address_not_ready) == 0
) by (namespace, endpoint)

# Services with Partial Endpoint Availability
count(
  (kube_endpoint_address_available > 0) 
  and 
  (kube_endpoint_address_not_ready > 0)
) by (namespace, endpoint)

# Service Endpoint Ready Count
kube_endpoint_address_available

# Service Endpoint Not Ready Count
kube_endpoint_address_not_ready

# ============================================
# NETWORK CONNECTIVITY (if using service mesh like Istio)
# ============================================

# Service-to-Service Connection Success Rate
(
  sum(rate(istio_requests_total{response_code!~"5.."}[5m]))
  /
  sum(rate(istio_requests_total[5m]))
) * 100

# Connection Failures by Service
sum(rate(istio_tcp_connections_closed_total[5m])) 
by (source_workload, destination_workload)

# Network Policy Deny Count
sum(rate(cilium_policy_l3_l4_denied_total[5m])) 
by (source, destination)
```

### Alert Definitions for Service/Network

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
    impact: service-unavailable
  annotations:
    summary: "Service {{ $labels.namespace }}/{{ $labels.endpoint }} has zero endpoints"
    description: |
      CRITICAL: Service completely unavailable
      
      Service: {{ $labels.namespace }}/{{ $labels.endpoint }}
      Available Endpoints: 0
      
      TOTAL SERVICE FAILURE:
      - Service cannot route traffic
      - All clients will fail
      - Cascading failures likely
      
      IMMEDIATE INVESTIGATION:
      1. Check backend pods:
         kubectl get pods -n {{ $labels.namespace }} -l <service-selector>
      2. Verify pod readiness probes
      3. Check service selector matches pods
      4. Review recent deployments
      
      BANKING IMPACT:
      - Check if customer-facing service
      - Assess transaction processing impact
      - Coordinate with business teams
    runbook_url: "https://runbooks.bank.internal/sre/gke/service-no-endpoints"
    incident_severity: "P1"

# WARNING: Service Endpoint Availability Low
- alert: ServiceEndpointAvailabilityLow
  expr: |
    (
      sum(kube_endpoint_address_available) by (namespace, endpoint)
      /
      sum(kube_endpoint_address_available + kube_endpoint_address_not_ready) by (namespace, endpoint)
    ) < 0.50
  for: 5m
  labels:
    severity: warning
    component: networking
    impact: reduced-capacity
  annotations:
    summary: "Service {{ $labels.namespace }}/{{ $labels.endpoint }} only {{ $value | humanizePercentage }} endpoints available"
    description: |
      WARNING: Service degraded capacity
      
      Service: {{ $labels.namespace }}/{{ $labels.endpoint }}
      Available: {{ $value | humanizePercentage }}
      
      REDUCED CAPACITY:
      - Less than 50% endpoints ready
      - Traffic concentration on fewer pods
      - Overload and failures likely
      
      INVESTIGATION:
      1. Check not-ready endpoints
      2. Review pod health checks
      3. Check for resource exhaustion
      4. Review rolling updates
    runbook_url: "https://runbooks.bank.internal/sre/gke/service-low-availability"
```

---

## Detailed Rationale for DNS/Network SLOs

### Why 99.95% for DNS?

1. **Mathematical Impact**: 
   - Average payment transaction requires ~50 DNS queries
   - 0.05% failure rate = 2.5 failures per 5000 transactions
   - With 1M transactions/day = 500 affected transactions
   - Banking: Unacceptable customer experience

2. **Cascading Failures**:
   - DNS failure causes connection timeout (30-60s typically)
   - Timeouts trigger retries
   - Retries create amplification (1 failure → 3 attempts)
   - Circuit breakers trip, affecting healthy services

3. **Regulatory Compliance**:
   - PCI-DSS requires 99.5% uptime minimum
   - Internal audit trails depend on service discovery
   - DNS == infrastructure, held to higher standard

### Why Measure P95 and P99 Latency Separately?

1. **P95 (50ms target)**:
   - Represents normal operational quality
   - 95% of queries complete within 50ms
   - Banking: Acceptable for transaction processing

2. **P99 (100ms target)**:
   - Catches tail latency affecting critical paths
   - Even 1% slow queries = thousands per day
   - High-value transactions often hit P99 due to complexity

3. **Different Root Causes**:
   - P95 violations: Capacity, caching, upstream latency
   - P99 violations: Resource contention, GC pauses, network issues

### Why Track Upstream DNS Separately?

1. **External Dependency Risk**:
   - Banking integrates with external systems (payment gateways, credit bureaus)
   - Upstream DNS failure = cannot reach external APIs
   - Need separate monitoring to distinguish internal vs external issues

2. **Compliance**:
   - Regulators require proof of due diligence on third-party dependencies
   - Separate metrics demonstrate monitoring rigor
