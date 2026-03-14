# Requirements: Production-Grade Kubernetes DevSecOps Platform (V2)

## Project Overview

This project builds a production-grade Kubernetes platform using kubeadm on cloud infrastructure, incorporating lessons learned from the V0 build and community feedback from 223+ engineers who reviewed the original implementation.

**Target Users:**
- DevOps/DevSecOps engineers building portfolio projects
- Junior/mid-level engineers learning production Kubernetes patterns
- Teams evaluating spec-driven development with Kiro

**Reference Implementation:** github.com/NanaGyamfiPrempeh30/k8s-devsecops

---

## Requirement 1: Infrastructure Provisioning

**User Story:** As a platform engineer, I want to provision cloud infrastructure using Terraform so that the entire environment is reproducible and version-controlled.

### Acceptance Criteria

1. WHEN the user runs `terraform apply`, the system SHALL provision a minimum of three compute instances with at least 4GB RAM and 2 vCPUs each.
2. WHEN the user specifies instance types below t3.medium (or equivalent), the Terraform configuration SHALL fail validation with a clear error message stating minimum resource requirements.
3. WHEN infrastructure is provisioned, the system SHALL create a VPC with public and private subnets, security groups allowing SSH (port 22), HTTP (port 80), HTTPS (port 443), and Kubernetes API (port 6443).
4. WHEN the user runs `terraform destroy`, the system SHALL cleanly remove all resources including handling Kubernetes finalizers that previously caused destruction loops.
5. WHEN Terraform provisioning completes, the system SHALL output the IP addresses of all nodes and the SSH connection command for the bastion host.
6. WHILE infrastructure is being provisioned, the system SHALL use Terraform state stored remotely (S3 or equivalent) to enable team collaboration.

---

## Requirement 2: Node Preparation and Cluster Bootstrap

**User Story:** As a platform engineer, I want automated node preparation and cluster initialization so that the cluster is consistent and repeatable across deployments.

### Acceptance Criteria

1. WHEN the preflight check script runs, the system SHALL verify that each node has at least 4GB RAM, 2 vCPUs, swap disabled, and required kernel modules (overlay, br_netfilter) loadable, and SHALL exit with a descriptive error if any check fails.
2. WHEN node preparation runs, the system SHALL disable swap both immediately and persistently via /etc/fstab, configure sysctl parameters for Kubernetes networking (bridge-nf-call-iptables, ip_forward), install containerd with SystemdCgroup enabled, and install kubelet, kubeadm, and kubectl at a pinned version.
3. WHEN kubeadm init runs on the master node, the system SHALL initialize the control plane with a specified pod network CIDR (192.168.0.0/16 for Calico), configure kubeconfig for the root user, and save the worker join command to a file.
4. WHEN worker nodes join the cluster, the system SHALL retrieve the join command from the master and execute it, and the node SHALL appear as Ready within 120 seconds.
5. IF kubeadm init fails, the system SHALL capture the full error log, check for common causes (insufficient memory, swap enabled, port conflicts), and suggest specific remediation steps.
6. WHEN all nodes have joined, the system SHALL verify that `kubectl get nodes` shows all nodes in Ready state before proceeding to the next phase.

---

## Requirement 3: Container Network Interface (CNI)

**User Story:** As a platform engineer, I want Calico CNI installed with network policy support so that pod networking is functional and pod-level security is enforced.

### Acceptance Criteria

1. WHEN the CNI installation phase runs, the system SHALL deploy Calico with the pod CIDR matching the kubeadm init configuration (192.168.0.0/16).
2. WHEN Calico is installed, the system SHALL deploy default-deny ingress and egress network policies in all application namespaces to enforce zero-trust networking.
3. WHEN the cert-manager webhook is deployed, the system SHALL create a NetworkPolicy allowing the Kubernetes API server to reach the cert-manager webhook pod on the required ports (10250, 6080) to prevent webhook timeout errors.
4. WHEN network policies are applied, the system SHALL verify that pod-to-pod communication within the same namespace works, cross-namespace communication is blocked by default, and DNS resolution via CoreDNS functions correctly.
5. IF Calico pods fail to reach Ready state, the system SHALL check for IP pool conflicts, CIDR mismatches, and kernel module availability.

