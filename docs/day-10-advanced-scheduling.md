---

# Day 10: Advanced Scheduling & Affinity

## The Office Seating Analogy

Imagine assigning employees to office desks:
- **Node Affinity** = "I want to sit in the quiet section"
- **Pod Affinity** = "I want to sit near my teammate"
- **Pod Anti-Affinity** = "I DON'T want to sit near that person"
- **Taints** = "This desk is reserved (unless you have permission)"
- **Tolerations** = "I have permission to sit at reserved desks"

```
┌──────────────────────────────────────────────────────┐
│                OFFICE FLOOR (CLUSTER)                 │
│                                                       │
│  QUIET SECTION (Node with label: type=quiet):        │
│  ┌────────┐ ┌────────┐ ┌────────┐                   │
│  │ Writer │ │ Analyst│ │ Empty  │                   │
│  │ wants  │ │ wants  │ │        │                   │
│  │ quiet  │ │ quiet  │ │        │                   │
│  └────────┘ └────────┘ └────────┘                   │
│                                                       │
│  TEAM AREA (Pods with affinity):                     │
│  ┌────────┐ ┌────────┐                              │
│  │Frontend│→│Backend │ ← Want to sit together       │
│  │  pod   │ │  pod   │   (low latency!)             │
│  └────────┘ └────────┘                              │
│                                                       │
│  DISTRIBUTED (Pods with anti-affinity):              │
│  ┌────────┐     ┌────────┐     ┌────────┐          │
│  │Database│  X  │Database│  X  │Database│          │
│  │ Rep 1  │  ↔  │ Rep 2  │  ↔  │ Rep 3  │          │
│  └────────┘     └────────┘     └────────┘          │
│  Stay apart! (high availability)                     │
│                                                       │
│  RESERVED DESK (Tainted node):                       │
│  ┌────────────────────────────────┐                 │
│  │ "GPU Required" (taint)         │                 │
│  │                                │                 │
│  │ Only ML pods with toleration   │                 │
│  │ can sit here                   │                 │
│  └────────────────────────────────┘                 │
└──────────────────────────────────────────────────────┘
```

---

## Node Affinity - Choosing Specific Nodes

