---
title: "ETL Aggregate Pipeline"
date: 2025-08-30T11:40:00+07:00
weight: 5
chapter: false
pre: "<b>5. </b>"
---

Trong phần này, chúng ta sẽ xây dựng ETL pipeline để chuyển đổi dữ liệu clickstream thô thành chuỗi thời gian có cấu trúc cho machine learning. Pipeline sẽ thực hiện data cleaning, aggregation, feature engineering và output dữ liệu dưới dạng Parquet partitioned để tối ưu performance. Đây là bước quan trọng để chuẩn bị dữ liệu phù hợp cho việc training forecasting models.

## 1. Mục tiêu

✅ **Chuyển clickstream thành chuỗi thời gian ngày × KEY**  
✅ **Tạo features và labels phù hợp cho forecasting**  
✅ **Thực hiện data quality checks và validation**  
✅ **Output Parquet partitioned cho performance tối ưu**

<div style="background: linear-gradient(135deg, #e6f3ff 0%, #dbeafe 100%); border: 1px solid #bfdbfe; border-left: 6px solid #0078d4; padding: 18px; margin: 18px 0; border-radius: 10px;">
  <strong>Mục tiêu</strong>: Chuyển clickstream thành chuỗi thời gian ngày × KEY (KEY ∈ {product_id, category_code, brand}). Thiết kế: Nhãn: purchases_t = count(event_type == 'purchase') theo ngày × KEY. Biến ngoại sinh: views_t, carts_t, price_mean_t. Conversion metrics: conv_rate = purchases_t / views_t. Lag features: purchases_t−1, purchases_t−7. Moving averages: MA7, MA28. Calendar features: day_of_week, is_weekend, holiday_flag.
</div>

## 2. ETL PIPELINE DESIGN

### 2.1 Data Schema Design

<div style="background: #f8fafc; border: 1px solid #e2e8f0; border-left: 4px solid #0078d4; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>🔧 Time Series Schema</strong>

**Input Schema (Raw Clickstream):**
- **event_time**: Timestamp của event
- **event_type**: 'view', 'cart', 'purchase'
- **product_id**: ID sản phẩm
- **category_code**: Mã danh mục
- **brand**: Tên thương hiệu
- **price**: Giá sản phẩm
- **user_id**: ID người dùng

**Output Schema (Daily Aggregated):**
- **date**: Ngày (YYYY-MM-DD)
- **key_type**: Loại key ('product_id', 'category_code', 'brand')
- **key_value**: Giá trị key cụ thể
- **purchases_t**: Số lượng purchase trong ngày
- **views_t**: Số lượng view trong ngày
- **carts_t**: Số lượng cart trong ngày
- **price_mean_t**: Giá trung bình trong ngày
- **conv_rate**: Tỷ lệ conversion (purchases/views)

</div>

### 2.2 Feature Engineering Strategy

<div style="background: #fff7ed; border: 1px solid #fed7aa; border-left: 4px solid #f59e0b; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>📊 Feature Categories</strong>

**1. Target Labels:**
- **purchases_t**: `count(event_type == 'purchase')` theo ngày × KEY

**2. Exogenous Variables:**
- **views_t**: Số lượng view events
- **carts_t**: Số lượng cart events  
- **price_mean_t**: Giá trung bình của sản phẩm

**3. Conversion Metrics:**
- **conv_rate**: `purchases_t / views_t`
- **cart_rate**: `carts_t / views_t`

**4. Lag Features:**
- **purchases_t-1**: Purchase của ngày trước
- **purchases_t-7**: Purchase của 7 ngày trước
- **views_t-1**: View của ngày trước

**5. Moving Averages:**
- **MA7**: Moving average 7 ngày
- **MA28**: Moving average 28 ngày

**6. Calendar Features:**
- **day_of_week**: Thứ trong tuần (0-6)
- **is_weekend**: Có phải cuối tuần
- **holiday_flag**: Có phải ngày lễ (từ external calendar)

</div>

### 2.3 ETL Pipeline Steps

<div style="background: #f1f5f9; border: 1px solid #e2e8f0; border-left: 4px solid #6366f1; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>🔄 Pipeline Workflow</strong>

