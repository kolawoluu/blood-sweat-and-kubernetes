# Day 15: Ingress & Traffic Management

## The Hotel Reception Analogy

Imagine a large hotel with hundreds of rooms:
- **LoadBalancer Service** = Separate entrance for each room (expensive!)
- **NodePort Service** = Random doors scattered around building (confusing!)
- **Ingress** = Single main reception desk that directs guests to correct room

```
┌──────────────────────────────────────────────────────┐
│              HOTEL (KUBERNETES CLUSTER)               │
│                                                       │
│  WITHOUT INGRESS (Expensive & Messy):                │
│  ┌────────────────────────────────────┐              │
│  │ Room 101 (web-app)                 │              │
│  │ Has own entrance + guard           │              │
│  │ Cost: $100/month (LoadBalancer)    │              │
│  └────────────────────────────────────┘              │
│  ┌────────────────────────────────────┐              │
│  │ Room 102 (api)                     │              │
│  │ Has own entrance + guard           │              │
│  │ Cost: $100/month (LoadBalancer)    │              │
│  └────────────────────────────────────┘              │
│  ┌────────────────────────────────────┐              │
│  │ Room 103 (admin)                   │              │
│  │ Has own entrance + guard           │              │
│  │ Cost: $100/month (LoadBalancer)    │              │
│  └────────────────────────────────────┘              │
│                                                       │
│  Total: $300/month for 3 services! 💸                │
│                                                       │
│  WITH INGRESS (Smart & Efficient):                   │
│  ┌────────────────────────────────────┐              │
│  │     MAIN RECEPTION (Ingress)       │              │
│  │     One entrance, one guard        │              │
│  │     Cost: $100/month               │              │
│  │                                    │              │
│  │ Guest says: "I want web-app"       │              │
│  │ Reception: "Room 101, this way →"  │              │
│  │                                    │              │
│  │ Guest says: "I want api"           │              │
│  │ Reception: "Room 102, this way →"  │              │
│  │                                    │              │
│  │ Routes by:                         │              │
│  │ - Domain name (web.example.com)    │              │
│  │ - Path (/api, /admin)              │              │
│  │ - Headers, methods, etc.           │              │
│  └────────────────────────────────────┘              │
│           ↓           ↓           ↓                  │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐           │
│  │Room 101 │   │Room 102 │   │Room 103 │           │
│  │web-app  │   │  api    │   │ admin   │           │
│  └─────────┘   └─────────┘   └─────────┘           │
│                                                       │
│  Total: $100/month for 3 services! 💰                │
│  Savings: $200/month (67% reduction)                 │
└──────────────────────────────────────────────────────┘
```

---

## What is Ingress?

**Ingress = Layer 7 (HTTP/HTTPS) load balancer + router**

```
Internet Traffic:
- https://web.example.com
- https://api.example.com
- https://admin.example.com
       ↓
┌─────────────────────────────────────┐
│    SINGLE LOAD BALANCER             │
│    (AWS ELB/ALB, GCP LB, etc.)      │
│    Public IP: 54.123.45.67          │
└─────────────────────────────────────┘
       ↓
┌─────────────────────────────────────┐
│    INGRESS CONTROLLER               │
│    (nginx, Traefik, HAProxy)        │
│    Running inside cluster           │
│                                     │
│    Routing rules:                   │
│    web.example.com → web-service    │
│    api.example.com → api-service    │
│    admin.example.com → admin-service│
└─────────────────────────────────────┘
       ↓       ↓        ↓
    ┌────┐  ┌────┐  ┌────┐
    │Web │  │API │  │Admin│
    │Pods│  │Pods│  │Pods│
    └────┘  └────┘  └────┘

One entry point, multiple services!
```

---

## Ingress Components

### 1. Ingress Resource (Configuration)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  # Rule 1: Host-based routing
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  
  # Rule 2: Different host
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  
  # Rule 3: Path-based routing (same host)
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 3000
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80

