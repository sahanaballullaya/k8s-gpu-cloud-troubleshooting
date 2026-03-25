# Node NotReady — Diagnostic Runbook

**Scope:** Kubernetes node in NotReady state
**Environment:** Minikube lab
**Author:** Sahana
**Last updated:** 2026-03-25

---

## What is Node NotReady

A node enters NotReady when the Kubernetes control plane stops receiving health status and heartbeats from the node.

The kubelet is responsible for reporting node health. If kubelet stops, crashes, or cannot communicate, the control plane marks the node as NotReady.

When a node is NotReady:

* Pods may become unreachable
* New pods may not schedule
* Existing workloads may be affected
* Cluster stability is impacted

---

## Before you touch anything

1. Do not restart Minikube immediately.
2. Confirm whether the issue is node-level or pod-level.
3. Collect node state and events before making changes.
4. Save evidence before fixing anything.

---

## Diagnostic steps

### Step 1 — Confirm the failure

```bash
kubectl get nodes
```

Optional, to watch the transition live:

```bash
kubectl get nodes -w
```

Example output:

```text
NAME       STATUS     ROLES           AGE   VERSION
minikube   NotReady   control-plane   5m    v1.xx.x
```

What this tells you:

* The issue is at node level, not just one pod.
* Kubernetes no longer considers the node healthy.

---

### Step 2 — Describe the node

```bash
kubectl describe node minikube
```

Focus on these sections.

**Conditions**

Typical example:

```text
Conditions:
  Type             Status
  Ready            Unknown
```

or:

```text
Conditions:
  Type             Status
  Ready            False
```

What this means:

* `Unknown` usually means the control plane is not receiving updates from kubelet.
* `False` means the node is reachable enough to report unhealthy status, but it is not ready to serve workloads.

**Events**

Look near the bottom for messages such as:

```text
Warning  NodeNotReady  kubelet  Node is not ready
```

or messages indicating kubelet stopped posting node status.

This step confirms whether the issue is heartbeat loss, node health degradation, or a scheduling problem.

---

### Step 3 — Check cluster-wide pod impact

```bash
kubectl get pods -A
```

Why this matters:

* It shows whether the problem is affecting only the node status or also impacting workloads.
* In Minikube, control plane and workload components share the same node, so node health issues can affect everything.

What to look for:

* Pods stuck in `Pending`
* Pods not becoming `Running`
* System pods showing instability or restarts
* Application pods becoming unavailable

This helps separate cause from impact.

---

### Step 4 — Validate kubelet status inside the node

SSH into the Minikube node:

```bash
minikube ssh
```

Check kubelet status:

```bash
sudo systemctl status kubelet
```

If `systemctl` is not available in your Minikube environment, use:

```bash
sudo service kubelet status
```

What to check:

* `active (running)` means kubelet is up
* `inactive`, `failed`, or `stopped` indicates the likely root cause

Exit when done:

```bash
exit
```

This is the most direct way to confirm whether kubelet is the issue.

---

### Step 5 — Check node conditions for pressure or resource issues

Run again:

```bash
kubectl describe node minikube
```

Look for these conditions:

```text
MemoryPressure
DiskPressure
PIDPressure
Ready
```

Interpretation:

* If `MemoryPressure=True`, the node may be under memory stress.
* If `DiskPressure=True`, disk space may be exhausted.
* If `PIDPressure=True`, the node may be running out of process IDs.
* If those are `False` but `Ready=False` or `Unknown`, kubelet communication is a stronger suspect.

This step prevents jumping to the wrong fix.

---

### Step 6 — Save your evidence

Before making any changes, save the outputs:

```bash
kubectl describe node minikube > node-describe.txt
kubectl get nodes -o wide > node-status.txt
kubectl get pods -A > pods-all.txt
```

Why this matters:

* Describe output can change after recovery.
* Saved evidence is useful for root cause analysis and documentation.
* It gives you artifacts for your GitHub project.

---

## Fix and verify

