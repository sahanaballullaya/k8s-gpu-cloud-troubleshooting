# Pending Pod — Diagnostic Runbook

**Scope:** Kubernetes pod stuck in Pending state due to insufficient node resources  
**Environment:** Minikube lab  
**Author:** Sahana  
**Last updated:** 2026-03-22

---

## What is a Pending Pod

A pod in Pending state means Kubernetes accepted the pod definition but the scheduler could not place it on any node. The container never starts — no image is pulled, no process runs.

The most common cause is resource requests that exceed what any node can provide. The scheduler looks at every node and if none can satisfy the CPU and memory request, the pod stays Pending indefinitely.

This is different from CrashLoopBackOff or OOMKilled where the container at least starts. Here the pod never leaves the scheduling queue.

---

## Before you touch anything

1. Do not delete the pod until you have collected all evidence
2. Check the Events section in describe pod first — the scheduler tells you exactly why it failed
3. Do not reduce resource requests blindly — understand what the application actually needs
4. Save evidence before making any changes

---

## Diagnostic steps

### Step 1 — Confirm the failure

```bash
kubectl get pods
```

Expected output:

```
NAME                          READY   STATUS    RESTARTS   AGE
pending-pod-6456d5cb8c-xxx    0/1     Pending   0          5m
```

Note — RESTARTS stays at 0 because the container never started. STATUS stays Pending indefinitely until you fix the resource request or free up node capacity.

---

### Step 2 — Read the scheduling failure reason

```bash
kubectl describe pod <pod-name>
```

Scroll to the Events section at the bottom:

```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  32s   default-scheduler  0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```

**FailedScheduling is your smoking gun.** The scheduler is telling you exactly what it checked and why it failed. Read the message carefully — it tells you how many nodes exist and what resource was insufficient.

Also check the resource requests in the same output:

```
Requests:
  cpu:     9999
  memory:  9999Gi
```

The request tells you what the pod asked for. If no node has that capacity it will never be scheduled.

---

### Step 3 — Check node resource allocation

```bash
kubectl describe node <node-name>
```

Scroll to the Allocated resources section:

```
Namespace   Name                               CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
---------   ----                               ------------  ----------  ---------------  -------------  ---
default     pending-pod-6456d5cb8c-ndrtm       1 (6%)        0 (0%)      64Mi (0%)        0 (0%)         8m
```

This shows exactly how much CPU and memory is already allocated on the node versus total capacity. Compare this against what the pod is requesting.

---

### Step 4 — Save evidence

```bash
kubectl describe pod <pod-name> > pending-describe.txt
kubectl describe node <node-name> > node-describe.txt
```

---

## Fix and verify

Fix the resource requests in the deployment YAML:

```bash
kubectl edit deployment pending-pod
```

Find and update:

```yaml
resources:
  requests:
    memory: "9999Gi"   # change this
    cpu: "9999"        # and this
```

To:

```yaml
resources:
  requests:
    memory: "64Mi"    # to this
    cpu: "1"          # and this
```

Save and exit. Kubernetes automatically creates a new pod with the updated requests.

Verify the fix:

```bash
kubectl get pods
```

Wait until STATUS shows Running. Then confirm the new requests:

```bash
kubectl describe pod <new-pod-name>
```

Look for:

```
Requests:
  cpu:     1
  memory:  64Mi
```

---

## Key difference from other failures

| | Pending Pod | CrashLoopBackOff | OOMKilled |
|---|---|---|---|
| Container starts | Never | Yes, keeps restarting | Yes, then killed |
| RESTARTS count | 0 | Climbs fast | Climbs |
| Where to look | Events section | Logs --previous | Reason field |
| Root cause | Scheduler can't place pod | App crashes on start | Memory limit exceeded |
| Fix | Reduce resource requests | Fix app config or code | Increase memory limit |

---

## Screenshots

### 01 — Pending pod YAML

![Pending pod YAML](screenshots/01-pendingpod-yaml.jpg)

Deployment YAML with resource requests set to 9999 CPU and 9999Gi memory — values no node can satisfy.

---

### 02 — Pod stuck in Pending

![Get pod status pending](screenshots/02-get-pod-status-pending.jpg)

`kubectl get pods` showing STATUS: Pending and RESTARTS: 0. The container never started.

---

### 03 — Describe pod events

![Pod describe event](screenshots/03-pod-describe-event.jpg)

`kubectl describe pod` showing FailedScheduling in the Events section. The scheduler message tells you exactly which resources were insufficient.

---

### 04 — Updated YAML file

![Updated YAML file](screenshots/04-updated_yaml_file.jpg)

Deployment YAML after fix — memory reduced to 64Mi and CPU reduced to 1.

---

### 05 — Pod running after fix

![Pod running](screenshots/05-pod-running.jpg)

`kubectl get pods` after applying the fix showing STATUS: Running.

---

### 06 — Describe pod after fix

![Pod describe success](screenshots/06-pod-describe-success.jpg)

`kubectl describe pod` after fix confirming updated resource requests are accepted and pod is healthy.

---

### 07 — Node describe

![Node describe](screenshots/07-node-describe.jpg)

`kubectl describe node` showing Allocated resources section — confirms node capacity and current allocation after fix.

---

## Lab reproduction

```bash
# Start cluster
minikube start

# Deploy pod with resource requests too high
kubectl apply -f pending-pod.yaml

# Observe failure
kubectl get pods

# Diagnose
kubectl describe pod <pod-name>
kubectl describe node <node-name>

# Save evidence
kubectl describe pod <pod-name> > pending-describe.txt
kubectl describe node <node-name> > node-describe.txt

# Fix — reduce resource requests
kubectl edit deployment pending-pod
# Change 9999 CPU and 9999Gi memory to 1 CPU and 64Mi, save and exit

# Verify
kubectl get pods
kubectl describe pod <new-pod-name>
```

---
