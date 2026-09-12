# kubelet

> Builds on: [cluster architecture](./architecture.md) and [kube-apiserver](./kube-apiserver.md). Read those first if you haven't.

## What it actually is

The **kubelet** is the agent that runs on **every worker node** (and, in a `kubeadm` cluster, also on control plane nodes, since control plane components run as regular containers too). It's the component that actually makes things happen locally — everything upstream (API server, scheduler, controller-manager) only ever *decides* things; the kubelet is what *executes* them on a given machine.

Unlike the control plane components, **kubelet is not a static pod itself** — it's a regular process/service running directly on the node's OS (managed by systemd), since it's the thing responsible for running static pods and containers in the first place. Something has to be running before anything else can start.

## What it does

1. **Watches the API server** for pods assigned to its own node (`.spec.nodeName == <this node>`).
2. For each assigned pod, tells the **container runtime** (via the CRI — Container Runtime Interface) to pull the image and start the container(s).
3. Continuously monitors those containers' health.
4. **Reports back** to the API server: node status (capacity, conditions, heartbeats) and pod status (running, failed, restarting, etc.).
5. Runs **static pods**: reads YAML manifest files from a local directory (default `/etc/kubernetes/manifests/`) and runs them directly — without ever going through the scheduler or API server for the *decision* to run them. This is exactly how control plane components (API server, etcd, scheduler, controller-manager) start on a `kubeadm` node — kubelet just finds their manifest files on disk and runs them.
6. Runs periodic **liveness/readiness/startup probes** you've configured on pods, and acts on the results (restarting a container that fails its liveness probe, for instance).

## Static pods, revisited

This is worth calling out clearly since it explains something you'll rely on throughout the course: static pods are **not managed through the API server at all** in the normal sense. kubelet watches a directory on disk (`/etc/kubernetes/manifests/` by default, configurable via `--pod-manifest-path` or the kubelet config file), and:

- Drop a valid pod YAML in there → kubelet starts it.
- Edit that file → kubelet detects the change and restarts the pod with new config.
- Delete the file → kubelet stops and removes the pod.

kubelet does mirror static pods into the API server as read-only "mirror pods" so you can *see* them with `kubectl get pods`, but you can't manage them with `kubectl delete`/`kubectl edit` — the file on disk is the only real source of truth for them.

## The node heartbeat

kubelet periodically reports node status to the API server (roughly every 10s by default). This is how the **node controller** (part of controller-manager) knows a node is alive. If heartbeats stop, the node controller eventually marks the node `NotReady` and, after a further grace period, evicts its pods (see [kube-controller-manager](./kube-controller-manager.md)).

## Where to find it / troubleshoot it

Since it's not a pod, you don't find it with `kubectl` — it's a systemd service on the node itself:

```bash
systemctl status kubelet
journalctl -u kubelet -f          # live logs
```

If kubelet is down on a node, that node won't run new pods, won't report status, and will eventually be marked `NotReady` by the control plane — even though any *already-running* containers on it might keep running for a while (container runtime keeps them alive independently), the node is effectively unmanaged until kubelet comes back.

## Config

Two places to look:
- Command-line flags/service definition, typically at `/var/lib/kubelet/kubeconfig` (for how it authenticates to the API server) and `/etc/systemd/system/kubelet.service.d/` (drop-in service config).
- A dedicated kubelet config file (commonly `/var/lib/kubelet/config.yaml`), which sets things like the pod manifest path, cgroup driver, and eviction thresholds.

## Key idea to remember

kubelet is the **only component that actually causes a container to run**. Every other component in the cluster is essentially "upstream" of it — deciding, tracking, or reacting — but kubelet is where decisions turn into reality on an actual machine. It's also the one component fundamental enough that it can't depend on the API server existing yet, which is exactly why it's the thing responsible for bootstrapping the control plane itself via static pods.

## Exam tips

- Not a pod — troubleshoot it with `systemctl`/`journalctl`, not `kubectl`.
- Know the default static pod manifest path (`/etc/kubernetes/manifests/`) cold — you'll use it for etcd backup/restore, forcing control plane component restarts, and general troubleshooting.
- If a node shows `NotReady`, SSH into it and check kubelet's status/logs first — common root causes include kubelet being stopped, misconfigured, or unable to reach the API server (cert/network issues).
- Remember mirror pods: you can *see* static pods via `kubectl get pods` but can't delete/edit them that way — you must edit/remove the manifest file on the node.
