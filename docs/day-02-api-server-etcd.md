# Day 2: API Server & etcd - Understanding the Brain and Memory

## The Library Analogy

Imagine a huge library system:
- **API Server** = The librarian (front desk person)
- **etcd** = The actual library shelves storing all books
- **Controllers** = Library patrons reading books and taking actions

---

## API Server - The Smart Librarian

### What Does API Server Do?

**Analogy:** The world's strictest but most helpful librarian

```
┌─────────────────────────────────────────────────────┐
│              YOU (via kubectl)                      │
│         "I want to borrow 'Pod' book"               │
└─────────────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────┐
│                  API SERVER                         │
│                                                     │
│  Step 1: AUTHENTICATION                             │
│  "Who are you? Show me your library card!"          │
│  ✓ Valid token/certificate                          │
│                                                     │
│  Step 2: AUTHORIZATION                              │
│  "Are you allowed to borrow this book?"             │
│  ✓ Check RBAC rules                                 │
│                                                     │
│  Step 3: VALIDATION                                 │
│  "Is this a valid book request?"                    │
│  ✓ Check if request makes sense                     │
│                                                     │
│  Step 4: PERSISTENCE                                │
│  "Let me write this down in the records..."         │
│  → Saves to etcd                                    │
└─────────────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────┐
│                     etcd                            │
│           (The library shelves)                     │
│  Stores: "User X borrowed Pod book at 10:00 AM"    │
└─────────────────────────────────────────────────────┘
```

---

### Why API Server is Special

**1. Single Point of Entry**
- Every request goes through API Server
- No backdoors! (Security)
- This includes:
  - Your kubectl commands
  - Controller Manager checking state
  - Scheduler assigning pods
  - kubelet reporting status

**2. Stateless**
- API Server has no memory of its own
- Gets all information from etcd
- Can run multiple API Servers (high availability)
- If one dies, use another!

**3. RESTful API**
```
GET    /api/v1/pods          → List all pods
POST   /api/v1/pods          → Create new pod
DELETE /api/v1/pods/nginx-1  → Delete specific pod
PATCH  /api/v1/pods/nginx-1  → Update pod
```

---

### How API Server Processes Requests

**Example: Creating a Pod**

```
Step-by-Step: kubectl create pod nginx

1. kubectl → API Server
   POST /api/v1/namespaces/default/pods
   {
     "name": "nginx",
     "image": "nginx:latest"
   }

2. API Server: Authentication
   → Checks your kubeconfig token
   → "Valid user? YES ✓"

3. API Server: Authorization  
   → Checks RBAC rules
   → "Can this user create pods? YES ✓"

4. API Server: Admission Controllers
   → LimitRanger: "Does pod have resource limits?"
   → PodSecurity: "Is pod configuration secure?"
   → Many other checks...
   → ALL PASS ✓

5. API Server: Validation
   → Schema validation
   → "Is this valid YAML/JSON? YES ✓"
   → "Do all required fields exist? YES ✓"

6. API Server → etcd
   → Writes pod definition to database
   → etcd returns: "Saved successfully"

7. API Server → You
   → Returns: "Pod created!"

8. Controllers watching etcd see new pod
   → Scheduler: "New pod needs node assignment!"
   → Assigns to node
   
9. kubelet on that node sees pod assigned
   → Starts the pod
```

**Time taken:** Usually 100-500ms for entire flow!

---

## etcd - The Cluster's Memory

### What is etcd?

**Analogy:** Like a super-organized filing cabinet

```
┌──────────────────────────────────────────────────┐
│                    etcd                          │
│         (Distributed Key-Value Store)            │
│                                                  │
│  📁 /registry/pods/                              │
│     └─ default/                                  │
│        ├─ nginx-1    → {pod details}             │
│        ├─ nginx-2    → {pod details}             │
│        └─ nginx-3    → {pod details}             │
│                                                  │
│  📁 /registry/services/                          │
│     └─ default/                                  │
│        └─ web        → {service details}         │
│                                                  │
│  📁 /registry/deployments/                       │
│     └─ default/                                  │
│        └─ app        → {deployment details}      │
│                                                  │
│  📁 /registry/secrets/                           │
│     └─ default/                                  │
│        └─ db-pass    → {base64 encoded}          │
│                                                  │
│  📁 /registry/configmaps/                        │
│  📁 /registry/replicasets/                       │
│  📁 /registry/nodes/                             │
│  ... (everything in your cluster!)               │
└──────────────────────────────────────────────────┘
```

---

### How etcd Works

**Key Concepts:**

