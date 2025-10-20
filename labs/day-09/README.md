# Day 9 Lab: Observability - The Nervous System

> **"Today you'll break monitoring and watch your cluster become blind. You'll learn why observability is the nervous system of Kubernetes."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch monitoring components work in real-time
- Break monitoring and see metrics disappear
- Test logging and tracing failures
- Practice observability procedures
- Understand the observability architecture
- Learn why observability is critical for production clusters

## ⚠️ Prerequisites

- Day 8 lab completed
- Minikube cluster running
- 4-5 hours of uninterrupted time
- Understanding of monitoring concepts

## 🛠️ Setup (30 minutes)

### Step 1: Verify Cluster Status

```bash
# Check cluster is running
kubectl get nodes
kubectl get pods -n kube-system

# Clean up previous lab resources
kubectl delete pod rbac-test security-context-test secure-pod vulnerable-pod psp-test complex-security-pod invalid-security-pod
kubectl delete deployment chaos-test
kubectl delete service chaos-test
kubectl delete networkpolicy deny-all allow-web-to-backend
kubectl delete podsecuritypolicy restrictive-psp
kubectl delete namespace security-test
```

### Step 2: Setup Monitoring

```bash
# Open 4 terminals
# Terminal 1: kubectl get pods --all-namespaces --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: kubectl get services --watch
# Terminal 4: Your command terminal
```

### Step 3: Install Monitoring Stack

```bash
# Install Prometheus
kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml

# Install Grafana
kubectl apply -f https://raw.githubusercontent.com/grafana/grafana/main/deploy/kubernetes/grafana.yaml

# Check monitoring components
kubectl get pods -n monitoring
kubectl get pods -n grafana
```

---

## 🧪 Lab Part 1: Watch Monitoring Work (45 minutes)

### Exercise 1.1: Create Test Workload

```bash
# Create namespace for testing
kubectl create namespace observability-test

# Create deployment
kubectl create deployment web --image=nginx -n observability-test --replicas=3

# Create service
kubectl expose deployment web --port=80 -n observability-test

# Check deployment
kubectl get pods -l app=web -n observability-test
kubectl get svc web -n observability-test
```

### Exercise 1.2: Test Metrics Collection

```bash
# Check if metrics are being collected
kubectl top nodes
kubectl top pods -n observability-test

# If metrics not available, install metrics server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Wait for metrics server to be ready
kubectl wait --for=condition=ready pod -l k8s-app=metrics-server -n kube-system

# Check metrics again
kubectl top nodes
kubectl top pods -n observability-test
```

### Exercise 1.3: Test Logging

```bash
# Check pod logs
kubectl logs -l app=web -n observability-test

# Check specific pod logs
kubectl logs <pod-name> -n observability-test

# Follow logs in real-time
kubectl logs -f <pod-name> -n observability-test
# Exit with Ctrl+C
```

### Exercise 1.4: Test Events

```bash
# Check events
kubectl get events -n observability-test

# Check events for specific pod
kubectl describe pod <pod-name> -n observability-test

# Watch events in real-time
kubectl get events -n observability-test --watch
# Exit with Ctrl+C
```

---

## 💀 Lab Part 2: Break #11 - Stop Monitoring (1 hour)

> **"Time to break the monitoring. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test deployment
kubectl create deployment chaos-test --image=nginx -n observability-test --replicas=2

# Create service
kubectl expose deployment chaos-test --port=80 -n observability-test

# Verify everything is working
kubectl get pods -l app=chaos-test -n observability-test
kubectl get svc chaos-test -n observability-test
```

### Exercise 2.2: Break Metrics Server

```bash
# Delete metrics server
kubectl delete deployment metrics-server -n kube-system

# Check metrics
kubectl top nodes
# Error: metrics not available

kubectl top pods -n observability-test
# Error: metrics not available
```

### Exercise 2.3: Observe the Chaos

**Watch Terminal 1 (pods):**
- Pods keep running
- No metrics available
- No resource monitoring

**Test monitoring:**
```bash
# Try to get metrics
kubectl top nodes
# Error: metrics not available

# Try to get pod metrics
kubectl top pods -n observability-test
# Error: metrics not available

