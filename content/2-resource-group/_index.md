---
title: "IDENTITY & Resource Group Foundation"
date: 2025-08-30T11:10:00+07:00
weight: 2
chapter: false
pre: "<b>2. </b>"
---

Trong phần này, chúng ta sẽ tạo nền tảng cơ bản cho hệ thống MLOps bằng cách thiết lập Resource Group và phân quyền Identity & Access Management (IAM). Resource Group sẽ là container logic để quản lý tập trung các tài nguyên Azure liên quan đến dự án, trong khi IAM đảm bảo việc kiểm soát truy cập và bảo mật theo nguyên tắc least privilege. Đây là bước đầu tiên quan trọng trước khi triển khai các thành phần MLOps khác như Azure ML Workspace, Storage, và Compute Resources.

## 1. Mục tiêu

✅ **Tạo Resource Group riêng cho môi trường development**  
✅ **Thiết lập tagging strategy chuẩn enterprise**  
✅ **Cấu hình RBAC permissions phù hợp**  
✅ **Đảm bảo cost tracking và governance**

<div style="background: linear-gradient(135deg, #e6f3ff 0%, #dbeafe 100%); border: 1px solid #bfdbfe; border-left: 6px solid #0078d4; padding: 18px; margin: 18px 0; border-radius: 10px;">
  <strong>TASK 2 — IDENTITY & RESOURCE GROUP</strong><br>
  <strong>Mục tiêu</strong>: RG + AAD/RBAC, tạm 4 người = Contributor ở scope RG. UI: Resource groups → Create retail-dev-rg (+ Tags). Entra ID: Users/Groups (CloudEngineer, DataEngineer, Analyst, Owners). App Registration (Service Principal) cho DevOps. Khi có AML WS: gán MI quyền AcrPull (ACR), Storage Blob Data Reader, Key Vault Secrets User, ML Workspace Contributor. Bằng chứng: Ảnh RG/Tags; Users/Groups; IAM assignments; App Registration. Done: 4 user OK; SP OK; quyền gán xong.
</div>

## 2. RESOURCE GROUP SETUP

### 2.1 Basic Configuration

<div style="background: #f8fafc; border: 1px solid #e2e8f0; border-left: 4px solid #0078d4; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>🔧 Resource Group Settings</strong>

**1. Basic Configuration:**
- **Resource Group name**: `retail-dev-rg`
- **Subscription**: [Your Azure subscription]
- **Region**: Southeast Asia (southeastasia)
- **Purpose**: Container cho tất cả Azure resources của MLOps project

**2. Naming Convention:**
- **Format**: `{project}-{environment}-rg`
- **Benefits**: Dễ dàng identify, automate, và manage lifecycle
- **Consistency**: Sẽ có `retail-staging-rg`, `retail-prod-rg` sau này
</div>

### 2.2 UI Flow (Azure Portal)

**Step 1: Navigate to Resource Groups**
```
Azure Portal → Resource groups → Create resource group
```

**Step 2: Basic Configuration**
- **Subscription**: [Your subscription]
- **Resource group name**: `retail-dev-rg`
- **Region**: Southeast Asia

**Step 3: Tags Configuration**
- **Environment**: `development`
- **Project**: `retail-forecast`
- **Owner**: `mlops-team`
- **CostCenter**: `ml-engineering`
- **Purpose**: `mlops-platform`
- **DataClassification**: `internal`
- **CreatedBy**: `{your-email}`

**Step 4: Review & Create**
```bash
# Xem lại cấu hình
Subscription: [Your Subscription]
Resource Group: retail-dev-rg  
Region: Southeast Asia
Tags: 7 tags configured

# Click "Create" để tạo Resource Group
```

### 2.3 Tags Strategy

<div style="background: #fff7ed; border: 1px solid #fed7aa; border-left: 4px solid #f59e0b; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>🏷️ Enterprise Tagging Strategy</strong>

**Required Tags cho MLOps Project:**

