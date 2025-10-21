# Day 7: Kubernetes Storage - Understanding Persistent Data

## The Storage Locker Analogy

Imagine a gym with lockers:
- **Containers** = Your workout session (temporary)
- **Pod** = Your gym visit (temporary)
- **PersistentVolume (PV)** = Physical locker (permanent storage space)
- **PersistentVolumeClaim (PVC)** = Your locker rental ticket
- **StorageClass** = Type of locker (small/medium/large, different costs)

```
┌──────────────────────────────────────────────────────┐
│                  THE GYM (KUBERNETES)                 │
│                                                       │
│  YOU (Pod):                                          │
│  "I need storage for my stuff"                       │
│                                                       │
│  ↓ Submit rental form                                │
│                                                       │
│  RENTAL DESK (PVC):                                  │
│  "I want a medium locker"                            │
│  Size: 10Gi                                          │
│  Type: SSD                                           │
│                                                       │
│  ↓ Finds available locker                            │
│                                                       │
│  LOCKER ROOM (Storage):                              │
│  ┌────────┐ ┌────────┐ ┌────────┐                   │
│  │Locker 1│ │Locker 2│ │Locker 3│                   │
│  │ 5Gi    │ │ 10Gi   │ │ 20Gi   │                   │
│  │ TAKEN  │ │ FREE ✓ │ │ TAKEN  │                   │
│  └────────┘ └────────┘ └────────┘                   │
│                                                       │
│  ↓ Assign Locker 2 to you                            │
│                                                       │
│  YOU:                                                │
│  ✓ Got locker #2 (10Gi)                              │
│  ✓ Can store gym clothes                             │
│  ✓ Comes back tomorrow, stuff still there!           │
│  ✓ Even if you leave gym, locker stays!              │
│                                                       │
│  IF YOU CANCEL RENTAL:                               │
│  - Return locker key (delete PVC)                    │
│  - Locker contents deleted (reclaim policy)          │
│  - Locker available for others                       │
└──────────────────────────────────────────────────────┘
```

---

## The Storage Problem

### Why We Need Persistent Storage

**Problem: Containers are ephemeral (temporary)**

```
Without persistent storage:

Day 1:
Pod starts: "myapp-abc123"
App writes data: user.txt
File saved in container filesystem
       ↓
Day 2:
Pod crashes (node failure)
       ↓
Kubernetes creates new pod: "myapp-def456"
       ↓
Try to read user.txt
FILE NOT FOUND! ❌
Data lost! 💥

This is BAD for:
- Databases (lose all data!)
- User uploads (files disappear!)
- Application state (sessions lost!)
- Logs (can't debug!)
```

**Solution: Persistent Volumes**

```
With persistent storage:

Day 1:
Pod starts: "myapp-abc123"
Mounts PersistentVolume: /data
App writes data: /data/user.txt
File saved in PersistentVolume ✓
       ↓
Day 2:
Pod crashes
       ↓
Kubernetes creates new pod: "myapp-def456"
Mounts SAME PersistentVolume: /data
       ↓
Try to read /data/user.txt
FILE FOUND! ✓
Data survived! 🎉

Pod is temporary
Storage is permanent!
```

---

## Core Storage Concepts

### 1. Volume (Pod-level storage)

**Lifetime: Same as pod**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: scratch
      mountPath: /tmp/data
  volumes:
  - name: scratch
    emptyDir: {}  ← Volume created when pod starts

What happens:
1. Pod starts → emptyDir created
2. Container writes to /tmp/data
3. Pod deleted → emptyDir deleted
4. Data lost!

Use cases:
- Temporary scratch space
- Shared storage between containers in same pod
- Cache that can be rebuilt
```

---

### 2. PersistentVolume (PV) - Cluster-level storage

**Lifetime: Independent of pods**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce  # Only one node can mount at a time
  persistentVolumeReclaimPolicy: Retain  # Keep data after claim deleted
  storageClassName: fast-ssd
  hostPath:  # Type of storage (this is local disk)
    path: /mnt/data

What this means:
- 10GB of storage available
- Can be claimed by a PVC
- Data survives pod deletion
- Data survives node failure (if network storage)
- Admin manages lifecycle
```

---

### 3. PersistentVolumeClaim (PVC) - Storage request

**User's request for storage**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: fast-ssd

What this means:
"I want 5GB of fast-ssd storage"
"Mount it as ReadWriteOnce"

Kubernetes will:
1. Find PV with ≥5Gi capacity
2. Match storageClassName: fast-ssd
3. Match accessModes
4. Bind PVC to PV
5. PVC now "owns" that PV

