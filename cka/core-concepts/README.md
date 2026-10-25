# Core Concepts — Review Order

Read in this order — each note builds on the ones before it:

1. [architecture.md](./architecture.md) — the map: control plane vs worker node, how components talk to each other
2. [etcd.md](./etcd.md) — the cluster's memory, foundational to everything else
3. [kube-apiserver.md](./kube-apiserver.md) — the front door everything routes through
4. [kube-scheduler.md](./kube-scheduler.md) — decides where pods go
5. [kube-controller-manager.md](./kube-controller-manager.md) — reconciliation loops reacting to state changes
6. [kubelet.md](./kubelet.md) — where decisions become reality on a node
7. [kube-proxy.md](./kube-proxy.md) — Service networking mechanism (read before services.md)
8. [pods.md](./pods.md) — the smallest deployable unit; ties the whole flow above together
9. [labels-and-selectors.md](./labels-and-selectors.md) — the tagging/filtering mechanism nearly everything below relies on
10. [replicaset.md](./replicaset.md) — keeping a target number of Pods running, via selectors
11. [services.md](./services.md) — stable networking in front of a shifting set of Pods
