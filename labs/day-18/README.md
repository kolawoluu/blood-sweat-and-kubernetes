# Day 18 Lab: Upgrades & Maintenance - The Maintenance Master

> **"Today you'll break upgrades and watch clusters fail. You'll learn why upgrades and maintenance are the maintenance master of Kubernetes."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch upgrade components work in real-time
- Break upgrades and see clusters fail
- Test upgrade failures
- Practice upgrade procedures
- Understand the upgrade architecture
- Learn why upgrades and maintenance are critical for cluster health

## ⚠️ Prerequisites

- Day 17 lab completed
- Minikube cluster running
- 4-5 hours of uninterrupted time
- Understanding of cluster concepts

## 🛠️ Setup (30 minutes)

### Step 1: Verify Cluster Status

```bash
# Check cluster is running
kubectl get nodes
kubectl get pods -n kube-system

# Clean up previous lab resources
kubectl delete pod sidecar-pattern ambassador-pattern adapter-pattern init-pattern controller-pattern observer-pattern factory-pattern singleton-pattern documented-pattern validation-pattern monitoring-pattern complex-pattern -n pattern-test
kubectl delete deployment application-operator controller-pattern -n pattern-test
kubectl delete crd applications.example.com
kubectl delete namespace pattern-test
```

### Step 2: Setup Upgrade Environment

```bash
# Create upgrade namespace
kubectl create namespace upgrade-test

# Create upgrade labels
kubectl label namespace upgrade-test environment=test
kubectl label namespace upgrade-test tier=upgrade
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

## 🧪 Lab Part 1: Watch Upgrades Work (45 minutes)

### Exercise 1.1: Check Current Cluster Version

```bash
# Check cluster version
kubectl version
# Should show client and server versions

# Check node versions
kubectl get nodes -o wide
# Should show node versions
```

### Exercise 1.2: Create Test Workload

```bash
# Create test deployment
kubectl create deployment test-app --image=nginx:1.20 -n upgrade-test --replicas=3

# Create service
kubectl expose deployment test-app --port=80 -n upgrade-test

# Check deployment
kubectl get deployment test-app -n upgrade-test
# READY: 3/3
```

### Exercise 3: Test Rolling Update

```bash
# Update deployment image
kubectl set image deployment/test-app nginx=nginx:1.21 -n upgrade-test

# Watch rolling update
kubectl get pods -l app=test-app -n upgrade-test --watch

# Pods should be updated
```

### Exercise 1.4: Test Rollback

```bash
# Rollback deployment
kubectl rollout undo deployment/test-app -n upgrade-test

# Watch rollback
kubectl get pods -l app=test-app -n upgrade-test --watch

# Pods should be rolled back
```

---

## 💀 Lab Part 2: Break #20 - Stop Upgrade (1 hour)

> **"Time to break the upgrade. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test workload
kubectl create deployment chaos-test --image=nginx:1.20 -n upgrade-test --replicas=3

# Create service
kubectl expose deployment chaos-test --port=80 -n upgrade-test

# Verify everything is working
kubectl get pods -l app=chaos-test -n upgrade-test
kubectl get svc chaos-test -n upgrade-test
```

### Exercise 2.2: Break Upgrade

```bash
# Start upgrade
kubectl set image deployment/chaos-test nginx=nginx:1.21 -n upgrade-test

# Break upgrade by stopping pods
kubectl delete pod -l app=chaos-test -n upgrade-test

# Check deployment status
kubectl get deployment chaos-test -n upgrade-test
# READY: 2/3 (one pod deleted)
```

### Exercise 2.3: Observe the Chaos

**Check deployment status:**
```bash
# Check deployment
kubectl get deployment chaos-test -n upgrade-test
# READY: 2/3

# Check pods
kubectl get pods -l app=chaos-test -n upgrade-test
# Should show 2 pods

# Check events
kubectl get events -n upgrade-test
# Should show: FailedCreate
```

### Exercise 2.4: Debug the Problem

```bash
# Check deployment status
kubectl describe deployment chaos-test -n upgrade-test
# Should show: Replicas: 3, Ready: 2

# Check pod events
kubectl describe pod <pod-name> -n upgrade-test
# Should show: FailedCreate

# Check events
kubectl get events -n upgrade-test
# Should show: FailedCreate
```

