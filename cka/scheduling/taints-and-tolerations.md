# Taints and Tolerations

> Builds on: [kube-scheduler.md](../core-concepts/kube-scheduler.md) — this is one of the filtering mechanisms briefly previewed there. Also builds on [labels-and-selectors.md](../core-concepts/labels-and-selectors.md) conceptually, though taints/tolerations are a separate mechanism, not the same thing as labels/selectors — don't mix them up (more on that below).

## The problem this solves

By default, the scheduler will place a Pod on **any** node that has room for it. Sometimes you want the opposite: certain nodes should **repel** Pods by default, and only let specific Pods land there. Example: control plane nodes shouldn't run your regular application Pods — only system components should be allowed there.

Taints and tolerations are how you achieve this.

## The core idea, in plain terms

- A **taint** goes on a **node**. It says: "don't schedule anything here, unless it specifically tolerates me."
- A **toleration** goes on a **Pod**. It says: "I'm allowed to ignore that particular taint."

Think of it like a "staff only" sign on a door (the taint). A toleration is like a staff badge — it lets you through *that specific* door, but it doesn't pull you toward it. It doesn't make the door more attractive or likely to be chosen; it just removes the "don't enter" restriction for you specifically.

**This is the single most important thing to understand, and the most common misunderstanding:** a toleration does **not** attract a Pod to a tainted node. It only permits the Pod to be placed there *if the scheduler was already considering it*. If you actually want a Pod to prefer or require a specific node, that's **[node affinity](./node-affinity.md)** (a separate mechanism, its own note) — taints/tolerations only ever repel-or-allow, never attract.

## Applying a taint to a node

```bash
kubectl taint nodes node1 key=value:NoSchedule
```

Format: `key=value:effect`

To remove it, add a trailing `-`:

```bash
kubectl taint nodes node1 key=value:NoSchedule-
```

## The three taint effects

| Effect | Meaning |
|---|---|
| **NoSchedule** | New Pods without a matching toleration will not be scheduled here. Pods already running here are left alone. |
| **PreferNoSchedule** | A "soft" version — the scheduler will *try* to avoid this node, but will use it if there's no other option. |
| **NoExecute** | Strongest effect. New Pods without a toleration won't be scheduled here, **and existing Pods already running here that don't tolerate it get evicted.** |

`NoExecute` is the one worth paying extra attention to — it actively kicks out Pods, not just blocks new ones.

## Adding a toleration to a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
  tolerations:
    - key: "key"
      operator: "Equal"
      value: "value"
      effect: "NoSchedule"
```

Fields:
- `key`, `value` — must match the taint's key/value (unless using `Exists`, below)
- `operator` — `Equal` (must match the exact value) or `Exists` (just needs the key to be present, value irrelevant — omit `value` entirely when using `Exists`)
- `effect` — must match the taint's effect (`NoSchedule`, `PreferNoSchedule`, or `NoExecute`)

## `NoExecute` and `tolerationSeconds`

For `NoExecute`, you can optionally add `tolerationSeconds` — this says "tolerate this taint, but only for this long, then evict me anyway":

```yaml
tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoExecute"
    tolerationSeconds: 3600
```

This is useful for things like "if a node becomes unreachable, give its Pods an hour before evicting them" rather than evicting instantly.

## A real example you already have in your cluster

Control plane nodes in a `kubeadm` cluster are tainted by default specifically so ordinary workloads don't land on them:

```bash
kubectl describe node <control-plane-node-name> | grep Taints
# Taints: node-role.kubernetes.io/control-plane:NoSchedule
```

This is why your application Pods only ever end up on worker nodes — nothing is attracting them there specifically, they're just repelled from the control plane node and the worker nodes are the only ones left.

## Useful commands

```bash
kubectl describe node <node-name> | grep Taints     # see a node's taints
kubectl describe pod <pod-name> | grep Tolerations   # see a pod's tolerations
```

## Key idea to remember

**Taint = repel by default. Toleration = permission to ignore a specific repel.** A toleration never attracts — it only cancels out one specific "keep out" sign. If a Pod needs to actually *prefer* or *require* a particular node, that's a different, separate mechanism ([node affinity](./node-affinity.md)), often used together with tolerations when you want both "must avoid everywhere else" and "must land exactly here."

## Exam tips

- Remember the exact format for the imperative command: `kubectl taint nodes <node> key=value:effect`, and the trailing `-` to remove one.
- Know all three effects cold, especially that only `NoExecute` evicts already-running Pods — `NoSchedule`/`PreferNoSchedule` only affect future scheduling decisions.
- A very common exam scenario: "this Pod isn't being scheduled onto this node" — check the node's taints first, then check whether the Pod has a matching toleration.
- Don't confuse this with [node affinity](./node-affinity.md) — if a scenario needs a Pod to *only ever* run on a specific set of nodes, tolerations alone are not enough; you need affinity/nodeSelector too.