**Step 1: Data Cleaning**
- Convert timezone to UTC
- Remove invalid records (price ≤ 0, empty IDs)
- Filter unknown event_types
- Validate data types

**Step 2: Aggregation**
- Group by (date, KEY) where KEY ∈ {product_id, category_code, brand}
- Calculate daily metrics: purchases, views, carts, price_mean
- Compute conversion rates

**Step 3: Feature Engineering**
- Generate lag features (t-1, t-7)
- Calculate moving averages (MA7, MA28)
- Add calendar features

**Step 4: Quality Checks**
- Log min/max dates
- Count unique keys
- Report dropped records percentage
- Validate data consistency

**Step 5: Output**
- Write Parquet files partitioned by key_type/date
- Register new Data Asset: retail_aggregates:1

</div>

## 3. AZURE ML STUDIO JOB CONFIGURATION

### 3.1 Prerequisites

**Required Components:**
- ✅ **Azure ML Workspace** đã tạo (từ Task 6 - Azure ML Workspace Setup)
- ✅ **Data Asset** `retail_data:1` (từ **Task 4: Data Asset Registration**)
- ✅ **Compute cluster** `cpu-cluster` (từ Task 6)
- ✅ **Environment** `retail-train-env` với pandas/dask (từ Task 6)
- ✅ **Storage Account** với datastore configured (từ **Task 3: Data Upload**)

### 3.2 Job Configuration

<div style="background: #f8fafc; border: 1px solid #e2e8f0; border-left: 4px solid #0078d4; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>🔧 Command Job Settings</strong>

**Basic Configuration:**
- **Job name**: `etl_aggregate`
- **Job type**: Command job
- **Environment**: `retail-train-env` (pandas/dask)
- **Compute**: `cpu-cluster`
- **Display name**: `ETL Aggregate Pipeline`

**Inputs:**
- **retail_data**: Data Asset `retail_data:1`
- **key_type**: Parameter `product_id|category_code|brand`

**Outputs:**
- **aggregates**: Folder `aggregates/` in datastore
- **Data Asset**: `retail_aggregates:1` (auto-registered)

**Command:**
```bash
python etl_aggregate.py --input_data ${{inputs.retail_data}} --key_type ${{inputs.key_type}} --output_path ${{outputs.aggregates}}
```

</div>

### 3.3 UI Flow (Azure ML Studio)

**Step 1: Navigate to Jobs**
```
Azure ML Studio → Jobs → +New → Command job
```

**Step 2: Basic Configuration**
- **Job name**: `etl_aggregate`
- **Environment**: `retail-train-env`
- **Compute**: `cpu-cluster`
- **Display name**: `ETL Aggregate Pipeline`

**Step 3: Inputs Configuration**
- **retail_data**: Select Data Asset `retail_data:1`
- **key_type**: Parameter with options `product_id`, `category_code`, `brand`

**Step 4: Outputs Configuration**
- **aggregates**: Output folder `aggregates/`
- **Auto-register**: Create Data Asset `retail_aggregates:1`

**Step 5: Command Configuration**
```bash
python etl_aggregate.py \
  --input_data ${{inputs.retail_data}} \
  --key_type ${{inputs.key_type}} \
  --output_path ${{outputs.aggregates}}
```

## 4. ETL IMPLEMENTATION

### 4.1 Python ETL Script