**Key Learning:** 
- Upgrade process is critical for cluster health
- Broken upgrade = partial deployment
- Some pods updated, some not
- Inconsistent state

### Exercise 2.5: Fix It

```bash
# Fix deployment
kubectl rollout restart deployment/chaos-test -n upgrade-test

# Wait for deployment to be ready
kubectl get deployment chaos-test -n upgrade-test
# READY: 3/3

# Check pods
kubectl get pods -l app=chaos-test -n upgrade-test
# Should show 3 pods
```

**Real-world parallel:** This happens when:
- Upgrade process fails
- Resource exhaustion
- Configuration errors
- Network issues

---

## 🔥 Lab Part 3: Maintenance Failures (1 hour)

> **"Time to test what happens when maintenance fails. This is where maintenance resilience is tested."**

### Exercise 3.1: Test Node Maintenance

```bash
# Cordon a node
kubectl cordon minikube-m02

# Check node status
kubectl get nodes
# minikube-m02   Ready,SchedulingDisabled

# Create new pods
kubectl create deployment maintenance-test --image=nginx -n upgrade-test --replicas=3

# Check pod distribution
kubectl get pods -l app=maintenance-test -n upgrade-test -o wide
# Should avoid cordoned node
```

### Exercise 3.2: Test Node Drain

```bash
# Drain a node
kubectl drain minikube-m02 --ignore-daemonsets --delete-emptydir-data

# Check pod distribution
kubectl get pods -l app=maintenance-test -n upgrade-test -o wide
# Pods should be evicted and recreated on other nodes
```

### Exercise 3.3: Test Node Uncordon

```bash
# Uncordon the node
kubectl uncordon minikube-m02

# Check node status
kubectl get nodes
# minikube-m02   Ready

# Create new pods
kubectl create deployment uncordon-test --image=nginx -n upgrade-test --replicas=2

# Check pod distribution
kubectl get pods -l app=uncordon-test -n upgrade-test -o wide
# Should be distributed across all nodes
```

### Exercise 3.4: Test Cluster Backup

```bash
# Backup cluster resources
kubectl get all -n upgrade-test -o yaml > upgrade-backup.yaml

# Check backup
ls -la upgrade-backup.yaml
# Should show backup file
```

---

## 🧠 Lab Part 4: Advanced Upgrade Scenarios (1 hour)

> **"Time to test advanced upgrade scenarios. This is where you become an upgrade expert."**

### Exercise 4.1: Test Blue-Green Deployment

```bash
# Create blue deployment
kubectl create deployment blue --image=nginx:1.20 -n upgrade-test --replicas=3

# Create green deployment
kubectl create deployment green --image=nginx:1.21 -n upgrade-test --replicas=3

# Create services
kubectl expose deployment blue --port=80 -n upgrade-test
kubectl expose deployment green --port=80 -n upgrade-test

# Check deployments
kubectl get deployments -n upgrade-test
# Should show: blue, green
```

### Exercise 4.2: Test Canary Deployment

```bash
# Create canary deployment
kubectl create deployment canary --image=nginx:1.21 -n upgrade-test --replicas=1

# Create canary service
kubectl expose deployment canary --port=80 -n upgrade-test

# Check canary
kubectl get deployment canary -n upgrade-test
# READY: 1/1
```

### Exercise 4.3: Test A/B Testing

```bash
# Create A/B testing deployment
kubectl create deployment ab-test --image=nginx:1.21 -n upgrade-test --replicas=2

# Create A/B testing service
kubectl expose deployment ab-test --port=80 -n upgrade-test

# Check A/B testing
kubectl get deployment ab-test -n upgrade-test
# READY: 2/2
```

### Exercise 4.4: Test Rollback Strategy

```bash
# Create rollback deployment
kubectl create deployment rollback-test --image=nginx:1.20 -n upgrade-test --replicas=3

# Update image
kubectl set image deployment/rollback-test nginx=nginx:1.21 -n upgrade-test

# Test rollback
kubectl rollout undo deployment/rollback-test -n upgrade-test

# Check rollback
kubectl get deployment rollback-test -n upgrade-test
# READY: 3/3
```

---

## 🎯 Lab Part 5: Maintenance Best Practices (45 minutes)

### Exercise 5.1: Test Maintenance Windows

