# Day 10 Lab: Advanced Scheduling - The Tetris Master

> **"Today you'll break advanced scheduling and watch pods get stuck in complex scenarios. You'll learn why advanced scheduling is like playing Tetris with your infrastructure."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch advanced scheduling work in real-time
- Break advanced scheduling and see pods stuck
- Test node affinity and taints
- Practice advanced scheduling procedures
- Understand the advanced scheduling architecture
- Learn why advanced scheduling is critical for complex workloads

## ⚠️ Prerequisites

- Day 9 lab completed
- Minikube cluster running
- 4-5 hours of uninterrupted time
- Understanding of basic scheduling concepts

## 🛠️ Setup (30 minutes)

### Step 1: Verify Cluster Status

```bash
# Check cluster is running
kubectl get nodes
kubectl get pods -n kube-system

# Clean up previous lab resources
kubectl delete pod tracing-test metrics-test complex-observability
kubectl delete deployment web chaos-test discovery-test
kubectl delete service web chaos-test discovery-test
kubectl delete configmap monitoring-config
kubectl delete prometheusrule kubernetes-alerts
kubectl delete namespace observability-test
```

### Step 2: Setup Monitoring

```bash
# Open 4 terminals
# Terminal 1: kubectl get pods --all-namespaces --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: kubectl get nodes --watch
# Terminal 4: Your command terminal
```

### Step 3: Check Advanced Scheduling Components

```bash
# Check scheduler
kubectl get pods -n kube-system | grep scheduler

# Check node resources
kubectl describe node minikube
kubectl describe node minikube-m02
kubectl describe node minikube-m03
```

---

## 🧪 Lab Part 1: Watch Advanced Scheduling Work (45 minutes)

### Exercise 1.1: Test Node Affinity

```bash
# Create namespace for testing
kubectl create namespace scheduling-test

# Label nodes
kubectl label node minikube disktype=ssd
kubectl label node minikube-m02 disktype=hdd
kubectl label node minikube-m03 disktype=ssd

# Check node labels
kubectl get nodes --show-labels
```

### Exercise 1.2: Create Pod with Node Affinity

```bash
# Create pod with node affinity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
  namespace: scheduling-test
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

# Check where pod was scheduled
kubectl get pod ssd-pod -n scheduling-test -o wide
# NODE: Should be minikube or minikube-m03 (SSD nodes)
```

### Exercise 1.3: Test Pod Affinity

```bash
# Create pod with pod affinity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: affinity-pod
  namespace: scheduling-test
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - ssd-pod
        topologyKey: kubernetes.io/hostname
  containers:
  - name: nginx
    image: nginx
EOF

# Check where pod was scheduled
kubectl get pod affinity-pod -n scheduling-test -o wide
# NODE: Should be same as ssd-pod
```

### Exercise 1.4: Test Pod Anti-Affinity

```bash
# Create pod with pod anti-affinity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: anti-affinity-pod
  namespace: scheduling-test
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - ssd-pod
        topologyKey: kubernetes.io/hostname
  containers:
  - name: nginx
    image: nginx
EOF

# Check where pod was scheduled
kubectl get pod anti-affinity-pod -n scheduling-test -o wide
# NODE: Should be different from ssd-pod
```

---

## 💀 Lab Part 2: Break #12 - Stop Advanced Scheduling (1 hour)

> **"Time to break the advanced scheduling. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test deployment
kubectl create deployment chaos-test --image=nginx -n scheduling-test --replicas=3

# Create service
kubectl expose deployment chaos-test --port=80 -n scheduling-test

# Verify everything is working
kubectl get pods -l app=chaos-test -n scheduling-test
kubectl get svc chaos-test -n scheduling-test
```

### Exercise 2.2: Break Scheduler

```bash
# Delete scheduler pod
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
kubectl create deployment test --image=nginx -n scheduling-test --replicas=3

# Check status
kubectl get pods -n scheduling-test
# All new pods: Pending

