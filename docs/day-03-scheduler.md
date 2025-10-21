# Day 3: Scheduler - Understanding the Tetris Master

## The Tetris Game Analogy

Imagine you're playing Tetris, but with a twist:
- **Scheduler** = You, the player deciding where pieces go
- **Nodes** = The columns in your Tetris board
- **Pods** = The falling Tetris pieces (different shapes and sizes)
- **Goal** = Fit all pieces perfectly without gaps or overflow

```
┌─────────────────────────────────────────────────────┐
│              KUBERNETES SCHEDULER                    │
│           "The Tetris Master"                        │
│                                                      │
│  Node 1      Node 2      Node 3      Node 4         │
│ ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐          │
│ │ Pod A│   │ Pod D│   │ Pod G│   │      │          │
│ │ 2CPU │   │ 1CPU │   │ 4CPU │   │      │          │
│ ├──────┤   ├──────┤   ├──────┤   │      │          │
│ │ Pod B│   │ Pod E│   │      │   │      │          │
│ │ 1CPU │   │ 2CPU │   │      │   │      │          │
│ ├──────┤   ├──────┤   │      │   │      │          │
│ │ Pod C│   │ Pod F│   │      │   │      │          │
│ │ 1CPU │   │ 1CPU │   │      │   │      │          │
│ └──────┘   └──────┘   └──────┘   └──────┘          │
│  4/8 CPU   4/8 CPU   4/8 CPU    0/8 CPU            │
│   Full!      Full!     Half      Empty              │
└─────────────────────────────────────────────────────┘

New Pod H arrives (needs 3 CPU)
Where should Scheduler place it?
❌ Node 1: Full
❌ Node 2: Full
✅ Node 3: Has 4 CPU available
✅ Node 4: Has 8 CPU available

Scheduler picks Node 4 (better future flexibility)
```

---

## What Does the Scheduler Do?

### The Job Description

**Scheduler = Matchmaker between Pods and Nodes**

```
Scheduler's Daily Routine:

1. Wake up (start process)
       ↓
2. Watch API Server for new pods
       ↓
3. See: "Pod X has no nodeName assigned"
       ↓
4. Analyze ALL nodes:
   - Which nodes CAN run this pod? (Filtering)
   - Which nodes are BEST for this pod? (Scoring)
       ↓
5. Pick the best node
       ↓
6. Tell API Server: "Put Pod X on Node 2"
       ↓
7. API Server updates pod: nodeName = "node-2"
       ↓
8. kubelet on node-2 sees pod assigned to it
       ↓
9. kubelet starts the pod
       ↓
10. Repeat forever (checking every 1 second)
```

**Time per pod:** Typically 5-50 milliseconds!

---

## How Scheduling Works: The Two-Phase Process

### Phase 1: Filtering (Elimination Round)

**Goal:** Find nodes that CAN run the pod

```
Pod Needs: 2 CPU, 4GB RAM, SSD disk

Check Node 1:
✅ Has 4 CPU available (>2 needed)
✅ Has 8GB RAM available (>4GB needed)
❌ Has HDD disk (needs SSD)
Result: Node 1 ELIMINATED

Check Node 2:
✅ Has 3 CPU available (>2 needed)
✅ Has 6GB RAM available (>4GB needed)
✅ Has SSD disk
Result: Node 2 PASSES ✓

Check Node 3:
✅ Has 8 CPU available (>2 needed)
✅ Has 16GB RAM available (>4GB needed)
✅ Has SSD disk
Result: Node 3 PASSES ✓

Check Node 4:
❌ Has 1 CPU available (<2 needed)
✅ Has 10GB RAM available
✅ Has SSD disk
Result: Node 4 ELIMINATED

Candidates: Node 2, Node 3
```

---

### Phase 2: Scoring (Competition Round)

**Goal:** Find the BEST node from candidates

