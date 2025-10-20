# Day 4 Lab: Controller Manager - The Autopilot

> **"Today you'll break the autopilot and watch your cluster become dumb. You'll learn why controllers are the heart of Kubernetes self-healing."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch controllers work in real-time
- Break controller manager and see no self-healing
- Test deployment rollouts and rollbacks
- Practice node controller failure scenarios
- Understand the control loop
- Learn why controllers are critical for declarative operations

## ⚠️ Prerequisites

- Day 3 lab completed
- Minikube cluster running
- 4-5 hours of uninterrupted time
- Understanding of deployments and replicasets

## 🛠️ Setup (30 minutes)

### Step 1: Verify Cluster Status

```bash
# Check cluster is running
kubectl get nodes
kubectl get pods -n kube-system

# Clean up previous lab resources
kubectl delete deployment spread chaos-test test avoid
kubectl delete pod memory-hog cpu-hog realistic ssd-only impossible manual no-toleration with-toleration high-priority-pod complex-scheduling
```

### Step 2: Setup Monitoring

```bash
# Open 4 terminals
# Terminal 1: kubectl get pods --all-namespaces --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: kubectl get replicasets --watch
# Terminal 4: Your command terminal
```

### Step 3: Check Controller Manager

```bash
# Check controller manager is running
kubectl get pods -n kube-system | grep controller-manager

# Check controller manager logs
kubectl logs kube-controller-manager-minikube -n kube-system
```

---

## 🧪 Lab Part 1: Watch Controllers Work (45 minutes)

### Exercise 1.1: Create Deployment and Watch Controllers

```bash
# Create deployment
kubectl create deployment auto-heal --image=nginx --replicas=3

# Watch pods being created
kubectl get pods -l app=auto-heal --watch

# In another terminal, delete a pod
kubectl delete pod <pod-name>

# Watch in first terminal:
# Pod shows: Terminating
# New pod immediately appears: ContainerCreating
# New pod: Running

# ReplicaSet controller noticed: desired=3, current=2
# Created new pod to reconcile!
```

### Exercise 1.2: Check Controller's Work

```bash
# See ReplicaSet
kubectl get replicasets

# Describe it
kubectl describe replicaset <replicaset-name>
# Events show:
#   Created pod auto-heal-xxx-abc
#   Created pod auto-heal-xxx-def (after you deleted one)
```

### Exercise 1.3: Test Scaling

```bash
# Scale deployment
kubectl scale deployment auto-heal --replicas=5

# Watch pods being created
kubectl get pods -l app=auto-heal --watch

# Scale down
kubectl scale deployment auto-heal --replicas=2

# Watch pods being terminated
kubectl get pods -l app=auto-heal --watch
```

### Exercise 1.4: Test Image Updates

```bash
# Update image
kubectl set image deployment/auto-heal nginx=nginx:1.20

# Watch rollout
kubectl rollout status deployment/auto-heal

# See ReplicaSets
kubectl get replicasets
# You'll see:
# auto-heal-abc123   0    0    0    5m   (old, scaling down)
# auto-heal-def456   5    5    5    30s  (new, scaling up)
```

---

## 💀 Lab Part 2: Break #6 - Stop Controller Manager (1 hour)

> **"Time to break the autopilot. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test deployment
kubectl create deployment chaos-test --image=nginx --replicas=3

# Verify it's running
kubectl get pods -l app=chaos-test
```

### Exercise 2.2: Break Controller Manager

```bash
# Find controller manager pod
kubectl get pods -n kube-system | grep controller-manager

# Delete controller manager pod (it will restart)
kubectl delete pod -n kube-system kube-controller-manager-minikube

# Wait 10 seconds, then prevent it from restarting
minikube ssh
sudo mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
exit
```

### Exercise 2.3: Observe the Chaos

**Watch Terminal 1 (pods):**
- Existing pods keep running
- No new pods can be created
- No self-healing

**Test self-healing:**
```bash
# Delete a pod
kubectl delete pod <any-pod-from-your-deployment>

# Watch Terminal 1
# Pod shows: Terminating
# Pod shows: Terminated
# NO NEW POD APPEARS!

kubectl get pods
# You now have 2/3 pods instead of 3/3

# ReplicaSet controller is offline!
# No auto-healing!
```

**More chaos:**
```bash
# Try scaling
kubectl scale deployment chaos-test --replicas=5

kubectl get pods
# Still only 2 pods!
# Controller not running to create new ones!

# Try creating new deployment
kubectl create deployment broken --image=nginx --replicas=3

