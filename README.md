# Self-Hosted Infrastructure

This repository contains the Infrastructure as Code (IaC) for a self-hosted Kubernetes cluster running on Proxmox. It uses Terraform to provision the infrastructure and a GitOps workflow with ArgoCD to manage applications.

See [INFRA_OVERVIEW.md](INFRA_OVERVIEW.md) for a detailed explanation of the core components and process.

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

See [NETWORK.md](NETWORK.md) and [STORAGE.md](STORAGE.md) for more details on those specific areas.

## Applications

The cluster applications are deployed and managed using two methods: bootstrapping with Helmfile and ongoing management with ArgoCD.

### Bootstrap (Helmfile)

The applications in `02-bootstrap/helmfile.yaml` are the core components required to get the cluster into a functional state, ready for GitOps. These are applied manually with `helmfile apply`.

| Application      | Namespace         | Purpose                                                      |
| ---------------- | ----------------- | ------------------------------------------------------------ |
| `metrics-server` | `kube-system`     | Provides resource metrics for pods and nodes.                |
| `cert-manager`   | `cert-manager`    | Manages TLS certificates for the cluster.                    |
| `ingress-nginx`  | `ingress-nginx`   | The Ingress controller that exposes HTTP/S services.         |
| `argo-cd`        | `argocd`          | The GitOps controller that syncs the cluster state with this repo. |
| `longhorn`       | `longhorn-system` | Provides persistent, replicated block storage for applications. |

### GitOps (ArgoCD)

Once ArgoCD is running, it takes over the management of all other applications. The application definitions are located in `03-gitops/applications/`. ArgoCD automatically detects changes in the `main` branch and applies them to the cluster.

**Platform Applications (`01-platform-bootstrap.yaml`)**

These are foundational services required for the cluster to operate effectively.

| Application    | Manages                                                     |
| -------------- | ----------------------------------------------------------- |
| `cert-manager` | `ClusterIssuer` and other certificate-related resources.    |
| `cilium`       | The CNI (Container Network Interface) for pod networking.   |
| `argocd`       | The Ingress resource for the ArgoCD UI.                     |
| `longhorn`     | The Ingress for the Longhorn UI and node disk configuration. |

**Media Applications (`02-media-bootstrap.yaml`)**

These are the user-facing applications.

| Application | Description                                 |
| ----------- | ------------------------------------------- |
| `navidrome` | A self-hosted music server and streamer.      |