```python
# etl_aggregate.py
import pandas as pd
import dask.dataframe as dd
import numpy as np
import argparse
from datetime import datetime, timedelta
import logging
from pathlib import Path

def setup_logging():
    """Setup logging configuration"""
    logging.basicConfig(
        level=logging.INFO,
        format='%(asctime)s - %(levelname)s - %(message)s'
    )
    return logging.getLogger(__name__)

def load_and_clean_data(input_path, logger):
    """Load and clean raw clickstream data"""
    logger.info("Loading raw clickstream data...")
    
    # Load data with Dask for large datasets
    df = dd.read_parquet(f"{input_path}/**/*.parquet")
    
    logger.info(f"Loaded {len(df)} records")
    
    # Data cleaning
    logger.info("Performing data cleaning...")
    
    # Convert to pandas for cleaning operations
    df = df.compute()
    
    # Convert timezone to UTC
    df['event_time'] = pd.to_datetime(df['event_time']).dt.tz_localize('UTC')
    
    # Remove invalid records
    initial_count = len(df)
    
    # Filter out invalid prices
    df = df[df['price'] > 0]
    
    # Filter out empty IDs
    df = df[df['product_id'].notna()]
    df = df[df['product_id'] != '']
    
    # Filter valid event types
    valid_events = ['view', 'cart', 'purchase']
    df = df[df['event_type'].isin(valid_events)]
    
    # Extract date
    df['date'] = df['event_time'].dt.date
    
    final_count = len(df)
    dropped_pct = (initial_count - final_count) / initial_count * 100
    
    logger.info(f"Dropped {initial_count - final_count} records ({dropped_pct:.2f}%)")
    logger.info(f"Final dataset: {final_count} records")
    
    return df

def aggregate_by_key(df, key_type, logger):
    """Aggregate data by date and key"""
    logger.info(f"Aggregating by {key_type}...")
    
    # Group by date and key
    grouped = df.groupby(['date', key_type]).agg({
        'event_type': [
            ('purchases', lambda x: (x == 'purchase').sum()),
            ('views', lambda x: (x == 'view').sum()),
            ('carts', lambda x: (x == 'cart').sum())
        ],
        'price': 'mean'
    }).reset_index()
    
    # Flatten column names
    grouped.columns = ['date', 'key_value', 'purchases_t', 'views_t', 'carts_t', 'price_mean_t']
    
    # Add key type
    grouped['key_type'] = key_type
    
    # Calculate conversion rates
    grouped['conv_rate'] = grouped['purchases_t'] / grouped['views_t'].replace(0, np.nan)
    grouped['cart_rate'] = grouped['carts_t'] / grouped['views_t'].replace(0, np.nan)
    
    # Sort by date and key
    grouped = grouped.sort_values(['date', 'key_value'])
    
    logger.info(f"Aggregated to {len(grouped)} records")
    logger.info(f"Date range: {grouped['date'].min()} to {grouped['date'].max()}")
    logger.info(f"Unique keys: {grouped['key_value'].nunique()}")
    
    return grouped

def add_features(df, logger):
    """Add lag features and moving averages"""
    logger.info("Adding engineered features...")
    
    # Sort by key_value and date
    df = df.sort_values(['key_value', 'date']).reset_index(drop=True)
    
    # Add lag features
    df['purchases_t-1'] = df.groupby('key_value')['purchases_t'].shift(1)
    df['purchases_t-7'] = df.groupby('key_value')['purchases_t'].shift(7)
    df['views_t-1'] = df.groupby('key_value')['views_t'].shift(1)
    
    # Add moving averages
    df['MA7'] = df.groupby('key_value')['purchases_t'].rolling(window=7, min_periods=1).mean().reset_index(0, drop=True)
    df['MA28'] = df.groupby('key_value')['purchases_t'].rolling(window=28, min_periods=1).mean().reset_index(0, drop=True)
    
    # Add calendar features
    df['date'] = pd.to_datetime(df['date'])
    df['day_of_week'] = df['date'].dt.dayofweek
    df['is_weekend'] = df['day_of_week'].isin([5, 6])
    
    # Add holiday flag (simplified - can be enhanced with external calendar)
    df['holiday_flag'] = 0  # Placeholder for holiday logic
    
    logger.info("Feature engineering completed")
    
    return df

def quality_checks(df, logger):
    """Perform data quality checks"""
    logger.info("Performing quality checks...")
    
    # Log statistics
    logger.info(f"Dataset statistics:")
    logger.info(f"  Total records: {len(df)}")
    logger.info(f"  Date range: {df['date'].min()} to {df['date'].max()}")
    logger.info(f"  Unique keys: {df['key_value'].nunique()}")
    logger.info(f"  Key type: {df['key_type'].iloc[0]}")
    
    # Check for missing values
    missing_counts = df.isnull().sum()
    if missing_counts.sum() > 0:
        logger.warning(f"Missing values found:")
        for col, count in missing_counts[missing_counts > 0].items():
            logger.warning(f"  {col}: {count}")
    
    # Check data consistency
    negative_values = (df[['purchases_t', 'views_t', 'carts_t']] < 0).sum().sum()
    if negative_values > 0:
        logger.error(f"Found {negative_values} negative values in count columns")
    
    logger.info("Quality checks completed")

def save_output(df, output_path, logger):
    """Save aggregated data as partitioned Parquet files"""
    logger.info(f"Saving output to {output_path}...")
    
    # Create output directory
    Path(output_path).mkdir(parents=True, exist_ok=True)
    
    # Convert date back to string for partitioning
    df['date_str'] = df['date'].dt.strftime('%Y-%m-%d')
    
    # Save as partitioned Parquet
    df.to_parquet(
        output_path,
        partition_cols=['key_type', 'date_str'],
        index=False
    )
    
    logger.info("Output saved successfully")
    
    # List output files
    output_files = list(Path(output_path).rglob('*.parquet'))
    logger.info(f"Created {len(output_files)} Parquet files")

def main():
    parser = argparse.ArgumentParser(description='ETL Aggregate Pipeline')
    parser.add_argument('--input_data', required=True, help='Input data path')
    parser.add_argument('--key_type', required=True, choices=['product_id', 'category_code', 'brand'], help='Key type for aggregation')
    parser.add_argument('--output_path', required=True, help='Output path')
    
    args = parser.parse_args()
    
    # Setup logging
    logger = setup_logging()
    
    try:
        # Load and clean data
        df = load_and_clean_data(args.input_data, logger)
        
        # Aggregate by key
        df_agg = aggregate_by_key(df, args.key_type, logger)
        
        # Add features
        df_features = add_features(df_agg, logger)
        
        # Quality checks
        quality_checks(df_features, logger)
        
        # Save output
        save_output(df_features, args.output_path, logger)
        
        logger.info("ETL pipeline completed successfully!")
        
    except Exception as e:
        logger.error(f"ETL pipeline failed: {str(e)}")
        raise

if __name__ == "__main__":
    main()
```

