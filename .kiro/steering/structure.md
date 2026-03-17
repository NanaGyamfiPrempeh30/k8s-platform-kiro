---
title: Project Structure
inclusion: always
---

# Project Structure

## Repository Layout

```
k8s-platform-kiro/
├── .kiro/
│   ├── specs/
│   │   ├── requirements.md          # EARS-format platform requirements
│   │   ├── design.md                # Technical architecture and design decisions
│   │   └── tasks.md                 # Implementation tasks generated from design
│   ├── steering/
│   │   ├── product.md               # Product overview, target users, objectives
│   │   ├── tech.md                  # Technology stack and tool constraints
│   │   └── structure.md             # This file — repo organization and conventions
│   └── hooks/
│       ├── validate-terraform.kiro.hook
│       ├── validate-yaml.kiro.hook
│       └── security-scan.kiro.hook
├── terraform/
│   ├── main.tf                      # Provider config and module calls
│   ├── variables.tf                 # Input variables with validation
│   ├── outputs.tf                   # IP addresses, connection commands
│   ├── terraform.tfvars.example     # Example variables (never commit actual .tfvars)
│   ├── versions.tf                  # Provider version constraints
│   └── modules/
│       ├── vpc/                     # VPC, subnets, security groups
│       ├── compute/                 # EC2 instances (master, workers)
│       └── destroy/                 # Clean destruction with finalizer handling
├── ansible/
│   ├── inventory/
│   │   ├── hosts.ini.example        # Inventory template
│   │   └── group_vars/
│   │       ├── all.yml              # Shared variables
│   │       ├── masters.yml          # Master-specific variables
│   │       └── workers.yml          # Worker-specific variables
│   ├── playbooks/
│   │   ├── 01-prepare-nodes.yml     # System prep, containerd, kubelet
│   │   ├── 02-init-master.yml       # kubeadm init, kubeconfig setup
│   │   ├── 03-join-workers.yml      # Worker node joining
│   │   └── 04-install-platform.yml  # Helm installs for platform services
│   ├── roles/                       # Reusable Ansible roles
│   └── site.yml                     # Master playbook orchestrating all phases
├── manifests/
│   ├── calico/                      # Calico CNI and network policies
│   │   ├── calico.yaml
│   │   └── default-deny-policies.yaml
│   ├── metallb/                     # MetalLB configuration
│   │   ├── metallb-config.yaml
│   │   └── ip-address-pool.yaml
│   ├── traefik/                     # Traefik Helm values and ingress templates
│   │   └── values.yaml
│   ├── cert-manager/                # ClusterIssuers and certificate templates
│   │   ├── cluster-issuers.yaml
│   │   └── certificate-template.yaml
│   ├── vault/                       # Vault Helm values and configuration
│   │   ├── values.yaml
│   │   └── cluster-secret-store.yaml
│   ├── external-secrets/            # ESO Helm values and ExternalSecret templates
│   │   └── values.yaml
│   ├── argocd/                      # ArgoCD Helm values and ApplicationSets
│   │   ├── values.yaml
│   │   └── applicationset.yaml
│   ├── monitoring/                  # Prometheus, Grafana, Loki values
│   │   ├── kube-prometheus-values.yaml
│   │   └── loki-values.yaml
│   ├── velero/                      # Velero Helm values and backup schedules
│   │   └── values.yaml
│   └── sample-app/                  # Sample application for deployment
│       ├── namespace.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       └── hpa.yaml
├── scripts/
│   ├── 00-preflight-check.sh        # Verify OS, RAM, CPU, swap, connectivity
│   ├── 01-prepare-nodes.sh          # Kernel modules, sysctl, containerd, kubelet
│   ├── 02-init-master.sh            # kubeadm init, kubeconfig, save join command
│   ├── 03-join-workers.sh           # kubeadm join on each worker
│   ├── 04-install-cni.sh            # Calico CNI and default network policies
│   ├── 05-install-metallb.sh        # MetalLB and IP pool configuration
│   ├── 06-install-traefik.sh        # Traefik ingress controller
│   ├── 07-install-certmanager.sh    # cert-manager and ClusterIssuers
│   ├── 08-install-vault.sh          # Vault init, unseal, KV v2 engine
│   ├── 09-install-eso.sh            # External Secrets Operator and ClusterSecretStore
│   ├── 10-install-argocd.sh         # ArgoCD and ingress configuration
│   ├── 11-install-monitoring.sh     # Prometheus, Grafana, Loki
│   ├── 12-install-velero.sh         # Velero backup configuration
│   ├── 99-verify-all.sh             # Full stack health verification
│   └── lib/
│       ├── common.sh                # Shared functions (logging, retry, wait_for_pod)
│       └── config.env               # User-configurable variables (IPs, versions, domains)
├── docs/
│   ├── architecture.md              # Architecture diagrams and component descriptions
│   ├── troubleshooting.md           # Known issues with root causes and fixes
│   ├── runbook.md                   # Step-by-step deployment guide for recorded sessions
│   └── v0-lessons-learned.md        # Regression prevention reference from V0
├── .gitignore
├── LICENSE
└── README.md
```

## Naming Conventions

- **Terraform files:** snake_case (e.g., `security_groups.tf`, `node_instances.tf`)
- **Ansible playbooks:** numbered prefix with kebab-case description (e.g., `01-prepare-nodes.yml`)
- **Kubernetes manifests:** kebab-case matching the resource name (e.g., `default-deny-policies.yaml`)
- **Shell scripts:** numbered prefix matching deployment order (e.g., `05-install-metallb.sh`)
- **Helm values files:** `values.yaml` within component-named directories
- **Documentation:** kebab-case markdown files (e.g., `lessons-learned.md`)

## File Conventions

- All YAML files use `.yaml` extension (not `.yml`) for Kubernetes manifests
- Ansible playbooks use `.yml` extension (Ansible convention)
- Shell scripts include a shebang (`#!/bin/bash`) and are executable (`chmod +x`)
- Every script sources `lib/config.env` for configuration values
- No secrets, passwords, API keys, or tokens in any committed file
- Example files use `.example` suffix (e.g., `terraform.tfvars.example`, `hosts.ini.example`)

## Git Conventions

- **Commit messages:** Conventional Commits format (`feat:`, `fix:`, `docs:`, `refactor:`, `chore:`)
- **Branch strategy:** `main` is the default branch; feature branches use `feat/<description>`
- **Commit frequency:** At least one commit per day when actively developing
- **Never commit:** `.tfvars`, `.tfstate`, `hosts.ini` with real IPs, `.env` files, unseal keys, tokens

## Import Patterns

- Terraform modules referenced via relative paths (`../modules/vpc`)
- Ansible roles referenced via `roles/` directory
- Helm charts installed from official repositories, version-pinned
- Shell scripts source shared libraries via `source "$(dirname "$0")/lib/common.sh"`
