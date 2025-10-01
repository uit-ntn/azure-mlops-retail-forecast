---
title: "Workbooks & KQL Monitoring Dashboard"
date: 2025-08-30T11:50:00+07:00
weight: 6
chapter: false
pre: "<b>6. </b>"
---

Trong phần này, chúng ta sẽ tạo dashboard real-time monitoring để theo dõi toàn bộ MLOps pipeline từ data ingestion, ETL processing, đến inference endpoints. Dashboard sẽ cung cấp insights về business funnel, data quality, drift detection, và performance metrics. Đây là bước quan trọng để đảm bảo reliability và performance của hệ thống production.

## 1. Mục tiêu

✅ **Tạo dashboard realtime theo dõi business funnel và data quality**  
✅ **Monitor data drift và input errors**  
✅ **Track latency và error rate của ETL + inference**  
✅ **Phát hiện sớm bất thường cho DevOps/DataOps**

<div style="background: linear-gradient(135deg, #e6f3ff 0%, #dbeafe 100%); border: 1px solid #bfdbfe; border-left: 6px solid #0078d4; padding: 18px; margin: 18px 0; border-radius: 10px;">
  <strong>TASK 6 — WORKBOOK & KQL NỀN</strong><br>
  <strong>Mục tiêu</strong>: Tạo dashboard realtime theo dõi: Phân bố view/cart/purchase (business funnel). Data drift & input errors (dữ liệu clickstream & inference). Latency & error rate (ETL + inference endpoint). Giúp DevOps/DataOps phát hiện sớm bất thường.
</div>

## 2. MONITORING ARCHITECTURE

### 2.1 Monitoring Components

<div style="background: #f8fafc; border: 1px solid #e2e8f0; border-left: 4px solid #0078d4; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>🔧 Monitoring Stack</strong>

**1. Data Sources:**
- **Application Insights**: ETL jobs, inference endpoints
- **Azure Monitor**: Infrastructure metrics
- **Log Analytics**: Centralized logging
- **Custom Events**: Business metrics tracking

**2. Dashboard Components:**
- **Business Funnel**: View → Cart → Purchase conversion
- **Data Quality**: Input validation, schema compliance
- **Drift Detection**: Feature distribution comparison
- **Performance**: Latency, error rates, throughput

**3. Alerting:**
- **Threshold-based**: Performance degradation
- **Anomaly detection**: Unusual patterns
- **Business KPIs**: Conversion rate drops

</div>

### 2.2 Dashboard Design Strategy

<div style="background: #fff7ed; border: 1px solid #fed7aa; border-left: 4px solid #f59e0b; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>📊 Dashboard Tabs</strong>

**Tab 1: Business Funnel**
- Line charts: Views, carts, purchases theo thời gian
- Stacked bar charts: Conversion rates
- Funnel visualization: View → Cart → Purchase
- Key metrics: Conversion rates, trends

**Tab 2: Data Quality**
- Error tables: Input validation failures
- Schema compliance: Missing fields, type mismatches
- Data freshness: Last update timestamps
- Quality scores: Overall data health

**Tab 3: Drift Detection**
- Feature distributions: Price, category, brand
- Baseline comparison: Historical vs current
- Statistical tests: Kolmogorov-Smirnov, Chi-square
- Alert thresholds: Significant drift detection

**Tab 4: Latency & Errors**
- Latency distribution: P50, P95, P99 percentiles
- Error rates: 4xx/5xx HTTP codes
- Throughput metrics: Requests per second
- Performance trends: Over time analysis

</div>

## 3. AZURE APPLICATION INSIGHTS SETUP

### 3.1 Prerequisites

**Required Components:**
- ✅ **Application Insights** đã tạo (từ Task 7 - Azure ML Workspace Setup)
- ✅ **ETL Jobs** đang chạy và log events (từ **Task 5: ETL Aggregate**)
- ✅ **Inference Endpoints** deployed (từ Task 8)
- ✅ **Custom telemetry** configured trong applications
- ✅ **Log Analytics Workspace** connected

### 3.2 UI Flow (Application Insights)

**Step 1: Navigate to Workbooks**
```
Azure Portal → Application Insights → Workbooks → +New
```