**1. Key-Value Store**
```
Key: /registry/pods/default/nginx
Value: {
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name": "nginx",
    "namespace": "default"
  },
  "spec": {
    "containers": [{
      "name": "nginx",
      "image": "nginx:1.21"
    }]
  }
}
```

**2. Distributed & Replicated**
```
┌─────────┐      ┌─────────┐      ┌─────────┐
│ etcd-1  │ ←──→ │ etcd-2  │ ←──→ │ etcd-3  │
│ Leader  │      │ Follower│      │ Follower│
└─────────┘      └─────────┘      └─────────┘
     │                │                │
     └────────────────┴────────────────┘
              Raft Consensus
```

- Run 3 or 5 etcd instances (always odd number)
- One is Leader, others are Followers
- Leader handles writes
- All must agree before write is confirmed (majority vote)
- If Leader dies, new election happens (~seconds)

**3. Consistent & Durable**
- Every write is persisted to disk
- Every write is replicated to majority
- If you write something, it's GUARANTEED to be there
- This is why etcd needs fast SSDs!

---

### Why etcd is Critical

**etcd stores EVERYTHING:**
- Every pod, service, deployment
- All configurations
- All secrets (base64 encoded)
- Cluster state
- Node information

**If etcd dies:**
```
Before:
- API Server knows: "5 pods are running on node-1"
- Controllers know: "I should maintain 5 pods"

After etcd crash:
- API Server: "I don't know what exists!" 😱
- Controllers: "I don't know desired state!" 😱
- But: kubelet keeps running existing pods ✓
```

Cluster becomes READ-ONLY and eventually UNUSABLE!

---

### etcd Performance Characteristics

**What makes etcd fast:**
- In-memory reads (milliseconds)
- WAL (Write-Ahead Log) for durability
- Batched writes
- Compaction to prevent disk growth

**What makes etcd slow:**
- Too many writes (thousands per second)
- Slow disks (HDD instead of SSD)
- Network latency between etcd nodes
- Large objects (secrets >1MB)

**Real numbers:**
- Good: 1000 writes/sec on SSD
- Bad: 100 writes/sec on HDD
- Disaster: 10 writes/sec on network storage

---

## Real-World Examples

### Example 1: Netflix Deployment

**Scenario:** Netflix deploys new version of video streaming service

```
1. Engineer: kubectl apply -f streaming-service.yaml

2. API Server receives request
   - Authenticates: Netflix engineer ✓
   - Authorizes: Can deploy to production? ✓
   - Validates: YAML syntax correct? ✓
   
3. API Server → etcd
   - Writes new deployment spec
   - etcd replicates to 3 nodes
   - Returns: "Write confirmed"
   
4. Deployment Controller watches etcd
   - Sees: "New deployment, needs 100 pods"
   - Creates 100 pod definitions
   
5. Scheduler watches etcd
   - Sees: "100 pods need placement"
   - Assigns pods to nodes based on:
     * Available CPU/memory
     * Geographic distribution
     * Pod anti-affinity rules
   
6. kubelets watch etcd
   - See: "I have 10 pods assigned to me"
   - Start containers
   
7. All states saved in etcd continuously
```

**Why this matters:**
- etcd stores: deployment config, pod assignments, node status
- If engineer asks "Where is my deployment?", API Server reads from etcd
- If node crashes, Controller reads from etcd, creates replacement pods

---

### Example 2: Airbnb Auto-Scaling

**Scenario:** Traffic spike during holiday weekend

```
1. Metrics show high CPU usage

2. HPA (HorizontalPodAutoscaler) decides:
   - "Need to scale from 10 pods to 50 pods"
   
3. HPA → API Server
   - PATCH /apis/apps/v1/deployments/booking-service
   - Change replicas: 10 → 50
   
4. API Server → etcd
   - Updates deployment spec in etcd
   - etcd confirms write
   
5. Deployment Controller watches etcd
   - Sees: "Replicas changed from 10 to 50"
   - Creates 40 new pods
   
6. Scheduler watches etcd
   - Assigns 40 new pods to nodes
   - Spreads across availability zones
   
7. kubelets start 40 new containers

All within ~30 seconds!
```

---

### Example 3: Spotify Disaster Recovery

**Scenario:** Data center power failure

```
Before disaster:
- 3 etcd nodes: DC1, DC2, DC3
- DC1 is Leader
- All data replicated

During disaster:
- DC1 loses power (Leader down!)
- DC2 and DC3 still running
- New election: DC2 becomes Leader (takes ~5 seconds)
- Cluster continues working! ✓

After recovery:
- DC1 comes back online
- Rejoins as Follower
- Catches up on missed data
- Cluster fully healthy again
```

