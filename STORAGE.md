# Storage

This document outlines the storage architecture, from the underlying physical disks on the Proxmox host to the persistent storage available within the Kubernetes cluster.

## Host-Level Storage

The Proxmox host provides physical disks for VM storage.

## In-Cluster Storage (Longhorn)

Persistent storage for applications running inside the Kubernetes cluster is provided by [Longhorn](https://longhorn.io/).

### Architecture

-   The `talos-worker-01` node is provisioned with two dedicated 5.5TB virtual disks from the Proxmox host (`data1` and `data2`).
-   Inside the Talos OS, these disks are mounted at `/var/mnt/sdb` and `/var/mnt/sdc`.
-   Longhorn is configured to use these two mount points as its storage locations, providing a total of 11TB of raw persistent storage capacity to the cluster.

### Redundancy

To provide data safety on a single-node cluster, Longhorn is configured with the following settings:

-   **Replica Count:** By default, Longhorn creates 2 replicas for each Persistent Volume (PV).
-   **Replica Scheduling:** The `replicaSoftAntiAffinity` setting is enabled. This tells Longhorn to try to schedule replicas on different disks. Since the worker node has two disks, each replica of a volume will be placed on a separate physical disk, protecting against a single disk failure.