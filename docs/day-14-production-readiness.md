# Day 14: Production Readiness & Best Practices

## The Space Launch Checklist Analogy

Launching a rocket requires extensive checklists. Same for production Kubernetes!

```
┌──────────────────────────────────────────────────────┐
│         PRODUCTION READINESS CHECKLIST                │
│                                                       │
│  ☐ SAFETY SYSTEMS:                                   │
│    ✓ High Availability (3+ control planes)           │
│    ✓ Backups (etcd snapshots)                        │
│    ✓ Disaster Recovery (tested!)                     │
│                                                       │
│  ☐ MONITORING:                                       │
│    ✓ Metrics (Prometheus)                            │
│    ✓ Logging (ELK/Loki)                              │
│    ✓ Alerting (PagerDuty/Slack)                      │
│    ✓ Dashboards (Grafana)                            │
│                                                       │
│  ☐ SECURITY:                                         │
│    ✓ RBAC configured                                 │
│    ✓ Network Policies                                │
│    ✓ Pod Security Standards                          │
│    ✓ Secrets encrypted                               │
│    ✓ Image scanning                                  │
│                                                       │
│  ☐ RELIABILITY:                                      │
│    ✓ Resource limits on ALL pods                     │
│    ✓ Health checks (liveness/readiness)             │
│    ✓ PodDisruptionBudgets                            │
│    ✓ Auto-scaling (HPA)                              │
│                                                       │
│  ☐ OPERATIONS:                                       │
│    ✓ GitOps (Infrastructure as Code)                 │
│    ✓ CI/CD pipelines                                 │
│    ✓ Runbooks documented                             │
│    ✓ On-call rotation                                │
│                                                       │
│  ☐ COST OPTIMIZATION:                                │
│    ✓ Resource requests = actual usage                │
│    ✓ Cluster autoscaler                              │
│    ✓ Spot instances for batch                        │
│    ✓ Cost monitoring/alerts                          │
└──────────────────────────────────────────────────────┘

ALL BOXES CHECKED? 
READY FOR PRODUCTION! 🚀
```

---

## High Availability Setup

**Production Control Plane:**

```
Development:
- 1 control plane node (single point of failure) ❌

Production:
- 3 control plane nodes (can survive 1 failure) ✓
- 5 control plane nodes (can survive 2 failures) ✓✓

Why odd numbers?
- etcd needs majority for quorum
- 3 nodes: Need 2 for quorum (can lose 1)
- 4 nodes: Need 3 for quorum (can lose 1) ← Same as 3!
- 5 nodes: Need 3 for quorum (can lose 2) ✓

3 nodes = most common
5 nodes = critical systems only
```

---

## Resource Management

**Golden Rule: Set limits on EVERYTHING!**

```yaml
# BAD - No limits
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
spec:
  containers:
  - name: app
    image: myapp:latest

# Can use unlimited CPU/memory!
# One pod can starve entire node! 💥

# GOOD - With limits
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        cpu: 100m        # Minimum guaranteed
        memory: 256Mi
      limits:
        cpu: 500m        # Maximum allowed
        memory: 512Mi

# Best practice: requests = limits (for predictable performance)
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 512Mi

# QoS Classes:
# 1. Guaranteed: requests = limits (best!)
# 2. Burstable: requests < limits (okay)
# 3. BestEffort: no limits (bad for production!)
```

---

## Health Checks Best Practices

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: production-app
spec:
  containers:
  - name: app
    image: myapp:v2
    ports:
    - containerPort: 8080
    
    # Startup probe (for slow-starting apps)
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      failureThreshold: 30  # 5 minutes to start
    
    # Readiness probe (remove from Service when busy)
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      successThreshold: 1
      failureThreshold: 3
    
    # Liveness probe (restart if deadlocked)
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3

Best practices:
✓ Use all three probes appropriately
✓ Readiness should be quick (not database query!)
✓ Liveness should be simple (just "am I alive?")
✓ Set generous timeouts (avoid false positives)
✓ Don't use liveness if readiness is enough
```

---

## PodDisruptionBudgets

**Prevent too many pods going down during maintenance:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2  # At least 2 pods must stay running
  selector:
    matchLabels:
      app: web

# Or use maxUnavailable:
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  maxUnavailable: 1  # At most 1 pod can be down
  selector:
    matchLabels:
      app: api

Use case:
kubectl drain node-1

Without PDB:
- All pods on node-1 evicted immediately
- If that's 10/10 web pods → OUTAGE! 💥

With PDB (minAvailable: 2):
- Evict pods one by one
- Wait for replacements to be Ready
- Always maintain 2+ pods Running
- No outage! ✓

PDB is your safety net during:
✓ Node upgrades
✓ Cluster scaling down
✓ Node maintenance
✓ Voluntary evictions
```