# Check events
kubectl get events -n scheduling-test
# Should show: FailedScheduling
```

### Exercise 2.4: Debug the Problem

```bash
# Check scheduler
kubectl get pods -n kube-system | grep scheduler
# NOT FOUND!

# Check pending pods
kubectl describe pod <pod-name> -n scheduling-test
# Events: FailedScheduling

# Check node resources
kubectl describe node minikube
# Should show available resources
```

**Key Learning:** 
- Scheduler is critical for pod placement
- No scheduler = no new pods
- Existing pods unaffected
- Cluster can't scale or add new services

### Exercise 2.5: Fix It

```bash
minikube ssh
sudo mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/
exit

# Wait for scheduler to restart
kubectl get pods -n kube-system | grep scheduler
# Running!

# Check pending pods
kubectl get pods -n scheduling-test
# All pods: Running
```

**Real-world parallel:** This happens when:
- Scheduler crashes
- Resource exhaustion
- Configuration errors
- Network issues

---

## 🔥 Lab Part 3: Taints and Tolerations (1 hour)

> **"Time to test taints and tolerations. This is where you learn to control pod placement."**

### Exercise 3.1: Test Node Taints

```bash
# Taint a node
kubectl taint node minikube-m02 special=true:NoSchedule

# Check node taints
kubectl describe node minikube-m02
# Should show: Taints: special=true:NoSchedule
```

### Exercise 3.2: Test Pod Without Toleration

```bash
# Create pod without toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration-pod
  namespace: scheduling-test
spec:
  containers:
  - name: nginx
    image: nginx
EOF

# Check where pod was scheduled
kubectl get pod no-toleration-pod -n scheduling-test -o wide
# NODE: Should be minikube or minikube-m03 (not m02)
```

### Exercise 3.3: Test Pod With Toleration

```bash
# Create pod with toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: toleration-pod
  namespace: scheduling-test
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

# Check where pod was scheduled
kubectl get pod toleration-pod -n scheduling-test -o wide
# NODE: Should be minikube-m02 (the tainted node)
```

### Exercise 3.4: Test Multiple Taints

```bash
# Add another taint
kubectl taint node minikube-m03 gpu=true:NoSchedule

# Check node taints
kubectl describe node minikube-m03
# Should show: Taints: gpu=true:NoSchedule

# Create pod with multiple tolerations
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-toleration-pod
  namespace: scheduling-test
spec:
  tolerations:
  - key: special
    operator: Equal
    value: "true"
    effect: NoSchedule
  - key: gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  containers:
  - name: nginx
    image: nginx
EOF

# Check where pod was scheduled
kubectl get pod multi-toleration-pod -n scheduling-test -o wide
# NODE: Should be minikube-m02 or minikube-m03
```

---

## 🧠 Lab Part 4: Advanced Scheduling Scenarios (1 hour)

> **"Time to test complex scheduling scenarios. This is where you become a scheduling expert."**

### Exercise 4.1: Test Resource Requests

```bash
# Create pod with high resource requests
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: high-resource-pod
  namespace: scheduling-test
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "2Gi"
        cpu: "1000m"
EOF

# Check where pod was scheduled
kubectl get pod high-resource-pod -n scheduling-test -o wide
# NODE: Should be on node with available resources
```

### Exercise 4.2: Test Node Selector

```bash
# Create pod with node selector
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: node-selector-pod
  namespace: scheduling-test
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: nginx
    image: nginx
EOF

# Check where pod was scheduled
kubectl get pod node-selector-pod -n scheduling-test -o wide
# NODE: Should be on SSD node
```

### Exercise 4.3: Test Topology Spread

```bash
# Create deployment with topology spread
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-test
  namespace: scheduling-test
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
kubectl get pods -l app=spread-test -n scheduling-test -o wide
# Should be evenly distributed across nodes
```

### Exercise 4.4: Test Priority Classes

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
  name: priority-pod
  namespace: scheduling-test
spec:
  priorityClassName: high-priority
  containers:
  - name: nginx
    image: nginx
EOF

# Check pod status
kubectl get pod priority-pod -n scheduling-test
# STATUS: Running
```

