# Day 1 Lab: Foundation & First Blood

> **"Welcome to your first day of Kubernetes hell. Today you'll break your cluster and fix it with your bare hands."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Set up a local Kubernetes cluster
- Deploy your first application
- Intentionally break the container runtime
- Destroy kube-proxy networking
- Understand what happens when components fail
- Fix everything and learn from the chaos

## ⚠️ Prerequisites

- 8GB+ RAM available
- Docker installed
- Terminal with tmux/screen (recommended)
- 3-4 hours of uninterrupted time
- Strong coffee ☕

## 🛠️ Setup (30 minutes)

### Step 1: Install Tools

```bash
# First, check your system architecture on Mac
uname -m
# Should show: x86_64, arm64, or aarch64

# Install minikube (choose the right version for your architecture)
# For macOS x86_64:
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube

# For macOS ARM64 (Apple Silicon):
# curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-arm64
# sudo install minikube-darwin-arm64 /usr/local/bin/minikube

# Install kubectl (choose the right version for your architecture)
# For macOS x86_64:
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl

# For macOS ARM64 (Apple Silicon):
# curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"
# sudo install kubectl /usr/local/bin/kubectl

# Verify installations
kubectl version --client
minikube version
```

### Step 2: Start Your Cluster

```bash
# Start minikube with sufficient resources
minikube start --cpus=4 --memory=4096 --driver=docker

# Verify cluster is running
kubectl get nodes
kubectl get pods -n kube-system
```

**📖 Need Installation Help?**
If you haven't installed minikube yet, visit the [official minikube documentation](https://minikube.sigs.k8s.io/docs/start/) for platform-specific installation instructions.

**Expected Output:**
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.28.3

NAMESPACE     NAME                               READY   STATUS
kube-system   coredns-5d78c9869d-abcde           1/1     Running
kube-system   etcd-minikube                      1/1     Running
kube-system   kube-apiserver-minikube            1/1     Running
kube-system   kube-controller-manager-minikube   1/1     Running
kube-system   kube-proxy-xyz                     1/1     Running
kube-system   kube-scheduler-minikube            1/1     Running
```

### Step 3: Setup Monitoring

```bash
# Open 3 terminal windows/tabs
# Terminal 1: Watch pods
kubectl get pods --watch

# Terminal 2: Watch events
kubectl get events --watch --sort-by='.lastTimestamp'

# Terminal 3: Your command terminal
# (Keep this for running commands)
```

---

## 🧪 Lab Part 1: Deploy Your First App (30 minutes)

### Exercise 1.1: Basic Deployment

```bash
# Create a deployment
kubectl create deployment web --image=nginx --replicas=3

# Check what happened
kubectl get deployments
kubectl get pods
kubectl get replicasets
```

**What you should see:**
- 1 Deployment created
- 1 ReplicaSet created  
- 3 Pods running

### Exercise 1.2: Expose Your App

```bash
# Create a service
kubectl expose deployment web --port=80 --type=NodePort

# Check the service
kubectl get services

# Use port-forward to access your app (works with any driver)
kubectl port-forward service/web 8080:80
```

**Test your app:**
```bash
# In another terminal, test your app
curl http://localhost:8080
# You should see nginx welcome page HTML

# Or open in browser: http://localhost:8080
```

### Exercise 1.3: Scale Your App

```bash
# Scale up to 5 replicas
kubectl scale deployment web --replicas=5

# Watch the pods being created
kubectl get pods --watch

# Scale back down
kubectl scale deployment web --replicas=2
```

**Questions to answer:**
1. How long does it take for new pods to be created?
2. What happens to the old pods when you scale down?
3. Can you still access your app during scaling?

---

## 💀 Lab Part 2: Break #1 - Kill Container Runtime (1 hour)

> **"Time to break things. This is where the real learning begins."**

### Exercise 2.1: Prepare for Chaos

```bash
# Make sure you have 3 terminals open
# Terminal 1: kubectl get pods --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: Your command terminal

# Create a test deployment
kubectl create deployment chaos-test --image=nginx --replicas=3

# Verify it's running
kubectl get pods -l app=chaos-test
```

### Exercise 2.2: Break the Container Runtime

```bash
# SSH into the minikube node
minikube ssh

# Check containerd status
sudo systemctl status containerd
# Should show: Active: active (running)

# BREAK IT!
sudo systemctl stop containerd