```
Scoring Criteria (each 0-100 points):

Node 2:
├─ Resource Balance: 75 points
│  (CPU: 3/8 used, RAM: 6/16 used - fairly balanced)
├─ Pod Spreading: 60 points
│  (Already has 5 pods from same deployment)
├─ Node Affinity: 50 points
│  (No specific affinity rules)
└─ Image Locality: 80 points
   (Already has the image cached)
───────────────────────────
Total: 265 points

Node 3:
├─ Resource Balance: 90 points
│  (CPU: 2/8 used, RAM: 4/16 used - well balanced)
├─ Pod Spreading: 90 points
│  (Only has 1 pod from this deployment)
├─ Node Affinity: 50 points
│  (No specific affinity rules)
└─ Image Locality: 20 points
   (Doesn't have image, needs to pull)
───────────────────────────
Total: 250 points

Winner: Node 2 (265 > 250)

But wait! Scheduler can be configured to weight criteria:
- If we value "Pod Spreading" more (to avoid single point of failure)
- Node 3 might win instead!
```

---

## Filtering Predicates (The Rules)

### 1. NodeResourcesFit

**Question:** Does node have enough resources?

```
Pod Requests:
- CPU: 500m (0.5 cores)
- Memory: 1Gi

Node Status:
- Total CPU: 4 cores
- Used CPU: 3 cores
- Available: 1 core ✓

- Total Memory: 8Gi
- Used Memory: 6Gi
- Available: 2Gi ✓

Result: PASS ✓
```

**Real example - Spotify:**
```
Music streaming pod needs:
- 2 CPU cores (for encoding)
- 4GB RAM (for caching songs)

Scheduler checks all 5000 nodes
Finds 500 nodes with enough resources
Proceeds to scoring phase
```

---

### 2. NodeUnschedulable

**Question:** Is node accepting new pods?

```
Node Status:
- spec.unschedulable: false → Accept pods ✓
- spec.unschedulable: true → Reject pods ❌

Use case:
- Admin doing maintenance
- kubectl cordon node-5
- Sets unschedulable: true
- Existing pods keep running
- New pods won't be scheduled there
```

---

### 3. NodeAffinity

**Question:** Does pod want this specific node?

```
Pod says: "I want nodes with SSD disks"

Node 1 labels:
- disktype: ssd ✓
Result: PASS

Node 2 labels:
- disktype: hdd ❌
Result: FAIL

Pod specification:
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
```

---

### 4. PodFitsHostPorts

**Question:** Is the port already in use?

```
Pod wants: hostPort 80

Node 1:
- Port 80 in use by another pod ❌
Result: FAIL

Node 2:
- Port 80 available ✓
Result: PASS

Real example:
- Ingress controller needs port 80/443
- Can only run ONE per node
- Scheduler ensures no conflicts
```

---

### 5. Taints and Tolerations

**Question:** Does pod tolerate node's taints?

```
Node has taint: "gpu=true:NoSchedule"
Meaning: "Only schedule pods that tolerate GPUs"

Pod A (without toleration):
Result: FAIL ❌ (can't tolerate GPU taint)

Pod B (with toleration):
tolerations:
- key: "gpu"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
Result: PASS ✓
```

**Real example - Google Cloud:**
```
GPU nodes are expensive!
Taint them: "nvidia.com/gpu=true:NoSchedule"

Regular pods: Can't schedule ❌
ML training pods: Add toleration, can schedule ✓

Prevents wasting expensive GPU nodes on regular workloads
```

---

## Scoring Priorities (The Ranking System)

### 1. LeastRequestedPriority

**Goal:** Spread pods across nodes, don't overload one node

```
Node 1: 80% CPU used → Score: 20
Node 2: 50% CPU used → Score: 50
Node 3: 20% CPU used → Score: 80

Scheduler prefers Node 3 (least loaded)

Formula:
Score = (Capacity - Requested) / Capacity × 100

Example:
Node has 10 CPU cores
Used: 3 CPU cores
Score = (10 - 3) / 10 × 100 = 70
```