```bash
# Create maintenance window deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: maintenance-window
  namespace: upgrade-test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: maintenance-window
  template:
    metadata:
      labels:
        app: maintenance-window
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
EOF

# Check deployment
kubectl get deployment maintenance-window -n upgrade-test
# READY: 3/3
```

### Exercise 5.2: Test Maintenance Monitoring

```bash
# Install maintenance monitoring
kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml

# Check monitoring
kubectl get pods -n monitoring
# Should show monitoring pods
```

### Exercise 5.3: Test Maintenance Documentation

```bash
# Create maintenance documentation
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: maintenance-docs
  namespace: upgrade-test
data:
  maintenance.md: |
    # Maintenance Procedures
    
    ## Cluster Upgrade
    1. Backup cluster state
    2. Update control plane
    3. Update worker nodes
    4. Verify cluster health
    
    ## Node Maintenance
    1. Cordon node
    2. Drain node
    3. Perform maintenance
    4. Uncordon node
    
    ## Application Updates
    1. Test in staging
    2. Deploy to production
    3. Monitor health
    4. Rollback if needed
EOF

# Check documentation
kubectl get configmap maintenance-docs -n upgrade-test
kubectl describe configmap maintenance-docs -n upgrade-test
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-18-reflections.md` and answer:

1. **What do upgrades and maintenance do?**
   - How do they manage cluster health?
   - What happens when they fail?
   - Why are they critical for cluster operations?

2. **What happens when upgrades fail?**
   - How do you detect upgrade failures?
   - What happens to applications?
   - How do you recover from upgrade failures?

3. **How do you use upgrades and maintenance?**
   - What's the difference between upgrades and maintenance?
   - How do you manage upgrade complexity?
   - What are the best practices?

4. **How do you debug upgrade issues?**
   - What tools do you use?
   - How do you check upgrade status?
   - How do you test upgrades?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Kubernetes upgrade failures**
- **Maintenance issues**
- **Upgrade complexity problems**
- **Maintenance best practices**

### Exercise 6.3: Advanced Upgrade Scenarios

```bash
# Create complex upgrade scenario
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: complex-upgrade
  namespace: upgrade-test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: complex-upgrade
  template:
    metadata:
      labels:
        app: complex-upgrade
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
EOF

# Test the upgrade
kubectl get deployment complex-upgrade -n upgrade-test
# READY: 3/3
```

---

## 🏆 Success Criteria

You've successfully completed Day 18 if you can:

✅ **Watch upgrade components work in real-time**  
✅ **Break upgrades and understand the impact**  
✅ **Test upgrade failures**  
✅ **Practice upgrade procedures**  
✅ **Understand the upgrade architecture**  
✅ **Debug upgrade issues**  

## 🚨 Common Issues & Solutions

### Issue: Upgrades not working
```bash
# Check deployment status
kubectl get deployment <name>

# Check deployment events
kubectl describe deployment <name>

# Check pod status
kubectl get pods -l app=<name>
```

### Issue: Maintenance not working
```bash
# Check node status
kubectl get nodes

# Check node events
kubectl describe node <node-name>

# Check pod distribution
kubectl get pods -o wide
```

### Issue: Rollback not working
```bash
# Check deployment history
kubectl rollout history deployment/<name>

# Check rollback status
kubectl rollout undo deployment/<name>

# Check deployment status
kubectl get deployment <name>
```

---

## 🎯 Tomorrow's Preview

**Day 19: Service Mesh - The Mesh Master**
- Watch service mesh components work
- Break service mesh and see traffic fail
- Test mesh failures
- Practice mesh procedures
- Understand the mesh architecture

**Get ready for the mesh master! 🔥**

---

## 📚 Additional Resources

- [Kubernetes Upgrades](https://kubernetes.io/docs/tasks/administer-cluster/cluster-management/)
- [Maintenance Procedures](https://kubernetes.io/docs/tasks/administer-cluster/cluster-management/)
- [Upgrade Strategies](https://kubernetes.io/docs/tasks/administer-cluster/cluster-management/)
- [Maintenance Best Practices](https://kubernetes.io/docs/tasks/administer-cluster/cluster-management/)

**Remember: Upgrades and maintenance are the maintenance master of Kubernetes - they keep everything running! 🔧**