# Verify it's dead
sudo systemctl status containerd
# Should show: Active: inactive (dead)
```

### Exercise 2.3: Observe the Chaos

**Watch Terminal 1 (pods):**
- Pods will show "Unknown" status after ~30 seconds
- No new pods can be created
- Existing pods become unreachable

**Watch Terminal 2 (events):**
- You'll see errors like:
  - "PLEG is not healthy: runtime network not ready"
  - "failed to get container stats: unable to connect to containerd"

**Try to create new pods:**
```bash
# Exit SSH first
exit

# Try to create a new pod
kubectl create deployment test --image=nginx
kubectl get pods
# Status: Pending (can't be scheduled because no container runtime)
```

### Exercise 2.4: Debug the Problem

```bash
# SSH back in
minikube ssh

# Check kubelet logs
sudo journalctl -u kubelet -f

# You'll see errors like:
# "failed to get container stats: unable to connect to containerd"
# "PLEG is not healthy: runtime network not ready"
```

**Key Learning:** kubelet depends on container runtime. No runtime = no container operations.

### Exercise 2.5: Fix It

```bash
# Still in SSH
sudo systemctl start containerd
sudo systemctl status containerd
# Should show: Active: active (running)

# Exit SSH
exit

# Watch Terminal 1 - pods should recover
# They go: Unknown → Terminating → ContainerCreating → Running
```

**Real-world parallel:** This happens when:
- Container runtime crashes due to OOM
- Disk space issues
- Bug in container runtime
- Hardware failures

---

## 🔥 Lab Part 3: Break #2 - Destroy kube-proxy (1 hour)

> **"Now let's break networking. This is where things get really interesting."**

### Exercise 3.1: Test Service Connectivity

```bash
# Create a test pod to access your service
kubectl create deployment test --image=busybox -- sleep 3600

# Wait for pod to be ready
kubectl wait --for=condition=ready pod -l app=test --timeout=60s

# Exec into the pod
kubectl exec -it deployment/test -- sh

# Inside the pod, test service connectivity
wget -O- http://web
# Should work and show nginx HTML

# Exit the test pod
exit

# Clean up
kubectl delete deployment test
```

### Exercise 3.2: Break kube-proxy

```bash
# Delete kube-proxy pod (it will restart)
kubectl delete pod -n kube-system -l k8s-app=kube-proxy

# Wait 30 seconds for it to restart
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Now SSH in and destroy iptables rules
minikube ssh

# Backup current iptables rules
sudo iptables-save > /tmp/iptables-backup.txt

# DESTROY ALL RULES
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -t mangle -F

# Verify rules are gone
sudo iptables -t nat -L | grep KUBE
# Should be empty!
```

### Exercise 3.3: Test the Breakage

```bash
# Exit SSH
exit

# Try to access your service again
kubectl create deployment test --image=busybox -- sleep 3600
kubectl wait --for=condition=ready pod -l app=test --timeout=60s
kubectl exec -it deployment/test -- sh

# Inside the pod
wget -O- http://web
# HANGS FOREVER - no response!

# Why? No iptables rules = no service routing
```

### Exercise 3.4: Debug the Problem

```bash
# Check if service still exists
kubectl get svc web
# NAME   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)
# web    NodePort   10.96.123.45    <none>        80:31234/TCP

# Check if pods are healthy
kubectl get pods
# All Running

# The problem is networking layer!
```

**Key Learning:** Services are just iptables rules. No kube-proxy = no service networking.

### Exercise 3.5: Fix It

```bash
# Option 1: Restore iptables rules
minikube ssh
sudo iptables-restore < /tmp/iptables-backup.txt
exit

# Option 2: Force kube-proxy to recreate rules
kubectl delete pod -n kube-system -l k8s-app=kube-proxy

# Wait 30 seconds, then test
kubectl create deployment test --image=busybox -- sleep 3600
kubectl wait --for=condition=ready pod -l app=test --timeout=60s
kubectl exec -it deployment/test -- sh
wget -O- http://web
# WORKS AGAIN!
exit
kubectl delete deployment test
```

**Real-world parallel:** This happens when:
- kube-proxy memory leak causes OOM
- iptables rules get corrupted
- Network policy conflicts
- System administrator accidentally clears iptables

---

## 🧠 Lab Part 4: Understanding What You Broke (30 minutes)

### Exercise 4.1: Analyze the Components

```bash
# Check all system pods
kubectl get pods -n kube-system

