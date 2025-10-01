---
title: "Infrastructure as Code - Foundation Layer"
date: 2025-08-30T12:00:00+07:00
weight: 7
chapter: false
pre: "<b>7. </b>"
---

Trong phần này, chúng ta sẽ dựng Foundation layer cho dự án bằng Infrastructure as Code (IaC) sử dụng Terraform và Bicep. Đây là nền tảng cần thiết để các Task 8-19 có thể chạy được, bao gồm Resource Group, Key Vault, Azure Container Registry, Storage Account, Azure ML Workspace, và Compute clusters. Việc sử dụng IaC đảm bảo reproducibility, version control, và consistency across environments.

## 1. Mục tiêu

✅ **Dựng Foundation layer bằng IaC (Terraform/Bicep)**  
✅ **Tạo tất cả tài nguyên cần thiết cho MLOps pipeline**  
✅ **Cấu hình RBAC và Managed Identity**  
✅ **Chuẩn bị nền tảng cho các Task 8-19**

<div style="background: linear-gradient(135deg, #e6f3ff 0%, #dbeafe 100%); border: 1px solid #bfdbfe; border-left: 6px solid #0078d4; padding: 18px; margin: 18px 0; border-radius: 10px;">
  <strong>TASK 7 — INFRASTRUCTURE AS CODE</strong><br>
  <strong>Mục tiêu</strong>: Dựng Foundation layer cho dự án bằng IaC: Resource Group, Key Vault, ACR, Storage (Blob/ADLS), Azure ML Workspace, Compute (AML). Đây là nền để các task 8–19 chạy được.
</div>

## 2. INFRASTRUCTURE ARCHITECTURE

### 2.1 Foundation Components

<div style="background: #f8fafc; border: 1px solid #e2e8f0; border-left: 4px solid #0078d4; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>🔧 Foundation Resources</strong>

**1. Resource Group:**
- **Name**: `rg-mlops-retail-{env}`
- **Purpose**: Container cho tất cả MLOps resources
- **Tags**: Environment, Project, Owner, CostCenter

**2. Key Vault:**
- **Name**: `kv-retail-{env}`
- **Features**: RBAC-enabled, purge protection, soft delete
- **Purpose**: Secure secrets và certificates management

**3. Azure Container Registry:**
- **Name**: `acrretail{env}`
- **Features**: Admin disabled, vulnerability scanning
- **Purpose**: Container images cho ML training và inference

**4. Storage Account:**
- **Name**: `stretail{env}`
- **Features**: Blob storage, ADLS Gen2, secure transfer
- **Purpose**: Data lake cho raw và processed data

**5. Azure ML Workspace:**
- **Name**: `amlws-retail-{env}`
- **Features**: System-assigned identity, integrated services
- **Purpose**: ML training, model registry, experiment tracking

**6. AML Compute:**
- **Name**: `cpu-cluster`
- **Features**: Scale-to-zero, auto-scaling
- **Purpose**: Training compute với cost optimization

</div>

### 2.2 Security & RBAC Design

<div style="background: #fff7ed; border: 1px solid #fed7aa; border-left: 4px solid #f59e0b; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>🔐 Security Architecture</strong>

**1. Managed Identity:**
- **AML Workspace**: System-assigned identity
- **ACR Access**: AcrPull role assignment
- **Storage Access**: Storage Blob Data Reader
- **Key Vault**: Key Vault Secrets User

**2. RBAC Configuration:**
- **Key Vault**: RBAC-enabled authorization
- **Resource Group**: Contributor role for team
- **Storage**: Least privilege access
- **ACR**: Managed identity authentication

**3. Encryption:**
- **At Rest**: Azure-managed keys
- **In Transit**: TLS 1.2 minimum
- **Key Vault**: Hardware Security Module (HSM)

**4. Network Security:**
- **Private Endpoints**: For production environments
- **Firewall Rules**: IP whitelisting if needed
- **VNet Integration**: For secure communication

</div>

### 2.3 Naming Convention & Tags

<div style="background: #f1f5f9; border: 1px solid #e2e8f0; border-left: 4px solid #6366f1; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>📋 Naming Standards</strong>

