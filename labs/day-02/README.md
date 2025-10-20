# Day 2 Lab: API Server & etcd - The Brain & Memory

> **"Today you'll explore the brain of Kubernetes. You'll corrupt etcd, break the API server, and learn why the control plane is so critical."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Explore etcd directly and understand what it stores
- Break etcd and see the cluster become read-only
- Corrupt API server configuration and recover
- Practice etcd backup and disaster recovery
- Understand the control plane architecture
- Learn why etcd is the most critical component

## ⚠️ Prerequisites

- Day 1 lab completed
- Minikube cluster running
- 4-5 hours of uninterrupted time
- Backup your work (etcd corruption is real!)

## 🛠️ Setup (30 minutes)

### Step 1: Verify Cluster Status

```bash
# Check cluster is running
kubectl get nodes
kubectl get pods -n kube-system

# Create some test data
kubectl create namespace important-app
kubectl create deployment critical --image=nginx -n important-app --replicas=3
kubectl create configmap important-config --from-literal=key=value -n important-app
kubectl create secret generic important-secret --from-literal=password=secret123 -n important-app

# Verify everything exists
kubectl get all -n important-app
kubectl get configmap important-config -n important-app
kubectl get secret important-secret -n important-app
```

### Step 2: Setup Monitoring

```bash
# Open 4 terminals
# Terminal 1: kubectl get pods --all-namespaces --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: kubectl get --raw /healthz -v 6
# Terminal 4: Your command terminal
```

---

## 🧪 Lab Part 1: Explore etcd (1 hour)

### Exercise 1.1: Access etcd Directly

```bash
# Get shell in etcd pod
kubectl exec -it etcd-minikube -n kube-system -- sh

# Set up etcdctl environment
export ETCDCTL_API=3
export ETCDCTL_CACERT=/var/lib/minikube/certs/etcd/ca.crt
export ETCDCTL_CERT=/var/lib/minikube/certs/etcd/server.crt
export ETCDCTL_KEY=/var/lib/minikube/certs/etcd/server.key

# Test connection
etcdctl endpoint health
# Should show: {"health":"true"}
```

### Exercise 1.2: Explore etcd Contents

```bash
# See all keys (first 20)
etcdctl get / --prefix --keys-only | head -20

# Count total objects
etcdctl get / --prefix --keys-only | wc -l
# My cluster: 287 objects

# Explore specific namespaces
etcdctl get /registry/namespaces --prefix --keys-only

# See your important-app namespace
etcdctl get /registry/namespaces/important-app
```

### Exercise 1.3: Examine Object Storage

```bash
# Look at your deployment
etcdctl get /registry/deployments/important-app/critical

# Look at your pods
etcdctl get /registry/pods --prefix --keys-only | grep important-app

# Look at your configmap
etcdctl get /registry/configmaps/important-app/important-config

# Look at your secret (notice it's base64 encoded!)
etcdctl get /registry/secrets/important-app/important-secret
```

### Exercise 1.4: Decode Secrets (Security Lesson!)

```bash
# Get the secret data
etcdctl get /registry/secrets/important-app/important-config

# You'll see base64 encoded data like:
# "password":"c2VjcmV0MTIz"  # This is base64!

# Decode it (still in etcd pod)
echo "c2VjcmV0MTIz" | base64 -d
# Output: secret123

# EXIT etcd pod
exit
```

**Key Learning:** etcd stores secrets in base64, not encrypted! This is why you need:
- etcd encryption at rest
- Network encryption (TLS)
- Access controls
- Regular backups

---

## 💀 Lab Part 2: Break #3 - Stop etcd (1 hour)

> **"Time to break the brain. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create more test data
kubectl create deployment chaos-test --image=nginx --replicas=5
kubectl create service clusterip chaos-test --tcp=80:80

# Verify everything is running
kubectl get all
```

### Exercise 2.2: Break etcd

```bash
# Find etcd container
minikube ssh
sudo docker ps | grep etcd
# Note the container ID

# BREAK IT!
sudo docker stop <etcd-container-id>

# Verify it's dead
sudo docker ps | grep etcd
# Should be empty
```

### Exercise 2.3: Observe the Chaos

**Watch Terminal 1 (pods):**
- Existing pods keep running
- No new pods can be created
- kubectl commands hang

**Watch Terminal 2 (events):**
- No new events generated
- Existing events remain

**Watch Terminal 3 (API health):**
```bash
# Try API health check
kubectl get --raw /healthz
# Hangs... eventually times out
```

**Try to create something:**
```bash
kubectl create deployment test --image=nginx
# Error: The connection to the server was refused

# But existing apps still work!
curl $(minikube service chaos-test --url)
# STILL WORKS!
```

### Exercise 2.4: Debug the Problem

```bash
# Check API server logs
kubectl logs kube-apiserver-minikube -n kube-system
# If kubectl doesn't work, SSH in:

