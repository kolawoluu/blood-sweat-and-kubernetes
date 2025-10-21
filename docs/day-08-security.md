# Day 8: Kubernetes Security - Understanding RBAC, Secrets, and Pod Security

## The Office Building Analogy

Imagine a secure office building:
- **Authentication** = Security guard checks your ID badge at entrance
- **Authorization (RBAC)** = Your badge determines which floors/rooms you can access
- **Secrets** = Locked safe containing sensitive documents
- **Pod Security** = Rules for what visitors can/cannot do in each room
- **Network Policies** = Doors between rooms (control who can visit whom)

```
┌──────────────────────────────────────────────────────┐
│            SECURE OFFICE BUILDING (CLUSTER)           │
│                                                       │
│  ENTRANCE (API Server):                              │
│  ┌────────────────────────────────────┐              │
│  │ Security Guard (Authentication)    │              │
│  │ "Show me your ID badge!"           │              │
│  │ ✓ Valid certificate                │              │
│  │ ✓ Valid token                      │              │
│  │ ✓ Valid username/password          │              │
│  └────────────────────────────────────┘              │
│            ↓ Authenticated                            │
│  ┌────────────────────────────────────┐              │
│  │ Badge Reader (Authorization/RBAC)  │              │
│  │ "What can you access?"             │              │
│  │                                    │              │
│  │ Employee Badge:                    │              │
│  │ ✓ Floor 1-5 (development)         │              │
│  │ ✓ Meeting rooms                   │              │
│  │ ❌ Executive floor                 │              │
│  │ ❌ Server room                     │              │
│  │                                    │              │
│  │ Manager Badge:                     │              │
│  │ ✓ All employee access +           │              │
│  │ ✓ Budget room                     │              │
│  │ ✓ HR records                      │              │
│  └────────────────────────────────────┘              │
│            ↓ Authorized                               │
│  OFFICES (Namespaces):                               │
│  ┌────────────────┐  ┌────────────────┐             │
│  │ Development    │  │ Production     │             │
│  │ (Loose rules)  │  │ (Strict rules) │             │
│  │                │  │                │             │
│  │ Anyone can:    │  │ Must follow:   │             │
│  │ - Deploy       │  │ - No root      │             │
│  │ - Debug        │  │ - ReadOnly FS  │             │
│  │ - Experiment   │  │ - Resource limits│            │
│  └────────────────┘  └────────────────┘             │
│                                                       │
│  SAFE (Secrets):                                     │
│  ┌────────────────────────────────────┐              │
│  │ 🔒 Combination lock                │              │
│  │ Contains:                          │              │
│  │ - Database passwords               │              │
│  │ - API keys                         │              │
│  │ - TLS certificates                 │              │
│  │                                    │              │
│  │ Only specific people know combo:   │              │
│  │ - Database pod                     │              │
│  │ - Backend service                  │              │
│  │ ❌ Frontend (doesn't need it)      │              │
│  └────────────────────────────────────┘              │
└──────────────────────────────────────────────────────┘
```

---

## The Three Layers of Security

### Layer 1: Authentication - Who are you?

**Methods to prove identity:**

```
1. Client Certificates (most common)
   User has: certificate + private key
   API Server has: CA certificate
   API Server verifies: "Certificate signed by our CA? Yes ✓"

2. Bearer Tokens (ServiceAccounts)
   Pod has: JWT token in /var/run/secrets/kubernetes.io/serviceaccount/token
   API Server verifies: "Token valid? Yes ✓"

3. Username/Password (Basic Auth)
   Rarely used, not recommended
   API Server checks: username/password against static file

4. OIDC (OpenID Connect)
   Enterprise setup
   "Log in with Google/Azure AD"
   Single sign-on

Authentication Process:
┌─────────────────────────────────────┐
│ kubectl get pods                    │
│ (Uses kubeconfig file)              │
└─────────────────────────────────────┘
       ↓
┌─────────────────────────────────────┐
│ API Server checks:                  │
│ "Is certificate valid?" ✓           │
│ "Signed by our CA?" ✓               │
│ "Not expired?" ✓                    │
│                                     │
│ Authenticated as: user@example.com  │
└─────────────────────────────────────┘
       ↓
     Proceed to Authorization
```

