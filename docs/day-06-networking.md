# Day 6: Kubernetes Networking - Understanding How Pods Talk

## The Postal System Analogy

Imagine a city postal system:
- **Pods** = Houses with unique addresses
- **Services** = Post office boxes (one address, multiple destinations)
- **CNI Plugin** = Road builders (create routes between houses)
- **kube-proxy** = Mail sorters (decide which house gets the mail)
- **CoreDNS** = Phone book (convert names to addresses)

```
┌──────────────────────────────────────────────────────┐
│            KUBERNETES NETWORKING CITY                 │
│                                                       │
│  ┌─────────────────────────────────────────────┐    │
│  │  PHONE BOOK (CoreDNS)                       │    │
│  │  "web-service" → 10.96.0.100                │    │
│  │  "database" → 10.96.0.200                   │    │
│  └─────────────────────────────────────────────┘    │
│                                                       │
│  ┌─────────────────────────────────────────────┐    │
│  │  POST OFFICE BOX (Service)                  │    │
│  │  Address: 10.96.0.100                       │    │
│  │  Mail goes to one of:                       │    │
│  │  - House 1 (Pod): 10.244.1.5               │    │
│  │  - House 2 (Pod): 10.244.1.6               │    │
│  │  - House 3 (Pod): 10.244.2.7               │    │
│  └─────────────────────────────────────────────┘    │
│                                                       │
│  ┌─────────────────────────────────────────────┐    │
│  │  MAIL SORTER (kube-proxy)                   │    │
│  │  Reads address, decides which house         │    │
│  │  Uses iptables rules as sorting guide       │    │
│  └─────────────────────────────────────────────┘    │
│                                                       │
│  ┌─────────────────────────────────────────────┐    │
│  │  ROAD BUILDER (CNI Plugin)                  │    │
│  │  Creates roads between houses               │    │
│  │  Even houses on different streets!          │    │
│  └─────────────────────────────────────────────┘    │
│                                                       │
│  HOUSES (Pods):                                      │
│  🏠 10.244.1.5 (nginx)                               │
│  🏠 10.244.1.6 (nginx)                               │
│  🏠 10.244.2.7 (nginx)                               │
│  🏠 10.244.2.8 (database)                            │
└──────────────────────────────────────────────────────┘
```

---

## Kubernetes Networking Rules

### The Four Fundamental Rules

**1. Every Pod gets its own IP address**
```
Pod A: 10.244.1.5
Pod B: 10.244.1.6
Pod C: 10.244.2.7

No sharing! Each pod = unique IP
Just like each house has unique address
```

**2. Pods can talk to all other pods without NAT**
```
Pod A (10.244.1.5) can directly reach:
- Pod B (10.244.1.6) ✓
- Pod C (10.244.2.7) ✓
- Pod D (10.244.3.8) ✓

No translation needed!
Direct communication!
```

**3. Nodes can talk to all pods (and vice versa)**
```
Node 1 (192.168.1.10) can reach:
- Any pod on Node 1
- Any pod on Node 2
- Any pod on Node 3

Pods can reach:
- Their own node
- Any other node
```

**4. Pod sees same IP that others see it as**
```
Pod A sees itself as: 10.244.1.5
Pod B sees Pod A as: 10.244.1.5
Node sees Pod A as: 10.244.1.5

No NAT! Same IP everywhere!
This is important for:
- Logging (consistent IPs)
- Security policies
- Service discovery
```

---

## CNI (Container Network Interface)

### What is CNI?

**CNI = Plugin that creates pod network**

```
When pod starts:

kubelet → CNI Plugin: "Create network for this pod"
       ↓
┌─────────────────────────────────────────┐
│ CNI Plugin does:                        │
│                                         │
│ 1. Create network namespace             │
│    (Isolated network for pod)           │
│                                         │
│ 2. Assign IP address                    │
│    From pod CIDR range                  │
│    Example: 10.244.1.5/24               │
│                                         │
│ 3. Create virtual network interfaces    │
│    veth pair: one in pod, one on node   │
│                                         │
│ 4. Set up routing                       │
│    Routes so pod can reach other pods   │
│                                         │
│ 5. Configure DNS                        │
│    /etc/resolv.conf in pod              │
└─────────────────────────────────────────┘
       ↓
Pod has network! Can communicate!
```

