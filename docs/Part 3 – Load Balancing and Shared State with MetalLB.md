# Kubernetes Homelab Series Part 2 - Secrets With SOPS and age <!-- omit from toc -->

---

## Table of Contents <!-- omit from toc -->
<!-- TOC -->
- [References](#references)
- [1. Introduction](#1-introduction)
- [2. Why load balancing changes the risk profile](#2-why-load-balancing-changes-the-risk-profile)
- [3. Why MetalLB](#3-why-metallb)
- [4. Scope and non-goals](#4-scope-and-non-goals)
- [5. Address planning and ownership](#5-address-planning-and-ownership)
  - [5.1 Selecting an address pool](#51-selecting-an-address-pool)
- [5.2 Ownership boundaries](#52-ownership-boundaries)
- [6. Installing MetalLB](#6-installing-metallb)
  - [6.1 Install MetalLB components](#61-install-metallb-components)
- [7. Configuring the address pool](#7-configuring-the-address-pool)
  - [7.1 IPAddressPool](#71-ipaddresspool)
  - [7.2 L2Advertisement](#72-l2advertisement)
- [8. Validating load balancing behaviour](#8-validating-load-balancing-behaviour)
  - [8.1 Deploy a test workload](#81-deploy-a-test-workload)
  - [8.2 Verify IP allocation](#82-verify-ip-allocation)
- [9. Failure considerations](#9-failure-considerations)
- [10. Summary](#10-summary)
<!-- /TOC -->

---

## References

- [Siderolabs.com - Talos on Proxmox](https://docs.siderolabs.com/talos/v1.11/platform-specific-installations/virtualized-platforms/proxmox)
- Kubernetes Homelab Series
  - [Part 1 - Introduction and Talos Installation](https://blog.dalydays.com/post/kubernetes-homelab-series-part-1-talos-linux-proxmox/)
  - [Part 2 - Secrets With SOPS and age](https://blog.dalydays.com/post/kubernetes-homelab-series-part-2-sops-and-age/)
  - [Part 3 - LoadBalancer With MetalLB](https://blog.dalydays.com/post/kubernetes-homelab-series-part-3-loadbalancer-with-metallb/)
  - [Part 4 - Certificates With cert-manager and Let's Encrypt](https://blog.dalydays.com/post/kubernetes-homelab-series-part-4-certificates-with-cert-manager-and-lets-encrypt/)
  - [Part 4.5 - Debug Pod](https://blog.dalydays.com/post/kubernetes-homelab-series-part-4.5-debug-pod/)
  - [Part 5 - Ingress Controllers With Traefik](https://blog.dalydays.com/post/kubernetes-homelab-series-part-5-ingress-controllers-with-traefik/)
  - [Part 6 - Storage With democratic-csi](https://blog.dalydays.com/post/kubernetes-homelab-series-part-6-storage-with-democratic-csi/)
  - [Part 7 - Backups With Velero](https://blog.dalydays.com/post/kubernetes-homelab-series-part-7-backups-with-velero/)
  - [Kubernetes - External Services And Ingress](https://blog.dalydays.com/post/kubernetes-ingress-to-external-service/)
  - [Upgrade Talos Linux and Kubernetes](https://blog.dalydays.com/post/kubernetes-talos-upgrades/)
  - - [Kubernetes Storage - OpenEBS Replicated Storage Mayastor](https://blog.dalydays.com/post/kubernetes-storage-with-openebs/)
- [Talos Factory - iso generator](https://factory.talos.dev/)

---

## 1. Introduction

Up to this point, the cluster has been deliberately inward-facing. 

Control plane access, secrets, and configuration have been treated as privileged concerns, with a strong emphasis on recovery, auditability, and controlled change. This was intentional. Once external access is introduced, mistakes become harder to contain and recovery paths narrower.

Part 3 introduces shared state in the form of load-balanced services. This is the first time workloads running inside the cluster become directly reachable from the network. As a result, failure modes expand from “cluster is unhealthy” to “clients are impacted”.

This part focuses on making that transition safely.

---

## 2. Why load balancing changes the risk profile

In Kubernetes, a Service of type LoadBalancer represents more than convenience. It is a contract between the cluster and its environment.

Once load balancing is enabled:

- IP addresses become externally significant
- Network ownership must be explicit
- Failure affects consumers, not just operators
- Configuration drift has visible consequences

For managed clouds, these concerns are abstracted away. In a homelab, they are not. MetalLB fills this gap by providing a predictable, controllable load-balancer implementation that aligns well with the platform discipline established earlier.

---

## 3. Why MetalLB

MetalLB is a natural fit for this environment because it:

- Operates entirely inside the cluster
- Does not require proprietary hardware or cloud integration
- Makes IP allocation explicit and auditable
- Works cleanly with static addressing and VIP-based control planes

Most importantly, MetalLB does not introduce hidden state. All behaviour is declared, versioned, and recoverable, which aligns directly with the promotion path defined in Part 2.

---

## 4. Scope and non-goals

This part will cover:

- Selecting an address pool and defining ownership
- Installing MetalLB in a controlled manner
- Exposing a test service via a LoadBalancer
- Understanding failure modes and recovery considerations

It will not yet cover ingress controllers or TLS. Those concerns build on top of load balancing and are introduced separately to avoid conflating responsibilities.

---

## 5. Address planning and ownership

Before installing MetalLB, you must decide which IP addresses the cluster is allowed to claim.

This is not a technical detail. It is a governance decision.

### 5.1 Selecting an address pool

The address pool must meet the following criteria:

- Within the same L2 network as the cluster nodes
- Outside of DHCP allocation ranges
- Not already in use by infrastructure or services

Eample:

```
Network: 192.168.88.0/24
DHCP range: 192.168.88.100 – 192.168.88.200

MetalLB pool: 192.168.88.60 – 192.168.88.69
```

This pool is small by design. Expanding it later is trivial; reclaiming addresses that have leaked into use is not.

## 5.2 Ownership boundaries

By defining an explicit pool, you are asserting:

- These IPs are owned by the cluster
- The cluster may assign them dynamically
- No other system should attempt to use them

Document this decision alongside your IP plan from Part 1.

---

## 6. Installing MetalLB

MetalLB is installed as a standard Kubernetes workload. No node-level configuration is required.

This guide uses the official manifests to keep the installation transparent and easy to audit.

### 6.1 Install MetalLB components

Apply the MetalLB manifests:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
```

Wait for all MetalLB pods to become ready:

```bash
kubectl -n metallb-system get pods
```

Do not proceed until all components are healthy.

Output:

```text
NAME                         READY   STATUS    RESTARTS   AGE
controller-c5c5d4c5d-9rzmt   1/1     Running   0          61s
speaker-497mm                1/1     Running   0          61s
speaker-8kl4g                1/1     Running   0          61s
speaker-chlgv                1/1     Running   0          61s
speaker-kzn85                1/1     Running   0          61s
speaker-l5q4x                1/1     Running   0          61s
speaker-mgnmn                1/1     Running   0          61s
```

---

## 7. Configuring the address pool

With MetalLB installed, define the address pool and advertisement behaviour.

MetalLB configuration is managed as Kubernetes manifests and stored under manifests/metallb/, separate from Talos configuration and encrypted artefacts.

Example full path

```text
talos-kube/
└── manifests/
    └── metallb/
        ├── ipaddresspool.yaml
        └── l2advertisement.yaml
```

### 7.1 IPAddressPool

Create an IP address pool:

```bash
mkdir -p manifests/metallb/
nano manifests/metallb/ipaddresspool.yaml
```

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.88.60-192.168.88.69
```

Apply it:

```bash
kubectl apply -f manifests/metallb/ipaddresspool.yaml
```

### 7.2 L2Advertisement

In a simple homelab network, Layer 2 advertisement is sufficient:


```bash
nano manifests/metallb/l2advertisement.yaml
```

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
    - default-pool
```

Apply it:

```bash
kubectl apply -f manifests/metallb/l2advertisement.yaml
```

At this point, MetalLB is capable of assigning external IPs.

---

## 8. Validating load balancing behaviour

Before exposing real workloads, validate behaviour using a simple test service.

Example full path

```text
talos-kube/
└── manifests/
    └── testing/
        ├── hello.yaml
        └── hello-service.yaml
```

### 8.1 Deploy a test workload

```bash
mkdir -p manifests/testing/
nano manifests/testing/hello-nginx.yaml
```

`manifests/testing/hello-nginx.yaml`

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: hello
          image: gcr.io/google-samples/hello-app:2.0
          ports:
            - containerPort: 8080
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            runAsNonRoot: true
```

```bash
nano manifests/testing/hello-service.yaml
```

`manifests/testing/hello-service.yaml`

```yml
apiVersion: v1
kind: Service
metadata:
  name: hello
  namespace: default
spec:
  selector:
    app: hello
  type: LoadBalancer
  ports:
    - name: http
      port: 80
      targetPort: 80
```



Apply and expose:

```bash
kubectl delete svc hello
kubectl apply -f manifests/testing/hello-service.yaml
```

### 8.2 Verify IP allocation

```bash
kubectl get svc hello
```

You should see an external IP assigned from the MetalLB pool.

Test connectivity from your network:

```bash
curl http://<external-ip>
```

---

## 9. Failure considerations

Introducing load balancing introduces new failure modes.

Common scenarios include:

- Node hosting the active speaker going offline
- ARP cache staleness on upstream devices
- IP conflicts due to overlapping DHCP ranges

MetalLB handles failover automatically, but recovery depends on correct address planning and clean network boundaries.

If failures occur, first validate:

- IP pool configuration
- Node health
- Network reachability at layer 2

---

## 10. Summary

This part introduced load balancing as the first form of shared, externally visible state in the platform.

By using MetalLB, the cluster gains a predictable and controllable mechanism for exposing services without relying on cloud-specific abstractions. IP ownership is explicit, allocation is deliberate, and behaviour is declared rather than inferred. These properties are essential once workloads are no longer isolated from the network.

Just as importantly, load balancing is treated as infrastructure, not convenience. Address pools, manifests, and validation steps are all managed as versioned intent, consistent with the repository and promotion model established earlier in the series.

With this foundation in place, the cluster is ready to support higher-level traffic management. The next part builds on this by introducing ingress controllers, where routing, hostnames, and TLS further expand the risk surface and reinforce the need for the discipline established so far.
