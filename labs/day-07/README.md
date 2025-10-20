# Day 7 Lab: Storage - The Memory System

> **"Today you'll break storage and watch pods lose their data. You'll learn why storage is the memory system of Kubernetes."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch storage components work in real-time
- Break storage and see pods lose data
- Test persistent volume failures
- Practice storage backup and recovery
- Understand the storage architecture
- Learn why storage is critical for stateful applications

## ⚠️ Prerequisites

- Day 6 lab completed
- Minikube cluster running
- 4-5 hours of uninterrupted time
- Understanding of storage concepts

## 🛠️ Setup (30 minutes)

### Step 1: Verify Cluster Status

```bash
# Check cluster is running
kubectl get nodes
kubectl get pods -n kube-system

# Clean up previous lab resources
kubectl delete deployment chaos-test test-backend
kubectl delete service chaos-test test-svc
kubectl delete namespace network-test
```

### Step 2: Setup Monitoring

```bash
# Open 4 terminals
# Terminal 1: kubectl get pods --all-namespaces --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: kubectl get pv,pvc --watch
# Terminal 4: Your command terminal
```

### Step 3: Check Storage Components

```bash
# Check storage classes
kubectl get storageclass

# Check persistent volumes
kubectl get pv

# Check persistent volume claims
kubectl get pvc
```

---

## 🧪 Lab Part 1: Watch Storage Work (45 minutes)

### Exercise 1.1: Create Persistent Volume

```bash
# Create persistent volume
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/test-pv
EOF

# Check PV
kubectl get pv test-pv
# STATUS: Available
```

### Exercise 1.2: Create Persistent Volume Claim

```bash
# Create PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# Check PVC
kubectl get pvc test-pvc
# STATUS: Bound

# Check PV
kubectl get pv test-pv
# STATUS: Bound
```

### Exercise 1.3: Create Pod with Storage

```bash
# Create pod with storage
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: storage-test
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: test-pvc
EOF

# Check pod
kubectl get pod storage-test
# STATUS: Running
```

### Exercise 1.4: Test Storage

```bash
# Write data to storage
kubectl exec storage-test -- sh -c "echo 'Hello from storage' > /data/test.txt"

# Read data from storage
kubectl exec storage-test -- cat /data/test.txt
# Should show: Hello from storage

# Check storage on host
minikube ssh
sudo cat /tmp/test-pv/test.txt
# Should show: Hello from storage
```

---

## 💀 Lab Part 2: Break #9 - Delete Storage (1 hour)

> **"Time to break the storage. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test data
kubectl exec storage-test -- sh -c "echo 'Important data' > /data/important.txt"
kubectl exec storage-test -- sh -c "echo 'Critical data' > /data/critical.txt"

# Verify data exists
kubectl exec storage-test -- ls -la /data/
# Should show: test.txt, important.txt, critical.txt
```

### Exercise 2.2: Break Storage

```bash
# Delete the pod
kubectl delete pod storage-test

# Delete the PVC
kubectl delete pvc test-pvc

# Delete the PV
kubectl delete pv test-pv

# Check storage on host
minikube ssh
sudo ls -la /tmp/test-pv/
# Should show: test.txt, important.txt, critical.txt
# Data still exists on host!
```

### Exercise 2.3: Observe the Chaos

**Watch Terminal 1 (pods):**
- Pod shows "Terminating" then disappears
- No storage available for new pods

**Try to create new pod:**
```bash
# Create new pod with same storage
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: storage-test-2
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: test-pvc
EOF

# Check pod status
kubectl get pod storage-test-2
# STATUS: Pending

# Check events
kubectl describe pod storage-test-2
# Events: FailedMount: Unable to mount volumes
```

### Exercise 2.4: Debug the Problem

```bash
# Check PVC
kubectl get pvc test-pvc
# Error: not found

# Check PV
kubectl get pv test-pv
# Error: not found

# Check storage on host
minikube ssh
sudo ls -la /tmp/test-pv/
# Data still exists on host!
```

**Key Learning:** 
- Storage is persistent even when pods are deleted
- PVC/PV deletion doesn't delete host data
- Pods can't start without storage
- Data recovery is possible

### Exercise 2.5: Fix It

```bash
# Recreate storage
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/test-pv
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# Check pod status
kubectl get pod storage-test-2
# STATUS: Running

