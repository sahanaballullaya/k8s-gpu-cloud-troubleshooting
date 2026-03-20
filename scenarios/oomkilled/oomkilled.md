# OOMKilled — Diagnostic Runbook

**Scope:** Kubernetes pod killed due to memory limit exceeded  
**Environment:** Minikube lab  
**Author:** Sahana  
**Last updated:** 2026-03-20

---

## What is OOMKilled

OOM stands for Out Of Memory. When a container exceeds its memory limit the Linux kernel kills the process immediately. Kubernetes records this as Reason: OOMKilled. The container never gets a chance to log anything — it is killed mid-execution.

This is different from CrashLoopBackOff where the application crashes itself. Here the kernel forcibly terminates the process from outside.

---

## Before you touch anything

1. Do not delete or restart the pod until you have collected all evidence
2. Check Reason field first — OOMKilled is definitive before you even look at exit code
3. Do not raise memory limits without understanding why the container exceeded them
4. Save evidence before making any changes

---

## Diagnostic steps

### Step 1 — Confirm the failure

```bash
kubectl get pods -w
```

Expected output:

```
NAME                          READY   STATUS      RESTARTS   AGE
oom-killer-695c8cd756-52mbd   0/1     OOMKilled   1          5s
oom-killer-695c8cd756-52mbd   0/1     OOMKilled   2          12s
oom-killer-695c8cd756-52mbd   0/1     CrashLoopBackOff  3   25s
```

Note — OOMKilled appears first, then transitions to CrashLoopBackOff after backoff kicks in. Both indicate the same root cause.

---

### Step 2 — Read the exit code and reason

```bash
kubectl describe pod <pod-name>
```

Scroll to Last State:

```
Last State:   Terminated
  Reason:     OOMKilled
  Exit Code:  137
```

**Reason: OOMKilled is your smoking gun.** This is Kubernetes telling you exactly what happened. In some environments like Minikube the exit code may show 1 instead of 137 — always trust the Reason field over the exit code.

Also check memory limits in the same output:

```
Limits:
  memory:  50Mi
Requests:
  memory:  50Mi
```

The limit tells you the ceiling the container was given. If the application needed more than this it gets killed.

---

### Step 3 — Read logs from crashed container

```bash
kubectl logs <pod-name> --previous
```

Unlike CrashLoopBackOff where logs show the application output, OOMKilled logs show the kernel kill signal:

```
stress: FAIL: [1] (415) <-- worker 7 got signal 9
stress: WARN: [1] (417) now reaping child worker processes
stress: FAIL: [1] (421) kill error: No such process
stress: FAIL: [1] (451) failed run completed in 0s
```

Signal 9 is SIGKILL — the kernel forcibly terminated the process. This confirms OOMKilled from the application side.

Logs may sometimes be empty — the process was killed so fast nothing was written. Empty logs on an OOMKilled pod is normal. Your evidence is in the Reason field, not the logs.

---

### Step 4 — Check node memory

```bash
kubectl describe node
```

Scroll to the allocated resources table:

```
Resource    Requests     Limits
memory      220Mi (3%)   220Mi (3%)
```

This shows total memory allocated across all pods on the node. If memory limits are close to node capacity multiple pods may be competing for memory.

---

### Step 5 — Save evidence

```bash
kubectl logs <pod-name> --previous > oom-previous.log
kubectl describe pod <pod-name> > oom-describe.txt
kubectl describe node > node-describe.txt
```

---

## Fix and verify

Do not delete the pod manually. Fix the memory limit at the Deployment level:

```bash
kubectl edit deployment oom-killer
```

Find and update:

```yaml
resources:
  limits:
    memory: "50Mi"    # change this
```

To:

```yaml
resources:
  limits:
    memory: "200Mi"   # to this
```

Save and exit. Kubernetes automatically rolls out a new pod with the updated limit.

Verify the fix:

```bash
kubectl get pods -w
```

Wait until STATUS shows Running and RESTARTS stays at 0. Then confirm the new limit:

```bash
kubectl describe pod <new-pod-name>
```

Look for:

```
Limits:
  memory: 200Mi
```

---

## Key difference from CrashLoopBackOff

| | CrashLoopBackOff | OOMKilled |
|---|---|---|
| Who kills it | Application exits itself | Linux kernel kills it |
| Exit code | 1, 126, 127 | 137 (or 1 in Minikube) |
| Reason field | Error | OOMKilled |
| Logs available | Yes — use --previous | Sometimes empty |
| Fix | Fix app config or code | Increase memory limit |
| Evidence | Exit code + logs | Reason field + signal 9 in logs |

---

## Screenshots

### 01 — OOMKilled forming

![OOMKilled watch](screenshots/01-oomkilled-watch.jpg)

`kubectl get pods -w` showing STATUS cycling through OOMKilled and into CrashLoopBackOff with RESTARTS climbing.

---

### 02 — Describe pod output

![Describe pod OOMKilled](screenshots/02-describe-pod-oomkilled.jpg)

`kubectl describe pod` showing Reason: OOMKilled, Exit Code, and memory limits. The Reason field is the definitive evidence.

---

### 03 — Logs from crashed container

![Logs previous OOM](screenshots/03-logs-previous-oom.jpg)

`kubectl logs --previous` showing signal 9 in the stress output. Signal 9 is SIGKILL — the kernel terminated the process.

---

### 04 — Node memory

![Describe node memory](screenshots/04-describe-node-memory.jpg)

`kubectl describe node` showing allocated memory across all pods on the node.

---

### 05 — Healthy pod running after fix

![Healthy memory running](screenshots/05-healthy-memory-running.jpg)

`kubectl get pods -w` after increasing memory limit showing STATUS: Running and RESTARTS: 0.

---

## Lab reproduction

```bash
# Start cluster
minikube start

# Deploy pod with memory limit too low
kubectl apply -f oom-deployment.yaml

# Observe failure
kubectl get pods -w

# Diagnose
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous

# Save evidence
kubectl logs <pod-name> --previous > oom-previous.log
kubectl describe pod <pod-name> > oom-describe.txt
kubectl describe node > node-describe.txt

# Fix — increase memory limit
kubectl edit deployment oom-killer
# Change 50Mi to 200Mi, save and exit

# Verify
kubectl get pods -w
kubectl describe pod <new-pod-name>
```

---
