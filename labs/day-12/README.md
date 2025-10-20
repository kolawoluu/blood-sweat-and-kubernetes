# Day 12 Lab: Jobs & CronJobs - The Task Scheduler

> **"Today you'll break Jobs and CronJobs and watch tasks fail. You'll learn why Jobs and CronJobs are the task scheduler of Kubernetes."**

## 🎯 Learning Objectives

By the end of this lab, you will:
- Watch Jobs and CronJobs work in real-time
- Break Jobs and CronJobs and see tasks fail
- Test job failures and retries
- Practice job procedures
- Understand the job architecture
- Learn why Jobs and CronJobs are critical for batch processing

## ⚠️ Prerequisites

- Day 11 lab completed
- Minikube cluster running
- 4-5 hours of uninterrupted time
- Understanding of basic scheduling concepts

## 🛠️ Setup (30 minutes)

### Step 1: Verify Cluster Status

```bash
# Check cluster is running
kubectl get nodes
kubectl get pods -n kube-system

# Clean up previous lab resources
kubectl delete statefulset web init-test complex-statefulset -n stateful-test
kubectl delete service web init-test complex-statefulset -n stateful-test
kubectl delete pvc --all -n stateful-test
kubectl delete pv --all
kubectl delete pdb web-pdb -n stateful-test
kubectl delete namespace stateful-test
```

### Step 2: Setup Monitoring

```bash
# Open 4 terminals
# Terminal 1: kubectl get pods --all-namespaces --watch
# Terminal 2: kubectl get events --watch --sort-by='.lastTimestamp'
# Terminal 3: kubectl get jobs,cronjobs --watch
# Terminal 4: Your command terminal
```

### Step 3: Check Job Components

```bash
# Check if job controller is running
kubectl get pods -n kube-system | grep controller-manager

# Check job controller logs
kubectl logs kube-controller-manager-minikube -n kube-system
```

---

## 🧪 Lab Part 1: Watch Jobs Work (45 minutes)

### Exercise 1.1: Create Simple Job

```bash
# Create namespace for testing
kubectl create namespace job-test

# Create simple job
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: simple-job
  namespace: job-test
spec:
  template:
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'echo "Hello from job" && sleep 10']
      restartPolicy: Never
EOF

# Check job status
kubectl get job simple-job -n job-test
# NAME         COMPLETIONS   DURATION   AGE
# simple-job   1/1          10s        30s
```

### Exercise 1.2: Watch Job Execution

```bash
# Watch job pods
kubectl get pods -l job-name=simple-job -n job-test --watch

# You should see:
# simple-job-xxx   Pending
# simple-job-xxx   ContainerCreating
# simple-job-xxx   Running
# simple-job-xxx   Completed

# Check job logs
kubectl logs -l job-name=simple-job -n job-test
# Should show: Hello from job
```

### Exercise 1.3: Create Parallel Job

```bash
# Create parallel job
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
  namespace: job-test
spec:
  parallelism: 3
  completions: 6
  template:
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'echo "Hello from parallel job" && sleep 5']
      restartPolicy: Never
EOF

# Check job status
kubectl get job parallel-job -n job-test
# NAME           COMPLETIONS   DURATION   AGE
# parallel-job    6/6          15s        30s

# Check job pods
kubectl get pods -l job-name=parallel-job -n job-test
# Should show 6 completed pods
```

### Exercise 1.4: Create CronJob

```bash
# Create CronJob
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cronjob
  namespace: job-test
spec:
  schedule: "*/1 * * * *"  # Every minute
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: busybox
            image: busybox
            command: ['sh', '-c', 'echo "Hello from cronjob" && date']
          restartPolicy: OnFailure
EOF

# Check CronJob status
kubectl get cronjob hello-cronjob -n job-test
# NAME            SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# hello-cronjob   */1 * * * *   False     0        <none>          30s
```

---

## 💀 Lab Part 2: Break #14 - Stop Job Controller (1 hour)

> **"Time to break the job controller. This is where things get really scary."**

### Exercise 2.1: Prepare for Chaos

```bash
# Create test job
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: chaos-job
  namespace: job-test
spec:
  template:
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'echo "Chaos job" && sleep 30']
      restartPolicy: Never
EOF

# Check job status
kubectl get job chaos-job -n job-test
# NAME        COMPLETIONS   DURATION   AGE
# chaos-job   0/1          10s        10s
```

### Exercise 2.2: Break Job Controller

```bash
# Delete controller manager pod
kubectl delete pod -n kube-system kube-controller-manager-minikube

# Wait 10 seconds, then prevent it from restarting
minikube ssh
sudo mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
exit
```

### Exercise 2.3: Observe the Chaos

**Watch Terminal 1 (pods):**
- Existing job pods keep running
- No new job operations

**Test job operations:**
```bash
# Try to create new job
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: test-job
  namespace: job-test
spec:
  template:
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'echo "Test job"']
      restartPolicy: Never
EOF

# Check job status
kubectl get job test-job -n job-test
# COMPLETIONS: 0/1 (not running!)

# Check pods
kubectl get pods -l job-name=test-job -n job-test
# No pods created!
```