**Resource Naming Pattern:**
- **Resource Group**: `rg-mlops-retail-{env}`
- **Key Vault**: `kv-retail-{env}`
- **ACR**: `acrretail{env}` (no hyphens)
- **Storage**: `stretail{env}` (no hyphens)
- **AML Workspace**: `amlws-retail-{env}`
- **Compute**: `cpu-cluster`

**Environment Suffixes:**
- **Development**: `dev`
- **Staging**: `staging`
- **Production**: `prod`

**Standard Tags:**
- **Project**: `MLOpsRetailForecast`
- **Environment**: `{dev/staging/prod}`
- **Component**: `ml`
- **Owner**: `DataTeam`
- **CostCenter**: `ML-Platform`

</div>

## 3. TERRAFORM IMPLEMENTATION

### 3.1 Prerequisites

**Required Components:**
- ✅ **Azure Subscription** với Contributor permissions
- ✅ **Terraform** >= 1.0 installed
- ✅ **Azure CLI** >= 2.30 installed và configured
- ✅ **Git** repository với infrastructure code
- ✅ **Backend storage** cho Terraform state (optional)

### 3.2 File Structure

```
azure/infra/
├── main.tf                    # Main infrastructure resources
├── variables.tf               # Variable definitions
├── outputs.tf                 # Output values
├── providers.tf               # Provider configuration
├── backend.tf                 # Backend configuration
├── dev.tfvars                 # Development environment variables
├── prod.tfvars                # Production environment variables
├── online_endpoint.tf         # ML endpoint configuration (optional)
├── alerts.tf                  # Monitoring alerts (Task 15)
├── main.bicep                 # Bicep alternative
└── parameters/                # Bicep parameter files
    ├── main.dev.json
    └── main.prod.json
```

### 3.3 Main Infrastructure (main.tf)

```hcl
# ---------- Resource Group ----------
resource "azurerm_resource_group" "rg" {
  name     = "${var.name_prefix}-rg"
  location = var.location
  tags     = var.tags
}

# ---------- Log Analytics & App Insights ----------
resource "azurerm_log_analytics_workspace" "law" {
  name                = "${var.name_prefix}-log"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
  tags                = var.tags
}

resource "azurerm_application_insights" "appi" {
  name                = "${var.name_prefix}-appi"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name
  application_type    = "web"
  workspace_id        = azurerm_log_analytics_workspace.law.id
  tags                = var.tags
}

# ---------- Storage (Blob/ADLS) ----------
resource "azurerm_storage_account" "st" {
  name                             = replace("${var.name_prefix}st", "-", "")
  resource_group_name              = azurerm_resource_group.rg.name
  location                         = var.location
  account_tier                     = "Standard"
  account_replication_type         = "LRS"
  min_tls_version                  = "TLS1_2"
  allow_nested_items_to_be_public  = false
  
  # Enable ADLS Gen2
  is_hns_enabled = true
  
  # Enable versioning and soft delete
  blob_properties {
    delete_retention_policy {
      days = 7
    }
    versioning_enabled = true
  }
  
  tags = var.tags
}

resource "azurerm_storage_container" "data" {
  name                  = "data"
  storage_account_name  = azurerm_storage_account.st.name
  container_access_type = "private"
}

# ---------- Key Vault (RBAC) ----------
data "azurerm_client_config" "current" {}

resource "azurerm_key_vault" "kv" {
  name                       = replace("${var.name_prefix}-kv", "_", "-")
  location                   = var.location
  resource_group_name        = azurerm_resource_group.rg.name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "standard"
  purge_protection_enabled   = true
  soft_delete_retention_days = 7
  enable_rbac_authorization  = true
  tags                       = var.tags
}

# ---------- Azure Container Registry ----------
resource "azurerm_container_registry" "acr" {
  name                = replace("${var.name_prefix}acr", "-", "")
  resource_group_name = azurerm_resource_group.rg.name
  location            = var.location
  sku                 = var.acr_sku
  admin_enabled       = false
  
  # Enable vulnerability scanning
  security_policy {
    trust_policy {
      enabled = true
    }
  }
  
  tags = var.tags
}

# ---------- Azure ML Workspace ----------
resource "azurerm_machine_learning_workspace" "aml" {
  name                    = "${var.name_prefix}-amlws"
  location                = var.location
  resource_group_name     = azurerm_resource_group.rg.name

  application_insights_id = azurerm_application_insights.appi.id
  key_vault_id            = azurerm_key_vault.kv.id
  storage_account_id      = azurerm_storage_account.st.id
  container_registry_id   = azurerm_container_registry.acr.id

  identity { type = "SystemAssigned" }
  tags = var.tags
}

# ---------- AML Compute Cluster ----------
resource "azurerm_machine_learning_compute_cluster" "cpu" {
  name                          = "cpu-cluster"
  location                      = var.location
  machine_learning_workspace_id = azurerm_machine_learning_workspace.aml.id
  vm_size                       = "Standard_DS3_v2"

  scale_settings {
    min_node_count = 0
    max_node_count = 4
    scale_down_nodes_after_idle_duration = "PT15M"
  }
  tags = var.tags
}

# ---------- RBAC Assignments ----------
# AML Workspace MI → ACR Pull
resource "azurerm_role_assignment" "acr_pull_to_aml" {
  scope                = azurerm_container_registry.acr.id
  role_definition_name = "AcrPull"
  principal_id         = azurerm_machine_learning_workspace.aml.identity[0].principal_id
}

# AML Workspace MI → Storage Blob Data Reader
resource "azurerm_role_assignment" "storage_reader_to_aml" {
  scope                = azurerm_storage_account.st.id
  role_definition_name = "Storage Blob Data Reader"
  principal_id         = azurerm_machine_learning_workspace.aml.identity[0].principal_id
}

# AML Workspace MI → Key Vault Secrets User
resource "azurerm_role_assignment" "kv_secrets_to_aml" {
  scope                = azurerm_key_vault.kv.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_machine_learning_workspace.aml.identity[0].principal_id
}
```