| Tag Name | Value | Purpose | Example |
|----------|-------|---------|---------|
| `Environment` | `development` | Môi trường deployment | dev/staging/prod |
| `Project` | `retail-forecast` | Tên dự án | retail-forecast |
| `Owner` | `mlops-team` | Team sở hữu | mlops-team |
| `CostCenter` | `ml-engineering` | Cost allocation | ml-engineering |
| `CreatedBy` | `{your-email}` | Người tạo | user@company.com |
| `Purpose` | `mlops-platform` | Mục đích sử dụng | mlops-platform |
| `DataClassification` | `internal` | Phân loại dữ liệu | internal/confidential |

</div>

## 3. IDENTITY & ACCESS MANAGEMENT

### 3.1 Entra ID Users & Groups Setup

**User Groups cần tạo:**

<div style="background: #e6f3ff; border: 1px solid #b3d9ff; border-left: 4px solid #0078d4; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>👥 Team Structure</strong>

**1. CloudEngineer Group:**
- **Role**: Infrastructure management, DevOps
- **Members**: 1-2 Cloud Engineers
- **Scope**: Resource Group Contributor

**2. DataEngineer Group:**
- **Role**: Data pipeline, ETL processes
- **Members**: 1-2 Data Engineers  
- **Scope**: Resource Group Contributor

**3. Analyst Group:**
- **Role**: Data analysis, reporting
- **Members**: 1-2 Data Analysts
- **Scope**: Resource Group Contributor

**4. Owners Group:**
- **Role**: Project management, oversight
- **Members**: 1-2 Project Owners
- **Scope**: Resource Group Contributor

</div>

**UI Flow:**
```
Azure Portal → Azure Active Directory → Groups → New group
```

**Group Configuration:**
- **Group type**: Security
- **Group name**: `{role}-retail-dev` (e.g., `CloudEngineer-retail-dev`)
- **Description**: `{role} team for retail forecast development`
- **Members**: Add users to respective groups

### 3.2 Service Principal (App Registration)

**Purpose**: DevOps automation, CI/CD pipelines

**UI Flow:**
```
Azure Portal → Azure Active Directory → App registrations → New registration
```

**Configuration:**
- **Name**: `sp-retail-devops`
- **Supported account types**: Single tenant
- **Redirect URI**: Not required for service principal

**API Permissions:**
- `Azure Service Management`: `user_impersonation`
- `Microsoft Graph`: `Directory.Read.All`

**Certificates & Secrets:**
- Create client secret (expires in 12 months)
- Save secret value for CI/CD pipeline

## 4. RBAC PERMISSIONS SETUP

### 4.1 Resource Group Level Permissions

**Assign Contributor role cho 4 groups:**

<div style="background: #f1f5f9; border: 1px solid #e2e8f0; border-left: 4px solid #6366f1; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>🔑 RBAC Assignments</strong>

**UI Flow:**
```
Resource Group retail-dev-rg → Access control (IAM) → Add role assignment
```

**Role Assignments:**
- **CloudEngineer-retail-dev** → `Contributor`
- **DataEngineer-retail-dev** → `Contributor`  
- **Analyst-retail-dev** → `Contributor`
- **Owners-retail-dev** → `Contributor`
- **sp-retail-devops** → `Contributor`

**Scope**: Resource Group level (inherits to all resources)

</div>

### 4.2 Azure ML Workspace Managed Identity Permissions

**Khi có Azure ML Workspace, gán cho Managed Identity:**

<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>🤖 ML Workspace MI Permissions</strong>

**Required Role Assignments:**

| Resource | Role | Purpose |
|----------|------|---------|
| **Azure Container Registry** | `AcrPull` | Pull container images |
| **Storage Account** | `Storage Blob Data Reader` | Read training data |
| **Key Vault** | `Key Vault Secrets User` | Access secrets/certificates |
| **Azure ML Workspace** | `Machine Learning Workspace Contributor` | Manage ML resources |

**Scope**: Specific resource level (not Resource Group)

</div>

## 5. CLI COMMANDS (REFERENCE)

### 5.1 Azure CLI Setup