### Step 7 — Restart kubelet

SSH into the node:

```bash
minikube ssh
```

Start kubelet:

```bash
sudo systemctl start kubelet
```

If needed:

```bash
sudo service kubelet start
```

Exit:

```bash
exit
```

This restores heartbeat communication if kubelet being stopped was the cause.

---

### Step 8 — Verify node recovery

```bash
kubectl get nodes
```

Optional:

```bash
kubectl get nodes -w
```

Expected output:

```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   8m    v1.xx.x
```

What this confirms:

* kubelet is communicating again
* Kubernetes now considers the node healthy

---

### Step 9 — Validate cluster stability

```bash
kubectl get pods -A
```

Check that:

* system pods are stable
* application pods are not failing
* no new scheduling issues appear

This confirms the recovery is complete, not partial.

---

## Screenshots

### 01 — Cluster ready

![Cluster ready](screenshots/01-cluster-ready.jpg)

`kubectl get nodes` showing the node in Ready state.

---

### 02 — System pods running

![System pods](screenshots/02-system-pods-running.jpg)

`kubectl get pods -A` showing all core components in Running state.

---

### 03 — SSH into Minikube

![Minikube SSH](screenshots/03-minikube-ssh.jpg)

Accessing the node using `minikube ssh`.

---

### 04 — Stop kubelet

![Stop kubelet](screenshots/04-stop-kubelet.jpg)

Stopping kubelet to simulate node failure.

---

### 05 — Node NotReady

![Node NotReady](screenshots/05-node-notready.jpg)

`kubectl get nodes` showing STATUS as NotReady.

---

### 06 — Describe node

![Describe node](screenshots/06-describe-node-notready.jpg)

`kubectl describe node` showing Ready=False/Unknown and events.

---

### 07 — System pods impact

![Pods impact](screenshots/07-system-pods-impact.jpg)

`kubectl get pods -A` showing cluster impact after node failure.

---

### 08 — Start kubelet

![Start kubelet](screenshots/08-start-kubelet.jpg)

Restarting kubelet inside Minikube node.

---

### 09 — Node recovered

![Node recovered](screenshots/09-node-recovered.jpg)

`kubectl get nodes` showing node back to Ready state.

---

## Lab reproduction

```bash
# Start cluster
minikube start

# Confirm healthy node
kubectl get nodes
kubectl get pods -A

# SSH into Minikube node
minikube ssh

# Stop kubelet to simulate failure
sudo systemctl stop kubelet

# Exit node shell
exit

# Observe node becoming NotReady
kubectl get nodes -w

# Diagnose
kubectl describe node minikube
kubectl get pods -A

# Save evidence
kubectl describe node minikube > node-describe.txt
kubectl get nodes -o wide > node-status.txt
kubectl get pods -A > pods-all.txt

# Recover
minikube ssh
sudo systemctl start kubelet
exit

# Verify recovery
kubectl get nodes -w
kubectl get pods -A
```

---

## Root cause

The node entered NotReady because kubelet was stopped, which caused the control plane to stop receiving node status and heartbeats.

Without kubelet, Kubernetes cannot reliably manage or assess the node.

---

## Fix summary

| Issue                | Fix                          |
| -------------------- | ---------------------------- |
| kubelet stopped      | Restart kubelet              |
| lost heartbeats      | Restore node communication   |
| node marked NotReady | Verify node returns to Ready |

---

## Key learning

* Node readiness depends heavily on kubelet health.
* `kubectl describe node` is the main diagnostic command.
* `kubectl get pods -A` helps measure cluster-wide impact.
* Evidence should be collected before recovery steps.

---

## Interview summary

A Kubernetes node becomes NotReady when the control plane stops receiving valid heartbeats or health updates from kubelet. I would first confirm the issue using `kubectl get nodes`, then diagnose using `kubectl describe node`, check pod impact across namespaces, validate kubelet status on the node, save evidence, and recover by restarting kubelet if that is the root cause.
