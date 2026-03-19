# Microsoft Fabric Capacity Autoscaling Setup Guide

This guide walks through setting up the Fabric Capacity Scaling notebook, which automatically scales a Microsoft Fabric capacity between SKUs (e.g., F64 → F128) using Azure Resource Manager.

---

## Architecture Overview

The solution uses a **Fabric notebook** that:

1. Retrieves Service Principal credentials from **Azure Key Vault**
2. Authenticates to Azure using the Service Principal
3. Reads the current Fabric capacity SKU via the **Azure Resource Manager API**
4. Sends a PATCH request to scale the capacity to the target SKU
5. Polls until the scaling operation completes

---

## Prerequisites

- An **Azure subscription** with an existing Microsoft Fabric capacity (e.g., F64)
- A **Microsoft Fabric workspace** with notebook capabilities
- Access to **Microsoft Entra ID** (to create an App Registration)
- Access to **Azure Key Vault** (to store secrets)
- **Owner** or **Contributor** role on the Fabric capacity Azure resource

---

## Step 1: Create a Service Principal (App Registration)

1. Go to the **Azure Portal** → **Microsoft Entra ID** → **App registrations**
2. Click **New registration**
   - **Name:** e.g., `Fabric-Capacity-Scaler`
   - **Supported account types:** Single tenant
   - Click **Register**
3. Note the following values (you will store these in Key Vault):
   - **Application (client) ID** — from the Overview page
   - **Directory (tenant) ID** — from the Overview page
4. Go to **Certificates & secrets** → **New client secret**
   - Set a description and expiration (recommended: 12–24 months)
   - **Copy the secret value immediately** — it will not be shown again
5. **Add at least two owners** to the App Registration to prevent lockout if one person leaves the organization

> **Important:** Track the client secret expiration date and rotate it before it expires.

---

## Step 2: Grant the Service Principal Azure RBAC Permissions

The Service Principal needs permission to manage the Fabric capacity resource.

1. Go to the **Azure Portal** → navigate to your **Fabric capacity resource**
   - Path: Subscriptions → \<your subscription\> → Resource Groups → \<your resource group\> → \<Fabric capacity\>
2. Click **Access control (IAM)** → **Add role assignment**
3. Assign the **Contributor** role to the Service Principal (search by the App Registration name)
4. Click **Review + assign**

---

## Step 3: Set Up Azure Key Vault

### Create the Key Vault (if one doesn't already exist)

1. Go to the **Azure Portal** → **Key Vaults** → **Create**
2. Choose a name, region, and resource group
3. Under **Access configuration**, choose either:
   - **Azure role-based access control (RBAC)** (recommended), or
   - **Vault access policy**

### Store the Service Principal secrets

Create three secrets in the Key Vault:

| Secret Name        | Value                                      |
| ------------------ | ------------------------------------------ |
| `sp-tenant-id`     | The Directory (tenant) ID from Step 1      |
| `sp-client-id`     | The Application (client) ID from Step 1    |
| `sp-client-secret`  | The client secret value from Step 1        |

### Grant Key Vault access

The following identities need **Get** permission on secrets:

| Identity | Why |
| -------- | --- |
| **Each Fabric user** who will run or schedule the notebook | `mssparkutils.credentials.getSecret()` authenticates as the calling user |
| **A security group** containing all admins (recommended) | Prevents lockout when individuals leave |

**If using RBAC:** Assign the **Key Vault Secrets User** role.  
**If using access policies:** Add a policy with **Get** secret permission.

> **Recommendation:** Use an **Entra security group** for Key Vault access instead of individual users. This way, removing or adding team members only requires updating group membership.

---

## Step 4: Configure the Fabric Workspace

### Add the Service Principal to the workspace

1. Open the **Fabric portal** → navigate to your workspace
2. Click **Manage access** (gear icon or "..." menu)
3. Add the Service Principal (App Registration name) as an **Admin** (required — the notebook acquires a Power BI API token that needs admin-level workspace access)

### Import the notebook

1. In your Fabric workspace, click **Import** → **Notebook**
2. Upload the `ScaleUp-F64_to_F128-ARM-Template-Final.ipynb` file

---

## Step 5: Configure the Notebook Parameters

Open the notebook and update the following cells:

### Cell: Azure Resource Information

