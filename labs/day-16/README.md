# Day 16 Lab: Configuration Management - The Config Master

> **"Today you'll break configuration and watch apps fail. You'll learn why configuration management is the config master of Kubernetes."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch configuration components work in real-time
- Break configuration and see apps fail
- Test configuration failures
- Practice configuration procedures
- Understand the configuration architecture
- Learn why configuration management is critical for app deployment

## ⚠️ Prerequisites

- Day 15 lab completed
- Minikube cluster running
- 4-5 hours of uninterrupted time
- Understanding of configuration concepts

## 🛠️ Setup (30 minutes)

### Step 1: Verify Cluster Status

```bash
# Check cluster is running
kubectl get nodes
kubectl get pods -n kube-system

# Clean up previous lab resources
kubectl delete deployment backend backend-1 backend-2 backend-canary backend-blue backend-green -n ingress-test
kubectl delete service backend backend-1 backend-2 backend-canary backend-blue backend-green -n ingress-test
kubectl delete ingress test-ingress chaos-ingress load-balance-ingress tls-ingress path-ingress host-ingress annotation-ingress rate-limit-ingress auth-ingress canary-ingress blue-green-ingress ab-testing-ingress complex-ingress -n ingress-test
kubectl delete secret test-tls basic-auth -n ingress-test
kubectl delete namespace ingress-test
```

### Step 2: Setup Configuration Environment

```bash
# Create configuration namespace
kubectl create namespace config-test

# Create configuration labels
kubectl label namespace config-test environment=test
kubectl label namespace config-test tier=configuration
```

### Step 3: Setup Monitoring

```bash
# Open 4 terminals
# Terminal 1: kubectl get pods --all-namespaces --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: kubectl get configmaps,secrets --watch
# Terminal 4: Your command terminal
```

---

## 🧪 Lab Part 1: Watch Configuration Work (45 minutes)

### Exercise 1.1: Create ConfigMap

```bash
# Create ConfigMap
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: config-test
data:
  app.properties: |
    app.name=Test App
    app.version=1.0.0
    app.environment=test
  database.properties: |
    db.host=localhost
    db.port=5432
    db.name=testdb
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

# Check ConfigMap
kubectl get configmap app-config -n config-test
kubectl describe configmap app-config -n config-test
```

### Exercise 1.2: Create Secret

```bash
# Create Secret
kubectl create secret generic app-secret \
  --from-literal=db-password=super-secret-password \
  --from-literal=api-key=test-api-key \
  --from-literal=jwt-secret=jwt-secret-key \
  -n config-test

# Check Secret
kubectl get secret app-secret -n config-test
kubectl describe secret app-secret -n config-test
```

### Exercise 1.3: Create Pod with Configuration

```bash
# Create pod with configuration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
  namespace: config-test
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: app-config
      mountPath: /etc/app
      readOnly: true
    - name: app-secret
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: app-config
    configMap:
      name: app-config
  - name: app-secret
    secret:
      secretName: app-secret
EOF

# Check pod
kubectl get pod config-pod -n config-test
# STATUS: Running
```

### Exercise 1.4: Test Configuration

```bash
# Check configuration files
kubectl exec config-pod -n config-test -- ls -la /etc/app
# Should show: app.properties, database.properties, nginx.conf

kubectl exec config-pod -n config-test -- cat /etc/app/app.properties
# Should show: app.name=Test App, etc.

# Check secret files
kubectl exec config-pod -n config-test -- ls -la /etc/secrets
# Should show: db-password, api-key, jwt-secret

kubectl exec config-pod -n config-test -- cat /etc/secrets/db-password
# Should show: super-secret-password
```

---

## 💀 Lab Part 2: Break #18 - Delete Configuration (1 hour)

> **"Time to break the configuration. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test workload
kubectl create deployment chaos-test --image=nginx -n config-test --replicas=2

# Create service
kubectl expose deployment chaos-test --port=80 -n config-test

# Create ConfigMap
kubectl create configmap chaos-config --from-literal=key=value -n config-test

# Create Secret
kubectl create secret generic chaos-secret --from-literal=password=secret -n config-test

# Create pod with configuration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: chaos-config-pod
  namespace: config-test
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: chaos-config
      mountPath: /etc/config
      readOnly: true
    - name: chaos-secret
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: chaos-config
    configMap:
      name: chaos-config
  - name: chaos-secret
    secret:
      secretName: chaos-secret
EOF

# Verify everything is working
kubectl get pods -l app=chaos-test -n config-test
kubectl get svc chaos-test -n config-test
kubectl get configmap chaos-config -n config-test
kubectl get secret chaos-secret -n config-test
kubectl get pod chaos-config-pod -n config-test
```

### Exercise 2.2: Break Configuration

```bash
# Delete ConfigMap
kubectl delete configmap chaos-config -n config-test

