# ReplicationController and ReplicaSet

> Builds on: [pods.md](./pods.md) and [labels-and-selectors.md](./labels-and-selectors.md) — read both first, especially labels-and-selectors, since this entire object is built on the selector mechanism it describes.

## The problem this solves

From `pods.md`: a bare Pod you create directly is fragile. Nothing watches it — if it crashes or the node dies, it's just gone. **ReplicationController** and **ReplicaSet** exist to fix exactly that: they are controller objects whose entire job is to keep a specified *number* of Pod replicas running, at all times, automatically.

This is the "desired state vs actual state" reconciliation loop from `architecture.md`, applied specifically to Pod count: you declare "I want 3 replicas," and the controller continuously checks reality and corrects it if it drifts.

## ReplicationController vs. ReplicaSet — what's the actual difference?

**ReplicationController (RC)** is the original, older object that did this. **ReplicaSet (RS)** is its direct replacement and does the same job, better:

| | ReplicationController | ReplicaSet |
|---|---|---|
| Status | Legacy, effectively deprecated | Current, standard |
| Selector support | Equality-based only (`app: nginx`) | Equality-based **and** set-based (`in`, `notin`, `exists`) — see `labels-and-selectors.md` |
| API | `apiVersion: v1` | `apiVersion: apps/v1` |

**You will basically never create a ReplicationController in practice or expect to be tested on writing one from scratch** — it's covered mainly so you recognize it and understand what ReplicaSet replaced. ReplicaSet is what you'll actually use (and even that, indirectly — see below).

## How a ReplicaSet works

1. You define a ReplicaSet with a desired `replicas` count and a **pod template** (what a Pod should look like) plus a **selector** (which Pods it's responsible for).
2. The ReplicaSet controller (part of `kube-controller-manager`) continuously counts how many currently-running Pods match its selector.
3. Too few? It creates new Pods from the template (via the API server, same flow as always) to make up the difference.
4. Too many? It deletes extras.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
```

Note there are **two places `app: nginx` needs to match**: the top-level `selector.matchLabels` and the `template.metadata.labels`. If the template's labels don't satisfy the selector, the object would be logically self-contradictory — a ReplicaSet whose own selector could never match the Pods it creates, since each new Pod wouldn't count toward what it's looking for. Because that situation can never resolve itself, **Kubernetes doesn't even let you create it** — the API server validates this at creation time and rejects the object outright with an error like:

```
error: ReplicaSet.apps "nginx-rs" is invalid: spec.template.metadata.labels:
Invalid value: map[string]string{"app":"wrong-label"}: `selector` does not match template `labels`
```

This is one of the most common hands-on mistakes when writing ReplicaSet YAML from scratch — but the good news is you'll see this exact validation error immediately rather than something silently broken.

## The selector adopts existing Pods too

Because of how selectors work (see `labels-and-selectors.md`), a ReplicaSet doesn't only create new Pods — if matching Pods *already exist* when it's created, it counts them toward its `replicas` total and "adopts" them as if it had created them itself. This is a genuinely useful exam trick: you can pre-create Pods manually with the right labels, then point a ReplicaSet at them with a lower replica count than the number of existing Pods, and watch it delete the extras — a good way to test your understanding of exactly how the matching/reconciliation works.

## Scaling

```bash
kubectl scale replicaset nginx-rs --replicas=5
# or edit the YAML's replicas field and re-apply
kubectl apply -f replicaset.yaml
```

## Useful commands

```bash
kubectl get rs
kubectl describe rs nginx-rs      # Events section shows scaling actions taken
kubectl delete rs nginx-rs        # by default also deletes its Pods
kubectl delete rs nginx-rs --cascade=orphan   # deletes the RS but leaves Pods running, unmanaged
```

## Where this is going: you won't create ReplicaSets directly either

Just like a bare Pod is normally created *through* a ReplicaSet rather than by itself, in practice a ReplicaSet is normally created *through* a **Deployment** — the next object in the course. A Deployment manages ReplicaSets the same way a ReplicaSet manages Pods, and adds rolling updates/rollbacks on top. You still need ReplicaSets understood properly though, since that's the layer actually doing the replica-count enforcement underneath every Deployment.

## Key idea to remember

This is the same manager/selector pattern as everything else in Kubernetes: a ReplicaSet doesn't track "its" Pods by identity — it continuously re-evaluates "how many currently-existing Pods match my selector" and reconciles that number toward `replicas`. Selector correctness is everything; get the selector or the template labels wrong, and the whole mechanism breaks silently or behaves unexpectedly.

## Exam tips

- Double-check `selector.matchLabels` matches `template.metadata.labels` exactly whenever writing a ReplicaSet YAML — this is the single most common error.
- Know that deleting Pods managed by a ReplicaSet doesn't actually remove them for long — the controller notices the count dropped and immediately creates replacements. (To actually remove them, scale down or delete the ReplicaSet.)
- `kubectl delete rs <name> --cascade=orphan` is worth knowing explicitly — it's an easy way to test/demonstrate the adoption behavior, and a realistic scenario question.
- You almost certainly won't need to write a ReplicationController from scratch — recognize it, know it's legacy, move on.
