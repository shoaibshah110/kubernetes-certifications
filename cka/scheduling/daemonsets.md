# DaemonSets

> Builds on: [replicaset.md](../core-concepts/replicaset.md) and [taints-and-tolerations.md](./taints-and-tolerations.md). You've already seen this object mentioned in [kube-proxy.md](../core-concepts/kube-proxy.md) — this is the full note on it.

## The problem this solves

A ReplicaSet answers "keep N copies of this Pod running, wherever makes sense." Sometimes what you actually want is different: **"run exactly one copy of this Pod on every node — no more, no less, and automatically on any node added later too."** That's a DaemonSet.

This is the right tool whenever something needs to exist **per-node**, rather than at some arbitrary total count. You've already met real examples: `kube-proxy` runs this way (every node needs its own networking rules), and so do things like log-collection agents, monitoring agents (e.g. `node-exporter`), and node-level storage daemons.

## How it's different from a ReplicaSet

| | ReplicaSet | DaemonSet |
|---|---|---|
| You specify | A `replicas` count | Nothing — count is implicit |
| Actual count | Whatever number you asked for | **Exactly one per eligible node**, always |
| New node joins cluster | Nothing happens automatically | A Pod is automatically created on it |
| Node removed | Nothing special | Its Pod is removed with it |

There's no `replicas` field on a DaemonSet at all — the "desired count" is simply defined as "one per matching node," so it scales itself as nodes come and go.

## Creating one

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
  labels:
    app: node-monitor
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      containers:
        - name: node-monitor
          image: node-monitor:latest
```

Notice this looks almost identical to a ReplicaSet YAML (same `selector`/`template.metadata.labels` matching rule from `replicaset.md` applies here too) — just `kind: DaemonSet` and no `replicas` field, since it wouldn't mean anything here.

There's no imperative `kubectl create daemonset` command (unlike `kubectl create deployment`) — you write/generate the YAML directly, or use `--dry-run=client -o yaml` on a Deployment and hand-edit `kind` and remove `replicas` as a shortcut to get the right skeleton.

## How it schedules Pods onto every node

Under the hood, a DaemonSet Pod is scheduled the same way any other Pod is — through the normal scheduler flow from `architecture.md` — but the DaemonSet controller automatically adds a **node affinity** rule targeting each specific node, effectively saying "this exact Pod must run on this exact node." So nothing magic or separate is happening — it's reusing the mechanisms you already know, just applied automatically on your behalf, one Pod-to-node pairing per node.

## Running on control plane nodes too

Recall from `taints-and-tolerations.md` that control plane nodes are tainted to keep ordinary Pods off them. But things like `kube-proxy` clearly *do* run there — every node, control plane included, needs networking rules. DaemonSets handle this by default: the DaemonSet controller automatically adds tolerations for the standard control-plane taints to its own Pods, so a DaemonSet's "one per node" promise genuinely includes control plane nodes unless you've configured otherwise. If you ever need to deliberately **exclude** a DaemonSet from certain nodes, you combine this with `nodeSelector`/node affinity on the DaemonSet's own template, same as any other Pod.

## Update strategy

Like Deployments, DaemonSets support rolling updates when the Pod template changes:

```yaml
spec:
  updateStrategy:
    type: RollingUpdate   # default — replaces Pods node by node automatically
    # type: OnDelete       # alternative — a Pod is only replaced once you manually delete it
```

`RollingUpdate` is the default and usually what you want. `OnDelete` is for cases where you want tighter manual control over exactly when each node's Pod gets replaced.

## Useful commands

```bash
kubectl get daemonsets -A              # see all DaemonSets across namespaces
kubectl describe daemonset <name>      # check desired/current/ready counts, and events
kubectl rollout status daemonset <name>
```

`kubectl describe daemonset` shows `Desired`, `Current`, `Ready`, and `Up-to-date` node counts — a quick way to confirm it's actually covering every node you expect.

## Key idea to remember

A DaemonSet isn't a different scheduling mechanism — it's the same Pod-creation-and-scheduling machinery you already know, with the controller automatically generating a per-node node affinity rule (and the control-plane taint tolerations) so that "one Pod per node" happens without you manually managing node counts yourself.

## Exam tips

- Know there's no `replicas` field, and why — the desired count is implicit ("one per matching node"), not something you declare.
- `kubectl get daemonset` — check `DESIRED` vs `CURRENT` vs `READY` columns when troubleshooting a DaemonSet that isn't covering every node (often caused by taints the DaemonSet's Pods don't tolerate, if you've customized its tolerations away from the defaults).
- Remember `kube-proxy` itself is deployed as a DaemonSet — a concrete, already-familiar example to reason from.
- If a task says "ensure this runs on every node, including control plane," that's your cue it's a DaemonSet question, not a Deployment/ReplicaSet one.