What this does:
Request to https://web.example.com
→ Routes to web-service:80

Request to https://api.example.com
→ Routes to api-service:8080

Request to https://app.example.com/api/users
→ Routes to api-service:8080

Request to https://app.example.com/admin
→ Routes to admin-service:3000

Request to https://app.example.com/
→ Routes to web-service:80
```

---

### 2. Ingress Controller (The Worker)

**Ingress Resource = Configuration (what to do)**
**Ingress Controller = Implementation (does the work)**

**Popular Ingress Controllers:**

```
1. Nginx Ingress Controller (Most Popular)
   ✓ Mature, battle-tested
   ✓ Rich features
   ✓ Good performance
   ✓ Widely documented
   Use for: General purpose, most cases

2. Traefik
   ✓ Modern, dynamic configuration
   ✓ Great dashboard
   ✓ Automatic service discovery
   ✓ Built-in Let's Encrypt
   Use for: Dynamic environments, microservices

3. HAProxy Ingress
   ✓ Extremely fast (C-based)
   ✓ Low resource usage
   ✓ Enterprise-grade
   Use for: High-performance needs

4. AWS ALB Ingress Controller
   ✓ Native AWS integration
   ✓ Uses Application Load Balancer
   ✓ AWS-specific features
   Use for: AWS EKS clusters

5. Istio Ingress Gateway
   ✓ Service mesh integration
   ✓ Advanced traffic management
   ✓ mTLS, observability
   Use for: If already using Istio

6. Kong Ingress
   ✓ API Gateway features
   ✓ Plugins (auth, rate limiting, etc.)
   ✓ Enterprise API management
   Use for: API-heavy architectures
```

---

### Installing Nginx Ingress Controller

```bash
# Install via Helm (easiest)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# What gets created:
kubectl get all -n ingress-nginx

NAME                                            READY   STATUS
pod/ingress-nginx-controller-abc123             1/1     Running

NAME                                         TYPE           EXTERNAL-IP
service/ingress-nginx-controller             LoadBalancer   54.123.45.67
service/ingress-nginx-controller-admission   ClusterIP      10.96.0.100

NAME                                       READY
deployment.apps/ingress-nginx-controller   1/1

# The LoadBalancer gets a public IP
# Point your DNS to this IP:
# web.example.com → 54.123.45.67
# api.example.com → 54.123.45.67
# admin.example.com → 54.123.45.67

# All domains → Same IP!
# Ingress Controller routes based on Host header
```

---

## Path Types Explained

```yaml
pathType: Prefix
# Matches: /api, /api/, /api/users, /api/v2/users
# Basically: starts with /api

pathType: Exact
# Matches: /api ONLY
# Not: /api/, /api/users

pathType: ImplementationSpecific
# Depends on Ingress Controller
# Usually similar to Prefix

Example:
paths:
- path: /api
  pathType: Prefix
  backend:
    service:
      name: api-service

