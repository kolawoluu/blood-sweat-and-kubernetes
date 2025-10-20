# Day 8 Lab: Security - The Immune System

> **"Today you'll break security and watch your cluster become vulnerable. You'll learn why security is the immune system of Kubernetes."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch security components work in real-time
- Break security and see pods become vulnerable
- Test RBAC and network policy failures
- Practice security hardening procedures
- Understand the security architecture
- Learn why security is critical for production clusters

## ⚠️ Prerequisites

- Day 7 lab completed
- Minikube cluster running
- 4-5 hours of uninterrupted time
- Understanding of security concepts

## 🛠️ Setup (30 minutes)

### Step 1: Verify Cluster Status

```bash
# Check cluster is running
kubectl get nodes
kubectl get pods -n kube-system

# Clean up previous lab resources
kubectl delete pod storage-test-2 storage-test-3 migration-test complex-storage-test
kubectl delete pvc test-pvc test-pvc-2 expandable-pvc complex-pvc fast-pvc
kubectl delete pv test-pv test-pv-2 expandable-pv complex-pv
kubectl delete storageclass fast-ssd
kubectl delete namespace storage-test
```

### Step 2: Setup Monitoring

```bash
# Open 4 terminals
# Terminal 1: kubectl get pods --all-namespaces --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: kubectl get roles,rolebindings --watch
# Terminal 4: Your command terminal
```

### Step 3: Check Security Components

```bash
# Check RBAC components
kubectl get roles --all-namespaces
kubectl get rolebindings --all-namespaces
kubectl get clusterroles
kubectl get clusterrolebindings

# Check security contexts
kubectl get pods -o yaml | grep -A 10 securityContext
```

---

## 🧪 Lab Part 1: Watch Security Work (45 minutes)

### Exercise 1.1: Test RBAC

```bash
# Create namespace for testing
kubectl create namespace security-test

# Create service account
kubectl create serviceaccount test-user -n security-test

# Check service account
kubectl get serviceaccount test-user -n security-test
```

### Exercise 1.2: Create Role and RoleBinding

```bash
# Create role
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: security-test
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
EOF

# Create role binding
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: security-test
subjects:
- kind: ServiceAccount
  name: test-user
  namespace: security-test
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

### Exercise 1.3: Test RBAC with Service Account

```bash
# Create pod with service account
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: rbac-test
  namespace: security-test
spec:
  serviceAccountName: test-user
  containers:
  - name: nginx
    image: nginx
EOF

# Wait for pod to be ready
kubectl wait --for=condition=ready pod rbac-test -n security-test

# Test RBAC from inside pod
kubectl exec rbac-test -n security-test -- sh -c "kubectl get pods -n security-test"
# Should work - has permission to list pods

kubectl exec rbac-test -n security-test -- sh -c "kubectl get pods -n default"
# Should fail - no permission to access default namespace
```

### Exercise 1.4: Test Security Context

```bash
# Create pod with security context
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: security-context-test
  namespace: security-test
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
EOF

# Check pod status
kubectl get pod security-context-test -n security-test
# STATUS: Running

# Check security context
kubectl describe pod security-context-test -n security-test
# Should show security context settings
```

---

## 💀 Lab Part 2: Break #10 - Disable RBAC (1 hour)

> **"Time to break the security. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test deployment
kubectl create deployment chaos-test --image=nginx -n security-test --replicas=2

# Create service
kubectl expose deployment chaos-test --port=80 -n security-test

# Verify everything is working
kubectl get pods -l app=chaos-test -n security-test
kubectl get svc chaos-test -n security-test
```

### Exercise 2.2: Break RBAC

```bash
# Disable RBAC by modifying API server
minikube ssh
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/backup.yaml

# Edit API server config
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml

# Find the command section and REMOVE this line:
# - --authorization-mode=Node,RBAC

# Save and exit (:wq)
```

### Exercise 2.3: Observe the Chaos

**Watch Terminal 1 (pods):**
- API server restarts
- Pods may show "Unknown" status temporarily

**Test security bypass:**
```bash
# Wait for API server to restart
kubectl get nodes
# Should work

# Test RBAC bypass
kubectl exec rbac-test -n security-test -- sh -c "kubectl get pods -n default"
# Should work now - RBAC is disabled!

# Test cluster admin access
kubectl exec rbac-test -n security-test -- sh -c "kubectl get secrets -n kube-system"
# Should work - no authorization!

# Test privileged access
kubectl exec rbac-test -n security-test -- sh -c "kubectl get nodes"
# Should work - no authorization!
```