---

## Auto-Scaling

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 3
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50        # Max 50% increase at once
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1         # Remove max 1 pod at a time
        periodSeconds: 60

How it works:
Current: 3 pods at 80% CPU
       ↓
HPA: "CPU > 70% target, scale up!"
       ↓
Calculates: 3 * (80/70) = 3.4 → 4 pods
       ↓
Creates 1 new pod
       ↓
Wait 60 seconds (stabilization)
       ↓
Check again: 4 pods at 60% CPU
       ↓
HPA: "Within target, no change needed"

Traffic drops:
4 pods at 30% CPU
       ↓
HPA: "CPU < 70% target, can scale down"
       ↓
But wait 300 seconds (stabilization)
       ↓
Still low? Remove 1 pod
       ↓
3 pods at 40% CPU
```

---

### Cluster Autoscaler

```yaml
# Automatically adds/removes nodes

When HPA can't schedule pods:
1. HPA wants to scale from 50 to 60 pods
2. Only 55 pods fit on current nodes
3. 5 pods stay Pending (insufficient resources)
       ↓
Cluster Autoscaler sees Pending pods
       ↓
Adds nodes to cluster
       ↓
Pending pods schedule on new nodes
       ↓
All 60 pods Running! ✓

When nodes underutilized:
1. Nodes at 20% utilization
2. Pods can fit on fewer nodes
       ↓
Cluster Autoscaler:
- Drains underutilized node
- Pods rescheduled elsewhere
- Node removed from cluster
       ↓
Cost savings! 💰

Configuration:
--min-nodes=3
--max-nodes=100
--scale-down-delay-after-add=10m
--scale-down-unneeded-time=10m

