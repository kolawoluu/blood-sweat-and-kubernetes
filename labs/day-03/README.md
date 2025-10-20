# Day 3 Lab: Scheduler - The Tetris Master

> **"Today you'll break the scheduler and watch pods get stuck in Pending forever. You'll learn why scheduling is like playing Tetris with your infrastructure."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch scheduler work in real-time
- Break scheduler and see pods stuck in Pending
- Test resource exhaustion scenarios
- Practice node affinity and manual scheduling
- Understand scheduling algorithms
- Learn why scheduler is critical for new workloads

## ⚠️ Prerequisites

- Day 2 lab completed
- Minikube cluster running
- 4-5 hours of uninterrupted time
- Understanding of resource requests/limits

## 🛠️ Setup (30 minutes)

### Step 1: Create Multi-Node Cluster

```bash
# Delete existing cluster
minikube delete

# Start multi-node cluster
minikube start --nodes=3 --cpus=2 --memory=2048

# Verify 3 nodes
kubectl get nodes
# NAME           STATUS   ROLES
# minikube       Ready    control-plane
# minikube-m02   Ready    <none>
# minikube-m03   Ready    <none>
```

### Step 2: Setup Monitoring

```bash
# Open 4 terminals
# Terminal 1: kubectl get pods --all-namespaces --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: kubectl get nodes --watch
# Terminal 4: Your command terminal
```

### Step 3: Check Node Resources

```bash
# Check node capacity
kubectl describe node minikube
kubectl describe node minikube-m02
kubectl describe node minikube-m03

# Note the capacity:
# Capacity:
#   cpu:     2
#   memory:  2048Mi
```

---

## 🧪 Lab Part 1: Watch Scheduler Work (45 minutes)

### Exercise 1.1: Basic Scheduling

```bash
# Create deployment with many replicas
kubectl create deployment spread --image=nginx --replicas=9

# Watch pods being scheduled
kubectl get pods -o wide --watch

# You should see pods distributed across all 3 nodes
# Scheduler tries to spread pods evenly
```

### Exercise 1.2: Observe Scheduling Events

```bash
# Watch scheduling events
kubectl get events --watch --field-selector reason=Scheduled

# You'll see events like:
# Pod spread-xxx-abc scheduled to minikube-m02
# Pod spread-xxx-def scheduled to minikube-m03
# Pod spread-xxx-ghi scheduled to minikube
```

### Exercise 1.3: Check Node Resource Usage

```bash
# Check which nodes have which pods
kubectl get pods -o wide

# Check node resource usage
kubectl top nodes
# If metrics not available, use:
kubectl describe node minikube-m02 | grep -A 5 "Allocated resources"
```

### Exercise 1.4: Test Scheduling with Different Resources

```bash
# Create pod with specific resource requests
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-test
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "1000m"
        memory: "1Gi"
EOF

# Check where it was scheduled
kubectl get pod resource-test -o wide
```

---

## 💀 Lab Part 2: Break #5 - Stop Scheduler (1 hour)

> **"Time to break the Tetris master. This is where things get really interesting."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test deployment
kubectl create deployment chaos-test --image=nginx --replicas=3

# Verify it's running
kubectl get pods -l app=chaos-test
```

### Exercise 2.2: Break the Scheduler

```bash
# Find scheduler pod
kubectl get pods -n kube-system | grep scheduler

# Delete scheduler pod (it will restart)
kubectl delete pod -n kube-system kube-scheduler-minikube

# Wait 10 seconds, then prevent it from restarting
minikube ssh
sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/
exit
```

### Exercise 2.3: Observe the Chaos

**Watch Terminal 1 (pods):**
- Existing pods keep running
- New pods can't be scheduled

**Try to create new pods:**
```bash
kubectl create deployment test --image=nginx --replicas=3

# Check status
kubectl get pods
# NAME                    READY   STATUS    RESTARTS   AGE
# test-5d4d4d4d-abc       0/1     Pending   0          30s
# test-5d4d4d4d-def       0/1     Pending   0          30s
# test-5d4d4d4d-ghi       0/1     Pending   0          30s

# All stuck in Pending!
```

### Exercise 2.4: Debug the Problem

```bash
# Describe a pending pod
kubectl describe pod test-5d4d4d4d-abc