---

### Layer 2: Authorization (RBAC) - What can you do?

**RBAC = Role-Based Access Control**

**Four key objects:**

```
1. ServiceAccount
   Identity for pods
   
   Example:
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: backend-sa
     namespace: production

2. Role (namespace-scoped)
   Set of permissions
   
   Example:
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: pod-reader
     namespace: production
   rules:
   - apiGroups: [""]
     resources: ["pods"]
     verbs: ["get", "list", "watch"]
   
   Means: "Can read pods in production namespace"

3. ClusterRole (cluster-wide)
   Like Role but across all namespaces
   
   Example:
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: cluster-admin
   rules:
   - apiGroups: ["*"]
     resources: ["*"]
     verbs: ["*"]
   
   Means: "Can do ANYTHING in cluster"

4. RoleBinding (connect Role to User/ServiceAccount)
   Grants permissions
   
   Example:
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: backend-pod-reader
     namespace: production
   subjects:
   - kind: ServiceAccount
     name: backend-sa
     namespace: production
   roleRef:
     kind: Role
     name: pod-reader
     apiGroup: rbac.authorization.k8s.io
   
   Means: "backend-sa can read pods in production"
```

---

### How RBAC Works - Complete Flow

```
Pod with ServiceAccount "backend-sa" wants to list pods:

Step 1: Pod makes API request
┌─────────────────────────────────────┐
│ curl https://kubernetes.default/    │
│      api/v1/namespaces/production/  │
│      pods                           │
│                                     │
│ Headers:                            │
│ Authorization: Bearer <token>       │
└─────────────────────────────────────┘
       ↓
Step 2: API Server authenticates
┌─────────────────────────────────────┐
│ Token valid? ✓                      │
│ Identity: backend-sa in production  │
└─────────────────────────────────────┘
       ↓
Step 3: API Server checks authorization
┌─────────────────────────────────────┐
│ Find all RoleBindings for backend-sa│
│                                     │
│ Found: backend-pod-reader           │
│ Role: pod-reader                    │
│                                     │
│ Check if pod-reader allows:         │
│ - Resource: pods ✓                  │
│ - Verb: list ✓                      │
│ - Namespace: production ✓           │
│                                     │
│ ALLOWED! ✓                          │
└─────────────────────────────────────┘
       ↓
Step 4: Return pod list
┌─────────────────────────────────────┐
│ Response: [pod1, pod2, pod3]        │
└─────────────────────────────────────┘

If backend-sa tried to DELETE pods:
- pod-reader doesn't have "delete" verb
- Request DENIED! ❌
- Response: 403 Forbidden
```

---

## Real-World RBAC Examples

### Example 1: Developer Access

**Requirement:** Developers can deploy and debug in dev namespace, read-only in production

```yaml
# Developer Role in dev namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["*"]  # Full access in dev!

---
# Developer Role in production namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-readonly
  namespace: production
rules:
- apiGroups: ["", "apps"]
  resources: ["pods", "deployments", "services"]
  verbs: ["get", "list", "watch"]  # Read-only in production!

---
# Bind to user
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-developer
  namespace: dev
subjects:
- kind: User
  name: alice@example.com
roleRef:
  kind: Role
  name: developer

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-developer-readonly
  namespace: production
subjects:
- kind: User
  name: alice@example.com
roleRef:
  kind: Role
  name: developer-readonly

Result:
Alice in dev namespace:
✓ Can create deployments
✓ Can delete pods
✓ Can exec into pods
✓ Can view logs

Alice in production namespace:
✓ Can view deployments
✓ Can view pods
✓ Can view logs
❌ Cannot create anything
❌ Cannot delete anything
❌ Cannot exec into pods (security!)
```

---

### Example 2: CI/CD Service Account

**Requirement:** Jenkins needs to deploy apps but not delete infrastructure

