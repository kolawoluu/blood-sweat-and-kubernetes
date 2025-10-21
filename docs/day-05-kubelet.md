# Day 5: kubelet - Understanding the Node Agent

## The Construction Site Supervisor Analogy

Imagine a construction site:
- **kubelet** = Site supervisor with a clipboard
- **Control plane** = Head office sending blueprints
- **Container runtime** = Workers with tools
- **Pods** = Buildings being constructed

```
┌─────────────────────────────────────────────────────┐
│              CONSTRUCTION SITE (NODE)                │
│                                                      │
│  ┌────────────────────────────────────┐             │
│  │  SUPERVISOR (kubelet)              │             │
│  │  📋 Clipboard with instructions    │             │
│  │                                    │             │
│  │  Every 10 seconds:                 │             │
│  │  1. Check head office for orders   │             │
│  │  2. Tell workers what to build     │             │
│  │  3. Inspect buildings (health)     │             │
│  │  4. Report back to head office     │             │
│  │  5. Fix problems if found          │             │
│  └────────────────────────────────────┘             │
│           ↓ Orders                                   │
│  ┌────────────────────────────────────┐             │
│  │  WORKERS (Container Runtime)       │             │
│  │  🔨 Actually build things          │             │
│  │  - Pull materials (images)         │             │
│  │  - Start construction (containers) │             │
│  │  - Stop work if told               │             │
│  └────────────────────────────────────┘             │
│           ↓ Results                                  │
│  ┌────────────────────────────────────┐             │
│  │  BUILDINGS (Pods/Containers)       │             │
│  │  🏢 Running applications           │             │
│  │  - nginx building                  │             │
│  │  - database building               │             │
│  │  - cache building                  │             │
│  └────────────────────────────────────┘             │
└─────────────────────────────────────────────────────┘
```

---

## What is kubelet?

**kubelet = The agent running on every worker node**

**Primary responsibilities:**
1. **Watch** API Server for pods assigned to this node
2. **Start** containers via container runtime
3. **Monitor** pod and container health
4. **Report** status back to API Server
5. **Restart** failed containers
6. **Clean up** terminated pods

```
kubelet's Daily Routine:

┌─────────────────────────────────────┐
│ 1. Register node with API Server    │
│    "Hi, I'm node-worker-1"          │
│    "I have 8 CPU, 32GB RAM"         │
└─────────────────────────────────────┘
       ↓
┌─────────────────────────────────────┐
│ 2. Watch API Server (every 10s)     │
│    "Any pods assigned to me?"       │
└─────────────────────────────────────┘
       ↓
┌─────────────────────────────────────┐
│ 3. See pod assigned!                │
│    Pod: nginx-abc                   │
│    Image: nginx:1.21                │
│    Port: 80                         │
└─────────────────────────────────────┘
       ↓
┌─────────────────────────────────────┐
│ 4. Tell Container Runtime           │
│    "Pull nginx:1.21 image"          │
│    "Start container"                │
│    "Expose port 80"                 │
└─────────────────────────────────────┘
       ↓
┌─────────────────────────────────────┐
│ 5. Monitor container health         │
│    Every 10s:                       │
│    - Is container running?          │
│    - Health checks passing?         │
│    - Using too many resources?      │
└─────────────────────────────────────┘
       ↓
┌─────────────────────────────────────┐
│ 6. Report to API Server             │
│    "Pod nginx-abc is Running"       │
│    "Using 100m CPU, 200Mi memory"   │
│    "Health: Good ✓"                 │
└─────────────────────────────────────┘
       ↓
     (Loop forever!)
```

---

## How kubelet Starts a Pod

### Complete Pod Lifecycle

