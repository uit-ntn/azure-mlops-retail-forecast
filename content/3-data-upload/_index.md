---
title: "Data Upload & Storage Configuration"
date: 2025-08-30T11:20:00+07:00
weight: 3
chapter: false
pre: "<b>3. </b>"
---

Trong phần này, chúng ta sẽ thiết lập Azure Storage Account và upload dữ liệu ecommerce dataset vào container `data/raw/ecommerce/`. Đây là bước quan trọng để chuẩn bị dữ liệu cho quá trình training và validation trong Azure ML pipeline. Chúng ta sẽ sử dụng Azure Portal UI và AzCopy để upload các file CSV.gz với tổng dung lượng ≥4GB.

## 1. Mục tiêu

✅ **Tạo Azure Storage Account với hierarchical namespace (ADLS Gen2)**  
✅ **Upload dataset 2019-*.csv.gz vào container data/raw/ecommerce/**  
✅ **Đảm bảo dữ liệu sẵn sàng cho Azure ML training**  
✅ **Cấu hình proper access permissions và security**

## 2. Dataset Overview

**Retail E-commerce Dataset (2019)**
- **Source**: Kaggle E-commerce Dataset
- **Format**: CSV files compressed với gzip (.csv.gz)
- **Time Range**: 2019 (12 tháng dữ liệu)
- **Size**: ≥4GB tổng dung lượng
- **Structure**: Transaction data với product, user, category information

<div style="background: linear-gradient(135deg, #e6f3ff 0%, #dbeafe 100%); border: 1px solid #bfdbfe; border-left: 6px solid #0078d4; padding: 18px; margin: 18px 0; border-radius: 10px;">
  <strong>Mục tiêu</strong>: Tải các file tháng 2019-*.csv.gz vào Storage data/raw/ecommerce/. UI: Azure Portal / Storage Explorer (hoặc AzCopy). Bằng chứng: Ảnh data/raw/ecommerce/ hiển thị các .csv.gz (≥4GB tổng). Done: Dữ liệu sẵn sàng đọc.
</div>

## 3. AZURE STORAGE SETUP

### 3.1 Storage Account Configuration

<div style="background: #f8fafc; border: 1px solid #e2e8f0; border-left: 4px solid #0078d4; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>🔧 Storage Account Settings</strong>

**1. Basic Configuration:**
- **Storage account name**: `retailforecastdev` (unique globally)
- **Performance**: Standard (cost-effective cho large datasets)
- **Redundancy**: LRS (Locally-redundant storage)
- **Location**: Southeast Asia (matching Resource Group)

**2. Advanced Features:**
- **Hierarchical namespace**: ✅ Enabled (ADLS Gen2)
- **Access tier**: Hot (frequent access)
- **Blob soft delete**: ✅ Enabled (7 days retention)
- **Versioning**: ✅ Enabled (data protection)
</div>

### 3.2 UI Flow (Azure Portal)

**Step 1: Create Storage Account**
```
Azure Portal → Storage accounts → Create storage account
```

**Step 2: Basic Configuration**
- **Subscription**: [Your subscription]
- **Resource group**: `retail-dev-rg`
- **Storage account name**: `retailforecastdev`
- **Region**: Southeast Asia
- **Performance**: Standard
- **Redundancy**: Locally-redundant storage (LRS)

**Step 3: Advanced Settings**
- **Hierarchical namespace**: ✅ Enable (ADLS Gen2)
- **Access tier**: Hot
- **Blob soft delete**: ✅ Enable (7 days)
- **Versioning**: ✅ Enable

**Step 4: Networking & Security**
- **Network access**: Allow access from all networks (dev environment)
- **Secure transfer required**: ✅ Enabled
- **Allow Blob public access**: ❌ Disabled (security)

### 3.3 Container Structure Setup

Sau khi tạo Storage Account, tạo container structure:

```bash
# Container hierarchy
data/
├── raw/
│   └── ecommerce/          # Raw CSV.gz files
├── processed/
│   └── features/           # Processed features
├── models/
│   └── artifacts/          # Model artifacts
└── logs/
    └── training/           # Training logs
```

## 4. DATA UPLOAD METHODS

### 4.1 Method 1: Azure Portal (Storage Explorer)

**Advantages**: User-friendly, visual interface, drag-and-drop
**Best for**: Small datasets, manual uploads, testing

**Steps:**
1. Navigate to Storage Account → Containers
2. Create container `data` (if not exists)
3. Create folder structure: `raw/ecommerce/`
4. Upload CSV.gz files via drag-and-drop
5. Verify file sizes and permissions

### 4.2 Method 2: AzCopy (Recommended)

**Advantages**: High performance, resume capability, batch operations
**Best for**: Large datasets, automation, production uploads

```bash
# Install AzCopy (if not installed)
# Windows
winget install Microsoft.AzCopy

# macOS
brew install azcopy

# Linux
wget https://aka.ms/downloadazcopy-v10-linux
tar -xzf downloadazcopy-v10-linux.tar.gz
sudo ./install.sh
```

**Authentication Setup:**
```bash
# Login to Azure
az login

# Get storage account key
STORAGE_KEY=$(az storage account keys list \
  --resource-group retail-dev-rg \
  --account-name retailforecastdev \
  --query '[0].value' -o tsv)

# Set environment variable
export AZCOPY_STORAGE_KEY=$STORAGE_KEY
```

**Upload Commands:**
```bash
# Upload single file
azcopy copy "2019-Jan.csv.gz" \
  "https://retailforecastdev.dfs.core.windows.net/data/raw/ecommerce/2019-Jan.csv.gz" \
  --recursive=false

# Upload multiple files (batch)
azcopy copy "./datasets/" \
  "https://retailforecastdev.dfs.core.windows.net/data/raw/ecommerce/" \
  --recursive=true \
  --include-pattern="2019-*.csv.gz"

# Upload with progress and resume capability
azcopy copy "./datasets/" \
  "https://retailforecastdev.dfs.core.windows.net/data/raw/ecommerce/" \
  --recursive=true \
  --include-pattern="2019-*.csv.gz" \
  --log-level=INFO \
  --overwrite=prompt
```


## 5. SECURITY & ACCESS CONTROL

### 5.1 RBAC Permissions

**Storage Account Level:**
- **ML Engineers**: `Storage Blob Data Contributor`
- **Data Scientists**: `Storage Blob Data Reader`
- **DevOps**: `Storage Account Contributor`
- **Azure ML Workspace MI**: `Storage Blob Data Reader`

### 5.2 Access Policies

```bash
# Assign Storage Blob Data Reader to Azure ML Managed Identity
az role assignment create \
  --assignee <aml-workspace-managed-identity-id> \
  --role "Storage Blob Data Reader" \
  --scope "/subscriptions/<subscription-id>/resourceGroups/retail-dev-rg/providers/Microsoft.Storage/storageAccounts/retailforecastdev"

# Assign Storage Blob Data Contributor to ML team
az role assignment create \
  --assignee <ml-team-object-id> \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/<subscription-id>/resourceGroups/retail-dev-rg/providers/Microsoft.Storage/storageAccounts/retailforecastdev"
```

### 5.3 Network Security

**Development Environment:**
- Allow access from all networks (for development)
- Enable secure transfer (HTTPS only)
- Disable public blob access

**Production Considerations:**
- Private endpoints for secure access
- VNet integration
- IP whitelisting for specific ranges

## 6. DATA VALIDATION

### 6.1 File Verification

```bash
# List uploaded files
az storage blob list \
  --account-name retailforecastdev \
  --container-name data \
  --prefix "raw/ecommerce/" \
  --output table

# Check file sizes
az storage blob list \
  --account-name retailforecastdev \
  --container-name data \
  --prefix "raw/ecommerce/" \
  --query "[].{Name:name, Size:properties.contentLength}" \
  --output table

# Calculate total size
az storage blob list \
  --account-name retailforecastdev \
  --container-name data \
  --prefix "raw/ecommerce/" \
  --query "[].properties.contentLength" \
  --output tsv | awk '{sum += $1} END {print "Total size:", sum/1024/1024/1024, "GB"}'
```

### 6.2 Data Quality Checks

```python
# Python script to validate uploaded data
from azure.storage.filedatalake import DataLakeServiceClient
from azure.identity import DefaultAzureCredential
import pandas as pd
import gzip

def validate_uploaded_data():
    credential = DefaultAzureCredential()
    service_client = DataLakeServiceClient(
        account_url="https://retailforecastdev.dfs.core.windows.net",
        credential=credential
    )
    
    file_system_client = service_client.get_file_system_client("data")
    directory_client = file_system_client.get_directory_client("raw/ecommerce")
    
    # List files
    files = list(directory_client.list_paths())
    
    print(f"📁 Found {len(files)} files:")
    total_size = 0
    
    for file_path in files:
        file_client = directory_client.get_file_client(file_path.name)
        properties = file_client.get_file_properties()
        size_mb = properties.size / (1024 * 1024)
        total_size += properties.size
        
        print(f"  ✅ {file_path.name}: {size_mb:.2f} MB")
        
        # Quick validation - try to read first few rows
        try:
            download = file_client.download_file()
            with gzip.open(download, 'rt') as f:
                # Read first 5 lines
                for i, line in enumerate(f):
                    if i >= 5:
                        break
                print(f"    📊 File format valid")
        except Exception as e:
            print(f"    ❌ Error reading file: {e}")
    
    total_gb = total_size / (1024 * 1024 * 1024)
    print(f"\n📈 Total dataset size: {total_gb:.2f} GB")
    
    if total_gb >= 4.0:
        print("✅ Dataset size requirement met (≥4GB)")
    else:
        print("⚠️  Dataset size below requirement (<4GB)")

if __name__ == "__main__":
    validate_uploaded_data()
```

## 7. BẰNG CHỨNG HOÀN THÀNH

### 7.1 Ảnh 1: Storage Account Overview
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📋 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Storage Account overview page</li>
<li>✅ Account name: <code>retailforecastdev</code></li>
<li>✅ Status: <code>Active</code></li>
<li>✅ Hierarchical namespace: <code>Enabled</code></li>
<li>✅ Access tier: <code>Hot</code></li>
</ul>
</div>

### 7.2 Ảnh 2: Container Structure
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📁 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Container <code>data</code> created</li>
<li>✅ Folder structure: <code>raw/ecommerce/</code></li>
<li>✅ All CSV.gz files visible</li>
<li>✅ File sizes displayed</li>
</ul>
</div>

### 7.3 Ảnh 3: File Details & Total Size
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📊 Yêu cầu screenshot:</strong>
<ul>
<li>✅ List of all 2019-*.csv.gz files</li>
<li>✅ Individual file sizes visible</li>
<li>✅ Total size calculation (≥4GB)</li>
<li>✅ Upload timestamps</li>
</ul>
</div>

### 7.4 Ảnh 4: Access Control (IAM)
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>🔐 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Storage Account → Access control (IAM)</li>
<li>✅ Role assignments visible</li>
<li>✅ Azure ML Managed Identity permissions</li>
<li>✅ Team member permissions</li>
</ul>
</div>

### 7.5 Ảnh 5: AzCopy Upload Progress
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📤 Yêu cầu screenshot:</strong>
<ul>
<li>✅ AzCopy command execution</li>
<li>✅ Upload progress showing files being transferred</li>
<li>✅ Transfer speed and completion percentage</li>
<li>✅ Final success message</li>
</ul>
</div>

### 7.6 Ảnh 6: Storage Account Properties
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>⚙️ Yêu cầu screenshot:</strong>
<ul>
<li>✅ Storage Account → Properties</li>
<li>✅ Hierarchical namespace: Enabled</li>
<li>✅ Access tier: Hot</li>
<li>✅ Replication: Locally-redundant storage (LRS)</li>
<li>✅ Secure transfer required: Enabled</li>
</ul>
</div>

### 7.7 Ảnh 7: Data Validation Results
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📊 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Python validation script output</li>
<li>✅ File count and individual sizes</li>
<li>✅ Total size calculation (≥4GB)</li>
<li>✅ File format validation results</li>
</ul>
</div>

## 8. TIÊU CHÍ HOÀN THÀNH

### 8.1 Functional Requirements
- [x] **Storage Account created**: `retailforecastdev` với ADLS Gen2 enabled
- [x] **Container structure**: `data/raw/ecommerce/` folder created
- [x] **Files uploaded**: All 2019-*.csv.gz files uploaded successfully
- [x] **Size requirement**: Total dataset size ≥4GB
- [x] **Access permissions**: Proper RBAC configured

### 8.2 Data Quality Requirements
- [x] **File integrity**: All files uploaded without corruption
- [x] **Format validation**: CSV.gz format verified
- [x] **Accessibility**: Files accessible by Azure ML workspace
- [x] **Security**: Secure transfer enabled, public access disabled

### 8.3 Performance Requirements
- [x] **Upload performance**: Efficient upload method used (AzCopy recommended)
- [x] **Storage tier**: Hot access tier for frequent access
- [x] **Network optimization**: Optimal region selection (Southeast Asia)

## 9. AUTOMATION SCRIPTS

### 9.1 Complete Upload Script

```bash
#!/bin/bash
# upload-dataset.sh

# Configuration
STORAGE_ACCOUNT="retailforecastdev"
RESOURCE_GROUP="retail-dev-rg"
CONTAINER_NAME="data"
DATASET_PATH="./datasets"
TARGET_PATH="raw/ecommerce"

echo "🚀 Starting dataset upload to Azure Storage..."

# Get storage account key
echo "📋 Getting storage account key..."
STORAGE_KEY=$(az storage account keys list \
  --resource-group $RESOURCE_GROUP \
  --account-name $STORAGE_ACCOUNT \
  --query '[0].value' -o tsv)

if [ -z "$STORAGE_KEY" ]; then
    echo "❌ Failed to get storage account key"
    exit 1
fi

# Create container if not exists
echo "📁 Creating container if not exists..."
az storage container create \
  --account-name $STORAGE_ACCOUNT \
  --account-key $STORAGE_KEY \
  --name $CONTAINER_NAME

# Upload files using AzCopy
echo "📤 Uploading files using AzCopy..."
azcopy copy "$DATASET_PATH/" \
  "https://$STORAGE_ACCOUNT.dfs.core.windows.net/$CONTAINER_NAME/$TARGET_PATH/" \
  --recursive=true \
  --include-pattern="2019-*.csv.gz" \
  --log-level=INFO

# Verify upload
echo "✅ Verifying upload..."
az storage blob list \
  --account-name $STORAGE_ACCOUNT \
  --account-key $STORAGE_KEY \
  --container-name $CONTAINER_NAME \
  --prefix "$TARGET_PATH/" \
  --output table

# Calculate total size
echo "📊 Calculating total size..."
TOTAL_SIZE=$(az storage blob list \
  --account-name $STORAGE_ACCOUNT \
  --account-key $STORAGE_KEY \
  --container-name $CONTAINER_NAME \
  --prefix "$TARGET_PATH/" \
  --query "[].properties.contentLength" \
  --output tsv | awk '{sum += $1} END {print sum/1024/1024/1024}')

echo "📈 Total dataset size: ${TOTAL_SIZE} GB"

if (( $(echo "$TOTAL_SIZE >= 4.0" | bc -l) )); then
    echo "✅ Dataset upload completed successfully!"
    echo "✅ Size requirement met (≥4GB)"
else
    echo "⚠️  Warning: Dataset size below requirement (<4GB)"
fi
```

### 9.2 Terraform Configuration

```hcl
# storage.tf
resource "azurerm_storage_account" "main" {
  name                     = "retailforecastdev"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  account_kind             = "StorageV2"
  
  # Enable ADLS Gen2
  is_hns_enabled = true
  
  # Security settings
  min_tls_version                 = "TLS1_2"
  allow_nested_items_to_be_public = false
  
  # Enable versioning and soft delete
  versioning_enabled = true
  blob_properties {
    delete_retention_policy {
      days = 7
    }
    versioning_enabled = true
  }
  
  tags = {
    Environment        = "development"
    Project           = "retail-forecast"
    Purpose           = "data-storage"
    DataClassification = "internal"
  }
}

resource "azurerm_storage_container" "data" {
  name                  = "data"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}
```

## 10. LƯU Ý QUAN TRỌNG

### 10.1 Cost Optimization
<div style="background: #fffbeb; border: 1px solid #fed7aa; border-left: 4px solid #f59e0b; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>💰 Storage Cost Considerations</strong>
<ul>
<li><strong>Access Tier</strong>: Hot tier for frequent access during development</li>
<li><strong>Lifecycle Management</strong>: Consider Cool/Archive tiers for older data</li>
<li><strong>Compression</strong>: CSV.gz already compressed, good for storage efficiency</li>
<li><strong>Redundancy</strong>: LRS sufficient for development, consider GRS for production</li>
</ul>
</div>

### 10.2 Data Governance
- **Data Lineage**: Track data sources and transformations
- **Retention Policies**: Implement data retention rules
- **Access Logging**: Monitor data access patterns
- **Compliance**: Ensure data handling meets regulatory requirements

### 10.3 Next Steps
1. ✅ **Data Upload completed** ← Current step
2. 🔄 **Azure ML Workspace setup** ← Next step
3. 🔄 **Data Asset registration**
4. 🔄 **Training pipeline configuration**
5. 🔄 **Model training execution**

{{% notice info %}}
**Best Practice**: Use AzCopy for large file uploads (>100MB) as it provides better performance, resume capability, and progress tracking compared to Azure Portal uploads.
{{% /notice %}}

{{% notice warning %}}
**Security Note**: Never store storage account keys in code or configuration files. Use Azure Key Vault or Managed Identity for secure access in production environments.
{{% /notice %}}

## 11. USEFUL LINKS

- **Azure Storage Documentation**: https://docs.microsoft.com/en-us/azure/storage/
- **AzCopy Documentation**: https://docs.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-v10
- **ADLS Gen2 Best Practices**: https://docs.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-best-practices
- **Storage Security Guide**: https://docs.microsoft.com/en-us/azure/storage/common/security-recommendations

Data upload hoàn tất! 🎉 Dữ liệu đã sẵn sàng cho **Task 4: Azure ML Workspace Setup**.