# Check if pods are still running
kubectl get pods -n observability-test
# All pods: Running
```

### Exercise 2.4: Debug the Problem

```bash
# Check metrics server
kubectl get pods -n kube-system | grep metrics-server
# NOT FOUND!

# Check if metrics server is running
kubectl get deployment metrics-server -n kube-system
# Error: not found

# Check API server logs
kubectl logs kube-apiserver-minikube -n kube-system
# Should show metrics server connection errors
```

**Key Learning:** 
- Metrics server is critical for resource monitoring
- No metrics = no resource visibility
- Pods keep running but no monitoring
- Cluster becomes "blind" to resource usage

### Exercise 2.5: Fix It

```bash
# Reinstall metrics server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Wait for metrics server to be ready
kubectl wait --for=condition=ready pod -l k8s-app=metrics-server -n kube-system

# Check metrics
kubectl top nodes
# Should work again

kubectl top pods -n observability-test
# Should work
```

**Real-world parallel:** This happens when:
- Metrics server crashes
- Resource exhaustion
- Configuration errors
- Network issues

---

## 🔥 Lab Part 3: Logging Failures (1 hour)

> **"Time to test what happens when logging fails. This is where observability resilience is tested."**

### Exercise 3.1: Test Logging

```bash
# Check pod logs
kubectl logs -l app=chaos-test -n observability-test

# Check specific pod logs
kubectl logs <pod-name> -n observability-test

# Follow logs in real-time
kubectl logs -f <pod-name> -n observability-test
# Exit with Ctrl+C
```

### Exercise 3.2: Break Logging

```bash
# Stop kubelet on a node (this will break logging)
minikube ssh -n minikube-m02
sudo systemctl stop kubelet
exit

# Check pod logs
kubectl logs <pod-name> -n observability-test
# May fail or show old logs

# Check node status
kubectl get nodes
# minikube-m02   NotReady
```

### Exercise 3.3: Test Logging Recovery

```bash
# Restart kubelet
minikube ssh -n minikube-m02
sudo systemctl start kubelet
exit

# Wait for node to recover
kubectl get nodes
# minikube-m02   Ready

# Check pod logs
kubectl logs <pod-name> -n observability-test
# Should work again
```

### Exercise 3.4: Test Log Aggregation

```bash
# Install log aggregation
kubectl apply -f https://raw.githubusercontent.com/fluent/fluentd-kubernetes-daemonset/master/fluentd-daemonset.yaml

# Check log aggregation
kubectl get pods -n kube-system | grep fluentd

# Check logs
kubectl logs -l app=fluentd -n kube-system
```

**Real-world parallel:** This happens when:
- Logging system crashes
- Storage issues
- Network problems
- Resource exhaustion

---

## 🧠 Lab Part 4: Tracing and APM (1 hour)

> **"Time to test distributed tracing. This is where you learn about application performance monitoring."**

### Exercise 4.1: Install Tracing

```bash
# Install Jaeger
kubectl apply -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/main/deploy/kubernetes/jaeger-operator.yaml

# Check Jaeger
kubectl get pods -n jaeger
kubectl get svc -n jaeger
```

### Exercise 4.2: Test Tracing

```bash
# Create pod with tracing
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: tracing-test
  namespace: observability-test
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
EOF

# Check pod status
kubectl get pod tracing-test -n observability-test
# STATUS: Running

# Test tracing
kubectl exec tracing-test -n observability-test -- sh -c "curl -s http://localhost"
# Should work
```

### Exercise 4.3: Test APM

```bash
# Install APM agent
kubectl apply -f https://raw.githubusercontent.com/elastic/elasticsearch/main/deploy/kubernetes/elasticsearch.yaml

# Check APM
kubectl get pods -n elasticsearch
kubectl get svc -n elasticsearch
```

### Exercise 4.4: Test Monitoring Integration

```bash
# Create monitoring dashboard
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: monitoring-config
  namespace: observability-test
data:
  dashboard.json: |
    {
      "dashboard": {
        "title": "Kubernetes Monitoring",
        "panels": [
          {
            "title": "Pod CPU Usage",
            "type": "graph",
            "targets": [
              {
                "expr": "rate(container_cpu_usage_seconds_total[5m])"
              }
            ]
          }
        ]
      }
    }
