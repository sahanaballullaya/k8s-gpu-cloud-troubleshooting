# CrashLoopBackOff — Diagnostic Runbook

**Scope:** Kubernetes pod stuck in CrashLoopBackOff  
**Environment:** Minikube lab  
**Author:** Sahana  
**Last updated:** 2026-03-18

---

## What is CrashLoopBackOff

A pod enters CrashLoopBackOff when its container exits repeatedly and Kubernetes keeps restarting it. After each crash Kubernetes doubles the wait before retrying — 10s, 20s, 40s, 80s, up to 5 minutes. The pod is not running. The workload is stopped.

---

## Before you touch anything

1. Do not delete or restart the pod until you have collected all evidence
2. Read the exit code before reading logs — it tells you the category of failure
3. Save evidence to files before making any changes — logs disappear when pods are deleted

---

## Diagnostic steps

### Step 1 — Confirm the failure

```bash
kubectl get pods -w
```

Watch the STATUS column. You will see it cycle:

```
NAME      READY   STATUS             RESTARTS   AGE
crasher   0/1     Error              1          8s
crasher   0/1     CrashLoopBackOff   2          18s
crasher   0/1     CrashLoopBackOff   3          45s
```

RESTARTS climbing tells you how long it has been failing.

---

### Step 2 — Read the exit code and events

```bash
kubectl describe pod <pod-name>
```

Scroll up to find Last State:

```
Last State:   Terminated
  Reason:     Error
  Exit Code:  1
```

Exit code tells you the category before you read a single log line:

| Exit Code | Meaning |
|---|---|
| 137 | Kernel killed it — memory problem |
| 1 | Application crashed — check logs |
| 126 | Command not executable — image problem |
| 127 | Command not found — wrong image |

Scroll to the bottom for Events:

```
Warning  BackOff  kubelet  Back-off restarting failed container
```

---

### Step 3 — Read logs from the crashed container

```bash
kubectl logs <pod-name> --previous
```

The `--previous` flag is critical. Without it you see logs from the current container which just restarted and is empty. `--previous` retrieves logs from the last crashed instance — that is where the error lives.

Read the last line first. That is where the process died.

---

### Step 4 — Check the node

```bash
kubectl describe node
```

Look at Conditions:

```
MemoryPressure   False
DiskPressure     False
Ready            True
```

Ready=True means the node is healthy and the problem is in the pod, not the machine.

---

### Step 5 — Save your evidence

Do this before making any changes:

```bash
kubectl logs <pod-name> --previous > crasher-previous.log
kubectl describe pod <pod-name> > crasher-describe.txt
kubectl describe node > node-describe.txt
```

---

## Fix and verify

Only act after you understand the root cause.

```bash
kubectl delete pod <pod-name>
```

Deploy the corrected version. Then watch:

```bash
kubectl get pods -w
```

Wait until STATUS shows Running and RESTARTS stays at 0 for at least 3 minutes.

Verify the application is actually working:

```bash
kubectl logs <pod-name> -f
```

---

## Screenshots

### 01 — CrashLoopBackOff forming

![CrashLoopBackOff watch](screenshots/01-crashloopbackoff-watch.jpg)

`kubectl get pods -w` showing STATUS cycling and RESTARTS climbing.

---

### 02 — Describe pod output

![Describe pod exit code and events](screenshots/02-describe-pod-exitcode-events.jpg)

`kubectl describe pod` showing Exit Code 1 and the BackOff warning in Events.

---

### 03 — Logs from crashed container

![Logs previous flag](screenshots/03-logs-previous-flag.jpg)

`kubectl logs --previous` retrieving output from the dead container. The `--previous` flag is visible in the command.

---

### 04 — Healthy pod running

![Healthy pod running](screenshots/04-healthy-pod-running.jpg)

`kubectl get pods` after fix showing STATUS: Running and RESTARTS: 0.

---

## Lab reproduction

```bash
# Start cluster
minikube start

# Deploy crashing pod
kubectl run crasher \
  --image=busybox \
  --restart=Always \
  -- /bin/sh -c "echo 'starting'; sleep 2; exit 1"

# Observe
kubectl get pods -w

# Diagnose
kubectl describe pod crasher
kubectl logs crasher --previous

# Save evidence
kubectl logs crasher --previous > crasher-previous.log
kubectl describe pod crasher > crasher-describe.txt
kubectl describe node > node-describe.txt

# Fix
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

- [ImagePullBackOff](../imagepullbackoff/imagepullbackoff-runbook.md) — container image cannot be fetched
- [OOMKilled](../oomkilled/oomkilled-runbook.md) — exit code 137, memory limit exceeded