### Exercise 2.4: Debug the Problem

```bash
# Check controller manager
kubectl get pods -n kube-system | grep controller-manager
# NOT FOUND!

# Check job status
kubectl describe job test-job -n job-test
# Should show: FailedCreate

# Check events
kubectl get events -n job-test
# Should show: FailedCreate
```

**Key Learning:** 
- Job controller is critical for job operations
- No controller = no job execution
- Existing job pods unaffected
- Jobs become "frozen"

### Exercise 2.5: Fix It

```bash
minikube ssh
sudo mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/
exit

# Wait for controller manager to restart
kubectl get pods -n kube-system | grep controller-manager
# Running!

# Check job status
kubectl get job test-job -n job-test
# COMPLETIONS: 1/1

# Check pods
kubectl get pods -l job-name=test-job -n job-test
# Pod: Completed
```

**Real-world parallel:** This happens when:
- Controller manager crashes
- Resource exhaustion
- Configuration errors
- Network issues

---

## 🔥 Lab Part 3: Job Failures and Retries (1 hour)

> **"Time to test what happens when jobs fail. This is where job resilience is tested."**

### Exercise 3.1: Test Job Failure

```bash
# Create failing job
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: failing-job
  namespace: job-test
spec:
  backoffLimit: 3
  template:
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'echo "Failing job" && exit 1']
      restartPolicy: Never
EOF

# Watch job execution
kubectl get pods -l job-name=failing-job -n job-test --watch

# You should see:
# failing-job-xxx   Pending
# failing-job-xxx   ContainerCreating
# failing-job-xxx   Running
# failing-job-xxx   Failed
# failing-job-yyy   Pending
# failing-job-yyy   ContainerCreating
# failing-job-yyy   Running
# failing-job-yyy   Failed
# failing-job-zzz   Pending
# failing-job-zzz   ContainerCreating
# failing-job-zzz   Running
# failing-job-zzz   Failed
# failing-job-aaaa  Pending
# failing-job-aaaa  ContainerCreating
# failing-job-aaaa  Running
# failing-job-aaaa  Failed

# Job retries 3 times (backoffLimit: 3)
```

### Exercise 3.2: Test Job Success After Failure

```bash
# Create job that succeeds after failures
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: retry-job
  namespace: job-test
spec:
  backoffLimit: 5
  template:
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'if [ -f /tmp/fail ]; then echo "Still failing" && exit 1; else echo "Success" && touch /tmp/fail; fi']
      restartPolicy: Never
EOF

# Watch job execution
kubectl get pods -l job-name=retry-job -n job-test --watch

# Job should fail first time, then succeed
```

### Exercise 3.3: Test Job Timeout

```bash
# Create job with timeout
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: timeout-job
  namespace: job-test
spec:
  activeDeadlineSeconds: 30
  template:
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'echo "Long running job" && sleep 60']
      restartPolicy: Never
EOF

# Watch job execution
kubectl get pods -l job-name=timeout-job -n job-test --watch

# Job should be killed after 30 seconds
```

### Exercise 3.4: Test Job Cleanup

```bash
# Check job status
kubectl get jobs -n job-test
# Should show all jobs

# Delete completed jobs
kubectl delete job simple-job parallel-job chaos-job test-job failing-job retry-job timeout-job -n job-test

# Check job pods
kubectl get pods -n job-test
# Should show only CronJob pods
```

---

## 🧠 Lab Part 4: CronJob Management (1 hour)

> **"Time to test CronJob management. This is where you learn about scheduled tasks."**

### Exercise 4.1: Test CronJob Execution

```bash
# Wait for CronJob to execute
kubectl get cronjob hello-cronjob -n job-test
# Should show: ACTIVE: 1

# Check CronJob jobs
kubectl get jobs -l cronjob=hello-cronjob -n job-test
# Should show scheduled jobs

# Check CronJob pods
kubectl get pods -l cronjob=hello-cronjob -n job-test
# Should show completed pods
```

### Exercise 4.2: Test CronJob Suspension

```bash
# Suspend CronJob
kubectl patch cronjob hello-cronjob -n job-test -p '{"spec":{"suspend":true}}'

# Check CronJob status
kubectl get cronjob hello-cronjob -n job-test
# SUSPEND: True

# Wait for next schedule time
# CronJob should not create new jobs
```

### Exercise 4.3: Test CronJob Resume

```bash
# Resume CronJob
kubectl patch cronjob hello-cronjob -n job-test -p '{"spec":{"suspend":false}}'

# Check CronJob status
kubectl get cronjob hello-cronjob -n job-test
# SUSPEND: False

# Wait for next schedule time
# CronJob should create new jobs
```

### Exercise 4.4: Test CronJob History

```bash
# Check CronJob history
kubectl get jobs -l cronjob=hello-cronjob -n job-test

# Check CronJob status
kubectl describe cronjob hello-cronjob -n job-test
# Should show: Last Schedule Time

# Check CronJob logs
kubectl logs -l cronjob=hello-cronjob -n job-test
# Should show: Hello from cronjob
```

---

## 🎯 Lab Part 5: Advanced Job Scenarios (45 minutes)

