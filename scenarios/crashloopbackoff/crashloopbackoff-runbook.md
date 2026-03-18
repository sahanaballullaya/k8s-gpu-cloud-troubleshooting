# CrashLoopBackOff — Diagnostic Runbook

**Scope:** Kubernetes pod stuck in CrashLoopBackOff  
**Environment:** GPU cloud / HPC (Crusoe CKE, bare metal K8s)  
**Author:** Sahana  
**Last updated:** 2026-03-18

---

## What is CrashLoopBackOff

A pod enters CrashLoopBackOff when its container exits repeatedly and Kubernetes runs out of patience. The container starts, crashes, Kubernetes restarts it, it crashes again. After each cycle Kubernetes doubles the wait before retrying — 10s, 20s, 40s, 80s, up to 5 minutes. The pod is not running. The customer's workload is stopped.

At Crusoe this means a training job, inference server, or batch workload is producing nothing while the customer waits. Treat every CrashLoopBackOff as urgent until proven otherwise.

---

## Before you touch anything

1. Do not delete or restart the pod until you have collected all evidence
2. Do not assume the cause — read the exit code first
3. Do not contact the customer until you have a hypothesis
4. Save all evidence to files before making any changes — logs disappear when pods are deleted

---

## Diagnostic steps

### Step 1 — Establish scope

```bash
kubectl get pods
```

Read three fields:

| Field | What it tells you |
|---|---|
| STATUS | Current state — CrashLoopBackOff confirms the loop |
| RESTARTS | How many times it has crashed — indicates how long failing |
| AGE | How long the pod has existed |

Expected output:
```
NAME            READY   STATUS             RESTARTS   AGE
training-job    0/1     CrashLoopBackOff   17         43m
```

RESTARTS: 17 over 43 minutes means it has been silently failing since before the customer noticed. Note this for your RCA timeline.

---

### Step 2 — Identify the failure layer

```bash
kubectl describe pod <pod-name>
```

Read in this order:

**Exit Code** — found in the Containers section, under Last State

```
Last State:   Terminated
  Reason:     Error
  Exit Code:  1
```

Exit code tells you the category of failure before you read a single log line:

| Exit Code | Meaning | Where to look next |
|---|---|---|
| 137 | OOMKilled — kernel killed the process | Memory limits, node memory pressure |
| 1 | Application crashed | Application logs, config, env vars |
| 126 | Command not executable | Image entrypoint, file permissions |
| 127 | Command not found | Wrong image, wrong entrypoint |
| 143 | SIGTERM — graceful shutdown | Usually intentional, check restart policy |
| 0 | Exited cleanly | Should not be restarting — check restart policy |

**Events** — found at the bottom of describe output

```
Warning  BackOff             kubelet  Back-off restarting failed container
Warning  OOMKilling          kubelet  Memory cgroup out of memory
Warning  FailedMount         kubelet  Unable to attach or mount volumes
         Insufficient        kubelet  0/3 nodes available: insufficient nvidia.com/gpu
```

Events tell you if Kubernetes itself had a problem separate from the application.

---

### Step 3 — Read the crash evidence

```bash
kubectl logs <pod-name> --previous
```

The `--previous` flag retrieves logs from the last crashed container instance. Without this flag you see the current container's logs, which are empty immediately after a restart.

Read the last line first. That is where the process died.

Common patterns at Crusoe:

| Log message | Root cause | Escalation |
|---|---|---|
| `CUDA error: invalid device ordinal` | GPU not visible to container | SRE + hardware team |
| `NCCL error: unhandled system error` | GPU-to-GPU communication failed | Networking team (InfiniBand) |
| `No such file or directory: /data/` | Volume not mounted | Check PVC, escalate storage team |
| `Connection refused 10.x.x.x:PORT` | Dependency not reachable on startup | Check if service is running |
| `environment variable X not set` | Missing config in pod spec | Customer config error, fix directly |
| `torch.distributed.DistBackendError` | NCCL / distributed init failed | Networking team (RDMA/InfiniBand) |
| `Permission denied` | Wrong user or volume permissions | Check securityContext and PVC |

Save logs before proceeding:

```bash
kubectl logs <pod-name> --previous > <pod-name>-previous.log
kubectl describe pod <pod-name> > <pod-name>-describe.txt
```

---

### Step 4 — Check the node

Find which node the pod is running on:

```bash
kubectl get pod <pod-name> -o wide
```

Then inspect that node:

```bash
kubectl describe node <node-name>
```

**Conditions section** — node health:

```
MemoryPressure   False    healthy
DiskPressure     False    healthy
PIDPressure      False    healthy
Ready            True     node is schedulable
```

Any condition other than Ready=True is a node-level problem affecting all pods on that host.

**Capacity vs Allocatable** — GPU availability:

```
Capacity:
  nvidia.com/gpu:   8
Allocatable:
  nvidia.com/gpu:   7
```

Capacity minus Allocatable shows hardware faults. One missing GPU means one H100 has failed on that node. This is your escalation to SRE with the node name and GPU count discrepancy as evidence.

**Allocated resources** — node saturation:

```
nvidia.com/gpu    8/8
```

If GPUs are fully allocated, a pending pod is waiting for capacity — not a pod-level problem.

Save node state:

```bash
kubectl describe node <node-name> > node-describe.txt
```

---

### Step 5 — Check storage if logs pointed there

```bash
kubectl get pvc
kubectl describe pvc <pvc-name>
```

| PVC Status | Meaning | Action |
|---|---|---|
| Bound | Volume exists and attached | Problem is in the mount config |
| Pending | Volume never provisioned | Escalate storage team |
| Lost | Underlying storage disappeared | Sev1 — escalate storage team immediately |