```bash
# Login to Azure
az login

# Set subscription
az account set --subscription "your-subscription-id"

# Create Resource Group with tags
az group create \
  --name retail-dev-rg \
  --location southeastasia \
  --tags \
    Environment=development \
    Project=retail-forecast \
    Owner=mlops-team \
    CostCenter=ml-engineering \
    Purpose=mlops-platform \
    DataClassification=internal \
    CreatedBy=$(az account show --query user.name --output tsv)
```

### 5.2 RBAC Assignments

```bash
# Get Resource Group ID
RG_ID=$(az group show --name retail-dev-rg --query id --output tsv)

# Assign Contributor role to groups
for GROUP in "CloudEngineer-retail-dev" "DataEngineer-retail-dev" "Analyst-retail-dev" "Owners-retail-dev"; do
  az role assignment create \
    --assignee $GROUP \
    --role "Contributor" \
    --scope $RG_ID
done

# Assign Contributor to Service Principal
az role assignment create \
  --assignee $(az ad sp list --display-name sp-retail-devops --query '[0].objectId' -o tsv) \
  --role "Contributor" \
  --scope $RG_ID
```

### 5.3 Service Principal Creation

```bash
# Create Service Principal
APP=$(az ad sp create-for-rbac \
  --name sp-retail-devops \
  --role Contributor \
  --scopes $RG_ID \
  --sdk-auth)

# Save credentials
echo "$APP" > sp-retail-devops.json

# Login using Service Principal
az login --service-principal \
  -u $(jq -r .clientId sp-retail-devops.json) \
  -p $(jq -r .clientSecret sp-retail-devops.json) \
  --tenant $(jq -r .tenantId sp-retail-devops.json)
```

## 6. BẰNG CHỨNG HOÀN THÀNH

### 6.1 Ảnh 1: Resource Group Overview
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📋 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Trang Overview của <code>retail-dev-rg</code></li>
<li>✅ Hiển thị Region: <code>Southeast Asia</code></li>
<li>✅ Status: <code>Active</code></li>
<li>✅ Subscription name visible</li>
</ul>
</div>

### 6.2 Ảnh 2: Tags Configuration
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>🏷️ Yêu cầu screenshot:</strong>
<ul>
<li>✅ Tab "Tags" của Resource Group</li>
<li>✅ Hiển thị đầy đủ 7 tags đã cấu hình</li>
<li>✅ Values chính xác theo bảng trên</li>
</ul>
</div>

### 6.3 Ảnh 3: Entra ID Users/Groups
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>👥 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Azure AD → Groups</li>
<li>✅ Hiển thị 4 groups: CloudEngineer, DataEngineer, Analyst, Owners</li>
<li>✅ Group names và member counts visible</li>
</ul>
</div>

### 6.4 Ảnh 4: App Registration
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>🔐 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Azure AD → App registrations</li>
<li>✅ Application: <code>sp-retail-devops</code></li>
<li>✅ Client ID và tenant ID visible</li>
<li>✅ Certificates & secrets configured</li>
</ul>
</div>

### 6.5 Ảnh 5: IAM Assignments
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>🔑 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Resource Group → Access control (IAM)</li>
<li>✅ Role assignments tab</li>
<li>✅ 5 assignments visible: 4 groups + 1 service principal</li>
<li>✅ All assigned <code>Contributor</code> role</li>
</ul>
</div>

### 6.6 Ảnh 6: Group Memberships
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>👥 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Azure AD → Groups → CloudEngineer-retail-dev → Members</li>
<li>✅ List of users in the group</li>
<li>✅ User names and email addresses visible</li>
<li>✅ Member count matches expected</li>
</ul>
</div>

### 6.7 Ảnh 7: Service Principal Credentials
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>🔐 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Azure AD → App registrations → sp-retail-devops → Certificates & secrets</li>
<li>✅ Client secrets section</li>
<li>✅ Secret value hidden (shows as dots/asterisks)</li>
<li>✅ Expiration date visible</li>
</ul>
</div>

