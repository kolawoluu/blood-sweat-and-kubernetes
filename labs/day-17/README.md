# Day 17 Lab: Advanced Patterns - The Pattern Master

> **"Today you'll break advanced patterns and watch apps fail. You'll learn why advanced patterns are the pattern master of Kubernetes."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch advanced patterns work in real-time
- Break advanced patterns and see apps fail
- Test pattern failures
- Practice pattern procedures
- Understand the pattern architecture
- Learn why advanced patterns are critical for complex applications

## ⚠️ Prerequisites

- Day 16 lab completed
- Minikube cluster running
- 4-5 hours of uninterrupted time
- Understanding of basic patterns

## 🛠️ Setup (30 minutes)

### Step 1: Verify Cluster Status

```bash
# Check cluster is running
kubectl get nodes
kubectl get pods -n kube-system

# Clean up previous lab resources
kubectl delete pod config-pod chaos-config-pod -n config-test
kubectl delete deployment chaos-test -n config-test
kubectl delete service chaos-test -n config-test
kubectl delete configmap app-config chaos-config template-config base-config env-config invalid-config secure-config validation-config documented-config complex-config -n config-test
kubectl delete secret app-secret chaos-secret encrypted-secret -n config-test
kubectl delete namespace config-test
```

### Step 2: Setup Pattern Environment

```bash
# Create pattern namespace
kubectl create namespace pattern-test

# Create pattern labels
kubectl label namespace pattern-test environment=test
kubectl label namespace pattern-test tier=pattern
```

### Step 3: Setup Monitoring

```bash
# Open 4 terminals
# Terminal 1: kubectl get pods --all-namespaces --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: kubectl get crd --watch
# Terminal 4: Your command terminal
```

---

## 🧪 Lab Part 1: Watch Advanced Patterns Work (45 minutes)

### Exercise 1.1: Create Custom Resource Definition

```bash
# Create CRD
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applications.example.com
spec:
  group: example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              name:
                type: string
              version:
                type: string
              replicas:
                type: integer
          status:
            type: object
            properties:
              phase:
                type: string
  scope: Namespaced
  names:
    plural: applications
    singular: application
    kind: Application
EOF

# Check CRD
kubectl get crd applications.example.com
kubectl describe crd applications.example.com
```

### Exercise 1.2: Create Custom Resource

```bash
# Create custom resource
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Application
metadata:
  name: test-app
  namespace: pattern-test
spec:
  name: test-app
  version: 1.0.0
  replicas: 3
EOF

# Check custom resource
kubectl get applications -n pattern-test
kubectl describe application test-app -n pattern-test
```

### Exercise 1.3: Create Operator

```bash
# Create operator deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: application-operator
  namespace: pattern-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: application-operator
  template:
    metadata:
      labels:
        app: application-operator
    spec:
      containers:
      - name: operator
        image: nginx
        ports:
        - containerPort: 80
        env:
        - name: OPERATOR_NAME
          value: application-operator
        - name: OPERATOR_NAMESPACE
          value: pattern-test
EOF

# Check operator
kubectl get deployment application-operator -n pattern-test
kubectl get pods -l app=application-operator -n pattern-test
```

### Exercise 1.4: Test Operator

```bash
# Check operator logs
kubectl logs -l app=application-operator -n pattern-test

# Check custom resource
kubectl get applications -n pattern-test
kubectl describe application test-app -n pattern-test
```

---

## 💀 Lab Part 2: Break #19 - Stop Operator (1 hour)

> **"Time to break the operator. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test custom resource
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Application
metadata:
  name: chaos-app
  namespace: pattern-test
spec:
  name: chaos-app
  version: 1.0.0
  replicas: 2
EOF

# Verify everything is working
kubectl get applications -n pattern-test
kubectl get deployment application-operator -n pattern-test
```

### Exercise 2.2: Break Operator

```bash
# Delete operator
kubectl delete deployment application-operator -n pattern-test

# Check operator
kubectl get deployment application-operator -n pattern-test
# Error: not found
```

### Exercise 2.3: Observe the Chaos

**Check custom resources:**
```bash
# Check custom resources
kubectl get applications -n pattern-test
# Should show: test-app, chaos-app

# Check custom resource status
kubectl describe application test-app -n pattern-test
# Should show: No operator to manage it

kubectl describe application chaos-app -n pattern-test
# Should show: No operator to manage it
```

### Exercise 2.4: Debug the Problem

```bash
# Check operator
kubectl get deployment application-operator -n pattern-test
# Error: not found

# Check custom resources
kubectl get applications -n pattern-test
# Should show resources but no management