### 3.4 Variables Configuration

```hcl
# variables.tf
variable "location" {
  type        = string
  default     = "southeastasia"
  description = "Azure region for resources"
}

variable "name_prefix" {
  type        = string
  default     = "retail-dev"
  description = "Prefix for resource naming"
}

variable "acr_sku" {
  type        = string
  default     = "Standard"
  description = "ACR SKU (Basic, Standard, Premium)"
}

variable "tags" {
  type = map(string)
  default = {
    Project     = "MLOpsRetailForecast"
    Environment = "dev"
    Component   = "ml"
    Owner       = "DataTeam"
    CostCenter  = "ML-Platform"
  }
  description = "Standard tags for all resources"
}

# Online endpoint/deployment variables
variable "endpoint_name" {
  type        = string
  default     = "retail-forecast-endpoint"
  description = "Name for ML online endpoint"
}

variable "model_name" {
  type        = string
  default     = "retail-forecast"
  description = "Model name for deployment"
}

variable "model_version" {
  type        = string
  default     = "1"
  description = "Model version for deployment"
}

variable "infer_image_tag" {
  type        = string
  default     = "latest"
  description = "Inference container image tag"
}

# Alert configuration
variable "action_group_id" {
  type        = string
  default     = ""
  description = "Resource ID of Action Group for alerts"
}
```

### 3.5 Environment-Specific Configuration

```hcl
# dev.tfvars
location     = "southeastasia"
name_prefix  = "retail-dev"
acr_sku      = "Standard"

endpoint_name   = "retail-forecast-endpoint"
model_name      = "retail-forecast"
model_version   = "1"
infer_image_tag = "latest"

tags = {
  Project     = "MLOpsRetailForecast"
  Environment = "dev"
  Component   = "ml"
  Owner       = "DataTeam"
  CostCenter  = "ML-Platform"
}

action_group_id = ""
```

```hcl
# prod.tfvars
location     = "southeastasia"
name_prefix  = "retail-prod"
acr_sku      = "Premium"

endpoint_name   = "retail-forecast-endpoint"
model_name      = "retail-forecast"
model_version   = "1"
infer_image_tag = "latest"

tags = {
  Project     = "MLOpsRetailForecast"
  Environment = "prod"
  Component   = "ml"
  Owner       = "DataTeam"
  CostCenter  = "ML-Platform"
}

action_group_id = "/subscriptions/{sub-id}/resourceGroups/rg-alerts/providers/microsoft.insights/actiongroups/prod-alerts"
```

