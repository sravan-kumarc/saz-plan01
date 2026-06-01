# GitHub Actions OIDC Authentication with Azure

## Goal

Authenticate GitHub Actions to Azure without using client secrets.

Architecture:

```text
GitHub Actions
      ↓
OIDC Token
      ↓
Microsoft Entra ID
      ↓
Federated Credential
      ↓
Service Principal
      ↓
Azure Subscription
```

Benefits:

* No Client Secret
* No Secret Rotation
* More Secure
* Microsoft Recommended
* GitHub Recommended

---

# Step 1: Create App Registration

Azure Portal:

```text
Microsoft Entra ID
    ↓
App Registrations
    ↓
New Registration
```

Example:

```text
terraform-sp-sk
```

Save:

```text
Application (Client) ID
Tenant ID
```

---

# Step 2: Create Service Principal

CLI:

```bash
az ad sp create --id <CLIENT_ID>
```

Verify:

```bash
az ad sp show --id <CLIENT_ID>
```

---

# Step 3: Assign Subscription Permissions

Azure Portal:

```text
Subscriptions
    ↓
Access Control (IAM)
    ↓
Add Role Assignment
```

Role:

```text
Contributor
```

Assign to:

```text
terraform-sp-sk
```

For production, use least privilege.

---

# Step 4: Configure Federated Credential

Azure Portal:

```text
Entra ID
    ↓
App Registrations
    ↓
terraform-sp-sk
    ↓
Certificates & Secrets
    ↓
Federated Credentials
    ↓
Add Credential
```

Select:

```text
GitHub Actions deploying Azure resources
```

Fill:

```text
Organization:
sravan-kumarc

Repository:
saz-plan01

Entity Type:
Branch

Branch:
main
```

Azure automatically generates:

```text
repo:sravan-kumarc/saz-plan01:ref:refs/heads/main
```

IMPORTANT:

The generated subject must exactly match the GitHub workflow subject claim.

---

# Step 5: GitHub Repository Secrets

Repository:

```text
Settings
    ↓
Secrets and Variables
    ↓
Actions
```

Add:

```text
AZURE_CLIENT_ID
AZURE_TENANT_ID
AZURE_SUBSCRIPTION_ID
```

Do NOT create:

```text
AZURE_CLIENT_SECRET
```

OIDC removes the need for client secrets.

---

# Step 6: GitHub Workflow Permissions

Required:

```yaml
permissions:
  id-token: write
  contents: read
```

Without id-token permission OIDC will fail.

---

# Step 7: Azure Login Action

```yaml
- name: Azure Login
  uses: azure/login@v3
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

# Step 8: Verify Authentication

```yaml
- name: Verify Azure Login
  run: az account show
```

Expected:

```json
{
  "name": "Azure subscription 1",
  "state": "Enabled"
}
```

---

# Full Working Example

```yaml
name: Azure OIDC Login

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:
  test-oidc:

    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v3
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Verify Login
        run: az account show

      - name: List Resource Groups
        run: az group list --output table
```

---

# Common Errors

## Error

```text
AADSTS70025
No configured federated identity credentials
```

Cause:

```text
Missing Federated Credential
```

Fix:

```text
App Registration
    ↓
Federated Credentials
    ↓
Add GitHub Credential
```

---

## Error

```text
Subject claim does not match
```

Cause:

```text
Wrong Repository
Wrong Branch
Wrong Organization
```

Verify:

```text
repo:<org>/<repo>:ref:refs/heads/<branch>
```

matches exactly.

---

## Error

```text
Insufficient privileges
```

Cause:

```text
No RBAC role assignment
```

Fix:

```text
Subscription
    ↓
IAM
    ↓
Contributor
```

---

# Production Flow

```text
GitHub Actions
      ↓
OIDC
      ↓
Azure Login
      ↓
Terraform
      ↓
ACR
      ↓
Container Apps
      ↓
AKS
```

This is the recommended authentication model for modern Azure CI/CD.