### Exercise 5.1: Test Job with Init Container

```bash
# Create job with init container
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: init-job
  namespace: job-test
spec:
  template:
    spec:
      initContainers:
      - name: init
        image: busybox
        command: ['sh', '-c', 'echo "Init container" > /shared/init.txt']
        volumeMounts:
        - name: shared
          mountPath: /shared
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'cat /shared/init.txt && echo "Main container"']
        volumeMounts:
        - name: shared
          mountPath: /shared
      volumes:
      - name: shared
        emptyDir: {}
      restartPolicy: Never
EOF

# Check job status
kubectl get job init-job -n job-test
# COMPLETIONS: 1/1

# Check job logs
kubectl logs -l job-name=init-job -n job-test
# Should show: Init container, Main container
```

### Exercise 5.2: Test Job with Resource Limits

```bash
# Create job with resource limits
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: resource-job
  namespace: job-test
spec:
  template:
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'echo "Resource job" && sleep 10']
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
      restartPolicy: Never
EOF

# Check job status
kubectl get job resource-job -n job-test
# COMPLETIONS: 1/1

# Check job pods
kubectl get pods -l job-name=resource-job -n job-test
# Should show completed pods
```

### Exercise 5.3: Test Job with Node Selector

```bash
# Create job with node selector
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: node-selector-job
  namespace: job-test
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'echo "Node selector job" && sleep 10']
      restartPolicy: Never
EOF

# Check job status
kubectl get job node-selector-job -n job-test
# COMPLETIONS: 1/1

# Check job pods
kubectl get pods -l job-name=node-selector-job -n job-test -o wide
# Should be scheduled on SSD node
```

---

## 📝 Lab Part 6: Documentation & Reflection (30 minutes)

### Exercise 6.1: Document Your Learnings

Create a file called `day-12-reflections.md` and answer:

1. **What do Jobs and CronJobs do?**
   - How do they manage batch processing?
   - What happens when they fail?
   - Why are they critical for scheduled tasks?

2. **What happens when job controller fails?**
   - How do you detect job failures?
   - What happens to running jobs?
   - How do you recover from job failures?

3. **How do you use Jobs and CronJobs?**
   - What's the difference between Jobs and CronJobs?
   - How do you manage job failures?
   - What are the best practices?

4. **How do you debug job issues?**
   - What tools do you use?
   - How do you check job status?
   - How do you test jobs?

### Exercise 6.2: Real-World Connections

Research these incidents:
- **Kubernetes job failures**
- **CronJob best practices**
- **Job scheduling issues**
- **Batch processing failures**

### Exercise 6.3: Advanced Job Scenarios

```bash
# Create complex job scenario
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: complex-job
  namespace: job-test
spec:
  parallelism: 2
  completions: 4
  backoffLimit: 3
  activeDeadlineSeconds: 300
  template:
    spec:
      initContainers:
      - name: init
        image: busybox
        command: ['sh', '-c', 'echo "Init" > /shared/init.txt']
        volumeMounts:
        - name: shared
          mountPath: /shared
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'cat /shared/init.txt && echo "Main" && sleep 10']
        volumeMounts:
        - name: shared
          mountPath: /shared
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
      volumes:
      - name: shared
        emptyDir: {}
      restartPolicy: Never
EOF

# Test the job
kubectl get job complex-job -n job-test
# COMPLETIONS: 4/4
```

---

## 🏆 Success Criteria

You've successfully completed Day 12 if you can:

✅ **Watch Jobs and CronJobs work in real-time**  
✅ **Break Jobs and CronJobs and understand the impact**  
✅ **Test job failures and retries**  
✅ **Practice job procedures**  
✅ **Understand the job architecture**  
✅ **Debug job issues**  

## 🚨 Common Issues & Solutions

### Issue: Jobs not running
```bash
# Check job status
kubectl get job <name>

# Check job events
kubectl describe job <name>

# Check controller manager logs
kubectl logs kube-controller-manager-minikube -n kube-system
```

### Issue: CronJobs not scheduling
```bash
# Check CronJob status
kubectl get cronjob <name>

# Check CronJob events
kubectl describe cronjob <name>

# Check CronJob schedule
kubectl get cronjob <name> -o yaml
```

### Issue: Jobs failing
```bash
# Check job pods
kubectl get pods -l job-name=<name>

# Check job pod logs
kubectl logs <pod-name>

# Check job pod events
kubectl describe pod <pod-name>
```

---

## 🎯 Tomorrow's Preview

**Day 13: Multi-cluster - The Federation**
- Watch multi-cluster components work
- Break multi-cluster and see clusters disconnect
- Test cluster federation failures
- Practice multi-cluster procedures
- Understand the multi-cluster architecture

**Get ready for more chaos! 🔥**

---

## 📚 Additional Resources

- [Kubernetes Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cronjob/)
- [Job Scheduling](https://kubernetes.io/docs/concepts/workloads/controllers/job/#job-patterns)
- [Batch Processing](https://kubernetes.io/docs/concepts/workloads/controllers/job/#batch-processing)

**Remember: Jobs and CronJobs are the task scheduler of Kubernetes - they run your batch processing! ⏰**