### 6.8 Ảnh 8: Login Test Results
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>🔑 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Azure CLI login successful</li>
<li>✅ <code>az account show</code> output showing correct subscription</li>
<li>✅ <code>az group show</code> command working</li>
<li>✅ User/SP permissions verified</li>
</ul>
</div>

## 7. TIÊU CHÍ HOÀN THÀNH

### 7.1 Functional Requirements
- [x] **Resource Group created**: `retail-dev-rg` hiển thị trong portal
- [x] **Correct region**: Southeast Asia location
- [x] **Tagging complete**: Đầy đủ 7 tags theo chuẩn
- [x] **Active status**: Resource Group ở trạng thái Active

### 7.2 Identity & Access Requirements
- [x] **4 User Groups created**: CloudEngineer, DataEngineer, Analyst, Owners
- [x] **Service Principal created**: sp-retail-devops for DevOps
- [x] **RBAC configured**: All 5 entities assigned Contributor role
- [x] **Scope correct**: Permissions at Resource Group level

### 7.3 Security Requirements
- [x] **Least privilege**: Appropriate role assignments
- [x] **Audit logging**: Activity log tracking enabled
- [x] **Access review**: Verify quyền truy cập phù hợp

## 8. AUTOMATION SCRIPT

### 8.1 Complete Setup Script

```bash
#!/bin/bash
# setup-resource-group.sh

# Variables
SUBSCRIPTION_ID="your-subscription-id"
RG_NAME="retail-dev-rg"
LOCATION="southeastasia"

# Set subscription
az account set --subscription $SUBSCRIPTION_ID

# Create resource group with tags
echo "🏗️ Creating Resource Group..."
az group create \
  --name $RG_NAME \
  --location $LOCATION \
  --tags \
    Environment=development \
    Project=retail-forecast \
    Owner=mlops-team \
    CostCenter=ml-engineering \
    Purpose=mlops-platform \
    DataClassification=internal \
    CreatedBy=$(az account show --query user.name --output tsv)

# Get Resource Group ID
RG_ID=$(az group show --name $RG_NAME --query id --output tsv)

# Create Service Principal
echo "🔐 Creating Service Principal..."
az ad sp create-for-rbac \
  --name sp-retail-devops \
  --role Contributor \
  --scopes $RG_ID \
  --sdk-auth > sp-retail-devops.json

echo "✅ Resource Group and Service Principal created successfully!"
echo "📁 Service Principal credentials saved to sp-retail-devops.json"

# Display Resource Group info
echo "📋 Resource Group Details:"
az group show --name $RG_NAME --output table
```

## 9. HƯỚNG DẪN ĐĂNG NHẬP

### 9.1 Đăng nhập bằng User Account

**Cho 4 user groups (CloudEngineer, DataEngineer, Analyst, Owners):**

```bash
# Đăng nhập bằng Azure CLI
az login

# Chọn đúng subscription
az account set --subscription "your-subscription-id"

# Verify quyền truy cập
az group show --name retail-dev-rg
```

**Kiểm tra quyền truy cập:**
```bash
# Xem role assignments của user hiện tại
az role assignment list --assignee $(az account show --query user.name -o tsv) --scope /subscriptions/{subscription-id}/resourceGroups/retail-dev-rg

# Test tạo resource trong Resource Group
az group show --name retail-dev-rg --output table
```

### 9.2 Đăng nhập bằng Service Principal

**Cho DevOps automation (sp-retail-devops):**

```bash
# Method 1: Sử dụng file credentials đã tạo
az login --service-principal \
  -u $(jq -r .clientId sp-retail-devops.json) \
  -p $(jq -r .clientSecret sp-retail-devops.json) \
  --tenant $(jq -r .tenantId sp-retail-devops.json)

# Method 2: Sử dụng environment variables
export AZURE_CLIENT_ID=$(jq -r .clientId sp-retail-devops.json)
export AZURE_CLIENT_SECRET=$(jq -r .clientSecret sp-retail-devops.json)
export AZURE_TENANT_ID=$(jq -r .tenantId sp-retail-devops.json)

az login --service-principal \
  -u $AZURE_CLIENT_ID \
  -p $AZURE_CLIENT_SECRET \
  --tenant $AZURE_TENANT_ID
```

