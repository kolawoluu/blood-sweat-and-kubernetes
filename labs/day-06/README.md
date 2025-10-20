# Day 6 Lab: Networking - The Nervous System

> **"Today you'll break the networking and watch pods lose connectivity. You'll learn why networking is the nervous system of Kubernetes."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch networking components work in real-time
- Break CNI and see pods lose connectivity
- Test service networking failures
- Practice network policy enforcement
- Understand the networking architecture
- Learn why networking is critical for pod communication

## ⚠️ Prerequisites

- Day 5 lab completed
- Minikube cluster running
- 4-5 hours of uninterrupted time
- Understanding of networking concepts

## 🛠️ Setup (30 minutes)

### Step 1: Verify Cluster Status

```bash
# Check cluster is running
kubectl get nodes
kubectl get pods -n kube-system

# Clean up previous lab resources
kubectl delete deployment kubelet-test chaos-test node-test avoid
kubectl delete pod resource-test memory-hog security-test lifecycle-test complex-pod
```

### Step 2: Setup Monitoring

```bash
# Open 4 terminals
# Terminal 1: kubectl get pods --all-namespaces --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: kubectl get services --watch
# Terminal 4: Your command terminal
```

### Step 3: Check Networking Components

```bash
# Check CNI plugin
kubectl get pods -n kube-system | grep -E "(cni|flannel|calico|weave)"

# Check kube-proxy
kubectl get pods -n kube-system | grep kube-proxy

# Check CoreDNS
kubectl get pods -n kube-system | grep coredns
```

---

## 🧪 Lab Part 1: Watch Networking Work (45 minutes)

### Exercise 1.1: Create Services and Watch Networking

```bash
# Create deployment
kubectl create deployment web --image=nginx --replicas=3

# Create service
kubectl expose deployment web --port=80 --type=ClusterIP

# Check service
kubectl get svc web
# NAME   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
# web    ClusterIP   10.96.123.45    <none>        80/TCP
```

### Exercise 1.2: Test Service Connectivity

```bash
# Create test pod
kubectl create deployment test --image=busybox --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=ready pod -l app=test --timeout=60s
kubectl exec -it deployment/test -- sh

# Inside the pod, test service connectivity
nslookup web
# Should resolve to service IP

wget -O- http://web
# Should return nginx HTML

# Exit the test pod
exit
kubectl delete deployment test
```

### Exercise 1.3: Check iptables Rules

```bash
# SSH into node
minikube ssh

# Check iptables rules created by kube-proxy
sudo iptables -t nat -L | grep KUBE
# You'll see rules like:
# KUBE-SERVICES  all  --  anywhere             anywhere
# KUBE-SVC-xxx   tcp  --  anywhere             10.96.123.45        tcp dpt:http

# Check specific service rules
sudo iptables -t nat -L KUBE-SVC-xxx
# Shows load balancing rules to pod IPs
```

### Exercise 1.4: Test Pod-to-Pod Communication

```bash
# Create another deployment
kubectl create deployment backend --image=nginx --replicas=2

# Get pod IPs
kubectl get pods -o wide
# Note the pod IPs

# Test pod-to-pod communication
kubectl create deployment test --image=busybox --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=ready pod -l app=test --timeout=60s
kubectl exec -it deployment/test -- sh

# Inside the pod, test direct pod IP
wget -O- http://<pod-ip>
# Should work (direct pod communication)

# Exit the test pod
exit
kubectl delete deployment test
```

---

## 💀 Lab Part 2: Break #8 - Stop CNI Plugin (1 hour)

> **"Time to break the networking. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test deployment
kubectl create deployment chaos-test --image=nginx --replicas=3

# Create service
kubectl expose deployment chaos-test --port=80 --type=ClusterIP

# Verify everything is working
kubectl get pods -l app=chaos-test
kubectl get svc chaos-test
```

### Exercise 2.2: Break CNI Plugin

```bash
# Find CNI plugin pod
kubectl get pods -n kube-system | grep -E "(cni|flannel|calico|weave)"

# Delete CNI plugin pod
kubectl delete pod -n kube-system <cni-pod-name>

# Wait 30 seconds for it to restart
kubectl get pods -n kube-system | grep -E "(cni|flannel|calico|weave)"
```

### Exercise 2.3: Observe the Chaos

**Watch Terminal 1 (pods):**
- Pods may show "NotReady" status
- Network connectivity issues

**Test connectivity:**
```bash
# Try to access service
kubectl create deployment test --image=busybox --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=ready pod -l app=test --timeout=60s
kubectl exec -it deployment/test -- sh

