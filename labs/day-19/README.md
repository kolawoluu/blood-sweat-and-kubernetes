# Day 19 Lab: Service Mesh - The Mesh Master

> **"Today you'll break service mesh and watch traffic fail. You'll learn why service mesh is the mesh master of Kubernetes."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch service mesh components work in real-time
- Break service mesh and see traffic fail
- Test mesh failures
- Practice mesh procedures
- Understand the mesh architecture
- Learn why service mesh is critical for microservices

## ⚠️ Prerequisites

- Day 18 lab completed
- Minikube cluster running
- 4-5 hours of uninterrupted time
- Understanding of microservices concepts

## 🛠️ Setup (30 minutes)

### Step 1: Verify Cluster Status

```bash
# Check cluster is running
kubectl get nodes
kubectl get pods -n kube-system

# Clean up previous lab resources
kubectl delete deployment test-app chaos-test maintenance-test uncordon-test blue green canary ab-test rollback-test maintenance-window complex-upgrade -n upgrade-test
kubectl delete service test-app chaos-test maintenance-test uncordon-test blue green canary ab-test rollback-test maintenance-window -n upgrade-test
kubectl delete configmap maintenance-docs -n upgrade-test
kubectl delete namespace upgrade-test
```

### Step 2: Setup Service Mesh Environment

```bash
# Create service mesh namespace
kubectl create namespace mesh-test

# Create service mesh labels
kubectl label namespace mesh-test environment=test
kubectl label namespace mesh-test tier=mesh
```

### Step 3: Setup Monitoring

```bash
# Open 4 terminals
# Terminal 1: kubectl get pods --all-namespaces --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: kubectl get virtualservices,destinationrules --watch
# Terminal 4: Your command terminal
```

---

## 🧪 Lab Part 1: Watch Service Mesh Work (45 minutes)

### Exercise 1.1: Install Istio

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install Istio
istioctl install --set values.defaultRevision=default

# Check Istio installation
kubectl get pods -n istio-system
# Should show Istio components
```

### Exercise 1.2: Enable Istio Injection

```bash
# Enable Istio injection for namespace
kubectl label namespace mesh-test istio-injection=enabled

# Check namespace labels
kubectl get namespace mesh-test --show-labels
# Should show: istio-injection=enabled
```

### Exercise 1.3: Create Microservices

```bash
# Create frontend service
kubectl create deployment frontend --image=nginx -n mesh-test --replicas=2

# Create backend service
kubectl create deployment backend --image=nginx -n mesh-test --replicas=2

# Create services
kubectl expose deployment frontend --port=80 -n mesh-test
kubectl expose deployment backend --port=80 -n mesh-test

# Check services
kubectl get all -n mesh-test
```

### Exercise 1.4: Test Service Mesh

```bash
# Check pods
kubectl get pods -n mesh-test
# Should show Istio sidecar containers

# Check Istio sidecar
kubectl describe pod <pod-name> -n mesh-test
# Should show: istio-proxy container
```

---

## 💀 Lab Part 2: Break #21 - Stop Service Mesh (1 hour)

> **"Time to break the service mesh. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test workload
kubectl create deployment chaos-test --image=nginx -n mesh-test --replicas=2

# Create service
kubectl expose deployment chaos-test --port=80 -n mesh-test

# Create VirtualService
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: chaos-vs
  namespace: mesh-test
spec:
  hosts:
  - chaos-test
  http:
  - route:
    - destination:
        host: chaos-test
        port:
          number: 80
EOF

# Verify everything is working
kubectl get pods -l app=chaos-test -n mesh-test
kubectl get svc chaos-test -n mesh-test
kubectl get virtualservice chaos-vs -n mesh-test
```

### Exercise 2.2: Break Service Mesh

```bash
# Delete Istio control plane
kubectl delete deployment istiod -n istio-system

# Check Istio status
kubectl get pods -n istio-system
# Should show: No resources found
```

### Exercise 2.3: Observe the Chaos

**Check service mesh status:**
```bash
# Check Istio components
kubectl get pods -n istio-system
# Should show: No resources found

# Check VirtualService
kubectl get virtualservice chaos-vs -n mesh-test
# Should show: No resources found

# Check pods
kubectl get pods -n mesh-test
# Should show pods without Istio sidecar
```

### Exercise 2.4: Debug the Problem

```bash
# Check Istio status
kubectl get pods -n istio-system
# Should show: No resources found

# Check VirtualService
kubectl get virtualservice chaos-vs -n mesh-test
# Should show: No resources found

# Check events
kubectl get events -n mesh-test
# Should show: No Istio events
```

