# Day 13: Multi-Cluster & Federation

## The Branch Office Analogy

**Single Cluster** = One office building
**Multi-Cluster** = Multiple branch offices across cities

```
┌──────────────────────────────────────────────────────┐
│           COMPANY (Multi-Cluster Setup)               │
│                                                       │
│  HEADQUARTERS (Primary Cluster):                     │
│  ┌────────────────────────────────┐                  │
│  │ New York Office                │                  │
│  │ - Main operations              │                  │
│  │ - Customer database            │                  │
│  │ - 1000 employees               │                  │
│  └────────────────────────────────┘                  │
│                                                       │
│  BRANCH OFFICES (Secondary Clusters):                │
│  ┌────────────────┐  ┌────────────────┐             │
│  │ London Office  │  │ Tokyo Office   │             │
│  │ - EU customers │  │ - APAC customers│            │
│  │ - Local cache  │  │ - Local cache  │             │
│  │ - 200 employees│  │ - 150 employees│             │
│  └────────────────┘  └────────────────┘             │
│                                                       │
│  WHY MULTIPLE OFFICES?                               │
│  ✓ Geographic distribution (lower latency)           │
│  ✓ Data residency (GDPR, compliance)                │
│  ✓ High availability (one fails, others work)       │
│  ✓ Blast radius containment (limit damage)          │
│  ✓ Scale beyond single cluster limits               │
└──────────────────────────────────────────────────────┘
```

---

## Why Multi-Cluster?

**1. Geographic Distribution**
```
Single cluster in US:
- US users: 10ms latency ✓
- EU users: 100ms latency ❌
- APAC users: 200ms latency ❌

Multi-cluster (US, EU, APAC):
- US users → US cluster: 10ms ✓
- EU users → EU cluster: 10ms ✓
- APAC users → APAC cluster: 10ms ✓

Better user experience worldwide!
```

**2. High Availability**
```
Single cluster:
- Cluster fails → Everything down! 💥

Multi-cluster:
- US cluster fails → EU and APAC still work ✓
- 66% capacity remains
- Users redirected to healthy clusters

Disaster recovery built-in!
```

**3. Data Residency**
```
GDPR requires:
- EU user data must stay in EU

Single global cluster:
- All data in one region ❌
- Cannot comply with GDPR

Multi-cluster:
- EU cluster in EU region ✓
- US cluster in US region ✓
- Data never leaves region
- Compliant! ✓
```

**4. Blast Radius Containment**
```
Single cluster:
- Bad deployment breaks everything
- All customers affected

Multi-cluster:
- Deploy to dev cluster first
- Then staging cluster
- Then prod-us cluster
- Then prod-eu cluster
- Problem? Stop rollout!
- Only 25% affected vs 100%
```

---

## Multi-Cluster Patterns

### Pattern 1: Active-Active (All clusters serving traffic)

```
┌─────────────────────────────────────────────────────┐
│              GLOBAL LOAD BALANCER                    │
│         (Routes based on geography)                  │
└─────────────────────────────────────────────────────┘
              ↓           ↓           ↓
    ┌─────────┴─────┐ ┌──┴──────┐ ┌──┴────────┐
    │               │ │         │ │           │
┌───┴────┐    ┌────┴┴──┐  ┌────┴┴──┐  ┌─────┴────┐
│ US-West│    │ US-East│  │   EU   │  │   APAC   │
│Cluster │    │Cluster │  │Cluster │  │ Cluster  │
│        │    │        │  │        │  │          │
│ Web App│    │Web App │  │Web App │  │ Web App  │
│Database│    │Database│  │Database│  │ Database │
│        │    │        │  │        │  │          │
│Active ✓│    │Active ✓│  │Active ✓│  │Active ✓  │
└────────┘    └────────┘  └────────┘  └──────────┘

Benefits:
✓ Low latency worldwide
✓ High availability
✓ Load distributed
✓ All capacity utilized

Challenges:
❌ Data replication (consistency)
❌ More complex (4 clusters to manage)
❌ Higher cost (4x infrastructure)
```

