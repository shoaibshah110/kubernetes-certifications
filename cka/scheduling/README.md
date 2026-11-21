# Scheduling — Review Order

1. [taints-and-tolerations.md](./taints-and-tolerations.md) — how nodes repel Pods by default, and how Pods get permission to ignore that
2. [node-affinity.md](./node-affinity.md) — nodeSelector and node affinity: how to attract a Pod toward specific nodes, and how this combines with taints/tolerations
3. [resource-requirements.md](./resource-requirements.md) — requests vs limits, how the scheduler uses them, and what happens when a container exceeds each
4. [daemonsets.md](./daemonsets.md) — running exactly one Pod per node, automatically, using the same scheduling mechanisms as everything else
5. [static-pods.md](./static-pods.md) — Pods placed by a manifest file on one node's disk, bypassing the scheduler entirely; includes the Static Pod vs DaemonSet comparison