```
API Server: "kubelet, run pod X on your node"
       ↓
┌─────────────────────────────────────────────────────┐
│ PHASE 1: PREPARATION                                 │
│                                                      │
│ kubelet checks:                                      │
│ ✓ Do I have enough CPU/memory?                      │
│ ✓ Can I mount required volumes?                     │
│ ✓ Are secrets/configmaps available?                 │
└─────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────┐
│ PHASE 2: CREATE POD SANDBOX                          │
│                                                      │
│ kubelet → Container Runtime:                         │
│ "Create network namespace for pod"                  │
│                                                      │
│ Container Runtime:                                   │
│ - Creates network namespace                          │
│ - Sets up network (CNI plugin)                       │
│ - Assigns IP address to pod                          │
│ - Creates pause container (holds namespace)          │
└─────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────┐
│ PHASE 3: PULL IMAGES                                 │
│                                                      │
│ kubelet → Container Runtime:                         │
│ "Pull image nginx:1.21"                              │
│                                                      │
│ Container Runtime:                                   │
│ - Checks if image exists locally (cache)             │
│ - If not, pulls from registry                        │
│ - Downloads layers                                   │
│ - Extracts filesystem                                │
└─────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────┐
│ PHASE 4: CREATE CONTAINERS                           │
│                                                      │
│ For each container in pod:                           │
│ kubelet → Container Runtime:                         │
│ "Create container with these settings"               │
│                                                      │
│ Container Runtime:                                   │
│ - Creates container from image                       │
│ - Sets resource limits (CPU/memory)                  │
│ - Mounts volumes                                     │
│ - Sets environment variables                         │
│ - Configures security context                        │
└─────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────┐
│ PHASE 5: START CONTAINERS                            │
│                                                      │
│ kubelet → Container Runtime:                         │
│ "Start the container"                                │
│                                                      │
│ Container Runtime:                                   │
│ - Runs container process                             │
│ - Container starts executing                         │
│ - nginx process starts listening on port 80          │
└─────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────┐
│ PHASE 6: HEALTH CHECKS                               │
│                                                      │
│ kubelet runs probes:                                 │
│ - Startup probe (is app starting?)                   │
│ - Readiness probe (is app ready for traffic?)       │
│ - Liveness probe (is app still healthy?)            │
└─────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────┐
│ PHASE 7: UPDATE STATUS                               │
│                                                      │
│ kubelet → API Server:                                │
│ "Pod is Running"                                     │
│ "IP: 10.244.1.5"                                     │
│ "Ready: True"                                        │
└─────────────────────────────────────────────────────┘

Total time: Usually 5-30 seconds
(Depends on image size and pull time)
```

---

## Key Concepts

### 1. Pod Sandbox (The pause container)

**What is it?** Every pod has a "pause" container that does nothing!

```
Why the pause container?

Problem without it:
- Pod has 2 containers: nginx + sidecar
- nginx starts first, gets network namespace
- nginx crashes and restarts
- New nginx gets different network namespace
- Pod IP changes! ❌
- Service endpoints break!

Solution with pause container:
- pause container starts first
- Creates and holds network namespace
- Gets pod IP: 10.244.1.5
- nginx joins this namespace
- sidecar joins this namespace
- nginx crashes and restarts
- Rejoins same namespace
- Pod IP unchanged! ✓

The pause container:
- Does nothing (literally just sleeps)
- Never crashes
- Keeps network namespace alive
- All pod containers share its namespace
```

**Real example:**
```bash
docker ps

CONTAINER ID   IMAGE                  COMMAND
abc123         nginx:1.21             "nginx"
def456         fluent/fluentd         "fluentd"  
xyz789         k8s.gcr.io/pause:3.5   "/pause"  ← This!

The pause container keeps the pod's network alive!
```

---

### 2. PLEG (Pod Lifecycle Event Generator)

**What is PLEG?** kubelet's way of knowing what's happening with containers

