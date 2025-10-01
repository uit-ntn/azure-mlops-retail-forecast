---
title: "Data Asset Registration"
date: 2025-08-30T11:30:00+07:00
weight: 4
chapter: false
pre: "<b>4. </b>"
---

Trong phần này, chúng ta sẽ tạo Data Asset trong Azure ML Studio để đăng ký dữ liệu ecommerce đã upload vào Azure Storage. Data Asset sẽ cung cấp versioning, metadata tracking, và seamless integration với Azure ML training jobs. Đây là bước quan trọng để chuẩn bị dữ liệu cho quá trình training và đảm bảo reproducibility trong ML pipeline.

## 1. Mục tiêu

✅ **Tạo Data Asset với URI folder trỏ tới data/raw/ecommerce/**  
✅ **Cấu hình proper metadata và versioning**  
✅ **Đảm bảo Data Asset sẵn sàng cho training jobs**  
✅ **Verify accessibility từ Azure ML workspace**

<div style="background: linear-gradient(135deg, #e6f3ff 0%, #dbeafe 100%); border: 1px solid #bfdbfe; border-left: 6px solid #0078d4; padding: 18px; margin: 18px 0; border-radius: 10px;">
  <strong>TASK 4 — DATA ASSET (AZURE ML STUDIO)</strong><br>
  <strong>Mục tiêu</strong>: Tạo Data Asset (URI folder) trỏ tới data/raw/ecommerce/. UI: Studio → Data → +Create (File/Folder). Bằng chứng: Card retail_data:1 (ảnh). Done: Asset dùng được trong Jobs.
</div>

## 2. AZURE ML STUDIO SETUP

### 2.1 Prerequisites

**Required Components:**
- ✅ **Azure ML Workspace** đã tạo (từ Task 5 - sẽ tạo trong bước tiếp theo)
- ✅ **Storage Account** `retailforecastdev` với ADLS Gen2 enabled (từ **Task 3: Data Upload**)
- ✅ **Data uploaded** vào `data/raw/ecommerce/` container (từ **Task 3: Data Upload**)
- ✅ **Resource Group** `retail-dev-rg` với proper RBAC (từ **Task 2: Resource Group**)
- ✅ **Managed Identity permissions** cho Azure ML workspace access storage (sẽ setup trong Task 5)

<div style="background: #f8fafc; border: 1px solid #e2e8f0; border-left: 4px solid #0078d4; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>🔧 Data Asset Configuration</strong>

**1. Asset Type:**
- **Type**: URI folder (recommended cho large datasets)
- **Name**: `retail_data`
- **Version**: `1` (auto-incrementing)
- **Description**: `Retail e-commerce dataset 2019 - Raw CSV files`

**2. Storage Reference:**
- **Storage Type**: Azure Data Lake Storage Gen2
- **Container**: `data`
- **Folder Path**: `raw/ecommerce/`
- **Authentication**: Managed Identity (recommended)

</div>

### 2.2 UI Flow (Azure ML Studio)

**Step 1: Navigate to Azure ML Studio**
```
Azure ML Studio → Data → +Create → Data asset
```

**Step 2: Asset Configuration**
- **Asset type**: URI folder
- **Asset name**: `retail_data`
- **Description**: `Retail e-commerce dataset 2019 - Raw CSV files for training`

**Step 3: Data Source Configuration**
- **Data source type**: Azure Data Lake Storage Gen2
- **Storage account**: `retailforecastdev`
- **Container**: `data`
- **Folder path**: `raw/ecommerce/`
- **Authentication**: Managed Identity

**Step 4: Advanced Settings**
- **Skip validation**: ❌ Disabled (verify data integrity)
- **Register new version**: ✅ Enabled
- **Tags**: 
  - `Environment`: `development`
  - `Dataset`: `ecommerce`
  - `Year`: `2019`

### 2.3 Data Asset Properties

<div style="background: #fff7ed; border: 1px solid #fed7aa; border-left: 4px solid #f59e0b; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>📋 Asset Metadata</strong>

**Basic Information:**
- **Name**: `retail_data`
- **Version**: `1`
- **Type**: URI folder
- **Size**: ≥4GB (total dataset size)
- **File Count**: 12 files (2019-*.csv.gz)

**Storage Details:**
- **Storage Account**: `retailforecastdev`
- **Container**: `data`
- **Path**: `raw/ecommerce/`
- **Authentication**: Managed Identity

**Data Schema:**
- **Format**: CSV (compressed with gzip)
- **Encoding**: UTF-8
- **Delimiter**: Comma (,)
- **Headers**: Present

</div>

## 3. DATA ASSET CREATION

### 3.1 Method 1: Azure ML Studio UI (Recommended)

**Advantages**: User-friendly, visual interface, immediate validation
**Best for**: Interactive setup, validation, testing

**Detailed Steps:**
1. **Navigate to Data Assets**
   ```
   Azure ML Studio → Data → Data assets → +Create
   ```

2. **Configure Basic Information**
   - **Asset name**: `retail_data`
   - **Asset type**: URI folder
   - **Description**: `Retail e-commerce dataset 2019 - Raw CSV files for training`

3. **Configure Data Source**
   - **Data source type**: Azure Data Lake Storage Gen2
   - **Subscription**: [Your subscription]
   - **Storage account**: `retailforecastdev`
   - **Container**: `data`
   - **Folder path**: `raw/ecommerce/`

4. **Authentication Settings**
   - **Authentication method**: Managed Identity
   - **Identity**: System-assigned managed identity của Azure ML workspace

5. **Review & Create**
   - Verify all settings
   - Click "Create" to register the asset

### 3.2 Method 2: Azure CLI

```bash
# Create data asset using Azure CLI
az ml data create \
  --name retail_data \
  --version 1 \
  --type uri_folder \
  --path "azureml://datastores/workspaceblobstore/paths/raw/ecommerce/" \
  --description "Retail e-commerce dataset 2019 - Raw CSV files for training" \
  --tags Environment=development Dataset=ecommerce Year=2019
```

**Verify Data Asset Creation:**
```bash
# List data assets
az ml data list --name retail_data --output table

# Show specific asset details
az ml data show --name retail_data --version 1
```

## 4. DATA ASSET VALIDATION

### 4.1 Asset Verification

```bash
# Verify asset creation
az ml data show --name retail_data --version 1 --query "{name:name, version:version, type:type, path:path}"

# Check asset size and file count
az ml data show --name retail_data --version 1 --query "{size:size, fileCount:fileCount}"
```

### 4.2 Data Accessibility Test

```python
# Python script to test data asset accessibility
from azure.ai.ml import MLClient
from azure.ai.ml.entities import Data
from azure.identity import DefaultAzureCredential
import os

def test_data_asset():
    # Initialize ML client
    credential = DefaultAzureCredential()
    ml_client = MLClient(
        credential=credential,
        subscription_id="your-subscription-id",
        resource_group_name="retail-dev-rg",
        workspace_name="retail-ml-workspace"
    )
    
    # Get data asset
    data_asset = ml_client.data.get(name="retail_data", version="1")
    
    print(f"📊 Data Asset Information:")
    print(f"  Name: {data_asset.name}")
    print(f"  Version: {data_asset.version}")
    print(f"  Type: {data_asset.type}")
    print(f"  Path: {data_asset.path}")
    print(f"  Size: {data_asset.size} bytes")
    
    # Test data access
    try:
        # List files in the asset
        files = ml_client.data.list(name="retail_data", version="1")
        print(f"\n📁 Files in dataset:")
        for file in files:
            print(f"  ✅ {file.name}")
        
        print(f"\n✅ Data asset is accessible and ready for training jobs!")
        
    except Exception as e:
        print(f"❌ Error accessing data asset: {e}")

if __name__ == "__main__":
    test_data_asset()
```

### 4.3 Training Job Integration Test

```python
# Test data asset in training job context
from azure.ai.ml import MLClient, command
from azure.ai.ml.entities import Data
from azure.identity import DefaultAzureCredential

def create_test_job():
    credential = DefaultAzureCredential()
    ml_client = MLClient(
        credential=credential,
        subscription_id="your-subscription-id",
        resource_group_name="retail-dev-rg",
        workspace_name="retail-ml-workspace"
    )
    
    # Create test training job using the data asset
    job = command(
        code="./src",
        command="python train.py --data ${{inputs.retail_data}}",
        inputs={
            "retail_data": Data(
                type="uri_folder",
                path="azureml://datastores/workspaceblobstore/paths/raw/ecommerce/"
            )
        },
        environment="azureml://registries/azureml/environments/sklearn-1.0/labels/latest",
        compute="cpu-cluster",
        display_name="test-retail-data-access"
    )
    
    print("✅ Training job created successfully with data asset reference!")
    return job

if __name__ == "__main__":
    create_test_job()
```

## 5. BẰNG CHỨNG HOÀN THÀNH

### 5.1 Ảnh 1: Data Asset Creation Form
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📋 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Azure ML Studio → Data → +Create form</li>
<li>✅ Asset name: <code>retail_data</code></li>
<li>✅ Asset type: URI folder selected</li>
<li>✅ Data source configuration visible</li>
<li>✅ Storage account and path configured</li>
</ul>
</div>

### 5.2 Ảnh 2: Data Asset Card (retail_data:1)
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📊 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Data assets list showing <code>retail_data:1</code></li>
<li>✅ Asset type: URI folder</li>
<li>✅ Version: 1</li>
<li>✅ Size and file count visible</li>
<li>✅ Status: Active/Ready</li>
</ul>
</div>

### 5.3 Ảnh 3: Data Asset Details
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📁 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Data asset details page</li>
<li>✅ Storage path: <code>raw/ecommerce/</code></li>
<li>✅ Authentication method: Managed Identity</li>
<li>✅ File list showing 2019-*.csv.gz files</li>
<li>✅ Total size ≥4GB</li>
</ul>
</div>

### 5.4 Ảnh 4: Data Asset in Training Job
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>🔧 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Training job creation form</li>
<li>✅ Data asset <code>retail_data:1</code> selected as input</li>
<li>✅ Asset path properly referenced</li>
<li>✅ Job configuration showing data integration</li>
</ul>
</div>

### 5.5 Ảnh 5: Data Asset Validation Results
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>✅ Yêu cầu screenshot:</strong>
<ul>
<li>✅ Python validation script output</li>
<li>✅ Asset accessibility confirmation</li>
<li>✅ File count and size verification</li>
<li>✅ Ready for training jobs message</li>
</ul>
</div>

## 6. TIÊU CHÍ HOÀN THÀNH

### 6.1 Functional Requirements
- [x] **Data Asset created**: `retail_data:1` registered in Azure ML Studio
- [x] **Correct path**: Points to `data/raw/ecommerce/` folder
- [x] **Asset type**: URI folder configured properly
- [x] **Versioning**: Version 1 created successfully
- [x] **Metadata**: Proper description and tags applied

### 6.2 Data Integration Requirements
- [x] **Storage integration**: Connected to `retailforecastdev` storage account
- [x] **Authentication**: Managed Identity configured
- [x] **Data accessibility**: Asset accessible from training jobs
- [x] **File validation**: All 2019-*.csv.gz files recognized

### 6.3 Training Job Readiness
- [x] **Job integration**: Asset usable in training job inputs
- [x] **Path resolution**: Correct URI path in job configuration
- [x] **Data loading**: Training scripts can access data via asset reference
- [x] **Version control**: Asset versioning working properly

## 7. AUTOMATION SCRIPTS

### 7.1 Complete Data Asset Creation Script

```bash
#!/bin/bash
# create-data-asset.sh

# Configuration
SUBSCRIPTION_ID="your-subscription-id"
RESOURCE_GROUP="retail-dev-rg"
WORKSPACE_NAME="retail-ml-workspace"
ASSET_NAME="retail_data"
ASSET_VERSION="1"
STORAGE_ACCOUNT="retailforecastdev"
CONTAINER="data"
FOLDER_PATH="raw/ecommerce"

echo "🚀 Creating Data Asset in Azure ML Studio..."

# Login to Azure
az login

# Set subscription
az account set --subscription $SUBSCRIPTION_ID

# Create data asset
az ml data create \
  --name $ASSET_NAME \
  --version $ASSET_VERSION \
  --type uri_folder \
  --path "azureml://datastores/workspaceblobstore/paths/$FOLDER_PATH/" \
  --description "Retail e-commerce dataset 2019 - Raw CSV files for training" \
  --tags Environment=development Dataset=ecommerce Year=2019

# Verify creation
echo "✅ Verifying data asset creation..."
az ml data show --name $ASSET_NAME --version $ASSET_VERSION --output table

echo "📊 Data Asset Information:"
az ml data show --name $ASSET_NAME --version $ASSET_VERSION --query "{name:name, version:version, type:type, path:path, size:size}"

echo "✅ Data asset created successfully!"
```

### 7.2 Python SDK Script

```python
# create_data_asset.py
from azure.ai.ml import MLClient
from azure.ai.ml.entities import Data
from azure.identity import DefaultAzureCredential
import os

def create_data_asset():
    # Configuration
    subscription_id = "your-subscription-id"
    resource_group = "retail-dev-rg"
    workspace_name = "retail-ml-workspace"
    
    # Initialize ML client
    credential = DefaultAzureCredential()
    ml_client = MLClient(
        credential=credential,
        subscription_id=subscription_id,
        resource_group_name=resource_group,
        workspace_name=workspace_name
    )
    
    # Create data asset
    data_asset = Data(
        name="retail_data",
        version="1",
        type="uri_folder",
        path="azureml://datastores/workspaceblobstore/paths/raw/ecommerce/",
        description="Retail e-commerce dataset 2019 - Raw CSV files for training",
        tags={
            "Environment": "development",
            "Dataset": "ecommerce",
            "Year": "2019"
        }
    )
    
    # Register the asset
    ml_client.data.create_or_update(data_asset)
    
    print("✅ Data asset created successfully!")
    
    # Verify creation
    created_asset = ml_client.data.get(name="retail_data", version="1")
    print(f"📊 Asset Details:")
    print(f"  Name: {created_asset.name}")
    print(f"  Version: {created_asset.version}")
    print(f"  Type: {created_asset.type}")
    print(f"  Path: {created_asset.path}")

if __name__ == "__main__":
    create_data_asset()
```

## 8. LƯU Ý QUAN TRỌNG

### 8.1 Data Asset Best Practices
<div style="background: #fffbeb; border: 1px solid #fed7aa; border-left: 4px solid #f59e0b; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>⚠️ Important Considerations</strong>
<ul>
<li><strong>Versioning</strong>: Always use versioning for data changes</li>
<li><strong>Metadata</strong>: Include comprehensive descriptions and tags</li>
<li><strong>Validation</strong>: Test data accessibility before using in jobs</li>
<li><strong>Security</strong>: Use Managed Identity for secure access</li>
<li><strong>Performance</strong>: URI folder is optimal for large datasets</li>
</ul>
</div>

### 8.2 Troubleshooting Common Issues

**Issue 1: "Data asset not found"**
```bash
# Verify asset exists
az ml data list --name retail_data

# Check workspace context
az ml workspace show --name retail-ml-workspace --resource-group retail-dev-rg
```

**Issue 2: "Access denied to storage"**
```bash
# Check Managed Identity permissions
az role assignment list --assignee <ml-workspace-mi-id> --scope /subscriptions/<sub-id>/resourceGroups/retail-dev-rg/providers/Microsoft.Storage/storageAccounts/retailforecastdev
```

**Issue 3: "Invalid data path"**
```bash
# Verify storage account and container
az storage container show --name data --account-name retailforecastdev

# Check folder path exists
az storage blob list --container-name data --prefix "raw/ecommerce/" --account-name retailforecastdev
```

### 8.3 Next Steps
1. ✅ **Data Asset created** ← Current step
2. 🔄 **Training job configuration** ← Next step
3. 🔄 **Model training execution**
4. 🔄 **Model registration**
5. 🔄 **Endpoint deployment**

{{% notice info %}}
**Best Practice**: Always test data asset accessibility before using in training jobs. Use the validation scripts provided to ensure proper integration.
{{% /notice %}}

{{% notice warning %}}
**Security Note**: Ensure Managed Identity has proper permissions to access the storage account. Verify RBAC assignments before creating data assets.
{{% /notice %}}

## 9. USEFUL LINKS

- **Azure ML Data Assets**: https://docs.microsoft.com/en-us/azure/machine-learning/concept-data
- **Data Asset Creation**: https://docs.microsoft.com/en-us/azure/machine-learning/how-to-create-register-data-assets
- **Data Versioning**: https://docs.microsoft.com/en-us/azure/machine-learning/how-to-version-track-datasets
- **Managed Identity**: https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/

Data Asset registration hoàn tất! 🎉 Asset đã sẵn sàng cho **Task 5: Training Job Configuration**.
