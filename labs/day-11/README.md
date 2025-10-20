# Day 11 Lab: StatefulSets - The Memory System

> **"Today you'll break StatefulSets and watch pods lose their state. You'll learn why StatefulSets are the memory system of Kubernetes."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch StatefulSets work in real-time
- Break StatefulSets and see pods lose state
- Test persistent volume failures
- Practice StatefulSet procedures
- Understand the StatefulSet architecture
- Learn why StatefulSets are critical for stateful applications

## ⚠️ Prerequisites

- Day 10 lab completed
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
kubectl delete pod ssd-pod affinity-pod anti-affinity-pod no-toleration-pod toleration-pod multi-toleration-pod high-resource-pod node-selector-pod priority-pod impossible-pod resource-exhaustion-pod complex-scheduling-pod
kubectl delete deployment chaos-test test spread-test
kubectl delete service chaos-test
kubectl delete priorityclass high-priority
kubectl delete namespace scheduling-test
```

### Step 2: Setup Monitoring

```bash
# Open 4 terminals
# Terminal 1: kubectl get pods --all-namespaces --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: kubectl get pv,pvc --watch
# Terminal 4: Your command terminal
```

### Step 3: Check StatefulSet Components

```bash
# Check if StatefulSet controller is running
kubectl get pods -n kube-system | grep controller-manager

# Check storage classes
kubectl get storageclass
```

---

## 🧪 Lab Part 1: Watch StatefulSets Work (45 minutes)

### Exercise 1.1: Create StatefulSet

```bash
# Create namespace for testing
kubectl create namespace stateful-test

# Create StatefulSet
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: stateful-test
spec:
  serviceName: web
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF

# Check StatefulSet
kubectl get statefulset web -n stateful-test
# NAME   READY   AGE
# web    3/3     30s
```

### Exercise 1.2: Watch Pod Creation

```bash
# Watch pods being created
kubectl get pods -l app=web -n stateful-test --watch

# You should see:
# web-0   Pending
# web-0   ContainerCreating
# web-0   Running
# web-1   Pending
# web-1   ContainerCreating
# web-1   Running
# web-2   Pending
# web-2   ContainerCreating
# web-2   Running

# Pods are created in order: 0, 1, 2
```

### Exercise 1.3: Check Persistent Volumes

```bash
# Check PVCs
kubectl get pvc -n stateful-test
# NAME        STATUS   VOLUME
# www-web-0   Bound    pvc-xxx
# www-web-1   Bound    pvc-yyy
# www-web-2   Bound    pvc-zzz

# Check PVs
kubectl get pv
# Should show 3 PVs bound to the PVCs
```

### Exercise 1.4: Test StatefulSet Features

```bash
# Check pod names
kubectl get pods -l app=web -n stateful-test
# NAME    READY   STATUS
# web-0   Running
# web-1   Running
# web-2   Running

# Check pod IPs
kubectl get pods -l app=web -n stateful-test -o wide
# Pods have stable network identities

# Test pod-to-pod communication
kubectl exec web-0 -n stateful-test -- sh -c "curl -s http://web-1"
# Should work - pods can communicate by name
```

---

## 💀 Lab Part 2: Break #13 - Stop StatefulSet Controller (1 hour)

> **"Time to break the StatefulSet controller. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test data
kubectl exec web-0 -n stateful-test -- sh -c "echo 'Data from web-0' > /usr/share/nginx/html/index.html"
kubectl exec web-1 -n stateful-test -- sh -c "echo 'Data from web-1' > /usr/share/nginx/html/index.html"
kubectl exec web-2 -n stateful-test -- sh -c "echo 'Data from web-2' > /usr/share/nginx/html/index.html"

# Verify data
kubectl exec web-0 -n stateful-test -- cat /usr/share/nginx/html/index.html
kubectl exec web-1 -n stateful-test -- cat /usr/share/nginx/html/index.html
kubectl exec web-2 -n stateful-test -- cat /usr/share/nginx/html/index.html
```

### Exercise 2.2: Break StatefulSet Controller

```bash
# Delete controller manager pod
kubectl delete pod -n kube-system kube-controller-manager-minikube

# Wait 10 seconds, then prevent it from restarting
minikube ssh
sudo mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
exit
```

### Exercise 2.3: Observe the Chaos

**Watch Terminal 1 (pods):**
- Existing pods keep running
- No new StatefulSet operations

**Test StatefulSet operations:**
```bash
# Try to scale StatefulSet
kubectl scale statefulset web --replicas=5 -n stateful-test

# Check StatefulSet status
kubectl get statefulset web -n stateful-test
# READY: 3/5 (not scaling!)

# Check pods
kubectl get pods -l app=web -n stateful-test
# Still only 3 pods!

# Try to delete a pod
kubectl delete pod web-1 -n stateful-test

# Check pods
kubectl get pods -l app=web -n stateful-test
# web-1 is gone, but no replacement created!
```

### Exercise 2.4: Debug the Problem

```bash
# Check controller manager
kubectl get pods -n kube-system | grep controller-manager
# NOT FOUND!

# Check StatefulSet status
kubectl describe statefulset web -n stateful-test
# Should show: Replicas: 5, Ready: 3

# Check events
kubectl get events -n stateful-test
# Should show: FailedCreate
```