```
PLEG is like a motion detector:

Every 1 second:
┌─────────────────────────────────────┐
│ PLEG → Container Runtime:           │
│ "What containers are running?"      │
└─────────────────────────────────────┘
       ↓
┌─────────────────────────────────────┐
│ Container Runtime responds:         │
│ - Container A: Running              │
│ - Container B: Exited               │
│ - Container C: Started (new!)       │
└─────────────────────────────────────┘
       ↓
┌─────────────────────────────────────┐
│ PLEG detects changes:               │
│ - Container B exited! (was running) │
│ - Container C started! (was absent) │
└─────────────────────────────────────┘
       ↓
┌─────────────────────────────────────┐
│ kubelet reacts:                     │
│ - Container B: Restart it           │
│ - Container C: Start health checks  │
└─────────────────────────────────────┘

If PLEG fails:
- kubelet can't detect container state changes
- Can't restart crashed containers
- Node becomes "NotReady"
- Disaster! 💥
```

**Real incident - Shopify Black Friday:**
```
Timeline:

23:50 - Black Friday begins
        - Massive traffic spike
        - Disk I/O saturates

23:55 - Container runtime slow to respond
        - PLEG polls take >3 seconds (should be <100ms)
        - PLEG timeout: "Container runtime not responding!"

00:00 - PLEG marked unhealthy
        - Node marked NotReady
        - No new pods scheduled to node
        - But existing pods still running!

00:05 - 50 nodes affected
        - 50% of cluster capacity lost
        - Remaining nodes overloaded

00:10 - Engineers investigate
        - Disk I/O at 100%
        - Container runtime queued up requests
        - PLEG timing out

00:15 - Fix: Throttle disk I/O
        - Reduce container logs
        - PLEG responds fast again
        - Nodes become Ready

00:30 - Full recovery

Lesson: PLEG health = Node health
Fast disks = Critical!
```

---

### 3. Health Checks (Probes)

**Three types of probes:**

```
┌─────────────────────────────────────────────────────┐
│ 1. STARTUP PROBE                                     │
│    "Is the app starting?"                            │
│                                                      │
│    Use when: App takes long to start (1+ minutes)   │
│    Example: ML model loading                         │
│                                                      │
│    startup probe:                                    │
│      httpGet: /started                               │
│      initialDelaySeconds: 0                          │
│      periodSeconds: 10                               │
│      failureThreshold: 30  (try for 5 minutes)      │
│                                                      │
│    If fails: Container killed and restarted          │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ 2. READINESS PROBE                                   │
│    "Is app ready to serve traffic?"                  │
│                                                      │
│    Use when: App healthy but temporarily busy        │
│    Example: Database connection pool full            │
│                                                      │
│    readiness probe:                                  │
│      httpGet: /ready                                 │
│      periodSeconds: 5                                │
│      failureThreshold: 2                             │
│                                                      │
│    If fails: Removed from Service endpoints          │
│    Container NOT killed, keeps running               │
│    When passes again: Added back to Service          │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ 3. LIVENESS PROBE                                    │
│    "Is app still healthy?"                           │
│                                                      │
│    Use when: App can get stuck/deadlocked            │
│    Example: Infinite loop, memory leak               │
│                                                      │
│    liveness probe:                                   │
│      httpGet: /health                                │
│      periodSeconds: 10                               │
│      failureThreshold: 3                             │
│                                                      │
│    If fails: Container killed and restarted          │
│    Last resort! Don't overuse!                       │
└─────────────────────────────────────────────────────┘
```

---

### Probe Methods

```
1. HTTP GET
   httpGet:
     path: /health
     port: 8080
   
   kubelet makes HTTP request
   Success: Status 200-399
   Failure: Other status codes or timeout

2. TCP Socket
   tcpSocket:
     port: 3306
   
   kubelet tries to open TCP connection
   Success: Connection established
   Failure: Connection refused or timeout

3. Exec Command
   exec:
     command:
     - cat
     - /tmp/healthy
   
   kubelet runs command in container
   Success: Exit code 0
   Failure: Non-zero exit code
```

---

### Real Example - E-commerce Checkout

