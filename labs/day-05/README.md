# Day 5 Lab: kubelet - The Node Agent

> **"Today you'll break the node agent and watch pods become unreachable. You'll learn why kubelet is the most critical component on every node."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch kubelet work in real-time
- Break kubelet and see pods become unreachable
- Test container runtime failures
- Practice node maintenance procedures
- Understand the node agent architecture
- Learn why kubelet is critical for pod lifecycle

## ⚠️ Prerequisites

- Day 4 lab completed
- Minikube cluster running
- 4-5 hours of uninterrupted time
- Understanding of pod lifecycle

## 🛠️ Setup (30 minutes)

### Step 1: Verify Cluster Status

```bash
# Check cluster is running
kubectl get nodes
kubectl get pods -n kube-system

# Clean up previous lab resources
kubectl delete deployment auto-heal chaos-test broken avoid rollout-test complex-deployment
kubectl delete daemonset test-daemonset
kubectl delete statefulset test-statefulset
kubectl delete job test-job
```

### Step 2: Setup Monitoring

```bash
# Open 4 terminals
# Terminal 1: kubectl get pods --all-namespaces --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: kubectl get nodes --watch
# Terminal 4: Your command terminal
```

### Step 3: Check kubelet Status

```bash
# Check kubelet is running on all nodes
kubectl get nodes
# All nodes should show: Ready

# Check kubelet logs
minikube ssh
sudo journalctl -u kubelet -f
# Exit with Ctrl+C
```

---

## 🧪 Lab Part 1: Watch kubelet Work (45 minutes)

### Exercise 1.1: Create Pods and Watch kubelet

```bash
# Create deployment
kubectl create deployment kubelet-test --image=nginx --replicas=3

# Watch pods being created
kubectl get pods -l app=kubelet-test --watch

# In another terminal, check kubelet logs
minikube ssh
sudo journalctl -u kubelet -f
# You'll see logs like:
# "Successfully assigned pod to node"
# "Pulling image nginx"
# "Successfully pulled image nginx"
# "Created container nginx"
# "Started container nginx"
```

### Exercise 1.2: Test Pod Lifecycle

```bash
# Delete a pod
kubectl delete pod <pod-name>

# Watch kubelet logs
# You'll see:
# "Stopping container nginx"
# "Stopped container nginx"
# "Removed container nginx"
# "Successfully assigned pod to node"
# "Pulling image nginx"
# "Successfully pulled image nginx"
# "Created container nginx"
# "Started container nginx"
```

### Exercise 1.3: Test Container Runtime Integration

```bash
# Check container runtime
minikube ssh
sudo systemctl status containerd
# Should show: Active: active (running)

# Check running containers
sudo docker ps
# You'll see your nginx containers

# Check container logs
sudo docker logs <container-id>
# You'll see nginx logs
```

### Exercise 1.4: Test Resource Management

```bash
# Create pod with resource limits
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
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
EOF

# Check pod status
kubectl get pod resource-test
# STATUS: Running

# Check resource usage
kubectl top pod resource-test
# If metrics not available, check node resources
kubectl describe node minikube | grep -A 5 "Allocated resources"
```

---

## 💀 Lab Part 2: Break #7 - Stop kubelet (1 hour)

> **"Time to break the node agent. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test deployment
kubectl create deployment chaos-test --image=nginx --replicas=3

# Verify it's running
kubectl get pods -l app=chaos-test
```

### Exercise 2.2: Break kubelet

```bash
# Stop kubelet on a node
minikube ssh -n minikube-m02
sudo systemctl stop kubelet

