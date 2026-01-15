# Fabric Capacity Autoscale Automation (F64 ⇄ F128)
**Disclaimer: The attached diagrams and code are provided AS IS without warranty of any kind and should not be interpreted as an offer or commitment on the part of Microsoft, and Microsoft cannot guarantee the accuracy of any information presented. MICROSOFT MAKES NO WARRANTIES, EXPRESS OR IMPLIED, IN THIS DIAGRAM(s) CODE SAMPLE(s).**

## Overview

This repository documents an automated approach for **scaling Microsoft Fabric capacity** using Fabric notebooks and the Azure Resource Manager (ARM) API.

The solution enables **time-based autoscaling** (for example, scaling up during business hours and scaling down after hours) using secure, non-interactive authentication.

The implementation uses:
- A **Service Principal** (Microsoft Entra ID)
- **Azure RBAC** for capacity management
- **Azure Key Vault** for secret storage
- **Fabric Notebooks** for execution and scheduling

Two notebooks are used:
- **Scale Up**: F64 → F128
- **Scale Down**: F128 → F64

Each notebook is independently schedulable.

---

## High-Level Architecture

```
Fabric Notebook
   |
   |-- MSAL Authentication (Service Principal)
   |
Azure Resource Manager (ARM)
   |
   |-- PATCH Microsoft.Fabric/capacities
   |
Fabric Capacity (SKU Updated)
```

---

## Step 1: Create a Service Principal (Entra ID)

1. Navigate to **Microsoft Entra ID → App registrations**
2. Create a new app registration (single tenant)
3. Record the following values:
   - **Tenant ID**
   - **Client ID**
4. Create a **Client Secret**
   - Copy the secret value immediately (it cannot be retrieved later)

This service principal is used by the Fabric notebooks to authenticate to Azure Resource Manager.

---

## Step 2: Assign Azure RBAC Permissions

The service principal must have permission to modify the Fabric capacity **Azure resource**.

### Recommended scope (least privilege)
- The **Fabric capacity resource**, or
- The **Resource Group** containing the Fabric capacity

### Required permission
The role must include:

```
Microsoft.Fabric/capacities/write
```

Common options:
- Assign **Contributor** at the capacity or resource group scope
- Or create a **custom RBAC role** scoped to Fabric capacity operations

Without this permission, scaling attempts will fail with `AuthorizationFailed`.

---

## Step 3: Store Secrets in Azure Key Vault

To support unattended automation, all configuration and credentials are stored in **Azure Key Vault** and retrieved at runtime.

### Required Key Vault secrets

| Secret Name | Description |
|------------|-------------|
| `TENANT_ID` | Entra tenant ID |
| `CLIENT_ID` | Service principal client ID |
| `CLIENT_SECRET` | Service principal secret |

These secrets are retrieved in Fabric notebooks using:
- Fabric Key Vault references, or
- `mssparkutils.credentials.getSecret(...)`

No secrets are hardcoded in notebooks.

---

## Step 4: Fabric Notebook Logic

Both notebooks (scale up and scale down) follow the same logic. The only difference is the **target SKU**.

### 1. Define the target SKU

```python
TARGET_SKU = "F64"   # or "F128" for scale-up
API_VERSION = "2023-11-01"
```

---

### 2. Authenticate to Azure Resource Manager

- Uses **MSAL** with the service principal
- Requests an access token for:

```
https://management.azure.com/.default
```

This token is used for all ARM API calls.

---

### 3. Retrieve the current capacity state

The notebook:
- Calls ARM to retrieve the Fabric capacity resource
- Reads the current SKU (for example, `F128`)

If the capacity is already at the target SKU, the notebook exits without making changes.

---

### 4. Update the Fabric capacity SKU

If a change is required, the notebook sends an ARM request:

```http
PATCH
/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.Fabric/capacities/{capacityName}
```

Request body:

```json
{
  "sku": {
    "name": "F64",
    "tier": "Fabric"
  }
}
```

Azure processes the SKU change asynchronously.

---

## Step 5: Automation & Scheduling

Each notebook is scheduled independently within Fabric.

| Notebook | Typical Schedule |
|--------|------------------|
| Scale Up (F64 → F128) | Weekdays, start of business hours |
| Scale Down (F128 → F64) | Weekdays, end of business hours |

This approach enables:
- High performance during peak usage
- Cost optimization during off-hours
- Fully automated, repeatable scaling

---

## Key Design Decisions

- **Two separate notebooks**
  - Clear intent (scale up vs scale down)
  - Simple scheduling
  - Easier auditing and troubleshooting
- **ARM-based scaling**
  - Uses the official Azure control plane
- **Service principal + Key Vault**
  - Secure, enterprise-ready automation
  - No embedded credentials

---

## Common Troubleshooting

### AuthorizationFailed
- Service principal does not have sufficient RBAC permissions on the capacity or resource group

### Notebook runs but capacity does not change
- Incorrect subscription ID, resource group, or capacity name
- Incorrect ARM API version
- Capacity is already scaling or at the target SKU

### Scaling takes several minutes
- Capacity scaling is an asynchronous Azure resource update by design

---

## Summary

This automation pattern provides:
- Deterministic Fabric capacity scaling
- Secure, credential-free execution
- Enterprise-ready scheduling
- Clear separation of scale-up and scale-down logic

It is suitable for **production Fabric environments** where capacity requirements vary predictably over time.