### 4.2 Environment Configuration

```yaml
# retail-train-env.yml
name: retail-train-env
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.8
  - pandas>=1.3.0
  - dask>=2021.0.0
  - numpy>=1.21.0
  - pyarrow>=5.0.0
  - fastparquet>=0.7.0
  - pip
  - pip:
    - azure-ai-ml>=1.0.0
    - azure-identity>=1.7.0
```

## 5. JOB EXECUTION & MONITORING

### 5.1 Job Submission

```bash
# Submit job via Azure CLI
az ml job create \
  --file etl_aggregate_job.yml \
  --resource-group retail-dev-rg \
  --workspace-name retail-ml-workspace
```

### 5.2 Job Configuration YAML

```yaml
# etl_aggregate_job.yml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
command: >
  python etl_aggregate.py
  --input_data ${{inputs.retail_data}}
  --key_type ${{inputs.key_type}}
  --output_path ${{outputs.aggregates}}

code: ./src
environment: azureml://registries/azureml/environments/retail-train-env/labels/latest
compute: azureml://subscriptions/{subscription-id}/resourceGroups/retail-dev-rg/providers/Microsoft.MachineLearningServices/workspaces/retail-ml-workspace/computes/cpu-cluster

inputs:
  retail_data:
    type: uri_folder
    path: azureml://datastores/workspaceblobstore/paths/raw/ecommerce/
  key_type:
    type: string
    default: product_id

outputs:
  aggregates:
    type: uri_folder

display_name: ETL Aggregate Pipeline
description: Transform clickstream data to time series format
tags:
  task: etl
  pipeline: aggregate
  dataset: retail
```

### 5.3 Monitoring Job Execution