**Real example - Netflix:**
- 10,000 video streaming pods
- Scheduler spreads across all nodes
- Prevents "hot spots" where one node is overloaded
- Even load distribution = better performance

---

### 2. BalancedResourceAllocation

**Goal:** Keep CPU and Memory usage proportional

```
Node 1:
- CPU: 80% used
- Memory: 20% used
- Imbalanced! Score: 40

Node 2:
- CPU: 50% used
- Memory: 50% used
- Balanced! Score: 100

Node 3:
- CPU: 20% used
- Memory: 80% used
- Imbalanced! Score: 40

Scheduler prefers Node 2

Why?
- If Node 1 fills memory, CPU is wasted
- If Node 3 fills CPU, memory is wasted
- Node 2 will exhaust both simultaneously = efficient!
```

---

### 3. ImageLocalityPriority

**Goal:** Prefer nodes that already have the image

```
Pod needs image: nginx:1.21

Node 1:
- Has nginx:1.21 cached → Score: 100
- No pull needed, starts fast!

Node 2:
- Doesn't have nginx:1.21 → Score: 0
- Must pull image (takes 30 seconds)

Scheduler prefers Node 1

Image size matters:
- Small image (10MB): Not important
- Large image (5GB): Very important!
```

**Real example - Machine Learning:**
```
ML model image: 20GB
- Pull time: 10 minutes!
- Pod startup: 10 minutes + model load time

With ImageLocalityPriority:
- Scheduler prefers nodes with image cached
- Pod starts in seconds instead of minutes
- Critical for rapid scaling
```

---

### 4. InterPodAffinity/AntiAffinity

**Goal:** Schedule pods near or far from each other

```
Web pods: "I want to be near Cache pods"
(Lower latency between web and cache)

affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - cache
      topologyKey: kubernetes.io/hostname

Result:
Node 1: Has cache pod → Score: 100
Node 2: No cache pod → Score: 0

Scheduler puts web pod on Node 1
```

**AntiAffinity Example:**
```
Database replicas: "Stay AWAY from each other"
(If node fails, don't lose all replicas)

antiAffinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - database
      topologyKey: kubernetes.io/hostname

Result:
db-1 on Node 1
db-2 on Node 2 (different node!)
db-3 on Node 3 (different node!)

Node failure = only lose 1 replica, not all
```

---

## Real-World Scheduling Scenarios

### Scenario 1: E-commerce Flash Sale (Amazon Prime Day)

**Challenge:**
- Normal traffic: 1000 pods
- Flash sale: Need 10,000 pods in 5 minutes
- Nodes: 500 available

**Scheduler's Work:**
```
Minute 0: 1000 pods running
       ↓
Flash sale starts
       ↓
HPA (autoscaler): "Create 9000 more pods!"
       ↓
Scheduler sees: 9000 new pods need placement
       ↓
Phase 1 - Filtering:
- Check all 500 nodes
- Find 450 nodes with resources
- Eliminate 50 full/tainted nodes
       ↓
Phase 2 - Scoring:
- Score 450 nodes for each pod
- Spread pods evenly
- Consider existing load
       ↓
Result:
- 9000 pods placed in ~45 seconds
- Average 20 pods per node
- Traffic handled successfully!
```

**Without good scheduling:**
- Pods pile up on few nodes
- Those nodes overload and crash
- Cascading failures
- Site goes down! 💥

---

### Scenario 2: Machine Learning Training (OpenAI)

**Challenge:**
- Training job needs 16 GPUs
- Must be on same network segment (low latency)
- Each GPU costs $10/hour