minikube ssh
sudo docker logs <api-server-container-id>
# See: "failed to connect to etcd"
```

**Key Learning:** 
- etcd is the single source of truth
- API Server can't function without etcd
- Existing workloads unaffected (they don't need API)
- Cluster becomes read-only

### Exercise 2.5: Fix It

```bash
# Still in SSH
sudo docker start <etcd-container-id>

# Wait 30 seconds
exit

# Test
kubectl get pods
# Works again!
```

**Real-world parallel:** This happens when:
- etcd disk fills up
- etcd process crashes
- Network partition isolates etcd
- Hardware failure

---

## 🔥 Lab Part 3: Break #4 - Corrupt API Server Config (1 hour)

> **"Now let's break the front door. This is where things get really interesting."**

### Exercise 3.1: Backup API Server Config

```bash
# API server runs as "static pod" - managed by kubelet
# Config in: /etc/kubernetes/manifests/

minikube ssh
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/backup.yaml

# Verify backup
sudo ls -la /tmp/backup.yaml
```

### Exercise 3.2: Corrupt the Configuration

```bash
# Edit API server config
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml

# Find the command section and ADD this invalid flag:
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.49.2
    - --this-flag-does-not-exist=true  # ADD THIS LINE
    # ... rest of config

# Save and exit (:wq)
```

### Exercise 3.3: Watch It Break

```bash
# Check API server status
sudo docker ps | grep kube-apiserver
# You'll see it restarting constantly

# Check logs
sudo docker logs <api-server-container-id>
# Error: unknown flag: --this-flag-does-not-exist

# Try kubectl from your machine
exit
kubectl get nodes
# Error: connection refused
```

**The cluster is DEAD:**
```bash
kubectl get pods       # Fails
kubectl get services   # Fails
kubectl get nodes      # Fails

# Even existing apps might stop working soon
# (if kube-proxy or kubelet needs to talk to API)
```

### Exercise 3.4: Debug the Problem

```bash
minikube ssh

# Check API server logs
sudo docker logs <api-server-container-id>
# See: "unknown flag: --this-flag-does-not-exist"

# Check if API server is running
sudo docker ps | grep kube-apiserver
# Shows restarting constantly
```

**Key Learning:** 
- API Server validates all flags at startup
- Invalid flags cause immediate failure
- Static pods are managed by kubelet, not API server
- Configuration errors can kill the entire cluster

### Exercise 3.5: Fix It (Disaster Recovery)

```bash
# Restore backup
sudo cp /tmp/backup.yaml /etc/kubernetes/manifests/kube-apiserver.yaml

# Watch it recover
sudo docker ps | grep kube-apiserver
# Should show running (not restarting)

# Wait 30 seconds
exit

kubectl get nodes
# Works!
```

**Real-world parallel:** This happens when:
- Kubernetes upgrade with wrong flags
- Configuration drift
- Manual edits to manifests
- Automation errors

---

## 🧠 Lab Part 4: etcd Backup & Disaster Recovery (1 hour)

> **"Time to practice real disaster recovery. This is where you become a hero."**

### Exercise 4.1: Create Test Data

```bash
# Create critical application
kubectl create namespace production
kubectl create deployment web-app --image=nginx -n production --replicas=3
kubectl create service clusterip web-app --tcp=80:80 -n production
kubectl create configmap app-config --from-literal=database=prod-db -n production
kubectl create secret generic db-credentials --from-literal=password=super-secret -n production

# Verify everything exists
kubectl get all -n production
kubectl get configmap app-config -n production
kubectl get secret db-credentials -n production
```

### Exercise 4.2: Backup etcd

```bash
# Get shell in etcd pod
kubectl exec -it etcd-minikube -n kube-system -- sh

# Set up etcdctl
export ETCDCTL_API=3
export ETCDCTL_CACERT=/var/lib/minikube/certs/etcd/ca.crt
export ETCDCTL_CERT=/var/lib/minikube/certs/etcd/server.crt
export ETCDCTL_KEY=/var/lib/minikube/certs/etcd/server.key

# Create backup
etcdctl snapshot save /var/lib/minikube/etcd-backup.db

# Verify backup
etcdctl snapshot status /var/lib/minikube/etcd-backup.db
# Shows: hash, revision, total keys, total size

exit
```

### Exercise 4.3: Simulate Disaster

```bash
# DELETE EVERYTHING (simulate complete cluster loss)
kubectl delete namespace production

# Verify it's gone
kubectl get namespace production
# Error: not found

# Oh no! Critical app is gone!
```

### Exercise 4.4: Restore from Backup

```bash
minikube ssh

# Stop etcd (required for restore)
sudo docker stop $(sudo docker ps | grep etcd | awk '{print $1}')

# Restore from backup
sudo ETCDCTL_API=3 etcdctl snapshot restore \
  /var/lib/minikube/etcd-backup.db \
  --data-dir=/var/lib/minikube/etcd-restore \
  --cacert=/var/lib/minikube/certs/etcd/ca.crt \
  --cert=/var/lib/minikube/certs/etcd/server.crt \
  --key=/var/lib/minikube/certs/etcd/server.key

