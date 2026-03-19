# Microsoft Fabric Capacity Autoscaling Setup Guide

This guide covers setting up the Fabric Capacity Scaling notebook, which automatically scales a Microsoft Fabric capacity between SKUs (e.g., F64 → F128) using Azure Resource Manager.

---

## Architecture Overview

The solution uses a **Fabric notebook** that:

1. Authenticates using the **workspace managed identity** (no secrets, no user dependency)
2. Reads the current Fabric capacity SKU via the **Azure Resource Manager API**
3. Sends a PATCH request to scale the capacity to the target SKU
4. Polls until the scaling operation completes

### Why Managed Identity?

| | Old Approach (Service Principal + Key Vault) | Best Practice (Workspace Managed Identity) |
|-|----------------------------------------------|-------------------------------------------|
| Secrets to manage | 3 (tenant ID, client ID, client secret) | **None** |
| Key Vault required | Yes | **No** |
| Secret expiration risk | Yes (must rotate) | **No** |
| Breaks when user leaves | Yes (Key Vault + schedule) | **No** |
| User identity dependency | Yes (getSecret + pipeline) | **No** |

---

## Prerequisites

- An **Azure subscription** with an existing Microsoft Fabric capacity
- A **Microsoft Fabric workspace** with workspace identity enabled
- **Contributor** role on the Fabric capacity Azure resource (granted to the workspace identity)

---

## Step 1: Enable Workspace Identity

1. Open the **Fabric portal** → navigate to your workspace
2. Go to **Workspace settings** → **Identity**
3. Enable the **workspace identity**
4. Note the identity name — you will need it for the RBAC assignment

> If workspace identity is not available, ensure your Fabric tenant admin has enabled this feature in the admin portal.

---

## Step 2: Grant the Workspace Identity Azure RBAC Permissions

The workspace identity needs permission to manage the Fabric capacity resource.

1. Go to the **Azure Portal** → navigate to your **Fabric capacity resource**
   - Path: Subscriptions → \<your subscription\> → Resource Groups → \<your resource group\> → \<Fabric capacity\>
2. Click **Access control (IAM)** → **Add role assignment**
3. Select the **Contributor** role
4. Under **Members**, choose **Managed identity** → **Select members**
5. Find your workspace identity by name and select it
6. Click **Review + assign**

---

## Step 3: Import the Notebook

1. In your Fabric workspace, click **Import** → **Notebook**
2. Upload the `ScaleUp-Fabric-Capacity-BestPractice.ipynb` file

---

## Step 4: Configure the Notebook Parameters

Open the notebook and update the configuration cell:

```python
SUBSCRIPTION_ID = "<your-azure-subscription-id>"
RESOURCE_GROUP  = "<your-resource-group-name>"
CAPACITY_NAME   = "<your-fabric-capacity-name>"
TARGET_SKU      = "F128"        # Target SKU to scale to
API_VERSION     = "2023-11-01"  # ARM API version
```

- **SUBSCRIPTION_ID:** Found in Azure Portal → Subscriptions
- **RESOURCE_GROUP:** The resource group containing your Fabric capacity
- **CAPACITY_NAME:** The name of your Fabric capacity resource
- **TARGET_SKU:** The SKU to scale to (e.g., `F2`, `F4`, `F8`, `F16`, `F32`, `F64`, `F128`, `F256`)

---

## Step 5: Test the Notebook

1. Open the notebook in the Fabric workspace
2. Run all cells sequentially
3. Verify the output at each step:
   - **Step 2 (Authenticate)** — should print "Authenticated using workspace managed identity."
   - **Step 3 (Verify Current SKU)** — should print the capacity name, resource ID, and current SKU
   - **Step 4 (Scale)** — should either scale the capacity or print "No action needed"

---

## Step 6: Schedule the Notebook

To run the notebook on a schedule (e.g., scale up during business hours):

1. Create a **Data Pipeline** in your Fabric workspace
2. Add a **Notebook activity** pointing to this notebook
3. Configure a **Schedule trigger** with the desired recurrence

> Because the notebook uses the **workspace identity** (not the user's identity), the schedule will continue to work regardless of who created it or whether their account is active.

---

## Creating a Scale-Down Notebook

To scale back down after business hours, duplicate the notebook and change:

```python
TARGET_SKU = "F64"
```

Schedule this via a second pipeline trigger.

---

## Maintenance & Operations

### What Requires Maintenance

| Component | Action | Frequency |
|-----------|--------|-----------|
| Workspace identity RBAC | Verify role assignment still exists | Annually or after subscription changes |
| Notebook parameters | Update if capacity resource moves | As needed |

### What Does NOT Require Maintenance

- **No secrets to rotate** — managed identity handles authentication
- **No Key Vault to manage** — no secrets stored anywhere
- **No schedule to re-create** — not tied to a user identity
- **No App Registration to maintain** — no Service Principal

### Admin Turnover

When someone leaves the organization:
- **Nothing breaks** — the workspace identity is tied to the workspace, not a person
- A new workspace admin only needs to be assigned by a Fabric tenant admin if the departing user was the sole admin

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
| ------- | ------------ | --- |
| `getToken()` fails | Workspace identity not enabled | Enable in Workspace Settings → Identity |
| `403 Forbidden` on ARM call | Workspace identity lacks RBAC | Grant Contributor on the capacity resource |
| `404 Not Found` | Wrong `SUBSCRIPTION_ID`, `RESOURCE_GROUP`, or `CAPACITY_NAME` | Verify values match the Azure Portal |
| Scale times out | Capacity in transitional state | Wait and retry; check Azure Service Health |
| `429 Too Many Requests` | ARM rate limiting | Built-in retry logic handles this automatically |

---

## Required Azure Resource Values Reference

Collect these values before starting setup:

| Value | Where to Find It |
| ----- | ---------------- |
| Subscription ID | Azure Portal → Subscriptions |
| Resource Group | Azure Portal → Resource Groups |
| Fabric Capacity Name | Azure Portal → Microsoft Fabric → Capacities |

---

## Migrating from the Service Principal Version

If you were previously using the Service Principal + Key Vault version of this notebook:

1. **Enable workspace identity** (Step 1 above)
2. **Grant RBAC** to the workspace identity (Step 2 above)
3. **Import the new notebook** and configure the parameters
4. **Update or re-create your pipeline** to point to the new notebook
5. **Verify it works** by running manually

After confirming the new version works, you can optionally clean up:
- Remove the old notebook from the workspace
- Remove the Service Principal's RBAC role assignment on the capacity
- Delete the Key Vault secrets (`sp-tenant-id`, `sp-client-id`, `sp-client-secret`) if no longer needed
- Delete the App Registration if it was only used for this purpose