**Scheduler's Work:**
```
Pod requests: 16 GPU, specific node affinity

Phase 1 - Filtering:
Node 1: Has 8 GPUs ❌ (not enough)
Node 2: Has 16 GPUs ✓
Node 3: Has 16 GPUs ✓
Node 4: Has 4 GPUs ❌ (not enough)
Node 5: Has 16 GPUs, but in different rack ❌

Candidates: Node 2, Node 3

Phase 2 - Scoring:
Node 2:
- Already has ML framework cached: +50
- Same availability zone: +30
- Low current load: +20
Total: 100

Node 3:
- No cached images: +0
- Same availability zone: +30
- High current load: +5
Total: 35

Winner: Node 2

Result:
- Training starts immediately (images cached)
- Low latency to other components
- Cost-efficient (no waste of expensive GPUs)
```

---

### Scenario 3: Multi-Region Gaming (Fortnite)

**Challenge:**
- Players from different continents
- Need low-latency game servers
- Each server handles one game lobby

**Scheduler's Work:**
```
Player in Europe joins game
       ↓
Create lobby pod for European game
       ↓
Phase 1 - Filtering with topology:
Node in US-East: ❌ (too far, >100ms latency)
Node in US-West: ❌ (too far, >150ms latency)
Node in EU-West: ✓ (close, <20ms latency)
Node in EU-Central: ✓ (close, <30ms latency)
Node in Asia: ❌ (too far, >200ms latency)

Candidates: EU-West, EU-Central

Phase 2 - Scoring:
EU-West:
- 10ms from player: +90
- High load: +20
Total: 110

EU-Central:
- 25ms from player: +70
- Low load: +80
Total: 150

Winner: EU-Central

Result:
- Player gets 25ms latency (great experience)
- Server not overloaded
- Smooth gameplay!
```

---

## Advanced Scheduling Features

### 1. Pod Priority and Preemption

**Concept:** Some pods are more important than others

```
Priority Classes:
- Critical: 100000 (system pods)
- High: 10000 (production services)
- Medium: 1000 (staging)
- Low: 100 (batch jobs)

Scenario:
Cluster is FULL (no resources)
       ↓
Critical pod arrives (needs resources)
       ↓
Scheduler: "No room? I'll make room!"
       ↓
Finds lowest priority pod (batch job, priority 100)
       ↓
Evicts batch job pod
       ↓
Schedules critical pod in its place
       ↓
Batch job goes back to pending (will reschedule when resources free)
```

**Real example - Banking System:**
```
Priority 100000: Transaction processing (NEVER evict)
Priority 10000: User authentication (critical)
Priority 1000: Report generation (important)
Priority 100: Data analytics (can wait)

During high load:
- Analytics pods get evicted first
- Transactions keep processing
- Users can still log in
- Reports might be delayed
- But core business continues!
```

---

### 2. Topology Spread Constraints

**Goal:** Spread pods across different failure domains

```
You have 9 pods for your app
3 availability zones: us-east-1a, us-east-1b, us-east-1c

Without topology spread:
Zone 1a: 7 pods  ⚠️ (too many!)
Zone 1b: 2 pods
Zone 1c: 0 pods

If Zone 1a fails → Lose 7/9 pods = 78% capacity lost!

With topology spread:
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule

Result:
Zone 1a: 3 pods ✓
Zone 1b: 3 pods ✓
Zone 1c: 3 pods ✓

If Zone 1a fails → Lose only 3/9 pods = 33% capacity lost
Other zones handle the load!
```

**Real example - Netflix:**
```
Movie streaming service: 300 pods
Spread across:
- 3 AWS regions (for DR)
- 9 availability zones (3 per region)
- 500 nodes (spread evenly)

Result:
- Each zone: ~33 pods
- Each region: ~100 pods
- Single zone failure: Lose only 11% capacity
- Service stays online with minor degradation
```

---

### 3. Resource Requests vs Limits

**Critical Concept:** Scheduler uses REQUESTS, not LIMITS