kubectl get pods -l app=broken
# NO PODS CREATED!
# Deployment controller not running!
```

### Exercise 2.4: Debug the Problem

```bash
# Check deployments
kubectl get deployments
# chaos-test   2/5     2            2           5m   <-- Not reconciling!
# broken       0/3     0            0           1m   <-- Never created pods!

# Check ReplicaSets
kubectl get replicasets
# All show desired vs current mismatch

# Check controller manager
kubectl get pods -n kube-system | grep controller-manager
# NOT FOUND!
```

**Key Learning:** 
- Controller Manager is critical for self-healing
- No controllers = no declarative operations
- Existing pods unaffected (kubelet still works)
- Cluster becomes "dumb" - can't maintain desired state

### Exercise 2.5: Fix It

```bash
minikube ssh
sudo mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/
exit

# Wait 30 seconds
# Watch Terminal 1 - pods start creating!

kubectl get pods
# chaos-test: 5/5 Running
# broken: 3/3 Running

# Everything reconciled!
```

**Real-world parallel:** This happens when:
- Controller Manager crashes during incident
- Memory leaks in controllers
- Configuration errors
- Resource exhaustion

---

## 🔥 Lab Part 3: Node Controller Failure Simulation (1 hour)

> **"Time to test what happens when nodes become unhealthy. This is where the node controller shines."**

### Exercise 3.1: Test Node Cordon

```bash
# Mark node as unschedulable
kubectl cordon minikube-m02

# Check nodes
kubectl get nodes
# minikube-m02   Ready,SchedulingDisabled

# Create pods - they avoid cordoned node
kubectl create deployment avoid --image=nginx --replicas=6
kubectl get pods -o wide
# All pods on minikube and minikube-m03, none on m02
```

### Exercise 3.2: Simulate Node Failure

```bash
# Stop kubelet on a node
minikube ssh -n minikube-m02
sudo systemctl stop kubelet
exit
```

**Watch Node Controller react:**
```bash
# Terminal 1: Watch nodes
kubectl get nodes --watch

# After ~40 seconds:
# minikube-m02   NotReady,SchedulingDisabled

# After 5 minutes (default timeout):
# Pods on that node marked for eviction

# Check pod status
kubectl get pods -o wide
# Pods on m02 show: Terminating or Unknown

# Node Controller creates new pods on healthy nodes
# (because Controller Manager sees desired vs actual state mismatch)
```

### Exercise 3.3: Debug Node Failure

```bash
# Check node condition
kubectl describe node minikube-m02

# Conditions:
#   Ready   Unknown   Kubelet stopped posting node status

# Events:
#   NodeNotReady  Node minikube-m02 status is now: NotReady
```

### Exercise 3.4: Fix Node Failure

```bash
minikube ssh -n minikube-m02
sudo systemctl start kubelet
exit

# Node recovers
kubectl get nodes
# minikube-m02   Ready,SchedulingDisabled

# Uncordon it
kubectl uncordon minikube-m02

# Now it can receive pods again
```

**Real-world parallel:** This happens when:
- EC2 instances fail
- Network partitions
- Hardware failures
- kubelet crashes

---

## 🧠 Lab Part 4: Deployment Rollout & Rollback (1 hour)

> **"Time to test the deployment controller. This is where you learn about rolling updates."**

### Exercise 4.1: Test Rollout

```bash
# Create deployment
kubectl create deployment rollout-test --image=nginx:1.19 --replicas=5

# Watch rollout
kubectl rollout status deployment/rollout-test

# See ReplicaSets
kubectl get replicasets
# rollout-test-abc123   5    5    5    30s  (current)
```

### Exercise 4.2: Update Image

```bash
# Update image
kubectl set image deployment/rollout-test nginx=nginx:1.20

# Watch in real-time
kubectl get replicasets --watch

# You see:
# rollout-test-abc123   5    5    5    2m   (old, scaling down)
# rollout-test-def456   1    1    1    5s   (new, scaling up)
# rollout-test-abc123   4    4    4    2m   (old, scaling down)
# rollout-test-def456   2    2    2    10s  (new, scaling up)
# ...continues until all pods are new
```

### Exercise 4.3: Test Rollback

```bash
# Check rollout history
kubectl rollout history deployment/rollout-test

# Rollback to previous version
kubectl rollout undo deployment/rollout-test

# Watch rollback
kubectl rollout status deployment/rollout-test

# See ReplicaSets again
kubectl get replicasets
# rollout-test-def456   5    5    5    2m   (new, scaling down)
# rollout-test-abc123   1    1    1    5s   (old, scaling up)
```

### Exercise 4.4: Test Rollout Pause

```bash
# Start new rollout
kubectl set image deployment/rollout-test nginx=nginx:1.21