# Inside the pod
nslookup chaos-test
# May fail or hang

wget -O- http://chaos-test
# May fail or hang

# Exit the test pod
exit
kubectl delete deployment test
```

### Exercise 2.4: Debug the Problem

```bash
# Check pod status
kubectl get pods -l app=chaos-test

# Check pod events
kubectl describe pod <pod-name>
# May show network-related errors

# Check node status
kubectl get nodes
# Nodes may show "NotReady"
```

**Key Learning:** 
- CNI plugin is critical for pod networking
- No CNI = no pod-to-pod communication
- Services may not work
- Pods may become unreachable

### Exercise 2.5: Fix It

```bash
# CNI plugin should restart automatically
kubectl get pods -n kube-system | grep -E "(cni|flannel|calico|weave)"

# Wait for it to be ready
kubectl wait --for=condition=ready pod -n kube-system -l k8s-app=flannel

# Test connectivity again
kubectl create deployment test --image=busybox --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=ready pod -l app=test --timeout=60s
kubectl exec -it deployment/test -- sh
wget -O- http://chaos-test
# Should work again
exit
kubectl delete deployment test
```

**Real-world parallel:** This happens when:
- CNI plugin crashes
- Network configuration errors
- Resource exhaustion
- Bug in CNI plugin

---

## 🔥 Lab Part 3: Service Networking Failures (1 hour)

> **"Time to test what happens when service networking breaks. This is where kube-proxy's resilience is tested."**

### Exercise 3.1: Test Service Networking

```bash
# Create service
kubectl create service clusterip test-svc --tcp=80:80

# Create endpoints manually
kubectl create deployment test-backend --image=nginx --replicas=2

# Check service and endpoints
kubectl get svc test-svc
kubectl get endpoints test-svc
```

### Exercise 3.2: Break kube-proxy

```bash
# Delete kube-proxy pod
kubectl delete pod -n kube-system -l k8s-app=kube-proxy

# Wait for it to restart
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# SSH into node and destroy iptables rules
minikube ssh
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -t mangle -F
```

### Exercise 3.3: Test Service Breakage

```bash
# Exit SSH
exit

# Try to access service
kubectl create deployment test --image=busybox --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=ready pod -l app=test --timeout=60s
kubectl exec -it deployment/test -- sh

# Inside the pod
wget -O- http://test-svc
# HANGS FOREVER - no response!

# Why? No iptables rules = no service routing
exit
kubectl delete deployment test
```

### Exercise 3.4: Fix Service Networking

```bash
# Force kube-proxy to recreate rules
kubectl delete pod -n kube-system -l k8s-app=kube-proxy

# Wait for it to restart
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Test connectivity again
kubectl create deployment test --image=busybox --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=ready pod -l app=test --timeout=60s
kubectl exec -it deployment/test -- sh
wget -O- http://test-svc
# Should work again
exit
kubectl delete deployment test
```

**Real-world parallel:** This happens when:
- kube-proxy crashes
- iptables rules get corrupted
- Network policy conflicts
- System administrator accidentally clears iptables

---

## 🧠 Lab Part 4: Network Policy Enforcement (1 hour)

> **"Time to test network policies. This is where you learn to secure your cluster networking."**

### Exercise 4.1: Create Network Policy

```bash
# Create namespace for testing
kubectl create namespace network-test

# Create deployments
kubectl create deployment web --image=nginx -n network-test --replicas=2
kubectl create deployment backend --image=nginx -n network-test --replicas=2

# Create services
kubectl expose deployment web --port=80 -n network-test
kubectl expose deployment backend --port=80 -n network-test
```

### Exercise 4.2: Test Without Network Policy

```bash
# Test connectivity between pods
kubectl create deployment test --image=busybox -n network-test --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=ready pod -l app=test -n network-test --timeout=60s
kubectl exec -it deployment/test -n network-test -- sh

# Inside the pod
wget -O- http://web
# Should work

wget -O- http://backend
# Should work

# Exit the test pod
exit
```

### Exercise 4.3: Apply Network Policy

```bash
# Create network policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: network-test
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# Test connectivity again
kubectl create deployment test --image=busybox -n network-test --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=ready pod -l app=test -n network-test --timeout=60s
kubectl exec -it deployment/test -n network-test -- sh

# Inside the pod
wget -O- http://web
# Should fail - network policy blocks it