---

### Popular CNI Plugins

**1. Calico**
```
How it works:
- Uses BGP (Border Gateway Protocol)
- Creates routes on each node
- No overlay network (direct routing)

Pros:
✓ Fast (no encapsulation overhead)
✓ Network policies (firewall rules)
✓ Scalable (thousands of nodes)

Cons:
❌ Requires BGP knowledge
❌ More complex setup

Best for: Large clusters, need NetworkPolicies
```

**2. Flannel**
```
How it works:
- Uses VXLAN overlay network
- Encapsulates pod traffic in UDP packets
- Simple, easy to understand

Pros:
✓ Simple setup
✓ Works everywhere
✓ Easy to debug

Cons:
❌ Slower (encapsulation overhead)
❌ No NetworkPolicies

Best for: Small clusters, getting started
```

**3. Cilium**
```
How it works:
- Uses eBPF (extended Berkeley Packet Filter)
- Programs kernel directly
- Very fast!

Pros:
✓ Extremely fast
✓ Advanced features (service mesh)
✓ API-aware NetworkPolicies

Cons:
❌ Requires modern kernel (4.9+)
❌ Complex to troubleshoot

Best for: High-performance needs, modern infrastructure
```

---

### How Pod-to-Pod Communication Works

**Scenario: Pod A (10.244.1.5 on Node 1) → Pod B (10.244.2.7 on Node 2)**

```
┌─────────────────────────────────────────────────────┐
│ NODE 1                                               │
│                                                      │
│  Pod A (10.244.1.5)                                 │
│  "I want to talk to 10.244.2.7"                     │
│                                                      │
│  Step 1: Check routing table                        │
│  10.244.2.0/24 → via Node 2 (192.168.1.20)         │
│                                                      │
│  Step 2: Send packet to Node 2                      │
│  Packet leaves Node 1                               │
└─────────────────────────────────────────────────────┘
              ↓ Physical network
┌─────────────────────────────────────────────────────┐
│ NODE 2                                               │
│                                                      │
│  Step 3: Packet arrives at Node 2                   │
│  Destination: 10.244.2.7                            │
│                                                      │
│  Step 4: Node 2 routes to pod network               │
│  10.244.2.7 is on this node                         │
│                                                      │
│  Step 5: Deliver to Pod B                           │
│  Pod B (10.244.2.7) receives packet!                │
└─────────────────────────────────────────────────────┘

Total time: Usually 1-5ms!
```

---

### Real Example - Microservices Communication

```
E-commerce application:

Frontend Pod (10.244.1.5)
       ↓ Calls API
Backend Pod (10.244.2.10)
       ↓ Calls Database
Database Pod (10.244.3.15)

Frontend makes request:
GET http://10.244.2.10:8080/api/products

Packet journey:
1. Frontend (Node 1) → sends to 10.244.2.10
2. CNI routing: "10.244.2.x is on Node 2"
3. Packet sent to Node 2
4. Node 2 delivers to Backend pod
5. Backend processes request
6. Backend queries Database (10.244.3.15)
7. Similar routing to Node 3
8. Database returns data
9. Backend returns to Frontend
10. Frontend renders page

All direct IP communication!
No DNS needed (but usually used)
Fast! Typically 1-10ms total
```

---

## Services - The Load Balancer

### Why Services?

**Problem without Services:**
```
You have 3 nginx pods:
- 10.244.1.5
- 10.244.1.6
- 10.244.2.7

Frontend wants to call nginx:
- Which IP to use? 🤔
- What if pod crashes? Pod IP changes! 💥
- How to load balance? Manual! 😰

This is terrible!
```

**Solution: Services**
```
Create Service "nginx-service"
- Gets stable IP: 10.96.0.100
- DNS name: nginx-service.default.svc.cluster.local

Frontend calls: nginx-service:80
- Service load balances to one of 3 pods
- If pod crashes, service automatically removes it
- If new pod added, service includes it
- Simple! Stable! Reliable! ✨
```

---

### How Services Work

**Service Types:**

**1. ClusterIP (default)**
```
Virtual IP accessible only inside cluster

apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80        ← Service port
    targetPort: 8080 ← Container port

Result:
- Service IP: 10.96.0.100
- Only accessible from inside cluster
- Perfect for internal services
```