---

## Requirement 4: Load Balancer

**User Story:** As a platform engineer, I want MetalLB providing LoadBalancer services so that services can be exposed with stable external IPs in a bare-metal-style environment.

### Acceptance Criteria

1. WHEN MetalLB is installed, the system SHALL configure an IP address pool within the node subnet range (e.g., 10.0.1.200-10.0.1.220 for a 10.0.1.x node network).
2. WHEN a Kubernetes Service of type LoadBalancer is created, the system SHALL assign an external IP from the MetalLB pool within 30 seconds.
3. IF the MetalLB IP pool range does not match the node subnet, the system SHALL fail with a clear error indicating the mismatch, since this was a root cause of services stuck in Pending state in V0.
4. WHEN MetalLB is operational, the system SHALL verify that at least one test service receives an external IP successfully.

---

## Requirement 5: Ingress Controller

**User Story:** As a platform engineer, I want Traefik as the ingress controller so that HTTP/HTTPS traffic is routed to services within the cluster.

### Acceptance Criteria

1. WHEN Traefik is installed via Helm, the system SHALL configure it with both web (port 80) and websecure (port 443) entrypoints, with web as the default entrypoint for initial setup.
2. WHEN an Ingress resource is created, the system SHALL use the annotation `traefik.ingress.kubernetes.io/router.entrypoints: web` by default to prevent 404 errors caused by missing TLS secrets, which was the primary ingress debugging issue in V0.
3. WHEN TLS certificates are available via cert-manager, the system SHALL support upgrading individual Ingress resources to use the websecure entrypoint with the corresponding TLS secret.
4. WHEN Traefik receives a request with a Host header, the system SHALL route it to the correct backend service, including when accessed via SSH tunnels where the Host header may include a port number.
5. WHEN Traefik is operational, the system SHALL expose Prometheus metrics via a ServiceMonitor with the label `release: prometheus` to ensure Prometheus scrapes Traefik metrics.

---

## Requirement 6: Certificate Management

**User Story:** As a platform engineer, I want cert-manager handling TLS certificates so that services can be secured with automated certificate provisioning and renewal.

### Acceptance Criteria

1. WHEN cert-manager is installed, the system SHALL create three ClusterIssuers: self-signed (for testing), Let's Encrypt staging, and Let's Encrypt production.
2. WHEN cert-manager is deployed before other platform services, the system SHALL be operational and ready to issue certificates before Traefik ingresses are created, preventing the TLS debugging cycle experienced in V0.
3. WHEN a Certificate resource is created, the DNS names in the certificate SHALL match the host field in the corresponding Ingress resource exactly.
4. IF the cluster uses nip.io domains, the system SHALL default to self-signed certificates, since Let's Encrypt does not issue certificates for nip.io domains.
5. WHERE the cluster has a registered domain, the system SHALL support Let's Encrypt production certificates with HTTP-01 challenge validation via Traefik.

---

## Requirement 7: Secrets Management

**User Story:** As a platform engineer, I want HashiCorp Vault managing secrets with External Secrets Operator syncing them into Kubernetes so that no secrets are hardcoded in manifests or environment variables.

### Acceptance Criteria

1. WHEN Vault is installed, the system SHALL deploy it with a persistent storage backend (not dev mode) so that secrets survive pod restarts, addressing the V0 limitation where dev mode stored secrets only in memory.
2. WHEN Vault is initialized, the system SHALL use auto-unseal via cloud KMS (or documented manual unseal procedure) and store the root token and unseal keys securely.
3. WHEN the External Secrets Operator is configured with a ClusterSecretStore pointing to Vault, the system SHALL use the path format `secret/data/<path>` for KV v2, not `secret/<path>`, which caused empty secrets in V0.
4. WHEN an ExternalSecret resource is created, the system SHALL sync the corresponding Vault secret into a Kubernetes Secret within 60 seconds and report Synced status.
5. IF an ExternalSecret fails to sync, the system SHALL report a clear error in the ExternalSecret status indicating whether the issue is authentication, path format, or Vault availability.
6. WHEN Vault UI is exposed via ingress, the system SHALL use the HTTP entrypoint initially and document the login method (token-based for initial setup).

