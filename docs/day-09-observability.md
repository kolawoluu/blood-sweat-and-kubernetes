# Day 9: Observability - Logs, Metrics, and Debugging

## The Hospital Monitoring Analogy

Imagine a hospital patient monitoring system:
- **Logs** = Patient's medical history (what happened?)
- **Metrics** = Vital signs monitor (heart rate, blood pressure)
- **Traces** = Following symptoms through body systems
- **Debugging** = Doctor diagnosis process

```
┌──────────────────────────────────────────────────────┐
│            PATIENT MONITORING (CLUSTER)               │
│                                                       │
│  MEDICAL HISTORY (Logs):                             │
│  ┌────────────────────────────────────┐              │
│  │ 10:00 - Patient checked in         │              │
│  │ 10:15 - Blood pressure taken       │              │
│  │ 10:30 - ERROR: High fever detected │              │
│  │ 10:45 - Medication administered    │              │
│  │ 11:00 - Symptoms improving         │              │
│  └────────────────────────────────────┘              │
│                                                       │
│  VITAL SIGNS (Metrics):                              │
│  ┌────────────────────────────────────┐              │
│  │ Heart Rate: 75 bpm    [📈 Graph]   │              │
│  │ Blood Pressure: 120/80             │              │
│  │ Temperature: 98.6°F                │              │
│  │ O2 Saturation: 98%                 │              │
│  │ CPU: 45% Memory: 60%               │              │
│  └────────────────────────────────────┘              │
│                                                       │
│  DIAGNOSIS TOOLS (Debugging):                        │
│  ┌────────────────────────────────────┐              │
│  │ 1. Check vital signs (metrics)    │              │
│  │ 2. Review medical history (logs)   │              │
│  │ 3. Run tests (kubectl commands)    │              │
│  │ 4. Physical exam (exec into pod)   │              │
│  └────────────────────────────────────┘              │
└──────────────────────────────────────────────────────┘
```

---

## The Three Pillars of Observability

### 1. Logs - What Happened?

**Container logs = stdout/stderr from application**

```bash
# View logs
kubectl logs pod-name

# Real-time logs (like tail -f)
kubectl logs -f pod-name

# Previous container (if crashed)
kubectl logs pod-name --previous

# Specific container in multi-container pod
kubectl logs pod-name -c container-name

# Last 100 lines
kubectl logs pod-name --tail=100

# Since last hour
kubectl logs pod-name --since=1h
```

**Log example:**
```
2025-10-12 10:15:23 INFO  Starting application
2025-10-12 10:15:24 INFO  Connected to database
2025-10-12 10:15:25 INFO  Server listening on :8080
2025-10-12 10:30:45 ERROR Failed to process request: connection timeout
2025-10-12 10:30:46 WARN  Retrying connection...
2025-10-12 10:30:47 INFO  Connection restored
```

---

### Problem: Logs are Ephemeral!

```
Pod lifecycle:
Day 1: Pod starts, writes logs
Day 2: Pod crashes and restarts
Day 2: Old logs GONE! ❌

Solution: Centralized logging!
```

**Centralized Logging Architecture:**
```
┌─────────────────────────────────────────────────────┐
│  NODE 1                NODE 2                       │
│  ┌──────┐ ┌──────┐    ┌──────┐ ┌──────┐            │
│  │ Pod1 │ │ Pod2 │    │ Pod3 │ │ Pod4 │            │
│  │ logs │ │ logs │    │ logs │ │ logs │            │
│  └──┬───┘ └──┬───┘    └──┬───┘ └──┬───┘            │
│     │        │           │        │                 │
│     └────┬───┴───────────┴────┬───┘                │
│          ↓                     ↓                     │
│  ┌────────────────┐   ┌────────────────┐            │
│  │ Fluentd/       │   │ Fluentd/       │            │
│  │ Fluent-bit     │   │ Fluent-bit     │            │
│  │ (Log shipper)  │   │ (Log shipper)  │            │
│  └───────┬────────┘   └───────┬────────┘            │
│          │                     │                     │
│          └──────────┬──────────┘                     │
│                     ↓                                │
│          ┌──────────────────────┐                   │
│          │ Elasticsearch/Loki   │                   │
│          │ (Log storage)        │                   │
│          └──────────┬───────────┘                   │
│                     ↓                                │
│          ┌──────────────────────┐                   │
│          │ Kibana/Grafana       │                   │
│          │ (Search & visualize) │                   │
│          └──────────────────────┘                   │
└─────────────────────────────────────────────────────┘

Benefits:
✓ Logs survive pod crashes
✓ Search across all pods
✓ Historical logs retained
✓ Correlation between services
✓ Alerting on log patterns
```