---

## 🎯 Lab Part 5: Advanced Scheduling Debugging (45 minutes)

### Exercise 5.1: Test Scheduling Failures

```bash
# Create pod with impossible requirements
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: impossible-pod
  namespace: scheduling-test
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

# Check pod status
kubectl get pod impossible-pod -n scheduling-test
# STATUS: Pending

# Check events
kubectl describe pod impossible-pod -n scheduling-test
# Events: 0/3 nodes available: 3 node(s) didn't match node affinity rules
```

### Exercise 5.2: Test Resource Exhaustion

```bash
# Create pod requesting more resources than available
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-exhaustion-pod
  namespace: scheduling-test
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "10Gi"  # More than any node has!
        cpu: "10"
EOF

# Check pod status
kubectl get pod resource-exhaustion-pod -n scheduling-test
# STATUS: Pending

# Check events
kubectl describe pod resource-exhaustion-pod -n scheduling-test
# Events: 0/3 nodes available: 3 Insufficient memory
```

### Exercise 5.3: Test Scheduling Debugging

```bash
# Check scheduler logs
kubectl logs kube-scheduler-minikube -n kube-system

# Check node resources
kubectl describe node minikube
kubectl describe node minikube-m02
kubectl describe node minikube-m03

# Check pod events
kubectl get events -n scheduling-test
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-10-reflections.md` and answer:

1. **What does advanced scheduling do?**
   - How does node affinity work?
   - What happens when it fails?
   - Why is it critical for complex workloads?

2. **What happens when scheduling fails?**
   - How do you detect scheduling failures?
   - What happens to pods?
   - How do you recover from scheduling failures?

3. **How do you use taints and tolerations?**
   - What's the difference between taints and tolerations?
   - How do you control pod placement?
   - What are the best practices?

4. **How do you debug scheduling issues?**
   - What tools do you use?
   - How do you check scheduling status?
   - How do you test scheduling?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Kubernetes scheduling failures**
- **Advanced scheduling best practices**
- **Scheduling algorithm changes**
- **Resource exhaustion scenarios**

### Exercise 6.3: Advanced Scheduling Scenarios

```bash
# Create complex scheduling scenario
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: complex-scheduling-pod
  namespace: scheduling-test
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
            - complex-scheduling-pod
        topologyKey: kubernetes.io/hostname
  tolerations:
  - key: special
    operator: Equal
    value: "true"
    effect: NoSchedule
  priorityClassName: high-priority
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
EOF

# Check pod status
kubectl get pod complex-scheduling-pod -n scheduling-test
# STATUS: Running
```

---

## 🏆 Success Criteria

You've successfully completed Day 10 if you can:

✅ **Watch advanced scheduling work in real-time**  
✅ **Break advanced scheduling and understand the impact**  
✅ **Test node affinity and taints**  
✅ **Practice advanced scheduling procedures**  
✅ **Understand the advanced scheduling architecture**  
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

### Issue: Scheduling not working
```bash
# Check scheduler status
kubectl get pods -n kube-system | grep scheduler

# Check node taints
kubectl describe node <node-name>

# Check pod tolerations
kubectl describe pod <pod-name>
```

### Issue: Pods not distributed evenly
```bash
# Check topology spread constraints
kubectl describe pod <pod-name>

# Check node labels
kubectl get nodes --show-labels

# Check pod affinity rules
kubectl describe pod <pod-name>
```

---

## 🎯 Tomorrow's Preview

**Day 11: StatefulSets - The Memory System**
- Watch StatefulSets work
- Break StatefulSets and see pods lose state
- Test persistent volume failures
- Practice StatefulSet procedures
- Understand the StatefulSet architecture

**Get ready for more chaos! 🔥**

---

## 📚 Additional Resources

- [Kubernetes Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Node Affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Pod Topology Spread](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/)

**Remember: Advanced scheduling is like playing Tetris with your infrastructure - it tries to fit pods perfectly! 🎮**