# Delete Secret
kubectl delete secret chaos-secret -n config-test

# Check configuration
kubectl get configmap chaos-config -n config-test
# Error: not found

kubectl get secret chaos-secret -n config-test
# Error: not found
```

### Exercise 2.3: Observe the Chaos

**Check pod status:**
```bash
# Check pod status
kubectl get pod chaos-config-pod -n config-test
# STATUS: Running (but configuration is gone!)

# Check pod events
kubectl describe pod chaos-config-pod -n config-test
# Should show: FailedMount
```

**Test configuration access:**
```bash
# Try to access configuration
kubectl exec chaos-config-pod -n config-test -- ls -la /etc/config
# Should show: No such file or directory

kubectl exec chaos-config-pod -n config-test -- ls -la /etc/secrets
# Should show: No such file or directory
```

### Exercise 2.4: Debug the Problem

```bash
# Check pod status
kubectl get pod chaos-config-pod -n config-test
# STATUS: Running (but no configuration!)

# Check pod events
kubectl describe pod chaos-config-pod -n config-test
# Should show: FailedMount

# Check configuration
kubectl get configmap chaos-config -n config-test
# Error: not found

kubectl get secret chaos-secret -n config-test
# Error: not found
```

**Key Learning:** 
- Configuration is critical for app functionality
- No configuration = app may fail
- Pods keep running but no config
- Apps become "blind" to configuration

### Exercise 2.5: Fix It

```bash
# Recreate ConfigMap
kubectl create configmap chaos-config --from-literal=key=value -n config-test

# Recreate Secret
kubectl create secret generic chaos-secret --from-literal=password=secret -n config-test

# Check configuration
kubectl get configmap chaos-config -n config-test
kubectl get secret chaos-secret -n config-test

# Check pod status
kubectl get pod chaos-config-pod -n config-test
# STATUS: Running

# Test configuration access
kubectl exec chaos-config-pod -n config-test -- ls -la /etc/config
# Should show: key

kubectl exec chaos-config-pod -n config-test -- ls -la /etc/secrets
# Should show: password
```

**Real-world parallel:** This happens when:
- Configuration is accidentally deleted
- Configuration management system fails
- Human error
- Automation errors

---

## 🔥 Lab Part 3: Configuration Failures (1 hour)

> **"Time to test what happens when configuration fails. This is where configuration resilience is tested."**

### Exercise 3.1: Test Configuration Updates

```bash
# Update ConfigMap
kubectl patch configmap app-config -n config-test --patch '{"data":{"app.properties":"app.name=Updated App\napp.version=2.0.0\napp.environment=test"}}'

# Check ConfigMap
kubectl get configmap app-config -n config-test
kubectl describe configmap app-config -n config-test
```

### Exercise 3.2: Test Configuration Rollback

```bash
# Rollback ConfigMap
kubectl patch configmap app-config -n config-test --patch '{"data":{"app.properties":"app.name=Test App\napp.version=1.0.0\napp.environment=test"}}'

# Check ConfigMap
kubectl get configmap app-config -n config-test
kubectl describe configmap app-config -n config-test
```

### Exercise 3.3: Test Configuration Validation

```bash
# Create invalid ConfigMap
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: invalid-config
  namespace: config-test
data:
  invalid.properties: |
    invalid.config=test
    missing.value=
    broken.config
EOF

# Check ConfigMap
kubectl get configmap invalid-config -n config-test
kubectl describe configmap invalid-config -n config-test
```

### Exercise 3.4: Test Configuration Backup

```bash
# Backup configuration
kubectl get configmap app-config -n config-test -o yaml > app-config-backup.yaml
kubectl get secret app-secret -n config-test -o yaml > app-secret-backup.yaml

# Check backup
ls -la app-config-backup.yaml app-secret-backup.yaml
# Should show backup files
```

---

## 🧠 Lab Part 4: Advanced Configuration Scenarios (1 hour)

> **"Time to test advanced configuration scenarios. This is where you become a configuration expert."**

### Exercise 4.1: Test Configuration Templates

```bash
# Create configuration template
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: template-config
  namespace: config-test
data:
  template.properties: |
    app.name=\${APP_NAME}
    app.version=\${APP_VERSION}
    app.environment=\${APP_ENVIRONMENT}
    db.host=\${DB_HOST}
    db.port=\${DB_PORT}
EOF

# Check ConfigMap
kubectl get configmap template-config -n config-test
kubectl describe configmap template-config -n config-test
```

### Exercise 4.2: Test Configuration Inheritance

```bash
# Create base configuration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: base-config
  namespace: config-test
data:
  base.properties: |
    app.name=Base App
    app.version=1.0.0
    app.environment=base
EOF

# Create environment-specific configuration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: config-test
data:
  env.properties: |
    app.environment=production
    db.host=prod-db
    db.port=5432
EOF