```python
# monitor_job.py
from azure.ai.ml import MLClient
from azure.identity import DefaultAzureCredential
import time

def monitor_job(job_name):
    credential = DefaultAzureCredential()
    ml_client = MLClient(
        credential=credential,
        subscription_id="your-subscription-id",
        resource_group_name="retail-dev-rg",
        workspace_name="retail-ml-workspace"
    )
    
    job = ml_client.jobs.get(job_name)
    
    print(f"Job Status: {job.status}")
    print(f"Start Time: {job.creation_context.created_at}")
    
    while job.status in ['NotStarted', 'Starting', 'Preparing', 'Running']:
        time.sleep(30)
        job = ml_client.jobs.get(job_name)
        print(f"Status: {job.status}")
        
        if job.status == 'Failed':
            print("Job failed!")
            break
        elif job.status == 'Completed':
            print("Job completed successfully!")
            break
    
    return job

if __name__ == "__main__":
    job = monitor_job("etl_aggregate")
```

## 6. BẰNG CHỨNG HOÀN THÀNH

### 6.1 Ảnh 1: Job Creation Form
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📋 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Azure ML Studio → Jobs → +New → Command job</li>
<li>✅ Job name: <code>etl_aggregate</code></li>
<li>✅ Environment: <code>retail-train-env</code></li>
<li>✅ Compute: <code>cpu-cluster</code></li>
<li>✅ Input data asset: <code>retail_data:1</code></li>
</ul>
</div>

### 6.2 Ảnh 2: Job Run Succeeded
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>✅ Yêu cầu screenshot:</strong>
<ul>
<li>✅ Jobs list showing <code>etl_aggregate</code></li>
<li>✅ Status: <code>Completed</code></li>
<li>✅ Duration and execution time visible</li>
<li>✅ Input/Output assets linked</li>
</ul>
</div>

### 6.3 Ảnh 3: Output Directory Structure
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📁 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Storage Account → containers → data → aggregates/</li>
<li>✅ Partitioned structure: key_type=.../date=YYYY-MM-DD/</li>
<li>✅ Parquet files visible in partitions</li>
<li>✅ File sizes and counts displayed</li>
</ul>
</div>

### 6.4 Ảnh 4: Job Logs & Statistics
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📊 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Job logs showing execution steps</li>
<li>✅ Statistics: số key, min/max ngày</li>
<li>✅ Tỷ lệ drop records</li>
<li>✅ Quality checks results</li>
</ul>
</div>

### 6.5 Ảnh 5: New Data Asset Registration
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📊 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Data Assets list showing <code>retail_aggregates:1</code></li>
<li>✅ Asset type: URI folder</li>
<li>✅ Path: <code>aggregates/</code></li>
<li>✅ Status: Active/Ready</li>
</ul>
</div>

## 7. TIÊU CHÍ HOÀN THÀNH

### 7.1 Functional Requirements
- [x] **Job executed**: `etl_aggregate` Run Succeeded
- [x] **Data transformation**: Clickstream → Time series completed
- [x] **Features created**: Labels, exogenous variables, lag features, moving averages
- [x] **Output format**: Parquet partitioned by key_type/date
- [x] **Data Asset**: `retail_aggregates:1` registered

### 7.2 Data Quality Requirements
- [x] **Data cleaning**: Invalid records removed, timezone converted
- [x] **Aggregation**: Daily metrics calculated correctly
- [x] **Feature engineering**: Lag features and moving averages computed
- [x] **Quality checks**: Statistics logged, validation passed

### 7.3 Performance Requirements
- [x] **Partitioning**: Optimized I/O with partitioned structure
- [x] **Scalability**: Dask used for large dataset processing
- [x] **Efficiency**: Job completed within reasonable time
- [x] **Storage**: Output properly organized in datastore

## 8. AUTOMATION SCRIPTS

### 8.1 Complete Job Submission Script

```bash
#!/bin/bash
# submit-etl-job.sh

# Configuration
SUBSCRIPTION_ID="your-subscription-id"
RESOURCE_GROUP="retail-dev-rg"
WORKSPACE_NAME="retail-ml-workspace"
JOB_NAME="etl_aggregate"

echo "🚀 Submitting ETL Aggregate Job..."

# Login to Azure
az login

# Set subscription
az account set --subscription $SUBSCRIPTION_ID

# Submit job
az ml job create \
  --file etl_aggregate_job.yml \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $WORKSPACE_NAME

# Monitor job
echo "📊 Monitoring job execution..."
az ml job show --name $JOB_NAME --resource-group $RESOURCE_GROUP --workspace-name $WORKSPACE_NAME

echo "✅ Job submitted successfully!"
```