---

## Requirement 8: GitOps Deployment

**User Story:** As a platform engineer, I want ArgoCD managing application deployments via GitOps so that all changes are auditable, repeatable, and automated.

### Acceptance Criteria

1. WHEN ArgoCD is installed, the system SHALL be accessible via ingress on the web (HTTP) entrypoint and SHALL display the initial admin password retrieval command in the deployment output.
2. WHEN an ArgoCD Application resource is created pointing to a Git repository, the system SHALL sync the application and report Synced and Healthy status within 120 seconds.
3. WHEN a change is pushed to the Git repository, ArgoCD SHALL detect the change and auto-sync (or notify for manual sync, depending on configuration) within the configured polling interval.
4. WHEN ArgoCD is accessed via SSH tunnel, the system SHALL function correctly on port 80 to avoid Host header port mismatch issues encountered in V0 when tunneling on ports 8080 or 9090.
5. WHEN multiple environments are required (dev/prod), the system SHALL support ArgoCD ApplicationSets with directory or list generators for dynamic multi-environment deployments.
6. WHEN the sample application is deployed via ArgoCD, it SHALL include a Deployment, Service, and Ingress resource, with health and readiness probes configured.

---

## Requirement 9: Observability Stack

**User Story:** As a platform engineer, I want Prometheus and Grafana providing metrics collection and visualization so that cluster and application health is observable.

### Acceptance Criteria

1. WHEN the kube-prometheus-stack is installed via Helm, the system SHALL deploy Prometheus, Grafana, AlertManager, and node-exporter with persistent storage.
2. WHEN ServiceMonitors are created for platform components (Traefik, Vault, ArgoCD), they SHALL include the label `release: prometheus` to ensure Prometheus discovers and scrapes them, fixing the scraping gap from V0.
3. WHEN Grafana is accessible via ingress, the system SHALL include pre-configured dashboards for cluster overview (Dashboard ID 17346), node metrics, and pod metrics.
4. WHEN Prometheus is operational, the system SHALL scrape all configured targets and report them as UP in the Prometheus targets page.
5. IF the Prometheus Operator pod crashes or scales to zero, the system SHALL have a documented recovery procedure, since this caused cascading monitoring failures in V0.
6. WHEN log aggregation is required, the system SHALL support Loki or EFK stack deployment for centralized logging, addressing the log aggregation gap identified by community feedback.

---

## Requirement 10: RBAC and Access Control

**User Story:** As a platform engineer, I want role-based access control configured so that the principle of least privilege is enforced across the cluster.

### Acceptance Criteria

1. WHEN the cluster is operational, the system SHALL create dedicated ServiceAccounts for each platform component (ArgoCD, Vault, Prometheus) rather than using default ServiceAccounts.
2. WHEN RBAC is configured, the system SHALL create Roles and RoleBindings limiting each component to the minimum permissions required for its namespace.
3. WHEN a new namespace is created for application workloads, the system SHALL automatically apply a default NetworkPolicy (deny-all ingress/egress) and a ResourceQuota to prevent resource exhaustion.
4. WHEN cluster-admin access is required for debugging, the system SHALL document the procedure for temporary privilege escalation and log the action.

---

## Requirement 11: Backup and Disaster Recovery

**User Story:** As a platform engineer, I want automated backup and recovery procedures so that the cluster can be restored after failures.

### Acceptance Criteria

1. WHEN Velero is installed, the system SHALL configure automated daily backups of all namespaces, PersistentVolumes, and cluster-scoped resources to an object storage backend (S3 or equivalent).
2. WHEN an etcd backup script runs, the system SHALL create a snapshot of the etcd datastore and store it outside the cluster with a timestamp.
3. WHEN a disaster recovery test is performed, the system SHALL restore a namespace from backup within 15 minutes and verify all pods are running and services are accessible.
4. WHEN Vault data needs recovery, the system SHALL have a documented procedure for restoring the Vault backend from a persistent volume backup.

