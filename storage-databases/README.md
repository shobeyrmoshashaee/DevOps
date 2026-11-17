# 💾 Storage & Databases

## Overview

This area contains storage and database engineering projects focused on availability, performance, backup, recovery, and production operations.

## Scope

- Block, file, and object-storage architectures
- Longhorn, Ceph, Rook Ceph, NFS, and S3-compatible object storage
- Database deployment, replication, clustering, and high availability
- Backup, snapshot, restore, migration, and disaster recovery
- Capacity planning, performance tuning, and storage observability
- Stateful workloads on virtual machines, containers, and Kubernetes

## Projects

| Project | Installation Method | Description |
|---|---|---|
| [Kubernetes NFS Dynamic Provisioning](./kubernetes-nfs-subdir-provisioner-helm/) | Helm | NFS server preparation, official NFS Subdirectory External Provisioner chart, StorageClass, PVC testing, and operations |

## Project Organization

Each storage project uses an independent subdirectory with a dedicated README, sanitized source files, deployment instructions, validation, troubleshooting, and operational guidance.

> [!NOTE]
> The future manifest-based NFS provisioner project will be documented separately and clearly labeled as the **Deployment-based** installation method.