---

### 2. Metrics - How Much?

**Metrics = Numerical measurements over time**

**Key metrics:**
```
Pod metrics:
- CPU usage (millicores)
- Memory usage (bytes)
- Network I/O (bytes/sec)
- Disk I/O (bytes/sec)

Node metrics:
- CPU usage (%)
- Memory usage (%)
- Disk usage (%)
- Network throughput

Application metrics:
- Request rate (requests/sec)
- Error rate (errors/sec)
- Response time (milliseconds)
- Active connections
```

**View metrics with kubectl:**
```bash
# Requires metrics-server installed
kubectl top nodes
NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-1   850m         42%    4096Mi          51%
node-2   1200m        60%    6144Mi          77%

kubectl top pods
NAME           CPU(cores)   MEMORY(bytes)
backend-abc    250m         512Mi
frontend-def   100m         256Mi
database-ghi   800m         2048Mi
```

---

### Prometheus - The Metrics Standard

**How Prometheus works:**
```
┌─────────────────────────────────────────────────────┐
│  Prometheus Server                                   │
│  "I'll check everyone's metrics every 15 seconds"   │
│                                                      │
│  Every 15 seconds:                                   │
│  1. Scrape /metrics endpoint from pods              │
│  2. Store time-series data                          │
│  3. Make available for queries                      │
└─────────────────────────────────────────────────────┘
       ↓ Scrapes every 15s
┌─────────────────────────────────────────────────────┐
│  Application Pods                                    │
│  ┌──────────────────────────────────┐               │
│  │ App exposes /metrics endpoint:   │               │
│  │                                  │               │
│  │ http_requests_total 1543         │               │
│  │ http_request_duration_ms 45      │               │
│  │ active_connections 23            │               │
│  │ error_rate 0.01                  │               │
│  └──────────────────────────────────┘               │
└─────────────────────────────────────────────────────┘

Query example:
rate(http_requests_total[5m])
"Requests per second over last 5 minutes"

Result: 150 requests/sec
```

**Real example - API monitoring:**
```
Metrics exposed:
http_requests_total{method="GET",path="/api/users",status="200"} 50000
http_requests_total{method="GET",path="/api/users",status="500"} 150
http_request_duration_seconds_sum{path="/api/users"} 2500
http_request_duration_seconds_count{path="/api/users"} 50150

Queries:
1. Request rate:
   rate(http_requests_total[5m])
   → 100 requests/sec

2. Error rate:
   rate(http_requests_total{status=~"5.."}[5m]) /
   rate(http_requests_total[5m])
   → 0.3% errors

3. Average latency:
   rate(http_request_duration_seconds_sum[5m]) /
   rate(http_request_duration_seconds_count[5m])
   → 50ms average

4. 95th percentile latency:
   histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
   → 120ms (95% of requests faster than this)
```

---

### 3. Traces - Where Did It Go?

**Distributed tracing = Follow request through microservices**

```
User Request: "Get product details"
       ↓
┌─────────────────────────────────────────────────────┐
│ Frontend Service (Trace ID: abc123)                  │
│ Span 1: 15ms                                         │
│ "Received request"                                   │
└─────────────────────────────────────────────────────┘
       ↓ Calls
┌─────────────────────────────────────────────────────┐
│ API Gateway (Trace ID: abc123)                       │
│ Span 2: 45ms                                         │
│ "Routing to product service"                         │
└─────────────────────────────────────────────────────┘
       ↓ Calls
┌─────────────────────────────────────────────────────┐
│ Product Service (Trace ID: abc123)                   │
│ Span 3: 200ms                                        │
│ "Fetching from database"                             │
└─────────────────────────────────────────────────────┘
       ↓ Calls
┌─────────────────────────────────────────────────────┐
│ Database (Trace ID: abc123)                          │
│ Span 4: 180ms ← SLOW! Problem here!                 │
│ "SELECT * FROM products WHERE id=123"                │
└─────────────────────────────────────────────────────┘

Total: 440ms
Where time spent:
- Frontend: 15ms (3%)
- API Gateway: 45ms (10%)
- Product Service: 20ms (5%)
- Database: 180ms (41%) ← Bottleneck!
- Network: 180ms (41%)

Action: Optimize database query or add caching!
```

---

## Debugging Techniques

### Technique 1: Describe Pod

