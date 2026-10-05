# Azure Secure Cloud Network Foundation — Step-by-Step Deployment Guide

This guide provides step-by-step instructions for deploying, validating, and tearing down the **Azure Secure Cloud Network Foundation** project.

---

## Prerequisite: Cost Governance Setup

Before deploying infrastructure, configure spending safeguards.

### Step 1: Create a Budget Alert
1. In the Azure Portal search bar, search for **Cost Management + Billing**.
2. Select **Budgets** under *Cost management* in the left menu.
3. Click **+ Add**.
4. Set the scope to your subscription.
5. Enter Budget Name: `Project-Budget-5USD`.
6. Set **Amount**: `$5.00` (Monthly).
7. Under **Alert conditions**, set thresholds:
   * `50%` ($2.50)
   * `80%` ($4.00)
   * `100%` ($5.00)
8. Enter your notification email address and click **Create**.

### Step 2: Create Target Resource Group
1. Search for **Resource groups** $\rightarrow$ Click **+ Create**.
2. **Subscription:** Choose your active subscription.
3. **Resource group:** `rg-network-foundation-prod`
4. **Region:** `East US` (or closest region).
5. Click **Review + create** $\rightarrow$ **Create**.

---

## Phase 1: Virtual Network & Subnet Provisioning

1. Search for **Virtual networks** $\rightarrow$ Click **+ Create**.
2. **Basics Tab:**
   * **Resource group:** `rg-network-foundation-prod`
   * **Name:** `vnet-foundation-eastus`
   * **Region:** `East US`
3. **IP Addresses Tab:**
   * Set **IPv4 address space**: `10.0.0.0/16`
   * Remove any default subnet configurations.
   * Add the following subnets:
     * **Subnet 1:** Name = `snet-web` | Address range = `10.0.1.0/24`
     * **Subnet 2:** Name = `snet-app` | Address range = `10.0.2.0/24`
     * **Subnet 3:** Name = `snet-mgmt` | Address range = `10.0.3.0/24`
4. Click **Review + create** $\rightarrow$ **Create**.

---

## Phase 2: Network Security Groups (NSGs) & Micro-Segmentation

### Step 1: Create NSG Resources
Create three separate Network Security Groups in `rg-network-foundation-prod`:
1. `nsg-web`
2. `nsg-app`
3. `nsg-mgmt`

### Step 2: Configure Rules on `nsg-app`
1. Navigate to **Network security groups** $\rightarrow$ **`nsg-app`**.
2. Under **Settings**, click **Inbound security rules** $\rightarrow$ Click **+ Add**.

**Rule 1 — Allow Inbound Web Subnet Traffic:**
* **Source:** `IP Addresses`
* **Source IP addresses:** `10.0.1.0/24`
* **Source port ranges:** `*`
* **Destination:** `Any`
* **Service:** `Custom` | **Destination port ranges:** `80,443`
* **Protocol:** `TCP`
* **Action:** `Allow`
* **Priority:** `100`
* **Name:** `Allow-WebSubnet-Inbound`
* Click **Add**.

**Rule 2 — Deny Inbound Management Subnet Traffic:**
* Click **+ Add** again.
* **Source:** `IP Addresses`
* **Source IP addresses:** `10.0.3.0/24`
* **Source port ranges:** `*`
* **Destination:** `Any`
* **Service:** `Custom` | **Destination port ranges:** `*`
* **Protocol:** `Any`
* **Action:** `Deny`
* **Priority:** `200`
* **Name:** `Deny-MgmtSubnet-Inbound`
* Click **Add**.

### Step 3: Associate NSGs to Subnets
1. In `nsg-app`, go to **Subnets** $\rightarrow$ Click **+ Associate**.
2. Select `vnet-foundation-eastus` $\rightarrow$ Select `snet-app` $\rightarrow$ Click **OK**.
3. Repeat association for `nsg-web` $\rightarrow$ `snet-web` and `nsg-mgmt` $\rightarrow$ `snet-mgmt`.

---

## Phase 3: PaaS Storage Isolation via Private Endpoint

### Step 1: Deploy Storage Account
1. Search for **Storage accounts** $\rightarrow$ Click **+ Create**.
2. **Resource Group:** `rg-network-foundation-prod`
3. **Storage account name:** `stnetfoundation` + *[your-initials]*
4. **Region:** `East US`
5. **Primary service:** Azure Blob Storage | **Performance:** Standard | **Redundancy:** LRS.
6. **Networking Tab:**
   * Select **Disable public access and use private endpoints**.
7. Click **Review + create** $\rightarrow$ **Create**.

### Step 2: Configure Private Endpoint
1. Go to your new Storage Account $\rightarrow$ **Networking** $\rightarrow$ **Private endpoint connections** tab.
2. Click **+ Private endpoint**.
3. **Name:** `pe-storage-blob`
4. **Target sub-resource:** `blob`
5. **Networking Tab:**
   * **Virtual network:** `vnet-foundation-eastus`
   * **Subnet:** `snet-app`
6. **Private DNS integration:** Set to **Yes** (Integrates with `privatelink.blob.core.windows.net`).
7. Click **Review + create** $\rightarrow$ **Create**.

---

## Phase 4: Compute Provisioning & Diagnostics Verification

### Step 1: Deploy Lightweight Test VM
1. Search for **Virtual machines** $\rightarrow$ Click **+ Create** $\rightarrow$ **Azure virtual machine**.
2. **Resource group:** `rg-network-foundation-prod`
3. **VM name:** `vm-app-01`
4. **Region:** `East US`
5. **Size:** `Standard_B1s` (1 vCPU, 1 GiB RAM)
6. **Administrator account:** Select Username/Password authentication.
7. **Networking Tab:**
   * **Virtual network:** `vnet-foundation-eastus`
   * **Subnet:** `snet-app` (`10.0.2.0/24`)
   * **Public IP:** Select **None**
8. Click **Review + create** $\rightarrow$ **Create**.

### Step 2: Validate Traffic Rules (IP Flow Verify)
1. Search for **Network Watcher** in the main search bar.
2. Select **IP flow verify** under *Network diagnostic tools*.
3. **Target Resource:** Select `vm-app-01` and its attached NIC interface.

**Test Case A (Allowed Connection):**
* **Protocol:** `TCP` | **Direction:** `Inbound`
* **Local Port:** `80` | **Remote IP:** `10.0.1.5` | **Remote Port:** `80`
* Click **Verify IP flow**.
* **Expected Result:** `Access allowed` matching `Allow-WebSubnet-Inbound`.

**Test Case B (Denied Connection):**
* Change **Remote IP** to `10.0.3.5`.
* Click **Verify IP flow**.
* **Expected Result:** `Access denied` matching `Deny-MgmtSubnet-Inbound`.

### Step 3: Validate Private Endpoint Routing (Next Hop)
1. Inside **Network Watcher**, select **Next hop**.
2. **Target Resource:** Select `vm-app-01`.
3. **Source IP address:** `10.0.2.5` (VM IP).
4. **Destination IP address:** `10.0.2.4` (Private Endpoint IP).
5. Click **Next hop**.
6. **Expected Result:** **Next hop type:** `PrivateEndpoint`.

---

## Phase 5: Resource Cleanup & Teardown

To avoid incurring unexpected passive charges:

1. Search for **Resource groups** $\rightarrow$ Select `rg-network-foundation-prod`.
2. Click **Delete resource group** on the top action bar.
3. Type `rg-network-foundation-prod` into the text box to confirm.
4. Click **Delete**.