**Types:**
1. **Required** - MUST match (pod won't schedule otherwise)
2. **Preferred** - SHOULD match (but will schedule anyway)

```yaml
# Required: Pod MUST run on SSD nodes
apiVersion: v1
kind: Pod
metadata:
  name: database-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: db
    image: postgres

# Preferred: Pod prefers high-memory nodes
apiVersion: v1
kind: Pod
metadata:
  name: cache-pod
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: memory
            operator: In
            values:
            - high
  containers:
  - name: redis
    image: redis

# Operators available:
# - In: label value in list
# - NotIn: label value not in list
# - Exists: label exists (any value)
# - DoesNotExist: label doesn't exist
# - Gt: greater than (numbers)
# - Lt: less than (numbers)
```

**Real example - GPU workloads:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-training
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: accelerator
            operator: In
            values:
            - nvidia-tesla-v100
            - nvidia-tesla-a100
  containers:
  - name: training
    image: tensorflow:latest-gpu
    resources:
      limits:
        nvidia.com/gpu: 2

Result:
✓ Only schedules on nodes with V100 or A100 GPUs
❌ Won't schedule on CPU-only nodes
❌ Won't schedule on nodes with older GPUs

Why? GPU nodes are expensive!
Don't want regular workloads using them.
```

---

## Pod Affinity - Scheduling Near Other Pods

**Use case: Co-locate related services for low latency**

```yaml
# Web pods want to be near cache pods
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - redis-cache
            topologyKey: kubernetes.io/hostname
      containers:
      - name: web
        image: webapp:latest

What this means:
- Web pods MUST run on same node as redis-cache pods
- topologyKey: kubernetes.io/hostname = same physical node
- If no node has redis-cache, web pod stays Pending!

Alternative topologyKeys:
- kubernetes.io/hostname = same node
- topology.kubernetes.io/zone = same availability zone
- topology.kubernetes.io/region = same cloud region
```

**Real example - Microservices:**
```yaml
# API Gateway wants to be in same zone as Backend
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 6
  template:
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: tier
                  operator: In
                  values:
                  - backend
              topologyKey: topology.kubernetes.io/zone
      containers:
      - name: gateway
        image: api-gateway:v2

Result:
- Gateway pods prefer same zone as backend
- Reduced network latency (<1ms vs 5-10ms cross-zone)
- But will schedule elsewhere if needed (preferred, not required)
- Cost savings (no cross-AZ data transfer fees)

AWS example:
- Backend in us-east-1a
- Gateway scheduled to us-east-1a (preferred)
- Latency: 0.5ms
- Cross-AZ data transfer: $0

Without affinity:
- Gateway might be in us-east-1b
- Latency: 8ms
- Cross-AZ data transfer: $0.01/GB
```

---

## Pod Anti-Affinity - Scheduling Away From Other Pods

**Use case: Spread replicas for high availability**

```yaml
# Database replicas should be on different nodes
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - database
            topologyKey: kubernetes.io/hostname
      containers:
      - name: postgres
        image: postgres:14

What this means:
- Database pods CANNOT be on same node
- database-0 on node-1
- database-1 on node-2 (not node-1!)
- database-2 on node-3 (not node-1 or node-2!)

If only 2 nodes available:
- database-0 and database-1 schedule
- database-2 stays Pending (no node without database pod!)

Solution: Use preferred anti-affinity
```

**Preferred anti-affinity (more flexible):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 10
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web
              topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx

Result:
- Scheduler tries to spread pods across nodes
- If not enough nodes, multiple pods per node OK
- Best-effort spreading
- All 10 replicas will schedule (even on 3 nodes)

Distribution with 3 nodes:
node-1: 4 pods
node-2: 3 pods
node-3: 3 pods

Not perfect, but acceptable!
```

---

## Taints and Tolerations

**Taint = Repel pods (unless they tolerate it)**

```bash
# Add taint to node
kubectl taint nodes node-1 gpu=true:NoSchedule

# Three taint effects:
# NoSchedule: Don't schedule new pods (existing stay)
# PreferNoSchedule: Avoid if possible (soft)
# NoExecute: Evict existing pods too! (strong)
```

**Toleration = Permission to ignore taint**

```yaml
# Pod with toleration
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: ml
    image: tensorflow:gpu
    resources:
      limits:
        nvidia.com/gpu: 1

Result:
✓ This pod CAN schedule on node-1 (has toleration)
❌ Regular pods CANNOT (no toleration)

Use case: Dedicate expensive GPU nodes for GPU workloads only
```

**Real example - Node maintenance:**
```bash
# Mark node for maintenance
kubectl taint nodes node-2 maintenance=true:NoSchedule
kubectl taint nodes node-2 maintenance=true:NoExecute

Effect of NoExecute:
1. New pods won't schedule on node-2
2. Existing pods evicted from node-2!
3. Pods recreated on other nodes
4. Node is empty, ready for maintenance

After maintenance:
kubectl taint nodes node-2 maintenance=true:NoSchedule-
kubectl taint nodes node-2 maintenance=true:NoExecute-

Node back in rotation!
```

---

## Topology Spread Constraints

**Goal: Even distribution across failure domains**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 9
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: web
      containers:
      - name: nginx
        image: nginx

What this means:
- maxSkew: 1 → At most 1 pod difference between zones
- topologyKey: zone → Distribute across availability zones
- whenUnsatisfiable: DoNotSchedule → Strict enforcement

Result with 3 zones:
zone-a: 3 pods ✓
zone-b: 3 pods ✓
zone-c: 3 pods ✓

Perfect distribution!

Without constraints:
zone-a: 7 pods ⚠️
zone-b: 2 pods
zone-c: 0 pods

Zone-a failure = 78% capacity lost!

With constraints:
zone-a: 3 pods ✓
Any zone failure = only 33% capacity lost
```

---

## Priority and Preemption

**Some pods are more important than others**

```yaml
# Priority Class
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Mission-critical services"

---
# Critical pod
apiVersion: v1
kind: Pod
metadata:
  name: payment-processor
spec:
  priorityClassName: high-priority
  containers:
  - name: payment
    image: payment:v2

---
# Low priority pod
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
spec:
  priorityClassName: low-priority
  containers:
  - name: batch
    image: batch:v1

Scenario: Cluster is FULL (no resources)
       ↓
High-priority pod arrives
       ↓
Scheduler: "No room, but this is high priority!"
       ↓
Find lowest priority pod (batch-job)
       ↓
Evict batch-job
       ↓
Schedule payment-processor in its place
       ↓
batch-job goes to Pending (will reschedule when space)

Result:
✓ Critical services always run
✓ Batch jobs give way
✓ Automatic priority-based scheduling
```

**Real example - E-commerce:**
```yaml
Priority levels:
1. System pods (10000000)
   - CoreDNS, kube-proxy, CNI

2. Critical services (1000000)
   - Payment processing
   - Checkout
   - User authentication

3. Important services (10000)
   - Product catalog
   - Shopping cart
   - Search

4. Nice-to-have (1000)
   - Recommendations
   - Analytics collection

5. Batch jobs (100)
   - Report generation
   - Data exports
   - ML training

During Black Friday rush:
- Cluster at 100% capacity
- Critical services preempt batch jobs
- Users can checkout (critical)
- Reports delayed (acceptable)
- Revenue protected! 💰
```

---

## Key Takeaways - Advanced Scheduling

**1. Node Affinity = Node Preference**
```
Required: MUST run on specific nodes
Preferred: SHOULD run on specific nodes

Use for:
- GPU workloads (specific hardware)
- SSD vs HDD (performance)
- Region/zone placement (compliance)
```

**2. Pod Affinity = Co-location**
```
Schedule near other pods

Benefits:
- Lower latency (same node/zone)
- Reduced network costs (same AZ)
- Improved performance

Use for:
- Cache near app
- Frontend near backend
- Microservices calling each other
```

**3. Pod Anti-Affinity = Spreading**
```
Schedule away from other pods

Benefits:
- High availability (spread replicas)
- Avoid single point of failure
- Better fault tolerance

Always use for:
- Database replicas
- Stateful applications
- Critical services
```

**4. Taints/Tolerations = Dedicated Nodes**
```
Taint = "Stay away unless you have permission"
Toleration = "I have permission"

Use for:
- GPU nodes (expensive!)
- Special hardware
- Multi-tenant isolation
- Node maintenance
```

**5. Priority = Importance Ranking**
```
High priority preempts low priority

Critical: Always run (payments)
Important: Usually run (catalog)
Batch: Run when space available

Ensures critical services never starve!
```

---
