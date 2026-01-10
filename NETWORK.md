## Network Topology

This homelab setup uses two separate networks on the Proxmox host: a main LAN for host access and a private NAT bridge for Talos cluster VMs.

             Internet
                 │
           ┌─────────────┐
           │ Home Router │ 192.168.1.1
           └─────┬───────┘
                 │ LAN (192.168.1.0/24)
                 │
           ┌─────────────┐
           │ Proxmox Host│
           │  vmbr0:     │ 192.168.1.55
           │  vmbr1:     │ 192.168.100.1 (NAT)
           └─────┬───────┘
                 │
      ┌──────────┴──────────┐
      │                     │
  [LAN VMs]             [Talos VMs]
   (vmbr0)              (vmbr1: 192.168.100.0/24)
                        Gateway: 192.168.100.1
                        NAT + IPv4 forwarding via host


### Key Points

- **Proxmox Host LAN (`vmbr0`)**
  - Static IP: `192.168.1.55/24`
  - Gateway: `192.168.1.1`
  - Provides access to home network and internet

- **Isolated Talos Network (`vmbr1`)**
  - Subnet: `192.168.100.0/24`
  - Gateway: `192.168.100.1` (host bridge)
  - Host performs NAT + IPv4 forwarding
  - Talos VMs have static IPs in this subnet

### Cluster Access from LAN

To allow devices on the main LAN (`192.168.1.0/24`) to seamlessly and securely access services running inside the isolated cluster (`192.168.100.0/24`), we use a combination of a local DNS server and firewall rules on the Proxmox host. This approach avoids any special configuration (like static routes) on client devices.

#### 1. DNS Resolution

A local DNS server (e.g., Pi-hole) must be configured on the LAN. This server is responsible for resolving cluster service hostnames (like `argocd.cluster`) to the **Proxmox host's IP address**: `192.168.1.55`.

#### 2. Port Forwarding (NAT) on Proxmox Host

The Proxmox host acts as a gateway, forwarding specific ports from the LAN to the cluster's ingress controller (`192.168.100.80`). This is configured using `iptables` rules directly on the Proxmox host.

Run these commands on the Proxmox host shell:

```bash
# 1. Forward incoming HTTP/S traffic (ports 80/443) to the ingress controller (DNAT)
# This rule redirects traffic destined for the Proxmox host to the cluster's ingress controller.
sudo iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 80 -j DNAT --to-destination 192.168.100.80
sudo iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 443 -j DNAT --to-destination 192.168.100.80

# 2. Forward Kubernetes API traffic (external port 6443) to the API server VIP (DNAT)
# This allows kubectl and k9s to connect to the cluster.
sudo iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 6443 -j DNAT --to-destination 192.168.100.50:6443

# 3. Forward SFTP traffic (external port 2222) to the SFTP service (DNAT)
# This allows SFTP clients to connect to the SFTP server.
sudo iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 2222 -j DNAT --to-destination 192.168.100.81:22

# 4. Fix the return address for all forwarded traffic (SNAT)
# These rules ensure replies from the cluster appear to come from the Proxmox host,
# preventing client connection issues.
sudo iptables -t nat -A POSTROUTING -p tcp --dport 80 -d 192.168.100.80 -j SNAT --to-source 192.168.100.1
sudo iptables -t nat -A POSTROUTING -p tcp --dport 443 -d 192.168.100.80 -j SNAT --to-source 192.168.100.1
sudo iptables -t nat -A POSTROUTING -p tcp --dport 6443 -d 192.168.100.50 -j SNAT --to-source 192.168.100.1
sudo iptables -t nat -A POSTROUTING -p tcp --dport 22 -d 192.168.100.81 -j SNAT --to-source 192.168.100.1
```

To make these rules persist after a reboot, install `iptables-persistent` and save the rules:

```bash
# Install the persistence package (if not already installed)
sudo apt-get update
sudo apt-get install iptables-persistent

# Save the currently active rules
sudo netfilter-persistent save
```

- **VM attachment**
  - Talos VMs → `vmbr1`
  - LAN/other VMs → `vmbr0`

- **Verified functionality**
  - Talos VMs can ping the Proxmox host via vmbr1
  - Talos VMs have internet access via NAT
  - Workstations on the LAN can access cluster services via DNS without client-side configuration.

### Assigned IP Addresses

The following static IP addresses are assigned from the `192.168.100.0/24` subnet, as defined in `01-infra/variables.tf`.

| Role                       | IP Address          | Description                               |
| -------------------------- | ------------------- | ----------------------------------------- |
| Control Plane Node         | `192.168.100.60`    | `talos-controlplane-01`                   |
| Worker Node                | `192.168.100.70`    | `talos-worker-01`                         |
| Kubernetes VIP             | `192.168.100.50`    | Virtual IP for the Kubernetes API server. |
| Ingress Load Balancer Pool | `192.168.100.80-85` | Range available for LoadBalancer services. |

---

## Tailscale Integration

