# kube-scheduler

> Builds on: [cluster architecture](./architecture.md) and [kube-apiserver](./kube-apiserver.md). Read those first if you haven't.

## What it actually is

The **kube-scheduler** has exactly one job: when a new pod is created with no node assigned, decide **which worker node it should run on**. That's it — it doesn't run the pod, doesn't start containers, doesn't talk to kubelet directly. It just makes the decision and writes that decision back through the API server.

It's a **watch loop**: the scheduler continuously watches the API server for pods where `.spec.nodeName` is empty, picks a node for each one, then updates that pod object (via the API server) with the chosen node name. From there, the kubelet on that node takes over.

## How it picks a node

The decision happens in two phases:

### 1. Filtering (which nodes are even eligible?)

Nodes that can't run the pod are eliminated first. Reasons include:

- Not enough CPU/memory available on the node for what the pod requests
- Node has a **taint** the pod doesn't **tolerate** (see below)
- Pod has a `nodeSelector` / node affinity rule the node doesn't match
- Port conflicts (pod needs a hostPort already in use on that node)

### 2. Scoring (of the eligible nodes, which is best?)

Remaining nodes are ranked by a set of scoring functions — e.g. preferring the node with the most free resources after placement, spreading pods across nodes for availability, honoring pod affinity/anti-affinity preferences. The highest-scoring node wins.

## Taints, tolerations, and affinity (brief preview)

You'll get dedicated notes on these later, but the scheduler is where they all take effect:

- **Taints** (on a node) repel pods, unless the pod has a matching **toleration**. Example: control plane nodes are tainted so ordinary pods don't get scheduled onto them.
- **Node affinity** / **nodeSelector** (on a pod) attracts a pod toward nodes with specific labels.
- **Pod affinity/anti-affinity** attracts or repels pods based on what *other pods* are already running on a node.

## What if no node fits?

The pod stays in `Pending` state. This is a very common real-world and exam troubleshooting scenario — `kubectl describe pod` will show scheduling failure events explaining exactly why (e.g. "0/3 nodes are available: 3 Insufficient cpu").

## Bypassing the scheduler entirely

You can manually set `.spec.nodeName` directly in a pod's manifest at creation time — this skips the scheduler completely, since the scheduler only acts on pods with an empty `nodeName`. This is a known trick for both understanding how scheduling works and for quick exam troubleshooting (e.g. temporarily forcing a pod onto a specific node).

## Where it runs

Like the API server, it's a **static pod** on the control plane node(s) in a `kubeadm` cluster:

```
/etc/kubernetes/manifests/kube-scheduler.yaml
```

```bash
kubectl get pods -n kube-system | grep scheduler
```

## Key idea to remember

The scheduler is a **decision-maker, not a doer**. It only ever writes "this pod belongs on this node" through the API server — the actual work of starting the container is entirely the kubelet's job on that chosen node. If the scheduler is down, existing pods keep running fine; *new* pods just get stuck in `Pending` forever.

## Exam tips

- If pods are stuck in `Pending`, check scheduler health first (`kubectl get pods -n kube-system`, `kubectl describe pod <pending-pod>` for the reason) — this is one of the most common troubleshooting scenarios.
- Know that you can inspect *why* a scheduling decision failed via the Events section of `kubectl describe pod`.
- Remember: manually setting `nodeName` in a pod spec bypasses the scheduler entirely — useful to know for both exam shortcuts and understanding failure scenarios.
- Static pod manifest path: `/etc/kubernetes/manifests/kube-scheduler.yaml`.