### 8.2 Python Job Management

```python
# manage_etl_jobs.py
from azure.ai.ml import MLClient, command
from azure.ai.ml.entities import Data
from azure.identity import DefaultAzureCredential

def submit_etl_job(key_type='product_id'):
    credential = DefaultAzureCredential()
    ml_client = MLClient(
        credential=credential,
        subscription_id="your-subscription-id",
        resource_group_name="retail-dev-rg",
        workspace_name="retail-ml-workspace"
    )
    
    # Create command job
    job = command(
        code="./src",
        command="python etl_aggregate.py --input_data ${{inputs.retail_data}} --key_type ${{inputs.key_type}} --output_path ${{outputs.aggregates}}",
        inputs={
            "retail_data": Data(type="uri_folder", path="azureml://datastores/workspaceblobstore/paths/raw/ecommerce/"),
            "key_type": key_type
        },
        outputs={
            "aggregates": {"type": "uri_folder"}
        },
        environment="azureml://registries/azureml/environments/retail-train-env/labels/latest",
        compute="cpu-cluster",
        display_name=f"ETL Aggregate - {key_type}"
    )
    
    # Submit job
    submitted_job = ml_client.jobs.create_or_update(job)
    
    print(f"✅ Job submitted: {submitted_job.name}")
    return submitted_job

def submit_all_key_types():
    """Submit ETL jobs for all key types"""
    key_types = ['product_id', 'category_code', 'brand']
    
    for key_type in key_types:
        job = submit_etl_job(key_type)
        print(f"Submitted job for {key_type}: {job.name}")

if __name__ == "__main__":
    submit_all_key_types()
```

## 9. LƯU Ý QUAN TRỌNG

### 9.1 ETL Best Practices
<div style="background: #fffbeb; border: 1px solid #fed7aa; border-left: 4px solid #f59e0b; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>⚠️ Important Considerations</strong>
<ul>
<li><strong>Data Quality</strong>: Always perform quality checks and log statistics</li>
<li><strong>Partitioning</strong>: Use proper partitioning for optimal I/O performance</li>
<li><strong>Error Handling</strong>: Implement robust error handling and logging</li>
<li><strong>Scalability</strong>: Use Dask for large datasets, pandas for smaller ones</li>
<li><strong>Monitoring</strong>: Monitor job execution and resource usage</li>
</ul>
</div>

### 9.2 Troubleshooting Common Issues

**Issue 1: "Job failed with memory error"**
```bash
# Increase compute instance size
# Use Dask for large datasets
# Implement data chunking
```

**Issue 2: "Data Asset not found"**
```bash
# Verify data asset exists
az ml data show --name retail_data --version 1

# Check workspace context
az ml workspace show --name retail-ml-workspace --resource-group retail-dev-rg
```

**Issue 3: "Output path not accessible"**
```bash
# Check compute permissions
# Verify datastore configuration
# Ensure output folder exists
```

### 9.3 Next Steps
1. ✅ **ETL Pipeline completed** ← Current step
2. 🔄 **Feature selection and engineering** ← Next step
3. 🔄 **Model training configuration**
4. 🔄 **Hyperparameter tuning**
5. 🔄 **Model evaluation and selection**

{{% notice info %}}
**Best Practice**: Always validate ETL output data quality before proceeding to model training. Use the quality checks and statistics to ensure data integrity.
{{% /notice %}}

{{% notice warning %}}
**Performance Note**: For large datasets, consider using Dask or Spark for distributed processing. Monitor memory usage and adjust compute resources accordingly.
{{% /notice %}}

## 10. USEFUL LINKS

- **Azure ML Jobs**: https://docs.microsoft.com/en-us/azure/machine-learning/concept-jobs
- **ETL Best Practices**: https://docs.microsoft.com/en-us/azure/architecture/data-guide/relational-data/etl
- **Parquet Format**: https://parquet.apache.org/
- **Dask Documentation**: https://docs.dask.org/

ETL Aggregate pipeline hoàn tất! 🎉 Time series dataset đã sẵn sàng cho **Task 6: Model Training**.
