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

# 2. Fix the return address for forwarded traffic (SNAT)
# This rule ensures replies from the cluster appear to come from the Proxmox host,
# preventing client connection issues.
sudo iptables -t nat -A POSTROUTING -p tcp --dport 80 -d 192.168.100.80 -j SNAT --to-source 192.168.100.1
sudo iptables -t nat -A POSTROUTING -p tcp --dport 443 -d 192.168.100.80 -j SNAT --to-source 192.168.100.1
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