```
Checkout service pod:

containers:
- name: checkout
  image: checkout:v2
  
  # Takes 30 seconds to load product catalog
  startupProbe:
    httpGet:
      path: /startup
      port: 8080
    periodSeconds: 5
    failureThreshold: 10  # 50 seconds max
  
  # Ready when database connection pool available
  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    periodSeconds: 3
    successThreshold: 1
    failureThreshold: 2   # 6 seconds of failures
  
  # Liveness for deadlock detection
  livenessProbe:
    httpGet:
      path: /health
      port: 8080
    periodSeconds: 10
    failureThreshold: 3   # 30 seconds of failures

Scenario 1: Normal startup
0s  - Container starts
0s  - Startup probe: Fail (app loading)
5s  - Startup probe: Fail (still loading)
10s - Startup probe: Fail
15s - Startup probe: Fail
20s - Startup probe: Fail
25s - Startup probe: Fail
30s - Startup probe: Pass! ✓ (app loaded)
30s - Readiness probe: Fail (connecting to DB)
33s - Readiness probe: Pass! ✓ (DB connected)
     - Pod added to Service endpoints
     - Starts receiving traffic!

Scenario 2: Database down
Pod running normally
Receiving traffic
       ↓
Database goes down
       ↓
0s  - Readiness probe: Fail (can't connect to DB)
3s  - Readiness probe: Fail
6s  - Pod removed from Service ✓
     - No new traffic sent to this pod
     - But container keeps running!
       ↓
Database comes back
       ↓
0s  - Readiness probe: Pass! ✓
     - Pod added back to Service
     - Starts receiving traffic again

Scenario 3: App deadlock
Pod running, receiving traffic
       ↓
App deadlocks (bug in code)
Responds to readiness: OK (quick check)
But can't actually process requests
       ↓
0s  - Liveness probe: Timeout (deadlocked)
10s - Liveness probe: Timeout
20s - Liveness probe: Timeout
30s - 3 failures! Kill container!
     - kubelet kills container
     - kubelet starts new container
     - Fresh start, no deadlock
     - Service resumes!
```

---

## What Happens When kubelet Dies?

### Immediate Impact

```
kubelet crashes on node
       ↓
┌─────────────────────────────────────────┐
│ IMMEDIATE (0-40 seconds):               │
│                                         │
│ ✓ Containers keep running               │
│   (Container runtime independent)       │
│                                         │
│ ✓ Services keep working                 │
│   (kube-proxy still routing)            │
│                                         │
│ ❌ Can't start new pods                 │
│    (No kubelet to talk to runtime)      │
│                                         │
│ ❌ Can't restart crashed containers     │
│    (No health monitoring)               │
│                                         │
│ ❌ Can't report status                  │
│    (API Server doesn't know pod state)  │
└─────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────┐
│ AFTER 40 SECONDS:                       │
│                                         │
│ Node Controller notices:                │
│ "Node hasn't reported in 40 seconds"   │
│                                         │
│ Node marked: Ready → NotReady           │
│                                         │
│ Scheduler: "Don't send new pods there" │
└─────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────┐
│ AFTER 5 MINUTES:                        │
│                                         │
│ Node Controller evicts pods:            │
│ "Node still not responding"             │
│                                         │
│ Pods marked: Terminating                │
│                                         │
│ ReplicaSet Controller:                  │
│ "Create replacement pods on other nodes"│
└─────────────────────────────────────────┘

But! Containers still running on dead node!
This creates "zombie pods" - running but unknown to cluster
```

---

### Real Incident - Target (2018)