```bash
kubectl describe pod problem-pod

# Shows:
# - Pod status and conditions
# - Events (what happened)
# - Resource requests/limits
# - Volume mounts
# - Container states

Example output:
Name:         backend-abc123
Status:       Running
Conditions:
  Type              Status
  Initialized       True
  Ready             False   ← Problem!
  ContainersReady   False
  PodScheduled      True

Events:
  Type     Reason     Message
  ----     ------     -------
  Normal   Scheduled  Successfully assigned to node-1
  Normal   Pulling    Pulling image "backend:v2"
  Normal   Pulled     Successfully pulled image
  Normal   Created    Created container
  Normal   Started    Started container
  Warning  Unhealthy  Readiness probe failed: HTTP probe failed with statuscode: 500

Diagnosis: App is running but readiness probe failing!
Check app logs for errors.
```

---

### Technique 2: Check Events

```bash
kubectl get events --sort-by='.lastTimestamp'

# Recent events across cluster
LAST SEEN   TYPE      REASON              MESSAGE
2m          Warning   FailedScheduling    0/3 nodes available: insufficient CPU
1m          Normal    Scheduled           Successfully assigned pod to node-2
30s         Warning   BackOff             Back-off restarting failed container
10s         Warning   Unhealthy           Liveness probe failed

# Events for specific pod
kubectl get events --field-selector involvedObject.name=pod-name

# Filter by namespace
kubectl get events -n production

# Watch events live
kubectl get events -w
```

---

### Technique 3: Exec Into Pod

```bash
# Get shell in running container
kubectl exec -it pod-name -- /bin/bash

# Now inside container:
# Check process list
ps aux

# Check network connections
netstat -tulpn

# Test connectivity
curl http://other-service

# Check DNS
nslookup database-service

# Check disk space
df -h

# Check environment variables
env | grep DB_

# Check files
ls -la /app/
cat /app/config.yaml

# Exit
exit
```

---

### Technique 4: Port Forward for Local Testing

```bash
# Forward pod port to localhost
kubectl port-forward pod/backend-abc 8080:80

# Now access from browser:
http://localhost:8080

# Or curl:
curl http://localhost:8080/health

# Forward service instead of pod
kubectl port-forward service/backend 8080:80

# Use case:
# - Test app locally without exposing publicly
# - Debug connectivity issues
# - Access internal services
```

---

### Technique 5: Copy Files From/To Pod

```bash
# Copy from pod
kubectl cp pod-name:/var/log/app.log ./app.log

# Copy to pod
kubectl cp ./config.yaml pod-name:/etc/app/config.yaml

# Specific container in multi-container pod
kubectl cp pod-name:/logs/error.log ./error.log -c app-container

# Use cases:
# - Retrieve log files
# - Get core dumps
# - Update configuration files (testing only!)
# - Extract data for analysis
```

---

## Real Debugging Scenario

**Problem: Application returning 500 errors**

```bash
# Step 1: Check if pods are running
kubectl get pods
NAME            READY   STATUS    RESTARTS   AGE
backend-abc     1/1     Running   5          10m  ← 5 restarts! Suspicious!

# Step 2: Describe pod
kubectl describe pod backend-abc
Events:
  Warning  BackOff  Back-off restarting failed container

# Container is crash-looping!

# Step 3: Check current logs
kubectl logs backend-abc
panic: failed to connect to database: connection refused

# App can't connect to database!

# Step 4: Check previous logs (from before crash)
kubectl logs backend-abc --previous
Connecting to database at database-service:5432
Error: dial tcp 10.96.0.100:5432: connect: connection refused

# Database service not responding

# Step 5: Check if database pods running
kubectl get pods -l app=database
NAME            READY   STATUS    RESTARTS
database-xyz    0/1     Pending   0

# Database pod is pending!

# Step 6: Why pending?
kubectl describe pod database-xyz
Events:
  Warning  FailedScheduling  0/3 nodes available:
    3 Insufficient memory

# Nodes don't have enough memory!

# Step 7: Check node resources
kubectl top nodes
NAME     CPU   MEMORY
node-1   45%   95%   ← Almost full!
node-2   60%   98%   ← Almost full!
node-3   55%   96%   ← Almost full!

# Cluster is out of memory!

# Solution:
# 1. Delete unused pods
kubectl delete pod old-test-pod

# 2. Or add more nodes
# 3. Or reduce resource requests on database

# Step 8: After database starts, backend recovers
kubectl get pods
NAME            READY   STATUS    RESTARTS
backend-abc     1/1     Running   0         ← No more restarts!
database-xyz    1/1     Running   0         ← Running!

# Step 9: Verify app working
kubectl logs backend-abc
Successfully connected to database
Server listening on :8080

# Fixed! 🎉
```

---

## Common Problems & Solutions