wget -O- http://backend
# Should fail - network policy blocks it

# Exit the test pod
exit
kubectl delete deployment test -n network-test
```

### Exercise 4.4: Create Specific Network Policy

```bash
# Create more specific network policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-backend
  namespace: network-test
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 80
EOF

# Test connectivity
kubectl create deployment test --image=busybox -n network-test --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=ready pod -l app=test -n network-test --timeout=60s
kubectl exec -it deployment/test -n network-test -- sh

# Inside the pod
wget -O- http://web
# Should work

wget -O- http://backend
# Should work

# Exit the test pod
exit
```

---

## 🎯 Lab Part 5: Advanced Networking Scenarios (45 minutes)

### Exercise 5.1: Test DNS Resolution

```bash
# Test DNS resolution
kubectl run test --image=busybox -it --rm -- sh

# Inside the pod
nslookup kubernetes
# Should resolve to service IP

nslookup web.network-test.svc.cluster.local
# Should resolve to service IP

# Exit the test pod
exit
```

### Exercise 5.2: Test Load Balancing

```bash
# Create multiple replicas
kubectl scale deployment web -n network-test --replicas=5

# Test load balancing
kubectl run test --image=busybox -it --rm -- sh

# Inside the pod, make multiple requests
for i in {1..10}; do
  wget -O- http://web.network-test.svc.cluster.local
  echo "Request $i"
done

# Exit the test pod
exit
```

### Exercise 5.3: Test Service Types

```bash
# Create NodePort service
kubectl expose deployment web --port=80 --type=NodePort -n network-test

# Get NodePort
kubectl get svc web -n network-test
# Note the NodePort

# Test external access
minikube service web -n network-test --url
# Should return URL for external access
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-06-reflections.md` and answer:

1. **What does CNI plugin do?**
   - How does it enable pod-to-pod communication?
   - What happens when it fails?
   - Why is it critical for networking?

2. **What does kube-proxy do?**
   - How does it create service networking?
   - What iptables rules does it create?
   - What happens when it fails?

3. **How do network policies work?**
   - What's the difference between ingress and egress?
   - How do you debug network policy issues?
   - What are the security implications?

4. **How do you debug networking issues?**
   - What tools do you use?
   - How do you check iptables rules?
   - How do you test connectivity?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Reddit kube-proxy outage (2020)**
- **Kubernetes networking failures**
- **CNI plugin issues in production**

### Exercise 6.3: Advanced Networking Scenarios

```bash
# Create complex networking scenario
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: complex-policy
  namespace: network-test
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: allowed-namespace
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 80
EOF

# Test the policy
kubectl run test --image=busybox -n network-test -it --rm -- sh
wget -O- http://web
# Should work or fail depending on policy
```

---

## 🏆 Success Criteria

You've successfully completed Day 6 if you can:

✅ **Watch networking components work in real-time**  
✅ **Break CNI and understand the impact**  
✅ **Test service networking failures**  
✅ **Practice network policy enforcement**  
✅ **Understand the networking architecture**  
✅ **Debug networking issues**  

## 🚨 Common Issues & Solutions

### Issue: Pods can't communicate
```bash
# Check CNI plugin status
kubectl get pods -n kube-system | grep -E "(cni|flannel|calico|weave)"

# Check pod network
kubectl get pods -o wide
# Check if pods have IP addresses
```

### Issue: Services not working
```bash
# Check kube-proxy status
kubectl get pods -n kube-system | grep kube-proxy

# Check iptables rules
minikube ssh
sudo iptables -t nat -L | grep KUBE
```

### Issue: DNS not working
```bash
# Check CoreDNS status
kubectl get pods -n kube-system | grep coredns

# Test DNS resolution
kubectl create deployment test --image=busybox --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=ready pod -l app=test --timeout=60s
kubectl exec -it deployment/test -- nslookup kubernetes
kubectl delete deployment test
```

---

## 🎯 Tomorrow's Preview

**Day 7: Storage - The Memory System**
- Watch storage components work
- Break storage and see pods lose data
- Test persistent volume failures
- Practice storage backup and recovery
- Understand the storage architecture

**Get ready for more chaos! 🔥**

---

## 📚 Additional Resources

- [Kubernetes Networking](https://kubernetes.io/docs/concepts/services-networking/)
- [CNI Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Service Networking](https://kubernetes.io/docs/concepts/services-networking/service/)

**Remember: Networking is the nervous system of Kubernetes - it connects everything! 🌐**
