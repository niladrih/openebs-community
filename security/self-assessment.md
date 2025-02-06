# OpenEBS Self-assessment

## Table of contents

- [OpenEBS Self-assessment](#openebs-self-assessment)
  - [Table of contents](#table-of-contents)
  - [Metadata](#metadata)
    - [Security links](#security-links)
  - [Overview](#overview)
    - [Background](#background)
      - [Storage Architecture and Solutions](#storage-architecture-and-solutions)
    - [Actors](#actors)
    - [Actions](#actions)
      - [PVC-PV Based Volume Operations](#pvc-pv-based-volume-operations)
      - [Privileged Node-Level Operations](#privileged-node-level-operations)
      - [Control-Plane Coordination and Exclusive Volume Access](#control-plane-coordination-and-exclusive-volume-access)
      - [Integration with Kubernetes RBAC and Cluster Administration](#integration-with-kubernetes-rbac-and-cluster-administration)
  - [Goals and Non-Goals](#goals-and-non-goals)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
  - [Self-assessment use](#self-assessment-use)
  - [Security Functions and Features](#security-functions-and-features)
  - [Secure development practices](#secure-development-practices)
  - [Security issue resolution](#security-issue-resolution)
  - [Appendix](#appendix)

## Metadata

|   |  |
| -- | -- |
| Assessment Stage | Incomplete | 
| Software | https://github.com/openebs/openebs  |
| Website | https://openebs.io/ |
| Security Provider | No  |
| Languages | Go, Rust |
| SBOM | - |
| | |

### Security links

## Overview

OpenEBS is an open-source storage solution designed specifically for Kubernetes environments. It provides a comprehensive set of storage controllers and management systems that enable organizations to handle storage in a cloud-native way. Through OpenEBS, development/operations teams can dynamically provision storage resources and manage storage operations across different types of storage systems, all while working seamlessly within their Kubernetes infrastructure. This storage orchestration works much like how Kubernetes orchestrates containers, bringing the same level of automation and flexibility to storage management that teams have come to expect from container management.

### Background

Kubernetes has revolutionized application deployment by providing a declarative orchestration layer for cloud-native workloads. Through its API objects, Kubernetes abstracts away complex management tasks, allowing developers to focus on their applications rather than infrastructure details. However, when applications need persistent storage, the management complexity increases significantly, particularly for stateful workloads that must maintain data across container restarts and migrations.
OpenEBS addresses this challenge by extending Kubernetes' declarative approach to storage management. It implements storage interface standards that seamlessly integrate with Kubernetes' native storage plugin architecture. This integration enables administrators to manage storage resources with the same declarative principles they use for managing applications.

#### Storage Architecture and Solutions

OpenEBS offers two primary approaches to storage provisioning, each designed for different use cases:

**Local Volume Architecture:**
Local volumes in OpenEBS provide a streamlined storage solution by creating storage resources directly on the Kubernetes node where the workload runs. This architecture leverages existing storage systems on the host machine, offering several advantages:
The direct connection between storage and compute minimizes latency and maximizes performance. This makes local volumes ideal for applications that handle their own data replication, such as distributed databases like MongoDB or key-value stores like Redis. The lightweight configuration reduces operational overhead while maintaining the benefits of dynamic provisioning.

**Replicated Storage Architecture:**
For workloads requiring enterprise-grade storage features, OpenEBS provides replicated storage capabilities. This solution deploys dedicated storage controllers within the Kubernetes cluster to manage data replication and availability. The replicated storage architecture delivers:

Built-in data redundancy across nodes
Automated failure detection and recovery
Dynamic scaling of storage resources
Advanced storage features typically found in enterprise storage systems

This approach particularly benefits applications that need guaranteed data persistence and high availability but don't implement their own replication mechanisms.

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

#### PVC-PV Based Volume Operations

- **Volume Provisioning, De-provisioning, Expansion, and Snapshot Handling:**
  - **What Happens:**  
    - OpenEBS registers its provisioner plugin names with the Kubernetes API, enabling dynamic provisioning and de-provisioning of Persistent Volumes (PVs) based on Persistent Volume Claims (PVCs).
    - The control-plane components (e.g., LocalPV storage control plane and Replicated PV Mayastor CSI Controller plugin) manage volume lifecycle events for various block and filesystem stacks.
    - These operations extend to volume expansion and snapshot creation/management, ensuring data consistency and availability.
  - **Security Considerations:**  
    - Access to these operations is governed by Kubernetes RBAC. Only clients with appropriate permissions can trigger these actions.
    - The control-plane components leverage Kubernetes primitives such as NodeAffinityLabels (for LocalPV) and VolumeAttachments (for Replicated PV Mayastor) to ensure volumes are pinned to a specific node and to enforce exclusive access.
  
- **Volume Access Control and Single-Tenancy:**
  - **What Happens:**  
    - The Replicated PV CSI plugins implement access control measures using CSI volume mode settings (e.g., `SINGLE_NODE_WRITER`) to ensure that a volume is attached exclusively to a single node.
    - NVMe Reservations are used for Replicated PV Mayastor to enforce single-tenancy at the block device level.
  - **Security Considerations:**  
    - These mechanisms prevent unauthorized concurrent access, reducing the risk of data corruption or leakage.
    - They ensure that volume access is strictly controlled and isolated at the hardware level, reinforcing the overall security posture of the storage service.

#### Privileged Node-Level Operations

- **Privileged Container Operations:**
  - **What Happens:**  
    - Node-level plugins, including those for LocalPV and Replicated PV Mayastor, run as privileged containers. This enables them to access system-level OS APIs and execute low-level operations such as direct I/O on block devices and hostPath mounts.
  - **Security Considerations:**  
    - Because these containers run with elevated privileges, their actions are highly security-sensitive.
    - Isolation is maintained by restricting these operations to only trusted components and by using stringent RBAC policies to control access to these node-level operations.

#### Control-Plane Coordination and Exclusive Volume Access

- **Control-Plane Coordination:**
  - **What Happens:**  
    - The OpenEBS control plane leverages Kubernetes APIs and primitives to manage storage resources, ensuring that volume operations (creation, expansion, deletion) are performed in an orderly and coordinated fashion.
    - This includes the use of NodeAffinityLabels to pin LocalPV volumes to a single node and the use of VolumeAttachments for Mayastor volumes to enforce exclusive (read-write-once) access.
  - **Security Considerations:**  
    - These coordination mechanisms ensure that conflicting operations are prevented, thereby reducing the risk of data inconsistency or access violations.
    - By enforcing strict volume placement and attachment policies, OpenEBS minimizes the risk of unauthorized multi-node access to storage resources.

#### Integration with Kubernetes RBAC and Cluster Administration

- **RBAC-Governed Operations:**
  - **What Happens:**  
    - OpenEBS exposes its storage services (provisioning, expansion, snapshot management) to the Kubernetes cluster. The access to these services is controlled by Kubernetes Role-Based Access Control (RBAC) configurations defined by the cluster administrator.
  - **Security Considerations:**  
    - This ensures that only authenticated and authorized users or systems can initiate critical storage actions.
    - The integration with Kubernetes RBAC provides an additional layer of security by enforcing least privilege principles for both control-plane and node-level operations.

## Goals and Non-Goals

### Goals

- **Data Security and Integrity:**  
  Ensure that data managed by OpenEBS is protected from unauthorized access and corruption. This includes securing data at rest and in transit and ensuring that storage operations (provisioning, expansion, snapshot management) are performed securely.

- **Resilience and Availability:**  
  Provide a highly available storage solution that maintains data durability across Kubernetes clusters. OpenEBS achieves this by using mechanisms such as volume replication, Kubernetes-native scheduling (e.g., NodeAffinityLabels and VolumeAttachments), and single-node writer guarantees to avoid concurrent access issues.

- **Secure Dynamic Storage Operations:**  
  Offer secure mechanisms for dynamically provisioning, expanding, and snapshotting volumes. These operations are integrated with Kubernetes RBAC and other access controls to ensure that only authorized users and systems can perform critical actions.

- **Access Control and Isolation:**  
  Enforce strict volume access controls using CSI volume modes (e.g., `SINGLE_NODE_WRITER`), NVMe Reservations, and Kubernetes primitives to ensure that volumes are exclusively attached and isolated, thereby minimizing risks of unauthorized concurrent access.

- **Compliance with Kubernetes Security Standards:**  
  Align with Kubernetes security best practices by integrating with Kubernetes’ RBAC and leveraging standard APIs for storage operations. This ensures that storage services provided by OpenEBS adhere to the security requirements of modern Kubernetes clusters.

### Non-Goals

- **Primary Database Functionality:**  
  OpenEBS is not designed to function as a primary database or data processing system. Its focus is on providing persistent storage for Kubernetes workloads rather than managing or processing data directly.

- **Comprehensive Data Security Solution:**  
  While OpenEBS implements robust security measures for storage operations, it does not provide a complete data security solution. Additional security measures (e.g., network security, encryption at rest, end-to-end encryption) may be required based on user requirements.

- **Broad Infrastructure Management:**  
  OpenEBS does not manage the underlying network infrastructure, container runtime security, or host operating system hardening. Its scope is limited to storage provisioning and management within the Kubernetes environment.

- **Supply Chain and Build Security:**  
  This assessment does not cover aspects of the OpenEBS build process, dependency management, or container image supply chain security. It focuses solely on the runtime security posture of the storage operations.

- **Advanced Data Processing or Analytics:**  
  OpenEBS does not include integrated data processing, advanced query capabilities, or native analytics functions. Its role is to provide reliable and secure storage services, leaving data processing to external systems.

## Self-assessment use

This self-assessment is created by the OpenEBS team to perform an internal analysis of the project’s security. It is not intended to provide a security audit of OpenEBS, or function as an independent assessment or attestation of OpenEBS’s security health.

This document serves to provide OpenEBS users with an initial understanding of OpenEBS’s security, where to find existing security documentation, OpenEBS plans for security, and general overview of OpenEBS security practices, both for development of OpenEBS as well as security of OpenEBS.

This document provides the CNCF TAG-Security with an initial understanding of OpenEBS to assist in a joint-assessment, necessary for projects under incubation. Taken together, this document and the joint-assessment serve as a cornerstone for if and when OpenEBS seeks graduation and is preparing for a security audit.

## Security Functions and Features

- **RBAC & Access Control:**  
  - Enforces Kubernetes RBAC for secure storage operations.
  - Uses CSI volume modes and NVMe Reservations to guarantee exclusive volume access.

- **Secure Communication:**  
  - Utilizes secure HTTP/gRPC channels between control-plane and node components.
  - Designed for integration with external encryption solutions.

- **Isolation & Privilege Separation:**  
  - Runs node-level plugins in tightly controlled, privileged containers.
  - Separates control-plane and node functions to minimize risk exposure.

- **Data Integrity & Resilience:**  
  - Implements secure volume lifecycle management (provisioning, expansion, deletion, snapshots).
  - Supports replication and failover to ensure high availability and durability.

- **Monitoring & Telemetry:**  
  - Exposes metrics via Prometheus-compatible endpoints.
  - Provides audit and telemetry data to support ongoing security evaluation.

- **Kubernetes Ecosystem Integration:**  
  - Leverages native Kubernetes APIs for consistent security policy enforcement.
  - Modular design enables easy integration with additional security tools.

## Secure development practices

- **Peer Review & Static Analysis:**  
  All code changes undergo mandatory peer reviews and automated static analysis to catch security issues early.

- **Comprehensive Testing:**  
  Extensive unit and integration tests validate functionality and secure behavior.

- **Automated CI/CD Pipelines:**  
  Secure pipelines enforce coding standards, run vulnerability scans, and perform dependency checks before deployment.

- **Adherence to Secure Coding Standards:**  
  The project follows best practices in error handling, input validation, and prevention of common vulnerabilities.

- **Dependency and Supply Chain Management:**  
  Regular audits ensure third-party libraries are secure and up to date, reducing supply chain risks.

- **Active Vulnerability Management:**  
  A structured process is in place for reporting, triaging, and resolving security vulnerabilities with timely updates.

## Security issue resolution

## Appendix