## 4. TERRAFORM DEPLOYMENT

### 4.1 Deployment Workflow

<div style="background: #f8fafc; border: 1px solid #e2e8f0; border-left: 4px solid #0078d4; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>🚀 Deployment Steps</strong>

**Step 1: Initialize Terraform**
```bash
cd azure/infra
terraform init
```

**Step 2: Plan Deployment**
```bash
terraform plan -var-file=dev.tfvars
```

**Step 3: Apply Infrastructure**
```bash
terraform apply -var-file=dev.tfvars
```

**Step 4: Verify Deployment**
```bash
terraform output
```

</div>

### 4.2 Complete Deployment Script

```bash
#!/bin/bash
# deploy-infrastructure.sh

# Configuration
ENVIRONMENT=${1:-dev}
SUBSCRIPTION_ID="your-subscription-id"
RESOURCE_GROUP="rg-mlops-retail-${ENVIRONMENT}"

echo "🚀 Deploying MLOps Infrastructure for ${ENVIRONMENT} environment..."

# Login to Azure
az login

# Set subscription
az account set --subscription $SUBSCRIPTION_ID

# Navigate to infrastructure directory
cd azure/infra

# Initialize Terraform
echo "📋 Initializing Terraform..."
terraform init

# Validate configuration
echo "✅ Validating Terraform configuration..."
terraform validate

# Plan deployment
echo "📊 Planning infrastructure deployment..."
terraform plan -var-file="${ENVIRONMENT}.tfvars" -out="${ENVIRONMENT}.tfplan"

# Apply infrastructure
echo "🏗️ Applying infrastructure..."
terraform apply "${ENVIRONMENT}.tfplan"

# Display outputs
echo "📋 Infrastructure deployment completed!"
echo "📊 Resource outputs:"
terraform output

# Verify resources
echo "🔍 Verifying deployed resources..."
az resource list --resource-group $RESOURCE_GROUP --output table

echo "✅ Infrastructure deployment completed successfully!"
```

### 4.3 Backend Configuration (Optional)

```hcl
# backend.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate"
    container_name       = "tfstate"
    key                  = "mlops-retail-dev.tfstate"
  }
}
```

## 5. BICEP IMPLEMENTATION

### 5.1 Bicep Alternative

```bicep
// main.bicep
param location string = resourceGroup().location
param environment string = 'dev'
param workspaceName string = 'amlws-retail-${environment}'
param storageName string = 'stretail${environment}'
param acrName string = 'acrretail${environment}'
param keyVaultName string = 'kv-retail-${environment}'

// Storage Account
resource storage 'Microsoft.Storage/storageAccounts@2022-09-01' = {
  name: storageName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    isHnsEnabled: true
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    deleteRetentionPolicy: {
      enabled: true
      days: 7
    }
  }
  tags: {
    Project: 'MLOpsRetailForecast'
    Environment: environment
    Component: 'ml'
  }
}

// Container Registry
resource acr 'Microsoft.ContainerRegistry/registries@2023-01-01-preview' = {
  name: acrName
  location: location
  sku: {
    name: 'Standard'
  }
  properties: {
    adminUserEnabled: false
    policies: {
      trustPolicy: {
        enabled: true
      }
    }
  }
  tags: {
    Project: 'MLOpsRetailForecast'
    Environment: environment
    Component: 'ml'
  }
}

// Key Vault
resource keyVault 'Microsoft.KeyVault/vaults@2022-07-01' = {
  name: keyVaultName
  location: location
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true
    enablePurgeProtection: true
    softDeleteRetentionInDays: 7
  }
  tags: {
    Project: 'MLOpsRetailForecast'
    Environment: environment
    Component: 'ml'
  }
}

// Application Insights
resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: 'appi-retail-${environment}'
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
  }
  tags: {
    Project: 'MLOpsRetailForecast'
    Environment: environment
    Component: 'ml'
  }
}

// Azure ML Workspace
resource amlWorkspace 'Microsoft.MachineLearningServices/workspaces@2023-04-01' = {
  name: workspaceName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    description: 'Azure ML workspace for retail forecasting'
    friendlyName: 'RetailML-${environment}'
    storageAccount: storage.id
    containerRegistry: acr.id
    keyVault: keyVault.id
    applicationInsights: appInsights.id
  }
  tags: {
    Project: 'MLOpsRetailForecast'
    Environment: environment
    Component: 'ml'
  }
}

// AML Compute Cluster
resource computeCluster 'Microsoft.MachineLearningServices/workspaces/computes@2023-04-01' = {
  parent: amlWorkspace
  name: 'cpu-cluster'
  location: location
  properties: {
    computeType: 'AmlCompute'
    properties: {
      vmSize: 'Standard_DS3_v2'
      vmPriority: 'Dedicated'
      scaleSettings: {
        minNodeCount: 0
        maxNodeCount: 4
        nodeIdleTimeBeforeScaleDown: 'PT15M'
      }
    }
  }
}

// RBAC Assignments
resource acrPullAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(amlWorkspace.id, 'AcrPull')
  scope: acr
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '7f951dda-4ed3-4680-a7ca-43fe172d538d')
    principalId: amlWorkspace.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Outputs
output workspaceId string = amlWorkspace.id
output workspaceName string = amlWorkspace.name
output storageAccountName string = storage.name
output acrName string = acr.name
output keyVaultName string = keyVault.name
```