**Step 2: Workbook Configuration**
- **Workbook name**: `Retail-Forecast-Monitoring`
- **Description**: `Real-time monitoring dashboard for retail forecasting MLOps pipeline`
- **Template**: Start from blank workbook

**Step 3: Add Data Sources**
- **Application Insights**: Primary data source
- **Log Analytics**: Additional metrics
- **Custom queries**: KQL scripts

**Step 4: Configure Tabs**
- Add 4 tabs: Business Funnel, Data Quality, Drift Detection, Latency & Errors
- Set refresh intervals: 1 minute for real-time monitoring
- Configure time ranges: Last 24 hours default

## 4. KQL SCRIPTS IMPLEMENTATION

### 4.1 Event Distribution Analysis

```kql
// event_dist.kql - Event type distribution analysis
customEvents
| where timestamp >= ago(24h)
| where name in ("view_event", "cart_event", "purchase_event")
| extend event_type = case(
    name == "view_event", "view",
    name == "cart_event", "cart", 
    name == "purchase_event", "purchase",
    "unknown"
)
| summarize 
    view_count = countif(event_type == "view"),
    cart_count = countif(event_type == "cart"),
    purchase_count = countif(event_type == "purchase")
    by bin(timestamp, 1h)
| extend 
    cart_rate = cart_count / view_count,
    purchase_rate = purchase_count / view_count,
    funnel_conversion = purchase_count / view_count
| order by timestamp asc
| render timechart with (title="Business Funnel Over Time")
```

### 4.2 Latency Monitoring

```kql
// latency.kql - Inference endpoint latency analysis
requests
| where timestamp >= ago(24h)
| where name contains "retail-forecast" or url contains "/predict"
| where success == true
| summarize 
    p50 = percentile(duration, 50),
    p95 = percentile(duration, 95),
    p99 = percentile(duration, 99),
    avg_latency = avg(duration),
    max_latency = max(duration),
    request_count = count()
    by bin(timestamp, 5m)
| order by timestamp asc
| render timechart with (title="Inference Latency Percentiles")
```

### 4.3 Error Analysis

```kql
// errors.kql - Error tracking and categorization
union
    requests,
    customEvents
| where timestamp >= ago(24h)
| extend error_type = case(
    resultCode >= 400 and resultCode < 500, "4xx_Client_Error",
    resultCode >= 500, "5xx_Server_Error",
    success == false, "Application_Error",
    "Success"
)
| summarize 
    total_requests = count(),
    error_count = countif(error_type != "Success"),
    error_rate = countif(error_type != "Success") * 100.0 / count(),
    client_errors = countif(error_type == "4xx_Client_Error"),
    server_errors = countif(error_type == "5xx_Server_Error"),
    app_errors = countif(error_type == "Application_Error")
    by bin(timestamp, 1h)
| order by timestamp asc
| render timechart with (title="Error Rate Analysis")
```

### 4.4 Data Drift Detection

```kql
// drift.kql - Feature distribution comparison
customEvents
| where timestamp >= ago(24h)
| where name == "data_processed"
| extend 
    price = todouble(customDimensions.price),
    category = tostring(customDimensions.category_code),
    brand = tostring(customDimensions.brand)
| where isnotnull(price) and price > 0
| summarize 
    price_mean = avg(price),
    price_std = stdev(price),
    price_p50 = percentile(price, 50),
    price_p95 = percentile(price, 95),
    category_dist = dcount(category),
    brand_dist = dcount(brand),
    record_count = count()
    by bin(timestamp, 1h)
| extend 
    price_cv = price_std / price_mean,  // Coefficient of variation
    drift_score = case(
        price_cv > 0.5, "High_Drift",
        price_cv > 0.3, "Medium_Drift",
        "Low_Drift"
    )
| order by timestamp asc
| render timechart with (title="Data Drift Indicators")
```

### 4.5 Business Funnel Analysis