```
Timeline:

14:00 - Normal operations
        - 200 worker nodes
        - 5000 pods running

14:15 - kubelet memory leak discovered
        - Affects 50 nodes
        - kubelet memory: 8GB → 16GB → 32GB

14:30 - Nodes start OOMKilling kubelet
        - kubelet dies, can't restart (no memory)
        - 50 nodes affected

14:32 - Pods on these nodes: Status Unknown
        - But containers still running!
        - Zombie pods consuming resources

14:37 - Node Controller marks nodes NotReady
        - No new pods scheduled there

14:42 - Node Controller evicts pods (after 5 min)
        - Expects pods to terminate
        - But no kubelet to terminate them!
        - Pods still running as zombies!

14:45 - ReplicaSet Controllers create replacements
        - 1250 new pods created (50 nodes × 25 avg)
        - Scheduled to healthy nodes
        - Healthy nodes now overloaded!

15:00 - Cascading failures
        - Healthy nodes overwhelmed
        - Some healthy nodes' kubelet OOM
        - More zombie pods created
        - Snowball effect!

15:30 - Emergency measures
        - Manually drain nodes
        - SSH in, forcefully kill containers
        - Restart nodes with fresh kubelet

16:30 - Partial recovery
        - 150 nodes healthy
        - 50 nodes being recovered

18:00 - Full recovery

Total impact: 4 hours
Root cause: kubelet memory leak + no resource limits
Lost revenue: Estimated $500k

Lessons learned:
1. Set resource limits on kubelet
2. Monitor kubelet health
3. Automated zombie pod cleanup
4. Better node lifecycle management
```

---

## Container Runtime Interface (CRI)

### How kubelet Talks to Container Runtime

```
┌─────────────────────────────────────────┐
│         kubelet                         │
│   (Doesn't care which runtime!)         │
└─────────────────────────────────────────┘
               ↓ CRI API
┌─────────────────────────────────────────┐
│   Container Runtime Interface (CRI)     │
│   (Standard API for container ops)      │
└─────────────────────────────────────────┘
               ↓
        ┌──────┴──────┐
        ↓             ↓
┌──────────────┐ ┌──────────────┐
│ containerd   │ │ CRI-O        │
│ (Docker's    │ │ (Red Hat)    │
│  successor)  │ │              │
└──────────────┘ └──────────────┘

CRI API Operations:
- RunPodSandbox() - Create pod network
- CreateContainer() - Create container
- StartContainer() - Start container
- StopContainer() - Stop container
- RemoveContainer() - Delete container
- ListPodSandbox() - List pods (PLEG uses this!)
- PodSandboxStatus() - Get pod status
- ContainerStatus() - Get container status
```

---

### Why CRI Matters

```
Before CRI (old days):
kubelet hardcoded to use Docker
Want to use rkt? Can't!
Want to use another runtime? Can't!

With CRI (now):
kubelet uses standard CRI API
Any runtime that implements CRI works!
- containerd ✓
- CRI-O ✓
- Kata Containers ✓
- gVisor ✓

Flexibility! Choose runtime for your needs:
- containerd: Standard, widely used
- CRI-O: Lightweight, Kubernetes-native
- Kata Containers: VM-based, more secure
- gVisor: Sandboxed, extra security
```

---

## Resource Management

### How kubelet Enforces Resource Limits

```
Pod requests:
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1000m
      memory: 2Gi

kubelet translates to cgroups:

1. CPU Limits (via cpu.cfs_quota_us):
   1000m = 1 core = 100,000 microseconds per 100ms
   
   If container tries to use >1 core:
   - Gets throttled
   - Slowed down
   - But keeps running

2. Memory Limits (via memory.limit_in_bytes):
   2Gi = 2,147,483,648 bytes
   
   If container tries to use >2Gi:
   - Gets OOMKilled immediately!
   - Container terminated
   - kubelet restarts it

Why different?
- CPU: Can share/throttle (not destructive)
- Memory: Can't take back (must kill)
```

---

### Real Example - Memory Leak

