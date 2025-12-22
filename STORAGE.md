# Storage

```bash
                ┌───────────────────────────┐
                │       Proxmox Host        │
                │       (server)            │
                └─────────────┬─────────────┘
                              │
               ┌──────────────┴──────────────┐
               │        ZFS Pools            │
               │                              │
     ┌─────────┴─────────┐          ┌─────────┴─────────┐
     │   tank_secure      │          │   tank_media      │
     │  Mirror RAID       │          │  Striped RAID     │
     │  ~2.41 T free      │          │  ~5.78 T free    │
     └─────────┬─────────┘          └─────────┬─────────┘
               │                              │
       ┌───────┴────────┐             ┌───────┴────────┐
       │ tank_secure/main │            │ tank_media/main │
       │ dataset         │            │ dataset         │
       └───────┬─────────┘             └───────┬────────┘
               │                              │
       ┌───────┴────────┐             ┌───────┴────────┐
       │ VM Disk Image   │            │ VM Disk Image   │
       │ vm-101-disk-0   │            │ vm-101-disk-0   │
       │ ~1.98 T used    │            │ ~3.97 T used    │
       │ ~2.41 T free    │            │ ~1.81 T free    │
       └───────┬────────┘             └───────┬────────┘
               │                              │
       ┌───────┴────────┐             ┌───────┴────────┐
       │ Kubernetes PV  │             │ Kubernetes PV  │
       │ pv-secure       │             │ pv-media       │
       │ capacity 2 Ti   │             │ capacity 4 Ti  │
       │ access RWO      │             │ access RWO     │
       └────────────────┘             └────────────────┘

```