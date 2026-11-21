# Static Pods

> Builds on: [kubelet.md](../core-concepts/kubelet.md), which already introduced static pods as part of explaining what kubelet does. This note goes deeper, specifically from the scheduling angle — why static pods matter to this section at all.

## Quick recap, then why this belongs in "Scheduling"

From `kubelet.md`: a static pod is a Pod whose manifest YAML sits directly on a node's filesystem (default `/etc/kubernetes/manifests/`), and **kubelet** on that node reads it and runs it directly — no API server round-trip needed to create it.

The reason this belongs in the Scheduling section specifically: **static pods completely bypass the scheduler.** There's no "which node is the best fit" decision happening at all — the Pod's node is decided simply by *which node's disk the file is sitting on*. This is the same "bypass the scheduler" idea briefly mentioned in `kube-scheduler.md` (manually setting `nodeName`) — except here it's not a field in the Pod spec, it's physically where the file lives.

## How kubelet finds these files

The directory kubelet watches is configurable — check either:

```bash
ps -ef | grep kubelet | grep -i manifest       # look for --pod-manifest-path flag
cat /var/lib/kubelet/config.yaml | grep staticPodPath   # or check the kubelet config file
```

Default in a `kubeadm` cluster: `/etc/kubernetes/manifests/`. This is also exactly where you'll find the control plane components themselves (`kube-apiserver.yaml`, `etcd.yaml`, `kube-scheduler.yaml`, `kube-controller-manager.yaml`) — this is how `kubeadm` bootstraps the control plane in the first place, using the very mechanism this note is about.

## Creating one

Just drop a valid Pod YAML into that directory on the node you want it to run on:

```bash
cp my-pod.yaml /etc/kubernetes/manifests/
```

kubelet notices the new file (it polls the directory periodically) and starts the Pod — no `kubectl apply` involved at all, and in fact no API server interaction is required to create it.

## Deleting one

Remove the file from disk:

```bash
rm /etc/kubernetes/manifests/my-pod.yaml
```

**`kubectl delete pod <name>` will not work as you'd expect here** — even if it appears to succeed, kubelet still has the manifest file and will simply recreate the Pod immediately. The file on disk is the real source of truth; the API only ever shows a read-only reflection of it (a "mirror pod," per `kubelet.md`).

## How to tell if a Pod is static, just by looking at it

A strong giveaway: static pod names get the **node's hostname automatically appended** as a suffix:

```
kube-apiserver-controlplane
etcd-controlplane
my-pod-node01
```

If you `kubectl get pods -n kube-system` and see names ending in what looks like a node name, that's your signal. You can confirm with:

```bash
kubectl describe pod <name> | grep -i "Controlled By"
# Controlled By:  Node/<node-name>
```

Static pods show `Node/<name>` here, instead of a controller like a ReplicaSet.

## Static Pods vs DaemonSets — don't mix these up

Both can feel like "a Pod that's tied to a node," but they're solving completely different problems, through completely different mechanisms:

| | Static Pod | DaemonSet |
|---|---|---|
| Defined by | A YAML file on one specific node's disk | A single cluster-wide object via the API server |
| Runs on | Only the node(s) where you've manually placed the file | **Every** matching node automatically |
| Created via | kubelet reading local disk, scheduler never involved | Normal API object → scheduler-equivalent node affinity (see `daemonsets.md`) |
| Managed with `kubectl`? | No (mirror pod is read-only) | Yes, fully manageable |
| New node joins cluster | Nothing happens — you'd have to manually copy the file there | A Pod is automatically created there too |

**Rule of thumb:** if the goal is "one per node, across the whole cluster, automatically, forever" → DaemonSet. If the goal is "this exact file must run on this exact node regardless of the API server" (e.g. bootstrapping the control plane before an API server even exists to talk to) → static pod.

## Key idea to remember

A static pod isn't scheduled — it's **placed by definition**, since the only "decision" is which node's filesystem you copied the file to. This is precisely why it's the mechanism `kubeadm` relies on to start the control plane: kubelet can run `kube-apiserver` as a static pod *before* there's any API server for a scheduler to talk to in the first place.

## Exam tips

- Know the default manifest path (`/etc/kubernetes/manifests/`) and how to find it if it's been customized (`ps -ef | grep kubelet`, or the kubelet config file).
- A common scenario: "identify which of these Pods is a static Pod" — check the name suffix and `Controlled By: Node/...` in `kubectl describe pod`.
- Remember: to actually remove or change a static pod, you edit/delete the file on the node itself — `kubectl` commands alone won't stick.
- Don't confuse with DaemonSets — if a question says "running on all nodes," that's DaemonSet; if it says "this specific node, manifest-based," that's a static pod.