PVC Status:
- Pending: Searching for matching PV
- Bound: Found and connected to PV
- Lost: PV deleted but PVC still exists
```

---

### 4. StorageClass - Dynamic provisioning

**Automatic PV creation**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs  # AWS EBS volumes
parameters:
  type: gp3  # SSD type
  fsType: ext4
  iopsPerGB: "50"

What this does:
When PVC requests "storageClassName: fast-ssd":
1. No need for pre-created PV!
2. StorageClass automatically creates AWS EBS volume
3. Creates PV pointing to that EBS
4. Binds PVC to new PV
5. Magic! ✨

No admin intervention needed!
```

---

## Complete Storage Flow

### Static Provisioning (Manual)

```
Step 1: Admin creates PV
┌─────────────────────────────────┐
│ Admin: kubectl apply -f pv.yaml │
│ PV created: 10Gi, fast-ssd      │
│ Status: Available               │
└─────────────────────────────────┘
       ↓
Step 2: User creates PVC
┌─────────────────────────────────┐
│ User: kubectl apply -f pvc.yaml │
│ PVC requests: 5Gi, fast-ssd     │
│ Status: Pending                 │
└─────────────────────────────────┘
       ↓
Step 3: Kubernetes binds them
┌─────────────────────────────────┐
│ Kubernetes: "PV matches PVC!"   │
│ PV: 10Gi ≥ 5Gi ✓                │
│ StorageClass: fast-ssd = fast-ssd ✓│
│ Binding...                      │
│ PV Status: Bound                │
│ PVC Status: Bound               │
└─────────────────────────────────┘
       ↓
Step 4: Pod uses PVC
┌─────────────────────────────────┐
│ Pod mounts PVC                  │
│ Writes data to /data            │
│ Data stored in PV               │
└─────────────────────────────────┘
       ↓
Step 5: Pod deleted
┌─────────────────────────────────┐
│ Pod gone ✗                      │
│ PVC still exists ✓              │
│ PV still exists ✓               │
│ Data safe! ✓                    │
└─────────────────────────────────┘
```

---

### Dynamic Provisioning (Automatic)

```
Step 1: User creates PVC
┌─────────────────────────────────┐
│ User: kubectl apply -f pvc.yaml │
│ PVC requests: 5Gi, fast-ssd     │
│ No PV exists yet!               │
└─────────────────────────────────┘
       ↓
Step 2: StorageClass provisions
┌─────────────────────────────────┐
│ StorageClass "fast-ssd":        │
│ "I see a PVC request!"          │
│                                 │
│ Call AWS API:                   │
│ "Create 5Gi EBS volume"         │
│                                 │
│ AWS: "Created vol-abc123"       │
│                                 │
│ Create PV pointing to vol-abc123│
└─────────────────────────────────┘
       ↓
Step 3: Bind PVC to new PV
┌─────────────────────────────────┐
│ PV: Created (5Gi)               │
│ PVC: Bound to PV                │
│ All automatic! ✨               │
└─────────────────────────────────┘
       ↓
Step 4: Pod uses PVC
┌─────────────────────────────────┐
│ Pod mounts PVC                  │
│ Actually mounts AWS EBS volume  │
│ Writes data                     │
└─────────────────────────────────┘

No admin needed!
Self-service storage!
```

---

## Access Modes

### Three types of access:

**1. ReadWriteOnce (RWO)**
```
One node can mount as read-write
Other nodes: Cannot mount

Example: Database volume
- Database pod on Node 1
- Can read and write
- If pod moves to Node 2, unmounts from Node 1, mounts on Node 2

Most common mode!

Use case:
- Databases (PostgreSQL, MySQL)
- Stateful apps
- Single-instance storage
```

**2. ReadOnlyMany (ROX)**
```
Multiple nodes can mount as read-only
Nobody can write

Example: Static website assets
- nginx pods on multiple nodes
- All read same images/CSS/JS
- Nobody modifies files

Use case:
- Static content
- Shared configuration
- ML models (read-only inference)
```

**3. ReadWriteMany (RWX)**
```
Multiple nodes can mount as read-write
All can read and write simultaneously

Example: Shared file storage
- Multiple pods writing logs
- All to same NFS share

Requires special storage!
- NFS
- CephFS
- Azure Files
Not supported by: EBS, GCE PD

Use case:
- Shared logs
- Collaborative editing
- Multi-writer workloads
```

---

## Reclaim Policies