At Crusoe storage is WEKA or Lustre. If multiple customers report mount failures simultaneously, suspect a storage cluster issue rather than individual pod config errors.

---

## Escalation map

| Evidence | Escalation | What to include |
|---|---|---|
| Exit 137, OOMKilled | SRE | Pod name, node name, memory limit vs usage |
| CUDA error, GPU count mismatch on node | SRE + hardware team | Node name, capacity vs allocatable output |
| NCCL error, InfiniBand port down | Networking team | Node name, ibstat output, error message |
| PVC Pending or Lost | Storage team | PVC name, describe output, affected customers |
| Node NotReady | SRE immediately | Node name, all pods affected on that host |
| Missing env var, wrong image | Customer | Config change needed, guide them directly |

---

## Blast radius check

Before restarting any pod, answer these questions:

- Is a training run mid-progress? Deleting the pod may lose checkpoint state
- How many GPUs does this pod hold? On an H100 host that is up to 8 GPUs going dark
- Are other pods on the same node? A node restart affects all of them
- Is the customer watching live? Communicate before acting
- What time is it for the customer?

---

## Fix and verify

Only act after root cause is confirmed and customer has acknowledged.

```bash
# Fix the underlying cause first (config, resource limit, volume)
# Then restart

kubectl delete pod <pod-name>
kubectl get pods -w
```

Watch RESTARTS stay at 0 for at least 3 minutes. Do not tell the customer it is resolved until you have confirmed stability.

Verify the application is actually working, not just the pod status:

```bash
kubectl logs <pod-name> -f
```

Confirm the application output shows expected behavior — model loading, dataset reading, NCCL initialization.

---

## Customer communication

**On detection:**
```
I can see your pod [name] has been in CrashLoopBackOff 
for [X] minutes with [N] restarts. I am investigating 
and will update you in 15 minutes.
```

**On root cause identified:**
```
Root cause identified — [specific error from logs]. 
This requires [change]. Before I proceed, can you 
confirm [checkpoint status / readiness to restart]?
```

**On resolution:**
```
Your pod has been running stably for 5 minutes. 
Logs confirm [expected behavior]. Let me know if 
you see any further issues.
```

---

## RCA template

```
Incident: CrashLoopBackOff on pod [name]
Customer: [name]
Severity: [P1/P2/P3]

Timeline:
  [time]  Customer reported workload stopped
  [time]  CrashLoopBackOff confirmed, RESTARTS: N
  [time]  Root cause identified: [exit code + log message]
  [time]  Fix applied: [what changed]
  [time]  Pod stable, customer confirmed

Root cause:
  [One sentence — specific error message and what caused it]

Blast radius:
  [How many GPUs affected, how long, whether checkpoint was lost]

Fix:
  [Exactly what was changed]

Prevention:
  [What runbook, validation, or alert would catch this earlier]
```

---

## Screenshots

Visual evidence from the lab reproduction below. Each screenshot corresponds to a diagnostic step.

### 01 — CrashLoopBackOff forming
`kubectl get pods -w` showing STATUS cycling from Error to CrashLoopBackOff with RESTARTS climbing.

![CrashLoopBackOff watch](screenshots/01-crashloopbackoff-watch.jpg)

What to notice: RESTARTS incrementing and the backoff timer increasing between each cycle. This is the first thing you check to establish how long the pod has been failing.

---

### 02 — Describe pod output
`kubectl describe pod crasher` showing Exit Code and Events section.

![Describe pod exit code and events](screenshots/02-describe-pod-exitcode-events.jpg)

What to notice: Exit Code 1 confirms application crash. The BackOff warning in Events with the repeat count (x17, x23 etc) shows how many cycles have occurred. These two pieces of evidence together tell you the category of failure before reading logs.

---

### 03 — Logs from crashed container
`kubectl logs crasher --previous` retrieving output from the dead container instance.

![Logs previous flag](screenshots/03-logs-previous-flag.jpg)

What to notice: The `--previous` flag is visible in the command. Without it you see empty logs from the freshly restarted container. This flag is the difference between finding evidence and finding nothing.

---

### 04 — Healthy pod running
`kubectl get pods` after fix showing STATUS: Running and RESTARTS: 0.

![Healthy pod running](screenshots/04-healthy-pod-running.jpg)

What to notice: RESTARTS stays at 0. This is your verification step — you watch this for 3+ minutes before telling the customer the issue is resolved.

---

## Lab reproduction

To reproduce this scenario in Minikube for training purposes:

```bash
# Start cluster
minikube start

# Deploy crashing pod
kubectl run crasher \
  --image=busybox \
  --restart=Always \
  -- /bin/sh -c "echo 'starting'; sleep 2; exit 1"

# Observe the loop
kubectl get pods -w

# Diagnose
kubectl describe pod crasher
kubectl logs crasher --previous

# Save evidence
kubectl logs crasher --previous > crasher-previous.log
kubectl describe pod crasher > crasher-describe.txt
kubectl describe node > node-describe.txt

# Fix — deploy stable pod
kubectl delete pod crasher
kubectl run healthy \
  --image=busybox \
  --restart=Always \
  -- /bin/sh -c "while true; do echo ok; sleep 5; done"

# Verify
kubectl get pods -w
kubectl logs healthy -f
```

---

## Related scenarios

- [OOMKilled](./oomkilled-runbook.md) — exit code 137, memory limit exceeded
- [ImagePullBackOff](./imagepullbackoff-runbook.md) — container image cannot be fetched
- [Pending pod](./pending-pod-runbook.md) — pod never scheduled, insufficient resources
- [Volume mount failure](./volume-mount-runbook.md) — PVC not attaching
