# 🩸 Blood, Sweat, and Kubernetes

> **"Learn Kubernetes by breaking it in 33+ production-like disaster scenarios. You'll cry, you'll rage-quit, you'll come back stronger. 19 days from zero to hero."**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](http://makeapullrequest.com)
[![Kubernetes](https://img.shields.io/badge/kubernetes-v1.28+-blue.svg)](https://kubernetes.io/)
[![Chaos Level](https://img.shields.io/badge/chaos-maximum-red.svg)](/)
![Tears Shed](https://img.shields.io/badge/tears%20shed-countless-blue)
![Clusters Destroyed](https://img.shields.io/badge/clusters%20destroyed-33+-red)
![Production Ready](https://img.shields.io/badge/production%20ready-yes-green)

**Warning**: This is not a gentle introduction to Kubernetes. This is a brutal, hands-on bootcamp where you'll break production-grade clusters and fix them with your bare hands. If you survive all 19 days, you'll understand Kubernetes better than 90% of "Senior DevOps Engineers."

---

## 🔥 What Makes This Different?

Most Kubernetes tutorials:
- ✨ Show you the happy path
- 🌈 Assume everything works perfectly  
- 📚 Teach theory without real-world pain

**This course:**
- 💀 Intentionally breaks things
- 🔥 Simulates production disasters
- 🩸 Forces you to debug and fix
- 💪 Builds muscle memory through pain

**You'll experience:**
- Kill the container runtime, watch pods die
- Corrupt etcd, recover from disaster
- Delete CNI, break all networking
- Crash the scheduler, see pods stuck forever
- Stop kubelet, trigger node failure
- Misconfigure Ingress, debug 503 errors
- And 27 more realistic failure scenarios

---

## 📋 Prerequisites

- **Basic Linux knowledge** - Can use terminal, SSH, understand processes
- **Basic Docker knowledge** - Understand containers, images
- **Time commitment** - 3-4 hours per day for 19 days
- **Mental fortitude** - Ability to push through frustration
- **Laptop/Desktop** - 8GB RAM minimum, 16GB recommended

**No prior Kubernetes knowledge required** - We start from zero.

---

## 🛠️ Setup

### Quick Start (Recommended)
```bash
# Clone repository
git clone https://github.com/yourusername/blood-sweat-and-kubernetes.git
cd blood-sweat-and-kubernetes

# Run setup script
chmod +x scripts/setup-minikube.sh
./scripts/setup-minikube.sh

# Verify setup
kubectl version
minikube status
```

### Alternative Setup Options

#### Option 1: Kind (Kubernetes in Docker)
```bash
chmod +x scripts/setup-kind.sh
./scripts/setup-kind.sh
```

#### Option 2: Manual Setup

**Prerequisites:**
- 2 CPUs or more
- 2GB of free memory
- 20GB of free disk space
- Internet connection
- Container or virtual machine manager (Docker, QEMU, Hyperkit, Hyper-V, KVM, Parallels, Podman, VirtualBox, or VMware Fusion/Workstation)

**Step 1: Detect Your Architecture**
```bash
# Check your system architecture
uname -m
# Output examples:
# x86_64 (Intel/AMD 64-bit)
# arm64 (Apple Silicon, ARM 64-bit)
# aarch64 (ARM 64-bit on Linux)
```

**Step 2: Install kubectl**
```bash
# For Linux x86_64
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# For macOS x86_64
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# For macOS ARM64 (Apple Silicon)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

**Step 3: Install minikube**
```bash
# For Linux x86_64
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# For macOS x86_64
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube

# For macOS ARM64 (Apple Silicon)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-arm64
sudo install minikube-darwin-arm64 /usr/local/bin/minikube

# For Linux ARM64
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-arm64
sudo install minikube-linux-arm64 /usr/local/bin/minikube
```

**Step 4: Start Cluster**
```bash
# Start minikube with Docker driver
minikube start --driver=docker --memory=4096 --cpus=2

# Alternative: Start with specific Kubernetes version
minikube start --driver=docker --memory=4096 --cpus=2 --kubernetes-version=v1.28.3
```

**📖 For Complete Installation Instructions:**
Visit the [official minikube documentation](https://minikube.sigs.k8s.io/docs/start/) for detailed installation steps for all platforms including Windows, Linux, and macOS with different architectures.

---

## 📚 Course Structure

### Week 1: Foundation & Core Components
- **Day 1**: Foundation - Cluster setup, kubectl basics, YAML fundamentals
- **Day 2**: API Server & etcd - The brain of Kubernetes
- **Day 3**: Scheduler - How pods get placed on nodes
- **Day 4**: Controller Manager - The automation engine
- **Day 5**: Kubelet - The node agent

### Week 2: Networking & Storage
- **Day 6**: Networking - CNI, Services, DNS, Network Policies
- **Day 7**: Storage - Volumes, PVs, PVCs, Storage Classes
- **Day 8**: Security - RBAC, Pod Security, Network Policies
- **Day 9**: Observability - Logging, Metrics, Tracing, Debugging

### Week 3: Advanced Workloads & Operations
- **Day 10**: Advanced Scheduling - Node affinity, taints, tolerations
- **Day 11**: StatefulSets - Managing stateful applications
- **Day 12**: Jobs & CronJobs - Batch processing and scheduled tasks
- **Day 13**: Multi-cluster - Federation, GitOps, disaster recovery

### Week 4: Production Readiness
- **Day 14**: Production Readiness - Health checks, resource limits, monitoring
- **Day 15**: Ingress & Traffic Management - Load balancing, SSL termination
- **Day 16**: Configuration Management - ConfigMaps, Secrets, Helm
- **Day 17**: Advanced Patterns - Operators, Custom Resources, Webhooks

### Week 5: Operations & Service Mesh
- **Day 18**: Upgrades & Maintenance - Rolling updates, cluster upgrades
- **Day 19**: Service Mesh - Istio, traffic management, security

---

## 🎯 Learning Objectives

By the end of this course, you will:

### Technical Skills
- ✅ Deploy and manage production-grade Kubernetes clusters
- ✅ Debug complex networking and storage issues
- ✅ Implement security best practices and compliance
- ✅ Design resilient, scalable architectures
- ✅ Automate deployments with GitOps workflows
- ✅ Monitor and troubleshoot cluster health
- ✅ Handle disaster recovery scenarios

### Operational Skills
- ✅ Incident response and troubleshooting
- ✅ Performance optimization and capacity planning
- ✅ Security hardening and vulnerability management
- ✅ Backup and disaster recovery procedures
- ✅ Multi-cluster management strategies

### Mental Resilience
- ✅ Stay calm under pressure during outages
- ✅ Think systematically when debugging
- ✅ Learn from failures and improve processes
- ✅ Communicate effectively during incidents

---

## 🚨 Disaster Scenarios

Each day includes 2-3 realistic failure scenarios:

### Infrastructure Failures
- Container runtime crashes
- etcd corruption and recovery
- Network partition simulation
- Storage failures and data loss
- Node failures and pod eviction

### Application Failures
- Resource exhaustion scenarios
- Memory leaks and OOM kills
- Database connection failures
- Service mesh configuration errors
- Ingress misconfigurations

### Security Incidents
- RBAC misconfigurations
- Network policy violations
- Secret exposure scenarios
- Pod security policy failures
- Admission controller bypasses

### Operational Disasters
- Rolling update failures
- Configuration drift detection
- Backup restoration procedures
- Multi-cluster split-brain scenarios
- Certificate expiration chaos

---

## 📖 How to Use This Course

### Daily Structure
1. **Morning**: Read the day's documentation (30-45 minutes)
2. **Lab Time**: Complete hands-on exercises (2-3 hours)
3. **Disaster Scenarios**: Break things and fix them (1-2 hours)
4. **Evening**: Review solutions and reflect (30 minutes)

### Pro Tips
- **Don't skip the disasters** - They're the most valuable part
- **Take notes** - Document your debugging process
- **Ask questions** - Use GitHub issues for help
- **Share your pain** - Join our community discussions
- **Practice daily** - Consistency beats intensity

### Getting Help
- 📖 Check the [solutions directory](solutions/) for answers
- 🐛 Open a [GitHub issue](https://github.com/yourusername/blood-sweat-and-kubernetes/issues) for bugs
- 💬 Join our [Discord community](https://discord.gg/your-invite) for discussions
- 📧 Email us at support@bloodsweatkubernetes.com

---

## 🏆 Certification

Complete all 19 days and submit your final project to earn the **"Blood, Sweat, and Kubernetes Survivor"** certificate. This isn't just a participation trophy - you'll have actually broken and fixed production-grade infrastructure.

### Final Project
Deploy a complete microservices application with:
- 5+ services with proper networking
- Database with persistent storage
- Monitoring and alerting
- Security policies and RBAC
- CI/CD pipeline
- Disaster recovery procedures

---

## 🤝 Contributing

We welcome contributions! See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

### Ways to Contribute
- 🐛 Report bugs and issues
- 📝 Improve documentation
- 🧪 Add new disaster scenarios
- 🔧 Fix setup scripts
- 🎨 Improve diagrams and visuals

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## 🙏 Acknowledgments

- Kubernetes community for the amazing platform
- All the engineers who shared their war stories
- The brave souls who tested this course and survived
- Contributors who made this project better

---

## ⚠️ Disclaimer

This course involves intentionally breaking production-like environments. Always:
- Use isolated test environments
- Never run these scenarios on production clusters
- Backup your work before starting each day
- Have a recovery plan ready

**Use at your own risk. We are not responsible for any data loss or system damage.**

---

## 📊 Course Statistics

- **Total Labs**: 57+ hands-on exercises
- **Disaster Scenarios**: 33+ realistic failures
- **Estimated Time**: 60-80 hours over 19 days
- **Average Tears Shed**: 3.2 per day (Day 6 is the worst)

---

**Ready to begin your journey? Start with [Day 1: Foundation](docs/day-01-foundation.md)**

*"The best way to learn Kubernetes is to break it, fix it, and break it again. Welcome to the chaos."*

## 📝 Note on Deprecated Commands

Some commands in this course use `kubectl run` which is deprecated but still functional. For modern alternatives:
- Use `kubectl create deployment` + `kubectl exec` instead of `kubectl run`
- Use `kubectl apply -f` for complex pod specifications
- Use `kubectl debug` for debugging scenarios