---

## Requirement 12: Automation Scripts

**User Story:** As a newcomer to the platform, I want shell automation scripts so that I can deploy the entire stack by running scripts in sequence without deep Kubernetes expertise.

### Acceptance Criteria

1. WHEN the automation scripts directory is structured, it SHALL follow a numbered sequence (00-preflight, 01-prepare-nodes, 02-init-master, etc.) with each script checking that its dependencies from previous scripts are met before executing.
2. WHEN any script is run, it SHALL be idempotent — running it multiple times SHALL produce the same result without errors, using patterns like `helm upgrade --install`, `kubectl apply`, and check-before-create.
3. WHEN the preflight check script (00-preflight-check.sh) runs, it SHALL verify OS version, RAM (minimum 4GB), CPU count (minimum 2), swap status, network connectivity between nodes, and required port availability.
4. WHEN all scripts complete successfully, a verification script (99-verify-all.sh) SHALL check node status, pod health across all namespaces, service endpoints, ingress routes, certificate validity, Vault seal status, and ArgoCD sync status.
5. WHEN a script fails, it SHALL output a clear error message with the failure reason and a suggested fix based on known issues from V0.
6. WHEN configuration values need customization (IP addresses, domain names, version numbers), they SHALL be defined in a single config.env file that all scripts source.

---

## Requirement 13: Terraform Lifecycle Management

**User Story:** As a platform engineer, I want Terraform to manage the full lifecycle including clean destruction so that infrastructure can be torn down without manual intervention.

### Acceptance Criteria

1. WHEN `terraform destroy` is executed, the system SHALL handle Kubernetes finalizers by patching them before deletion, preventing the infinite destruction loop encountered in V0 with provisioner-based deployments.
2. WHEN Phase 2 resources (Helm releases, ArgoCD applications) are destroyed, the system SHALL remove them in reverse dependency order: applications first, then operators, then namespaces.
3. WHEN Phase 1 resources (cluster infrastructure) are destroyed, the system SHALL run `kubeadm reset` on all nodes, clean CNI configurations, and remove NFS exports before terminating compute instances.
4. WHEN using Helm provider in Terraform instead of remote-exec provisioners, the system SHALL track all Helm releases in Terraform state for proper lifecycle management.

---

## Requirement 14: Documentation and Content

**User Story:** As a contributor or learner, I want comprehensive documentation so that I can understand the architecture, deploy the platform, and troubleshoot issues.

### Acceptance Criteria

1. WHEN the repository README is viewed, it SHALL include an architecture diagram, technology stack table, prerequisites, quick-start guide, and links to detailed documentation.
2. WHEN a deployment issue occurs, the troubleshooting section SHALL cover all known issues from V0 with their root causes and fixes, organized by component.
3. WHEN the platform is deployed for educational purposes, the documentation SHALL explain why each component exists and what problem it solves, not just how to install it.
4. WHEN sessions are recorded for educational content, the system SHALL have a clear deployment runbook with estimated timing per phase (approximately 75 minutes total).

---

## Requirement 15: Security Hardening

**User Story:** As a security-conscious engineer, I want the platform to follow security best practices so that it demonstrates production-worthy security posture.

### Acceptance Criteria

1. WHEN the cluster is operational, the system SHALL change all default passwords (ArgoCD admin, Grafana admin, Vault root token) from their installation defaults.
2. WHEN services are accessed from outside the cluster, the system SHALL use either SSH tunneling or a zero-trust access solution (Tailscale/Twingate) rather than exposing services publicly, as recommended by community feedback.
3. WHEN container images are deployed, the system SHALL pull from a private registry (Harbor or equivalent) with vulnerability scanning enabled.
4. WHEN secrets are stored, they SHALL never appear in plaintext in Git repositories, Terraform state files, or Kubernetes manifests (all secrets managed via Vault + External Secrets Operator).
5. WHEN the platform is audited, the system SHALL have Kubernetes audit logging enabled and Vault audit logging configured.