# Events section shows:
# Warning  FailedScheduling  pod unschedulable: no nodes available

# Check scheduler
kubectl get pods -n kube-system | grep scheduler
# NOT FOUND!

# Existing pods still run fine
kubectl get pods
# Old pods: Running
# New pods: Pending
```

**Key Learning:** 
- Scheduler only needed for NEW pods
- Existing pods unaffected
- No scheduling = pods stuck in Pending
- Cluster can't scale or add new services

### Exercise 2.5: Fix It

```bash
minikube ssh
sudo mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/
exit

# Wait 30 seconds
kubectl get pods -n kube-system | grep scheduler
# Running!

# Check pending pods
kubectl get pods
# All transition from Pending → Running!
```

**Real-world parallel:** This happens when:
- Scheduler crashes during traffic spike
- Scheduler pod runs out of memory
- Scheduler configuration errors
- Node failures affect scheduler

---

## 🔥 Lab Part 3: Resource Exhaustion (1 hour)

> **"Time to test what happens when nodes run out of resources. This is where scheduling gets really interesting."**

### Exercise 3.1: Test Memory Exhaustion

```bash
# Create pod requesting ENTIRE node's memory
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: memory-hog
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "3Gi"  # More than node has!
        cpu: "2"
EOF

# Check status
kubectl get pod memory-hog
# STATUS: Pending

kubectl describe pod memory-hog
# Events:
# Warning  FailedScheduling  0/3 nodes available: 
#   3 Insufficient memory
```

### Exercise 3.2: Test CPU Exhaustion

```bash
# Create pod requesting more CPU than available
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cpu-hog
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "3"  # More than node has!
EOF

# Check status
kubectl get pod cpu-hog
# STATUS: Pending

kubectl describe pod cpu-hog
# Events:
# Warning  FailedScheduling  0/3 nodes available: 
#   3 Insufficient cpu
```

### Exercise 3.3: Test Realistic Resource Requests

```bash
# Create pod with realistic resource requests
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: realistic
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
EOF

# This one schedules immediately
kubectl get pod realistic
# STATUS: Running
```

### Exercise 3.4: Test Node Affinity

```bash
# Label nodes
kubectl label node minikube-m02 disktype=ssd
kubectl label node minikube-m03 disktype=hdd

# Create pod with affinity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ssd-only
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
  - name: nginx
    image: nginx
EOF

# Check where it landed
kubectl get pod ssd-only -o wide
# NODE: minikube-m02 (the SSD node)
```

### Exercise 3.5: Test Impossible Affinity

```bash
# Create pod with impossible affinity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: impossible
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - nvme  # No node has this!
  containers:
  - name: nginx
    image: nginx
EOF

kubectl get pod impossible
# STATUS: Pending

kubectl describe pod impossible
# Events: 0/3 nodes available: 3 node(s) didn't match node affinity rules
```

---

## 🧠 Lab Part 4: Manual Scheduling (45 minutes)

> **"Time to bypass the scheduler and assign pods manually. This is where you become the scheduler."**

### Exercise 4.1: Manual Pod Assignment

```bash
# Create pod with nodeName set (bypasses scheduler)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: manual
spec:
  nodeName: minikube-m03  # Directly assign to node
  containers:
  - name: nginx
    image: nginx
EOF

# Check where it was scheduled
kubectl get pod manual -o wide
# NODE: minikube-m03
```

### Exercise 4.2: Test Node Cordon

```bash
# Cordon a node (prevent new pods)
kubectl cordon minikube-m02

# Check node status
kubectl get nodes
# minikube-m02   Ready,SchedulingDisabled

# Create pods - they avoid cordoned node
kubectl create deployment avoid --image=nginx --replicas=6
kubectl get pods -o wide
# All pods on minikube and minikube-m03, none on m02
```

### Exercise 4.3: Test Node Drain

```bash
# Drain a node (evict all pods)
kubectl drain minikube-m03 --ignore-daemonsets --delete-emptydir-data

# Check what happened
kubectl get pods -o wide
# Pods on m03 should be evicted and recreated on other nodes
```

### Exercise 4.4: Test Node Uncordon

```bash
# Uncordon the node
kubectl uncordon minikube-m02