```kql
// funnel.kql - Conversion funnel analysis
customEvents
| where timestamp >= ago(24h)
| where name in ("view_event", "cart_event", "purchase_event")
| extend user_id = tostring(customDimensions.user_id)
| where isnotnull(user_id)
| summarize 
    views = countif(name == "view_event"),
    carts = countif(name == "cart_event"),
    purchases = countif(name == "purchase_event")
    by user_id, bin(timestamp, 1h)
| summarize 
    total_users = dcount(user_id),
    users_with_views = countif(views > 0),
    users_with_carts = countif(carts > 0),
    users_with_purchases = countif(purchases > 0)
    by bin(timestamp, 1h)
| extend 
    view_to_cart_rate = users_with_carts * 100.0 / users_with_views,
    cart_to_purchase_rate = users_with_purchases * 100.0 / users_with_carts,
    overall_conversion_rate = users_with_purchases * 100.0 / users_with_views
| order by timestamp asc
| render timechart with (title="Conversion Funnel Rates")
```

## 5. WORKBOOK IMPLEMENTATION

### 5.1 Business Funnel Tab

```json
{
  "version": "Notebook/1.0",
  "items": [
    {
      "type": 1,
      "content": {
        "json": "## Business Funnel Analysis\n\nMonitor conversion rates and user behavior patterns in real-time."
      }
    },
    {
      "type": 3,
      "content": {
        "version": "KqlItem/1.0",
        "query": "customEvents\n| where timestamp >= ago(24h)\n| where name in (\"view_event\", \"cart_event\", \"purchase_event\")\n| extend event_type = case(\n    name == \"view_event\", \"view\",\n    name == \"cart_event\", \"cart\", \n    name == \"purchase_event\", \"purchase\",\n    \"unknown\"\n)\n| summarize \n    view_count = countif(event_type == \"view\"),\n    cart_count = countif(event_type == \"cart\"),\n    purchase_count = countif(event_type == \"purchase\")\n    by bin(timestamp, 1h)\n| extend \n    cart_rate = cart_count / view_count,\n    purchase_rate = purchase_count / view_count\n| render timechart",
        "size": 0,
        "title": "Event Distribution Over Time",
        "queryType": 0,
        "resourceType": "microsoft.insights/components"
      }
    }
  ]
}
```

### 5.2 Data Quality Tab

```json
{
  "type": 3,
  "content": {
    "version": "KqlItem/1.0",
    "query": "customEvents\n| where timestamp >= ago(24h)\n| where name == \"data_validation\"\n| extend \n    validation_error = tostring(customDimensions.error_type),\n    error_count = toint(customDimensions.error_count)\n| summarize \n    total_errors = sum(error_count),\n    error_types = dcount(validation_error)\n    by validation_error, bin(timestamp, 1h)\n| render barchart",
    "size": 0,
    "title": "Data Quality Issues",
    "queryType": 0,
    "resourceType": "microsoft.insights/components"
  }
}
```

### 5.3 Drift Detection Tab

```json
{
  "type": 3,
  "content": {
    "version": "KqlItem/1.0",
    "query": "customEvents\n| where timestamp >= ago(24h)\n| where name == \"feature_distribution\"\n| extend \n    price = todouble(customDimensions.price),\n    category = tostring(customDimensions.category),\n    brand = tostring(customDimensions.brand)\n| summarize \n    price_mean = avg(price),\n    price_std = stdev(price),\n    category_diversity = dcount(category),\n    brand_diversity = dcount(brand)\n    by bin(timestamp, 1h)\n| render timechart",
    "size": 0,
    "title": "Feature Distribution Drift",
    "queryType": 0,
    "resourceType": "microsoft.insights/components"
  }
}
```

### 5.4 Latency & Errors Tab

```json
{
  "type": 3,
  "content": {
    "version": "KqlItem/1.0",
    "query": "requests\n| where timestamp >= ago(24h)\n| where name contains \"retail-forecast\"\n| summarize \n    p50 = percentile(duration, 50),\n    p95 = percentile(duration, 95),\n    p99 = percentile(duration, 99),\n    error_rate = countif(success == false) * 100.0 / count()\n    by bin(timestamp, 5m)\n| render timechart",
    "size": 0,
    "title": "Latency and Error Rates",
    "queryType": 0,
    "resourceType": "microsoft.insights/components"
  }
}
```

## 6. CUSTOM TELEMETRY IMPLEMENTATION

### 6.1 ETL Job Telemetry