**Key Learning:** 
- Service mesh is critical for microservices communication
- No mesh = no advanced traffic management
- Services still work but no mesh features
- No traffic routing, security, or observability

### Exercise 2.5: Fix It

```bash
# Reinstall Istio
istioctl install --set values.defaultRevision=default

# Wait for Istio to be ready
kubectl get pods -n istio-system
# Should show Istio components

# Check VirtualService
kubectl get virtualservice chaos-vs -n mesh-test
# Should show: VirtualService
```

**Real-world parallel:** This happens when:
- Service mesh crashes
- Resource exhaustion
- Configuration errors
- Network issues

---

## 🔥 Lab Part 3: Mesh Failures (1 hour)

> **"Time to test what happens when service mesh fails. This is where mesh resilience is tested."**

### Exercise 3.1: Test Traffic Routing

```bash
# Create traffic routing
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: traffic-routing
  namespace: mesh-test
spec:
  hosts:
  - frontend
  http:
  - match:
    - headers:
        version:
          exact: v1
    route:
    - destination:
        host: frontend
        port:
          number: 80
  - match:
    - headers:
        version:
          exact: v2
    route:
    - destination:
        host: backend
        port:
          number: 80
EOF

# Check VirtualService
kubectl get virtualservice traffic-routing -n mesh-test
```

### Exercise 3.2: Test Load Balancing

```bash
# Create load balancing
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: load-balancing
  namespace: mesh-test
spec:
  host: frontend
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
EOF

# Check DestinationRule
kubectl get destinationrule load-balancing -n mesh-test
```

### Exercise 3.3: Test Circuit Breaker

```bash
# Create circuit breaker
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: circuit-breaker
  namespace: mesh-test
spec:
  host: backend
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 10
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutiveErrors: 3
      interval: 30s
      baseEjectionTime: 30s
EOF

# Check DestinationRule
kubectl get destinationrule circuit-breaker -n mesh-test
```

### Exercise 3.4: Test Security

```bash
# Create security policy
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: security-policy
  namespace: mesh-test
spec:
  selector:
    matchLabels:
      app: frontend
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/mesh-test/sa/frontend"]
    to:
    - operation:
        methods: ["GET"]
EOF

# Check AuthorizationPolicy
kubectl get authorizationpolicy security-policy -n mesh-test
```

---

## 🧠 Lab Part 4: Advanced Mesh Scenarios (1 hour)

> **"Time to test advanced mesh scenarios. This is where you become a mesh expert."**

### Exercise 4.1: Test Canary Deployment

```bash
# Create canary deployment
kubectl create deployment frontend-canary --image=nginx:1.21 -n mesh-test --replicas=1

# Create canary service
kubectl expose deployment frontend-canary --port=80 -n mesh-test

# Create canary routing
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: canary-routing
  namespace: mesh-test
spec:
  hosts:
  - frontend
  http:
  - route:
    - destination:
        host: frontend
        port:
          number: 80
      weight: 90
    - destination:
        host: frontend-canary
        port:
          number: 80
      weight: 10
EOF

# Check VirtualService
kubectl get virtualservice canary-routing -n mesh-test
```

### Exercise 4.2: Test A/B Testing

```bash
# Create A/B testing
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ab-testing
  namespace: mesh-test
spec:
  hosts:
  - frontend
  http:
  - match:
    - headers:
        user-agent:
          regex: ".*Chrome.*"
    route:
    - destination:
        host: frontend
        port:
          number: 80
  - match:
    - headers:
        user-agent:
          regex: ".*Firefox.*"
    route:
    - destination:
        host: frontend-canary
        port:
          number: 80
EOF

# Check VirtualService
kubectl get virtualservice ab-testing -n mesh-test
```

### Exercise 4.3: Test Observability

```bash
# Install Istio observability
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.19/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.19/samples/addons/grafana.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.19/samples/addons/jaeger.yaml

# Check observability
kubectl get pods -n istio-system
# Should show: prometheus, grafana, jaeger
```

### Exercise 4.4: Test Mesh Monitoring

```bash
# Check mesh metrics
kubectl get pods -n istio-system | grep prometheus

# Check mesh dashboards
kubectl get pods -n istio-system | grep grafana

# Check mesh tracing
kubectl get pods -n istio-system | grep jaeger
```

---

## 🎯 Lab Part 5: Mesh Best Practices (45 minutes)