# Point etcd to restored data
sudo mv /var/lib/minikube/etcd /var/lib/minikube/etcd-old
sudo mv /var/lib/minikube/etcd-restore /var/lib/minikube/etcd

# Restart etcd
sudo docker start $(sudo docker ps -a | grep etcd | awk '{print $1}')

exit

# Wait 30 seconds, then check
kubectl get namespace production
# IT'S BACK!

kubectl get all -n production
# Everything restored!
```

**Real-world parallel:** This is exactly what saved GitLab.com in 2017 when they accidentally deleted their production database.

---

## 🎯 Lab Part 5: Advanced etcd Operations (45 minutes)

### Exercise 5.1: etcd Performance Testing

```bash
# Get shell in etcd pod
kubectl exec -it etcd-minikube -n kube-system -- sh

# Set up etcdctl
export ETCDCTL_API=3
export ETCDCTL_CACERT=/var/lib/minikube/certs/etcd/ca.crt
export ETCDCTL_CERT=/var/lib/minikube/certs/etcd/server.crt
export ETCDCTL_KEY=/var/lib/minikube/certs/etcd/server.key

# Test etcd performance
etcdctl endpoint status --write-out=table
# Shows: version, db size, leader, raft index, etc.

# Test etcd latency
etcdctl endpoint health
# Shows: response time
```

### Exercise 5.2: etcd Defragmentation

```bash
# Check etcd database size
etcdctl endpoint status --write-out=table

# Defragment etcd (reclaim space)
etcdctl defrag

# Check size again
etcdctl endpoint status --write-out=table
# Should be smaller
```

### Exercise 5.3: etcd Member Management

```bash
# List etcd members
etcdctl member list

# Check etcd cluster health
etcdctl endpoint health --cluster
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-02-reflections.md` and answer:

1. **What does etcd store?**
   - How many objects are in your cluster?
   - What format are secrets stored in?
   - Why is etcd the single source of truth?

2. **What happens when etcd dies?**
   - Can you create new resources?
   - Do existing workloads keep running?
   - How long does it take to detect the failure?

3. **What happens when API Server config is corrupted?**
   - How does kubelet manage static pods?
   - What's the difference between static pods and regular pods?
   - How do you recover from configuration errors?

4. **How do you backup and restore etcd?**
   - What's the process for disaster recovery?
   - How do you verify backups work?
   - What are the risks of restore operations?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Salesforce etcd disk space issue (2019)**
- **GitLab.com database deletion and recovery (2017)**
- **Kubernetes API server upgrade failures**

### Exercise 6.3: Security Implications

```bash
# Check what's stored in etcd
kubectl exec -it etcd-minikube -n kube-system -- sh
export ETCDCTL_API=3
export ETCDCTL_CACERT=/var/lib/minikube/certs/etcd/ca.crt
export ETCDCTL_CERT=/var/lib/minikube/certs/etcd/server.crt
export ETCDCTL_KEY=/var/lib/minikube/certs/etcd/server.key

# Look at secrets
etcdctl get /registry/secrets --prefix --keys-only

# Decode a secret
etcdctl get /registry/secrets/default/important-secret
# Notice: base64 encoded, not encrypted!
```

**Key Learning:** etcd security is critical:
- Enable etcd encryption at rest
- Use network encryption (TLS)
- Implement access controls
- Regular backups
- Monitor etcd health

---

## 🏆 Success Criteria

You've successfully completed Day 2 if you can:

✅ **Explore etcd directly and understand its contents**  
✅ **Break etcd and understand the impact**  
✅ **Corrupt API server configuration and recover**  
✅ **Backup and restore etcd successfully**  
✅ **Explain why etcd is the most critical component**  
✅ **Understand etcd security implications**  

## 🚨 Common Issues & Solutions

### Issue: Can't access etcd
```bash
# Check etcd pod is running
kubectl get pods -n kube-system | grep etcd

# Check etcd logs
kubectl logs etcd-minikube -n kube-system
```

### Issue: etcd backup fails
```bash
# Check etcd is healthy
kubectl exec -it etcd-minikube -n kube-system -- etcdctl endpoint health

# Check disk space
kubectl exec -it etcd-minikube -n kube-system -- df -h
```

### Issue: API server won't start
```bash
# Check manifest syntax
minikube ssh
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Check for invalid flags
sudo docker logs <api-server-container-id>
```

---

## 🎯 Tomorrow's Preview

**Day 3: Scheduler - The Tetris Master**
- Watch scheduler work in real-time
- Break scheduler and see pods stuck in Pending
- Test resource exhaustion scenarios
- Practice node affinity and manual scheduling
- Understand scheduling algorithms

**Get ready for more chaos! 🔥**

---

## 📚 Additional Resources

- [etcd Documentation](https://etcd.io/docs/)
- [Kubernetes API Server](https://kubernetes.io/docs/concepts/overview/components/#kube-apiserver)
- [etcd Backup and Restore](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- [Static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)

**Remember: etcd is the brain of Kubernetes. Protect it with your life! 🧠**
