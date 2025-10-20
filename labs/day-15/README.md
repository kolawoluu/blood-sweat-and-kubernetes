# Day 15 Lab: Ingress & Traffic Management - The Traffic Controller

> **"Today you'll break ingress and watch traffic fail. You'll learn why ingress is the traffic controller of Kubernetes."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch ingress components work in real-time
- Break ingress and see traffic fail
- Test traffic management failures
- Practice ingress procedures
- Understand the ingress architecture
- Learn why ingress is critical for external access

## ⚠️ Prerequisites

- Day 14 lab completed
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
kubectl delete deployment chaos-test complex-production-app -n production
kubectl delete service chaos-test -n production
kubectl delete pod production-secure-pod -n production
kubectl delete networkpolicy production-netpol -n production
kubectl delete resourcequota production-quota -n production
kubectl delete role production-role -n production
kubectl delete rolebinding production-rolebinding -n production
kubectl delete serviceaccount production-sa -n production
```

### Step 2: Setup Ingress Environment

```bash
# Create ingress namespace
kubectl create namespace ingress-test

# Create ingress labels
kubectl label namespace ingress-test environment=test
kubectl label namespace ingress-test tier=ingress
```

### Step 3: Setup Monitoring

```bash
# Open 4 terminals
# Terminal 1: kubectl get pods --all-namespaces --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: kubectl get ingress --watch
# Terminal 4: Your command terminal
```

---

## 🧪 Lab Part 1: Watch Ingress Work (45 minutes)

### Exercise 1.1: Create Ingress Controller

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Check ingress controller
kubectl get pods -n ingress-nginx
# Should show ingress controller pods
```

### Exercise 1.2: Create Backend Services

```bash
# Create backend deployment
kubectl create deployment backend --image=nginx -n ingress-test --replicas=2

# Create backend service
kubectl expose deployment backend --port=80 -n ingress-test

# Check backend
kubectl get all -n ingress-test
```

### Exercise 1.3: Create Ingress Resource

```bash
# Create ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  namespace: ingress-test
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: test.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 80
EOF

# Check ingress
kubectl get ingress test-ingress -n ingress-test
# ADDRESS: Should show ingress IP
```

### Exercise 1.4: Test Ingress

```bash
# Get ingress IP
kubectl get ingress test-ingress -n ingress-test
# Note the ADDRESS

# Test ingress (replace with actual IP)
curl -H "Host: test.example.com" http://<ingress-ip>
# Should return nginx welcome page
```

---

## 💀 Lab Part 2: Break #17 - Stop Ingress Controller (1 hour)

> **"Time to break the ingress controller. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test workload
kubectl create deployment chaos-test --image=nginx -n ingress-test --replicas=2

# Create service
kubectl expose deployment chaos-test --port=80 -n ingress-test

# Create ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: chaos-ingress
  namespace: ingress-test
spec:
  rules:
  - host: chaos.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: chaos-test
            port:
              number: 80
EOF

# Verify everything is working
kubectl get pods -l app=chaos-test -n ingress-test
kubectl get svc chaos-test -n ingress-test
kubectl get ingress chaos-ingress -n ingress-test
```

### Exercise 2.2: Break Ingress Controller

```bash
# Delete ingress controller
kubectl delete deployment ingress-nginx-controller -n ingress-nginx

# Check ingress controller
kubectl get pods -n ingress-nginx
# Should show: No resources found
```

### Exercise 2.3: Observe the Chaos

**Try to access ingress:**
```bash
# Try to get ingress
kubectl get ingress chaos-ingress -n ingress-test
# ADDRESS: Should be empty

# Try to access ingress
curl -H "Host: chaos.example.com" http://<ingress-ip>
# Should fail - no ingress controller
```

### Exercise 2.4: Debug the Problem

```bash
# Check ingress controller
kubectl get pods -n ingress-nginx
# NOT FOUND!