### 5.2 Bicep Parameter Files

```json
// parameters/main.dev.json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environment": {
      "value": "dev"
    },
    "workspaceName": {
      "value": "amlws-retail-dev"
    },
    "storageName": {
      "value": "stretaildev"
    },
    "acrName": {
      "value": "acrretaildev"
    },
    "keyVaultName": {
      "value": "kv-retail-dev"
    }
  }
}
```

### 5.3 Bicep Deployment

```bash
#!/bin/bash
# deploy-bicep.sh

ENVIRONMENT=${1:-dev}
RESOURCE_GROUP="rg-mlops-retail-${ENVIRONMENT}"

echo "🚀 Deploying infrastructure with Bicep..."

# Create resource group if not exists
az group create --name $RESOURCE_GROUP --location southeastasia

# Deploy infrastructure
az deployment group create \
  --resource-group $RESOURCE_GROUP \
  --template-file main.bicep \
  --parameters @parameters/main.${ENVIRONMENT}.json

echo "✅ Bicep deployment completed!"
```

## 6. BẰNG CHỨNG HOÀN THÀNH

### 6.1 Ảnh 1: Terraform Plan Output
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📋 Yêu cầu screenshot:</strong>
<ul>
<li>✅ <code>terraform plan -var-file=dev.tfvars</code> output</li>
<li>✅ Resources to be created listed</li>
<li>✅ No errors or warnings</li>
<li>✅ Plan file generated successfully</li>
</ul>
</div>

### 6.2 Ảnh 2: Terraform Apply Success
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>✅ Yêu cầu screenshot:</strong>
<ul>
<li>✅ <code>terraform apply</code> completed successfully</li>
<li>✅ All resources created</li>
<li>✅ Apply time and resource count visible</li>
<li>✅ Output values displayed</li>
</ul>
</div>

### 6.3 Ảnh 3: Resource Group Overview
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>🏗️ Yêu cầu screenshot:</strong>
<ul>
<li>✅ Resource Group <code>rg-mlops-retail-dev</code></li>
<li>✅ All foundation resources visible</li>
<li>✅ Resource status: Succeeded</li>
<li>✅ Tags properly applied</li>
</ul>
</div>

### 6.4 Ảnh 4: Azure ML Workspace
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>🤖 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Azure ML Workspace <code>amlws-retail-dev</code></li>
<li>✅ Status: Active</li>
<li>✅ Integrated services: Storage, ACR, Key Vault, App Insights</li>
<li>✅ System-assigned identity configured</li>
</ul>
</div>

### 6.5 Ảnh 5: Compute Cluster
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>⚡ Yêu cầu screenshot:</strong>
<ul>
<li>✅ AML Compute <code>cpu-cluster</code></li>
<li>✅ Status: Succeeded</li>
<li>✅ Scale settings: 0-4 nodes, 15min idle</li>
<li>✅ VM Size: Standard_DS3_v2</li>
</ul>
</div>

