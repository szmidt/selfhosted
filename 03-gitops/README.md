# 03-gitops

In this stage, your Kubernetes cluster becomes fully GitOps-driven with automated deployment of platform services, observability stack, and demo workloads.
Through Argo CD, all components—including platform services, observability tools, and workloads—are deployed in a declarative and reproducible manner.

## Purpose

This layer turns your cluster into a fully GitOps-managed platform, enabling reproducible deployments, real-time observability, and traceable demo applications.

## Structure

| Layer         | Path                | Description                                     |
| ------------- |---------------------|-------------------------------------------------|
| Applications  | `applications/`     | Argo CD Application definitions (per stage)     |
| Platform Apps | `apps/01-platform/` | Argo CD, cert-manager, Cilium, Longhorn         |
| Media         | `apps/02-media/`    | Jellyfin, navidrome, sftp client, multiscrobler |
| Personal      | `apps/03-personal/` | Immich, Vaultwarden                            |

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

# 2. Apply media components
kubectl apply -f applications/02-media-bootstrap.yaml

# 3. Apply personal components
kubectl apply -f applications/03-personal-bootstrap.yaml
```

### Application breakdown

* `01-platform-bootstrap.yaml`

  * Adds Cilium to Argo CD management (already pre-installed)
  * Installs cert-manager with self-signed CA
  * Deploys Ingress resources for Argo CD and Longhorn UIs

* `02-media-bootstrap.yaml`

  * Adds media services, no explanation needed.

* `03-personal-bootstrap.yaml`

  * Deploys personal services: Immich (photo management) and Vaultwarden (password manager).

### Secrets 

Below secrets are required for services to work correctly.

#### Immich

Immich requires a PostgreSQL database with the `vector` extension. This is deployed using the `cloudnative-pg` operator.

The `cloudnative-pg` operator automatically creates a superuser secret, but Immich requires an application user secret with specific field names. Create this secret manually:

```bash
kubectl create secret generic immich-postgres-user \
  --namespace=personal \
  --from-literal=DB_USERNAME="immich" \
  --from-literal=DB_DATABASE_NAME="immich" \
  --from-literal=DB_PASSWORD="immich" \
  --from-literal=username="immich" \
  --from-literal=password="immich"
```

**Important**: Replace `"immich"` password values with a secure password for production use. The database user will be created by the CloudNativePG operator using the credentials from this secret.

#### SFTP

Define any password.

```bash
kubectl create secret generic sftp-user-credentials    --namespace=media    --from-literal=SFTP_USERS="olaf:password:1001:1001:music"
```

#### Multiscrobbler
Last.fm credentials are required for scrobbling. Create the secret with your actual Last.fm API credentials:

Then create:

```bash
kubectl create secret generic multiscrobbler-lastfm-creds \
  --namespace=media \
  --from-literal=LASTFM_API_KEY="api_key" \
  --from-literal=LASTFM_SECRET="api_secret"
```

Replace the placeholder values with your actual Last.fm API key and secret obtained from https://www.last.fm/api/account/create

#### Vaultwarden

Vaultwarden requires two secrets to be created manually: one for the PostgreSQL database and one for the admin token.

1. **Generate a secure admin token:**
   ```bash
   openssl rand -base64 48
   # Example output: jiQ1QYoUSsnJy2532Cq+Bk3AnL3bAEYYpv8tsIyEsY2Kma3LJ6K6d/bRBj+5zO4l
   ```

2. **Create PostgreSQL user secret:**
   ```bash
   kubectl create secret generic vaultwarden-postgres-user \
     --from-literal=username=vaultwarden \
     --from-literal=password=YOUR_SECURE_PASSWORD \
     --namespace personal
   ```

3. **Create admin token secret:**
   ```bash
   kubectl create secret generic vaultwarden-admin-token \
     --from-literal=adminToken=YOUR_GENERATED_TOKEN_HERE_FROM_STEP_1 \
     --namespace personal
   ```

**Important**: Replace `YOUR_SECURE_PASSWORD` with a strong password and `YOUR_GENERATED_TOKEN_HERE` with the token from step 1.

## Navigation

[← 02-bootstrap](../02-bootstrap/README.md) • [↑ Main project README](../README.md)
