# Infrastructure Overview

This document provides a high-level overview of the infrastructure managed in this repository.

## Goal

The primary goal of this setup is to use Terraform to automatically provision a production-ready Talos-based Kubernetes cluster on a Proxmox VE hypervisor. This approach is known as "Infrastructure as Code" (IaC), which allows for repeatable and version-controlled infrastructure.

## Architecture

The high-level architecture of the setup is as follows:

```
┌───────────────────────────────────────────────────┐
│                    Proxmox Host                   │
│ ┌─────────────────┐         ┌───────────────────┐ │
│ │ talos-cp-01 (VM)│         │ talos-wk-01 (VM)  │ │
│ │ 192.168.100.60  │         │  192.168.100.70   │ │
│ └───────┬─────────┘         └─────────┬─────────┘ │
└─────────┼─────────────────────────────┼───────────┘
          │                             │
          │     Kubernetes Cluster      │
          │                             │
┌─────────┴─────────────────────────────┴──────────┐
│ ┌──────────────────────────────────────────────┐ │
│ │      In-Cluster Services & Applications      │ │
│ │ ┌────────────┐ ┌──────────┐ ┌──────────────┐ │ │
│ │ │  Longhorn  │ │ Ingress  │ │   ArgoCD     │ │ │
│ │ │ (Storage)  │ │ (Network)│ │   (GitOps)   │ │ │
│ │ └────────────┘ └──────────┘ └──────────────┘ │ │
│ │ ┌────────────┐ ┌──────────┐ ┌──────────────┐ │ │
│ │ │ Cert-Mgr   │ │  Media   │ │  Monitoring  │ │ │
│ │ │  (TLS)     │ │ (Navi..) │ │ (Metrics)    │ │ │
│ │ └────────────┘ └──────────┘ └──────────────┘ │ │
│ └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

## Core Components

1.  **Terraform:** The orchestration tool used to define and manage the lifecycle of all infrastructure resources. The configuration files (`.tf`) in the `infra/` directory describe the desired state of the system.

2.  **Proxmox VE:** The open-source hypervisor that hosts the virtual machines for the Kubernetes cluster. Terraform communicates with the Proxmox API to create, configure, and destroy these VMs.

3.  **Talos Linux:** A modern, minimal, and secure Linux distribution designed specifically for running Kubernetes. It's used as the operating system for all control plane and worker nodes. Talos is configured via a YAML manifest, which Terraform generates and injects into the VMs.

4.  **Kubernetes:** The container orchestration platform. Once the Talos nodes are provisioned, they form a Kubernetes cluster, ready to run containerized applications.

## How It Works

The process is initiated by running `terraform apply` in the `infra/` directory. Here's a summary of what happens:

1.  **VM Provisioning:** Terraform connects to your Proxmox server and creates a set of virtual machines based on the settings in `variables.tf` and `proxmox_nodes.tf`. This includes defining CPU, memory, and disk resources. By default, it creates one control plane and one worker node.
2.  **Talos Configuration:** Terraform generates machine configurations for Talos, including network settings, Kubernetes version, and node roles.
3.  **Patches:** Custom configurations from the `patches/` directory are applied to the Talos configuration. These patches are used to customize the cluster, for example, by enabling `kubeprism` or disabling the default network CNI to use Cilium.
4.  **Cluster Initialization:** The VMs boot with the injected Talos configuration and automatically form a Kubernetes cluster. The control plane node establishes the API server, and the worker nodes join the cluster.

After the process completes, Terraform outputs a `kubeconfig` file, allowing you to immediately connect to and manage your new Kubernetes cluster with `kubectl`.