**Verify Service Principal permissions:**
```bash
# Kiểm tra quyền truy cập
az group show --name retail-dev-rg

# List resources trong Resource Group
az resource list --resource-group retail-dev-rg --output table
```

### 9.3 Troubleshooting Login Issues

**Common Issues & Solutions:**

<div style="background: #fffbeb; border: 1px solid #fed7aa; border-left: 4px solid #f59e0b; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>⚠️ Troubleshooting Guide</strong>

**1. "Insufficient privileges" error:**
```bash
# Kiểm tra user có được add vào group chưa
az ad user show --id user@company.com --query "memberOf"

# Verify group membership
az ad group member list --group CloudEngineer-retail-dev
```

**2. "Invalid credentials" for Service Principal:**
```bash
# Check if client secret expired
az ad app credential list --id $(jq -r .clientId sp-retail-devops.json)

# Regenerate client secret if needed
az ad app credential reset --id $(jq -r .clientId sp-retail-devops.json)
```

**3. "Resource group not found":**
```bash
# Verify subscription context
az account show --output table

# Switch to correct subscription
az account set --subscription "correct-subscription-id"
```

</div>

### 9.4 Test Access Permissions

**Verify Contributor role functionality:**

```bash
# Test 1: Read Resource Group
az group show --name retail-dev-rg --query "name"

# Test 2: List resources (should work with Contributor)
az resource list --resource-group retail-dev-rg --output table

# Test 3: Create a test storage account (optional)
az storage account create \
  --name teststorage$(date +%s) \
  --resource-group retail-dev-rg \
  --location southeastasia \
  --sku Standard_LRS

# Test 4: Clean up test resource
az storage account delete \
  --name teststorage$(date +%s) \
  --resource-group retail-dev-rg \
  --yes
```

## 10. LƯU Ý QUAN TRỌNG

### 10.1 Environment Strategy
<div style="background: #fffbeb; border: 1px solid #fed7aa; border-left: 4px solid #f59e0b; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>⚠️ Multi-Environment Setup</strong>
<ul>
<li><strong>Development</strong>: <code>retail-dev-rg</code> ← Current</li>
<li><strong>Staging</strong>: <code>retail-staging-rg</code> ← Future</li>
<li><strong>Production</strong>: <code>retail-prod-rg</code> ← Future</li>
</ul>
<p><em>Mỗi môi trường cần Resource Group riêng để isolation và governance.</em></p>
</div>

### 10.2 Cost Management
- 💰 **Budget alerts**: Thiết lập budget cho Resource Group
- 📊 **Cost analysis**: Monitor chi phí theo tags
- 🔄 **Lifecycle policies**: Auto-cleanup dev resources
- 📈 **Resource utilization**: Track usage metrics

### 10.3 Next Steps
1. ✅ **Resource Group created** ← Current step
2. 🔄 **Storage Account setup** ← Next step
3. 🔄 **Key Vault configuration**
4. 🔄 **Azure ML Workspace**
5. 🔄 **Container Registry setup**

{{% notice info %}}
**Best Practice**: Resource Group naming convention nên consistent across environments. Format: `{project}-{environment}-rg` giúp dễ dàng automation và governance.
{{% /notice %}}

{{% notice warning %}}
**Security Note**: Không assign Owner permissions cho service principals. Sử dụng Contributor role với custom policies khi cần thiết. Managed Identity permissions sẽ được gán khi tạo Azure ML Workspace.
{{% /notice %}}

## 11. USEFUL LINKS

- **Azure Resource Groups**: https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/overview
- **Azure RBAC**: https://docs.microsoft.com/en-us/azure/role-based-access-control/
- **Service Principals**: https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals
- **Azure AD Groups**: https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/active-directory-groups-create-azure-portal

Resource Group foundation hoàn tất! 🎉 Tiếp tục với **Task 3: Data Upload & Storage Configuration**.
