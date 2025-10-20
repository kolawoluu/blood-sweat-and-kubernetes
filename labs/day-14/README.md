# Day 14 Lab: Production Readiness - The Final Boss

> **"Today you'll break production and watch everything fail. You'll learn why production readiness is the final boss of Kubernetes."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch production components work in real-time
- Break production and see everything fail
- Test production failures and recovery
- Practice production procedures
- Understand the production architecture
- Learn why production readiness is critical for real-world deployments

## ⚠️ Prerequisites

- Day 13 lab completed
- Minikube cluster running
- 4-5 hours of uninterrupted time
- Understanding of all previous concepts

## 🛠️ Setup (30 minutes)

### Step 1: Verify Cluster Status

```bash
# Check cluster is running
kubectl get nodes
kubectl get pods -n kube-system

# Clean up previous lab resources
kubectl delete pod multi-cluster-test -n multi-cluster-test
kubectl delete deployment chaos-test cross-cluster-backend -n multi-cluster-test
kubectl delete service chaos-test cross-cluster -n multi-cluster-test
kubectl delete namespace multi-cluster-test
```

### Step 2: Setup Production Environment

```bash
# Create production namespace
kubectl create namespace production

# Create production labels
kubectl label namespace production environment=production
kubectl label namespace production tier=production
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

## 🧪 Lab Part 1: Watch Production Work (45 minutes)

### Exercise 1.1: Create Production Deployment

```bash
# Create production deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: production-app
  template:
    metadata:
      labels:
        app: production-app
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

# Check deployment
kubectl get deployment production-app -n production
# READY: 3/3
```

### Exercise 1.2: Create Production Service

```bash
# Create production service
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: production-service
  namespace: production
spec:
  selector:
    app: production-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

# Check service
kubectl get svc production-service -n production
# TYPE: ClusterIP
```

### Exercise 1.3: Create Production ConfigMap

```bash
# Create production config
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: production-config
  namespace: production
data:
  app.properties: |
    app.name=Production App
    app.version=1.0.0
    app.environment=production
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
EOF

# Check configmap
kubectl get configmap production-config -n production
```

### Exercise 1.4: Create Production Secret

```bash
# Create production secret
kubectl create secret generic production-secret \
  --from-literal=db-password=super-secret-password \
  --from-literal=api-key=production-api-key \
  -n production

# Check secret
kubectl get secret production-secret -n production
```

---

## 💀 Lab Part 2: Break #16 - Stop Production (1 hour)

> **"Time to break production. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test workload
kubectl create deployment chaos-test --image=nginx -n production --replicas=2

# Create service
kubectl expose deployment chaos-test --port=80 -n production

# Verify everything is working
kubectl get pods -l app=chaos-test -n production
kubectl get svc chaos-test -n production
```

### Exercise 2.2: Break Production

```bash
# Stop all nodes
minikube stop

# Check cluster status
kubectl get nodes
# Error: The connection to the server was refused
```

### Exercise 2.3: Observe the Chaos

**Try to access production:**
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
minikube status
# Should show: Stopped

# Check cluster logs
minikube logs
# Should show cluster shutdown logs
```

**Key Learning:** 
- Production is completely down
- No access to any resources
- All workloads unavailable
- Complete service outage

### Exercise 2.5: Fix It

```bash
# Start cluster
minikube start

# Wait for cluster to be ready
kubectl get nodes
# Should show 3 nodes

# Check production resources
kubectl get all -n production
# Should show all resources
```

**Real-world parallel:** This happens when:
- Cluster crashes
- Network partitions
- Hardware failures
- Configuration errors

---

## 🔥 Lab Part 3: Production Failures (1 hour)

> **"Time to test what happens when production fails. This is where production resilience is tested."**

### Exercise 3.1: Test Production Scaling

```bash
# Scale production deployment
kubectl scale deployment production-app --replicas=5 -n production

# Check scaling
kubectl get pods -l app=production-app -n production
# Should show 5 pods
```

### Exercise 3.2: Test Production Updates

```bash
# Update production image
kubectl set image deployment/production-app nginx=nginx:1.21 -n production

# Watch rolling update
kubectl get pods -l app=production-app -n production --watch

# Pods should be updated
```

### Exercise 3.3: Test Production Rollback

```bash
# Rollback production
kubectl rollout undo deployment/production-app -n production

# Watch rollback
kubectl get pods -l app=production-app -n production --watch

# Pods should be rolled back
```

### Exercise 3.4: Test Production Monitoring

```bash
# Install production monitoring
kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml

# Check monitoring
kubectl get pods -n monitoring
# Should show monitoring pods
```

---

## 🧠 Lab Part 4: Production Security (1 hour)

> **"Time to test production security. This is where you learn about production hardening."**

### Exercise 4.1: Test Production RBAC

```bash
# Create production service account
kubectl create serviceaccount production-sa -n production

