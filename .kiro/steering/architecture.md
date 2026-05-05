# Architecture

## Overview

The Amazon EFS CSI Driver is a Kubernetes CSI (Container Storage Interface) plugin that enables EKS/K8s pods to mount Amazon EFS and Amazon S3 Files filesystems as persistent volumes. It runs as a DaemonSet (node plugin) and a Deployment (controller) in `kube-system`.

## Key Entry Points

- `cmd/main.go` — binary entry point, parses flags, initializes the `Driver`
- `pkg/driver/driver.go` — `Driver` struct, gRPC server setup, registers CSI services
- `pkg/driver/controller.go` — CSI Controller (CreateVolume/DeleteVolume via access points)
- `pkg/driver/node.go` — CSI Node (NodePublishVolume/NodeUnpublishVolume via efs-utils mount)

## Component Diagram

```
┌─────────────────────────────────────────────────────────┐
│  Kubernetes (kubelet)                                   │
│    ↓ CSI gRPC calls                                     │
├─────────────────────────────────────────────────────────┤
│  Driver (pkg/driver/)                                   │
│  ├── IdentityServer  — GetPluginInfo, Probe            │
│  ├── ControllerServer — CreateVolume, DeleteVolume      │
│  │     └── cloud.Cloud interface (access point CRUD)    │
│  └── NodeServer — NodePublish/Unpublish                 │
│        ├── Mounter (mount.efs via efs-utils)            │
│        └── EfsWatchdog (stunnel/TLS management)         │
├─────────────────────────────────────────────────────────┤
│  Cloud Layer (pkg/cloud/)                               │
│  ├── cloud.go — Cloud interface impl (EFS + S3Files)    │
│  ├── metadata.go — MetadataService (node ID, region)    │
│  └── retry_manager.go — adaptive retry for AWS APIs     │
├─────────────────────────────────────────────────────────┤
│  External Dependencies                                  │
│  ├── AWS EFS API (aws-sdk-go-v2/service/efs)            │
│  ├── AWS S3Files API (aws-sdk-go-v2/service/s3files)    │
│  ├── efs-utils (mount helper, bundled in container)     │
│  └── Kubernetes API (node taints, secrets)              │
└─────────────────────────────────────────────────────────┘
```

## Data Flow

**Dynamic provisioning (Controller):**
1. PVC created → Kubernetes calls `CreateVolume`
2. Controller validates parameters, resolves filesystem ID
3. Calls `cloud.CreateAccessPoint` (EFS or S3Files API)
4. Returns volume ID in format `efs:<fs-id>::<ap-id>` or `s3files:<fs-id>::<ap-id>`

**Mount (Node):**
1. Pod scheduled → kubelet calls `NodePublishVolume`
2. Node resolves mount target IP (from volume context or API)
3. Invokes `mount.efs` (efs-utils) with TLS, access point, and mount options
4. EfsWatchdog manages stunnel processes for encryption in transit

## Key Packages

| Package | Responsibility |
|---------|---------------|
| `pkg/driver/` | CSI gRPC handlers, mount logic, GID allocation, watchdog |
| `pkg/cloud/` | AWS API abstraction (EFS, S3Files, metadata), retry logic |
| `pkg/util/` | Shared utilities (endpoint parsing, filesystem type enum) |
| `test/e2e/` | Ginkgo E2E tests against real EKS clusters |
| `charts/` | Helm chart for deployment |
| `deploy/` | Kustomize manifests |

## Deployment Model

- **Controller** (Deployment, 2 replicas): handles `CreateVolume`/`DeleteVolume`, needs IAM for EFS/S3Files API access
- **Node** (DaemonSet): handles `NodePublishVolume`/`NodeUnpublishVolume`, runs efs-utils, needs host mount namespace
- Both share the same binary with different leader-election behavior

## Cross-Account Support

The driver supports mounting EFS filesystems from a different AWS account. The node assumes a cross-account IAM role (via STS) specified in a Kubernetes Secret referenced by the StorageClass or PV.
