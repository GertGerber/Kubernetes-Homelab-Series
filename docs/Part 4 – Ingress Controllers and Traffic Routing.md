# Kubernetes Homelab Series Part 4 – Ingress Controllers and Traffic Routing <!-- omit from toc -->

---

## Table of Contents <!-- omit from toc -->
<!-- TOC -->
- [References](#references)
- [1. Introduction](#1-introduction)
- [2. Why ingress is not “just layer 7 load balancing”](#2-why-ingress-is-not-just-layer-7-load-balancing)
- [3. Scope and non-goals](#3-scope-and-non-goals)
- [4. Selecting an ingress controller](#4-selecting-an-ingress-controller)
- [5. Ingress as shared infrastructure](#5-ingress-as-shared-infrastructure)
- [6. Installing the ingress controller](#6-installing-the-ingress-controller)
  - [6.1 Repository location](#61-repository-location)
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

With load balancing in place, the cluster can now expose services to the network in a predictable and controlled way.

However, exposing individual services directly via LoadBalancer does not scale well. Each service consumes an IP address, routing logic lives outside the cluster, and operational consistency becomes harder to maintain as the number of workloads grows.

Ingress controllers address this by centralising traffic routing decisions inside the cluster. They introduce a new layer of abstraction, and with it, new responsibilities and risks.

This part introduces ingress deliberately, building on the load-balancing foundation established in Part 3.

---

## 2. Why ingress is not “just layer 7 load balancing”

It is tempting to think of ingress as simply a more advanced form of load balancing. In practice, it is a different class of problem.

Ingress introduces:

- Shared routing state across workloads
- Name-based and path-based traffic decisions
- A single failure domain for multiple applications
- A control plane that directly affects user traffic

Unlike a Service of type LoadBalancer, ingress resources are tightly coupled to application behaviour and naming. Mistakes here are immediately visible and often harder to roll back.

Because of this, ingress needs stronger discipline around ownership, versioning, and change control.

---

## 3. Scope and non-goals

This part covers:

- Selecting an ingress controller suitable for a homelab
- Installing the controller in a controlled manner
- Exposing a simple application via ingress
- Understanding basic failure and recovery considerations

It does not cover:

- TLS or certificate management
- Advanced routing rules
- Authentication or authorisation at the edge

Those concerns are introduced later to avoid conflating responsibilities.

---

## 4. Selecting an ingress controller

There are several mature ingress controllers available. The most common choices include NGINX, Traefik, and HAProxy.

For this series, Traefik is used because it:

- Integrates cleanly with Kubernetes primitives
- Has sensible defaults for smaller environments
- Requires minimal boilerplate to become productive
- Supports gradual adoption of more advanced features

The choice of controller is less important than how it is operated. The patterns described here apply regardless of implementation.

---

## 5. Ingress as shared infrastructure

An ingress controller is not an application component. It is shared infrastructure.

This has several implications:

- It should be deployed and upgraded deliberately
- Configuration changes affect multiple workloads
- Resource isolation and failure domains matter

Treating ingress as “just another deployment” increases the blast radius of mistakes. Instead, it should be managed with the same care as load balancing and secrets.

---

## 6. Installing the ingress controller

Ingress controllers are installed as Kubernetes workloads, but their manifests should be treated as platform intent rather than application code.

### 6.1 Repository location

Ingress manifests should live alongside other cluster-level intent:

```text
manifests/ingress/
```


This keeps them clearly separated from application workloads.