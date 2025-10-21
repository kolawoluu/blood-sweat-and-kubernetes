# Day 11: StatefulSets & Persistent Workloads

## The Apartment Building Analogy

**Deployment = Hotel**
- Rooms have numbers (room 101, 102, 103)
- Guests come and go
- Room 101 today ≠ Room 101 tomorrow
- No permanent identity

**StatefulSet = Apartment Building**
- Apartments have fixed numbers (Apt 1, 2, 3)
- Residents stay (stable identity)
- Apt 1 always = same resident
- Mailbox labeled with apartment number
- Storage locker assigned to apartment

```
┌──────────────────────────────────────────────────────┐
│         APARTMENT BUILDING (StatefulSet)              │
│                                                       │
│  APT 1 (pod-0):                                      │
│  ┌────────────────────────────────┐                  │
│  │ Name: database-0 (never changes)│                 │
│  │ Address: database-0.db-service │                  │
│  │ Storage: disk-1 (10GB)         │                  │
│  │ Mailbox: #1                    │                  │
│  └────────────────────────────────┘                  │
│                                                       │
│  APT 2 (pod-1):                                      │
│  ┌────────────────────────────────┐                  │
│  │ Name: database-1 (never changes)│                 │
│  │ Address: database-1.db-service │                  │
│  │ Storage: disk-2 (10GB)         │                  │
│  │ Mailbox: #2                    │                  │
│  └────────────────────────────────┘                  │
│                                                       │
│  APT 3 (pod-2):                                      │
│  ┌────────────────────────────────┐                  │
│  │ Name: database-2 (never changes)│                 │
│  │ Address: database-2.db-service │                  │
│  │ Storage: disk-3 (10GB)         │                  │
│  │ Mailbox: #3                    │                  │
│  └────────────────────────────────┘                  │
│                                                       │
│  MAILROOM (Headless Service):                        │
│  ┌────────────────────────────────┐                  │
│  │ Mail to "database-0"            │                  │
│  │ → Delivered to Apt 1            │                  │
│  │                                 │                  │
│  │ Mail to "database-1"            │                  │
│  │ → Delivered to Apt 2            │                  │
│  └────────────────────────────────┘                  │
└──────────────────────────────────────────────────────┘

If Apt 2 tenant moves out (pod deleted):
- New tenant moves into Apt 2 (pod recreated)
- Same apartment number (database-1)
- Same storage locker (disk-2)
- Same mailbox address
- IDENTITY PRESERVED! ✓
```

---

## StatefulSet vs Deployment

```
┌─────────────────────────────────────────────────────┐
│              DEPLOYMENT                              │
│                                                      │
│ Pods:                                                │
│ - Random names: web-abc123, web-def456              │
│ - Created/deleted in parallel                        │
│ - No ordering                                        │
│ - Interchangeable (any pod = any other)             │
│                                                      │
│ Storage:                                             │
│ - Usually shared or none                             │
│ - Or all pods use same PVC                          │
│                                                      │
│ Network:                                             │
│ - Load balanced via Service                          │
│ - No stable network identity                         │
│                                                      │
│ Best for:                                            │
│ ✓ Stateless apps (web servers)                      │
│ ✓ APIs                                               │
│ ✓ Microservices                                      │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│              STATEFULSET                             │
│                                                      │
│ Pods:                                                │
│ - Ordered names: db-0, db-1, db-2                   │
│ - Created sequentially (0→1→2)                       │
│ - Deleted in reverse (2→1→0)                         │
│ - Each pod unique and identifiable                   │
│                                                      │
│ Storage:                                             │
│ - Each pod gets own PVC automatically                │
│ - PVC survives pod deletion                          │
│ - Same PVC reattached on recreation                  │
│                                                      │
│ Network:                                             │
│ - Stable DNS names (db-0.service)                    │
│ - Each pod individually addressable                  │
│ - Headless Service for direct pod access             │
│                                                      │
│ Best for:                                            │
│ ✓ Databases (PostgreSQL, MySQL, MongoDB)            │
│ ✓ Distributed systems (Kafka, Zookeeper, etcd)     │
│ ✓ Anything needing stable identity                   │
└─────────────────────────────────────────────────────┘
```

---

## Creating a StatefulSet

```yaml
# Headless Service (for stable DNS)
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  clusterIP: None  # Headless!
  selector:
    app: postgres
  ports:
  - port: 5432

---
# StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: database  # Links to headless service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:  # Automatic PVC per pod!
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 10Gi

What gets created:
1. Pods: postgres-0, postgres-1, postgres-2
2. PVCs: data-postgres-0, data-postgres-1, data-postgres-2
3. Each PVC gets own PV (10Gi each)

DNS names:
- postgres-0.database.default.svc.cluster.local
- postgres-1.database.default.svc.cluster.local
- postgres-2.database.default.svc.cluster.local

Each pod can be accessed individually!
```