```yaml
# CI/CD ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-deployer
  namespace: production

---
# CI/CD Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: production
rules:
# Can manage deployments
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch"]

# Can manage services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "create", "update", "patch"]

# Can read pods (for status checking)
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

# Can read pod logs (for debugging)
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]

# CANNOT delete deployments (safety!)
# CANNOT delete services
# CANNOT exec into pods (security!)
# CANNOT access secrets

---
# Bind to ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-deployer-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: jenkins-deployer
  namespace: production
roleRef:
  kind: Role
  name: deployer

Usage in Jenkins:
# Jenkins pod uses this ServiceAccount
apiVersion: v1
kind: Pod
metadata:
  name: jenkins-agent
  namespace: production
spec:
  serviceAccountName: jenkins-deployer
  containers:
  - name: jenkins
    image: jenkins/jenkins:lts

Result:
Jenkins can:
✓ Deploy new versions (update deployment)
✓ Create new services
✓ Check rollout status
✓ Read logs for debugging

Jenkins cannot:
❌ Delete production deployments (safety!)
❌ Access database secrets
❌ Exec into pods
❌ Modify RBAC rules (can't escalate privilege!)
```

---

## Secrets - Protecting Sensitive Data

### What are Secrets?

**Secrets = Base64-encoded data stored in etcd**

**Important: Secrets are NOT encrypted by default!**
- Base64 is encoding, not encryption
- Anyone with etcd access can read secrets
- Need encryption-at-rest for true security

```yaml
# Creating a Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
data:
  username: YWRtaW4=      # admin (base64)
  password: c3VwZXJzZWNyZXQ=  # supersecret (base64)

# Using Secret in Pod
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
  - name: app
    image: backend:latest
    env:
    # Method 1: Environment variable
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    # Better Method 2: Mount as file
    volumeMounts:
    - name: secrets
      mountPath: /secrets
      readOnly: true
  volumes:
  - name: secrets
    secret:
      secretName: db-credentials

Inside container:
# Environment variable
$ echo $DB_USERNAME
admin

# Or read from file
$ cat /secrets/username
admin
$ cat /secrets/password
supersecret
```

---

### Secret Security Problems

**Problem 1: Secrets in logs**
```bash
# BAD CODE
echo "Connecting to database with password: $DB_PASSWORD"

# Container logs
kubectl logs backend-pod
Connecting to database with password: supersecret

# EXPOSED IN LOGS! 😱
# Anyone with logs access sees the secret!

# GOOD CODE
echo "Connecting to database..."
# Don't log secrets!
```

**Problem 2: Secrets in environment variables**
```bash
# Environment variables visible in many ways:

# Method 1: kubectl describe
kubectl describe pod backend-pod
Environment:
  DB_PASSWORD: <set to the key 'password' in secret 'db-credentials'>

# Method 2: Exec into pod
kubectl exec backend-pod -- env | grep PASSWORD
DB_PASSWORD=supersecret

# Method 3: Process list
kubectl exec backend-pod -- ps aux
# Command line args might contain secrets!

# BETTER: Use volume mounts
# Secrets in files, not environment variables
# Harder to accidentally expose
```

**Problem 3: Secrets readable in etcd**
```bash
# Without encryption-at-rest:
# Direct access to etcd
etcdctl get /registry/secrets/production/db-credentials

# Returns plaintext (base64 encoded)!
# Anyone with etcd access = has all secrets!

# SOLUTION: Enable encryption-at-rest
# In kube-apiserver config:
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

---

### Better Secrets Management

**Option 1: External Secrets Operator**
```yaml
# Store secrets in AWS Secrets Manager / HashiCorp Vault
# Sync to Kubernetes automatically

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
  target:
    name: db-credentials
  data:
  - secretKey: password
    remoteRef:
      key: production/database/password

Benefits:
✓ Secrets stored in dedicated secrets manager
✓ Audit log of access
✓ Automatic rotation
✓ Encryption at rest
✓ Fine-grained access control
```

**Option 2: Sealed Secrets**
```yaml
# Encrypt secrets so they can be stored in Git