# Check configurations
kubectl get configmap base-config -n config-test
kubectl get configmap env-config -n config-test
```

### Exercise 4.3: Test Configuration Encryption

```bash
# Create encrypted secret
kubectl create secret generic encrypted-secret \
  --from-literal=encrypted-data=encrypted-value \
  -n config-test

# Check secret
kubectl get secret encrypted-secret -n config-test
kubectl describe secret encrypted-secret -n config-test
```

### Exercise 4.4: Test Configuration Monitoring

```bash
# Install configuration monitoring
kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml

# Check monitoring
kubectl get pods -n monitoring
# Should show monitoring pods
```

---

## 🎯 Lab Part 5: Configuration Best Practices (45 minutes)

### Exercise 5.1: Test Configuration Security

```bash
# Create secure configuration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: secure-config
  namespace: config-test
data:
  secure.properties: |
    app.name=Secure App
    app.version=1.0.0
    app.environment=secure
    security.enabled=true
    security.encryption=true
EOF

# Check ConfigMap
kubectl get configmap secure-config -n config-test
kubectl describe configmap secure-config -n config-test
```

### Exercise 5.2: Test Configuration Validation

```bash
# Create validation configuration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: validation-config
  namespace: config-test
data:
  validation.properties: |
    app.name=Validation App
    app.version=1.0.0
    app.environment=validation
    validation.enabled=true
    validation.rules=strict
EOF

# Check ConfigMap
kubectl get configmap validation-config -n config-test
kubectl describe configmap validation-config -n config-test
```

### Exercise 5.3: Test Configuration Documentation

```bash
# Create documented configuration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: documented-config
  namespace: config-test
data:
  documented.properties: |
    # Application Configuration
    app.name=Documented App
    app.version=1.0.0
    app.environment=documented
    
    # Database Configuration
    db.host=localhost
    db.port=5432
    db.name=documented
    
    # Security Configuration
    security.enabled=true
    security.encryption=true
EOF

# Check ConfigMap
kubectl get configmap documented-config -n config-test
kubectl describe configmap documented-config -n config-test
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-16-reflections.md` and answer:

1. **What does configuration management do?**
   - How does it manage app configuration?
   - What happens when it fails?
   - Why is it critical for app deployment?

2. **What happens when configuration fails?**
   - How do you detect configuration failures?
   - What happens to apps?
   - How do you recover from configuration failures?

3. **How do you use configuration management?**
   - What's the difference between ConfigMaps and Secrets?
   - How do you manage configuration updates?
   - What are the best practices?

4. **How do you debug configuration issues?**
   - What tools do you use?
   - How do you check configuration status?
   - How do you test configuration?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Kubernetes configuration failures**
- **Configuration management issues**
- **Secret management problems**
- **Configuration best practices**

### Exercise 6.3: Advanced Configuration Scenarios

```bash
# Create complex configuration scenario
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: complex-config
  namespace: config-test
data:
  complex.properties: |
    app.name=Complex App
    app.version=1.0.0
    app.environment=complex
    db.host=complex-db
    db.port=5432
    db.name=complex
    security.enabled=true
    security.encryption=true
    monitoring.enabled=true
    monitoring.level=debug
    logging.enabled=true
    logging.level=info
EOF

# Test the configuration
kubectl get configmap complex-config -n config-test
```

---

## 🏆 Success Criteria

You've successfully completed Day 16 if you can:

✅ **Watch configuration components work in real-time**  
✅ **Break configuration and understand the impact**  
✅ **Test configuration failures**  
✅ **Practice configuration procedures**  
✅ **Understand the configuration architecture**  
✅ **Debug configuration issues**  

## 🚨 Common Issues & Solutions

### Issue: Configuration not accessible
```bash
# Check ConfigMap status
kubectl get configmap <name>

# Check Secret status
kubectl get secret <name>

# Check pod volume mounts
kubectl describe pod <pod-name>
```

### Issue: Configuration not updating
```bash
# Check ConfigMap data
kubectl get configmap <name> -o yaml

# Check pod configuration
kubectl exec <pod-name> -- cat /etc/config/<file>
```

### Issue: Configuration validation failing
```bash
# Check configuration syntax
kubectl get configmap <name> -o yaml

# Check pod logs
kubectl logs <pod-name>
```

---

## 🎯 Tomorrow's Preview

**Day 17: Advanced Patterns - The Pattern Master**
- Watch advanced patterns work
- Break advanced patterns and see apps fail
- Test pattern failures
- Practice pattern procedures
- Understand the pattern architecture

**Get ready for the pattern master! 🔥**

---

## 📚 Additional Resources

- [Kubernetes Configuration](https://kubernetes.io/docs/concepts/configuration/)
- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Configuration Best Practices](https://kubernetes.io/docs/concepts/configuration/)

**Remember: Configuration management is the config master of Kubernetes - it manages all app configuration! ⚙️**