### What happens when PVC is deleted?

**1. Retain (default for manual PVs)**
```
PVC deleted → PV becomes "Released"
Data still there!
Admin must manually clean up

Example:
User deletes PVC
       ↓
PV status: Released
Data on disk: Still there ✓
       ↓
Admin:
1. Backup data if needed
2. Delete data
3. Delete PV
4. Or: Recreate PVC to reuse PV

Best for: Production databases
Don't want accidental data loss!
```

**2. Delete (default for dynamic PVs)**
```
PVC deleted → PV deleted → Storage deleted
Everything gone!

Example:
User deletes PVC
       ↓
Kubernetes:
1. Deletes PV
2. Calls storage provisioner
       ↓
AWS:
"Delete EBS volume vol-abc123"
       ↓
All gone! ✗

Best for: Temporary storage, development
Want automatic cleanup
```

**3. Recycle (deprecated)**
```
Don't use! Being removed from Kubernetes.
Use Delete or Retain instead.
```

---

## Real-World Examples

### Example 1: WordPress (Blog Platform)

**Problem:**
```
WordPress needs:
- Store uploaded images
- Store theme files
- Store plugins
- Keep data when pod restarts

Without storage:
User uploads photo → Pod crashes → Photo lost! 😱
```

**Solution:**
```yaml
# PVC for WordPress files
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: gp3-ssd

---
# WordPress Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  replicas: 1  # Only 1 because RWO
  template:
    spec:
      containers:
      - name: wordpress
        image: wordpress:latest
        volumeMounts:
        - name: wordpress-data
          mountPath: /var/www/html  # WordPress files
      volumes:
      - name: wordpress-data
        persistentVolumeClaim:
          claimName: wordpress-pvc

Result:
✓ User uploads photo → Saved to PV
✓ Pod crashes → New pod mounts same PV
✓ Photo still there! 🎉
✓ All content persists!
```

---

### Example 2: MySQL Database

**Problem:**
```
Database MUST keep data!
- Customer records
- Order history
- User accounts

Pod restart = Data loss = Disaster! 💥
```

**Solution with StatefulSet:**
```yaml
# StorageClass for fast database storage
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: db-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io2  # High-performance SSD
  iopsPerGB: "100"
  fsType: ext4

---
# MySQL StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 1
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql  # Database files
  volumeClaimTemplates:  # Automatic PVC per pod!
  - metadata:
      name: mysql-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: db-storage
      resources:
        requests:
          storage: 100Gi

What happens:
1. StatefulSet creates pod: mysql-0
2. Automatically creates PVC: mysql-data-mysql-0
3. StorageClass provisions 100Gi AWS EBS
4. Pod starts, mounts volume
5. MySQL writes data to /var/lib/mysql
6. Data stored on EBS volume

Pod deleted:
1. Pod goes away
2. PVC remains! ✓
3. Data remains! ✓
4. New pod created
5. Mounts SAME PVC
6. Data intact! ✓

Database survives anything! 🛡️
```

---

### Example 3: Shared Log Storage (Multiple Writers)

**Problem:**
```
Microservices architecture:
- 10 services
- All write logs
- Want centralized log storage

Need ReadWriteMany!
```

**Solution:**
```yaml
# Create NFS-based StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-logs
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.example.com
  share: /exports/logs

---
# PVC for shared logs
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-logs
spec:
  accessModes:
    - ReadWriteMany  # Multiple pods can write!
  resources:
    requests:
      storage: 50Gi
  storageClassName: nfs-logs

---
# Service A writes logs
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-a
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: service-a:latest
        volumeMounts:
        - name: logs
          mountPath: /var/log/app
      volumes:
      - name: logs
        persistentVolumeClaim:
          claimName: shared-logs  # Same PVC!

---
# Service B writes logs
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-b
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: app
        image: service-b:latest
        volumeMounts:
        - name: logs
          mountPath: /var/log/app
      volumes:
      - name: logs
        persistentVolumeClaim:
          claimName: shared-logs  # Same PVC!

Result:
✓ 3 service-a pods writing to /var/log/app
✓ 5 service-b pods writing to /var/log/app
✓ All on shared NFS
✓ Centralized logging! 📝
```

---

## Storage Performance

### Performance Characteristics