# Verify it's dead
sudo systemctl status kubelet
# Should show: Active: inactive (dead)
```

### Exercise 2.3: Observe the Chaos

**Watch Terminal 1 (pods):**
- Pods on the affected node show "Unknown" status
- After ~5 minutes, pods are evicted
- New pods are created on healthy nodes

**Watch Terminal 2 (events):**
- You'll see events like:
  - "NodeNotReady: Node minikube-m02 status is now: NotReady"
  - "FailedMount: Unable to mount volumes"
  - "FailedAttachVolume: Failed to attach volume"

**Watch Terminal 3 (nodes):**
- Node shows "NotReady" status
- After ~40 seconds: "NotReady,SchedulingDisabled"

### Exercise 2.4: Debug the Problem

```bash
# Check node status
kubectl get nodes
# minikube-m02   NotReady,SchedulingDisabled

# Check node conditions
kubectl describe node minikube-m02
# Conditions:
#   Ready   Unknown   Kubelet stopped posting node status

# Check pod status
kubectl get pods -o wide
# Pods on m02 show: Unknown or Terminating
```

**Key Learning:** 
- kubelet is critical for pod lifecycle
- No kubelet = no pod operations
- Node becomes "NotReady"
- Pods are evicted after timeout

### Exercise 2.5: Fix It

```bash
# Still in SSH on m02
sudo systemctl start kubelet
sudo systemctl status kubelet
# Should show: Active: active (running)

# Exit SSH
exit

# Node recovers
kubectl get nodes
# minikube-m02   Ready

# Pods on the node should recover
kubectl get pods -o wide
```

**Real-world parallel:** This happens when:
- kubelet crashes due to OOM
- System administrator stops kubelet
- Hardware failures
- Network partitions

---

## 🔥 Lab Part 3: Container Runtime Failures (1 hour)

> **"Time to test what happens when the container runtime fails. This is where kubelet's resilience is tested."**

### Exercise 3.1: Test Container Runtime Stop

```bash
# Stop containerd
minikube ssh
sudo systemctl stop containerd

# Verify it's dead
sudo systemctl status containerd
# Should show: Active: inactive (dead)
```

### Exercise 3.2: Observe kubelet's Response

**Watch Terminal 1 (pods):**
- Pods show "Unknown" status
- No new pods can be created
- Existing pods become unreachable

**Watch Terminal 2 (events):**
- You'll see events like:
  - "PLEG is not healthy: runtime network not ready"
  - "failed to get container stats: unable to connect to containerd"
  - "Failed to create pod sandbox"

### Exercise 3.3: Test Pod Creation

```bash
# Exit SSH
exit

# Try to create new pods
kubectl create deployment test --image=nginx
kubectl get pods
# Status: Pending (can't be scheduled because no container runtime)
```

### Exercise 3.4: Fix Container Runtime

```bash
# SSH back in
minikube ssh
sudo systemctl start containerd
sudo systemctl status containerd
# Should show: Active: active (running)

# Exit SSH
exit

# Pods should recover
kubectl get pods
# All pods: Running
```

**Real-world parallel:** This happens when:
- Container runtime crashes due to OOM
- Disk space issues
- Bug in container runtime
- Hardware failures

---

## 🧠 Lab Part 4: Node Maintenance Procedures (1 hour)

> **"Time to practice real node maintenance. This is where you learn to be a node administrator."**

### Exercise 4.1: Test Node Drain

```bash
# Create pods on specific node
kubectl create deployment node-test --image=nginx --replicas=6

# Check pod distribution
kubectl get pods -o wide
# Pods should be distributed across nodes

# Drain a node
kubectl drain minikube-m02 --ignore-daemonsets --delete-emptydir-data

# Check what happened
kubectl get pods -o wide
# Pods on m02 should be evicted and recreated on other nodes
```

### Exercise 4.2: Test Node Cordon

```bash
# Cordon a node
kubectl cordon minikube-m03

# Check node status
kubectl get nodes
# minikube-m03   Ready,SchedulingDisabled

# Create new pods
kubectl create deployment avoid --image=nginx --replicas=3
kubectl get pods -o wide
# New pods should avoid cordoned node
```

### Exercise 4.3: Test Node Uncordon

```bash
# Uncordon the node
kubectl uncordon minikube-m03