### Problem 1: CrashLoopBackOff

**Symptoms:**
```bash
kubectl get pods
NAME       READY   STATUS             RESTARTS
app-pod    0/1     CrashLoopBackOff   8
```

**Causes:**
1. Application code bug (crashes on startup)
2. Missing configuration
3. Failed health checks
4. Insufficient resources

**Debug:**
```bash
# Check logs
kubectl logs app-pod
kubectl logs app-pod --previous

# Common errors:
# "panic: configuration file not found"
# → Missing ConfigMap mount

# "Error: ECONNREFUSED database:5432"
# → Database not ready or wrong hostname

# "Fatal: Out of memory"
# → Insufficient memory limit

# Check resource usage
kubectl top pod app-pod
```

---

### Problem 2: ImagePullBackOff

**Symptoms:**
```bash
kubectl get pods
NAME       READY   STATUS             RESTARTS
app-pod    0/1     ImagePullBackOff   0

kubectl describe pod app-pod
Events:
  Warning  Failed     Failed to pull image "myapp:v2": not found
  Warning  Failed     Error: ImagePullBackOff
```

**Causes:**
1. Image doesn't exist
2. Wrong image name/tag
3. Private registry without credentials
4. Network issues

**Solutions:**
```bash
# Check image name
kubectl get pod app-pod -o jsonpath='{.spec.containers[0].image}'
myapp:v2

# Verify image exists in registry
docker pull myapp:v2
# Error: manifest not found

# Fix: Use correct tag
kubectl set image deployment/app app=myapp:v1

# For private registry:
# Create docker-registry secret
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.com \
  --docker-username=user \
  --docker-password=pass

# Use in pod
spec:
  imagePullSecrets:
  - name: regcred
```

---

### Problem 3: High CPU/Memory Usage

**Symptoms:**
```bash
kubectl top pods
NAME       CPU    MEMORY
app-pod    1950m  4096Mi  ← Near limits!

kubectl describe pod app-pod
Limits:
  cpu: 2000m
  memory: 4096Mi
Requests:
  cpu: 100m
  memory: 256Mi
```

**Debug:**
```bash
# Exec into pod
kubectl exec -it app-pod -- /bin/bash

# Check processes
top
# See which process using CPU

# Check memory
free -h
# See memory usage

# For memory leak:
# Get heap dump (Java example)
jmap -dump:format=b,file=/tmp/heap.dump $(pidof java)

# Copy out for analysis
kubectl cp app-pod:/tmp/heap.dump ./heap.dump

# Analyze with tools like VisualVM
```

---

## Monitoring Best Practices

### 1. The Four Golden Signals (Google SRE)

```
1. Latency
   How long requests take
   Alert: p95 > 1 second

2. Traffic
   How many requests
   Alert: Sudden 50% drop (outage!)

3. Errors
   How many failures
   Alert: Error rate > 1%

4. Saturation
   How full resources are
   Alert: CPU > 80%, Memory > 90%
```

---

### 2. USE Method (for resources)

```
For each resource:

Utilization: How busy? (CPU %)
Saturation: How much waiting? (Queue depth)
Errors: Any errors? (Failed requests)

Example - Network:
Utilization: 60% bandwidth used
Saturation: 100 packets queued
Errors: 5 dropped packets

Action needed if saturation high!
```

---

### 3. RED Method (for services)

```
Rate: Requests per second
Errors: Errors per second  
Duration: Request latency

Example:
Rate: 1000 req/s (normal)
Errors: 5 req/s (0.5% error rate)
Duration: p50=50ms, p95=200ms, p99=500ms

Alert if:
- Rate drops suddenly
- Error rate >1%
- p99 duration >1s
```

---

## Key Takeaways

**1. Three Pillars = Complete Picture**
```
Logs: What happened (events)
Metrics: How much (numbers over time)
Traces: Where it went (request path)

Use all three for full observability!
```

**2. Centralize Everything**
```
Logs → Elasticsearch/Loki
Metrics → Prometheus
Traces → Jaeger/Tempo

Don't rely on kubectl logs!
Ephemeral and limited.
```

**3. Debug Systematically**
```
1. Check pod status (kubectl get pods)
2. Describe pod (kubectl describe)
3. View logs (kubectl logs)
4. Check events (kubectl get events)
5. Exec into pod (kubectl exec)

Follow the data!
```

**4. Monitor Golden Signals**
```
Latency, Traffic, Errors, Saturation

These tell you if service is healthy
Alert on these!
```

**5. Practice in Dev**
```
Break things intentionally
Practice debugging
Learn your tools
When production breaks, you're ready!
```

