# kube-proxy

> Builds on: [cluster architecture](./architecture.md) and [kubelet](./kubelet.md). Read those first if you haven't. This one will make more sense once we've covered Services — treat this as a first pass, we'll revisit when we get to networking.

## What it actually is

**kube-proxy** runs on **every node** (worker and control plane) and is responsible for making **Services** work at the network level. Where kubelet's job is "make containers run," kube-proxy's job is "make sure network traffic sent to a Service reaches one of the right pods" — even as those pods are created, destroyed, or moved around the cluster.

## The problem it solves

Pods are ephemeral — they get new IP addresses every time they're recreated. If you tried to talk to a specific pod IP directly, it would break the moment that pod restarted or got rescheduled. A **Service** gives you one stable IP/DNS name that stays constant, and routes traffic to whichever pods currently match its label selector. kube-proxy is what actually implements that routing on each node.

## How it works (conceptually)

1. kube-proxy watches the API server for **Services** and their **Endpoints** (the current list of pod IPs backing each Service).
2. Whenever that list changes (a pod is added/removed), kube-proxy updates the node's networking rules accordingly.
3. Those rules intercept traffic sent to a Service's IP and redirect it to one of the healthy backing pod IPs — typically load-balanced across them.

The actual mechanism for "updating networking rules" has evolved over Kubernetes versions:

- **iptables mode** (long-time default): kube-proxy writes `iptables` rules on the node that rewrite/redirect traffic destined for a Service IP to a chosen pod IP.
- **IPVS mode**: uses the Linux kernel's IP Virtual Server, which scales better with very large numbers of Services and offers more load-balancing algorithm choices.
- **userspace mode**: an older, slower approach, largely obsolete now.

You won't need to hand-write these rules — just understand that kube-proxy is *generating and maintaining* them automatically, node by node, in reaction to Service/Endpoint changes.

## Where it runs

Unlike kubelet, kube-proxy typically runs as a **DaemonSet** (a pod on every node, managed the normal Kubernetes way) rather than a static pod:

```bash
kubectl get daemonset -n kube-system kube-proxy
kubectl get pods -n kube-system | grep kube-proxy
```

Because it's a normal DaemonSet pod (not static), you *can* inspect and manage it with regular `kubectl` commands, unlike the true static-pod control plane components.

## Troubleshooting angle

If Services aren't routing traffic correctly on a particular node (e.g. you can reach a Service from some nodes but not others), kube-proxy on that node is a prime suspect:

```bash
kubectl logs -n kube-system <kube-proxy-pod-name>
```

Common root causes: kube-proxy pod crashed/not running on that node, or its configured mode (iptables/IPVS) has an issue with the underlying node's networking setup.

## Key idea to remember

kube-proxy doesn't make routing *decisions* about which pod is "best" the way the scheduler does for nodes — it just mechanically keeps each node's low-level network rules in sync with whatever the current, correct set of pod IPs is for each Service. It's the glue between the abstract, stable idea of a "Service" and the messy, constantly-changing reality of actual pod IPs.

## Exam tips

- Know it's typically a **DaemonSet**, not a static pod — so `kubectl` works normally against it, unlike the API server/scheduler/controller-manager/etcd.
- If a Service seems unreachable from specific nodes only (not all), suspect kube-proxy on those specific nodes rather than the Service definition itself.
- We'll come back to this in depth once we cover Services/Networking — for now, just know its role: **Services are the abstraction, kube-proxy is what implements it in the network.**