```
Application with memory leak:

Start: 100Mi memory
After 1 hour: 500Mi
After 2 hours: 1Gi
After 3 hours: 2Gi (limit!)
       ↓
Container tries to allocate more
       ↓
Linux OOM Killer:
"Container exceeded memory limit!"
       ↓
Kills container process
       ↓
kubelet detects: Container exited
       ↓
kubelet: "Restart container" (restart policy)
       ↓
New container starts: 100Mi
       ↓
Leak starts again...
       ↓
After 3 hours: OOMKilled again
       ↓
This repeats forever!

Pod status shows:
NAME              READY   RESTARTS   AGE
leaky-app-abc     1/1     47         3d

47 restarts = leaking 47 times!

Solution:
- Fix the leak (best)
- Or increase memory limit
- Or set restart policy: Never (let it die)
```

---

## Static Pods

### Special Pods Managed by kubelet

```
What are static pods?
- Defined in files on node
- Not in etcd!
- kubelet manages them directly
- No API Server needed!

Location: /etc/kubernetes/manifests/

Example: Control plane components
- kube-apiserver.yaml
- kube-controller-manager.yaml
- kube-scheduler.yaml
- etcd.yaml

kubelet watches this directory:
- File added? Start pod
- File modified? Restart pod
- File removed? Stop pod

Why useful?
- Bootstrap problem: API Server needs to run
- But API Server is itself a pod
- Static pod solves chicken-and-egg!

Flow:
1. kubelet starts (no API Server yet)
2. kubelet reads /etc/kubernetes/manifests/
3. kubelet starts kube-apiserver.yaml
4. API Server pod starts
5. Now API Server available!
6. Other components can start normally
```

---

### Static Pod Trick

```
Create static pod:
echo "
apiVersion: v1
kind: Pod
metadata:
  name: my-static-pod
spec:
  containers:
  - name: nginx
    image: nginx
" > /etc/kubernetes/manifests/my-static-pod.yaml

kubelet automatically:
- Sees new file
- Starts pod
- Registers with API Server (if available)

kubectl get pods
NAME                        READY   STATUS
my-static-pod-node1         1/1     Running

Try to delete:
kubectl delete pod my-static-pod-node1
pod "my-static-pod-node1" deleted

Check again:
kubectl get pods
NAME                        READY   STATUS
my-static-pod-node1         1/1     Running

It's back! kubelet recreated it (file still exists)

To actually delete:
rm /etc/kubernetes/manifests/my-static-pod.yaml

Now it's gone for real!
```

---

## Key Takeaways

### The 5 Most Important Concepts

**1. kubelet = Node Agent**
```
One kubelet per node
Responsibilities:
- Register node
- Watch for pods assigned to node
- Start containers
- Monitor health
- Report status
- Restart failures

It's the "worker" of Kubernetes!
```

**2. PLEG is Critical**
```
PLEG = Pod Lifecycle Event Generator
Polls container runtime every 1 second
Detects container state changes

If PLEG unhealthy:
- Node marked NotReady
- No new pods scheduled
- Existing pods keep running (usually)

Fast disks = Healthy PLEG!
```

**3. Three Types of Probes**
```
Startup: Is app starting? (kills if fails)
Readiness: Ready for traffic? (removes from Service if fails)
Liveness: Still healthy? (kills if fails)

Best practice:
- Always set readiness
- Use liveness sparingly
- Set generous timeouts
```

**4. Resource Limits are Enforced**
```
CPU limit: Throttled (slowed down)
Memory limit: OOMKilled (terminated)

Set limits on all pods!
Prevents one pod from starving others
```

**5. kubelet Failure = Serious**
```
No kubelet:
- Can't start new pods
- Can't restart crashed containers
- Can't report status
- Creates zombie pods

Monitor kubelet health!
Set resource limits on kubelet itself!
```

---

## Tomorrow (Day 6)

We'll explore:
- Kubernetes networking
- CNI plugins
- kube-proxy and Services
- DNS (CoreDNS)
- How pods communicate
- Network troubleshooting

**You now understand kubelet - the hardworking agent on every node that actually runs your containers! Tomorrow we'll see how these containers talk to each other across the network.** 🔌