---

### Pattern 2: Active-Passive (Primary + Backup)

```
Normal operation:
┌────────────────────────────────────┐
│    PRIMARY CLUSTER (Active)        │
│    Handling 100% of traffic        │
│    ✓ Serving users                 │
└────────────────────────────────────┘
              ↓ Replicates data
┌────────────────────────────────────┐
│    SECONDARY CLUSTER (Standby)     │
│    Ready but not serving           │
│    ✓ Data synced                   │
│    ✓ Apps deployed                 │
│    ⏸️  Waiting...                   │
└────────────────────────────────────┘

Disaster strikes primary:
┌────────────────────────────────────┐
│    PRIMARY CLUSTER (Down)          │
│    ❌ Failed                        │
└────────────────────────────────────┘
              ↓ Failover
┌────────────────────────────────────┐
│    SECONDARY CLUSTER (Active)      │
│    Now handling 100% traffic       │
│    ✓ Promoted to primary           │
└────────────────────────────────────┘

Benefits:
✓ Disaster recovery
✓ Fast failover (minutes)
✓ Lower cost (secondary can be smaller)

Challenges:
❌ Wasted capacity (secondary idle)
❌ Manual failover (usually)
❌ Data replication lag (RPO)
```

---

## GitOps Management

**Single source of truth: Git repository**

```yaml
# Git repository structure:
clusters/
├── us-west/
│   ├── web-app.yaml
│   ├── database.yaml
│   └── config.yaml
├── us-east/
│   ├── web-app.yaml
│   ├── database.yaml
│   └── config.yaml
└── eu-west/
    ├── web-app.yaml
    ├── database.yaml
    └── config.yaml

# ArgoCD Application for each cluster:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-us-west
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/k8s-configs
    path: clusters/us-west
    targetRevision: HEAD
  destination:
    server: https://us-west-cluster-api:6443
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-eu-west
spec:
  destination:
    server: https://eu-west-cluster-api:6443
  source:
    path: clusters/eu-west
  # ... similar config

Workflow:
1. Developer commits to Git
2. ArgoCD detects change
3. ArgoCD deploys to ALL clusters automatically!
4. Consistent across clusters
5. Audit trail in Git

Benefits:
✓ Declarative (Git as source of truth)
✓ Automated (no manual kubectl)
✓ Consistent (same process everywhere)
✓ Auditable (Git history)
✓ Rollback easy (Git revert)
```

---

## Cross-Cluster Communication

**Problem: Service in US cluster needs to call service in EU cluster**

```
US Cluster:
┌──────────────────────┐
│ Frontend Service     │
│ Needs to call:       │
│ backend-service      │
└──────────────────────┘
       ↓ How to reach?
EU Cluster:
┌──────────────────────┐
│ Backend Service      │
│ backend-service      │
│ (only internal IP)   │
└──────────────────────┘
```

**Solution 1: Expose via LoadBalancer**

```yaml
# EU Cluster - Expose backend
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
  - port: 80

# Gets external IP: 54.123.45.67

# US Cluster - Configure frontend
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
data:
  BACKEND_URL: "http://54.123.45.67"

# Frontend calls external IP
# Traffic goes: US → Internet → EU

Pros:
✓ Simple
✓ Works immediately

Cons:
❌ Traffic over internet (slower, cost)
❌ External IP exposure (security)
❌ No service mesh features
```

**Solution 2: Service Mesh (Istio Multi-Cluster)**