```python
# etl_telemetry.py - Add to ETL pipeline
from opencensus.ext.azure.log_exporter import AzureLogHandler
from opencensus.ext.azure.trace_exporter import AzureExporter
from opencensus.trace import config_integration
from opencensus.trace.samplers import ProbabilitySampler
from opencensus.trace.tracer import Tracer
import logging

def setup_telemetry():
    """Setup Application Insights telemetry"""
    # Configure logging
    logger = logging.getLogger(__name__)
    logger.addHandler(AzureLogHandler(connection_string="your-connection-string"))
    
    # Configure tracing
    config_integration.trace_integrations(['requests'])
    tracer = Tracer(
        exporter=AzureExporter(connection_string="your-connection-string"),
        sampler=ProbabilitySampler(rate=1.0)
    )
    
    return logger, tracer

def log_business_event(event_type, properties):
    """Log business events for funnel analysis"""
    logger, tracer = setup_telemetry()
    
    with tracer.span(name=f"{event_type}_event"):
        logger.info(f"Business event: {event_type}", extra={
            "custom_dimensions": {
                "event_type": event_type,
                "user_id": properties.get("user_id"),
                "product_id": properties.get("product_id"),
                "category_code": properties.get("category_code"),
                "brand": properties.get("brand"),
                "price": properties.get("price")
            }
        })

def log_data_quality_metrics(metrics):
    """Log data quality metrics"""
    logger, _ = setup_telemetry()
    
    logger.info("Data quality metrics", extra={
        "custom_dimensions": {
            "total_records": metrics["total_records"],
            "invalid_records": metrics["invalid_records"],
            "missing_values": metrics["missing_values"],
            "schema_errors": metrics["schema_errors"],
            "quality_score": metrics["quality_score"]
        }
    })

def log_feature_distribution(features):
    """Log feature distribution for drift detection"""
    logger, _ = setup_telemetry()
    
    logger.info("Feature distribution", extra={
        "custom_dimensions": {
            "price_mean": features["price_mean"],
            "price_std": features["price_std"],
            "category_distribution": features["category_dist"],
            "brand_distribution": features["brand_dist"]
        }
    })
```

### 6.2 Inference Endpoint Telemetry

```python
# inference_telemetry.py - Add to inference service
import time
from functools import wraps

def track_inference_metrics(func):
    """Decorator to track inference metrics"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        start_time = time.time()
        
        try:
            result = func(*args, **kwargs)
            duration = time.time() - start_time
            
            # Log successful inference
            logger.info("Inference completed", extra={
                "custom_dimensions": {
                    "duration_ms": duration * 1000,
                    "success": True,
                    "model_version": kwargs.get("model_version", "unknown"),
                    "input_size": len(str(kwargs.get("input_data", "")))
                }
            })
            
            return result
            
        except Exception as e:
            duration = time.time() - start_time
            
            # Log failed inference
            logger.error("Inference failed", extra={
                "custom_dimensions": {
                    "duration_ms": duration * 1000,
                    "success": False,
                    "error_type": type(e).__name__,
                    "error_message": str(e)
                }
            })
            
            raise
    
    return wrapper

# Usage in FastAPI endpoint
from fastapi import FastAPI
import logging

app = FastAPI()
logger = logging.getLogger(__name__)

@app.post("/predict")
@track_inference_metrics
async def predict(input_data: dict):
    # Your inference logic here
    return {"prediction": "result"}
```

## 7. BẰNG CHỨNG HOÀN THÀNH

### 7.1 Ảnh 1: Workbook Creation
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📋 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Application Insights → Workbooks → +New</li>
<li>✅ Workbook name: <code>Retail-Forecast-Monitoring</code></li>
<li>✅ 4 tabs configured: Business Funnel, Data Quality, Drift Detection, Latency & Errors</li>
<li>✅ Real-time refresh enabled</li>
</ul>
</div>

### 7.2 Ảnh 2: Business Funnel Dashboard
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📊 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Business Funnel tab active</li>
<li>✅ Line charts showing view/cart/purchase trends</li>
<li>✅ Conversion rates visible</li>
<li>✅ Real-time data updating</li>
</ul>
</div>

### 7.3 Ảnh 3: Data Quality Monitoring
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📁 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Data Quality tab showing error tables</li>
<li>✅ Input validation failures displayed</li>
<li>✅ Schema compliance metrics</li>
<li>✅ Quality scores visible</li>
</ul>
</div>