# Pause rollout
kubectl rollout pause deployment/rollout-test

# Check status
kubectl get replicasets
# rollout-test-def456   3    3    3    2m   (old, scaling down)
# rollout-test-ghi789   2    2    2    30s  (new, scaling up)
# Rollout paused at 2/5

# Resume rollout
kubectl rollout resume deployment/rollout-test

# Watch completion
kubectl rollout status deployment/rollout-test
```

---

## 🎯 Lab Part 5: Advanced Controller Scenarios (45 minutes)

### Exercise 5.1: Test DaemonSet Controller

```bash
# Create DaemonSet
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: test-daemonset
spec:
  selector:
    matchLabels:
      app: test-daemonset
  template:
    metadata:
      labels:
        app: test-daemonset
    spec:
      containers:
      - name: nginx
        image: nginx
EOF

# Check DaemonSet
kubectl get daemonset
kubectl get pods -l app=test-daemonset

# Should have 3 pods (one per node)
```

### Exercise 5.2: Test StatefulSet Controller

```bash
# Create StatefulSet
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: test-statefulset
spec:
  serviceName: test-statefulset
  replicas: 3
  selector:
    matchLabels:
      app: test-statefulset
  template:
    metadata:
      labels:
        app: test-statefulset
    spec:
      containers:
      - name: nginx
        image: nginx
EOF

# Check StatefulSet
kubectl get statefulset
kubectl get pods -l app=test-statefulset

# Should have 3 pods with ordered names
```

### Exercise 5.3: Test Job Controller

```bash
# Create Job
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: test-job
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx
        command: ["sh", "-c", "echo 'Job completed' && sleep 10"]
      restartPolicy: Never
EOF

# Check Job
kubectl get job
kubectl get pods -l job-name=test-job

# Wait for completion
kubectl wait --for=condition=complete job/test-job

# Check status
kubectl get job test-job
# STATUS: 1/1
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-04-reflections.md` and answer:

1. **What does the Controller Manager do?**
   - Which controllers does it run?
   - How does the control loop work?
   - Why is it critical for self-healing?

2. **What happens when Controller Manager dies?**
   - Can you create new workloads?
   - Do existing pods keep running?
   - What breaks in the cluster?

3. **How do deployments work?**
   - What's the difference between Deployment and ReplicaSet?
   - How do rolling updates work?
   - How do rollbacks work?

4. **How does the Node Controller work?**
   - What happens when a node becomes unhealthy?
   - How long does it take to detect node failure?
   - What happens to pods on failed nodes?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Monzo bank Controller Manager failure (2019)**
- **Kubernetes deployment rollout failures**
- **Node failure handling in production**

### Exercise 6.3: Advanced Controller Scenarios

```bash
# Create complex deployment scenario
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: complex-deployment
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 2
  selector:
    matchLabels:
      app: complex-deployment
  template:
    metadata:
      labels:
        app: complex-deployment
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
EOF

# Test rolling update
kubectl set image deployment/complex-deployment nginx=nginx:1.20

# Watch the rollout
kubectl rollout status deployment/complex-deployment
```

---

## 🏆 Success Criteria

You've successfully completed Day 4 if you can:

✅ **Watch controllers work in real-time**  
✅ **Break controller manager and understand the impact**  
✅ **Test deployment rollouts and rollbacks**  
✅ **Simulate node failures and recovery**  
✅ **Understand the control loop**  
✅ **Debug controller issues**  

## 🚨 Common Issues & Solutions

### Issue: Controllers not working
```bash
# Check controller manager logs
kubectl logs kube-controller-manager-minikube -n kube-system

# Check controller manager pod
kubectl get pods -n kube-system | grep controller-manager
```

### Issue: Deployments not updating
```bash
# Check deployment status
kubectl describe deployment <deployment-name>

# Check ReplicaSets
kubectl get replicasets

# Check pod events
kubectl describe pod <pod-name>
```

### Issue: Nodes not recovering
```bash
# Check node status
kubectl get nodes

# Check node conditions
kubectl describe node <node-name>

# Check kubelet logs
minikube ssh -n <node-name>
sudo journalctl -u kubelet
```

---

## 🎯 Tomorrow's Preview

**Day 5: kubelet - The Node Agent**
- Watch kubelet work in real-time
- Break kubelet and see pods become unreachable
- Test container runtime failures
- Practice node maintenance procedures
- Understand the node agent architecture

**Get ready for more chaos! 🔥**

---

## 📚 Additional Resources

- [Kubernetes Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Deployment Controller](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Node Controller](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [Rolling Updates](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment)

**Remember: Controllers are the autopilot of Kubernetes - they keep everything running smoothly! 🤖**