# Check node status
kubectl get nodes
# minikube-m03   Ready

# Now new pods can be scheduled on m03
```

### Exercise 4.4: Test Node Restart

```bash
# Restart a node
minikube ssh -n minikube-m02
sudo reboot
# Wait for node to come back up

# Check node status
kubectl get nodes
# Node should show: Ready

# Check pods
kubectl get pods -o wide
# Pods should be running on all nodes
```

---

## 🎯 Lab Part 5: Advanced kubelet Scenarios (45 minutes)

### Exercise 5.1: Test Pod Eviction

```bash
# Create pod with high resource usage
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
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "2Gi"
        cpu: "1000m"
EOF

# Check pod status
kubectl get pod memory-hog
# STATUS: Running
```

### Exercise 5.2: Test Pod Security

```bash
# Create pod with security context
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: security-test
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
EOF

# Check pod status
kubectl get pod security-test
# STATUS: Running
```

### Exercise 5.3: Test Pod Lifecycle Hooks

```bash
# Create pod with lifecycle hooks
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-test
spec:
  containers:
  - name: nginx
    image: nginx
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "echo 'Pod is stopping'"]
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo 'Pod is starting'"]
EOF

# Check pod status
kubectl get pod lifecycle-test
# STATUS: Running

# Check logs
kubectl logs lifecycle-test
# You should see: "Pod is starting"
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-05-reflections.md` and answer:

1. **What does kubelet do?**
   - How does it manage pod lifecycle?
   - What's its relationship with container runtime?
   - Why is it critical for node operations?

2. **What happens when kubelet dies?**
   - How long does it take to detect the failure?
   - What happens to pods on the node?
   - How does the cluster recover?

3. **What happens when container runtime fails?**
   - Can kubelet function without container runtime?
   - How does it handle container runtime failures?
   - What's the impact on pod operations?

4. **How do you maintain nodes?**
   - What's the process for node maintenance?
   - How do you safely restart nodes?
   - What are the risks of node operations?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Target kubelet memory leak (2018)**
- **Kubernetes node failure handling**
- **Container runtime failures in production**

### Exercise 6.3: Advanced kubelet Scenarios

```bash
# Create complex pod scenario
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: complex-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
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
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "echo 'Pod is stopping'"]
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo 'Pod is starting'"]
EOF

# Check pod status
kubectl get pod complex-pod
# STATUS: Running
```

---

## 🏆 Success Criteria

You've successfully completed Day 5 if you can:

✅ **Watch kubelet work in real-time**  
✅ **Break kubelet and understand the impact**  
✅ **Test container runtime failures**  
✅ **Practice node maintenance procedures**  
✅ **Understand the node agent architecture**  
✅ **Debug kubelet issues**  

## 🚨 Common Issues & Solutions

### Issue: kubelet won't start
```bash
# Check kubelet status
minikube ssh
sudo systemctl status kubelet

# Check kubelet logs
sudo journalctl -u kubelet
```

### Issue: Pods stuck in Unknown
```bash
# Check node status
kubectl get nodes

# Check node conditions
kubectl describe node <node-name>

# Check kubelet logs
minikube ssh -n <node-name>
sudo journalctl -u kubelet
```

### Issue: Container runtime failures
```bash
# Check container runtime status
minikube ssh
sudo systemctl status containerd

# Check container runtime logs
sudo journalctl -u containerd
```

---

## 🎯 Tomorrow's Preview

**Day 6: Networking - The Nervous System**
- Watch networking components work
- Break CNI and see pods lose connectivity
- Test service networking failures
- Practice network policy enforcement
- Understand the networking architecture

**Get ready for more chaos! 🔥**

---

## 📚 Additional Resources

- [kubelet Documentation](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)
- [Node Management](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Container Runtime Interface](https://kubernetes.io/docs/concepts/architecture/cri/)

**Remember: kubelet is the node agent - it's the most critical component on every node! 🖥️**