# 1. Install SealedSecrets controller
# 2. Seal your secret
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# 3. Sealed secret (safe to commit to Git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
spec:
  encryptedData:
    password: AgB7... (encrypted blob)

# 4. Controller decrypts and creates real Secret

Benefits:
✓ Secrets can be stored in Git (GitOps!)
✓ Only cluster can decrypt
✓ Encrypted with cluster's public key
```

---

## Pod Security

### Pod Security Standards (Baseline, Restricted)

**Three levels:**

**1. Privileged (no restrictions)**
```yaml
# Allows everything, including:
- Running as root
- Host network access
- Privileged containers
- Host path volumes

# Use for: System components only!
```

**2. Baseline (minimal restrictions)**
```yaml
# Blocks the most dangerous things:
❌ Privileged containers
❌ Host networking
❌ Host PID/IPC namespace
❌ Dangerous capabilities
✓ Can run as root (still allowed)

# Use for: Most workloads
```

**3. Restricted (very strict)**
```yaml
# Blocks almost everything unsafe:
❌ Running as root (must be non-root!)
❌ Privilege escalation
❌ All dangerous capabilities
❌ Host paths
❌ Writable root filesystem

# Use for: Production applications
```

---

### Enforcing Pod Security

**Namespace-level enforcement:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

Effect:
# Try to create privileged pod
kubectl apply -f privileged-pod.yaml -n production

# Blocked!
Error: pods "privileged-pod" is forbidden:
  violates PodSecurity "restricted:latest":
  allowPrivilegeEscalation != false,
  runAsNonRoot != true,
  securityContext not set

# Must create compliant pod:
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: cache
      mountPath: /cache
  volumes:
  - name: cache
    emptyDir: {}

This pod is accepted! ✓
```

---

### Real-World Security Example

**Scenario: Container Breakout Prevention**

```
Without security:
┌─────────────────────────────────────┐
│ Container running as root           │
│ Privileged mode enabled             │
│ Host filesystem mounted             │
└─────────────────────────────────────┘
       ↓
Attacker compromises container
       ↓
Attacker can:
✓ Access host filesystem
✓ Read other containers' data
✓ Escape to host system
✓ Compromise entire node! 💥

With security (restricted):
┌─────────────────────────────────────┐
│ Container running as user 1000      │
│ No privilege escalation             │
│ Read-only root filesystem           │
│ Dropped all Linux capabilities      │
│ seccomp profile (syscall filtering) │
└─────────────────────────────────────┘
       ↓
Attacker compromises container
       ↓
Attacker tries to:
❌ Escalate privileges → Blocked
❌ Write to filesystem → Read-only
❌ Access host → No permissions
❌ Use dangerous syscalls → Blocked

Attacker trapped in container! ✓
Blast radius minimized!
```

---

## Network Policies - Container Firewalls

### Default: All pods can talk to all pods

**Problem:**
```
Frontend pod → Can access database directly! 😱
Random pod → Can access payment service! 😱
Compromised pod → Can scan entire cluster! 😱
```

**Solution: Network Policies**

```yaml
# Deny all traffic to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-isolation
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Only backend can connect to database
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
  egress:
  # Database can only respond (no outbound connections)
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

Effect:
✓ Backend → Database: ALLOWED
❌ Frontend → Database: BLOCKED
❌ Random pod → Database: BLOCKED
❌ Database → Internet: BLOCKED

Security through network segmentation!
```

---

### Complete Security Example - E-commerce App

```yaml
# 1. Frontend Network Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Accept traffic from internet (via ingress controller)
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
  egress:
  # Can only call backend API
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

---
# 2. Backend Network Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Accept from frontend only
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
  egress:
  # Can call database
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  # Can call external payment API
  - to:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 443
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

---
# 3. Database Network Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Only backend can access
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  egress:
  # Database cannot initiate connections!
  # (Except DNS for resolving hostnames)
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

Result:
Internet → Frontend: ✓
Frontend → Backend: ✓
Backend → Database: ✓
Backend → Payment API: ✓

Frontend → Database: ❌ BLOCKED
Frontend → Payment API: ❌ BLOCKED
Database → Anywhere: ❌ BLOCKED
Random pod → Anything: ❌ BLOCKED

Micro-segmentation achieved! 🛡️
```

