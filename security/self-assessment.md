# OpenEBS Self-assessment

## Table of contents

- [OpenEBS Self-assessment](#openebs-self-assessment)
  - [Table of contents](#table-of-contents)
  - [Metadata](#metadata)
    - [Security links](#security-links)
  - [Overview](#overview)
    - [Background](#background)
    - [Actors](#actors)
    - [Actions](#actions)

## Metadata

### Security links

## Overview


### Background


### Actors

- **LocalPV Hostpath Provisioner:** A Kubernetes controller which serves PVs for LocalPV Hostpath PVCs. It creates/deletes Pods and PVs.
- **LocalPV Hostpath helper:** A Pod which handles creation/deletion for a LocalPV Hostpath volume. It runs with privileged access, mounts a Kubernetes hostPath. The path is pre-defined.
- **LocalPV ZFS Controller plugin:** A CSI-controller plugin which communicates with the Kubernetes API server to orchestrate volume provisioning, de-provisioning, expansion, snapshot operations for ZFS volumes on the Kubernetes cluster nodes.
- **LocalPV ZFS Node plugin:** A CSI-node plugin which uses a host's ZFS utils based RPC client to carry out volume provisioning, de-provisioning, expansion, snapshot operations for local ZFS volumes. It mounts hostpath directories on cluster hosts to enable communication with ZFS kernel modules and block device nodes.
- **LocalPV LVM Controller plugin:** A CSI-controller plugin which communicates with the Kubernetes API server to orchestrate volume provisioning, de-provisioning, expansion, snapshot creation for LVM volumes on the Kubernetes cluster nodes.
- **LocalPV LVM Node plugin:** A CSI-node plugin which uses in-built LVM RPC client to carry out volume provisioning, de-provisioning, expansion, snapshot creation for local ZFS volumes. It mounts hostpath directories on cluster hosts to enable communication with LVM kernel modules and block device nodes.
- **Replicated PV Mayastor Core Agent:** This is acts as a control-plane for a Mayastor cluster. Communitcates with other mayastor services via HTTP (gRPC).
- **Replicated PV Mayastor Etcd persistent store:** This persists the state of a Mayastor cluster. Uses replication and self-healing for redundancy and high-availability.
- **Replicated PV Mayastor HA Cluster Agent:** This is a Mayastor control-plane agent which provides highly available volume target management. This communicates to the Mayastor's core agent via HTTP (gRPC).
- **Replicated PV Mayastor HA Node Agent:** This is a Mayastor control-plane agent which mounts a hostpath directory and makes use of NVMe commands to execute volume target failovers.
- **Replicated PV Mayastor CSI Controller plugin:** This is a CSI-controller plugin which communicates with the Mayastor storage API (HTTP) and the Kubernetes APIs to orchestrate volume provisioning, de-provisioning, expansion, snapshot operations for Mayastor volumes
- **Replicated PV Mayastor CSI Node plugin:** This is a CSI-node plugin which communicates with the Mayastor control-plane via HTTP (gRPC) and executes host-level volumes operations. It mounts hostpath directories for accessing sysfs APIs and kernel device events.
- **Replicated PV Mayastor IO Engine:** This is a userspace storage controller which polls for IO requests and serves a volume target for Kubernetes containers. It consumes a high degree of CPU and memory resources to provide low-lantency, resilient storage. This communicates with the Mayastor control plane using HTTP (gRPC).
- **Replicated PV Mayastor IO Engine metrics exporter:** This exposes volume controller stats data in prometheus-compatible format. This communicates with IO engines using intra Pod IPC.
- **Replicated PV Mayastor Stats and Call-home plugin:** This is a plugin for reporting anonymous usage data from the Kubernetes cluster. It communicates with the Kubernetes API, and the Mayastor storage API to collect data.
- **Clients:** This actor interacts with an OpenEBS cluster using standard Kubernetes tools and/or specialised clients for accessing storage layer functionality. This is usually a Kubernetes cluster admin or a storage admin.

### Actions

- **PVC-PV based volume operations:** The OpenEBS cluster deployment registers provisioner plugin names with the Kubernetes cluster, and serves dynamic volume provisioning, de-provisioning, expansion, snapshot handling for different block and filesystem stacks. These are meant to plug into a Kubernetes cluster as a storage service. These services are accessible to Kubernetes cluster clients with adequate RBAC permissions. This is governed by a cluster administrator's RBAC configuration. The node-level plugins run as privileged containers to access system-software level OS APIs. The control-plane layers make use of Kubernetes primitives to ensure exclusive access to virtual storage devices:
  - LocalPV storage control plane uses Kubernetes NodeAffinityLabels to pin volumes to a single cluster node's host.
  - Replicated PV Mayastor uses Kuberentes VolumeAttachments to allow exclusive volume access (RWO mode) to a single Kubernetes node host.

- **Volume Access Control:** The Replicated PV CSI plugins make use of CSI volume mode SINGLE_NODE_WRITER and NVMe Reservations to ensure single-tenancy.
