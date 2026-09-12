# etcd

> Builds on: [cluster architecture](./architecture.md) — read that first if you haven't. There, etcd was described simply as "the memory." This note goes deeper.

## What etcd actually is

etcd is an open-source, **distributed key-value store**. It's not Kubernetes-specific — it's a general-purpose project (from CoreOS, now CNCF) that anything could use. Kubernetes happens to use it as its database.

"Key-value store" means data is stored as simple `key → value` pairs, like a giant dictionary/hash map — not tables and rows like a SQL database. For example, roughly:

```
/registry/pods/default/nginx-pod        → { ...full pod spec/status as JSON... }
/registry/deployments/default/nginx     → { ...deployment spec... }
/registry/nodes/worker-node-1           → { ...node info... }
```

**Everything** in a cluster — every pod, deployment, service, secret, configmap, namespace, node — is a key under `/registry/...` in etcd. When you run `kubectl get pods`, the API server is effectively reading this data out of etcd and formatting it for you.

## Why Kubernetes needs it to be "distributed" and reliable

etcd is the **single source of truth** for the entire cluster's state. If etcd's data is lost or corrupted, Kubernetes has no memory of what should exist — deployments, secrets, everything is effectively gone (the actual running containers might survive briefly, but nothing can be managed or reconciled anymore).

Because it's this critical, etcd is designed to run as a **cluster of its own** (typically 3 or 5 instances/nodes) rather than a single instance, so it can survive a machine going down without losing data. This is achieved through the **Raft consensus algorithm**.

### Raft, in plain terms

- One etcd instance is elected **leader**; the others are **followers**.
- Every write (e.g. "create this pod") goes to the leader first, which replicates it to followers.
- A write is only confirmed as successful once a **majority (quorum)** of instances have it saved.
- If the leader dies, the followers hold an election and pick a new leader.

**Why an odd number (3, 5, 7) of etcd instances?** Quorum = majority = `(n/2)+1`. With 3 nodes, quorum is 2 — the cluster tolerates 1 node failing. With 5 nodes, quorum is 3 — tolerates 2 failing. An even number (e.g. 4) gives you the same fault tolerance as one fewer (3) but costs more resources, so it's never worth it. This is a common exam-style gotcha.

## Where etcd runs in your cluster

Two common setups:

- **Stacked etcd** (default with `kubeadm`, and what you'll mostly see in the exam): etcd runs on the same node(s) as the other control plane components, as a static pod.
- **External etcd**: etcd runs on its own separate set of machines, decoupled from the control plane nodes. More resilient (a control plane node dying doesn't risk etcd data) but more infrastructure to manage.

You can see it as a static pod in a `kubeadm` cluster with:

```bash
kubectl get pods -n kube-system | grep etcd
# or, if kubectl isn't available (e.g. cluster is broken):
crictl ps | grep etcd
```

## Talking to etcd directly: etcdctl

`etcdctl` is etcd's own CLI tool — separate from `kubectl`. You'd use it to inspect etcd directly or, most importantly, to **back it up and restore it**.

There are two API versions; always use v3 (v2 is legacy):

```bash
export ETCDCTL_API=3
```

Because etcd's client communication is secured with TLS certificates, most `etcdctl` commands need to point at the certs (paths below are the standard `kubeadm` defaults):

```bash
etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  <command>
```

## Backup and restore — the big exam topic

This is the single most exam-relevant etcd skill: **you will very likely be asked to back up etcd and/or restore it from a snapshot.**

### Taking a snapshot (backup)

```bash
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

This produces a single `.db` file containing the entire cluster state at that moment.

You can check a snapshot's status/health:

```bash
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db --write-out=table
```

### Restoring from a snapshot

Restoring doesn't overwrite the running etcd in place — it creates a **new data directory** from the snapshot:

```bash
ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir=/var/lib/etcd-from-backup
```

Then you point etcd's static pod manifest at that new data directory (edit `/etc/kubernetes/manifests/etcd.yaml`, change the `--data-dir` and the hostPath volume to `/var/lib/etcd-from-backup`), and kubelet will automatically restart the etcd static pod using the restored data.

**Why this matters for the exam:** static pods are auto-restarted by kubelet whenever their manifest file in `/etc/kubernetes/manifests/` changes — no `kubectl apply` needed or even possible for them.

## Ports to remember

- **2379** — client port (used by kube-apiserver and `etcdctl` to talk to etcd)
- **2380** — peer port (used by etcd instances to talk to each other for Raft)

## Key idea to remember

etcd is not "a database Kubernetes happens to use" in a casual sense — it **is** the cluster's entire state. The API server is the only component that talks to it directly; everyone else (scheduler, controller-manager, kubelet) goes through the API server. Lose etcd (with no backup) and you lose the cluster's memory of everything it's supposed to be running, even if the actual containers are still physically running somewhere.

## Exam tips

- Memorize the `snapshot save` / `snapshot status` / `snapshot restore` commands — practice them until they're muscle memory, exact flags and all.
- Know the default cert paths under `/etc/kubernetes/pki/etcd/` for a `kubeadm` cluster.
- Remember: restore creates a new data directory; you then have to repoint the static pod manifest at it.
- Odd-number quorum reasoning (3 or 5 nodes, not 4) is a classic conceptual question.
- `ETCDCTL_API=3` — if commands seem to silently fail or behave oddly, check this is set.