# Check ingress status
kubectl describe ingress chaos-ingress -n ingress-test
# Should show: No ingress controller
```

**Key Learning:** 
- Ingress controller is critical for external access
- No controller = no external traffic
- Services still work internally
- External access completely broken

### Exercise 2.5: Fix It

```bash
# Reinstall ingress controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Wait for ingress controller to be ready
kubectl get pods -n ingress-nginx
# Should show ingress controller pods

# Check ingress
kubectl get ingress chaos-ingress -n ingress-test
# ADDRESS: Should show ingress IP
```

**Real-world parallel:** This happens when:
- Ingress controller crashes
- Resource exhaustion
- Configuration errors
- Network issues

---

## 🔥 Lab Part 3: Traffic Management Failures (1 hour)

> **"Time to test what happens when traffic management fails. This is where ingress resilience is tested."**

### Exercise 3.1: Test Load Balancing

```bash
# Create multiple backend services
kubectl create deployment backend-1 --image=nginx -n ingress-test --replicas=2
kubectl create deployment backend-2 --image=nginx -n ingress-test --replicas=2

# Create services
kubectl expose deployment backend-1 --port=80 -n ingress-test
kubectl expose deployment backend-2 --port=80 -n ingress-test

# Create ingress with multiple backends
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: load-balance-ingress
  namespace: ingress-test
spec:
  rules:
  - host: lb.example.com
    http:
      paths:
      - path: /backend-1
        pathType: Prefix
        backend:
          service:
            name: backend-1
            port:
              number: 80
      - path: /backend-2
        pathType: Prefix
        backend:
          service:
            name: backend-2
            port:
              number: 80
EOF

# Check ingress
kubectl get ingress load-balance-ingress -n ingress-test
```

### Exercise 3.2: Test SSL Termination

```bash
# Create TLS secret
kubectl create secret tls test-tls --cert=/dev/null --key=/dev/null -n ingress-test

# Create ingress with TLS
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: ingress-test
spec:
  tls:
  - hosts:
    - tls.example.com
    secretName: test-tls
  rules:
  - host: tls.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 80
EOF

# Check ingress
kubectl get ingress tls-ingress -n ingress-test
```

### Exercise 3.3: Test Path-based Routing

```bash
# Create ingress with path-based routing
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
  namespace: ingress-test
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: path.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-1
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: backend-2
            port:
              number: 80
EOF

# Check ingress
kubectl get ingress path-ingress -n ingress-test
```

### Exercise 3.4: Test Host-based Routing

```bash
# Create ingress with host-based routing
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
  namespace: ingress-test
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-1
            port:
              number: 80
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-2
            port:
              number: 80
EOF

# Check ingress
kubectl get ingress host-ingress -n ingress-test
```

---

## 🧠 Lab Part 4: Advanced Ingress Scenarios (1 hour)

> **"Time to test advanced ingress scenarios. This is where you become an ingress expert."**

### Exercise 4.1: Test Ingress Annotations

```bash
# Create ingress with annotations
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: annotation-ingress
  namespace: ingress-test
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "false"
    nginx.ingress.kubernetes.io/proxy-body-size: "1m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
spec:
  rules:
  - host: annotation.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 80
EOF

# Check ingress
kubectl get ingress annotation-ingress -n ingress-test
```

### Exercise 4.2: Test Ingress Rate Limiting

```bash
# Create ingress with rate limiting
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-limit-ingress
  namespace: ingress-test
  annotations:
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  rules:
  - host: rate.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 80
EOF

# Check ingress
kubectl get ingress rate-limit-ingress -n ingress-test
```

### Exercise 4.3: Test Ingress Authentication

```bash
# Create ingress with authentication
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-ingress
  namespace: ingress-test
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required'
spec:
  rules:
  - host: auth.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 80
EOF

# Check ingress
kubectl get ingress auth-ingress -n ingress-test
```

### Exercise 4.4: Test Ingress Monitoring

```bash
# Install ingress monitoring
kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml

