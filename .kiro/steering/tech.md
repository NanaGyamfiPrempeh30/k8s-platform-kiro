---
title: Technology Stack
inclusion: always
---

# Technology Stack

## Infrastructure Provisioning

- **Terraform** (>= 1.5) for all cloud resource provisioning (compute instances, VPC, security groups, S3 state backend)
- **Ansible** (>= 2.15) for node configuration management, cluster bootstrap, and platform service installation
- Do NOT suggest CloudFormation, Pulumi, or Crossplane as alternatives
- Do NOT use Terraform remote-exec provisioners for cluster bootstrap — use Ansible instead
- Terraform state MUST be stored remotely (S3 or equivalent), never locally

## Kubernetes

- **kubeadm** for cluster initialization — do NOT suggest managed Kubernetes (EKS, GKE, AKS)
- **Kubernetes version:** 1.31.x (pinned via apt-mark hold)
- **Container runtime:** containerd with SystemdCgroup enabled
- Minimum node sizing: 4GB RAM, 2 vCPUs per node (t3.medium equivalent)
- Control plane: single master node (HA with stacked etcd is a documented future upgrade)

## Networking

- **Calico** for CNI and network policy enforcement — do NOT suggest Cilium, Flannel, or Weave
- **MetalLB** for LoadBalancer service implementation in bare-metal environments
- IP address pool MUST be within the node subnet range
- Default-deny network policies in all application namespaces

## Ingress and TLS

- **Traefik** as the ingress controller — do NOT suggest nginx-ingress, HAProxy, or Istio gateway
- Default entrypoint: web (HTTP port 80) — do NOT configure websecure as default without verified TLS secrets
- **cert-manager** for automated TLS certificate management
- ClusterIssuers: self-signed (testing), Let's Encrypt staging, Let's Encrypt production
- cert-manager MUST be installed before creating any HTTPS ingress resources

## Secrets Management

- **HashiCorp Vault** in production mode with persistent storage — do NOT use dev mode
- **External Secrets Operator** for syncing Vault secrets into Kubernetes
- Vault KV v2 paths MUST use `secret/data/<path>` format, not `secret/<path>`
- Do NOT suggest AWS Secrets Manager, Azure Key Vault, or SOPS as alternatives for this project

## GitOps

- **ArgoCD** for declarative continuous delivery — do NOT suggest Flux
- Support ApplicationSets for multi-environment (dev/prod) deployments
- ArgoCD ingress MUST use the web (HTTP) entrypoint by default

## Observability

- **Prometheus** via kube-prometheus-stack Helm chart for metrics collection
- **Grafana** for visualization with pre-configured dashboards
- **Loki** for log aggregation (addressing V0 gap)
- All ServiceMonitors MUST include the label `release: prometheus`
- Do NOT suggest Datadog, New Relic, Dynatrace, or other commercial alternatives

## Backup and Recovery

- **Velero** for Kubernetes resource and persistent volume backup
- etcd snapshot backups via built-in etcdctl

## Load Testing

- **Locust** for generating synthetic traffic against deployed applications
- OpenTelemetry Demo or equivalent microservices application as the sample workload

## CI/CD

- **GitHub Actions** for CI pipelines
- **GitHub** as the Git hosting platform and container registry (ghcr.io)

## Code Formatting

- **Indentation:** 3 spaces (not 2 or 4)
- **Line endings:** LF (Unix-style)
- **Trailing whitespace:** Remove on save
- **Final newline:** Always include

## Shell Scripting

- All automation scripts in **Bash** (POSIX-compatible where possible)
- Scripts MUST be idempotent — safe to re-run without side effects
- Use `helm upgrade --install` instead of `helm install`
- Use `kubectl apply` instead of `kubectl create`

## Development Tools

- **Amazon Kiro** IDE and CLI for spec-driven development
- Steering files in `.kiro/steering/`
- Specs in `.kiro/specs/`
- Hooks in `.kiro/hooks/`
