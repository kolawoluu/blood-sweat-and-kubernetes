# Day 13 Lab: Multi-cluster - The Federation

> **"Today you'll break multi-cluster and watch clusters disconnect. You'll learn why multi-cluster is the federation of Kubernetes."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch multi-cluster components work in real-time
- Break multi-cluster and see clusters disconnect
- Test cluster federation failures
- Practice multi-cluster procedures
- Understand the multi-cluster architecture
- Learn why multi-cluster is critical for high availability

## ⚠️ Prerequisites

- Day 12 lab completed
- Minikube cluster running
- 4-5 hours of uninterrupted time
- Understanding of basic cluster concepts

## 🛠️ Setup (30 minutes)

### Step 1: Verify Cluster Status

```bash
# Check cluster is running
kubectl get nodes
kubectl get pods -n kube-system

# Clean up previous lab resources
kubectl delete job init-job resource-job node-selector-job complex-job -n job-test
kubectl delete cronjob hello-cronjob -n job-test
kubectl delete namespace job-test
```

### Step 2: Setup Multi-cluster Environment

```bash
# Create second cluster
minikube start --nodes=2 --cpus=2 --memory=2048 --profile=cluster2

# Check clusters
minikube profile list
# Should show: minikube, cluster2

# Switch to cluster2
minikube profile cluster2

# Check cluster2 status
kubectl get nodes
# Should show 2 nodes
```

### Step 3: Setup Monitoring

```bash
# Open 4 terminals
# Terminal 1: kubectl get pods --all-namespaces --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: kubectl get nodes --watch
# Terminal 4: Your command terminal
```

---

## 🧪 Lab Part 1: Watch Multi-cluster Work (45 minutes)

### Exercise 1.1: Create Cross-cluster Resources

```bash
# Create namespace for testing
kubectl create namespace multi-cluster-test

# Create deployment in cluster2
kubectl create deployment web --image=nginx -n multi-cluster-test --replicas=2

# Create service
kubectl expose deployment web --port=80 -n multi-cluster-test

# Check resources
kubectl get all -n multi-cluster-test
```

### Exercise 1.2: Test Cross-cluster Communication

```bash
# Create test pod
kubectl create deployment test --image=busybox -n multi-cluster-test --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=ready pod -l app=test -n multi-cluster-test --timeout=60s
kubectl exec -it deployment/test -n multi-cluster-test -- sh

# Inside the pod, test service connectivity
wget -O- http://web
# Should work

# Exit the test pod
exit
kubectl delete deployment test -n multi-cluster-test
```

### Exercise 1.3: Switch Between Clusters

```bash
# Switch to cluster1
minikube profile minikube

# Check cluster1 status
kubectl get nodes
# Should show 3 nodes

# Switch back to cluster2
minikube profile cluster2

# Check cluster2 status
kubectl get nodes
# Should show 2 nodes
```

### Exercise 1.4: Test Cluster Isolation

```bash
# Check resources in cluster2
kubectl get all -n multi-cluster-test
# Should show resources

# Switch to cluster1
minikube profile minikube

# Check resources in cluster1
kubectl get all -n multi-cluster-test
# Should show: No resources found

# Switch back to cluster2
minikube profile cluster2
```

---

## 💀 Lab Part 2: Break #15 - Stop Cluster (1 hour)

> **"Time to break the cluster. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test deployment
kubectl create deployment chaos-test --image=nginx -n multi-cluster-test --replicas=3

# Create service
kubectl expose deployment chaos-test --port=80 -n multi-cluster-test

# Verify everything is working
kubectl get pods -l app=chaos-test -n multi-cluster-test
kubectl get svc chaos-test -n multi-cluster-test
```

### Exercise 2.2: Break Cluster

```bash
# Stop cluster2
minikube stop --profile=cluster2

# Check cluster status
minikube profile list
# cluster2 should show: Stopped
```

### Exercise 2.3: Observe the Chaos

**Try to access cluster2:**
```bash
# Try to get nodes
kubectl get nodes
# Error: The connection to the server was refused

# Try to get pods
kubectl get pods
# Error: The connection to the server was refused

# Try to get services
kubectl get svc
# Error: The connection to the server was refused
```

### Exercise 2.4: Debug the Problem

```bash
# Check cluster status
minikube profile list
# cluster2: Stopped

# Check cluster logs
minikube logs --profile=cluster2
# Should show cluster shutdown logs
```

**Key Learning:** 
- Cluster is completely down
- No access to any resources
- All workloads unavailable
- Complete service outage

### Exercise 2.5: Fix It

```bash
# Start cluster2
minikube start --profile=cluster2

# Wait for cluster to be ready
kubectl get nodes
# Should show 2 nodes

# Check resources
kubectl get all -n multi-cluster-test
# Should show all resources
```

**Real-world parallel:** This happens when:
- Cluster crashes
- Network partitions
- Hardware failures
- Configuration errors

---

## 🔥 Lab Part 3: Cluster Federation Failures (1 hour)

> **"Time to test what happens when cluster federation fails. This is where multi-cluster resilience is tested."**

### Exercise 3.1: Test Cluster Federation

```bash
# Install cluster federation
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/federation-v2/main/deploy/federation-v2.yaml

# Check federation components
kubectl get pods -n federation-system
# Should show federation pods
```

### Exercise 3.2: Break Cluster Federation

```bash
# Delete federation components
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/federation-v2/main/deploy/federation-v2.yaml

# Check federation status
kubectl get pods -n federation-system
# Should show: No resources found
```

### Exercise 3.3: Test Cluster Isolation

```bash
# Check cluster1 resources
minikube profile minikube
kubectl get all -n multi-cluster-test
# Should show: No resources found