```
Storage Type Comparison:

1. hostPath (local disk):
   Throughput: 500 MB/s - 3 GB/s
   IOPS: 50,000 - 500,000
   Latency: <1ms
   Pros: ✓ Very fast!
   Cons: ❌ Not portable (tied to node)
         ❌ Data lost if node dies

2. AWS EBS (gp3):
   Throughput: 125 MB/s - 1 GB/s
   IOPS: 3,000 - 16,000
   Latency: <10ms
   Pros: ✓ Persistent
         ✓ Snapshots available
   Cons: ❌ Single AZ only
         ❌ Can't share (RWO only)

3. AWS EBS (io2):
   Throughput: Up to 4 GB/s
   IOPS: Up to 64,000
   Latency: <1ms
   Pros: ✓ Very fast!
         ✓ Persistent
   Cons: ❌ Expensive! ($$)
         ❌ Single AZ

4. NFS/EFS:
   Throughput: 100 MB/s - 10 GB/s
   IOPS: 1,000 - 500,000
   Latency: 1-10ms
   Pros: ✓ Shareable (RWX)
         ✓ Multi-AZ
   Cons: ❌ Network latency
         ❌ Variable performance

5. Ceph/Rook:
   Throughput: Variable
   IOPS: 1,000 - 100,000
   Latency: 1-5ms
   Pros: ✓ Distributed
         ✓ Highly available
         ✓ Self-hosted
   Cons: ❌ Complex to manage
         ❌ Resource intensive

Choose based on needs:
- Database → io2 (fast, consistent)
- General apps → gp3 (good balance)
- Shared files → EFS/NFS (multi-writer)
- Logs/cache → gp3 or EFS
```

---

## Common Problems & Solutions

### Problem 1: PVC Stuck in Pending

**Symptoms:**
```bash
kubectl get pvc
NAME      STATUS    VOLUME   CAPACITY
my-pvc    Pending            

kubectl describe pvc my-pvc
Events:
  Warning  ProvisioningFailed
    no volume plugin matched
```

**Causes & Solutions:**

**Cause 1: No matching PV**
```bash
# Check available PVs
kubectl get pv

# If empty or all Bound:
# Solution: Create PV or use dynamic provisioning
```

**Cause 2: StorageClass doesn't exist**
```bash
# Check StorageClasses
kubectl get storageclass

# If missing:
# Solution: Create StorageClass or fix name in PVC
```

**Cause 3: Size mismatch**
```yaml
# PVC wants 100Gi
# PV only has 50Gi

# Solution: Create bigger PV or reduce PVC request
```

---

### Problem 2: Pod Can't Mount Volume

**Symptoms:**
```bash
kubectl get pod
NAME       READY   STATUS              
my-pod     0/1     ContainerCreating   

kubectl describe pod my-pod
Events:
  Warning  FailedMount
    Unable to attach or mount volumes
    Volume is already exclusively attached to node
```

**Cause: ReadWriteOnce volume already mounted elsewhere**
```bash
# Check where volume is mounted
kubectl get pods -o wide | grep <pvc-name>

# Solution:
# 1. Delete old pod first
kubectl delete pod old-pod
# Wait for volume to detach (~30 seconds)

# 2. Then new pod can mount
kubectl apply -f new-pod.yaml
```

---

### Problem 3: Out of Space

**Symptoms:**
```bash
# App logs:
Error: No space left on device
Write failed: disk full

kubectl describe pvc my-pvc
Capacity: 10Gi
```

**Solutions:**

**Solution 1: Expand PVC (if supported)**
```yaml
# Edit PVC
kubectl edit pvc my-pvc

# Change:
resources:
  requests:
    storage: 10Gi  # Change to 20Gi

# Kubernetes automatically expands!
# (if StorageClass allows)
```

**Solution 2: Create new larger PVC**
```bash
# 1. Backup data
kubectl exec pod -- tar czf /tmp/backup.tar.gz /data

# 2. Copy backup out
kubectl cp pod:/tmp/backup.tar.gz ./backup.tar.gz

# 3. Create new larger PVC
kubectl apply -f new-larger-pvc.yaml

# 4. Create pod with new PVC
# 5. Restore data
kubectl cp ./backup.tar.gz pod:/tmp/
kubectl exec pod -- tar xzf /tmp/backup.tar.gz -C /
```

---

### Problem 4: Slow Disk Performance

**Symptoms:**
```bash
# App very slow
# Database queries taking forever

# Check disk I/O
kubectl exec pod -- iostat
# %iowait: 90%  ← Very high!
```

**Causes & Solutions:**

**Cause 1: Wrong storage type**
```yaml
# Using standard HDD for database
storageClassName: standard  # ❌ Slow!

# Solution: Use SSD
storageClassName: ssd  # ✓ Fast!
```