**2. NodePort**
```
Exposes service on each node's IP

spec:
  type: NodePort
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 31234  ← Opens this port on ALL nodes

Result:
- Can access via any node: <node-ip>:31234
- Traffic goes to service → pods
- Good for testing, not production
```

**3. LoadBalancer**
```
Creates cloud load balancer

spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080

Result:
- Cloud provider creates ELB/ALB/GCE LB
- Gets public IP: 54.123.45.67
- Internet → Load Balancer → Nodes → Pods
- Production ready!
```

---

### kube-proxy: The Traffic Director

**How kube-proxy implements Services:**

```
kube-proxy runs on every node
Watches: Services and Endpoints

When Service created:
       ↓
kube-proxy creates iptables rules
       ↓
Rules say: "Traffic to 10.96.0.100:80"
          → Send to one of:
             - 10.244.1.5:8080
             - 10.244.1.6:8080
             - 10.244.2.7:8080

How it picks:
- Random selection
- Roughly even distribution
- No session affinity (by default)
```

---

### iptables Rules Example

```
Service: nginx-service (10.96.0.100:80)
Backends: 
- Pod 1: 10.244.1.5:8080
- Pod 2: 10.244.1.6:8080
- Pod 3: 10.244.2.7:8080

iptables rules created by kube-proxy:

# Capture traffic to Service IP
-A KUBE-SERVICES -d 10.96.0.100/32 -p tcp -m tcp --dport 80 \
  -j KUBE-SVC-NGINX

# Load balance to backends
-A KUBE-SVC-NGINX -m statistic --mode random --probability 0.33 \
  -j KUBE-SEP-POD1
-A KUBE-SVC-NGINX -m statistic --mode random --probability 0.50 \
  -j KUBE-SEP-POD2
-A KUBE-SVC-NGINX -j KUBE-SEP-POD3

# DNAT to actual pods
-A KUBE-SEP-POD1 -p tcp -j DNAT --to-destination 10.244.1.5:8080
-A KUBE-SEP-POD2 -p tcp -j DNAT --to-destination 10.244.1.6:8080
-A KUBE-SEP-POD3 -p tcp -j DNAT --to-destination 10.244.2.7:8080

What this means:
- 33% chance → Pod 1
- 50% of remaining (33%) → Pod 2
- Rest (33%) → Pod 3
- Equals: ~33% to each pod!

Magic of iptables! 🎩✨
```

---

### Traffic Flow Example

```
Frontend pod wants to call nginx-service:

curl http://10.96.0.100:80

Step 1: Frontend sends packet
Source: 10.244.1.10:54321
Dest: 10.96.0.100:80

Step 2: iptables intercepts
"This is to 10.96.0.100!"
"Run KUBE-SVC-NGINX rules"

Step 3: Random selection
33% probability → picked Pod 2

Step 4: DNAT (Destination NAT)
Change dest: 10.96.0.100:80 → 10.244.1.6:8080

Step 5: Packet routed to Pod 2
CNI routing delivers it

Step 6: Pod 2 responds
Source: 10.244.1.6:8080
Dest: 10.244.1.10:54321

Step 7: iptables reverse SNAT
Change source: 10.244.1.6:8080 → 10.96.0.100:80

Step 8: Frontend receives response
Thinks it came from Service IP!

Frontend is happy! 😊
Never knew it talked to Pod 2 specifically!
```

---

## CoreDNS - The Phone Book

### What is CoreDNS?

**DNS = Translate names to IPs**

```
Without DNS:
curl http://10.96.0.100:80  ← Hard to remember!

With DNS:
curl http://backend:80  ← Easy to remember!

CoreDNS translates:
backend → 10.96.0.100
```

---

### DNS Names in Kubernetes

**Format:** `<service-name>.<namespace>.svc.cluster.local`

```
Examples:

Service "backend" in namespace "default":
- backend
- backend.default
- backend.default.svc.cluster.local
All resolve to: 10.96.0.100

Service "database" in namespace "production":
- database.production
- database.production.svc.cluster.local
Resolves to: 10.96.0.200

Short names only work within same namespace!
```

---

### How DNS Lookup Works