**Key Learning:** 
- StatefulSet controller is critical for StatefulSet operations
- No controller = no scaling, no self-healing
- Existing pods unaffected
- StatefulSet becomes "frozen"

### Exercise 2.5: Fix It

```bash
minikube ssh
sudo mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/
exit

# Wait for controller manager to restart
kubectl get pods -n kube-system | grep controller-manager
# Running!

# Check StatefulSet status
kubectl get statefulset web -n stateful-test
# READY: 5/5

# Check pods
kubectl get pods -l app=web -n stateful-test
# All 5 pods: Running
```

**Real-world parallel:** This happens when:
- Controller manager crashes
- Resource exhaustion
- Configuration errors
- Network issues

---

## 🔥 Lab Part 3: Persistent Volume Failures (1 hour)

> **"Time to test what happens when persistent volumes fail. This is where StatefulSet resilience is tested."**

### Exercise 3.1: Test Persistent Volume Failure

```bash
# Check PVCs
kubectl get pvc -n stateful-test
# Should show 5 PVCs

# Check PVs
kubectl get pv
# Should show 5 PVs

# Simulate PV failure by deleting PV
kubectl get pv | grep www-web-0
# Note the PV name

kubectl delete pv <pv-name>
# This will break the PV
```

### Exercise 3.2: Observe the Chaos

**Watch Terminal 1 (pods):**
- Pod with failed PV shows "Unknown" status
- No new pod can be created

**Check pod status:**
```bash
# Check pod status
kubectl get pods -l app=web -n stateful-test
# web-0 should show: Unknown

# Check pod events
kubectl describe pod web-0 -n stateful-test
# Should show: FailedMount
```

### Exercise 3.3: Test StatefulSet Recovery

```bash
# Delete the failed pod
kubectl delete pod web-0 -n stateful-test

# Check StatefulSet status
kubectl get statefulset web -n stateful-test
# READY: 4/5

# Check pods
kubectl get pods -l app=web -n stateful-test
# web-0 should be recreated but stuck in Pending
```

### Exercise 3.4: Fix Persistent Volume

```bash
# Recreate PV
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: <pv-name>
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/stateful-test
EOF

# Check pod status
kubectl get pods -l app=web -n stateful-test
# web-0 should be Running

# Check StatefulSet status
kubectl get statefulset web -n stateful-test
# READY: 5/5
```

**Real-world parallel:** This happens when:
- Storage system failures
- Disk failures
- Network storage disconnections
- Hardware failures

---

## 🧠 Lab Part 4: StatefulSet Scaling (1 hour)

> **"Time to test StatefulSet scaling. This is where you learn about ordered scaling."**

### Exercise 4.1: Test Ordered Scaling

```bash
# Scale up StatefulSet
kubectl scale statefulset web --replicas=5 -n stateful-test

# Watch pods being created
kubectl get pods -l app=web -n stateful-test --watch

# You should see:
# web-3   Pending
# web-3   ContainerCreating
# web-3   Running
# web-4   Pending
# web-4   ContainerCreating
# web-4   Running

# Pods are created in order: 3, 4
```

### Exercise 4.2: Test Ordered Scaling Down

```bash
# Scale down StatefulSet
kubectl scale statefulset web --replicas=3 -n stateful-test

# Watch pods being deleted
kubectl get pods -l app=web -n stateful-test --watch

# You should see:
# web-4   Terminating
# web-4   Terminated
# web-3   Terminating
# web-3   Terminated

# Pods are deleted in reverse order: 4, 3
```

### Exercise 4.3: Test StatefulSet Updates

```bash
# Update StatefulSet image
kubectl patch statefulset web -n stateful-test -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.20"}]}}}}'

# Watch rolling update
kubectl get pods -l app=web -n stateful-test --watch

# You should see:
# web-2   Terminating
# web-2   Terminated
# web-2   Pending
# web-2   ContainerCreating
# web-2   Running
# web-1   Terminating
# web-1   Terminated
# web-1   Pending
# web-1   ContainerCreating
# web-1   Running
# web-0   Terminating
# web-0   Terminated
# web-0   Pending
# web-0   ContainerCreating
# web-0   Running

# Pods are updated in reverse order: 2, 1, 0
```

### Exercise 4.4: Test StatefulSet Rollback

```bash
# Rollback StatefulSet
kubectl rollout undo statefulset web -n stateful-test

# Watch rollback
kubectl get pods -l app=web -n stateful-test --watch

# Pods are rolled back in reverse order: 2, 1, 0
```

---

## 🎯 Lab Part 5: Advanced StatefulSet Scenarios (45 minutes)

### Exercise 5.1: Test StatefulSet with Headless Service