Works with:
✓ AWS Auto Scaling Groups
✓ GCP Managed Instance Groups
✓ Azure Virtual Machine Scale Sets
```

---

## Monitoring Stack

**Production monitoring setup:**

```
┌─────────────────────────────────────────────────────┐
│                  MONITORING STACK                    │
│                                                      │
│  METRICS (What's happening now?):                   │
│  ┌────────────────────────────────┐                 │
│  │ Prometheus                     │                 │
│  │ - Scrapes metrics every 15s    │                 │
│  │ - Stores time-series data      │                 │
│  │ - 15 day retention             │                 │
│  └────────────────────────────────┘                 │
│           ↓                                          │
│  ┌────────────────────────────────┐                 │
│  │ Grafana                        │                 │
│  │ - Dashboards                   │                 │
│  │ - Visualizations               │                 │
│  │ - Real-time graphs             │                 │
│  └────────────────────────────────┘                 │
│                                                      │
│  LOGS (What happened?):                             │
│  ┌────────────────────────────────┐                 │
│  │ Fluent-bit (on each node)      │                 │
│  │ - Collects container logs      │                 │
│  │ - Ships to Elasticsearch       │                 │
│  └────────────────────────────────┘                 │
│           ↓                                          │
│  ┌────────────────────────────────┐                 │
│  │ Elasticsearch / Loki           │                 │
│  │ - Stores logs                  │                 │
│  │ - Full-text search             │                 │
│  │ - 30 day retention             │                 │
│  └────────────────────────────────┘                 │
│           ↓                                          │
│  ┌────────────────────────────────┐                 │
│  │ Kibana / Grafana               │                 │
│  │ - Log search                   │                 │
│  │ - Log analysis                 │                 │
│  └────────────────────────────────┘                 │
│                                                      │
│  ALERTS (Something wrong!):                         │
│  ┌────────────────────────────────┐                 │
│  │ Alertmanager                   │                 │
│  │ - Receives alerts              │                 │
│  │ - Groups/deduplicates          │                 │
│  │ - Routes to receivers          │                 │
│  └────────────────────────────────┘                 │
│           ↓                                          │
│  ┌─────────────┐  ┌──────────────┐                 │
│  │ PagerDuty   │  │ Slack        │                 │
│  │ (Critical)  │  │ (Warning)    │                 │
│  └─────────────┘  └──────────────┘                 │
└─────────────────────────────────────────────────────┘
```

---

## Essential Alerts

```yaml
# Critical alerts (page on-call)
groups:
- name: critical
  rules:
  # API Server down
  - alert: APIServerDown
    expr: up{job="apiserver"} == 0
    for: 5m
    severity: critical
    
  # etcd down
  - alert: etcdDown
    expr: up{job="etcd"} == 0
    for: 5m
    severity: critical
    
  # Node down
  - alert: NodeDown
    expr: up{job="kubelet"} == 0
    for: 10m
    severity: critical
    
  # Pod crash looping
  - alert: PodCrashLooping
    expr: rate(kube_pod_container_status_restarts_total[15m]) > 0.1
    for: 5m
    severity: critical
    
  # High error rate
  - alert: HighErrorRate
    expr: |
      sum(rate(http_requests_total{status=~"5.."}[5m])) /
      sum(rate(http_requests_total[5m])) > 0.05
    for: 5m
    severity: critical
    annotations:
      summary: "Error rate >5%"

# Warning alerts (Slack notification)
- name: warnings
  rules:
  # CPU high
  - alert: HighCPU
    expr: avg(rate(container_cpu_usage_seconds_total[5m])) > 0.8
    for: 15m
    severity: warning
    
  # Memory high
  - alert: HighMemory
    expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
    for: 15m
    severity: warning
    
  # Disk full soon
  - alert: DiskFillingUp
    expr: predict_linear(node_filesystem_free_bytes[6h], 4*3600) < 0
    for: 1h
    severity: warning
    annotations:
      summary: "Disk will be full in 4 hours"
```

---

## Backup Strategy

**Critical: Backup etcd regularly!**

```yaml
# CronJob for etcd backup
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  successfulJobsHistoryLimit: 7  # Keep 1 week
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: k8s.gcr.io/etcd:3.5.0
            command:
            - /bin/sh
            - -c
            - |
              BACKUP_FILE="/backup/etcd-$(date +%Y%m%d-%H%M%S).db"
              
              # Create snapshot
              ETCDCTL_API=3 etcdctl snapshot save $BACKUP_FILE \
                --endpoints=https://127.0.0.1:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/server.crt \
                --key=/etc/kubernetes/pki/etcd/server.key
              
              # Upload to S3
              aws s3 cp $BACKUP_FILE s3://backups/etcd/
              
              # Keep last 30 days locally
              find /backup -name "etcd-*.db" -mtime +30 -delete
              
              echo "Backup complete: $BACKUP_FILE"
            volumeMounts:
            - name: backup
              mountPath: /backup
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
          volumes:
          - name: backup
            hostPath:
              path: /var/etcd-backup
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          hostNetwork: true
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""

Backup checklist:
✓ Automated (CronJob)
✓ Frequent (every 6 hours)
✓ Off-site (S3)
✓ Tested (restore test monthly!)
✓ Monitored (alert if backup fails)
```

---

## GitOps - Infrastructure as Code

```
Git Repository:
├── clusters/
│   ├── production/
│   │   ├── namespaces/
│   │   │   ├── default.yaml
│   │   │   ├── monitoring.yaml
│   │   │   └── ingress.yaml
│   │   ├── applications/
│   │   │   ├── web-app.yaml
│   │   │   ├── api.yaml
│   │   │   └── database.yaml
│   │   └── infrastructure/
│   │       ├── prometheus.yaml
│   │       ├── grafana.yaml
│   │       └── ingress-nginx.yaml
│   ├── staging/
│   └── development/

All changes via Git:
1. Developer commits change
2. Pull request created
3. Automated tests run
4. Team reviews
5. Merged to main branch
6. ArgoCD syncs to cluster
7. Change deployed!

Benefits:
✓ Version control (Git history)
✓ Audit trail (who changed what)
✓ Rollback (Git revert)
✓ Disaster recovery (redeploy from Git)
✓ Collaboration (PR reviews)
✓ Consistency (same process everywhere)
```

---

## Cost Optimization

**1. Right-size resource requests:**
```yaml
# Before (wasteful):
resources:
  requests:
    cpu: 2000m      # Actual usage: 200m
    memory: 8Gi     # Actual usage: 500Mi

# Wasting: 1800m CPU, 7.5Gi memory!

# After (optimized):
resources:
  requests:
    cpu: 300m       # 200m + 50% headroom
    memory: 750Mi   # 500Mi + 50% headroom

# Savings: 80% CPU, 90% memory!
```

**2. Use Cluster Autoscaler:**
```
Without autoscaler:
- Provision for peak load
- 100 nodes running 24/7
- Low utilization off-peak
- Cost: $10,000/month

With autoscaler:
- Peak: 100 nodes
- Off-peak: 30 nodes
- Average: 50 nodes
- Cost: $5,000/month

50% cost savings! 💰
```

**3. Spot instances for batch workloads:**
```yaml
# Batch jobs can use spot instances
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-process
spec:
  template:
    spec:
      nodeSelector:
        node-lifecycle: spot
      tolerations:
      - key: "spot"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
      containers:
      - name: batch
        image: batch-processor:latest

Spot instances:
- 70% cheaper than on-demand!
- Can be terminated anytime
- Good for:
  ✓ Batch jobs (can restart)
  ✓ CI/CD builds
  ✓ Data processing
- Bad for:
  ❌ Production services
  ❌ Databases
  ❌ Stateful apps
```

---

## Disaster Recovery Procedures

**Monthly DR drill:**

```bash
# 1. Simulate cluster failure
# Delete all pods in production namespace
kubectl delete pods --all -n production

# 2. Verify auto-recovery
# Deployments should recreate pods
watch kubectl get pods -n production

# Expected: All pods return to Running within 5 minutes

# 3. Simulate complete cluster loss
# Switch to DR cluster
kubectl config use-context dr-cluster

# 4. Restore from etcd backup
# On DR cluster control plane:
ETCDCTL_API=3 etcdctl snapshot restore /backup/latest.db \
  --data-dir=/var/lib/etcd-restore \
  --initial-cluster=...

# 5. Verify services
kubectl get all --all-namespaces

# 6. Run smoke tests
./scripts/smoke-test.sh

# 7. Document findings
# What worked?
# What didn't?
# How long did it take?
# What can be improved?

# 8. Update runbook
```

---

## Production Checklist Summary

```
☑️ HIGH AVAILABILITY
   ✓ 3+ control plane nodes
   ✓ Multiple worker nodes
   ✓ Multi-AZ deployment
   ✓ Load balancer for API server

☑️ SECURITY
   ✓ RBAC configured (least privilege)
   ✓ Network Policies active
   ✓ Pod Security Standards enforced
   ✓ Secrets encrypted at rest
   ✓ Regular security audits

☑️ RELIABILITY
   ✓ Resource limits on all pods
   ✓ Health checks configured
   ✓ PodDisruptionBudgets set
   ✓ Auto-scaling enabled (HPA/CA)
   ✓ Multiple replicas (>= 3)

☑️ OBSERVABILITY
   ✓ Metrics collected (Prometheus)
   ✓ Logs centralized (ELK/Loki)
   ✓ Dashboards created (Grafana)
   ✓ Alerts configured
   ✓ On-call rotation established

☑️ DISASTER RECOVERY
   ✓ etcd backups automated
   ✓ Backups tested monthly
   ✓ DR cluster available
   ✓ Runbooks documented
   ✓ DR drills performed

☑️ OPERATIONS
   ✓ GitOps implemented
   ✓ CI/CD pipelines automated
   ✓ Infrastructure as Code
   ✓ Change management process
   ✓ Incident response procedures

☑️ COST OPTIMIZATION
   ✓ Resource requests right-sized
   ✓ Cluster autoscaler active
   ✓ Spot instances for batch
   ✓ Cost monitoring/alerts
   ✓ Regular cost reviews
```

---

## Key Takeaways - Production

**1. No Single Points of Failure**
```
Everything redundant:
✓ 3+ control planes
✓ 3+ worker nodes
✓ Multi-AZ deployment
✓ Replicated services

If one fails, others continue!
```

**2. Observe Everything**
```
Three pillars:
✓ Metrics (Prometheus)
✓ Logs (ELK/Loki)
✓ Traces (Jaeger)

Plus:
✓ Dashboards (Grafana)
✓ Alerts (PagerDuty)

Can't fix what you can't see!
```

**3. Automate Everything**
```
Automation reduces errors:
✓ GitOps (ArgoCD)
✓ CI/CD pipelines
✓ Auto-scaling
✓ Backups
✓ Updates

Humans make mistakes!
Automation is consistent!
```

**4. Test Failures Regularly**
```
Chaos engineering:
✓ Monthly DR drills
✓ Test backups work
✓ Simulate failures
✓ Measure recovery time

Hope is not a strategy!
Test before disaster strikes!
```

**5. Documentation is Critical**
```
Document:
✓ Architecture diagrams
✓ Runbooks for incidents
✓ On-call procedures
✓ Disaster recovery steps
✓ Troubleshooting guides

At 3 AM during outage:
You need clear docs!
```

---

## Congratulations! 🎉

**You've completed the 14-day Kubernetes journey!**

You now understand:
- ✓ Core components (API Server, etcd, Scheduler, Controllers)
- ✓ Networking (CNI, Services, DNS)
- ✓ Storage (PV, PVC, StatefulSets)
- ✓ Security (RBAC, Secrets, Pod Security)
- ✓ Observability (Logs, Metrics, Debugging)
- ✓ Advanced scheduling (Affinity, Taints)
- ✓ Batch processing (Jobs, CronJobs)
- ✓ Multi-cluster patterns
- ✓ Production best practices

**Next Steps:**
1. Get certified (CKA/CKAD)
2. Build real projects
3. Contribute to Kubernetes
4. Share your knowledge
5. Never stop learning!

**Remember:** Breaking things is how you learn!
Set up a test cluster and experiment!

**Good luck in your Kubernetes journey!** 🚀
