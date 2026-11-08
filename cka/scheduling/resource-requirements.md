# Resource Requirements and Limits

> Builds on: [pods.md](../core-concepts/pods.md) and [kube-scheduler.md](../core-concepts/kube-scheduler.md) — this is the mechanism behind the scheduler's "filtering" step mentioned there (eliminating nodes without enough free resources).

## The problem this solves

Nodes have finite CPU and memory. Without telling Kubernetes what a container actually needs, the scheduler is flying blind — it can't make good placement decisions, and a single misbehaving container could consume an entire node's resources and starve everything else on it. **Requests** and **limits** are how you describe a container's resource needs and boundaries.

## Requests vs Limits — two different jobs

This is the core distinction, and it trips people up because both are set in the same place with similar-looking syntax but mean very different things:

- **`requests`** — what a container is **guaranteed** to get, and what the **scheduler uses to decide placement**. "This container needs at least this much."
- **`limits`** — the **maximum** a container is ever allowed to use. "This container can never exceed this much."

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

## How the scheduler uses `requests`

When the scheduler filters nodes (the first phase from `kube-scheduler.md`), it eliminates any node that doesn't have at least the **requested** amount of CPU/memory still free. This is a reservation, not a cap — the container can use more later if it's available (up to its limit), but the scheduler guarantees this baseline is available when placing it.

**`limits` play no role in scheduling at all** — only `requests` do. A node could be selected for a Pod whose limit is very high, as long as its request fits.

## Units: CPU and memory

**CPU** is measured in "cores," but almost always written in **millicores** (`m`):
- `1` = one full CPU core
- `500m` = half a core
- `100m` = a tenth of a core

**Memory** is measured in bytes, with standard suffixes:
- `Mi` = mebibytes (1024-based) — e.g. `128Mi`
- `Gi` = gibibytes — e.g. `1Gi`
- (`M`/`G`, without the `i`, are the 1000-based SI versions — less commonly used but valid)

## What happens when a container exceeds its limit

This is where CPU and memory behave **completely differently** — a commonly tested distinction:

- **Exceeding a CPU limit** → the container is **throttled**. It's not killed; it's just slowed down, forced to use less CPU than it wants. CPU is "compressible" — you can take it away gradually.
- **Exceeding a memory limit** → the container is **killed** (OOMKilled — Out Of Memory Killed) and restarted (subject to `restartPolicy`, covered in `pods.md`). Memory is "incompressible" — you can't partially take memory back once it's in use, so the only option is to terminate the process.

You'll see this as `OOMKilled` in `kubectl describe pod` → container state, or `kubectl get pods` showing a high restart count with `CrashLoopBackOff` if it keeps happening.

## What happens if you don't set requests/limits at all

If you omit them entirely, the container has no guaranteed minimum and no cap — it can use as much as the node has available, and could starve other Pods on the same node (or get starved by them). In a shared/namespaced environment, cluster admins can enforce sane defaults automatically using a **LimitRange** (sets default/min/max per namespace) or a **ResourceQuota** (caps total resource usage across a whole namespace) — both are namespace-level objects, not something you set per-Pod, but worth recognizing by name.

## Useful commands

```bash
kubectl describe pod <name>     # shows requests/limits and current resource usage
kubectl top pod                  # live CPU/memory usage (requires metrics-server)
kubectl top node                 # live per-node resource usage
```

## Key idea to remember

**Requests = reservation used for scheduling. Limits = hard ceiling used at runtime.** They answer two different questions — "where should this go?" (requests) vs "how far can this grow once it's running?" (limits) — and exceeding each one has a completely different consequence: CPU gets throttled, memory gets the container killed.

## Exam tips

- Know the CPU/memory unit notations cold: `m` for millicores, `Mi`/`Gi` for memory.
- Remember only `requests` affect scheduling — if a Pod is stuck `Pending` with an "Insufficient cpu/memory" event, that's about requests not fitting on any node, not limits.
- Memorize the asymmetry: CPU limit exceeded → throttled (keeps running, slower). Memory limit exceeded → OOMKilled (terminated, restarted). This is a very common exam/troubleshooting question.
- If you see `CrashLoopBackOff` with repeated `OOMKilled` reasons in `kubectl describe pod`, the fix is usually to raise the memory limit (or fix a memory leak) — not touch CPU settings.