EOF

# Check monitoring config
kubectl get configmap monitoring-config -n observability-test
```

---

## 🎯 Lab Part 5: Advanced Observability Scenarios (45 minutes)

### Exercise 5.1: Test Custom Metrics

```bash
# Create custom metrics
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: metrics-test
  namespace: observability-test
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    env:
    - name: METRICS_PORT
      value: "9090"
EOF

# Check pod status
kubectl get pod metrics-test -n observability-test
# STATUS: Running

# Test custom metrics
kubectl exec metrics-test -n observability-test -- sh -c "curl -s http://localhost:9090/metrics"
# Should show custom metrics
```

### Exercise 5.2: Test Alerting

```bash
# Create alerting rules
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-alerts
  namespace: observability-test
spec:
  groups:
  - name: kubernetes.rules
    rules:
    - alert: HighCPUUsage
      expr: rate(container_cpu_usage_seconds_total[5m]) > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage detected"
EOF

# Check alerting rules
kubectl get prometheusrule kubernetes-alerts -n observability-test
```

### Exercise 5.3: Test Service Discovery

```bash
# Create service discovery
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: discovery-test
  namespace: observability-test
  labels:
    app: discovery-test
spec:
  selector:
    app: discovery-test
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: discovery-test
  namespace: observability-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: discovery-test
  template:
    metadata:
      labels:
        app: discovery-test
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# Check service discovery
kubectl get svc discovery-test -n observability-test
kubectl get pods -l app=discovery-test -n observability-test
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-09-reflections.md` and answer:

1. **What does observability do in Kubernetes?**
   - How does monitoring work?
   - What happens when it fails?
   - Why is it critical for production?

2. **What happens when monitoring fails?**
   - How do you detect monitoring failures?
   - What happens to metrics and logs?
   - How do you recover from monitoring failures?

3. **How do you implement observability?**
   - What tools do you use?
   - How do you set up monitoring?
   - What are the best practices?

4. **How do you debug observability issues?**
   - What tools do you use?
   - How do you check monitoring status?
   - How do you test observability?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Kubernetes monitoring failures**
- **Observability best practices**
- **Monitoring tool failures**
- **Log aggregation issues**

### Exercise 6.3: Advanced Observability Scenarios

```bash
# Create complex observability scenario
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: complex-observability
  namespace: observability-test
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    env:
    - name: METRICS_PORT
      value: "9090"
    - name: LOG_LEVEL
      value: "debug"
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

# Test the observability
kubectl get pod complex-observability -n observability-test
# STATUS: Running
```

---

## 🏆 Success Criteria

You've successfully completed Day 9 if you can:

✅ **Watch monitoring components work in real-time**  
✅ **Break monitoring and understand the impact**  
✅ **Test logging and tracing failures**  
✅ **Practice observability procedures**  
✅ **Understand the observability architecture**  
✅ **Debug observability issues**  

## 🚨 Common Issues & Solutions

### Issue: Metrics not available
```bash
# Check metrics server status
kubectl get pods -n kube-system | grep metrics-server

# Check metrics server logs
kubectl logs -n kube-system -l k8s-app=metrics-server
```

### Issue: Logs not working
```bash
# Check pod logs
kubectl logs <pod-name>

# Check node status
kubectl get nodes

# Check kubelet logs
minikube ssh
sudo journalctl -u kubelet
```

### Issue: Monitoring not working
```bash
# Check monitoring components
kubectl get pods -n monitoring

# Check monitoring logs
kubectl logs -n monitoring -l app=prometheus
```

---

## 🎯 Tomorrow's Preview

**Day 10: Advanced Scheduling - The Tetris Master**
- Watch advanced scheduling work
- Break advanced scheduling and see pods stuck
- Test node affinity and taints
- Practice advanced scheduling procedures
- Understand the advanced scheduling architecture

**Get ready for more chaos! 🔥**

---

## 📚 Additional Resources

- [Kubernetes Observability](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [Prometheus](https://prometheus.io/docs/)
- [Grafana](https://grafana.com/docs/)
- [Jaeger](https://www.jaegertracing.io/docs/)

**Remember: Observability is the nervous system of Kubernetes - it tells you what's happening! 🧠**