# Create production role
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: production-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch"]
EOF

# Create role binding
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: production-rolebinding
  namespace: production
subjects:
- kind: ServiceAccount
  name: production-sa
  namespace: production
roleRef:
  kind: Role
  name: production-role
  apiGroup: rbac.authorization.k8s.io
EOF

# Test RBAC
kubectl auth can-i get pods --as=system:serviceaccount:production:production-sa -n production
# Should return: yes
```

### Exercise 4.2: Test Production Network Policies

```bash
# Create production network policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: production-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: production-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: production-app
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: production-app
    ports:
    - protocol: TCP
      port: 80
EOF

# Check network policy
kubectl get networkpolicy production-netpol -n production
```

### Exercise 4.3: Test Production Pod Security

```bash
# Create production pod with security context
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: production-secure-pod
  namespace: production
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
EOF

# Check pod status
kubectl get pod production-secure-pod -n production
# STATUS: Running
```

### Exercise 4.4: Test Production Resource Limits

```bash
# Create production resource quota
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "10"
EOF

# Check resource quota
kubectl get resourcequota production-quota -n production
```

---

## 🎯 Lab Part 5: Advanced Production Scenarios (45 minutes)

### Exercise 5.1: Test Production Backup

```bash
# Backup production resources
kubectl get all -n production -o yaml > production-backup.yaml

# Check backup
ls -la production-backup.yaml
# Should show backup file
```

### Exercise 5.2: Test Production Disaster Recovery

```bash
# Delete production resources
kubectl delete all --all -n production

# Check resources
kubectl get all -n production
# Should show: No resources found

# Restore from backup
kubectl apply -f production-backup.yaml

# Check resources
kubectl get all -n production
# Should show all resources
```

### Exercise 5.3: Test Production Monitoring

```bash
# Install production monitoring
kubectl apply -f https://raw.githubusercontent.com/grafana/grafana/main/deploy/kubernetes/grafana.yaml

# Check monitoring
kubectl get pods -n grafana
# Should show monitoring pods
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-14-reflections.md` and answer:

1. **What does production readiness mean?**
   - How do you prepare for production?
   - What happens when production fails?
   - Why is it critical for real-world deployments?

2. **What happens when production fails?**
   - How do you detect production failures?
   - What happens to workloads?
   - How do you recover from production failures?

3. **How do you secure production?**
   - What security measures should you implement?
   - How do you harden production?
   - What are the best practices?

4. **How do you monitor production?**
   - What tools do you use?
   - How do you check production status?
   - How do you test production?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Kubernetes production failures**
- **Production security breaches**
- **Production monitoring failures**
- **Production best practices**

### Exercise 6.3: Advanced Production Scenarios

```bash
# Create complex production scenario
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: complex-production-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: complex-production-app
  template:
    metadata:
      labels:
        app: complex-production-app
    spec:
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
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
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/conf.d
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: production-config
EOF

# Test the production setup
kubectl get deployment complex-production-app -n production
# READY: 3/3
```

---

## 🏆 Success Criteria

You've successfully completed Day 14 if you can:

✅ **Watch production components work in real-time**  
✅ **Break production and understand the impact**  
✅ **Test production failures and recovery**  
✅ **Practice production procedures**  
✅ **Understand the production architecture**  
✅ **Debug production issues**  

## 🚨 Common Issues & Solutions

### Issue: Production not accessible
```bash
# Check cluster status
kubectl get nodes

# Check cluster logs
minikube logs

# Restart cluster
minikube start
```

### Issue: Production security not working
```bash
# Check RBAC status
kubectl get roles,rolebindings

# Check network policies
kubectl get networkpolicies

# Check pod security
kubectl describe pod <pod-name>
```

### Issue: Production monitoring not working
```bash
# Check monitoring components
kubectl get pods -n monitoring

# Check monitoring logs
kubectl logs -n monitoring -l app=prometheus
```

---

## 🎯 Tomorrow's Preview

**Day 15: Ingress & Traffic Management - The Traffic Controller**
- Watch ingress components work
- Break ingress and see traffic fail
- Test traffic management failures
- Practice ingress procedures
- Understand the ingress architecture

**Get ready for the traffic controller! 🔥**

---

## 📚 Additional Resources

- [Kubernetes Production](https://kubernetes.io/docs/concepts/cluster-administration/)
- [Production Security](https://kubernetes.io/docs/concepts/security/)
- [Production Monitoring](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [Production Best Practices](https://kubernetes.io/docs/concepts/cluster-administration/)

**Remember: Production readiness is the final boss of Kubernetes - it's where everything comes together! 🏆**