### Exercise 5.1: Test Mesh Security

```bash
# Create mesh security
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: mesh-security
  namespace: mesh-test
spec:
  mtls:
    mode: STRICT
EOF

# Check PeerAuthentication
kubectl get peerauthentication mesh-security -n mesh-test
```

### Exercise 5.2: Test Mesh Performance

```bash
# Create mesh performance
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: mesh-performance
  namespace: mesh-test
spec:
  host: frontend
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
EOF

# Check DestinationRule
kubectl get destinationrule mesh-performance -n mesh-test
```

### Exercise 5.3: Test Mesh Documentation

```bash
# Create mesh documentation
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: mesh-docs
  namespace: mesh-test
data:
  mesh.md: |
    # Service Mesh Procedures
    
    ## Traffic Management
    1. Create VirtualService
    2. Create DestinationRule
    3. Test traffic routing
    4. Monitor traffic
    
    ## Security
    1. Enable mTLS
    2. Create AuthorizationPolicy
    3. Test security
    4. Monitor security
    
    ## Observability
    1. Enable metrics
    2. Enable tracing
    3. Enable logging
    4. Monitor observability
EOF

# Check documentation
kubectl get configmap mesh-docs -n mesh-test
kubectl describe configmap mesh-docs -n mesh-test
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-19-reflections.md` and answer:

1. **What does service mesh do?**
   - How does it manage microservices communication?
   - What happens when it fails?
   - Why is it critical for microservices?

2. **What happens when service mesh fails?**
   - How do you detect mesh failures?
   - What happens to microservices?
   - How do you recover from mesh failures?

3. **How do you use service mesh?**
   - What's the difference between service mesh and services?
   - How do you manage mesh complexity?
   - What are the best practices?

4. **How do you debug mesh issues?**
   - What tools do you use?
   - How do you check mesh status?
   - How do you test mesh?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Kubernetes service mesh failures**
- **Mesh complexity issues**
- **Traffic management problems**
- **Mesh security breaches**

### Exercise 6.3: Advanced Mesh Scenarios

```bash
# Create complex mesh scenario
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: complex-mesh
  namespace: mesh-test
spec:
  hosts:
  - frontend
  http:
  - match:
    - headers:
        version:
          exact: v1
    route:
    - destination:
        host: frontend
        port:
          number: 80
      weight: 90
    - destination:
        host: frontend-canary
        port:
          number: 80
      weight: 10
  - match:
    - headers:
        version:
          exact: v2
    route:
    - destination:
        host: backend
        port:
          number: 80
EOF

# Test the mesh
kubectl get virtualservice complex-mesh -n mesh-test
```

---

## 🏆 Success Criteria

You've successfully completed Day 19 if you can:

✅ **Watch service mesh components work in real-time**  
✅ **Break service mesh and understand the impact**  
✅ **Test mesh failures**  
✅ **Practice mesh procedures**  
✅ **Understand the mesh architecture**  
✅ **Debug mesh issues**  

## 🚨 Common Issues & Solutions

### Issue: Service mesh not working
```bash
# Check Istio status
kubectl get pods -n istio-system

# Check mesh components
kubectl get virtualservices,destinationrules

# Check mesh logs
kubectl logs -n istio-system -l app=istiod
```

### Issue: Traffic not routing
```bash
# Check VirtualService
kubectl get virtualservice <name>

# Check DestinationRule
kubectl get destinationrule <name>

# Check mesh events
kubectl get events -n mesh-test
```

### Issue: Mesh security not working
```bash
# Check PeerAuthentication
kubectl get peerauthentication

# Check AuthorizationPolicy
kubectl get authorizationpolicy

# Check mesh security logs
kubectl logs -n istio-system -l app=istiod
```

---

## 🎯 Congratulations!

**You've completed the Blood, Sweat, and Kubernetes course!**

You've successfully:
- ✅ Broken every major Kubernetes component
- ✅ Fixed every disaster scenario
- ✅ Learned from real-world incidents
- ✅ Built muscle memory through pain
- ✅ Become a Kubernetes expert

**You are now ready for production! 🏆**

---

## 📚 Additional Resources

- [Istio Documentation](https://istio.io/docs/)
- [Service Mesh](https://kubernetes.io/docs/concepts/services-networking/service-mesh/)
- [Traffic Management](https://istio.io/docs/tasks/traffic-management/)
- [Security](https://istio.io/docs/tasks/security/)

**Remember: Service mesh is the mesh master of Kubernetes - it connects all microservices! 🕸️**