# Check data
kubectl exec storage-test-2 -- ls -la /data/
# Should show: test.txt, important.txt, critical.txt
# Data is back!
```

**Real-world parallel:** This happens when:
- Storage administrator deletes volumes
- Storage system failures
- Human error
- Automation errors

---

## 🔥 Lab Part 3: Storage System Failures (1 hour)

> **"Time to test what happens when the storage system fails. This is where storage resilience is tested."**

### Exercise 3.1: Test Storage System Failure

```bash
# Create storage with specific path
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-pv-2
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/test-pv-2
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc-2
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: storage-test-3
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: test-pvc-2
EOF

# Check pod status
kubectl get pod storage-test-3
# STATUS: Running
```

### Exercise 3.2: Simulate Storage System Failure

```bash
# SSH into node
minikube ssh

# Create the storage directory
sudo mkdir -p /tmp/test-pv-2

# Write some data
echo "Important data" | sudo tee /tmp/test-pv-2/data.txt

# Check data
sudo cat /tmp/test-pv-2/data.txt
# Should show: Important data

# SIMULATE STORAGE FAILURE - Delete the directory
sudo rm -rf /tmp/test-pv-2

# Exit SSH
exit
```

### Exercise 3.3: Test Pod with Failed Storage

```bash
# Check pod status
kubectl get pod storage-test-3
# STATUS: Running (but storage is gone!)

# Try to access data
kubectl exec storage-test-3 -- ls -la /data/
# Should show: No such file or directory

# Try to write data
kubectl exec storage-test-3 -- sh -c "echo 'New data' > /data/new.txt"
# Should fail: No such file or directory
```

### Exercise 3.4: Fix Storage System

```bash
# SSH back into node
minikube ssh

# Recreate storage directory
sudo mkdir -p /tmp/test-pv-2

# Restore data (if you have backup)
echo "Important data" | sudo tee /tmp/test-pv-2/data.txt

# Exit SSH
exit

# Test pod again
kubectl exec storage-test-3 -- ls -la /data/
# Should show: data.txt

kubectl exec storage-test-3 -- cat /data/data.txt
# Should show: Important data
```

**Real-world parallel:** This happens when:
- Storage system crashes
- Disk failures
- Network storage disconnections
- Hardware failures

---

## 🧠 Lab Part 4: Storage Backup and Recovery (1 hour)

> **"Time to practice real storage backup and recovery. This is where you become a storage hero."**

### Exercise 4.1: Create Important Data

```bash
# Create namespace for testing
kubectl create namespace storage-test

# Create storage
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: important-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/important-pv
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: important-pvc
  namespace: storage-test
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: important-app
  namespace: storage-test
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: important-pvc
EOF

# Wait for pod to be ready
kubectl wait --for=condition=ready pod important-app -n storage-test

# Create important data
kubectl exec important-app -n storage-test -- sh -c "echo 'Critical business data' > /data/business.txt"
kubectl exec important-app -n storage-test -- sh -c "echo 'User data' > /data/users.txt"
kubectl exec important-app -n storage-test -- sh -c "echo 'Configuration' > /data/config.txt"

# Verify data
kubectl exec important-app -n storage-test -- ls -la /data/
# Should show: business.txt, users.txt, config.txt
```

### Exercise 4.2: Backup Storage

```bash
# SSH into node
minikube ssh

# Create backup directory
sudo mkdir -p /tmp/backup

