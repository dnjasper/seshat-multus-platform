# Seshat 5G Telco Cloud — Platform, Networking & GitOps

![System Architecture](https://raw.githubusercontent.com/dnjasper/seshat-infra-platform/main/architecture.png)

This repository is the **Kubernetes platform, networking, observability, and GitOps layer** of a three-repository cloud-native 5G/Telco Cloud lab.

It sits between the AWS/EKS infrastructure provisioned by Repo1 and the 5G workloads maintained in Repo3.

The platform uses Flux to continuously reconcile Kubernetes resources while Multus provides secondary network interfaces required by the 5G user plane.

## Architecture Role

The project is separated into three repositories:

- **Repo1 — Infrastructure & Orchestration:** AWS, VPC, EKS, IAM, OIDC, ECR, Terraform, and infrastructure CI/CD.
- **Repo2 — Platform, Networking & GitOps (this repository):** Flux, Multus, secondary networking, IPAM, monitoring, and workload delivery definitions.
- **Repo3 — 5G Workloads:** Open5GS, UERANSIM, UPF, N6 gateway, vDU, and supporting application workloads.

Repo2 provides the Kubernetes platform services and GitOps control layer connecting the infrastructure to the telecom workloads.

## GitOps Architecture

Flux is bootstrapped against this repository and continuously reconciles the desired cluster state.

```text
GitHub
  |
  v
seshat-multus-platform
  |
  v
Flux
  |
  +---- Multus CNI
  |
  +---- N3 / N6 Networks
  |
  +---- Monitoring
  |
  +---- Open5GS Delivery
  |
  +---- UPF Delivery
  |
  +---- UERANSIM Delivery
  |
  +---- N6 Gateway Delivery
  |
  +---- vDU Delivery
```

Application workloads remain separated in Repo3 while Repo2 defines how Flux delivers those workloads into the cluster.

## Kubernetes Secondary Networking

The environment uses **Multus CNI** to provide telecom workloads with additional network interfaces beyond the default Kubernetes pod interface.

This allows the Kubernetes management/application network and the 5G user-plane networks to remain logically separated.

### N3 User Plane

The N3 NetworkAttachmentDefinition provides the network between the simulated gNB and UPF.

```text
NetworkAttachmentDefinition: telco-n3-plane
CNI: IPVLAN
Mode: L2
Host interface: ens6
IPAM: Whereabouts
Subnet: 192.168.100.0/24
Allocation range: 192.168.100.20 - 192.168.100.100
```

This network carries the N3 user-plane path used for GTP-U traffic between UERANSIM and the UPF.

### N6 Data Network

The N6 NetworkAttachmentDefinition provides the external data-network side of the UPF.

```text
NetworkAttachmentDefinition: telco-n6-plane
CNI: IPVLAN
Mode: L2
Host interface: ens7
IPAM: host-local
Subnet: 192.168.200.0/24
Allocation range: 192.168.200.150 - 192.168.200.200
```

The N6 network connects the UPF toward the N6 gateway and external network path.

## 5G Network Path

The platform networking supports the broader 5G environment:

```text
UERANSIM UE
     |
     v
UERANSIM gNB
     |
     | N3 / GTP-U
     | Multus + IPVLAN
     v
Open5GS UPF
     |
     | N6
     | Multus + IPVLAN
     v
N6 Gateway
     |
     v
External Network / Internet
```

Control-plane connectivity is handled separately by the Open5GS components deployed through Repo3.

## Platform Components

### Multus CNI

Multus provides multiple network interfaces to Kubernetes pods.

The Multus deployment includes:

- ServiceAccount
- ClusterRole
- ClusterRoleBinding
- DaemonSet
- NetworkAttachmentDefinitions
- Supporting configuration

### IP Address Management

The platform contains Whereabouts resources and CRDs for cluster-aware secondary network IP allocation.

N3 currently uses Whereabouts IPAM.

N6 uses host-local IPAM with a dedicated allocation range.

### Monitoring

Monitoring is deployed through Helm and includes the Prometheus/Grafana monitoring stack.

The monitoring configuration is maintained declaratively through Git and reconciled by Flux.

This provides observability for Kubernetes infrastructure and the telecom workloads running in the cluster.

## Workload Delivery Bridges

Repo2 contains dedicated Flux delivery definitions for the major workload groups maintained outside the platform repository:

```text
apps/
├── n6-gateway-bridge
├── open5gs-bridge
├── ueransim-bridge
├── upf-bridge
└── vdu-bridge
```

These resources keep platform ownership separated from application ownership while allowing Flux to manage the complete environment.

## Repository Structure

```text
.
├── apps
│   ├── monitoring
│   ├── multus-cni
│   ├── n6-gateway-bridge
│   ├── open5gs-bridge
│   ├── ueransim-bridge
│   ├── upf-bridge
│   ├── vdu-bridge
│   └── whereabouts
│
└── clusters
    └── my-cluster
        ├── flux-kustomization.yaml
        └── flux-system
            ├── gotk-components.yaml
            ├── gotk-sync.yaml
            └── kustomization.yaml
```

## Flux Bootstrap

Flux is bootstrapped against this repository:

```bash
flux bootstrap github \
  --owner=dnjasper \
  --repository=seshat-multus-platform \
  --branch=main \
  --path=./clusters/my-cluster \
  --read-write-key=true \
  --personal \
  --components-extra=image-reflector-controller,image-automation-controller
```

After bootstrap, Flux becomes responsible for reconciling the platform configuration from Git.

## Technology Stack

| Area | Technology |
|---|---|
| Container Orchestration | Kubernetes / Amazon EKS |
| GitOps | Flux |
| Secondary Networking | Multus CNI |
| Data Plane Networking | IPVLAN L2 |
| N3 IPAM | Whereabouts |
| N6 IPAM | host-local |
| Monitoring | Prometheus / Grafana |
| Package Deployment | Helm / Flux HelmRelease |
| Configuration | Kubernetes YAML / Kustomize |

## Project Context

This repository demonstrates the Kubernetes platform layer required to operate cloud-native telecom workloads rather than treating Kubernetes only as an application hosting platform.

The environment combines GitOps, observability, multi-interface pod networking, IP address management, Linux networking, and 5G workload delivery.

The broader project is used to practice telecom integration and troubleshooting across Kubernetes, Linux networking, AWS infrastructure, and 5G control/user-plane components.