```
Pod Definition:
resources:
  requests:
    cpu: 500m      ← Scheduler looks at this
    memory: 1Gi    ← Scheduler looks at this
  limits:
    cpu: 2000m     ← Kubelet enforces this
    memory: 2Gi    ← Kubelet enforces this

Scheduler's view:
"This pod needs 500m CPU and 1Gi memory"
"Find node with at least that much available"

Reality after pod starts:
Pod might use up to 2000m CPU and 2Gi memory!

This causes overcommitment:
Node has 4 CPU cores
3 pods each request 500m (total: 1.5 CPU)
Scheduler: "Plenty of room! ✓"

But each pod can burst to 2 CPU (total: 6 CPU)
Node only has 4 CPU → Throttling! ⚠️
```

**Real example - Spotify:**
```
Encoding pod:
- Requests: 1 CPU (guaranteed)
- Limits: 4 CPU (can burst during high load)

Normal load:
- Uses 1 CPU
- Node has plenty of resources

Music festival (high load):
- Uses 4 CPU (bursting)
- Multiple pods burst simultaneously
- Node CPU saturates
- Some pods get throttled
- But service stays up (degraded performance)

Without bursting ability:
- Would need 4 CPU request
- 4x more nodes needed
- 4x cost!
```

---

## What Happens When Scheduler Dies?

### Immediate Impact

```
Scheduler crashes
       ↓
New pods created: "I need a node!"
       ↓
Scheduler: (no response)
       ↓
Pods stuck in "Pending" state forever
       ↓
Can't scale up
Can't deploy new services
Can't replace crashed pods (auto-heal breaks)

But:
✓ Existing pods keep running (kubelet manages them)
✓ Services still work (kube-proxy handles networking)
✓ No immediate outage
```

---

### Recovery

```
Scheduler is a static pod (managed by kubelet)
       ↓
Kubelet detects: "Scheduler not responding"
       ↓
Kubelet restarts scheduler container
       ↓
Takes ~30 seconds
       ↓
Scheduler comes back up
       ↓
Sees all pending pods
       ↓
Schedules them in order of priority
       ↓
Cluster back to normal
```

**Real incident - Uber (2019):**
```
Time 0: Traffic spike, need to scale
Time 1: Scheduler crashes due to bug
Time 2: 500 pods stuck pending
Time 3: No new ride-matching services starting
Time 5: Customer impact - can't request rides
Time 8: Engineers notice, restart scheduler manually
Time 10: Scheduler processes 500 pending pods
Time 15: All pods scheduled, service recovered

Lost revenue: ~$50,000
Lesson learned: Monitor scheduler health!
```

---

## Scheduler Performance & Optimization

### How Fast is Scheduling?

```
Small cluster (100 nodes, 1000 pods):
- Time per pod: 5-10ms
- Throughput: 100 pods/sec
- Performance: Excellent ✓

Medium cluster (1000 nodes, 10,000 pods):
- Time per pod: 20-50ms
- Throughput: 50 pods/sec
- Performance: Good ✓

Large cluster (5000 nodes, 100,000 pods):
- Time per pod: 100-500ms
- Throughput: 10-20 pods/sec
- Performance: Needs tuning ⚠️

Very large cluster (10,000 nodes):
- Time per pod: 1-5 seconds
- Throughput: 5-10 pods/sec
- Performance: Custom scheduler needed 💥
```

---

### Scheduling Bottlenecks

**Problem 1: Too Many Nodes**
```
5000 nodes to check
Each pod needs filtering + scoring
5000 nodes × 20 filters = 100,000 checks per pod!

Solution:
- Use percentageOfNodesToScore (check random 50%)
- Results are "good enough"
- 10x faster scheduling
```

**Problem 2: Complex Pod Affinity**
```
Pod affinity: "Schedule near pods with label X"
Must check all existing pods for label X
100,000 existing pods = slow!

Solution:
- Cache pod-to-node mappings
- Index by labels
- Fast lookups instead of full scans
```