Tailscale operator is deployed in the `tailscale` namespace and provides secure access to cluster services via the tailnet. This complements the existing LAN access method by providing secure, certificate-based access from anywhere.

### Tailscale Ingress Configuration

Services are exposed to the tailnet using Tailscale Ingress resources. This provides automatic HTTPS certificates and secure access without exposing services to the public internet.

**Example Tailscale Ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: navidrome-tailscale
  namespace: media
spec:
  ingressClassName: tailscale
  tls:
    - hosts:
        - navidrome
  defaultBackend:
    service:
      name: navidrome
      port:
        number: 4533
```

**Key Points:**
- `ingressClassName: tailscale` - Uses Tailscale operator
- `tls.hosts: ["servicename"]` - Sets hostname (becomes `servicename.tailce4610.ts.net`)
- `defaultBackend` - Points to the internal service
- HTTPS certificates are automatically provisioned
- Requires HTTPS enabled on tailnet in admin console

### Access Methods

#### Tailnet Access (Secure, from anywhere)
Services are accessible via: `https://servicename.tailce4610.ts.net`

Current services:
- Navidrome: `https://navidrome.tailce4610.ts.net` ✅
- ArgocD: `https://argocd.tailce4610.ts.net` ✅
- Longhorn: `https://longhorn.tailce4610.ts.net` ✅

All services have been successfully configured with Tailscale ingress and are accessible via HTTPS with automatic certificates.

#### Internal Cluster Access (Within cluster)
Services also have internal ingress resources using `ingressClassName: nginx` for access within the cluster.

**Example Internal Ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: navidrome
  namespace: media
  annotations:
    cert-manager.io/cluster-issuer: ingress
spec:
  ingressClassName: nginx
  rules:
    - host: navidrome.cluster
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: navidrome
                port:
                  number: 4533
```

#### LAN Access (From local network)
Traditional access via Proxmox host NAT forwarding (as documented in previous sections):
- Navidrome: `http://navidrome.cluster` (via DNS + NAT)

### Prerequisites for Tailscale Ingress

1. **Tailscale Operator**: Deployed via Helm in `tailscale` namespace
2. **OAuth Client**: Configured with proper scopes (`Devices Core`, `Auth Keys`, `Services`)
3. **HTTPS Enabled**: Must be enabled in Tailscale admin console
4. **MagicDNS**: Enabled for automatic DNS resolution

### Troubleshooting Tailscale

**Common Issues:**

1. **HTTPS Not Enabled**: Check admin console, required for Tailscale ingress
2. **Proxy Pod Not Running**: Check `tailscale` namespace for proxy pods
3. **Service Not Accessible**: Verify service endpoints and proxy pod logs
4. **DNS Resolution**: Ensure MagicDNS is enabled

**Useful Commands:**

```bash
# Check Tailscale ingress status
kubectl get ingress -A | grep tailscale

# Check proxy pods
kubectl get pods -n tailscale -l "tailscale.com/parent-resource-type=ingress"

# Check proxy pod logs
kubectl logs -n tailscale ts-servicename-ingress-xxx-0

# Describe ingress for details
kubectl describe ingress servicename-tailscale -n namespace
```

### Service Configuration Pattern

For each service that needs tailnet access:

1. Create Tailscale ingress resource in service namespace
2. Use pattern: `servicename-tailscale` for ingress name
3. Point to existing internal service
4. Add to kustomization.yaml resources
5. Keep existing nginx ingress for internal cluster access
6. Continue using LAN access method for local users

### LoadBalancer Alternative

LoadBalancer services can also be used with `loadBalancerClass: tailscale`, but ingress method is preferred for:
- Automatic HTTPS certificates
- Better resource management  
- Easier DNS configuration

---

### Future Network Architecture (With Custom Router)

The current setup relies on the Proxmox host to perform routing and NAT via `iptables`, which is complex and centralizes network management on the hypervisor. The future plan is to delegate these responsibilities to a dedicated custom router.

This new architecture will involve:

1.  **Centralized Routing & Firewalling:** A custom router (e.g., pfSense, OPNsense) will manage all network traffic, replacing the `iptables` rules on the Proxmox host.
2.  **VLAN Segmentation:** The network will be segmented into multiple VLANs (e.g., `VLAN 10` for user devices, `VLAN 20` for servers).
3.  **Proxmox Simplification:** The `vmbr1` NAT bridge will be removed. The main bridge (`vmbr0`) will be made VLAN-aware, and VMs will be assigned to VLANs via a "VLAN Tag" in their network settings.
4.  **Direct Access with DNS:** The router will handle all DNS. Service hostnames (`argocd.cluster`, etc.) will resolve directly to the service's true IP address within the Server VLAN, rather than pointing to the Proxmox host.
5.  **Improved Security & Management:** Network access rules will be managed centrally on the router's firewall, providing a clearer and more robust security posture.

This change will simplify the Proxmox host's role to be purely a hypervisor and create a more scalable and professionally structured network.