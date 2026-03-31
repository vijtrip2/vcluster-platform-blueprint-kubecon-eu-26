# CLAUDE.md — AI Assistant Guide

This file provides context for AI assistants (Claude, Copilot, etc.) working in this repository.

---

## Repository Purpose

This is a **KubeCon EU 2026 workshop blueprint** for deploying a production-ready Kubernetes platform using [vind](https://www.vcluster.com/docs/vcluster/next/deploying-vclusters/vind) (vCluster-in-Docker) and the `kubara` platform distro. It demonstrates:

- GitOps at scale using Argo CD hub-and-spoke topology
- Platform engineering with Helm umbrella charts
- Security & compliance with Kyverno policies + External Secrets + Bitwarden
- Multi-tenancy with label-based fleet management
- AI/ML workloads with Ollama, NVIDIA GPU Operator, and kagent

There is **no application source code** in this repo — it is entirely YAML, Helm charts, and documentation.

---

## Repository Layout

```
vcluster-platform-blueprint-kubecon-eu-26/
├── README.md                          # Workshop overview & learning objectives
├── 00_PREREQUISITES.md                # Hardware/software requirements
├── 01_BOOTSTRAPPING.md                # Step-by-step setup guide (primary entry point)
├── 02_ADDITIONAL_USE_CASES.md         # Advanced scenarios: GPU, multi-cluster, collab
├── 100_ADDITIONAL_INFORMATION.md      # Troubleshooting & external resources
├── config.yaml                        # Main platform configuration (edit this first)
├── vcluster-config-controlplane.yaml  # vind cluster spec (Kubernetes version, nodes)
├── css-bitwarden.yaml                 # External Secrets / Bitwarden template
├── kubara                             # Pre-built platform bootstrap binary (arm64)
├── z_images/                          # Documentation screenshots
├── managed-service-catalog/helm/      # Base Helm umbrella charts for all platform services
└── customer-service-catalog/helm/controlplane/  # Per-cluster value overrides
```

---

## Key Concepts

### `kubara` Binary
The `kubara` binary is the platform orchestration tool. It is **not rebuilt from source** in this repo — treat it as a pre-built dependency. Workflow:

```bash
./kubara init --prep                                                  # Generate .env template
./kubara init                                                         # Generate config.yaml from .env
./kubara generate --helm                                              # Render Helm charts + overlays
./kubara bootstrap --with-es-css-file css-bitwarden.yaml controlplane --with-es-crds
```

### `config.yaml` — Central Configuration
This is the single file to edit for activating/deactivating platform services:

```yaml
services:
  argocd:
    status: activated       # Currently active
  ollama:
    status: deactivated     # Enable for local LLM workloads
  gpuOperator:
    status: deactivated     # Enable for GPU nodes
  kagent:
    status: deactivated     # Enable for AI agents
```

All service activation/deactivation is controlled here. Do not modify individual Helm chart values unless overriding per-cluster behavior.

### Managed vs. Customer Service Catalog
- **`managed-service-catalog/helm/`** — Upstream/base chart definitions. Treat as authoritative defaults.
- **`customer-service-catalog/helm/controlplane/`** — Per-cluster overrides. These inherit from managed and should be minimal.

Do not duplicate configuration between the two layers — put defaults in managed, put cluster-specific overrides in customer.

---

## Helm Chart Conventions

### Umbrella Chart Pattern
Every service chart wraps an upstream chart with a shared template library:

```yaml
# Chart.yaml
dependencies:
  - name: template-library
    repository: file://../template-library
    version: 0.0.9
  - name: {actual-service}
    repository: https://upstream-repo
    version: {pinned-version}
```

Always pin upstream chart versions — never use `>=` or floating ranges.

### Template Library (`managed-service-catalog/helm/template-library/`)
Provides `templateLibrary.util.merge` for deep-merging YAML values across chart layers. Use this pattern for value inheritance rather than duplicating keys.

### Naming Conventions
| Resource | Pattern | Example |
|---|---|---|
| Cluster name | `{name}-{stage}` | `controlplane-prod` |
| Helm chart dir | lowercase-hyphen | `argo-cd`, `cert-manager` |
| Namespace | service name or generic | `argocd`, `external-secrets` |
| Argo CD label | `argocd.argoproj.io/instance: {project}-{stage}` | `controlplane-prod` |

### Resource Requests/Limits
Use conservative defaults appropriate for local development:
- CPU requests: `10m`–`250m`
- Memory requests: `100Mi`–`500Mi`
- Memory limits: `100Mi`–`1024Mi`
- Avoid setting CPU limits (causes throttling)

---

## External Secrets & Bitwarden Pattern

Secrets are managed via External Secrets Operator backed by Bitwarden Secrets Manager:

```yaml
# ClusterSecretStore (css-bitwarden.yaml)
externalSecrets:
  secrets:
    {secret-name}:
      secretStoreRef: {cluster}-{stage}
      data:
        - secretKey: {k8s-key}
          remoteKey: {bitwarden-key}
```

When adding new secrets:
1. Add the secret to Bitwarden Secrets Manager
2. Reference it via `ExternalSecret` using the existing `ClusterSecretStore`
3. Do not hardcode secrets in values files or templates

---

## Argo CD GitOps Structure

### Projects & ApplicationSets
- Projects defined in `customer-service-catalog/helm/controlplane/argo-cd/values.yaml`
- ApplicationSets use cluster label selectors for fleet-wide deployments
- The **default Argo CD project is blocked** by Kyverno policy — always assign apps to a named project

### Cluster Labels for Fleet Management
Applications deploy to clusters matching labels, not hardcoded cluster names:
```yaml
matchLabels:
  environment: production
  tier: edge
```

Add new clusters by labeling them, not by editing ApplicationSet specs.

---

## Kyverno Policies

Policies live in `managed-service-catalog/helm/kyverno-policies/`. Key enforced rules:
- **No `latest` image tags** — always use digests or semver
- **No default Argo CD project** — apps must be assigned a named project
- **Read-only root filesystem** — containers must not write to `/`
- **Network policies required** — all namespaces must have NetworkPolicy
- **TLS required on all Ingresses** — Traefik enforces TLS options

When writing new Kubernetes resources, comply with these policies or expect admission failures.

---

## Networking & DNS

- LoadBalancer IPs are served by **MetalLB** from the Docker subnet `172.20.0.0/16`
- DNS uses **traefik.me** wildcard: `*.172.18.255.254.traefik.me` resolves to `172.18.255.254`
- All Ingress hostnames must be subdomains of the configured `DOMAIN_NAME` in `config.yaml`
- TLS certificates are provisioned by **cert-manager** using Let's Encrypt staging by default; switch to production issuer only for real deployments

---

## Development Workflow

### Initial Setup
Follow `01_BOOTSTRAPPING.md` exactly. The sequence matters:
1. Start vind cluster: `vcluster create controlplane --config vcluster-config-controlplane.yaml`
2. Run `./kubara init --prep` → fill `.env` → run `./kubara init`
3. Run `./kubara generate --helm`
4. Run `./kubara bootstrap ...`

### Making Changes
- Edit `config.yaml` to toggle services
- Edit `managed-service-catalog/helm/{service}/values.yaml` for default chart values
- Edit `customer-service-catalog/helm/controlplane/{service}/values.yaml` for cluster overrides
- Re-run `./kubara generate --helm` after config changes to regenerate chart manifests
- Argo CD will sync changes automatically once bootstrapped

### Adding a New Service
1. Create `managed-service-catalog/helm/{service}/` with `Chart.yaml` and `values.yaml`
2. Add `template-library` as a dependency
3. Create `customer-service-catalog/helm/controlplane/{service}/values.yaml` (can be empty)
4. Add the service entry to `config.yaml`
5. Add an Argo CD `Application` or `ApplicationSet` reference

---

## What NOT to Do

- **Do not** modify the `kubara` binary
- **Do not** hardcode cluster-specific values in `managed-service-catalog/` — put them in `customer-service-catalog/`
- **Do not** use `latest` image tags in any Helm values
- **Do not** put secrets (tokens, passwords, keys) in any YAML file — use External Secrets
- **Do not** push to `main` directly — use feature branches
- **Do not** remove pinned chart versions from `Chart.yaml` dependency blocks
- **Do not** create Kubernetes resources outside the defined namespace pattern

---

## Glossary

| Term | Meaning |
|---|---|
| vind | vCluster-in-Docker — runs a full Kubernetes cluster inside Docker containers |
| kubara | Platform distro bootstrap tool (open-source, from STACKIT) |
| Managed Service Catalog | Base Helm chart definitions shared across all clusters |
| Customer Service Catalog | Per-cluster Helm value overrides |
| kagent | AI agent deployment framework running in Kubernetes |
| traefik.me | Wildcard DNS service that resolves `*.{ip}.traefik.me` to `{ip}` |
| IT-Grundschutz | German Federal Office for IT Security baseline — enforced via Kyverno |

---

## Key Files Quick Reference

| Goal | File |
|---|---|
| Enable/disable a service | `config.yaml` |
| Change Kubernetes version or node count | `vcluster-config-controlplane.yaml` |
| Configure Bitwarden secrets | `css-bitwarden.yaml` |
| Override a service value for this cluster | `customer-service-catalog/helm/controlplane/{service}/values.yaml` |
| Change default chart behavior | `managed-service-catalog/helm/{service}/values.yaml` |
| Add or modify Argo CD projects/appsets | `customer-service-catalog/helm/controlplane/argo-cd/values.yaml` |
| Add a Kyverno policy | `managed-service-catalog/helm/kyverno-policies/templates/` |