# Check monitoring
kubectl get pods -n monitoring
# Should show monitoring pods
```

---

## 🎯 Lab Part 5: Advanced Traffic Management (45 minutes)

### Exercise 5.1: Test Ingress Canary

```bash
# Create canary deployment
kubectl create deployment backend-canary --image=nginx:1.21 -n ingress-test --replicas=1

# Create canary service
kubectl expose deployment backend-canary --port=80 -n ingress-test

# Create canary ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress
  namespace: ingress-test
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
  - host: canary.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-canary
            port:
              number: 80
EOF

# Check ingress
kubectl get ingress canary-ingress -n ingress-test
```

### Exercise 5.2: Test Ingress Blue-Green

```bash
# Create blue deployment
kubectl create deployment backend-blue --image=nginx:1.20 -n ingress-test --replicas=2

# Create green deployment
kubectl create deployment backend-green --image=nginx:1.21 -n ingress-test --replicas=2

# Create services
kubectl expose deployment backend-blue --port=80 -n ingress-test
kubectl expose deployment backend-green --port=80 -n ingress-test

# Create blue-green ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blue-green-ingress
  namespace: ingress-test
spec:
  rules:
  - host: blue-green.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-blue
            port:
              number: 80
EOF

# Check ingress
kubectl get ingress blue-green-ingress -n ingress-test
```

### Exercise 5.3: Test Ingress A/B Testing

```bash
# Create A/B testing ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ab-testing-ingress
  namespace: ingress-test
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Testing"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"
spec:
  rules:
  - host: ab.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-green
            port:
              number: 80
EOF

# Check ingress
kubectl get ingress ab-testing-ingress -n ingress-test
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-15-reflections.md` and answer:

1. **What does ingress do?**
   - How does it manage external traffic?
   - What happens when it fails?
   - Why is it critical for external access?

2. **What happens when ingress controller fails?**
   - How do you detect ingress failures?
   - What happens to external traffic?
   - How do you recover from ingress failures?

3. **How do you use ingress?**
   - What's the difference between ingress and services?
   - How do you manage traffic routing?
   - What are the best practices?

4. **How do you debug ingress issues?**
   - What tools do you use?
   - How do you check ingress status?
   - How do you test ingress?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Kubernetes ingress failures**
- **Traffic management issues**
- **Load balancing problems**
- **SSL termination failures**

### Exercise 6.3: Advanced Ingress Scenarios

```bash
# Create complex ingress scenario
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: complex-ingress
  namespace: ingress-test
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
  - host: complex.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 80
EOF

# Test the ingress
kubectl get ingress complex-ingress -n ingress-test
```

---

## 🏆 Success Criteria

You've successfully completed Day 15 if you can:

✅ **Watch ingress components work in real-time**  
✅ **Break ingress and understand the impact**  
✅ **Test traffic management failures**  
✅ **Practice ingress procedures**  
✅ **Understand the ingress architecture**  
✅ **Debug ingress issues**  

## 🚨 Common Issues & Solutions

### Issue: Ingress not accessible
```bash
# Check ingress controller
kubectl get pods -n ingress-nginx

# Check ingress status
kubectl get ingress <name>

# Check ingress events
kubectl describe ingress <name>
```

### Issue: Traffic not routing
```bash
# Check ingress rules
kubectl get ingress <name> -o yaml

# Check backend services
kubectl get svc

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

### Issue: SSL not working
```bash
# Check TLS secrets
kubectl get secrets

# Check ingress TLS configuration
kubectl describe ingress <name>
```

---

## 🎯 Tomorrow's Preview

**Day 16: Configuration Management - The Config Master**
- Watch configuration components work
- Break configuration and see apps fail
- Test configuration failures
- Practice configuration procedures
- Understand the configuration architecture

**Get ready for the config master! 🔥**

---

## 📚 Additional Resources

- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [Traffic Management](https://kubernetes.io/docs/concepts/services-networking/ingress/#traffic-routing)
- [SSL Termination](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls)

**Remember: Ingress is the traffic controller of Kubernetes - it manages all external traffic! 🚦**