Requests:
✓ /api → api-service
✓ /api/ → api-service
✓ /api/users → api-service
✓ /api/v1/products → api-service
❌ /apiv2 → NOT matched (doesn't start with /api/)
```

---

## SSL/TLS Termination

**Ingress handles HTTPS for you!**

```yaml
# Create TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>

---
# Ingress with TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - web.example.com
    - api.example.com
    secretName: tls-secret
  rules:
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80

Traffic flow:
User → https://web.example.com (encrypted)
       ↓
Ingress Controller:
- Terminates TLS (decrypts)
- Uses certificate from tls-secret
       ↓
Forwards to web-service (HTTP, inside cluster)
       ↓
web-service responds
       ↓
Ingress Controller encrypts response
       ↓
User receives HTTPS response

Benefits:
✓ Single place to manage certificates
✓ Services don't need to handle TLS
✓ Automatic renewal (with cert-manager)
```

---

## Automatic Certificate Management

**cert-manager = Automatic Let's Encrypt certificates**

```yaml
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Create ClusterIssuer (Let's Encrypt)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx

---
# Ingress with automatic certificate
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auto-tls-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - web.example.com
    secretName: web-tls  # cert-manager creates this automatically!
  rules:
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80

What happens:
1. You create Ingress with annotation
2. cert-manager sees annotation
3. cert-manager requests certificate from Let's Encrypt
4. Let's Encrypt validates domain ownership (HTTP challenge)
5. Certificate issued
6. cert-manager stores in Secret "web-tls"
7. Ingress uses certificate
8. HTTPS works! ✓
9. cert-manager auto-renews before expiry (90 days)

Free, automatic, always valid certificates! 🎉
```

---

## Advanced Routing

### 1. Header-Based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: header-routing
  annotations:
    nginx.ingress.kubernetes.io/server-snippet: |
      if ($http_x_version = "v2") {
        set $service_name "api-v2-service";
      }
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-v1-service  # Default
            port:
              number: 80

# Request with Header: X-Version: v2
# → Routes to api-v2-service

# Request without header
# → Routes to api-v1-service (default)

Use case: Canary deployments, A/B testing
```

---

### 2. Rate Limiting

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-limited-api
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"  # 10 requests/sec per IP
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80

# IP exceeds 10 req/sec
# Response: 503 Service Temporarily Unavailable

Protects your API from:
✓ Abuse
✓ DDoS attacks
✓ Accidental loops
✓ Misbehaving clients
```

---

### 3. Authentication (Basic Auth)

```yaml
# Create auth secret
htpasswd -c auth username
# Type password when prompted

kubectl create secret generic basic-auth \
  --from-file=auth \
  -n default

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: protected-admin
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required'
spec:
  rules:
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80

# Access admin.example.com
# Browser prompts for username/password
# Wrong credentials → 401 Unauthorized

Protects admin panels, internal tools!
```

---

### 4. URL Rewriting

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80

# User requests: /api/users
# Ingress rewrites to: /users
# api-service receives: /users (not /api/users)

Why?
- Service expects /users, not /api/users
- Clean URLs for service
- Ingress handles path prefix
```

---

## Real-World Example: Multi-Service Application

```yaml
# E-commerce platform with multiple services

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"  # 100 req/sec
    nginx.ingress.kubernetes.io/ssl-redirect: "true"  # Force HTTPS
spec:
  tls:
  - hosts:
    - shop.example.com
    secretName: shop-tls
  rules:
  - host: shop.example.com
    http:
      paths:
      # Homepage (React SPA)
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
      
      # Product API
      - path: /api/products
        pathType: Prefix
        backend:
          service:
            name: product-service
            port:
              number: 8080
      
      # User API
      - path: /api/users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 8080
      
      # Payment API (stricter rate limit)
      - path: /api/payments
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 8080
      
      # Admin panel (protected with auth)
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-panel
            port:
              number: 3000
      
      # Static assets (cached)
      - path: /static
        pathType: Prefix
        backend:
          service:
            name: cdn-service
            port:
              number: 80

Traffic routing:
shop.example.com/ → frontend (SPA loads)
shop.example.com/api/products → product-service
shop.example.com/api/users → user-service
shop.example.com/api/payments → payment-service
shop.example.com/admin → admin-panel
shop.example.com/static/logo.png → cdn-service

One domain, six services! ✨
```

---

## Service Types Comparison

```
┌─────────────────────────────────────────────────────┐
│  SERVICE TYPE          WHEN TO USE                  │
├─────────────────────────────────────────────────────┤
│  ClusterIP (Default)   Internal only                │
│  - No external access                               │
│  - Service-to-service                               │
│  - Database, cache, internal APIs                   │
│  Cost: Free                                         │
├─────────────────────────────────────────────────────┤
│  NodePort              Development, testing         │
│  - Access via Node IP + Port                        │
│  - Not for production (security)                    │
│  - Useful for local testing                         │
│  Cost: Free                                         │
├─────────────────────────────────────────────────────┤
│  LoadBalancer          Single service exposure      │
│  - Gets external IP                                 │
│  - One LoadBalancer per service                     │
│  - Expensive ($20/month each)                       │
│  Cost: High ($20+ per service)                      │
├─────────────────────────────────────────────────────┤
│  Ingress               Multiple services (HTTP)     │
│  - Single LoadBalancer                              │
│  - Routes to many services                          │
│  - SSL/TLS, auth, rate limiting                     │
│  Cost: Low ($20/month total)                        │
└─────────────────────────────────────────────────────┘

Cost Comparison (10 services):

Without Ingress (LoadBalancer per service):
10 services × $20 = $200/month 💸

With Ingress (Single LoadBalancer):
1 Ingress × $20 = $20/month 💰

Savings: $180/month (90%!)
```

---

## Troubleshooting Ingress

### Problem 1: 404 Not Found

```bash
# Check Ingress exists
kubectl get ingress
NAME            HOSTS              ADDRESS
main-ingress    web.example.com    54.123.45.67

# Describe Ingress
kubectl describe ingress main-ingress
# Check: Rules configured correctly?
# Check: Backend service exists?

# Check Ingress Controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Common issues:
❌ Wrong path in Ingress rule
❌ Service doesn't exist
❌ Service selector doesn't match pods
❌ DNS not pointing to LoadBalancer IP
```

---

### Problem 2: SSL Certificate Issues

```bash
# Check certificate
kubectl get certificate
NAME       READY   SECRET     AGE
web-tls    True    web-tls    5d

# If not ready:
kubectl describe certificate web-tls
# Check challenge status

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager

# Common issues:
❌ Domain not pointing to cluster
❌ HTTP challenge blocked by firewall
❌ Rate limit (Let's Encrypt: 5 certs/week)
❌ DNS propagation not complete
```

---

### Problem 3: Backend Service Unhealthy

```bash
# Check if Ingress can reach service
kubectl get svc web-service
NAME          TYPE        CLUSTER-IP     PORT(S)
web-service   ClusterIP   10.96.0.100    80/TCP

# Check service endpoints
kubectl get endpoints web-service
NAME          ENDPOINTS
web-service   10.244.1.5:80,10.244.1.6:80

# If no endpoints:
❌ No pods match service selector
❌ Pods not Ready (failing health checks)

# Test from Ingress Controller
kubectl exec -n ingress-nginx deployment/ingress-nginx-controller -- curl http://web-service.default.svc.cluster.local
```

---

## Key Takeaways

**1. Ingress = Single Entry Point**
```
One LoadBalancer → Many services
Saves money (90% for 10 services!)
Centralized management
Easy SSL/TLS
```

**2. Ingress Controller Required**
```
Ingress Resource = Config (what)
Ingress Controller = Worker (how)

Popular: Nginx, Traefik, HAProxy
Choose based on needs
```

**3. Powerful Routing**
```
✓ Host-based (web.com vs api.com)
✓ Path-based (/api, /admin)
✓ Header-based (canary, A/B)
✓ SSL/TLS termination
✓ Authentication
✓ Rate limiting
```

**4. cert-manager for Free SSL**
```
Automatic Let's Encrypt certificates
Auto-renewal (no expired certs!)
Just add annotation
HTTPS everywhere! ✓
```

**5. Production Essential**
```
Every production cluster needs Ingress!
Don't use LoadBalancer per service
Cost optimization
Flexibility
```

---

## Tomorrow (Day 16)

We'll explore:
- ConfigMaps (configuration management)
- Secrets best practices
- Helm (Kubernetes package manager)
- Templating and releases
- Chart repositories

**You now understand how traffic gets INTO your cluster and routes to the right services!** 🌐