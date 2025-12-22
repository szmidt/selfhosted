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

- **Workstation access**
  - Add a static route to reach Talos nodes from your workstation:
    ```powershell
    route -p add 192.168.100.0 mask 255.255.255.0 192.168.1.55
    ```

- **VM attachment**
  - Talos VMs → `vmbr1`
  - LAN/other VMs → `vmbr0`

- **Verified functionality**
  - Talos VMs can ping the Proxmox host via vmbr1
  - Talos VMs have internet access via NAT
  - Workstations can reach Talos nodes via the static route

