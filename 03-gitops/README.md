# 03-gitops

In this stage, your Kubernetes cluster becomes fully GitOps-driven with automated deployment of platform services, observability stack, and demo workloads.
Through Argo CD, all components—including platform services, observability tools, and workloads—are deployed in a declarative and reproducible manner.

## Purpose

This layer turns your cluster into a fully GitOps-managed platform, enabling reproducible deployments, real-time observability, and traceable demo applications.

Argo CD Applications are bootstrapped in a controlled order to ensure service readiness and interdependency handling. This approach enables reproducible, declarative deployment of all Kubernetes workloads via Git.

## Structure

| Layer         | Path                  | Description                                    |
| ------------- | --------------------- | ---------------------------------------------- |
| Applications  | `applications/`       | Argo CD Application definitions (per stage)    |
| Platform Apps | `apps/01-platform/`   | Argo CD, cert-manager, Cilium, Longhorn        |
| Monitoring    | `apps/02-monitoring/` | Prometheus, Grafana, Tempo, Loki, Jaeger, etc. |
| Demo          | `apps/03-demo/`       | OpenTelemetry Demo (`otel-demo`)               |

## Ingress Access

All Ingress resources created in this stage are configured to work with the NGINX Ingress Controller.
They receive static IPs from the Cilium LoadBalancer IP pool. By default, services are exposed via `192.168.100.80`.

TLS configuration blocks are included in the manifests but commented out. To enable TLS for Ingress resources, uncomment the relevant sections in the Ingress manifests. Then extract the self-signed certificate and add it to your local trust store:

```bash
kubectl get secret ingress-tls -n cert-manager -o jsonpath='{.data.tls\.crt}' | base64 -d > local-cluster-root-ca.crt
```

You can then add this certificate to your system trust store:

* **macOS**: open Keychain Access → drag the file into "System" → set trust to "Always Trust"
* **Linux**: copy to `/usr/local/share/ca-certificates/` and run `sudo update-ca-certificates`.

To access services via domain names, you can:

* Update your local `/etc/hosts` file (example below):

```bash
192.168.100.80  argocd.homelab.local grafana.homelab.local prometheus.homelab.local \
                alertmanager.homelab.local loki.homelab.local tempo.homelab.local \
                jaeger.homelab.local longhorn.homelab.local \
                otel-demo.homelab.local otel-demo-loadgen.homelab.local
```

* Or configure wildcard DNS entries (e.g., \*.homelab.local) pointing to the Ingress IP.

## Usage

> **Pre-requisite**: Ensure Argo CD is already running in your cluster. It was installed via Helm in the previous stage (`02-bootstrap`).

It is recommended to apply the Argo CD Applications in order, as each layer builds upon the previous one (e.g., monitoring components depend on platform services such as cert-manager and ingress).

### Step-by-step deployment

```bash
# 1. Apply platform components
kubectl apply -f applications/01-platform-bootstrap.yaml
```

### Application breakdown

* `01-platform-bootstrap.yaml`

  * Adds Cilium to Argo CD management (already pre-installed)
  * Installs cert-manager with self-signed CA
  * Deploys Ingress resources for Argo CD and Longhorn UIs

* `02-media-bootstrap.yaml`

  * Adds Navidrome for music management.

### Secrets 

#### SFTP
```bash
kubectl create secret generic sftp-user-credentials    --namespace=media    --from-literal=SFTP_USERS="olaf:password:1001:1001:music"
```

#### Multiscrobbler
Last.fm credentials are required for scrobbling. Create the secret with your actual Last.fm API credentials:

Go in the browser and hit `https://www.last.fm/api/auth?api_key=api_key&token=api_secret`.

Then create:

```bash
kubectl create secret generic multiscrobbler-lastfm-creds \
  --namespace=media \
  --from-literal=LASTFM_API_KEY="api_key" \
  --from-literal=LASTFM_SECRET="api_secret"
```

Replace the placeholder values with your actual Last.fm API key and secret obtained from https://www.last.fm/api/account/create

## Navigation

[← 02-bootstrap](../02-bootstrap/README.md) • [↑ Main project README](../README.md)