```
Pod wants to call "backend":

Step 1: Pod checks /etc/resolv.conf
nameserver 10.96.0.10  ← CoreDNS Service IP
search default.svc.cluster.local svc.cluster.local cluster.local

Step 2: Pod sends DNS query to 10.96.0.10
"What is IP of backend?"

Step 3: CoreDNS receives query
"backend... let me check"

Step 4: CoreDNS looks up in its database
backend.default.svc.cluster.local = 10.96.0.100

Step 5: CoreDNS responds
"backend is 10.96.0.100"

Step 6: Pod makes request
curl http://10.96.0.100:80

Step 7: kube-proxy routes to pod
(Service load balancing)

Step 8: Response returns

Total time: DNS lookup ~5ms + request ~10ms = ~15ms
```

---

### Real Example - Microservices

```
E-commerce application:

frontend pod:
- Calls "cart-service"
- Calls "product-service"
- Calls "payment-service"

Code looks like:
http.Get("http://cart-service/api/cart")
http.Get("http://product-service/api/products")
http.Get("http://payment-service/api/checkout")

No hardcoded IPs!
No load balancer configuration!
No service discovery code!

Just service names! ✨

DNS resolves:
cart-service → 10.96.0.101
product-service → 10.96.0.102
payment-service → 10.96.0.103

Services load balance:
cart-service → 3 cart pods
product-service → 5 product pods
payment-service → 2 payment pods

Automatic! Beautiful! Simple!
```

---

## What Breaks When Networking Fails?

### Scenario 1: CNI Plugin Failure

```
CNI plugin crashes or misconfigured

Symptoms:
❌ New pods stuck in ContainerCreating
❌ Pods have no IP address
❌ Can't communicate

Existing pods:
✓ Keep working (already have IPs)

Debug:
kubectl describe pod stuck-pod

Events:
Failed to create pod sandbox: 
  network: failed to setup network
  CNI plugin not responding

Fix:
- Check CNI plugin pods running
- Check CNI config files
- Restart CNI plugin
```

---

### Scenario 2: CoreDNS Failure

```
CoreDNS pods crash

Symptoms:
❌ DNS lookups fail
❌ Can't reach services by name
✓ Can reach by IP still

Example:
curl http://backend  ← Fails! "Name resolution failed"
curl http://10.96.0.100  ← Works! Direct IP

Impact:
- Most apps fail (use service names)
- Hard-coded IPs work (rare)
- Cluster barely functional

Fix:
- Scale CoreDNS deployment
- Check CoreDNS logs
- Verify CoreDNS Service exists
```

**Real incident - GitHub (2020):**
```
Timeline:

14:00 - CoreDNS memory leak
        - Using 8GB per pod (should be ~100MB)
        
14:30 - CoreDNS pods OOMKilled
        - All replicas crash
        - DNS resolution stops

14:31 - Cascading failures
        - Services can't find each other
        - Webhooks fail (DNS lookups)
        - CI/CD pipelines fail
        - User authentication fails

14:45 - Engineers notice
        - No CoreDNS pods running!
        - Manual restart

15:00 - CoreDNS back up
        - DNS working
        - Services reconnecting

15:30 - Full recovery

Root cause: CoreDNS cache growing without bounds
Fix: Update CoreDNS, set memory limits
Impact: 1.5 hour outage
```

---

### Scenario 3: kube-proxy Failure

```
kube-proxy crashes

Symptoms:
❌ Services don't work
❌ Traffic doesn't reach pods
✓ Direct pod IP works

Example:
curl http://backend  ← DNS works, returns IP
curl http://10.96.0.100  ← Times out! No routing

Impact:
- Services broken
- Load balancing broken
- Only direct pod communication works

Debug:
# Check iptables rules
iptables -t nat -L -n | grep KUBE
# Empty! No rules!

# Check kube-proxy
kubectl get pods -n kube-system | grep kube-proxy

Fix:
- Restart kube-proxy pods
- Wait for iptables rules to rebuild
```