### 6.6 Ảnh 6: RBAC Assignments
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>🔐 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Key Vault → Access control (IAM)</li>
<li>✅ AML Workspace MI assigned Key Vault Secrets User</li>
<li>✅ ACR → Access control showing AcrPull assignment</li>
<li>✅ Storage → Access control showing Blob Data Reader</li>
</ul>
</div>

## 7. TIÊU CHÍ HOÀN THÀNH

### 7.1 Functional Requirements
- [x] **Infrastructure deployed**: All foundation resources created successfully
- [x] **Resource Group**: `rg-mlops-retail-{env}` with proper naming
- [x] **Key Vault**: RBAC-enabled with security features
- [x] **ACR**: Container registry with vulnerability scanning
- [x] **Storage**: ADLS Gen2 with versioning and soft delete
- [x] **AML Workspace**: Integrated with all services
- [x] **Compute**: Scale-to-zero cluster configured

### 7.2 Security Requirements
- [x] **Managed Identity**: System-assigned identity for AML workspace
- [x] **RBAC**: Proper role assignments for cross-service access
- [x] **Encryption**: At-rest and in-transit encryption enabled
- [x] **Key Vault**: RBAC authorization, purge protection
- [x] **ACR**: Admin disabled, vulnerability scanning enabled

### 7.3 Naming & Tagging Requirements
- [x] **Naming convention**: Consistent across all resources
- [x] **Environment tagging**: Proper environment identification
- [x] **Standard tags**: Project, Environment, Component, Owner, CostCenter
- [x] **Resource organization**: Logical grouping and identification

## 8. AUTOMATION SCRIPTS

### 8.1 Complete Infrastructure Deployment

```bash
#!/bin/bash
# deploy-complete-infrastructure.sh

set -e

# Configuration
ENVIRONMENT=${1:-dev}
SUBSCRIPTION_ID=${2:-"your-subscription-id"}
RESOURCE_GROUP="rg-mlops-retail-${ENVIRONMENT}"

echo "🚀 Starting complete MLOps infrastructure deployment..."
echo "Environment: $ENVIRONMENT"
echo "Subscription: $SUBSCRIPTION_ID"

# Validate environment
if [[ ! "$ENVIRONMENT" =~ ^(dev|staging|prod)$ ]]; then
    echo "❌ Invalid environment. Use: dev, staging, or prod"
    exit 1
fi

# Login and set subscription
echo "🔐 Authenticating with Azure..."
az login
az account set --subscription $SUBSCRIPTION_ID

# Create resource group
echo "📁 Creating resource group..."
az group create \
  --name $RESOURCE_GROUP \
  --location southeastasia \
  --tags \
    Project=MLOpsRetailForecast \
    Environment=$ENVIRONMENT \
    Component=ml \
    Owner=DataTeam

# Navigate to infrastructure directory
cd azure/infra

# Initialize Terraform
echo "📋 Initializing Terraform..."
terraform init

# Validate configuration
echo "✅ Validating Terraform configuration..."
terraform validate

# Plan deployment
echo "📊 Planning infrastructure deployment..."
terraform plan \
  -var-file="${ENVIRONMENT}.tfvars" \
  -out="${ENVIRONMENT}.tfplan" \
  -detailed-exitcode

PLAN_EXIT_CODE=$?

if [ $PLAN_EXIT_CODE -eq 0 ]; then
    echo "✅ No changes needed"
elif [ $PLAN_EXIT_CODE -eq 2 ]; then
    echo "🔄 Changes detected, proceeding with deployment..."
    
    # Apply infrastructure
    echo "🏗️ Applying infrastructure..."
    terraform apply "${ENVIRONMENT}.tfplan"
    
    # Display outputs
    echo "📋 Infrastructure deployment completed!"
    echo "📊 Resource outputs:"
    terraform output
    
else
    echo "❌ Terraform plan failed"
    exit 1
fi

# Verify deployment
echo "🔍 Verifying deployed resources..."
az resource list \
  --resource-group $RESOURCE_GROUP \
  --output table

# Test AML workspace access
echo "🤖 Testing Azure ML Workspace access..."
WORKSPACE_NAME=$(terraform output -raw workspace_name)
az ml workspace show \
  --name $WORKSPACE_NAME \
  --resource-group $RESOURCE_GROUP

echo "✅ Infrastructure deployment completed successfully!"
echo "🎉 Foundation layer ready for MLOps pipeline!"
```

