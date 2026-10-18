# Pods

> Builds on: [architecture.md](./architecture.md) — this is the first object you'll actually create. Everything in that note's "how it fits together" flow was really describing how a Pod gets created and scheduled.

## What it actually is

A **Pod** is the smallest deployable unit in Kubernetes — not a container. This trips people up at first: you don't ask Kubernetes to run a container directly, you always ask it to run a **Pod**, which wraps one or more containers.

A Pod is a thin wrapper that gives its container(s):
- A single **IP address**, shared by everything inside the Pod
- A single **network namespace** — containers in the same Pod can reach each other via `localhost`
- Optionally, shared **storage volumes**
- A single scheduling decision — the whole Pod lands on one node together, never split across nodes

## Why wrap containers at all?

Because some workloads genuinely need more than one container running and tightly coupled together — sharing network and disk — while still being managed, scheduled, and scaled as a single unit. Example: a main app container plus a "sidecar" container that ships its logs somewhere, both needing to share the same filesystem and both living or dying together.

**Most Pods have exactly one container.** Multi-container Pods are the exception, used for specific patterns (sidecar, init containers, ambassador, adapter) — not the default way to run things. If you're not sure, one container per Pod is the right default.

## Creating a Pod

### Imperatively (quick, one-off)

```bash
kubectl run nginx --image=nginx
```

### Declaratively (the way you'll actually use in practice and in the exam)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f pod.yaml
```

Four fields every Kubernetes object YAML needs, Pods included: `apiVersion`, `kind`, `metadata`, `spec`. This structure is identical for every object type you'll create (Deployments, Services, etc.) — only what goes inside `spec` changes.

## A useful shortcut: generate YAML instead of writing it from scratch

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```

`--dry-run=client` creates nothing — it just prints what *would* be created. Combined with `-o yaml`, this gives you a correct starting YAML file to edit, rather than typing the structure from memory. This is a genuinely important exam technique — it's faster and less error-prone than writing YAML freehand under time pressure.

## Pod lifecycle (phases)

A Pod moves through phases as it's created and runs:

| Phase | Meaning |
|---|---|
| `Pending` | Accepted by the API server, but not yet scheduled to a node, or scheduled but image still being pulled |
| `Running` | Bound to a node, at least one container is running |
| `Succeeded` | All containers terminated successfully (exit code 0) — expected for one-off/batch Pods |
| `Failed` | At least one container terminated with a non-zero exit code |
| `Unknown` | The Pod's state can't be determined, usually because the node it's on can't be reached |

You check this with:

```bash
kubectl get pods
kubectl describe pod <name>   # includes Events — the real troubleshooting goldmine
```

`kubectl describe pod` is where you'll spend most of your troubleshooting time — the **Events** section at the bottom shows exactly what happened (image pull errors, scheduling failures, crashes) in chronological order.

## Restart policy

Every Pod has a `restartPolicy` (default `Always`), which controls what happens when a container inside it exits:

- `Always` — always restart the container (the default; correct for long-running apps like web servers)
- `OnFailure` — restart only if it exited with an error
- `Never` — don't restart (correct for one-off/batch jobs you expect to run once and finish)

## How this connects to what you already know

This is the exact flow from `architecture.md`, now with a name for the object being created:

1. You `kubectl apply` a Pod manifest → **kube-apiserver**.
2. API server writes it to **etcd**, with no node assigned yet (`Pending`).
3. **kube-scheduler** sees the unassigned Pod, picks a node, reports back.
4. API server records the assignment in etcd.
5. **kubelet** on that node sees the assignment and tells the **container runtime** to pull the image and start the container(s).
6. Pod moves to `Running`.

If you create a *bare* Pod directly like above, there's no controller watching to recreate it if it dies — delete it, and it's just gone. In practice, you'll almost always create Pods indirectly through a **Deployment** (coming up next), which uses the `kube-controller-manager`'s reconciliation loop to keep a target number of Pods running at all times.

## Key idea to remember

A Pod is **one scheduling unit, one IP, one group of containers that live and die together** — never a container by itself. A bare Pod with no controller managing it is fragile (nothing recreates it if it dies); that fragility is exactly the problem Deployments solve.

## Exam tips

- `kubectl run <name> --image=<image> --dry-run=client -o yaml > file.yaml` is one of the highest-value commands to know cold — generate first, edit second.
- When troubleshooting a broken Pod, always run `kubectl describe pod` and read the Events section before anything else.
- Know the difference between `Pending` (not yet scheduled, or image pulling) vs `Failed` (ran and errored) vs `Unknown` (node unreachable) — the exam tests whether you can read the actual cause rather than guess.
- Remember: Pods you create directly (no Deployment/ReplicaSet) are not self-healing. If the exam scenario is about a Pod not coming back after being deleted, check whether it's managed by a controller at all.