**Problem 3: Slow Scoring Functions**
```
Some scoring plugins are expensive
ImageLocality: Must check if image exists on node
BalancedResource: Must calculate resource usage

Solution:
- Cache image presence
- Cache resource calculations
- Refresh periodically (not every pod)
```

---

## Custom Schedulers & Scheduler Extenders

### When Default Scheduler Isn't Enough

**Use Case 1: GPU Scheduling (NVIDIA)**
```
Default scheduler doesn't understand GPU topology:
- Which GPUs are on same bus?
- Which have NVLink connections?
- Which have fastest memory?

Custom GPU scheduler:
- Understands GPU architecture
- Places ML training pods on best GPUs
- Optimizes for GPU-to-GPU communication
- 30% faster training times!
```

**Use Case 2: Cost-Optimized Scheduling (AWS Spot Instances)**
```
Default scheduler doesn't know:
- Which nodes are spot instances (cheap but can disappear)
- Which nodes are on-demand (expensive but stable)

Custom scheduler:
- Batch jobs → Spot instances
- Critical services → On-demand instances
- Preemptible pods → Cheaper nodes
- Save 70% on compute costs!
```

**Use Case 3: Latency-Aware Scheduling (Trading Platforms)**
```
Default scheduler doesn't measure:
- Network latency between nodes
- Proximity to exchange connections
- CPU cache affinity

Custom scheduler for stock trading:
- Places pods near exchange connections
- Ensures <100 microsecond latency
- Optimizes for co-location
- Milliseconds = millions of dollars!
```

---

## Debugging Scheduling Problems

### Problem: Pod Stuck in Pending

**Step 1: Describe the pod**
```bash
kubectl describe pod my-pod

Events:
  Warning  FailedScheduling  0/3 nodes available: 
    1 node(s) didn't match node affinity rules
    2 Insufficient cpu

Meaning:
- 3 nodes total in cluster
- 1 node eliminated (affinity rules)
- 2 nodes eliminated (not enough CPU)
- 0 nodes left that can run this pod!
```

**Step 2: Check resource requests**
```yaml
Pod requests:
  cpu: 8 cores
  memory: 32Gi

Node capacity:
  cpu: 4 cores  ← Problem! Pod needs more than node has
  memory: 16Gi  ← Problem!

Solution:
- Reduce pod requests
- Or add bigger nodes to cluster
```

**Step 3: Check node affinity**
```yaml
Pod requires:
  nodeAffinity:
    matchExpressions:
    - key: disktype
      operator: In
      values:
      - nvme

But no nodes have label disktype=nvme!

Solution:
- Label nodes: kubectl label node node-1 disktype=nvme
- Or remove affinity requirement
```

---

### Problem: Pods Not Spreading Evenly

**Symptom:**
```
Node 1: 50 pods (overloaded!)
Node 2: 5 pods
Node 3: 5 pods
```

**Cause 1: No anti-affinity**
```yaml
Without anti-affinity:
Scheduler picks Node 1 first (maybe has cached images)
All subsequent pods go to Node 1
Creates hot spot!

Solution: Add anti-affinity
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - my-app
      topologyKey: kubernetes.io/hostname

Now spreads across nodes!
```

**Cause 2: Node taints**
```
Node 2 tainted: maintenance=true:NoSchedule
Node 3 tainted: special=true:NoSchedule

Only Node 1 accepts pods
All pods pile up on Node 1!

Solution:
- Remove taints: kubectl taint node node-2 maintenance=true:NoSchedule-
- Or add tolerations to pods
```

---

## Key Takeaways

### The 5 Most Important Concepts

**1. Scheduler is a Matchmaker**
```
New pod → Scheduler finds best node → Pod runs

Two phases:
- Filtering: Which nodes CAN run it?
- Scoring: Which node is BEST?

Time: Usually 5-50ms per pod
```

**2. Filtering Eliminates Impossible Nodes**
```
Common filters:
✓ Enough resources? (CPU/memory)
✓ Node accepting pods? (not cordoned)
✓ Port available? (hostPort conflicts)
✓ Affinity rules match? (node labels)
✓ Tolerates taints? (special workloads)

Result: List of feasible nodes
```