# Backup storage
sudo cp -r /tmp/important-pv/* /tmp/backup/

# Verify backup
sudo ls -la /tmp/backup/
# Should show: business.txt, users.txt, config.txt

# Exit SSH
exit
```

### Exercise 4.3: Simulate Disaster

```bash
# Delete everything
kubectl delete pod important-app -n storage-test
kubectl delete pvc important-pvc -n storage-test
kubectl delete pv important-pv
kubectl delete namespace storage-test

# Simulate storage failure
minikube ssh
sudo rm -rf /tmp/important-pv
exit

# Oh no! Critical data is gone!
```

### Exercise 4.4: Restore from Backup

```bash
# SSH into node
minikube ssh

# Recreate storage directory
sudo mkdir -p /tmp/important-pv

# Restore from backup
sudo cp -r /tmp/backup/* /tmp/important-pv/

# Verify restoration
sudo ls -la /tmp/important-pv/
# Should show: business.txt, users.txt, config.txt

# Exit SSH
exit

# Recreate storage
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: important-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/important-pv
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: important-pvc
  namespace: storage-test
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: important-app
  namespace: storage-test
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: important-pvc
EOF

# Wait for pod to be ready
kubectl wait --for=condition=ready pod important-app -n storage-test

# Check data
kubectl exec important-app -n storage-test -- ls -la /data/
# Should show: business.txt, users.txt, config.txt

kubectl exec important-app -n storage-test -- cat /data/business.txt
# Should show: Critical business data

# Data is restored!
```

**Real-world parallel:** This is exactly what saved many companies during:
- Ransomware attacks
- Hardware failures
- Human errors
- System crashes

---

## 🎯 Lab Part 5: Advanced Storage Scenarios (45 minutes)

### Exercise 5.1: Test Storage Classes

```bash
# Check available storage classes
kubectl get storageclass

# Create storage class
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF

# Create PVC with storage class
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-pvc
spec:
  storageClassName: fast-ssd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# Check PVC status
kubectl get pvc fast-pvc
# STATUS: Pending (waiting for first consumer)
```

### Exercise 5.2: Test Storage Expansion

```bash
# Create storage with expansion
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: expandable-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/expandable-pv
  storageClassName: expandable
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: expandable-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: expandable
EOF

# Check storage
kubectl get pv expandable-pv
kubectl get pvc expandable-pvc
# Both should be Bound
```

### Exercise 5.3: Test Storage Migration

```bash
# Create pod with storage
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: migration-test
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: expandable-pvc
EOF

# Check pod status
kubectl get pod migration-test
# STATUS: Running

# Test storage
kubectl exec migration-test -- sh -c "echo 'Migration test data' > /data/migration.txt"
kubectl exec migration-test -- cat /data/migration.txt
# Should show: Migration test data
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-07-reflections.md` and answer:

1. **What does storage do in Kubernetes?**
   - How does persistent storage work?
   - What's the difference between PV and PVC?
   - Why is storage critical for stateful applications?

2. **What happens when storage fails?**
   - How do you detect storage failures?
   - What happens to pods with failed storage?
   - How do you recover from storage failures?

3. **How do you backup and restore storage?**
   - What's the process for storage backup?
   - How do you restore from backup?
   - What are the risks of storage operations?

4. **How do you debug storage issues?**
   - What tools do you use?
   - How do you check storage status?
   - How do you test storage connectivity?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **GitLab.com database deletion and recovery (2017)**
- **Kubernetes storage failures**
- **Storage backup and recovery procedures**

### Exercise 6.3: Advanced Storage Scenarios

```bash
# Create complex storage scenario
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: complex-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/complex-pv
  storageClassName: complex
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: complex-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: complex
---
apiVersion: v1
kind: Pod
metadata:
  name: complex-storage-test
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: complex-pvc
EOF

# Test the storage
kubectl exec complex-storage-test -- sh -c "echo 'Complex storage test' > /data/complex.txt"
kubectl exec complex-storage-test -- cat /data/complex.txt
# Should show: Complex storage test
```

---

## 🏆 Success Criteria

You've successfully completed Day 7 if you can:

✅ **Watch storage components work in real-time**  
✅ **Break storage and understand the impact**  
✅ **Test persistent volume failures**  
✅ **Practice storage backup and recovery**  
✅ **Understand the storage architecture**  
✅ **Debug storage issues**  

## 🚨 Common Issues & Solutions

### Issue: Pods can't mount storage
```bash
# Check PVC status
kubectl get pvc

# Check PV status
kubectl get pv

# Check pod events
kubectl describe pod <pod-name>
```

### Issue: Storage not persistent
```bash
# Check storage on host
minikube ssh
sudo ls -la /tmp/test-pv/

# Check pod volume mounts
kubectl describe pod <pod-name>
```

### Issue: Storage backup fails
```bash
# Check storage directory exists
minikube ssh
sudo ls -la /tmp/important-pv/

# Check backup directory
sudo ls -la /tmp/backup/
```

---

## 🎯 Tomorrow's Preview

**Day 8: Security - The Immune System**
- Watch security components work
- Break security and see pods become vulnerable
- Test RBAC and network policy failures
- Practice security hardening procedures
- Understand the security architecture

**Get ready for more chaos! 🔥**

---

## 📚 Additional Resources

- [Kubernetes Storage](https://kubernetes.io/docs/concepts/storage/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Volume Lifecycle](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#lifecycle)

**Remember: Storage is the memory system of Kubernetes - it remembers everything! 💾**