```bash
# Create headless service
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: stateful-test
spec:
  clusterIP: None
  selector:
    app: web
  ports:
  - port: 80
    name: web
EOF

# Test service
kubectl get svc web -n stateful-test
# CLUSTER-IP: None

# Test DNS resolution
kubectl create deployment test --image=busybox -n stateful-test --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=ready pod -l app=test -n stateful-test --timeout=60s
kubectl exec -it deployment/test -n stateful-test -- sh

# Inside the pod
nslookup web
# Should resolve to all pod IPs

nslookup web-0.web
# Should resolve to web-0 IP

nslookup web-1.web
# Should resolve to web-1 IP

# Exit the test pod
exit
kubectl delete deployment test -n stateful-test
```

### Exercise 5.2: Test StatefulSet with Init Containers

```bash
# Create StatefulSet with init container
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: init-test
  namespace: stateful-test
spec:
  serviceName: init-test
  replicas: 2
  selector:
    matchLabels:
      app: init-test
  template:
    metadata:
      labels:
        app: init-test
    spec:
      initContainers:
      - name: init
        image: busybox
        command: ['sh', '-c', 'echo "Init container" > /shared/init.txt']
        volumeMounts:
        - name: shared
          mountPath: /shared
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: shared
          mountPath: /shared
  volumeClaimTemplates:
  - metadata:
      name: shared
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF

# Check StatefulSet
kubectl get statefulset init-test -n stateful-test
# READY: 2/2

# Check init container logs
kubectl logs init-test-0 -c init -n stateful-test
# Should show: Init container

# Check shared data
kubectl exec init-test-0 -n stateful-test -- cat /shared/init.txt
# Should show: Init container
```

### Exercise 5.3: Test StatefulSet with Pod Disruption Budget

```bash
# Create Pod Disruption Budget
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
  namespace: stateful-test
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
EOF

# Check PDB
kubectl get pdb web-pdb -n stateful-test
# MIN AVAILABLE: 2

# Test PDB
kubectl delete pod web-0 -n stateful-test
# Should be allowed (3 pods, min 2)

kubectl delete pod web-1 -n stateful-test
# Should be allowed (2 pods, min 2)

kubectl delete pod web-2 -n stateful-test
# Should be blocked (1 pod, min 2)
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-11-reflections.md` and answer:

1. **What do StatefulSets do?**
   - How do they manage stateful applications?
   - What happens when they fail?
   - Why are they critical for stateful workloads?

2. **What happens when StatefulSet controller fails?**
   - How do you detect StatefulSet failures?
   - What happens to pods?
   - How do you recover from StatefulSet failures?

3. **How do you use StatefulSets?**
   - What's the difference between StatefulSets and Deployments?
   - How do you scale StatefulSets?
   - What are the best practices?

4. **How do you debug StatefulSet issues?**
   - What tools do you use?
   - How do you check StatefulSet status?
   - How do you test StatefulSets?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Kubernetes StatefulSet failures**
- **StatefulSet best practices**
- **StatefulSet scaling issues**
- **Persistent volume failures**

### Exercise 6.3: Advanced StatefulSet Scenarios

```bash
# Create complex StatefulSet scenario
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: complex-statefulset
  namespace: stateful-test
spec:
  serviceName: complex-statefulset
  replicas: 3
  selector:
    matchLabels:
      app: complex-statefulset
  template:
    metadata:
      labels:
        app: complex-statefulset
    spec:
      initContainers:
      - name: init
        image: busybox
        command: ['sh', '-c', 'echo "Init" > /shared/init.txt']
        volumeMounts:
        - name: shared
          mountPath: /shared
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: shared
          mountPath: /shared
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
  volumeClaimTemplates:
  - metadata:
      name: shared
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF

# Test the StatefulSet
kubectl get statefulset complex-statefulset -n stateful-test
# READY: 3/3
```

---

## 🏆 Success Criteria

You've successfully completed Day 11 if you can:

✅ **Watch StatefulSets work in real-time**  
✅ **Break StatefulSets and understand the impact**  
✅ **Test persistent volume failures**  
✅ **Practice StatefulSet procedures**  
✅ **Understand the StatefulSet architecture**  
✅ **Debug StatefulSet issues**  

## 🚨 Common Issues & Solutions

### Issue: StatefulSet not scaling
```bash
# Check StatefulSet status
kubectl get statefulset <name>

# Check StatefulSet events
kubectl describe statefulset <name>

# Check controller manager logs
kubectl logs kube-controller-manager-minikube -n kube-system
```

### Issue: Pods stuck in Pending
```bash
# Check pod events
kubectl describe pod <pod-name>

# Check PVC status
kubectl get pvc

# Check PV status
kubectl get pv
```

### Issue: StatefulSet not updating
```bash
# Check StatefulSet status
kubectl get statefulset <name>

# Check pod status
kubectl get pods -l app=<name>

# Check events
kubectl get events
```

---

## 🎯 Tomorrow's Preview

**Day 12: Jobs & CronJobs - The Task Scheduler**
- Watch Jobs and CronJobs work
- Break Jobs and CronJobs and see tasks fail
- Test job failures and retries
- Practice job procedures
- Understand the job architecture

**Get ready for more chaos! 🔥**

---

## 📚 Additional Resources

- [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [StatefulSet Scaling](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#scaling)
- [StatefulSet Updates](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#updating)
- [Pod Disruption Budgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)

**Remember: StatefulSets are the memory system of Kubernetes - they remember everything! 💾**