# Check events
kubectl get events -n pattern-test
# Should show: No operator events
```

**Key Learning:** 
- Operator is critical for custom resource management
- No operator = no custom resource management
- Custom resources exist but unmanaged
- No automation or reconciliation

### Exercise 2.5: Fix It

```bash
# Recreate operator
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: application-operator
  namespace: pattern-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: application-operator
  template:
    metadata:
      labels:
        app: application-operator
    spec:
      containers:
      - name: operator
        image: nginx
        ports:
        - containerPort: 80
        env:
        - name: OPERATOR_NAME
          value: application-operator
        - name: OPERATOR_NAMESPACE
          value: pattern-test
EOF

# Wait for operator to be ready
kubectl get deployment application-operator -n pattern-test
# READY: 1/1

# Check custom resources
kubectl get applications -n pattern-test
# Should show managed resources
```

**Real-world parallel:** This happens when:
- Operator crashes
- Resource exhaustion
- Configuration errors
- Network issues

---

## 🔥 Lab Part 3: Pattern Failures (1 hour)

> **"Time to test what happens when patterns fail. This is where pattern resilience is tested."**

### Exercise 3.1: Test Sidecar Pattern

```bash
# Create sidecar pattern
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-pattern
  namespace: pattern-test
spec:
  containers:
  - name: main-app
    image: nginx
    ports:
    - containerPort: 80
  - name: sidecar
    image: busybox
    command: ['sh', '-c', 'while true; do echo "Sidecar running"; sleep 30; done']
EOF

# Check pod
kubectl get pod sidecar-pattern -n pattern-test
# STATUS: Running
```

### Exercise 3.2: Test Ambassador Pattern

```bash
# Create ambassador pattern
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ambassador-pattern
  namespace: pattern-test
spec:
  containers:
  - name: main-app
    image: nginx
    ports:
    - containerPort: 80
  - name: ambassador
    image: nginx
    ports:
    - containerPort: 8080
    command: ['sh', '-c', 'while true; do echo "Ambassador running"; sleep 30; done']
EOF

# Check pod
kubectl get pod ambassador-pattern -n pattern-test
# STATUS: Running
```

### Exercise 3.3: Test Adapter Pattern

```bash
# Create adapter pattern
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: adapter-pattern
  namespace: pattern-test
spec:
  containers:
  - name: main-app
    image: nginx
    ports:
    - containerPort: 80
  - name: adapter
    image: busybox
    command: ['sh', '-c', 'while true; do echo "Adapter running"; sleep 30; done']
EOF

# Check pod
kubectl get pod adapter-pattern -n pattern-test
# STATUS: Running
```

### Exercise 3.4: Test Init Container Pattern

```bash
# Create init container pattern
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-pattern
  namespace: pattern-test
spec:
  initContainers:
  - name: init
    image: busybox
    command: ['sh', '-c', 'echo "Init container" && sleep 10']
  containers:
  - name: main-app
    image: nginx
    ports:
    - containerPort: 80
EOF

# Check pod
kubectl get pod init-pattern -n pattern-test
# STATUS: Running
```

---

## 🧠 Lab Part 4: Advanced Pattern Scenarios (1 hour)

> **"Time to test advanced pattern scenarios. This is where you become a pattern expert."**

### Exercise 4.1: Test Controller Pattern

```bash
# Create controller pattern
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: controller-pattern
  namespace: pattern-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: controller-pattern
  template:
    metadata:
      labels:
        app: controller-pattern
    spec:
      containers:
      - name: controller
        image: nginx
        ports:
        - containerPort: 80
        env:
        - name: CONTROLLER_NAME
          value: controller-pattern
        - name: CONTROLLER_NAMESPACE
          value: pattern-test
EOF

# Check controller
kubectl get deployment controller-pattern -n pattern-test
kubectl get pods -l app=controller-pattern -n pattern-test
```

### Exercise 4.2: Test Observer Pattern

```bash
# Create observer pattern
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: observer-pattern
  namespace: pattern-test
spec:
  containers:
  - name: observer
    image: busybox
    command: ['sh', '-c', 'while true; do echo "Observer running"; sleep 30; done']
  - name: subject
    image: nginx
    ports:
    - containerPort: 80
EOF

# Check pod
kubectl get pod observer-pattern -n pattern-test
# STATUS: Running
```

### Exercise 4.3: Test Factory Pattern

```bash
# Create factory pattern
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: factory-pattern
  namespace: pattern-test
spec:
  containers:
  - name: factory
    image: busybox
    command: ['sh', '-c', 'while true; do echo "Factory running"; sleep 30; done']
  - name: product
    image: nginx
    ports:
    - containerPort: 80
EOF

# Check pod
kubectl get pod factory-pattern -n pattern-test
# STATUS: Running
```

### Exercise 4.4: Test Singleton Pattern

```bash
# Create singleton pattern
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: singleton-pattern
  namespace: pattern-test
spec:
  containers:
  - name: singleton
    image: nginx
    ports:
    - containerPort: 80
    env:
    - name: SINGLETON_INSTANCE
      value: "true"
EOF