### Exercise 2.4: Debug the Problem

```bash
# Check API server logs
kubectl logs kube-apiserver-minikube -n kube-system
# Should show: authorization mode changed

# Check RBAC status
kubectl get clusterroles
# Should still exist but not enforced

# Test security
kubectl exec rbac-test -n security-test -- sh -c "kubectl get pods --all-namespaces"
# Should work - no authorization!
```

**Key Learning:** 
- RBAC is critical for security
- No RBAC = no authorization
- Any user can access anything
- Cluster becomes completely open

### Exercise 2.5: Fix It

```bash
# Restore backup
minikube ssh
sudo cp /tmp/backup.yaml /etc/kubernetes/manifests/kube-apiserver.yaml
exit

# Wait for API server to restart
kubectl get nodes
# Should work

# Test RBAC is back
kubectl exec rbac-test -n security-test -- sh -c "kubectl get pods -n default"
# Should fail again - RBAC is back!

# Test cluster admin access
kubectl exec rbac-test -n security-test -- sh -c "kubectl get secrets -n kube-system"
# Should fail - RBAC is back!
```

**Real-world parallel:** This happens when:
- RBAC is misconfigured
- Authorization is disabled
- Security policies are bypassed
- Human error in configuration

---

## 🔥 Lab Part 3: Network Policy Failures (1 hour)

> **"Time to test what happens when network policies fail. This is where network security is tested."**

### Exercise 3.1: Create Network Policy

```bash
# Create network policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: security-test
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# Test connectivity
kubectl create deployment test --image=busybox -n security-test --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=ready pod -l app=test -n security-test --timeout=60s
kubectl exec -it deployment/test -n security-test -- sh

# Inside the pod
wget -O- http://chaos-test
# Should fail - network policy blocks it

# Exit the test pod
exit
kubectl delete deployment test -n security-test
```

### Exercise 3.2: Break Network Policy

```bash
# Delete network policy
kubectl delete networkpolicy deny-all -n security-test

# Test connectivity again
kubectl create deployment test --image=busybox -n security-test --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=ready pod -l app=test -n security-test --timeout=60s
kubectl exec -it deployment/test -n security-test -- sh

# Inside the pod
wget -O- http://chaos-test
# Should work now - no network policy!

# Test external connectivity
wget -O- http://google.com
# Should work - no network policy!

# Exit the test pod
exit
kubectl delete deployment test -n security-test
```

### Exercise 3.3: Create Specific Network Policy

```bash
# Create more specific network policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-backend
  namespace: security-test
spec:
  podSelector:
    matchLabels:
      app: chaos-test
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: test
    ports:
    - protocol: TCP
      port: 80
EOF

# Test connectivity
kubectl create deployment test --image=busybox -n security-test --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=ready pod -l app=test -n security-test --timeout=60s
kubectl exec -it deployment/test -n security-test -- sh

# Inside the pod
wget -O- http://chaos-test
# Should work - network policy allows it

# Test external connectivity
wget -O- http://google.com
# Should fail - network policy blocks it

# Exit the test pod
exit
kubectl delete deployment test -n security-test
```

### Exercise 3.4: Test Network Policy Bypass

```bash
# Create pod with different labels
kubectl create deployment test-2 --image=busybox -n security-test --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=ready pod -l app=test-2 -n security-test --timeout=60s
kubectl exec -it deployment/test-2 -n security-test -- sh

# Inside the pod
wget -O- http://chaos-test
# Should fail - network policy blocks it (different labels)

# Exit the test pod
exit
kubectl delete deployment test-2 -n security-test
```

**Real-world parallel:** This happens when:
- Network policies are misconfigured
- Security policies are bypassed
- Network segmentation fails
- Human error in configuration

---

## 🧠 Lab Part 4: Security Hardening (1 hour)

> **"Time to practice real security hardening. This is where you become a security expert."**

### Exercise 4.1: Test Pod Security Standards

```bash
# Create pod with security standards
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: security-test
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
kubectl get pod secure-pod -n security-test
# STATUS: Running

# Check security context
kubectl describe pod secure-pod -n security-test
# Should show security context settings
```

### Exercise 4.2: Test Security Scanning

```bash
# Install security scanning tool
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# Check security scan results
kubectl get pods -n kube-system | grep kube-bench
kubectl logs -n kube-system -l app=kube-bench
```

### Exercise 4.3: Test Image Security

