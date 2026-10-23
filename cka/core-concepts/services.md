# Services

> Builds on: [pods.md](./pods.md), [labels-and-selectors.md](./labels-and-selectors.md), and [kube-proxy.md](./kube-proxy.md) — this is the piece kube-proxy.md told you we'd come back to. Read those three first.

## The problem this solves

Pods are disposable. A ReplicaSet can kill and recreate a Pod at any time (crash, rescheduling, scaling), and every time that happens, the new Pod gets a **brand new IP address**. If your frontend talked directly to a backend Pod's IP, that connection would break the moment the backend Pod was replaced.

A **Service** gives you one stable network identity — an IP address and a DNS name — that **never changes**, and sits in front of a shifting set of Pods, always routing traffic to whichever ones are currently healthy and match its selector.

## How a Service finds its Pods

Exactly the same mechanism as ReplicaSets (see `labels-and-selectors.md`): a Service has a `selector`, and it routes traffic to **any currently-running Pod whose labels match it** — not to Pods it "owns" or created, just whatever matches right now.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - port: 80          # the Service's own port
      targetPort: 80     # the port on the Pod to forward to
```

Behind the scenes, Kubernetes maintains an **Endpoints** object (or EndpointSlice, the newer/scalable version) listing the current IPs of every Pod matching that selector. This list updates automatically as Pods come and go — you never manage it yourself.

```bash
kubectl get endpoints nginx-service
```

If a Service's Endpoints list is empty, that's almost always a **selector mismatch** — the single most common Service troubleshooting scenario. Same class of bug as the ReplicaSet selector mismatch covered in `replicaset.md`.

## The three Service types

| Type | What it does | Typical use |
|---|---|---|
| **ClusterIP** (default) | Gives a stable IP reachable only *from inside* the cluster | Internal communication between your own components (e.g. frontend → backend) |
| **NodePort** | Everything ClusterIP does, plus opens the same static port (30000–32767) on **every node's** own IP | Quick/simple way to expose something externally, often for testing |
| **LoadBalancer** | Everything NodePort does, plus asks the cloud provider to provision an external load balancer pointing at it | Production external access on a cloud platform (AWS/GCP/Azure) — does nothing extra on bare-metal/local clusters |

Each type is a superset of the one before it — LoadBalancer includes NodePort behavior, which includes ClusterIP behavior.

```bash
kubectl expose deployment nginx --port=80 --target-port=80 --type=NodePort
```

## DNS: Services get a name, not just an IP

Kubernetes runs an internal DNS service (CoreDNS) that automatically gives every Service a resolvable name, so Pods never need to know a Service's IP at all — just its name:

```
<service-name>.<namespace>.svc.cluster.local
```

From a Pod in the same namespace, just `nginx-service` resolves correctly. From a different namespace, you need `nginx-service.<namespace>`. This is why Pods reference backends by Service name in config/env vars, never by IP.

## How traffic actually gets routed: kube-proxy

This is the piece `kube-proxy.md` previewed: a Service is just a stable *abstraction* — `kube-proxy`, running on every node, is what actually implements the routing by maintaining local networking rules (iptables or IPVS) that intercept traffic sent to the Service's IP and forward it to one of the real Pod IPs from the Endpoints list. The Service object itself doesn't "do" anything on its own; kube-proxy is the mechanism that makes it real on the network.

## Useful commands

```bash
kubectl get svc
kubectl describe svc nginx-service    # check selector + Endpoints together
kubectl get endpoints nginx-service   # confirm which Pods it currently targets
```

## Key idea to remember

A Service is a **stable address backed by a dynamic, selector-matched list of Pods** — the same manager/selector pattern from `replicaset.md`, applied to networking instead of replica count. The Service itself never changes IP even as every Pod behind it is destroyed and replaced repeatedly.

## Exam tips

- Empty Endpoints = selector mismatch, almost every time. `kubectl describe svc` and `kubectl get endpoints` together are the fastest way to confirm this.
- Know the three types cold, in order of "more capability": ClusterIP → NodePort → LoadBalancer.
- Remember the NodePort range: 30000–32767.
- `targetPort` must match the actual container port the Pod is listening on — a mismatch here (Service says 80, container actually listens on 8080) is a classic broken-but-no-error scenario, since the Service and Endpoints will look fine, but connections will fail.
