# Node Selectors and Node Affinity

> Builds on: [taints-and-tolerations.md](./taints-and-tolerations.md) — read that first. Where taints/tolerations only ever **repel** (or allow ignoring a repel), this note covers the mechanisms that actually **attract** a Pod toward specific nodes.

## The problem this solves

Sometimes you don't just want to keep a Pod *away* from certain nodes — you want to actively **require or prefer** that it lands on specific ones. Example: a Pod that needs a GPU should only run on nodes that actually have one. Taints/tolerations can't express this (a toleration only cancels a repel, it never pulls a Pod toward anything, as covered in `taints-and-tolerations.md`). This is what node selectors and node affinity are for.

Both work off the same foundation: **node labels**. You label nodes (same label mechanism as `labels-and-selectors.md`, just applied to Node objects instead of Pods), then tell a Pod which node labels it needs.

```bash
kubectl label nodes node1 disktype=ssd
```

## nodeSelector — the simple version

The most basic way to attract a Pod to a labeled node:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
  nodeSelector:
    disktype: ssd
```

This Pod will only be scheduled on a node labeled `disktype=ssd`. Simple, but limited — it's pure equality matching only. No "OR," no "exists but I don't care about the value," no "prefer this but don't require it." For anything more expressive than "this exact label equals this exact value," you need node affinity instead.

## Node affinity — the expressive version

Node affinity does the same job as `nodeSelector` but with much richer matching logic, and with a "required vs preferred" distinction.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
```

### The two scheduling rule types

| Rule | Meaning |
|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | **Hard requirement.** The Pod will not be scheduled at all unless a matching node exists. |
| `preferredDuringSchedulingIgnoredDuringExecution` | **Soft preference.** The scheduler tries to honor it, but will still place the Pod elsewhere if no matching node is available. |

(There's a third, newer type — `requiredDuringSchedulingRequiredDuringExecution` — which also evicts a running Pod if the node's labels change later. It's less commonly used and newer; know it exists, don't worry about it deeply for now.)

That long name actually tells you exactly what it means once you break it apart: *"during scheduling"* = when the Pod is first being placed, *"ignored during execution"* = once it's running, Kubernetes won't re-check this and kick the Pod off if the node's labels change later.

### Operators available (this is the real upgrade over nodeSelector)

- `In` — label's value must be one of the listed values
- `NotIn` — label's value must not be one of the listed values
- `Exists` — the label key must be present, any value
- `DoesNotExist` — the label key must not be present
- `Gt` / `Lt` — numeric greater-than / less-than comparison

This is the same `In`/`NotIn`/`Exists` vocabulary as the set-based selectors in `labels-and-selectors.md` — same idea, applied here to nodes instead of Pods.

## Taints/Tolerations vs Node Affinity — the comparison you actually need to understand

This is the single most important distinction in this whole topic, and the exam tests it directly:

| | Taints & Tolerations | Node Affinity |
|---|---|---|
| Lives on | Node (taint) + Pod (toleration) | Pod only |
| What it does | **Repels** Pods from a node by default; a toleration only removes that repulsion for specific Pods | **Attracts** a Pod toward nodes matching certain labels |
| Can it force a Pod onto a specific node? | No — a toleration never attracts, it only permits | Yes — `required...` can force it |
| Can it keep *other* Pods off a node? | Yes — that's its whole purpose | No — node affinity says nothing about what else can land there |

**The key realization:** these two mechanisms solve two different halves of the same problem, and neither one alone fully dictates "only this Pod can run on this node."

- A toleration lets a Pod **land on** a tainted node — but doesn't stop that Pod from also landing on every *other*, untainted node too.
- Node affinity makes a Pod **prefer/require** certain nodes — but doesn't stop *other, unrelated* Pods from also being scheduled there.

**To truly dedicate a node to only specific Pods, you need both together:** taint the node (so nothing lands there by default) + give your target Pods both a toleration for that taint (so they're *allowed* there) and node affinity for that node's labels (so they actually *go* there). This combination is a realistic, common exam scenario.

## Key idea to remember

**Taints/tolerations = repel, with permission to override. Node affinity = attract, optionally mandatory.** They're complementary, not alternatives — one restricts who can land, the other decides who wants to.

## Exam tips

- `nodeSelector` is fine for simple, single-label equality checks — don't overcomplicate with node affinity if the scenario is that simple.
- Know `requiredDuringSchedulingIgnoredDuringExecution` vs `preferredDuringSchedulingIgnoredDuringExecution` cold — "will it fail to schedule if no match exists, or just try its best" is a very testable distinction.
- Remember the operator list (`In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`) — scenarios often require matching multiple possible values, which `nodeSelector` literally cannot express.
- If a scenario says "only this Pod should run on this node, and no other Pods should either" — that's your cue it needs **both** taints/tolerations **and** node affinity, not just one.
