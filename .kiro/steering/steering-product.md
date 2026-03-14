---
title: Product Overview
inclusion: always
---

# Product Overview: K8s Platform Kiro

## Purpose

A production-grade Kubernetes DevSecOps platform built using spec-driven development with Amazon Kiro. This project demonstrates how AI-assisted specifications can generate infrastructure code that matches the quality of hand-built, battle-tested deployments.

The platform serves as both a functional Kubernetes environment and an educational reference for engineers learning production Kubernetes patterns. It incorporates lessons learned from a V0 build that was reviewed by 223+ engineers, with all identified gaps addressed in this V1 specification.

## Target Users

- DevOps and DevSecOps engineers building portfolio projects or learning production Kubernetes
- Junior to mid-level engineers who need a structured path from zero to production-ready infrastructure
- Teams evaluating spec-driven development for infrastructure projects
- Engineers transitioning from managed Kubernetes (EKS/GKE) to understanding the full stack

## Key Features

- Full Kubernetes cluster via kubeadm with Calico CNI and network policy enforcement
- Infrastructure as Code with Terraform for reproducible provisioning
- Configuration management with Ansible for node preparation and cluster bootstrap
- GitOps deployments via ArgoCD with multi-environment support (dev/prod)
- Centralized secrets management with HashiCorp Vault (production mode) and External Secrets Operator
- Full observability stack: Prometheus, Grafana, and Loki for metrics, dashboards, and log aggregation
- Automated TLS certificate management with cert-manager
- Backup and disaster recovery with Velero
- Load testing with Locust for realistic traffic generation
- Shell automation scripts enabling newcomers to deploy the full stack sequentially

## Business Objectives

- Demonstrate that spec-driven development can produce production-quality infrastructure code
- Create recorded educational content showing the spec-to-deployment workflow
- Provide a reusable reference architecture for Kubernetes DevSecOps platforms
- Build an open-source project that attracts contributors across experience levels

## Success Metrics

- All 15 requirements from the spec are implemented and verified
- Zero recurrence of V0 known issues (documented in requirements.md regression table)
- Platform supports deploying and load-testing a sample application (OpenTelemetry demo or equivalent)
- Recorded sessions demonstrate the full workflow for educational purposes
- Repository attracts community engagement (stars, forks, issues, PRs)
