# Azure Secure Cloud Network Foundation

An enterprise-grade, zero-trust network infrastructure built on Microsoft Azure. This project demonstrates core competencies in cloud networking, micro-segmentation, identity and access management (IAM), PaaS service isolation via Private Endpoints, and diagnostic troubleshooting using Azure Network Watcher—engineered entirely within strict cost-protection and free-tier boundaries.

---

## Architecture Overview

The topology implements a 3-tier segmented Virtual Network (`10.0.0.0/16`) designed to enforce least-privilege traffic flow and eliminate public IP exposure for sensitive backend and PaaS resources.

![Architecture Diagram](./diagrams/architechture-overview.drawio.svg)

### Network Topology Details
* **Virtual Network:** `vnet-foundation-eastus` (`10.0.0.0/16`)
* **Subnets:**
  * `snet-web` (`10.0.1.0/24`) — Public-facing web tier / ingress boundary.
  * `snet-app` (`10.0.2.0/24`) — Isolated application logic tier housing compute resources.
  * `snet-mgmt` (`10.0.3.0/24`) — Dedicated management and jumpbox subnet.
* **Storage Isolation:** Private Endpoint (`10.0.2.4`) attached to `stnetfoundation` Blob service with public access completely disabled.

---

## Security Architecture & Technical Design

### 1. Network Micro-Segmentation & NSG Logic
Each subnet is bounded by a dedicated Network Security Group (NSG). The Application Subnet (`nsg-app`) enforces explicit ingress filtering to prevent unauthorized lateral movement:

| Priority | Rule Name | Source | Destination | Port/Protocol | Action | Intent / Rationale |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **100** | `Allow-WebSubnet-Inbound` | `10.0.1.0/24` (Web) | `Any` | `80, 443 / TCP` | **Allow** | Permits necessary ingress web traffic from web tier. |
| **200** | `Deny-MgmtSubnet-Inbound` | `10.0.3.0/24` (Mgmt) | `Any` | `* / Any` | **Deny** | Enforces strict tier isolation; prevents lateral access from management. |
| **65000** | `AllowVnetInBound` | `VirtualNetwork` | `VirtualNetwork` | `* / Any` | **Allow** | Default Azure VNet inter-subnet system routing rule. |

### 2. PaaS Isolation via Private Endpoints
* **Public Access Block:** Storage account public endpoint disabled at network firewall layer.
* **Internal Routing:** Private Endpoint creates a Virtual Network Interface (NIC) inside `snet-app` (`10.0.2.4`) connected via Azure Private Link.
* **DNS Integration:** Utilizes `privatelink.blob.core.windows.net` Private DNS Zone for internal name resolution.

### 3. Identity & Role-Based Access Control (RBAC)
* Access constrained using **Azure RBAC** at the **Resource Group scope** (`rg-network-foundation-prod`).
* Implemented **Principle of Least Privilege (PoLP)** by assigning non-administrative **Reader** roles for audit and monitoring workflows.

---