---

## Non-Functional Requirements

### Performance
- WHILE the full platform stack is running, total infrastructure memory consumption SHALL not exceed 3.7GB per node, leaving headroom for application workloads on 4GB instances.
- WHEN Helm charts are installed, the system SHALL use `helm upgrade --install` to prevent failures from pre-existing releases.

### Reliability
- IF a single worker node fails, the system SHALL continue serving application workloads from the remaining worker node(s).
- WHEN the control plane is a single node (budget constraint), the documentation SHALL clearly state this is a single point of failure and document the HA upgrade path with stacked etcd.

### Cost
- WHILE running on AWS, the estimated infrastructure cost SHALL not exceed $6 for a 2-day deployment using t3.medium instances in eu-west-1.
- WHEN the project is complete, `terraform destroy` SHALL remove all billable resources within 10 minutes.

### Compatibility
- WHEN deployed on bare-metal or non-AWS environments, the platform SHALL function with MetalLB and kubeadm without AWS-specific dependencies, maintaining cloud-agnostic portability as noted by community reviewer Steve Atkinson.

---

## Community-Requested Improvements (V1 Backlog)

These items were suggested by community reviewers of the V0 article and are tracked as future enhancements:

| Item | Suggested By | Priority | Status |
|------|-------------|----------|--------|
| Default-deny network policies with Calico | Nick Gole, Aldy | High | Included in Req 3 |
| Vault production mode (not dev mode) | Nick Gole | High | Included in Req 7 |
| RBAC with least-privilege ServiceAccounts | Nick Gole | High | Included in Req 10 |
| Velero backup and disaster recovery | Boris Sadkhin, Nick Gole | High | Included in Req 11 |
| Log aggregation (Loki/EFK) | Nick Gole | Medium | Included in Req 9 |
| Zero-trust access (Twingate/Tailscale) | Emailbob | Medium | Included in Req 15 |
| Install cert-manager early in the sequence | Buff Developer | Medium | Included in Req 6 |
| Gateway API replacing Ingress | Aldy | Low | Future V2 |
| Cilium replacing Calico (eBPF) | Aldy | Low | Future V2 |
| Istio Ambient service mesh | Buff Developer | Low | Future V2 |
| Talos Linux replacing kubeadm | Christophe Deliens | Low | Future V2 |
| OpenBao replacing Vault | Scott Molinari | Low | Future V2 |
| K3s/Rancher for lighter footprint | Ravindra Singh | Low | Future V2 |
| HA control plane with stacked etcd | Multiple commenters | Low | Future V2 |

---

## Known Issues from V0 (Regression Prevention)

These issues MUST NOT recur in V1. Each is addressed by specific acceptance criteria above.

| V0 Issue | Root Cause | Prevention (Requirement) |
|----------|-----------|------------------------|
| OOM on cluster init | t2.micro/t3.small instances | Req 1 AC-1, Req 2 AC-1 |
| Traefik 404 on all routes | HTTPS entrypoint with no TLS secret | Req 5 AC-2 |
| cert-manager webhook timeout | Calico blocking apiserver-to-webhook | Req 3 AC-3 |
| MetalLB services stuck Pending | IP pool outside node subnet | Req 4 AC-3 |
| External Secrets empty | Vault KV v2 path missing /data/ | Req 7 AC-3 |
| Prometheus not scraping Traefik | ServiceMonitor missing release label | Req 9 AC-2 |
| ArgoCD inaccessible via tunnel | Host header port mismatch on 8080 | Req 8 AC-4 |
| Helm install fails on retry | Pre-existing release name | Non-Functional AC-2 |
| terraform destroy loops forever | Kubernetes finalizers on resources | Req 13 AC-1 |
| SSH lockout on bastion | iptables rules blocking port 22 | Req 15 AC-2 |
| Vault secrets lost on restart | Dev mode stores in memory only | Req 7 AC-1 |