---

## Real Security Incidents

### Incident 1: Tesla Kubernetes Breach (2018)

```
What happened:
1. Kubernetes dashboard exposed to internet
2. No authentication configured
3. Attacker accessed dashboard
4. Full cluster admin access!
5. Deployed cryptocurrency miners
6. Stole AWS credentials from environment variables

Timeline:
Day 1: Dashboard exposed
Day 30: Attacker discovers it
Day 35: Miners deployed
Day 40: AWS bill spikes (noticed!)
Day 41: Investigation begins
Day 42: Breach discovered

Damage:
- $thousands in compute costs
- AWS credentials stolen
- Potential data access
- Reputation damage

Root causes:
❌ No authentication
❌ Dashboard exposed publicly
❌ No network policies
❌ Secrets in environment variables
❌ No resource limits (miners used all CPU)

Fixes applied:
✓ Dashboard behind VPN
✓ RBAC strictly configured
✓ Network policies implemented
✓ Secrets in Vault, not env vars
✓ Resource limits on all pods
✓ Regular security audits
```

---

### Incident 2: Capital One Breach (2019)

```
What happened:
1. Misconfigured WAF role (too permissive)
2. Attacker compromised WAF
3. Used WAF role to access S3
4. Downloaded 100M+ customer records

Not directly Kubernetes, but relevant lessons:
- Principle of least privilege!
- WAF should NOT have S3 access
- RBAC should be minimal
- Regular audits critical

In Kubernetes context:
ServiceAccounts should have minimal RBAC:
❌ cluster-admin for everything
✓ Specific roles for specific tasks

NetworkPolicies should isolate:
❌ All pods can talk to all pods
✓ Only necessary connections allowed
```

---

## Key Takeaways

### The 5 Most Important Security Concepts

**1. Defense in Depth (Multiple Layers)**
```
Layer 1: Authentication (who are you?)
Layer 2: Authorization/RBAC (what can you do?)
Layer 3: Pod Security (how can you run?)
Layer 4: Network Policies (who can you talk to?)
Layer 5: Secrets Management (how is data protected?)

If attacker bypasses one layer, others still protect!
```

**2. Principle of Least Privilege**
```
Give minimum permissions needed, nothing more!

❌ cluster-admin for CI/CD
✓ deployer role (create/update only)

❌ All pods can access secrets
✓ Only pods that need them

❌ Root user in containers
✓ Non-root user (UID 1000)

Less privilege = Less damage if compromised!
```

**3. Secrets are NOT Encrypted by Default**
```
Kubernetes Secrets = Base64 encoding
Base64 ≠ Encryption!

Solutions:
✓ Enable encryption-at-rest
✓ Use external secrets manager (Vault)
✓ Use Sealed Secrets for GitOps
✓ Never log secrets
✓ Prefer volume mounts over env vars
```

**4. Network Policies are Essential**
```
Default: All pods can talk to all pods
This is DANGEROUS!

Always implement NetworkPolicies:
✓ Deny all by default
✓ Allow only necessary connections
✓ Isolate tiers (frontend/backend/database)
✓ Block egress to internet (where possible)

Micro-segmentation = Contained breaches!
```

**5. Regular Security Audits**
```
Check monthly:
- Who has cluster-admin?
- Are there overly permissive roles?
- Are secrets properly managed?
- Are Network Policies in place?
- Are pods running as root?
- Are resource limits set?

Tools:
- kubectl-who-can (check permissions)
- kube-bench (CIS benchmarks)
- Polaris (best practices)
- Falco (runtime security)
```

---

## Tomorrow (Day 9)

We'll explore:
- Logging and log aggregation
- Metrics and monitoring (Prometheus)
- Debugging techniques
- Troubleshooting common issues
- Observability best practices

**You now understand Kubernetes security - how to protect your cluster from attacks! Tomorrow we'll learn how to observe what's happening in your cluster.** 🔒