---

## Ordered Deployment

**StatefulSet creates pods in order (0→1→2)**

```
Timeline:

0:00 - Create StatefulSet with replicas=3
       ↓
0:05 - Create postgres-0
       Pod: Pending
       PVC: data-postgres-0 created
       ↓
0:10 - PV bound to PVC
       Pod: ContainerCreating
       ↓
0:15 - Container starts
       Pod: Running
       ↓
       Wait for postgres-0 to be Ready! ✓
       ↓
0:30 - NOW create postgres-1
       (Only after postgres-0 is Ready!)
       ↓
0:40 - postgres-1: Running and Ready ✓
       ↓
       NOW create postgres-2
       (Only after postgres-1 is Ready!)
       ↓
0:50 - postgres-2: Running and Ready ✓
       ↓
All 3 pods running!

Total time: 50 seconds
Sequential, never parallel!

Why?
- Databases need initialization order
- Primary must start before replicas
- Leader election needs ordering
- Data replication setup
```

---

## Stable Network Identity

```yaml
# Client pod connects to specific database replica
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_PRIMARY_HOST
      value: "postgres-0.database"  # Always primary!
    - name: DB_REPLICA_HOST
      value: "postgres-1.database"  # Always replica!

In code:
// Write to primary
primary_conn = connect("postgres-0.database:5432")
primary_conn.execute("INSERT INTO users ...")

// Read from replica (load balancing reads)
replica_conn = connect("postgres-1.database:5432")
users = replica_conn.query("SELECT * FROM users")

Benefits:
✓ Primary always postgres-0 (predictable)
✓ Can target specific replicas
✓ Even if pods restart, same names
✓ Configuration doesn't change
```

---

## Persistent Storage Per Pod

**Each pod gets its own PVC automatically:**

```bash
# Create StatefulSet with 3 replicas
kubectl apply -f postgres-statefulset.yaml

# Check PVCs
kubectl get pvc
NAME                STATUS   VOLUME      CAPACITY
data-postgres-0     Bound    pv-abc      10Gi
data-postgres-1     Bound    pv-def      10Gi
data-postgres-2     Bound    pv-ghi      10Gi

# Check pods
kubectl get pods
NAME          READY   STATUS    RESTARTS
postgres-0    1/1     Running   0
postgres-1    1/1     Running   0
postgres-2    1/1     Running   0

# Delete postgres-1 (simulate failure)
kubectl delete pod postgres-1

# StatefulSet recreates it
kubectl get pods
NAME          READY   STATUS    RESTARTS
postgres-0    1/1     Running   0
postgres-1    1/1     Running   0  ← NEW pod!
postgres-2    1/1     Running   0

# Check PVCs - SAME ONES!
kubectl get pvc
NAME                STATUS   VOLUME      CAPACITY
data-postgres-0     Bound    pv-abc      10Gi
data-postgres-1     Bound    pv-def      10Gi  ← Same PVC!
data-postgres-2     Bound    pv-ghi      10Gi

# Data survived pod deletion! ✓
```

---

## Scaling StatefulSets

**Scale up (creates pods in order):**
```bash
kubectl scale statefulset postgres --replicas=5

# Watch creation
kubectl get pods -w
NAME          READY   STATUS
postgres-0    1/1     Running
postgres-1    1/1     Running
postgres-2    1/1     Running
postgres-3    0/1     Pending      ← Create 3 first
postgres-3    0/1     ContainerCreating
postgres-3    1/1     Running      ← Wait for Ready
postgres-4    0/1     Pending      ← Now create 4
postgres-4    0/1     ContainerCreating
postgres-4    1/1     Running

# New PVCs created
kubectl get pvc
data-postgres-3     Bound    pv-jkl      10Gi  ← New!
data-postgres-4     Bound    pv-mno      10Gi  ← New!

Sequential scale-up!
```

**Scale down (deletes pods in reverse order):**
```bash
kubectl scale statefulset postgres --replicas=2

# Watch deletion
kubectl get pods -w
NAME          READY   STATUS
postgres-0    1/1     Running
postgres-1    1/1     Running
postgres-2    1/1     Running
postgres-3    1/1     Running
postgres-4    1/1     Terminating  ← Delete highest first
postgres-4    0/1     Terminating
# Wait for 4 to fully terminate
postgres-3    1/1     Terminating  ← Now delete 3
postgres-3    0/1     Terminating

# Check PVCs - STILL THERE!
kubectl get pvc
data-postgres-0     Bound    10Gi
data-postgres-1     Bound    10Gi
data-postgres-2     Bound    10Gi
data-postgres-3     Bound    10Gi  ← Still exists!
data-postgres-4     Bound    10Gi  ← Still exists!

# PVCs NOT deleted automatically!
# If you scale back up, data is preserved!

kubectl scale statefulset postgres --replicas=5
# postgres-3 and postgres-4 recreated
# Mount same PVCs
# Data intact! ✓
```