**Why this works:**
- Raft consensus: Need majority (2 out of 3)
- Even with 1 etcd down, cluster survives
- This is why you run 3 or 5 etcd nodes!

---

## API Server & etcd Relationship

### The Dance Between Them

```
┌────────────────────────────────────────────────────────┐
│                   API SERVER                           │
│                                                        │
│  Watch Cache (in-memory):                              │
│  ├─ Pods cache                                         │
│  ├─ Services cache                                     │
│  └─ Deployments cache                                  │
│                                                        │
│  Why cache?                                            │
│  - Faster reads (memory vs disk)                       │
│  - Less load on etcd                                   │
│  - Updated by etcd watch streams                       │
└────────────────────────────────────────────────────────┘
            ↕ Bidirectional Communication
┌────────────────────────────────────────────────────────┐
│                      etcd                              │
│                                                        │
│  Write Path:                                           │
│  API Server → etcd: Save new data                      │
│                                                        │
│  Read Path:                                            │
│  API Server → etcd: Get initial data                   │
│                                                        │
│  Watch Path:                                           │
│  etcd → API Server: "Data changed! Update cache"       │
└────────────────────────────────────────────────────────┘
```

---

### What Happens on API Server Crash

```
API Server Crashes:
       ↓
New requests fail (kubectl commands timeout)
       ↓
Controllers can't read/write state
       ↓
But: Other API Server instances take over! (if HA)
       ↓
Or: kubelet restarts API Server (static pod)
       ↓
Cluster resumes normal operation
```

**Production Setup:**
- Run 3+ API Servers behind load balancer
- If one crashes, others handle traffic
- No single point of failure

---

### What Happens on etcd Crash

```
One etcd Node Crashes:
       ↓
If you have 3 nodes total:
- 2 still running = Majority ✓
- Cluster continues working
       ↓
Write example:
- API Server writes to Leader
- Leader replicates to 1 Follower (2 total)
- That's majority (2 out of 3) ✓
- Write confirmed!

All etcd Nodes Crash:
       ↓
NO MAJORITY = Cluster DEAD 💀
       ↓
API Server can't read or write
       ↓
kubectl commands fail
       ↓
Controllers stop working
       ↓
But: Existing pods keep running!
       ↓
Need to restore from backup
```

---

## etcd Data Organization

### How Kubernetes Objects are Stored

```
etcd Key Structure:
/registry/<resource-type>/<namespace>/<name>

Examples:
/registry/pods/default/nginx-abc123
/registry/services/production/web-api
/registry/deployments/staging/app-v2
/registry/secrets/kube-system/admin-token
/registry/configmaps/default/app-config
/registry/namespaces/production
/registry/nodes/worker-node-1
```

### Actual etcd Data Example

**Pod stored in etcd:**
```json
Key: /registry/pods/default/nginx-abc123

Value: {
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": {
    "name": "nginx-abc123",
    "namespace": "default",
    "uid": "12345-67890-abcdef",
    "resourceVersion": "12345",
    "creationTimestamp": "2024-10-12T10:00:00Z",
    "labels": {
      "app": "nginx"
    }
  },
  "spec": {
    "containers": [{
      "name": "nginx",
      "image": "nginx:1.21",
      "ports": [{"containerPort": 80}]
    }],
    "nodeName": "worker-node-2"
  },
  "status": {
    "phase": "Running",
    "podIP": "10.244.1.5",
    "startTime": "2024-10-12T10:00:05Z"
  }
}
```

**Why this structure matters:**
- Fast lookups: `/registry/pods/default/` lists all pods in namespace
- Hierarchical: Easy to query by namespace
- Versioned: `resourceVersion` tracks changes
- Complete: Everything needed to recreate pod

---

## Backup & Disaster Recovery

### Why Backup etcd?

**Scenario: Complete Cluster Loss**

```
Without Backup:
❌ All pod definitions lost
❌ All service configurations lost  
❌ All secrets lost
❌ All custom resources lost
❌ Must manually recreate everything
❌ Hours/days of work

With Backup:
✅ Restore etcd snapshot
✅ All configurations back instantly
✅ Redeploy applications
✅ Cluster operational in minutes
```

---

### How etcd Backup Works

```
Backup Process:
1. etcdctl snapshot save /backup/etcd-snapshot.db
   - Creates point-in-time snapshot
   - Captures entire etcd database
   - Typically 100MB - 2GB size

2. Copy snapshot to safe location
   - Cloud storage (S3, GCS)
   - Different data center
   - Multiple copies

3. Schedule regular backups
   - Hourly: for very critical clusters
   - Daily: for most production clusters
   - Weekly: for dev/test clusters
```

---