### 7.4 Ảnh 4: Drift Detection Dashboard
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📊 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Drift Detection tab active</li>
<li>✅ Feature distribution comparisons</li>
<li>✅ Baseline vs current distributions</li>
<li>✅ Drift scores and alerts</li>
</ul>
</div>

### 7.5 Ảnh 5: Latency & Error Monitoring
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>⚡ Yêu cầu screenshot:</strong>
<ul>
<li>✅ Latency & Errors tab showing performance metrics</li>
<li>✅ P50, P95, P99 latency percentiles</li>
<li>✅ Error rates (4xx/5xx) displayed</li>
<li>✅ Throughput metrics visible</li>
</ul>
</div>

### 7.6 Ảnh 6: KQL Scripts Repository
<div style="background: #fef2f2; border: 1px solid #fecaca; border-left: 4px solid #ef4444; padding: 15px; margin: 15px 0; border-radius: 8px;">
<strong>📝 Yêu cầu screenshot:</strong>
<ul>
<li>✅ Repository showing KQL files in azure/monitor/kql/</li>
<li>✅ Files: event_dist.kql, latency.kql, errors.kql, drift.kql, funnel.kql</li>
<li>✅ Commit log showing "Task 6 — add KQL scripts"</li>
<li>✅ File contents visible</li>
</ul>
</div>

## 8. TIÊU CHÍ HOÀN THÀNH

### 8.1 Functional Requirements
- [x] **Workbook created**: `Retail-Forecast-Monitoring` in Application Insights
- [x] **Dashboard tabs**: 4 tabs configured with proper visualizations
- [x] **Real-time monitoring**: Dashboard updating every minute
- [x] **KQL scripts**: 5 KQL files committed to repository
- [x] **Telemetry**: Custom events logged from applications

### 8.2 Monitoring Requirements
- [x] **Business funnel**: View/cart/purchase tracking working
- [x] **Data quality**: Error detection and validation metrics
- [x] **Drift detection**: Feature distribution monitoring
- [x] **Performance**: Latency and error rate tracking
- [x] **Alerting**: Thresholds configured for anomaly detection

### 8.3 Documentation Requirements
- [x] **KQL scripts**: Well-documented and version-controlled
- [x] **Workbook JSON**: Exported and stored in repository
- [x] **Telemetry code**: Examples provided for integration
- [x] **Troubleshooting**: Common issues and solutions documented

## 9. AUTOMATION SCRIPTS

### 9.1 Workbook Deployment Script

```bash
#!/bin/bash
# deploy-workbook.sh

# Configuration
SUBSCRIPTION_ID="your-subscription-id"
RESOURCE_GROUP="retail-dev-rg"
APP_INSIGHTS_NAME="retail-ml-insights"
WORKBOOK_FILE="retail-forecast-monitoring.json"

echo "🚀 Deploying Monitoring Workbook..."

# Login to Azure
az login

# Set subscription
az account set --subscription $SUBSCRIPTION_ID

# Get Application Insights resource ID
APP_INSIGHTS_ID=$(az monitor app-insights component show \
  --app retail-ml-insights \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

# Deploy workbook
az monitor workbook create \
  --resource-group $RESOURCE_GROUP \
  --name "Retail-Forecast-Monitoring" \
  --display-name "Retail Forecast Monitoring" \
  --source-id $APP_INSIGHTS_ID \
  --file $WORKBOOK_FILE

echo "✅ Workbook deployed successfully!"
```

### 9.2 KQL Script Validation

```bash
#!/bin/bash
# validate-kql-scripts.sh

KQL_DIR="azure/monitor/kql"
SCRIPTS=("event_dist.kql" "latency.kql" "errors.kql" "drift.kql" "funnel.kql")

echo "🔍 Validating KQL Scripts..."

for script in "${SCRIPTS[@]}"; do
    if [ -f "$KQL_DIR/$script" ]; then
        echo "✅ $script exists"
        
        # Basic syntax validation
        if grep -q "render" "$KQL_DIR/$script"; then
            echo "✅ $script contains render statement"
        else
            echo "❌ $script missing render statement"
        fi
        
        if grep -q "where timestamp" "$KQL_DIR/$script"; then
            echo "✅ $script contains time filtering"
        else
            echo "❌ $script missing time filtering"
        fi
        
    else
        echo "❌ $script not found"
    fi
done

echo "📊 KQL Script validation completed!"
```

