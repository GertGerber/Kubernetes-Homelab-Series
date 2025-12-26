# Kubernetes Homelab Series Part 4 – Ingress Controllers and Traffic Routing <!-- omit from toc -->

---

## Table of Contents <!-- omit from toc -->
<!-- TOC -->
- [References](#references)
- [1. Introduction](#1-introduction)
- [2. Why ingress is not just layer 7 load balancing](#2-why-ingress-is-not-just-layer-7-load-balancing)
- [3. Scope and non-goals](#3-scope-and-non-goals)
- [4. Selecting an ingress controller](#4-selecting-an-ingress-controller)
- [5. Ingress as shared infrastructure](#5-ingress-as-shared-infrastructure)
- [6. Installing the ingress controller](#6-installing-the-ingress-controller)
  - [6.1 Repository location](#61-repository-location)
  - [6.2 Install Helm](#62-install-helm)
  - [6.3 Prerequisites](#63-prerequisites)
- [7. Installing Traefik](#7-installing-traefik)
  - [7.1 Helm values as intent](#71-helm-values-as-intent)
  - [7.2 Installing Traefik via Helm](#72-installing-traefik-via-helm)
  - [7.3 Observations and expectations](#73-observations-and-expectations)
- [8. Validating ingress behaviour](#8-validating-ingress-behaviour)
  - [8.1 Prerequisites](#81-prerequisites)
  - [8.2 Deploy a restricted-compatible test application](#82-deploy-a-restricted-compatible-test-application)
  - [8.3 Create a ClusterIP service for the application](#83-create-a-clusterip-service-for-the-application)
  - [8.4 Create an Ingress resource](#84-create-an-ingress-resource)
  - [8.5 Test routing without DNS changes](#85-test-routing-without-dns-changes)
  - [8.6 Optional: Add DNS for convenience](#86-optional-add-dns-for-convenience)
  - [8.7 Basic failure checks](#87-basic-failure-checks)
- [9. Failure considerations](#9-failure-considerations)
- [10. Summary](#10-summary)
- [Appendix A – Platform operator notes for ingress](#appendix-a--platform-operator-notes-for-ingress)
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

With external load balancing in place from Part 3, the cluster can now receive traffic from the network in a predictable way.

Exposing every workload directly via a Service of type LoadBalancer does not scale. Each service consumes an IP address, routing logic is fragmented, and operational consistency degrades as the platform grows.

Ingress controllers address this by centralising HTTP routing inside the cluster. They introduce a shared control plane that translates hostnames and paths into service-level routing decisions.

This part introduces ingress deliberately, building on the foundations established in Parts 1 to 3 and treating ingress as platform infrastructure rather than application configuration.

---

## 2. Why ingress is not just layer 7 load balancing

Ingress is often described as layer 7 load balancing. While technically accurate, this framing understates its operational impact.

Ingress introduces shared routing state across workloads, name and path-based traffic decisions, and a single failure domain that can affect many applications simultaneously.

Unlike a LoadBalancer Service, ingress configuration is tightly coupled to application identity and naming. Errors are immediately visible and harder to isolate.

For this reason, ingress requires stronger discipline around ownership, versioning, and change control than application-level resources.

---

## 3. Scope and non-goals

This part covers selecting an ingress controller suitable for a homelab, installing it declaratively, exposing a simple application, and validating routing behaviour.

It does not cover TLS, certificate management, advanced routing, or authentication. Those concerns are intentionally deferred to later parts to avoid conflating responsibilities.

---

## 4. Selecting an ingress controller

There are several mature ingress controllers available. The most common choices include NGINX, Traefik, and HAProxy.

Traefik is used in this series because it integrates cleanly with Kubernetes primitives, has sensible defaults for smaller environments, and allows incremental adoption of more advanced features.

The specific controller matters less than how it is operated. The patterns in this guide apply regardless of implementation.

---

## 5. Ingress as shared infrastructure

An ingress controller is shared platform infrastructure, not an application component.

Configuration changes affect multiple workloads, and failures have a wider blast radius than typical deployments. As with MetalLB and secrets management, ingress should be deployed and upgraded deliberately with declarative intent.

---

## 6. Installing the ingress controller

Ingress controllers are installed as Kubernetes workloads but must be treated as **platform components**.

Configuration must be versioned, installation repeatable, and changes auditable. Helm is used to manage this lifecycle.

### 6.1 Repository location

Ingress manifests should live alongside other cluster-level intent:

```text
talos-kube/
└── manifests/
    └── ingress/
```

```bash
mkdir -p manifests/ingress/
```

This mirrors the structure used for MetalLB and reinforces the separation between platform intent and application workloads.

### 6.2 Install Helm

Helm is used to install and manage the ingress controller in a repeatable way. 

Install Helm on the workstation VM.

Ubuntu or Debian:

```bash
sudo apt-get update
sudo apt-get install -y snapd
sudo snap install helm --classic
helm version
```

macOS (if you ever run the workstation tooling locally)

```bash
brew install helm
helm version
```

div style="
  border-left: 4px solid #6f42c1;
  background: rgba(111, 66, 193, 0.12);
  padding: 0.75em 1em;
  border-radius: 10px;
  margin: 1em 0;
">
  <strong>📝 Note</strong><br>
  Helm is part of your platform toolchain. Treat it like kubectl and talosctl: keep it updated deliberately and prefer pinned behaviour over surprise upgrades.
</div>

### 6.3 Prerequisites

Before proceeding, confirm MetalLB is installed, at least one LoadBalancer service has an external IP, kubectl access works, and Helm is available.

---

## 7. Installing Traefik

Traefik is installed using the official Helm chart with all configuration captured in a committed values file.

### 7.1 Helm values as intent

Create a values file at:

```text
talos-kube/
└── manifests/
    └── ingress/
        └── traefik-values.yaml
```

```bash
nano manifests/ingress/traefik-values.yaml
```

If your cluster enforces Pod Security Standards (restricted), include the securityContext and podSecurityContext settings below to avoid admission warnings and ensure consistent behaviour across environments.

`manifests/ingress/traefik-values.yaml`

```yml
deployment:
  replicas: 2

service:
  type: LoadBalancer

ports:
  web:
    port: 80
    expose:
      default: true
    exposedPort: 80

providers:
  kubernetesIngress:
    enabled: true

ingressClass:
  enabled: true
  isDefaultClass: true

logs:
  general:
    level: INFO

# Enforce restricted-compatible defaults
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true

podSecurityContext:
  seccompProfile:
    type: RuntimeDefault
```

This configuration deliberately keeps Traefik minimal:

- HTTP only
- No TLS
- No dashboard exposure
- No advanced routing features

Those are introduced later, once the basics are stable.

### 7.2 Installing Traefik via Helm

Add the Traefik Helm repository:

```bash
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
```

Install Traefik using the committed values file:

```bash
helm install traefik traefik/traefik \
  --namespace ingress-system \
  --create-namespace \
  --values manifests/ingress/traefik-values.yaml
```

You should see an external IP allocated from the MetalLB pool.

This IP becomes the single entry point for HTTP traffic into the cluster.

**If changes were made to manifests/ingress/traefik-values.yaml**

```bash
helm upgrade traefik traefik/traefik \
  --namespace ingress-system \
  --values manifests/ingress/traefik-values.yaml
```

Then confirm pods restart cleanly:

```bash
kubectl -n ingress-system get pods
kubectl -n ingress-system describe pod -l app.kubernetes.io/name=traefik | sed -n '1,120p'
```

### 7.3 Observations and expectations

At this stage Traefik is running, has an external IP via MetalLB, but no applications are exposed. TLS is intentionally not configured yet.

---

## 8. Validating ingress behaviour

At this point Traefik is running, reachable, and has an external IP via MetalLB. Now we validate that ingress routing works end to end before introducing TLS or more complex routing.

This validation focuses on three things:

- The ingress controller can receive traffic from the network
- Hostname-based routing behaves predictably
- Failures are observable and recoverable

### 8.1 Prerequisites

Confirm Traefik has an external IP:

```bash
kubectl -n ingress-system get svc traefik
```

### 8.2 Deploy a restricted-compatible test application

Create a test deployment as declarative intent:

Filename and directory
`manifests/testing/hello-app.yaml`

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello1
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello1
  template:
    metadata:
      labels:
        app: hello1
    spec:
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: hello1
          image: gcr.io/google-samples/hello-app:2.0
          ports:
            - containerPort: 8080
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            runAsNonRoot: true
            runAsUser: 1000
```

Apply it:

```BASH
kubectl apply -f manifests/testing/hello-app.yaml
kubectl get pods -l app=hello1
```

### 8.3 Create a ClusterIP service for the application

Ingress routes to a Service, not directly to a Pod. Create a service manifest:

Filename and directory
`manifests/testing/hello-app-service.yaml`

```yml
apiVersion: v1
kind: Service
metadata:
  name: hello1
spec:
  selector:
    app: hello1
  ports:
    - port: 80
      targetPort: 8080
```

Apply it:

```bash
kubectl apply -f manifests/testing/hello-app-service.yaml
kubectl get svc hello1
```

### 8.4 Create an Ingress resource

Create an Ingress that routes a hostname to the test service:

Filename and directory
`manifests/testing/hello-app-ingress.yaml`

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello1
spec:
  ingressClassName: traefik
  rules:
    - host: hello1.home.arpa
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello1
                port:
                  number: 80
```

Apply it:

```bash
kubectl apply -f manifests/testing/hello-app-ingress.yaml
kubectl get ingress hello1
```

### 8.5 Test routing without DNS changes

Before touching your DNS, validate routing using an explicit Host header.

From your workstation:

```bash
curl -H "Host: hello.home.arpa" http://<TRAEFIK_LB_IP>/
```

Expected behaviour:

You receive a response from the hello application

The request is routed through Traefik to the Service, then to the Pod

If this works, ingress routing is functional.

### 8.6 Optional: Add DNS for convenience

If you want this to work without specifying a Host header, create a DNS record in your local resolver:

```text
hello.home.arpa  ->  <TRAEFIK_LB_IP>
```

Then test normally

```bash
curl http://hello.home.arpa/
```

This is optional. DNS integration becomes more important once you introduce TLS and real applications.

### 8.7 Basic failure checks

If routing fails, these commands narrow the cause quickly:

Check the ingress resource:

```bash
kubectl describe ingress hello
```

Check Traefik logs:

```bash
kubectl describe ingress hello1
kubectl -n ingress-system logs -l app.kubernetes.io/name=traefik --tail=200
```

The most common issues are:

- Service selector does not match pod labels
- Ingress class mismatch
- Traefik service has no external IP

---

## 9. Failure considerations

Ingress consolidates traffic behind a single entry point. Misconfiguration can affect multiple workloads simultaneously. Multiple replicas mitigate pod failure but not configuration errors. This is why ingress changes should be deliberate and reviewed.

---

## 10. Summary

Ingress adds flexibility but concentrates risk. By treating ingress controllers as shared infrastructure and managing them declaratively, the platform gains consistent routing without sacrificing control.

---

## Appendix A – Platform operator notes for ingress

- Prefer one ingress controller per cluster
- Version-control all Helm values
- Treat ingress changes as platform changes, not app changes
- Test routing with Host headers before DNS updates
- Expect configuration errors to have immediate impact
- Avoid enabling dashboards or admin endpoints by default

Ingress is a force multiplier. Operated carefully, it simplifies the platform. Operated casually, it amplifies mistakes.