### Real Disaster Recovery Example: GitLab.com (2017)

**What Happened:**
1. Engineer accidentally deletes production database
2. Primary database gone
3. Realize backups are broken!
4. Panic! 😱

**Recovery:**
```
Hour 0: Database deleted
     ↓
Hour 1: Discover backups not working
     ↓
Hour 2: Find one working backup (6 hours old)
     ↓
Hour 3: Restore etcd from snapshot
     ↓
Hour 4: Restore application databases
     ↓
Hour 5: Systems back online
     ↓
Result: Lost 6 hours of data
Lesson: TEST YOUR BACKUPS!
```

**GitLab's Changes After:**
- Automated backup testing
- Multiple backup locations
- More frequent backups
- Better access controls

---

## Performance & Scalability

### etcd Performance Limits

**Small Cluster (100 nodes):**
- etcd size: ~500 MB
- Objects: ~5,000 pods
- Writes/sec: 500
- Reads/sec: 10,000
- Performance: Excellent ✓

**Medium Cluster (500 nodes):**
- etcd size: ~2 GB
- Objects: ~25,000 pods
- Writes/sec: 1,000
- Reads/sec: 50,000
- Performance: Good ✓

**Large Cluster (5,000 nodes):**
- etcd size: ~8 GB
- Objects: ~150,000 pods
- Writes/sec: 2,000
- Reads/sec: 100,000+
- Performance: Requires tuning ⚠️

**At Scale Issues:**
- etcd database size grows
- More compaction needed
- Network bandwidth matters
- Need faster disks (NVMe SSD)

---

### API Server Performance

**What Slows Down API Server:**

1. **Too many requests**
```
Normal: 1,000 requests/sec → Fast ✓
High: 10,000 requests/sec → Slow ⚠️
Extreme: 50,000 requests/sec → Crash 💥
```

2. **Large objects**
```
ConfigMap with 1 KB data → Fast ✓
ConfigMap with 1 MB data → Slow ⚠️
Secret with 5 MB data → Very slow 💥
```

3. **List all pods**
```
List 100 pods → 50ms ✓
List 10,000 pods → 5 seconds ⚠️
List 100,000 pods → 30+ seconds 💥
```

**Solutions:**
- Use pagination for large lists
- Cache frequently accessed data
- Run multiple API Server instances
- Use filters to reduce data returned

---

## Real-World Use Cases

### Use Case 1: Microservices Platform (Uber)

**Challenge:**
- 4,000+ microservices
- 100,000+ pods
- Constant deployments (every 2 minutes)

**How API Server & etcd Handle It:**
```
Deployment Flow:
1. Engineer pushes code
2. CI/CD → API Server: Deploy new version
3. API Server → etcd: Update deployment
4. Deployment Controller → Creates new pods
5. Scheduler → Assigns to nodes
6. Old pods → Gracefully terminated

All data flows through API Server → etcd

etcd Optimizations:
- 5 etcd nodes (high availability)
- NVMe SSDs (fast writes)
- Dedicated etcd servers (no sharing)
- Hourly backups to 3 locations
```

---

### Use Case 2: Multi-Tenant Platform (Shopify)

**Challenge:**
- Thousands of merchant stores
- Each needs isolated environment
- Dynamic scaling
- Strict security

**How It Works:**
```
Per Merchant:
- Namespace in Kubernetes
- ResourceQuotas (limits resources)
- NetworkPolicies (isolates network)
- RBAC (controls access)

All stored in etcd:
/registry/namespaces/merchant-123/
├─ pods/
├─ services/
├─ resourcequotas/
├─ networkpolicies/
└─ rolebindings/

API Server ensures:
- Merchant A can't see Merchant B's data
- Each merchant stays within quota
- Network isolation enforced
```

---

### Use Case 3: Gaming Platform (Epic Games - Fortnite)

**Challenge:**
- Massive traffic spikes
- Game lobbies created instantly
- Each lobby = set of pods
- Need fast scaling

**How It Works:**
```
Player Joins Game:
1. API call → Create lobby
       ↓
2. API Server → etcd: Save lobby config
       ↓
3. Controller → Creates lobby pods:
   - Game server pod
   - Voice chat pod
   - Stats pod
       ↓
4. Scheduler → Places on low-latency nodes
       ↓
5. Player connected in ~2 seconds

Peak Traffic:
- 10,000 lobbies/minute
- 30,000 new pods/minute
- etcd handles: 2,000 writes/sec
- Multiple API Servers load balanced
```

---

## Monitoring & Observability

### What to Monitor