# Check cluster2 resources
minikube profile cluster2
kubectl get all -n multi-cluster-test
# Should show resources
```

### Exercise 3.4: Test Cluster Recovery

```bash
# Reinstall federation
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/federation-v2/main/deploy/federation-v2.yaml

# Wait for federation to be ready
kubectl get pods -n federation-system
# Should show federation pods
```

**Real-world parallel:** This happens when:
- Federation controller crashes
- Network issues between clusters
- Configuration errors
- Resource exhaustion

---

## 🧠 Lab Part 4: Multi-cluster Management (1 hour)

> **"Time to test multi-cluster management. This is where you learn about cluster operations."**

### Exercise 4.1: Test Cluster Scaling

```bash
# Scale deployment in cluster2
kubectl scale deployment chaos-test --replicas=5 -n multi-cluster-test

# Check scaling
kubectl get pods -l app=chaos-test -n multi-cluster-test
# Should show 5 pods
```

### Exercise 4.2: Test Cluster Updates

```bash
# Update deployment image
kubectl set image deployment/chaos-test nginx=nginx:1.20 -n multi-cluster-test

# Watch rolling update
kubectl get pods -l app=chaos-test -n multi-cluster-test --watch

# Pods should be updated
```

### Exercise 4.3: Test Cluster Backup

```bash
# Backup cluster2 resources
kubectl get all -n multi-cluster-test -o yaml > cluster2-backup.yaml

# Check backup
ls -la cluster2-backup.yaml
# Should show backup file
```

### Exercise 4.4: Test Cluster Restore

```bash
# Delete resources
kubectl delete all --all -n multi-cluster-test

# Check resources
kubectl get all -n multi-cluster-test
# Should show: No resources found

# Restore from backup
kubectl apply -f cluster2-backup.yaml

# Check resources
kubectl get all -n multi-cluster-test
# Should show all resources
```

---

## 🎯 Lab Part 5: Advanced Multi-cluster Scenarios (45 minutes)

### Exercise 5.1: Test Cross-cluster Networking

```bash
# Create service in cluster2
kubectl create service clusterip cross-cluster --tcp=80:80 -n multi-cluster-test

# Create endpoints
kubectl create deployment cross-cluster-backend --image=nginx -n multi-cluster-test --replicas=2

# Check service
kubectl get svc cross-cluster -n multi-cluster-test
kubectl get endpoints cross-cluster -n multi-cluster-test
```

### Exercise 5.2: Test Cluster Load Balancing

```bash
# Create load balancer service
kubectl expose deployment chaos-test --port=80 --type=LoadBalancer -n multi-cluster-test

# Check service
kubectl get svc chaos-test -n multi-cluster-test
# TYPE: LoadBalancer
```

### Exercise 5.3: Test Cluster Monitoring

```bash
# Install monitoring in cluster2
kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml

# Check monitoring
kubectl get pods -n monitoring
# Should show monitoring pods
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-13-reflections.md` and answer:

1. **What does multi-cluster do?**
   - How does it manage multiple clusters?
   - What happens when it fails?
   - Why is it critical for high availability?

2. **What happens when cluster fails?**
   - How do you detect cluster failures?
   - What happens to workloads?
   - How do you recover from cluster failures?

3. **How do you use multi-cluster?**
   - What's the difference between single and multi-cluster?
   - How do you manage multiple clusters?
   - What are the best practices?

4. **How do you debug multi-cluster issues?**
   - What tools do you use?
   - How do you check cluster status?
   - How do you test multi-cluster?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Kubernetes multi-cluster failures**
- **Cluster federation issues**
- **Cross-cluster networking problems**
- **Multi-cluster best practices**

### Exercise 6.3: Advanced Multi-cluster Scenarios

```bash
# Create complex multi-cluster scenario
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-cluster-test
  namespace: multi-cluster-test
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: nginx-config
EOF

# Test the multi-cluster setup
kubectl get pod multi-cluster-test -n multi-cluster-test
# STATUS: Running
```

---

## 🏆 Success Criteria

You've successfully completed Day 13 if you can:

✅ **Watch multi-cluster components work in real-time**  
✅ **Break multi-cluster and understand the impact**  
✅ **Test cluster federation failures**  
✅ **Practice multi-cluster procedures**  
✅ **Understand the multi-cluster architecture**  
✅ **Debug multi-cluster issues**  

## 🚨 Common Issues & Solutions

### Issue: Cluster not accessible
```bash
# Check cluster status
minikube profile list

# Check cluster logs
minikube logs --profile=<cluster-name>

# Restart cluster
minikube start --profile=<cluster-name>
```

### Issue: Federation not working
```bash
# Check federation components
kubectl get pods -n federation-system

# Check federation logs
kubectl logs -n federation-system -l app=federation-controller
```

### Issue: Cross-cluster networking not working
```bash
# Check network policies
kubectl get networkpolicies

# Check service connectivity
kubectl get svc
kubectl get endpoints
```

---

## 🎯 Tomorrow's Preview

**Day 14: Production Readiness - The Final Boss**
- Watch production components work
- Break production and see everything fail
- Test production failures and recovery
- Practice production procedures
- Understand the production architecture

**Get ready for the final boss! 🔥**

---

## 📚 Additional Resources

- [Kubernetes Multi-cluster](https://kubernetes.io/docs/concepts/cluster-administration/federation/)
- [Cluster Federation](https://kubernetes.io/docs/concepts/cluster-administration/federation/)
- [Cross-cluster Networking](https://kubernetes.io/docs/concepts/services-networking/)
- [Multi-cluster Management](https://kubernetes.io/docs/concepts/cluster-administration/)

**Remember: Multi-cluster is the federation of Kubernetes - it connects everything! 🌐**