**Real incident - Reddit (2020):**
```
Timeline:

09:00 - kube-proxy memory leak
        - Using 4GB per node (should be ~100MB)
        
09:30 - Nodes start OOMing
        - kube-proxy killed on 50 nodes
        - Service routing breaks on those nodes

09:35 - Traffic black holes
        - 30% of requests fail
        - Hitting nodes without kube-proxy
        - Requests time out

09:40 - Load balancer removes bad nodes
        - Health checks fail
        - Only good nodes receive traffic
        - Capacity reduced

10:00 - Emergency fix
        - Restart kube-proxy with memory limit
        - Nodes recover one by one

11:00 - Full capacity restored

Root cause: kube-proxy memory leak with many services
Fix: Update kube-proxy, set resource limits
Impact: 2 hour partial outage
```

---

## Network Policies

### Firewall Rules for Pods

**By default: All pods can talk to all pods**

```
Problem:
- Frontend can access Database directly! 😱
- Any pod can access any pod
- No security!

Solution: NetworkPolicies
```

---

### Example NetworkPolicy

```yaml
# Block all traffic to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432

What this does:
✓ Backend pods can reach Database on port 5432
❌ Frontend pods CANNOT reach Database
❌ Any other pod CANNOT reach Database

Egress (not specified):
❌ Database cannot initiate connections
✓ Can respond to requests
```

---

### Real Example - Security Breach Prevention

```
Without NetworkPolicy:
Attacker compromises frontend pod
       ↓
Frontend can access database directly
       ↓
Attacker dumps all customer data
       ↓
Breach! 💥

With NetworkPolicy:
Attacker compromises frontend pod
       ↓
Frontend tries to access database
       ↓
NetworkPolicy blocks: "Frontend not allowed!"
       ↓
Attack stopped! ✓

Layers of security:
1. NetworkPolicy (network level)
2. RBAC (API level)
3. Pod Security Standards (pod level)

Defense in depth! 🛡️
```

---

## Performance & Troubleshooting

### Network Performance

```
Pod-to-Pod (same node):
Latency: <1ms
Bandwidth: 10-40 Gbps (node NIC speed)

Pod-to-Pod (different node):
Latency: 1-5ms (data center)
Latency: 50-100ms (different region)
Bandwidth: Limited by network

Service (via kube-proxy):
Overhead: ~0.1ms (iptables processing)
Minimal impact!

DNS lookup:
First lookup: ~5ms (query CoreDNS)
Cached: <1ms (local cache)
```

---

### Troubleshooting Tools

**1. Check pod networking:**
```bash
# Get pod IP
kubectl get pod mypod -o wide

# Test connectivity
kubectl exec mypod -- ping 10.244.1.5

# Check DNS
kubectl exec mypod -- nslookup backend

# Check iptables rules
iptables -t nat -L -n | grep <service-ip>
```

**2. Network debugging pod:**
```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash

# Inside pod, you have:
- ping
- traceroute
- curl
- dig
- tcpdump
- and more!
```

**3. Check CNI:**
```bash
# List CNI pods
kubectl get pods -n kube-system | grep calico

# Check CNI logs
kubectl logs -n kube-system <cni-pod>

# Verify CNI config
cat /etc/cni/net.d/*
```

---

## Key Takeaways

### The 5 Most Important Concepts

**1. Every Pod Gets an IP**
```
- Unique IP per pod
- No port conflicts
- Direct pod-to-pod communication
- No NAT needed

This is what makes Kubernetes networking simple!
```

**2. CNI Creates the Network**
```
- Assigns pod IPs
- Creates virtual interfaces
- Sets up routing
- Connects pods across nodes

Without CNI, pods have no network!
```

**3. Services Provide Stable Access**
```
- Stable IP for set of pods
- DNS name for easy access
- Load balancing built-in
- Survives pod restarts

Never use pod IPs directly, always use Services!
```

**4. kube-proxy Routes Traffic**
```
- Uses iptables rules
- Load balances to pods
- Updates automatically
- Runs on every node

Service magic happens here!
```

**5. CoreDNS Enables Service Discovery**
```
- Translates service names to IPs
- Makes microservices communication simple
- No hardcoded IPs needed
- Automatic and reliable

DNS is critical! Monitor it closely!
```

---

## Tomorrow (Day 7)

We'll explore:
- Persistent storage
- PersistentVolumes (PV)
- PersistentVolumeClaims (PVC)
- StorageClasses
- StatefulSets with storage
- Data persistence strategies

**You now understand Kubernetes networking - how pods find and talk to each other! Tomorrow we'll learn how to save data permanently with persistent storage.** 🌐