```bash
# Create pod with vulnerable image
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: vulnerable-pod
  namespace: security-test
spec:
  containers:
  - name: nginx
    image: nginx:1.14  # Old version with vulnerabilities
EOF

# Check pod status
kubectl get pod vulnerable-pod -n security-test
# STATUS: Running

# Scan for vulnerabilities
kubectl exec vulnerable-pod -n security-test -- sh -c "apt-get update && apt-get install -y curl"
kubectl exec vulnerable-pod -n security-test -- sh -c "curl -s https://vulners.com/api/v3/search/lucene/?q=nginx:1.14"
```

### Exercise 4.4: Test Secret Management

```bash
# Create secret
kubectl create secret generic test-secret --from-literal=password=secret123 -n security-test

# Check secret
kubectl get secret test-secret -n security-test
kubectl describe secret test-secret -n security-test

# Test secret access
kubectl exec rbac-test -n security-test -- sh -c "kubectl get secret test-secret -n security-test"
# Should fail - no permission to access secrets
```

---

## 🎯 Lab Part 5: Advanced Security Scenarios (45 minutes)

### Exercise 5.1: Test Admission Controllers

```bash
# Create pod with invalid security context
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: invalid-security-pod
  namespace: security-test
spec:
  securityContext:
    runAsUser: 0  # Root user
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: true
      readOnlyRootFilesystem: false
      runAsNonRoot: false
      runAsUser: 0
EOF

# Check pod status
kubectl get pod invalid-security-pod -n security-test
# STATUS: Running (admission controllers may not block this in minikube)
```

### Exercise 5.2: Test Security Policies

```bash
# Create pod security policy (Note: PodSecurityPolicy is deprecated)
# Use Pod Security Standards instead for modern clusters
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restrictive-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
EOF

# Note: For modern clusters, use Pod Security Standards instead:
# kubectl label namespace security-test pod-security.kubernetes.io/enforce=restricted

# Test pod with security policy
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: psp-test
  namespace: security-test
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
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
EOF

# Check pod status
kubectl get pod psp-test -n security-test
# STATUS: Running
```

### Exercise 5.3: Test Security Monitoring

```bash
# Install security monitoring
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-hunter/main/job.yaml

# Check security scan results
kubectl get pods -n kube-system | grep kube-hunter
kubectl logs -n kube-system -l app=kube-hunter
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-08-reflections.md` and answer:

1. **What does RBAC do?**
   - How does it control access?
   - What happens when it's disabled?
   - Why is it critical for security?

2. **What do network policies do?**
   - How do they control network traffic?
   - What happens when they fail?
   - Why are they important for security?

3. **How do you harden Kubernetes?**
   - What security best practices should you follow?
   - How do you scan for vulnerabilities?
   - What are the security implications?

4. **How do you debug security issues?**
   - What tools do you use?
   - How do you check security status?
   - How do you test security policies?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Kubernetes security vulnerabilities**
- **RBAC misconfigurations**
- **Network policy failures**
- **Security hardening procedures**

### Exercise 6.3: Advanced Security Scenarios

```bash
# Create complex security scenario
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: complex-security-pod
  namespace: security-test
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
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: nginx-config
EOF

# Test the security
kubectl get pod complex-security-pod -n security-test
# STATUS: Running
```

---

## 🏆 Success Criteria

You've successfully completed Day 8 if you can:

✅ **Watch security components work in real-time**  
✅ **Break security and understand the impact**  
✅ **Test RBAC and network policy failures**  
✅ **Practice security hardening procedures**  
✅ **Understand the security architecture**  
✅ **Debug security issues**  

## 🚨 Common Issues & Solutions

### Issue: RBAC not working
```bash
# Check RBAC status
kubectl get clusterroles
kubectl get clusterrolebindings

# Check API server logs
kubectl logs kube-apiserver-minikube -n kube-system
```

### Issue: Network policies not working
```bash
# Check network policy status
kubectl get networkpolicies

# Check pod labels
kubectl get pods --show-labels
```

### Issue: Security contexts not working
```bash
# Check pod security context
kubectl describe pod <pod-name>

# Check security policy
kubectl get podsecuritypolicies
```

---

## 🎯 Tomorrow's Preview

**Day 9: Observability - The Nervous System**
- Watch monitoring components work
- Break monitoring and see metrics disappear
- Test logging and tracing failures
- Practice observability procedures
- Understand the observability architecture

**Get ready for more chaos! 🔥**

---

## 📚 Additional Resources

- [Kubernetes Security](https://kubernetes.io/docs/concepts/security/)
- [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

**Remember: Security is the immune system of Kubernetes - it protects everything! 🛡️**
