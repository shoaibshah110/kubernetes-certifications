# kube-controller-manager

> Builds on: [cluster architecture](./architecture.md) and [kube-apiserver](./kube-apiserver.md). Read those first if you haven't.

## What it actually is

The **kube-controller-manager** is the component that continuously enforces the "desired state vs actual state" idea described in the architecture note. In practice, it isn't one single controller — it's a single binary that bundles together **many independent controllers**, each responsible for one type of resource, all running the same basic loop:

1. **Watch** the current state of some resource (via the API server).
2. **Compare** it to the desired state.
3. If they differ, **take action** to correct it (via the API server again).
4. Repeat, forever.

This loop is called a **control loop** or **reconciliation loop**, and it's the fundamental pattern behind almost everything Kubernetes automates.

## Some of the controllers bundled inside it

| Controller | What it watches for and does |
|---|---|
| **Node controller** | Watches node health (via kubelet heartbeats). If a node stops responding, marks it `NotReady` and, after a grace period, evicts its pods so they can be rescheduled elsewhere. |
| **Replication controller** (and ReplicaSet controller) | Ensures the number of running pod replicas matches what's requested — creates new pods if too few exist, deletes extras if too many. |
| **Deployment controller** | Manages rolling updates/rollbacks by orchestrating ReplicaSets underneath a Deployment. |
| **Endpoints controller** | Keeps a Service's list of matching pod IPs (Endpoints) up to date as pods come and go. |
| **Namespace controller** | Handles cleanup when a namespace is deleted — makes sure everything inside it is garbage collected too. |
| **Service Account & Token controllers** | Creates default service accounts and their tokens for new namespaces. |

You don't need to memorize every controller — the important idea is that **each one only cares about its own narrow slice of state**, and none of them talk to each other. They all independently watch and act through the API server.

## A concrete example

Say a Deployment says "run 3 replicas of nginx," and one of those pods' nodes crashes:

1. The **node controller** notices the node stopped sending heartbeats, marks it `NotReady`, then eventually evicts the pods that were on it.
2. The **ReplicaSet controller** notices there are now only 2 running replicas of nginx instead of 3 (desired), so it creates a new pod to make up the difference.
3. That new pod (with no node assigned) is picked up by the **scheduler**, which assigns it a healthy node.
4. The **kubelet** on that node starts the container.

Note how many independent components cooperated here without ever talking to each other directly — each one just watches the API server and reacts.

## Where it runs

Static pod on the control plane node(s) in a `kubeadm` cluster:

```
/etc/kubernetes/manifests/kube-controller-manager.yaml
```

```bash
kubectl get pods -n kube-system | grep controller-manager
```

## A couple of useful flags

- `--node-monitor-grace-period` — how long to wait before marking an unresponsive node `NotReady` (default 40s).
- `--pod-eviction-timeout` — how long to wait after that before evicting the dead node's pods (default 5m in older versions; behavior has evolved across versions, worth checking current defaults).

## Key idea to remember

The controller-manager is **many small, independent "keep reality matching desired state" loops bundled into one process**. None of them do the actual scheduling or container-running themselves — they just detect drift and issue corrective requests through the API server, same as any other client.

## Exam tips

- If self-healing seems broken (e.g. deleted pods aren't being recreated, or a Deployment isn't reconciling), check controller-manager health.
- Know the node → `NotReady` → eviction → rescheduling chain — it's a common scenario for "a node went down, what happens to its pods and how long does it take" questions.
- Static pod manifest path: `/etc/kubernetes/manifests/kube-controller-manager.yaml`.