**etcd Health:**
```
✅ Leader elections (should be rare)
✅ Database size (should grow slowly)
✅ Disk latency (should be <10ms)
✅ Network latency between nodes (<50ms)
✅ Failed proposals (should be 0)
✅ Compaction happening regularly
```

**API Server Health:**
```
✅ Request latency (p99 < 1 second)
✅ Error rate (< 0.1%)
✅ Authentication failures
✅ Rate limiting events
✅ Watch cache hit rate (> 90%)
✅ Memory usage
```

---

### Key Metrics to Watch

**etcd:**
```
etcd_server_has_leader: 1 (healthy) or 0 (problem!)
etcd_disk_backend_commit_duration: <25ms (good)
etcd_disk_wal_fsync_duration: <10ms (good)
etcd_network_peer_round_trip_time: <50ms (good)
etcd_mvcc_db_total_size_in_bytes: Growing slowly ✓
```

**API Server:**
```
apiserver_request_duration_seconds: p99 < 1s
apiserver_request_total: Count by code
apiserver_current_inflight_requests: < 400
apiserver_longrunning_gauge: Active watch connections
```

---

## Common Problems & Solutions

### Problem 1: etcd Database Too Large

**Symptoms:**
- Slow writes
- High disk usage
- Backup takes forever
- Restore takes forever

**Causes:**
```
Too many events stored: 
- Kubernetes keeps events for 1 hour
- But accumulate quickly
- 1000 pods × 10 events = 10,000 events

Too many old revisions:
- etcd keeps history
- Never cleaned up
- Database grows infinitely
```

**Solutions:**
```
1. Enable automatic compaction
   etcd --auto-compaction-retention=1h

2. Defragment regularly
   etcdctl defrag

3. Reduce event TTL
   kube-apiserver --event-ttl=30m

4. Clean up unused resources
   kubectl delete pods --field-selector status.phase=Succeeded
```

---

### Problem 2: API Server Overloaded

**Symptoms:**
- kubectl commands slow/timeout
- Applications can't deploy
- "connection refused" errors

**Causes:**
```
Too many watch connections:
- Every controller watches resources
- Each watch = persistent connection
- 1000 controllers × 10 watches = 10,000 connections

Too many list requests:
- "kubectl get pods --all-namespaces" expensive
- Operators frequently listing resources
- No pagination used
```

**Solutions:**
```
1. Use pagination
   kubectl get pods --limit=500

2. Use field selectors
   kubectl get pods --field-selector status.phase=Running

3. Cache on client side
   Don't repeatedly list same data

4. Run multiple API Servers
   3-5 instances behind load balancer
```

---

### Problem 3: Slow etcd Writes

**Symptoms:**
- Deployments take minutes instead of seconds
- "context deadline exceeded" errors
- High API Server latency

**Causes:**
```
Slow disks:
- Using HDD instead of SSD
- Network-attached storage with latency
- Disk I/O saturated

High network latency:
- etcd nodes in different regions
- >50ms latency between nodes
- Raft consensus slowed down
```

**Solutions:**
```
1. Use fast local SSDs
   NVMe SSD: ~100,000 IOPS
   Regular SSD: ~10,000 IOPS
   HDD: ~100 IOPS ❌ DON'T USE

2. Colocate etcd nodes
   Same data center
   Same rack if possible
   <5ms network latency

3. Tune etcd
   --heartbeat-interval=100
   --election-timeout=1000
```

---

## Key Takeaways

### The 5 Most Important Things

**1. API Server is the Gateway**
- Everything goes through it
- No backdoors
- Handles authentication, authorization, validation
- Can scale horizontally (multiple instances)

**2. etcd is the Source of Truth**
- Stores all cluster state
- If etcd dies, cluster dies
- Must be backed up regularly
- Requires fast storage (SSD)

**3. They Work as a Team**
```
Write: You → API Server → etcd
Read: You → API Server (cache) → etcd (if not cached)
Watch: etcd → API Server → Controllers
```

**4. High Availability is Critical**
```
Production Setup:
- 3+ API Servers
- 3 or 5 etcd nodes (odd number)
- Load balancer in front
- Regular backups
```

**5. Performance Matters**
```
Fast etcd = Fast cluster
Slow etcd = Slow everything

Invest in:
✅ Fast SSDs (NVMe preferred)
✅ Low-latency network
✅ Regular maintenance (compaction)
✅ Monitoring
```

---

## Tomorrow (Day 3)

We'll explore:
- Scheduler in detail
- How pods get placed on nodes
- Affinity, taints, tolerations
- Resource requests and limits
- What happens when resources run out

**You now understand the brain (API Server) and memory (etcd) of Kubernetes! These two components work together to keep your cluster running smoothly.** 🧠💾