# Check node status
kubectl get nodes
# minikube-m02   Ready

# Now new pods can be scheduled on m02
```

---

## 🎯 Lab Part 5: Advanced Scheduling (45 minutes)

### Exercise 5.1: Test Pod Topology Spread

```bash
# Create deployment with topology spread
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-test
spec:
  replicas: 6
  selector:
    matchLabels:
      app: spread-test
  template:
    metadata:
      labels:
        app: spread-test
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: spread-test
      containers:
      - name: nginx
        image: nginx
EOF

# Check pod distribution
kubectl get pods -o wide
# Should be evenly distributed across nodes
```

### Exercise 5.2: Test Taints and Tolerations

```bash
# Taint a node
kubectl taint node minikube-m02 special=true:NoSchedule

# Create pod without toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration
spec:
  containers:
  - name: nginx
    image: nginx
EOF

# Check status
kubectl get pod no-toleration
# STATUS: Pending

# Create pod with toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: special
    operator: Equal
    value: "true"
    effect: NoSchedule
  containers:
  - name: nginx
    image: nginx
EOF

# Check status
kubectl get pod with-toleration
# STATUS: Running
```

### Exercise 5.3: Test Priority Classes

```bash
# Create priority class
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "High priority class"
EOF

# Create pod with priority
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: high-priority-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: nginx
    image: nginx
EOF

# Check status
kubectl get pod high-priority-pod
# STATUS: Running
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-03-reflections.md` and answer:

1. **What does the scheduler do?**
   - How does it decide where to place pods?
   - What factors does it consider?
   - Why is it critical for new workloads?

2. **What happens when scheduler dies?**
   - Can you create new pods?
   - Do existing pods keep running?
   - How long does it take to detect the failure?

3. **What causes pods to be unschedulable?**
   - Resource exhaustion scenarios
   - Node affinity rules
   - Taints and tolerations
   - How do you debug scheduling issues?

4. **How do you control pod placement?**
   - Node affinity vs node selector
   - Taints and tolerations
   - Manual scheduling
   - Topology spread constraints

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Uber scheduler crash during traffic spike (2019)**
- **Pinterest resource request misconfiguration (2018)**
- **Kubernetes scheduling algorithm changes**

### Exercise 6.3: Advanced Scheduling

```bash
# Create complex scheduling scenario
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: complex-scheduling
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
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - complex-scheduling
        topologyKey: kubernetes.io/hostname
  tolerations:
  - key: special
    operator: Equal
    value: "true"
    effect: NoSchedule
  containers:
  - name: nginx
    image: nginx
EOF

# Check where it was scheduled
kubectl get pod complex-scheduling -o wide
```

---

## 🏆 Success Criteria

You've successfully completed Day 3 if you can:

✅ **Watch scheduler work in real-time**  
✅ **Break scheduler and understand the impact**  
✅ **Test resource exhaustion scenarios**  
✅ **Use node affinity and manual scheduling**  
✅ **Understand scheduling algorithms**  
✅ **Debug scheduling issues**  

## 🚨 Common Issues & Solutions

### Issue: Pods stuck in Pending
```bash
# Check pod events
kubectl describe pod <pod-name>

# Check node resources
kubectl describe node <node-name>

# Check scheduler logs
kubectl logs kube-scheduler-minikube -n kube-system
```

### Issue: Scheduler won't start
```bash
# Check scheduler manifest
minikube ssh
sudo cat /etc/kubernetes/manifests/kube-scheduler.yaml

# Check scheduler logs
sudo docker logs <scheduler-container-id>
```

### Issue: Pods not distributed evenly
```bash
# Check node labels
kubectl get nodes --show-labels

# Check pod affinity rules
kubectl describe pod <pod-name>
```

---

## 🎯 Tomorrow's Preview

**Day 4: Controller Manager - The Autopilot**
- Watch controllers work in real-time
- Break controller manager and see no self-healing
- Test deployment rollouts and rollbacks
- Practice node controller failure scenarios
- Understand the control loop

**Get ready for more chaos! 🔥**

---

## 📚 Additional Resources

- [Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Node Affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Pod Topology Spread](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/)

**Remember: The scheduler is like a Tetris master - it tries to fit pods perfectly on nodes! 🎮**