**Cause 2: IOPS limit hit**
```yaml
# AWS EBS gp3: 3000 IOPS default
# Database needs 10,000 IOPS

# Solution: Provision more IOPS
kind: StorageClass
parameters:
  type: io2
  iopsPerGB: "100"  # Higher IOPS!
```

**Cause 3: Network storage latency**
```yaml
# Using EFS for database ❌
# Network latency: 5-10ms

# Solution: Use local SSD or EBS
# Local: <1ms latency
# EBS: <1ms latency
```

---

## Data Protection Strategies

### 1. Volume Snapshots

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-snapshot-daily
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: mysql-data

What this does:
- Takes point-in-time snapshot of PVC
- Stored in cloud (EBS snapshot, etc.)
- Can restore from snapshot
- Incremental (only changed blocks)

Use cases:
- Daily backups
- Before upgrades
- Disaster recovery
```

---

### 2. Backup to Object Storage

```bash
# Backup to S3
kubectl exec db-pod -- mysqldump -u root -p db > dump.sql
aws s3 cp dump.sql s3://backups/mysql/$(date +%Y%m%d).sql

# Restore from S3
aws s3 cp s3://backups/mysql/20250101.sql dump.sql
kubectl cp dump.sql db-pod:/tmp/
kubectl exec db-pod -- mysql -u root -p db < /tmp/dump.sql

Pros:
✓ Cheap storage
✓ Long retention
✓ Cross-region copies

Cons:
❌ Slower restore
❌ Manual process
❌ Application downtime
```

---

### 3. Replication

```yaml
# MySQL with replication
# Primary: Writes
# Replicas: Read-only copies

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  replicas: 3  # 1 primary + 2 replicas
  # ... config for replication

Benefits:
✓ High availability
✓ Automatic failover
✓ Read scaling
✓ Data protected on multiple volumes
```

---

## Key Takeaways

### The 5 Most Important Concepts

**1. Containers are Ephemeral, Storage is Permanent**
```
Container dies → Data in container lost
PersistentVolume → Data survives

Always use PVs for important data!
```

**2. PVC is Your Storage Request**
```
You say: "I want 10GB of SSD storage"
Kubernetes finds or creates it
You mount it in your pod
Simple! 

PVC = Your interface to storage
```

**3. StorageClass Enables Self-Service**
```
Without StorageClass:
- Admin creates PVs manually
- Limited, slow, error-prone

With StorageClass:
- Automatic provisioning
- Unlimited (cloud-backed)
- Self-service for developers
- Much better!
```

**4. Access Modes Matter**
```
RWO (ReadWriteOnce): Most common
- One node at a time
- Databases, stateful apps

ROX (ReadOnlyMany): Rare
- Multiple readers
- Static content

RWX (ReadWriteMany): Special storage
- Multiple writers
- Requires NFS/CephFS/etc
- Expensive, slower

Choose correctly for your use case!
```

**5. Always Backup Important Data**
```
Strategies:
1. Volume snapshots (easy)
2. Export to object storage (cheap)
3. Replication (high availability)

Test your backups!
Test your restore process!
Don't wait for disaster to find out backups don't work!
```

---

## Real-World Best Practices

### 1. Set Storage Limits
```yaml
resources:
  requests:
    storage: 10Gi
  limits:
    storage: 20Gi  # Some clouds support this

Why:
- Prevent one app from filling disk
- Cost control
- Quota management
```

### 2. Use StorageClasses for Tiers
```yaml
# Fast tier (databases)
kind: StorageClass
metadata:
  name: fast
parameters:
  type: io2

# Medium tier (apps)
kind: StorageClass
metadata:
  name: standard
parameters:
  type: gp3

# Slow tier (archives)
kind: StorageClass
metadata:
  name: archive
parameters:
  type: sc1
```

### 3. Monitor Disk Usage
```bash
# Prometheus metrics
kubelet_volume_stats_used_bytes
kubelet_volume_stats_capacity_bytes

# Alert when >80% full
# Auto-expand or notify ops
```

### 4. Reclaim Policy = Retain in Production
```yaml
# Production databases
persistentVolumeReclaimPolicy: Retain

Why:
- Prevents accidental deletion
- Admin reviews before cleanup
- Safety first!
```

---

**You now understand Kubernetes storage - how to keep your data safe and persistent! This completes the foundational concepts of Kubernetes.** 💾

The Days 4-7 theory posts are now complete with the same detailed, beginner-friendly approach covering Controller Manager, kubelet, Networking, and Storage!