# Describe each component
kubectl describe pod etcd-minikube -n kube-system
kubectl describe pod kube-apiserver-minikube -n kube-system
kubectl describe pod kube-scheduler-minikube -n kube-system
kubectl describe pod kube-controller-manager-minikube -n kube-system
```

### Exercise 4.2: Check Logs

```bash
# Check logs for each component
kubectl logs etcd-minikube -n kube-system
kubectl logs kube-apiserver-minikube -n kube-system
kubectl logs kube-scheduler-minikube -n kube-system
kubectl logs kube-controller-manager-minikube -n kube-system
```

### Exercise 4.3: Understand the Architecture

```bash
# Check node information
kubectl describe node minikube

# Look at the conditions
# Ready: True
# MemoryPressure: False
# DiskPressure: False
# PIDPressure: False
```

---

## 🎯 Lab Part 5: Advanced Breakage (30 minutes)

### Exercise 5.1: Break CoreDNS

```bash
# Delete CoreDNS
kubectl delete daemonset coredns -n kube-system

# Try DNS lookup
kubectl create deployment test --image=busybox -- sleep 3600
kubectl wait --for=condition=ready pod -l app=test --timeout=60s
kubectl exec -it deployment/test -- nslookup kubernetes
# What breaks? Why?
kubectl delete deployment test

# Fix it - CoreDNS is managed by the cluster
# If deleted, restart the cluster or use:
kubectl rollout restart daemonset/coredns -n kube-system
# Or restart minikube:
# minikube stop && minikube start
```

### Exercise 5.2: Resource Exhaustion

```bash
# Create a pod that requests more memory than available
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
        memory: "10Gi"  # More than node has!
        cpu: "4"
EOF

# Check status
kubectl get pod memory-hog
kubectl describe pod memory-hog
# What happens? Why?
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-01-reflections.md` and answer:

1. **What happens when containerd dies?**
   - How long does it take for pods to show "Unknown"?
   - Can you create new pods?
   - Do existing pods keep running?

2. **What creates iptables rules?**
   - What component manages service networking?
   - What happens when kube-proxy is down?
   - How do you fix broken service networking?

3. **What is the role of each component?**
   - API Server: What does it do?
   - etcd: What does it store?
   - Scheduler: When does it run?
   - Controller Manager: What does it control?
   - kubelet: What does it manage?
   - kube-proxy: What does it create?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Container runtime failures in production** (common causes: OOM, disk space, bugs)
- **kube-proxy networking issues** (memory leaks, iptables corruption)
- **API server configuration problems** (upgrade failures, invalid flags)

### Exercise 6.3: Prepare for Tomorrow

```bash
# Clean up your cluster
kubectl delete deployment web chaos-test
kubectl delete service web

# But keep the cluster running for tomorrow
minikube status
```

---

## 🏆 Success Criteria

You've successfully completed Day 1 if you can:

✅ **Deploy and access applications**  
✅ **Intentionally break container runtime and fix it**  
✅ **Destroy networking and restore it**  
✅ **Explain what each component does**  
✅ **Debug issues using logs and events**  
✅ **Understand the impact of component failures**  

## 🚨 Common Issues & Solutions

### Issue: Minikube won't start
```bash
# Check Docker is running
docker ps

# Delete and recreate
minikube delete
minikube start --cpus=4 --memory=4096
```

### Issue: Can't access services
```bash
# Check if service exists
kubectl get svc

# Check if pods are running
kubectl get pods

# Check iptables rules
minikube ssh
sudo iptables -t nat -L | grep KUBE
```

### Issue: Pods stuck in Pending
```bash
# Check node resources
kubectl describe node minikube

# Check pod events
kubectl describe pod <pod-name>
```

---

## 🎯 Tomorrow's Preview

**Day 2: API Server & etcd - The Brain & Memory**
- Explore etcd directly
- Break etcd and see what happens
- Corrupt API server configuration
- Practice etcd backup and restore
- Understand the control plane architecture

**Get ready for more chaos! 🔥**

---

## 📚 Additional Resources

- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Minikube Documentation](https://minikube.sigs.k8s.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Container Runtime Interface](https://kubernetes.io/docs/concepts/architecture/cri/)

**Remember: The best way to learn Kubernetes is to break it, fix it, and break it again! 💪**