### 9.3 Telemetry Setup Script

```python
# setup_telemetry.py
from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry import trace
from opentelemetry.exporter.azure_monitor import AzureMonitorTraceExporter
import logging

def configure_monitoring():
    """Configure Azure Monitor telemetry"""
    
    # Configure Azure Monitor
    configure_azure_monitor(
        connection_string="your-connection-string",
        enable_live_metrics=True
    )
    
    # Setup logging
    logging.basicConfig(level=logging.INFO)
    logger = logging.getLogger(__name__)
    
    # Setup tracing
    tracer = trace.get_tracer(__name__)
    
    return logger, tracer

def create_custom_events():
    """Create custom events for monitoring"""
    logger, tracer = configure_monitoring()
    
    # Business events
    with tracer.start_as_current_span("business_event"):
        logger.info("Business event logged", extra={
            "custom_dimensions": {
                "event_type": "view",
                "user_id": "user123",
                "product_id": "prod456"
            }
        })
    
    # Data quality events
    with tracer.start_as_current_span("data_quality"):
        logger.info("Data quality check", extra={
            "custom_dimensions": {
                "quality_score": 0.95,
                "invalid_records": 5,
                "total_records": 1000
            }
        })
    
    print("✅ Telemetry configured successfully!")

if __name__ == "__main__":
    create_custom_events()
```

## 10. LƯU Ý QUAN TRỌNG

### 10.1 Monitoring Best Practices
<div style="background: #fffbeb; border: 1px solid #fed7aa; border-left: 4px solid #f59e0b; padding: 20px; margin: 20px 0; border-radius: 8px;">
<strong>⚠️ Important Considerations</strong>
<ul>
<li><strong>Performance Impact</strong>: Minimize telemetry overhead in production</li>
<li><strong>Data Retention</strong>: Configure appropriate retention policies</li>
<li><strong>Alert Fatigue</strong>: Set meaningful thresholds to avoid noise</li>
<li><strong>Cost Management</strong>: Monitor Application Insights usage and costs</li>
<li><strong>Security</strong>: Ensure sensitive data is not logged in custom dimensions</li>
</ul>
</div>

### 10.2 Troubleshooting Common Issues

**Issue 1: "No data in dashboard"**
```bash
# Check Application Insights connection
# Verify custom events are being logged
# Check time range settings
# Validate KQL query syntax
```

**Issue 2: "High latency in queries"**
```bash
# Optimize KQL queries with proper indexing
# Use appropriate time ranges
# Consider data sampling for large datasets
# Check Application Insights performance
```

**Issue 3: "Missing custom events"**
```bash
# Verify telemetry configuration
# Check connection string
# Validate custom event logging code
# Monitor Application Insights ingestion
```

### 10.3 Next Steps
1. ✅ **Monitoring Dashboard completed** ← Current step
2. 🔄 **Alert configuration** ← Next step (Task 15)
3. 🔄 **Automated remediation**
4. 🔄 **Performance optimization**
5. 🔄 **Cost monitoring and optimization**

{{% notice info %}}
**Best Practice**: Regularly review and update monitoring dashboards based on operational insights. Use the KQL scripts as a foundation for custom alerts and automated responses.
{{% /notice %}}

{{% notice warning %}}
**Cost Note**: Application Insights can generate significant costs with high-volume telemetry. Monitor usage and implement sampling strategies for cost optimization.
{{% /notice %}}

## 11. USEFUL LINKS

- **Application Insights Workbooks**: https://docs.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-overview
- **KQL Query Language**: https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/
- **Custom Telemetry**: https://docs.microsoft.com/en-us/azure/azure-monitor/app/api-custom-events-metrics
- **Monitoring Best Practices**: https://docs.microsoft.com/en-us/azure/azure-monitor/insights/ai-insights-overview

Monitoring dashboard hoàn tất! 🎉 Real-time monitoring đã sẵn sàng cho **Task 7: Alert Configuration**.
