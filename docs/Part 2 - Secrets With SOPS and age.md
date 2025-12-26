# Kubernetes Homelab Series Part 2 - Secrets With SOPS and age <!-- omit from toc -->

---

## Table of Contents <!-- omit from toc -->
<!-- TOC -->
- [References](#references)
- [1. What Is SOPS and age?](#1-what-is-sops-and-age)
- [2. Installation and Setup](#2-installation-and-setup)
- [2.1 Installation](#21-installation)
- [2.2 Setup](#22-setup)
- [3. Repository Layout](#3-repository-layout)
  - [3.1 High-level structure](#31-high-level-structure)
  - [3.2 Directory intent](#32-directory-intent)
  - [3.3. Operational model](#33-operational-model)
  - [3.4. Forward compatibility](#34-forward-compatibility)
- [5. Promotion Path](#5-promotion-path)
  - [5.1 Stages of promotion](#51-stages-of-promotion)
    - [5.1.1. Intent (authoritative)](#511-intent-authoritative)
    - [5.1.2. Rendered output (transient)](#512-rendered-output-transient)
    - [5.1.3. Encrypted artefact (durable)](#513-encrypted-artefact-durable)
    - [5.1.4. Versioned state (source of truth)](#514-versioned-state-source-of-truth)
  - [5.2 Operational discipline](#52-operational-discipline)
  - [5.3 Looking ahead](#53-looking-ahead)
- [6. Key Ownership and Recovery Expectations](#6-key-ownership-and-recovery-expectations)
- [7. Encrypting and Decrypting Files](#7-encrypting-and-decrypting-files)
  - [7.1 Encrypting rendered outputs](#71-encrypting-rendered-outputs)
  - [7.2 Decrypting for change](#72-decrypting-for-change)
  - [7.3 Editing discipline](#73-editing-discipline)
  - [7.4 Relationship to GitOps](#74-relationship-to-gitops)
- [8. What About Sealed Secrets?](#8-what-about-sealed-secrets)
- [9. Failure and Recovery Scenarios](#9-failure-and-recovery-scenarios)
  - [9.1 Loss of the age private key](#91-loss-of-the-age-private-key)
    - [Impact](#impact)
    - [Recovery path](#recovery-path)
    - [Mitigation](#mitigation)
  - [9.2 Loss of the git repository](#92-loss-of-the-git-repository)
    - [Impact](#impact-1)
    - [Recovery path](#recovery-path-1)
    - [Mitigation](#mitigation-1)
  - [9.3 Workstation rebuild or loss](#93-workstation-rebuild-or-loss)
    - [Impact](#impact-2)
    - [Mitigation](#mitigation-2)
- [10. Closing note](#10-closing-note)
- [11. Summary](#11-summary)
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

## 1. What Is SOPS and age?

[SOPS](https://github.com/getsops/sops}
SOPS (Secrets OPerationS) is a tool used to encrypt values in structured data formats such as YAML and JSON. Rather than encrypting an entire file, SOPS encrypts only selected values, leaving the overall file structure intact. This makes it well suited to storing configuration files safely in source control, including public repositories.

[https://github.com/getsops/sops](https://github.com/getsops/sops)

age (pronounced “ah-gay”) is a modern file encryption tool with a deliberately simple design. When used with SOPS, age provides the cryptography while SOPS handles selective encryption of YAML values. SOPS supports several backends such as AWS KMS and GCP KMS, but for a homelab environment age is an excellent fit as it is cloud-agnostic and easy to reason about.

[AGE](https://github.com/FiloSottile/age)

PGP is another common option for homelab users, but age offers a simpler mental model and modern cryptographic defaults, which reduces operational friction.

---

## 2. Installation and Setup

This section walks through installing SOPS and age, then using them to encrypt Talos Linux `secrets.yaml` and machine configuration files so they can be safely committed to a Git repository.

If you prefer a video walkthrough, this talk provides a good overview of SOPS concepts and workflows:
[https://www.youtube.com/watch?v=V2PRhxphH2w](https://www.youtube.com/watch?v=V2PRhxphH2w)

There is also a VS Code extension that transparently encrypts and decrypts files on save, which can be useful once you are comfortable with the basics:
[https://marketplace.visualstudio.com/items?itemName=signageos.signageos-vscode-sops](https://marketplace.visualstudio.com/items?itemName=signageos.signageos-vscode-sops)

For clarity and auditability, this guide sticks to running SOPS commands manually. These steps can later be automated or embedded into a CI pipeline if required.

---

## 2.1 Installation

From this point on, rendered Talos configuration files are treated as transient build outputs. Only encrypted artefacts are committed to version control.

The following examples assume Linux on amd64. macOS users may prefer installing both tools via Homebrew. Adjust accordingly for other platforms.

```bash
# Set versions (update periodically)
SOPS_VERSION="v3.11.0"
AGE_VERSION="v1.2.1"

# Install SOPS (Linux amd64)
curl -sSLo sops "https://github.com/getsops/sops/releases/download/${SOPS_VERSION}/sops-${SOPS_VERSION}.linux.amd64"
sudo install -m 0755 sops /usr/local/bin/sops
sops --version --check-for-updates

# Install age (Linux amd64)
curl -sSLo age.tar.gz "https://github.com/FiloSottile/age/releases/download/${AGE_VERSION}/age-${AGE_VERSION}-linux-amd64.tar.gz"
tar -xzf age.tar.gz
sudo install -m 0755 age/age /usr/local/bin/age
sudo install -m 0755 age/age-keygen /usr/local/bin/age-keygen
age --version
age-keygen --version
```

For environments with a higher trust bar, both projects publish checksums and signatures that can be verified before installation.

---

## 2.2 Setup

Create an age key pair. This is similar in concept to generating an SSH key.

```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
```

Store the private key securely, for example in a password manager such as Bitwarden. This key is the root of trust for decrypting your secrets.

The public key is included in the same file and will be used for encryption.

Next, create a `.sops.yaml` file in the root of your repository. This file defines which files SOPS should encrypt, which fields within those files should be encrypted, and which keys to use.

```yaml
---
creation_rules:
  - path_regex: '(^|.*/)secrets(\.encrypted)?\.ya?ml$'
    age: age1replace-with-your-public-key

  - path_regex: '(^|.*/)(controlplane|worker)(\.encrypted)?\.ya?ml$'
    encrypted_regex: '(^token|crt|key|id|secret|secretboxEncryptionSecret)$'
    age: age1replace-with-your-public-key
```

<div style="
  border-left: 4px solid #cf222e;
  background: rgba(207, 34, 46, 0.12);
  padding: 0.75em 1em;
  border-radius: 10px;
  margin: 1em 0;
">
  <strong>❌ Warning</strong><br>
  Replace "age1replace-with-your-public-key" with your key generated in previous step.
</div>


This configuration encrypts all values in `secrets.yaml` and selectively encrypts sensitive fields in Talos machine configuration files.

Create a `.gitignore` file to ensure plaintext secrets are never committed. Adjust paths to match your own Talos workflow.

```gitignore
secrets.yaml
talosconfig
controlplane.yaml
worker.yaml
kubeconfig
```

Encrypt your files before committing them:

```bash
sops --encrypt secrets.yaml > secrets.encrypted.yaml
sops --encrypt _out/controlplane.yaml > _out/controlplane.encrypted.yaml
sops --encrypt _out/worker.yaml > _out/worker.encrypted.yaml
```

Initialise and commit the repository, ensuring only encrypted files are staged.

With encryption keys and rules in place, the next step is to define how configuration, build outputs, and encrypted artefacts are organised on disk. This structure underpins the workflows used throughout the remainder of the series.

---

## 3. Repository Layout

This repository is structured to separate **intent**, **build outputs**, and **sensitive material**, while supporting a Git-driven operating model.

The guiding principle is simple:
only **encrypted artefacts and declarative intent** are committed to version control. Plaintext outputs are treated as transient and disposable.

### 3.1 High-level structure

```text
talos-kube/
├── patches/                    # Declarative intent (per-node and shared patches)
│   ├── cp1.yaml
│   ├── cp2.yaml
│   ├── cp3.yaml
│   ├── w1.yaml
│   ├── w2.yaml
│   ├── w3.yaml
│   └── resolvers.yaml
│
├── rendered/                   # Plaintext build outputs (not committed)
│   ├── talos-kube-cp1.yaml
│   ├── talos-kube-cp2.yaml
│   ├── talos-kube-cp3.yaml
│   ├── talos-kube-w1.yaml
│   ├── talos-kube-w2.yaml
│   └── talos-kube-w3.yaml
│
├── encrypted/                  # SOPS-encrypted artefacts (committed)
│   ├── secrets.encrypted.yaml
│   ├── controlplane.encrypted.yaml
│   └── worker.encrypted.yaml
│
├── .sops.yaml                  # Encryption rules and key configuration
├── .gitignore
└── README.md
```

### 3.2 Directory intent

**patches/**
Contains the authoritative, human-maintained declaration of node identity and behaviour. These files are small, reviewable, and safe to commit. They represent intent rather than output.

**rendered/**
Contains fully assembled Talos machine configurations produced by combining base configs with patches. These files are plaintext by design and are treated as ephemeral build artefacts. They must never be committed.

**encrypted/**
Contains SOPS-encrypted artefacts that are safe to store in version control. These represent the durable state of the platform and form the basis for future GitOps workflows.

### 3.3. Operational model

The expected workflow is:

1. Modify intent in patches/
2. Render plaintext configs locally
3. Encrypt outputs using SOPS
4. Commit only encrypted artefacts

Plaintext files may be regenerated at any time and should be assumed disposable. Loss of the repository does not compromise secrets, and loss of a workstation does not prevent recovery, provided encrypted artefacts and age keys are preserved.

### 3.4. Forward compatibility

This layout deliberately mirrors how GitOps controllers operate:

- Declarative intent lives in git
- Sensitive material remains encrypted at rest
- Build and render steps can be automated later

When a GitOps controller such as Flux or Argo CD is introduced, this structure requires minimal change. Decryption simply moves from a human-driven step to a controlled reconcile process.

---

## 5. Promotion Path

The promotion path describes how changes move from **local intent** to **durable, versioned state**. Defining this explicitly avoids ambiguity around what is authoritative, what is disposable, and what represents the source of truth for the platform.

This is intentionally a lightweight process. The goal is consistency and recoverability, not ceremony.

### 5.1 Stages of promotion

Changes flow through the following stages:

1. Intent
2. Rendered output
3. Encrypted artefact
4. Versioned state

Each stage has a clear purpose and trust boundary.

#### 5.1.1. Intent (authoritative)

Intent lives in small, human-maintained files such as:

- Node patches
- Shared configuration overlays
- Repository-level policy files

These files are:

- Plaintext
- Reviewable
- Safe to commit
- Treated as the primary expression of desired state

Changes should always begin here.

#### 5.1.2. Rendered output (transient)

Rendered output is produced locally by combining intent with tooling such as talosctl.

Examples include:

- Fully assembled Talos machine configurations
- Generated bootstrap material prior to encryption

Rendered files are:

- Plaintext
- Ephemeral
- Never committed
- Regenerable at any time

They exist only long enough to validate correctness and enable the next promotion step.

#### 5.1.3. Encrypted artefact (durable)

Rendered outputs that contain sensitive material are encrypted using SOPS and promoted into encrypted artefacts.

At this stage:

- Secrets are protected at rest
- Files are safe to commit
- Recovery no longer depends on a specific workstation

Encrypted artefacts represent the durable state of the platform.

#### 5.1.4. Versioned state (source of truth)

Once committed, encrypted artefacts become the system of record.

From this point:

- Git history provides auditability
- Rollback is explicit and controlled
- Recovery is deterministic, assuming key availability

The repository, not the live cluster, is treated as the source of truth.

### 5.2 Operational discipline

A few rules keep this promotion path effective:

- Plaintext artefacts must never be committed, even temporarily
- Encrypted artefacts should only be modified via decryption, edit, and re-encryption
- Keys are managed separately from the repository and backed up deliberately

This discipline ensures the platform remains operable under failure conditions, including workstation loss or rebuild.

### 5.3 Looking ahead

In later parts of the series, this promotion path becomes automated.

GitOps controllers such as Flux or Argo CD simply formalise what is already happening manually: reconciling a declared, versioned state into the running cluster. By establishing the promotion path early, the transition to automation requires minimal conceptual change.

---

## 6. Key Ownership and Recovery Expectations

The age private key generated in this section represents the root of trust for all encrypted material protected by SOPS in this platform.

In a single-operator homelab, this key will typically be owned by the individual responsible for cluster lifecycle and recovery. In a shared or team environment, ownership should be explicit and agreed upfront.

At a minimum:

- The private key must be backed up outside the cluster, ideally in a password manager or offline secure storage.
- Loss of the private key means permanent loss of access to all encrypted Talos secrets and configuration material.
- Regenerating keys is possible, but requires decrypting and re-encrypting all protected files, which assumes the original key is still available.
- As with backups, recovery should be tested periodically to ensure keys and encrypted artefacts can be restored together.
- GitOps automation does not remove the need for key ownership; it simply shifts where decryption occurs.

Treat this key in the same category as cluster root credentials. It should be protected accordingly and tested periodically as part of backup and recovery drills.

---

## 7. Encrypting and Decrypting Files

This section operationalises the promotion path defined earlier. The commands themselves are simple; the discipline around when and why they are used is what matters.

The goal is to promote plaintext rendered output into encrypted, versioned artefacts without ever committing sensitive material.

### 7.1 Encrypting rendered outputs

Encryption is always performed on **rendered plaintext files**, producing **encrypted artefacts** suitable for version control.

Example workflow:

```bash
# Encrypt Talos secrets
sops --encrypt secrets.yaml > encrypted/secrets.encrypted.yaml

# Encrypt control plane configuration
sops --encrypt rendered/controlplane.yaml > encrypted/controlplane.encrypted.yaml

# Encrypt worker configuration
sops --encrypt rendered/worker.yaml > encrypted/worker.encrypted.yaml
```

At this point:

- Plaintext files remain local and disposable
- Encrypted artefacts are safe to commit
- The repository now contains the durable state of the platform

Only files in the encrypted/ directory should ever be staged in git.

### 7.2 Decrypting for change

When changes are required, the process is reversed locally.

Encrypted artefacts are decrypted only long enough to apply a modification, then immediately re-encrypted.

```bash
sops --decrypt encrypted/secrets.encrypted.yaml > secrets.yaml
```

After editing:

```bash
sops --encrypt secrets.yaml > encrypted/secrets.encrypted.yaml
rm secrets.yaml
```

Plaintext files should be removed once promotion is complete.

This ensures the working directory returns to a safe baseline state after each change.

### 7.3 Editing discipline

A few guardrails keep this workflow safe and predictable:

- Decrypted files exist only on trusted workstations
- Decrypted files are never committed
- Encryption happens immediately after changes are made
- Git history reflects intent, not secrets

This discipline scales cleanly from a single-operator homelab to a shared environment without changing the underlying model.

### 7.4 Relationship to GitOps

This manual encrypt–commit workflow mirrors how GitOps systems operate:

- Desired state lives in git
- Secrets remain encrypted at rest
- Reconciliation is repeatable and auditable

Later in the series, tooling will automate these steps, but the trust boundaries and promotion rules remain unchanged.

---

## 8. What About Sealed Secrets?

This section clarifies what Sealed Secrets are useful for, and just as importantly, what they are not intended to replace.

Sealed Secrets are a Kubernetes-specific mechanism for encrypting Secret resources. They are not a replacement for SOPS, as they cannot be used outside the cluster. Talos `secrets.yaml` is a good example of something Sealed Secrets cannot protect.

Sealed Secrets work well for application-level Kubernetes secrets. They encrypt data using a controller running in the cluster and generate standard Secret resources at runtime. Access to those Secrets still needs to be controlled carefully.

Many environments use both approaches:

* SOPS and age for cluster bootstrap, Talos configuration, and GitOps-managed YAML
* Sealed Secrets for in-cluster application secrets

As this homelab evolves, a natural next step is integrating SOPS with a GitOps controller such as Flux or Argo CD, where decryption happens automatically at reconcile time. This reduces manual handling of secrets while keeping them encrypted at rest in git.

---

## 9. Failure and Recovery Scenarios

This section outlines expected failure scenarios and their recovery implications. The intent is not to exhaustively document disaster recovery, but to make assumptions explicit and ensure the platform can be recovered deliberately rather than accidentally.

The scenarios below assume that the promotion path and repository discipline described earlier are followed.

### 9.1 Loss of the age private key

#### Impact
Loss of the age private key results in permanent loss of access to all SOPS-encrypted artefacts. Encrypted Talos secrets and configuration files cannot be recovered or regenerated without the original key.

#### Recovery path

- There is no technical recovery without the key.
- A new key pair may be generated, but only if decrypted material is still available elsewhere.
- All secrets and encrypted artefacts must then be re-encrypted using the new key.

#### Mitigation

- Back up the private key outside the cluster and repository.
- Store it in secure, durable storage.
- Periodically test recovery alongside encrypted artefact restores.

### 9.2 Loss of the git repository

#### Impact

Loss of the repository does not compromise secrets, but it does remove the system of record for platform configuration and history.

#### Recovery path

- Restore the repository from backup or recreate it from encrypted artefacts.
- Decrypt encrypted artefacts using the age key.
- Re-render and reapply Talos configuration as required.

#### Mitigation

- Back up the repository independently of the cluster.
- Treat git history as an operational asset, not just documentation.
- Avoid making changes directly on live systems without committing intent.

### 9.3 Workstation rebuild or loss

#### Impact

Loss of the administrative workstation removes local tooling, plaintext artefacts, and working state, but does not affect the cluster itself.

####b Recovery path

- Rebuild the workstation.
- Reinstall tooling.
- Restore the git repository and age private key.
- Decrypt encrypted artefacts and continue operations.

#### Mitigation

- Keep the workstation stateless by design.
- Ensure all durable state lives in encrypted artefacts and git.
- Avoid storing unique or unrecoverable information locally.

## 10. Closing note

These scenarios reinforce a central theme of this series: **recovery is a design property, not an afterthought**

By separating intent, build outputs, and encrypted state, the platform remains operable even when individual components fail. This discipline enables the next stages of the series, where shared state and automation increase the consequences of error but also the benefits of consistency.

---

## 11. Summary

This part establishes a disciplined approach to secrets management that treats recovery, auditability, and long-term operation as first-class concerns.

By introducing SOPS and age alongside a clear repository structure and promotion path, secrets are no longer ad-hoc files tied to a single workstation or operator. Instead, they become managed artefacts that can be versioned, reviewed, and recovered deliberately.

Key ownership and failure scenarios are made explicit to avoid false assumptions about safety or reversibility. Losses are survivable when boundaries are respected, and irreversible when they are not. This clarity is intentional.

With these foundations in place, the platform is now ready to introduce shared state and external access. The next parts of the series build on this by adding networking, ingress, and certificates, while continuing to rely on the same principles of declarative intent, controlled promotion, and recoverable operations.

The tooling may evolve. The discipline should not.