```python
SUBSCRIPTION_ID = "<your-azure-subscription-id>"
RESOURCE_GROUP  = "<your-resource-group-name>"
CAPACITY_NAME   = "<your-fabric-capacity-name>"
TARGET_SKU      = "F128"          # Target SKU to scale to
API_VERSION     = "2023-11-01"    # ARM API version
```

- **SUBSCRIPTION_ID:** Found in Azure Portal → Subscriptions
- **RESOURCE_GROUP:** The resource group containing your Fabric capacity
- **CAPACITY_NAME:** The name of your Fabric capacity resource (not the display name)
- **TARGET_SKU:** The SKU to scale to (e.g., `F64`, `F128`, `F256`)

### Cell: Key Vault Configuration

```python
keyVaultEndpoint = '<your-key-vault-name>'
```

Replace with the **name** of your Azure Key Vault (not the full URL).

---

## Step 6: Test the Notebook

1. Open the notebook in the Fabric workspace
2. Run all cells sequentially
3. Verify the output at each step:
   - **"Assign service principal credential information"** cell — should retrieve secrets from Key Vault without errors
   - **"Acquire Tokens and create the API headers"** cell — should authenticate successfully
   - **"Verify the Current SKU"** cells — should print the current SKU (e.g., `Current SKU: F64`)
   - **"Scale Up to F128"** cell — should either scale the capacity or print "No action needed"

---

## Step 7: Schedule the Notebook (Optional)

To run the notebook on a schedule (e.g., scale up during business hours):

1. Create a **Data Pipeline** in your Fabric workspace
2. Add a **Notebook activity** that points to this notebook
3. Configure a **Schedule trigger** with the desired recurrence

> **Important:** The pipeline runs under the identity of the user who configures the schedule. If that user's account is deactivated, the schedule will stop working. See the Maintenance section below.

---

## Creating a Scale-Down Notebook

To scale back down (e.g., F128 → F64 after business hours), duplicate the notebook and change:

```python
TARGET_SKU = "F64"
```

Schedule this notebook to run after business hours via a second pipeline trigger.

---

## Maintenance & Operations

### Rotating the Client Secret

Client secrets expire. Before expiration:

1. Go to **Entra ID** → **App registrations** → your app → **Certificates & secrets**
2. Create a **new** client secret
3. Update the `sp-client-secret` value in **Azure Key Vault**
4. Delete the old secret from Entra ID after confirming the notebook works

### Admin Turnover (Avoiding Lockout)

If the person who set this up leaves the organization:

| Action | Who Can Do It |
| ------ | ------------- |
| Reassign workspace admin | Fabric tenant admin (Admin Portal → Workspaces) |
| Re-create the pipeline schedule | New workspace admin |
| Grant Key Vault access to new admin | Key Vault owner or RBAC admin |
| Add new owner to App Registration | Existing App Registration owner or Global Admin |
| Rotate client secret | App Registration owner |

### Recommended Resilience Practices

- **Use Entra security groups** for Key Vault access, workspace roles, and App Registration ownership — never rely on a single user
- **Document the App Registration name and Key Vault name** in a shared team location
- **Set calendar reminders** for client secret expiration dates
- **Ensure at least two people** are owners of the App Registration and admins of the Fabric workspace

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
| ------- | ------------ | --- |
| `getSecret()` fails with 403 | Running user lacks Key Vault access | Grant Key Vault Secrets User role |
| `401 Unauthorized` on ARM call | Client secret expired or SP lacks RBAC | Rotate secret / check IAM role |
| Capacity not found | Wrong `SUBSCRIPTION_ID`, `RESOURCE_GROUP`, or `CAPACITY_NAME` | Verify values in Azure Portal |
| Scheduled run fails after admin leaves | Schedule bound to deactivated user | Re-create schedule under active user |
| Scale times out | Capacity in transitional state | Wait and retry; check Azure Service Health |

---

## Required Azure Resource Values Reference

Collect these values before starting setup:

| Value | Where to Find It |
| ----- | ---------------- |
| Subscription ID | Azure Portal → Subscriptions |
| Resource Group | Azure Portal → Resource Groups |
| Fabric Capacity Name | Azure Portal → Microsoft Fabric → Capacities |
| Key Vault Name | Azure Portal → Key Vaults |
| Tenant ID | Entra ID → Overview |
| Client ID | Entra ID → App registrations → Overview |
| Client Secret | Entra ID → App registrations → Certificates & secrets |