```yaml
# Setup:
1. Install Istio on both clusters
2. Configure mutual TLS between clusters
3. Create ServiceEntry for remote service

# US Cluster - ServiceEntry for EU backend
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: backend-eu
spec:
  hosts:
  - backend.eu.svc.cluster.local
  location: MESH_EXTERNAL
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
  endpoints:
  - address: istio-gateway.eu-cluster.com
    ports:
      http: 80

# Frontend calls: http://backend.eu.svc.cluster.local
# Istio routes across clusters
# Encrypted, authenticated, monitored!

Pros:
✓ Secure (mTLS)
✓ Monitored (telemetry)
✓ Traffic management
✓ Service-to-service auth

Cons:
❌ Complex setup
❌ Requires Istio everywhere
❌ Performance overhead (minimal)
```

---

## Real-World E-commerce Example

**Global E-commerce Platform:**

```
Architecture:
3 regions: US, EU, APAC
Each region: 2 clusters (primary + DR)
Total: 6 clusters

US Region:
├── us-east-1 (Primary)
│   ├── Web App (1000 pods)
│   ├── API Gateway (500 pods)
│   ├── Product Service (300 pods)
│   ├── Checkout Service (200 pods)
│   └── Database (Primary)
│
└── us-west-2 (DR)
    ├── Web App (100 pods - scaled down)
    ├── Database (Replica - read-only)
    └── Ready to scale up on failover

EU Region:
├── eu-west-1 (Primary)
│   └── Same services as US
└── eu-central-1 (DR)

APAC Region:
├── ap-northeast-1 (Primary)
│   └── Same services as US
└── ap-southeast-1 (DR)

Traffic routing (DNS-based):
- US users → us-east-1
- EU users → eu-west-1
- APAC users → ap-northeast-1

If us-east-1 fails:
1. Health check detects failure
2. DNS updated (automatic)
3. US traffic → us-west-2
4. us-west-2 scales up (HPA)
5. 5-minute failover time
6. Users experience brief slowdown

Data replication:
- Orders: Real-time replication (critical)
- Products: Eventual consistency (cache)
- User sessions: Redis cluster (geo-replicated)

Deployment strategy:
1. Deploy to us-west-2 (DR/canary)
2. Monitor for 1 hour
3. Deploy to us-east-1 (primary)
4. Monitor for 1 hour
5. Deploy to eu-west-1
6. Deploy to ap-northeast-1
7. Gradual rollout = 6 hours total

Cost:
- 6 clusters running
- DR clusters scaled down (20% of primary)
- Total: ~2.5x cost of single cluster
- But: Global presence, HA, DR

Worth it? For large company: YES!
Downtime cost >> Infrastructure cost
```

---

## Key Takeaways - Multi-Cluster

**1. Why Multi-Cluster?**
```
✓ High Availability (disaster recovery)
✓ Geographic distribution (low latency)
✓ Compliance (data residency)
✓ Blast radius (containment)
✓ Scale (beyond single cluster limits)
```

**2. Active-Active vs Active-Passive**
```
Active-Active:
- All clusters serve traffic
- Best performance
- Higher cost

Active-Passive:
- One active, others standby
- Lower cost
- DR focused
```

**3. Management Complexity**
```
Single cluster: kubectl
Multi-cluster: Need automation!

Solutions:
✓ GitOps (ArgoCD, Flux)
✓ Cluster API
✓ Terraform
✓ Custom tooling

Don't manage 10 clusters manually!
```

**4. Cross-Cluster Communication**
```
Options:
- LoadBalancer + External IP (simple)
- Service Mesh (Istio multi-cluster)
- VPN/Private network
- API Gateway

Choose based on:
- Security needs
- Performance requirements
- Complexity tolerance
```

**5. Start Small, Grow**
```
Don't start with 10 clusters!

Evolution:
1. Single cluster (dev/staging/prod namespaces)
2. Add DR cluster (active-passive)
3. Add regional clusters (if needed)
4. Add service mesh (if needed)

Complexity grows with business needs!
```

---

**Tomorrow (Day 14): We'll cover Production Readiness & Best Practices - the space launch checklist for Kubernetes!** 🚀