---

## Update Strategies

**1. RollingUpdate (default):**
```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0

# Update image
kubectl set image statefulset/postgres postgres=postgres:15

# Watch update (in reverse order!)
postgres-2: Terminating → Running (new version)
# Wait for postgres-2 healthy
postgres-1: Terminating → Running (new version)
# Wait for postgres-1 healthy
postgres-0: Terminating → Running (new version)

Updates highest ordinal first (2→1→0)
Why? Minimize risk to primary (usually pod-0)
```

**2. OnDelete (manual control):**
```yaml
spec:
  updateStrategy:
    type: OnDelete

# Update image
kubectl set image statefulset/postgres postgres=postgres:15

# Nothing happens automatically!
# Must manually delete pods to trigger update

kubectl delete pod postgres-2
# postgres-2 recreated with new image

kubectl delete pod postgres-1
# postgres-1 recreated with new image

# You control update timing!
# Good for databases (update during maintenance window)
```

---

## Real-World Example: MySQL Primary-Replica

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  primary.cnf: |
    [mysqld]
    log-bin=mysql-bin
    server-id=1
  replica.cnf: |
    [mysqld]
    super-read-only
    server-id=2

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      # Determine if primary or replica
      - name: init-mysql
        image: mysql:8.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Generate server-id from pod ordinal
          [[ $(hostname) =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          
          if [[ $ordinal -eq 0 ]]; then
            # Primary
            cp /mnt/config-map/primary.cnf /mnt/conf.d/
          else
            # Replica
            cp /mnt/config-map/replica.cnf /mnt/conf.d/
            # Set unique server-id
            sed "s/server-id=2/server-id=$((100 + $ordinal))/" \
              /mnt/config-map/replica.cnf > /mnt/conf.d/server.cnf
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql-config
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi

Result:
- mysql-0: Primary (accepts writes)
- mysql-1: Replica (read-only, replicates from primary)
- mysql-2: Replica (read-only, replicates from primary)

Application connects to:
- Writes → mysql-0.mysql (primary)
- Reads → mysql-1.mysql or mysql-2.mysql (replicas)

Load balanced reads, consistent writes!
```

---

## Common StatefulSet Patterns

**1. Quorum-based systems (etcd, Zookeeper):**
```yaml
# Need odd number for majority voting
replicas: 3  # or 5, or 7

# Each member knows others by DNS
- etcd-0.etcd
- etcd-1.etcd
- etcd-2.etcd

# Leader election happens automatically
# Survives individual member failures
```

**2. Kafka cluster:**
```yaml
replicas: 3

# Each broker has unique ID (from ordinal)
# Topics partitioned across brokers
# Data persists in PVCs
# Rebalancing on scale up/down
```

**3. MongoDB replica set:**
```yaml
replicas: 3

# Primary-Secondary-Secondary architecture
# Automatic failover
# Each member has stable DNS for replica set config
```

---

## Key Takeaways - StatefulSets

**1. Stable Identity = Predictable**
```
Pod names never change:
- app-0, app-1, app-2 (always!)

DNS names stable:
- app-0.service.namespace.svc.cluster.local

Can configure based on identity:
- app-0 = primary
- app-1, app-2 = replicas
```

**2. Ordered Operations = Safe**
```
Create: 0 → 1 → 2 (sequential)
Update: 2 → 1 → 0 (reverse)
Delete: 2 → 1 → 0 (reverse)

Why?
- Databases need init order
- Leader election needs ordering
- Safe updates (test on replicas first)
```

**3. Persistent Storage Automatic**
```
volumeClaimTemplates = PVC per pod

Benefits:
✓ Data survives pod deletion
✓ Automatic PVC creation
✓ Same PVC on pod recreation
✓ Scale up = new PVCs
✓ Scale down = PVCs remain (data safe!)
```

**4. Use When You Need**
```
✓ Stable network identity
✓ Ordered deployment/updates
✓ Persistent storage per instance
✓ Stateful applications

Examples:
✓ Databases
✓ Message queues
✓ Distributed caches
✓ Cluster coordinators
```

**5. Not a Silver Bullet**
```
StatefulSets don't provide:
❌ Automatic failover
❌ Replication setup
❌ Backup/restore
❌ Application-level clustering

You still need:
✓ Application handles replication
✓ Init containers for setup
✓ Health checks configured
✓ Backup strategy
```

---

**Tomorrow (Day 12): We'll explore Jobs, CronJobs, and batch processing - how to run tasks that complete rather than run forever!** 🗂️