# Check pod
kubectl get pod singleton-pattern -n pattern-test
# STATUS: Running
```

---

## 🎯 Lab Part 5: Pattern Best Practices (45 minutes)

### Exercise 5.1: Test Pattern Documentation

```bash
# Create documented pattern
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: documented-pattern
  namespace: pattern-test
  annotations:
    pattern.example.com/type: "Sidecar"
    pattern.example.com/description: "Sidecar pattern for logging"
    pattern.example.com/version: "1.0.0"
spec:
  containers:
  - name: main-app
    image: nginx
    ports:
    - containerPort: 80
  - name: sidecar
    image: busybox
    command: ['sh', '-c', 'while true; do echo "Documented sidecar"; sleep 30; done']
EOF

# Check pod
kubectl get pod documented-pattern -n pattern-test
# STATUS: Running
```

### Exercise 5.2: Test Pattern Validation

```bash
# Create validation pattern
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: validation-pattern
  namespace: pattern-test
spec:
  containers:
  - name: main-app
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
  - name: validator
    image: busybox
    command: ['sh', '-c', 'while true; do echo "Validation running"; sleep 30; done']
EOF

# Check pod
kubectl get pod validation-pattern -n pattern-test
# STATUS: Running
```

### Exercise 5.3: Test Pattern Monitoring

```bash
# Create monitoring pattern
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: monitoring-pattern
  namespace: pattern-test
spec:
  containers:
  - name: main-app
    image: nginx
    ports:
    - containerPort: 80
  - name: monitor
    image: busybox
    command: ['sh', '-c', 'while true; do echo "Monitoring running"; sleep 30; done']
EOF

# Check pod
kubectl get pod monitoring-pattern -n pattern-test
# STATUS: Running
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-17-reflections.md` and answer:

1. **What do advanced patterns do?**
   - How do they manage complex applications?
   - What happens when they fail?
   - Why are they critical for complex applications?

2. **What happens when patterns fail?**
   - How do you detect pattern failures?
   - What happens to applications?
   - How do you recover from pattern failures?

3. **How do you use advanced patterns?**
   - What's the difference between basic and advanced patterns?
   - How do you manage pattern complexity?
   - What are the best practices?

4. **How do you debug pattern issues?**
   - What tools do you use?
   - How do you check pattern status?
   - How do you test patterns?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Kubernetes pattern failures**
- **Advanced pattern issues**
- **Pattern complexity problems**
- **Pattern best practices**

### Exercise 6.3: Advanced Pattern Scenarios

```bash
# Create complex pattern scenario
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: complex-pattern
  namespace: pattern-test
  annotations:
    pattern.example.com/type: "Complex"
    pattern.example.com/description: "Complex pattern with multiple components"
    pattern.example.com/version: "1.0.0"
spec:
  initContainers:
  - name: init
    image: busybox
    command: ['sh', '-c', 'echo "Init container" && sleep 10']
  containers:
  - name: main-app
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
  - name: sidecar
    image: busybox
    command: ['sh', '-c', 'while true; do echo "Sidecar running"; sleep 30; done']
  - name: ambassador
    image: nginx
    ports:
    - containerPort: 8080
    command: ['sh', '-c', 'while true; do echo "Ambassador running"; sleep 30; done']
EOF

# Test the pattern
kubectl get pod complex-pattern -n pattern-test
# STATUS: Running
```

---

## 🏆 Success Criteria

You've successfully completed Day 17 if you can:

✅ **Watch advanced patterns work in real-time**  
✅ **Break advanced patterns and understand the impact**  
✅ **Test pattern failures**  
✅ **Practice pattern procedures**  
✅ **Understand the pattern architecture**  
✅ **Debug pattern issues**  

## 🚨 Common Issues & Solutions

### Issue: Patterns not working
```bash
# Check pattern status
kubectl get pods -l app=<pattern-name>

# Check pattern logs
kubectl logs <pod-name>

# Check pattern events
kubectl describe pod <pod-name>
```

### Issue: Custom resources not working
```bash
# Check CRD status
kubectl get crd <crd-name>

# Check custom resource status
kubectl get <custom-resource>

# Check custom resource events
kubectl describe <custom-resource>
```

### Issue: Operators not working
```bash
# Check operator status
kubectl get deployment <operator-name>

# Check operator logs
kubectl logs -l app=<operator-name>

# Check operator events
kubectl describe deployment <operator-name>
```

---

## 🎯 Tomorrow's Preview

**Day 18: Upgrades & Maintenance - The Maintenance Master**
- Watch upgrade components work
- Break upgrades and see clusters fail
- Test upgrade failures
- Practice upgrade procedures
- Understand the upgrade architecture

**Get ready for the maintenance master! 🔥**

---

## 📚 Additional Resources

- [Kubernetes Patterns](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Pattern Best Practices](https://kubernetes.io/docs/concepts/workloads/pods/)

**Remember: Advanced patterns are the pattern master of Kubernetes - they manage complex applications! 🎭**