### 8.2 Infrastructure Validation Script

```bash
#!/bin/bash
# validate-infrastructure.sh

ENVIRONMENT=${1:-dev}
RESOURCE_GROUP="rg-mlops-retail-${ENVIRONMENT}"

echo "🔍 Validating infrastructure deployment..."

# Check resource group
echo "📁 Checking resource group..."
az group show --name $RESOURCE_GROUP --query "name" -o tsv

# Check all resources
echo "🏗️ Checking deployed resources..."
RESOURCES=(
    "azurerm_storage_account"
    "azurerm_key_vault"
    "azurerm_container_registry"
    "azurerm_machine_learning_workspace"
    "azurerm_application_insights"
)

for resource_type in "${RESOURCES[@]}"; do
    echo "Checking $resource_type..."
    az resource list \
      --resource-group $RESOURCE_GROUP \
      --resource-type $resource_type \
      --query "[].name" -o tsv
done

# Check RBAC assignments
echo "🔐 Checking RBAC assignments..."
WORKSPACE_ID=$(az ml workspace show \
  --name "amlws-retail-${ENVIRONMENT}" \
  --resource-group $RESOURCE_GROUP \
  --query identity.principalId -o tsv)

echo "AML Workspace Principal ID: $WORKSPACE_ID"

# Check compute cluster
echo "⚡ Checking compute cluster..."
az ml compute show \
  --name cpu-cluster \
  --workspace-name "amlws-retail-${ENVIRONMENT}" \
  --resource-group $RESOURCE_GROUP

echo "✅ Infrastructure validation completed!"
```

## 9. LƯU Ý QUAN TRỌNG

### 9.1 Infrastructure Best Practices
<div style="background: #fffbeb; border: 1px solid #fed7aa; border-left: 4px solid #f59e0b; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>⚠️ Important Considerations</strong>
<ul>
<li><strong>State Management</strong>: Use remote backend for Terraform state</li>
<li><strong>Environment Isolation</strong>: Separate resource groups per environment</li>
<li><strong>Cost Optimization</strong>: Use appropriate SKUs for dev vs prod</li>
<li><strong>Security</strong>: Enable all security features from the start</li>
<li><strong>Monitoring</strong>: Set up alerts and monitoring from day one</li>
</ul>
</div>

### 9.2 Troubleshooting Common Issues

**Issue 1: "Resource name conflicts"**
```bash
# Check naming convention
# Ensure unique names across Azure
# Use proper naming patterns
```

**Issue 2: "RBAC permission errors"**
```bash
# Verify user has Contributor role on subscription
# Check Key Vault access policies
# Validate Managed Identity assignments
```

**Issue 3: "Terraform state conflicts"**
```bash
# Use remote backend for state
# Lock state during operations
# Backup state before major changes
```

### 9.3 Next Steps
1. ✅ **Foundation Infrastructure completed** ← Current step
2. 🔄 **Azure ML Workspace setup** ← Next step
3. 🔄 **Data asset registration**
4. 🔄 **Model training configuration**
5. 🔄 **Endpoint deployment**

{{% notice info %}}
**Best Practice**: Always validate infrastructure with `terraform plan` before applying changes. Use environment-specific variable files to maintain consistency across deployments.
{{% /notice %}}

{{% notice warning %}}
**Security Note**: Never commit sensitive information like connection strings or keys to version control. Use Azure Key Vault for secrets management.
{{% /notice %}}

## 10. USEFUL LINKS

- **Terraform Azure Provider**: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs
- **Azure Bicep**: https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/
- **Azure ML Workspace**: https://docs.microsoft.com/en-us/azure/machine-learning/concept-workspace
- **Infrastructure Best Practices**: https://docs.microsoft.com/en-us/azure/architecture/framework/

Foundation infrastructure hoàn tất! 🎉 Nền tảng đã sẵn sàng cho **Task 8: Azure ML Workspace Configuration**.