**3. Scoring Ranks Remaining Nodes**
```
Common scores:
- Least requested (spread load)
- Balanced resources (efficient use)
- Image locality (faster startup)
- Pod affinity (place near related pods)
- Pod anti-affinity (place far from replicas)

Result: Best node selected
```

**4. Resource Requests Matter**
```
Scheduler uses requests (not limits) for placement
requests = guaranteed minimum
limits = maximum allowed

Pod with high limit but low request:
- Easy to schedule (low request)
- But might cause node overload (high limit)

Best practice: Set requests = limits (predictable)
```

**5. Scheduling Failures Have Clear Causes**
```
Pod pending = one of:
❌ Not enough resources anywhere
❌ Affinity rules too strict
❌ Node taints block pod
❌ Port conflicts
❌ PVC not available

Always check: kubectl describe pod <name>
Events section tells you exactly why!
```

---

## Real-World Best Practices

### 1. Set Resource Requests Accurately

**Bad:**
```yaml
resources:
  requests:
    cpu: 100m      # Too low! App uses 2 CPU
    memory: 128Mi  # Too low! App uses 4Gi

Result:
- Pod schedules easily
- But causes node overload
- Performance problems
- OOMKilled
```

**Good:**
```yaml
resources:
  requests:
    cpu: 2000m     # Matches actual usage
    memory: 4Gi    # Matches actual usage
  limits:
    cpu: 2000m     # Same as request = predictable
    memory: 4Gi

Result:
- Scheduler makes informed decisions
- No node overload
- Predictable performance
```

---

### 2. Use Pod Anti-Affinity for Reliability

**Bad:**
```yaml
# No anti-affinity
replicas: 3

Result:
All 3 replicas might land on same node
Node failure = complete outage!
```

**Good:**
```yaml
replicas: 3
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: critical-service
        topologyKey: kubernetes.io/hostname

Result:
Replicas spread across different nodes
Node failure = only 1/3 capacity lost
Service stays up!
```

---

### 3. Use Topology Spread for Multi-AZ

**Bad:**
```yaml
# No topology constraints
replicas: 9

Result:
Zone A: 7 pods
Zone B: 2 pods
Zone C: 0 pods

Zone A failure = 78% capacity lost!
```

**Good:**
```yaml
replicas: 9
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: my-app

Result:
Zone A: 3 pods
Zone B: 3 pods
Zone C: 3 pods

Any zone failure = only 33% capacity lost!
```

---

### 4. Use Node Affinity for Special Hardware

**Use Case: GPU Workloads**
```yaml
# ML training pod
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: accelerator
          operator: In
          values:
          - nvidia-tesla-v100
resources:
  limits:
    nvidia.com/gpu: 2

Result:
- Only schedules on nodes with V100 GPUs
- Doesn't waste expensive GPU nodes on non-GPU workloads
- Cost-efficient
```

---

### 5. Monitor Scheduler Health

**Key Metrics:**
```
scheduler_pending_pods: Number waiting to be scheduled
  Normal: 0-10
  Alert: >100

scheduler_schedule_attempts_total: How many attempts
  Should be steady
  Alert: If increasing rapidly

scheduler_scheduling_duration_seconds: Time to schedule
  p50: <50ms
  p99: <500ms
  Alert: If >1s

scheduler_failed_scheduling_attempts: Failed attempts
  Should be rare
  Alert: If increasing
```

---

## Tomorrow (Day 4)

We'll explore:
- Controller Manager in depth
- How controllers maintain desired state
- Self-healing mechanisms
- Deployment strategies
- What happens when controllers fail

**You now understand the Scheduler - the Tetris master that fits pods onto nodes efficiently! Tomorrow we'll learn about the autopilot that keeps everything running smoothly.** 🎮