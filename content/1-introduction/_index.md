---
title: "Thiết kế kiến trúc MLOps"
date: 2025-08-30T11:00:00+07:00
weight: 1
chapter: false
pre: "<b>1. </b>"
---


**Azure MLOps Retail Forecast** là một hệ thống MLOps end-to-end hoàn chỉnh được xây dựng trên Azure Cloud, tự động hóa toàn bộ quy trình từ provisioning resources, huấn luyện mô hình với Azure ML, triển khai inference endpoints, đến monitoring và cost optimization. Dự án được thiết kế để đảm bảo tính mở rộng, độ tin cậy và bảo mật cao cho các ứng dụng Machine Learning trong thực tế.

## 1. Kiến trúc MLOps trên Azure Cloud

<div style="text-align: center; margin: 30px 0;">
{{< bordered src="/images/01-introduction/Azure-MLOps-Architecture.png" title="Kiến trúc Azure MLOps end-to-end cho Retail Forecasting" >}}
</div>

### 2. Mục tiêu dự án

**Tự động hóa hoàn toàn quy trình Azure MLOps:**
- ☁️ **Azure Foundation**: Resource Groups, Key Vault, RBAC, và Managed Identities
- 🤖 **Azure ML Platform**: Training jobs, Model Registry, và Compute clusters với auto-scaling
- 🚀 **Managed Endpoints**: Serverless inference với blue/green deployment
- 📊 **Monitoring & Security**: Application Insights, Azure Monitor, và Activity Log
- 🔄 **Azure DevOps Pipeline**: Automated CI/CD từ code changes → build → train → deploy
- 💰 **Cost Optimization**: Scale-to-zero compute và lifecycle policies

### 3. Flow tổng quát

```mermaid
graph LR
    A[Resource Group] --> B[Key Vault + ACR]
    B --> C[Azure ML Workspace]
    C --> D[Training Job + Model Registry]
    D --> E[Container Build + Push]
    E --> F[Managed Online Endpoint]
    F --> G[Application Insights]
    G --> H[Azure Monitor + Alerts]
    
    style A fill:#0078d4
    style C fill:#10b981
    style F fill:#8b5cf6
    style H fill:#ef4444
```

**Foundation → ML Training → Deployment → Monitoring → CI/CD → Advanced Features**

## 4. 📁 Project Structure

Dự án được tổ chức theo cấu trúc modularity với separation of concerns rõ ràng, hỗ trợ cả Azure và AWS MLOps:

```
azure-mlops-retail-forecast/
├── README.md                         # 📖 Project overview & setup guide
├── core/                             # 🔧 Dependencies chung cho ML
│   └── requirements.txt              # numpy, pandas, scikit-learn, mlflow, fastapi, pytest
│
├── server/                           # 🌐 Backend inference API (FastAPI)
│   ├── Dockerfile                    # Container definition cho inference API
│   ├── README.md                     # Hướng dẫn setup server
│   ├── requirements.txt              # Dependencies cho API server
│   └── app/                          # FastAPI application code
│
├── azure/                            # 🔵 Azure MLOps Configuration
│   ├── aml/                          # Azure Machine Learning
│   │   ├── train.Dockerfile          # Training container image
│   │   ├── infer.Dockerfile          # Inference container image
│   │   ├── train-job.yml             # AML training job definition
│   │   ├── deployment/               # Online endpoint configs
│   │   │   ├── endpoint.yml          # Managed online endpoint
│   │   │   └── blue.yml              # Blue deployment config
│   │   ├── environments/             # ML environments
│   │   │   └── conda.yml             # Conda environment
│   │   └── README.md                 # 📖 Azure ML documentation
│   │
│   ├── infra/                        # 🏗️ Infrastructure as Code
│   │   ├── main.tf                   # Terraform configuration
│   │   ├── online_endpoint.tf        # Azure ML endpoint (AzAPI)
│   │   ├── alerts.tf                 # Application Insights alerts
│   │   ├── main.bicep                # Bicep alternative
│   │   ├── dev.tfvars                # Development variables
│   │   ├── prod.tfvars               # Production variables
│   │   ├── parameters/               # Bicep parameter files
│   │   └── README.md                 # 📖 Infrastructure documentation
│   │
│   ├── k8s/                          # ☸️ Kubernetes manifests cho AKS
│   │   ├── deployment.yaml           # Pod deployment
│   │   ├── service.yaml              # Service exposure
│   │   ├── hpa.yaml                  # Horizontal Pod Autoscaler
│   │   └── README.md                 # 📖 Kubernetes documentation
│   │
│   ├── monitor/                      # 📊 Monitoring & Observability
│   │   ├── alerts.bicep              # Application Insights alerts
│   │   ├── kql/                      # KQL queries
│   │   │   ├── error.kql             # Error tracking queries
│   │   │   └── latency.kql           # Latency monitoring queries
│   │   └── README.md                 # 📖 Monitoring documentation
│   │
│   ├── azure-pipelines.yml           # Azure DevOps CI/CD
│   └── README.md                     # 📖 Azure setup overview
│
├── tests/                            # 🧪 Test directory
└── static/images/
    └── Azure-MLOps-Architecture.png
```

### 4.1 Cấu trúc thư mục chi tiết & Ý nghĩa

#### 🔧 **Core Components**
**📂 `core/` - Shared ML Dependencies**
- **Mục đích**: Centralized dependencies cho toàn bộ ML pipeline
- **Nội dung**: Common libraries như numpy, pandas, scikit-learn, mlflow
- **Lợi ích**: Đảm bảo version consistency across environments

#### 🌐 **Inference Layer**
**📂 `server/` - FastAPI Inference Service**
- **Mục đích**: Production-ready REST API cho model inference
- **Dockerfile**: Container definition với optimized Python runtime
- **app/**: FastAPI application với health checks và monitoring
- **Lợi ích**: Scalable, async inference với automatic documentation

#### 🔵 **Azure MLOps Platform**

**📂 `azure/aml/` - Azure Machine Learning**
- **train.Dockerfile**: Custom training environment với ML frameworks
- **infer.Dockerfile**: Lightweight inference container
- **train-job.yml**: Command job specification với compute targets
- **deployment/**: Managed online endpoint configs với blue/green deployment
- **environments/**: Reproducible conda environments
- **Ý nghĩa**: Serverless ML training và deployment với auto-scaling

**📂 `azure/infra/` - Infrastructure as Code**
- **main.tf**: Terraform root module cho Azure resources
- **main.bicep**: Bicep alternative với native Azure support
- **online_endpoint.tf**: AzAPI provider cho ML endpoints
- **alerts.tf**: Application Insights monitoring rules
- **tfvars files**: Environment-specific configurations
- **Ý nghĩa**: Reproducible, version-controlled infrastructure

**📂 `azure/k8s/` - Container Orchestration**
- **deployment.yaml**: Pod specifications với resource limits
- **service.yaml**: Load balancer và service discovery
- **hpa.yaml**: Horizontal Pod Autoscaler cho dynamic scaling
- **Ý nghĩa**: Advanced deployment patterns khi cần custom networking

**📂 `azure/monitor/` - Observability Stack**
- **alerts.bicep**: Infrastructure-level monitoring rules
- **kql/**: Kusto Query Language cho log analysis
- **error.kql**: Error tracking và alerting queries
- **latency.kql**: Performance monitoring queries
- **Ý nghĩa**: Comprehensive monitoring với custom dashboards

#### 🚀 **DevOps Integration**
**📂 `azure-pipelines.yml` - CI/CD Automation**
- **Multi-stage pipeline**: Build → Test → Train → Deploy
- **Automated testing**: Unit tests, integration tests, model validation
- **Blue/green deployment**: Zero-downtime production updates
- **Ý nghĩa**: End-to-end automation từ code commit đến production

#### 🧪 **Quality Assurance**
**📂 `tests/` - Testing Framework**
- **Unit tests**: ML pipeline components testing
- **Integration tests**: Infrastructure và service connectivity
- **End-to-end tests**: Complete workflow validation
- **Ý nghĩa**: Đảm bảo reliability và regression prevention

### 5 Architecture Benefits

✅ **Modularity**: Clear separation of concerns  
✅ **Scalability**: Auto-scaling compute và endpoints  
✅ **Reproducibility**: IaC và containerized environments  
✅ **Observability**: Comprehensive monitoring stack  
✅ **Security**: RBAC, Key Vault, private endpoints  
✅ **Cost Optimization**: Scale-to-zero và lifecycle policies

## 6. Công nghệ sử dụng

### 6.1 Azure Foundation & Security Stack
- **Resource Management**: Resource Groups với RBAC và tagging
- **Secret Management**: Azure Key Vault với Managed Identities
- **Container Registry**: Azure Container Registry với security scanning
- **Identity & Access**: Azure Active Directory integration
- **Network Security**: Private endpoints và network isolation

### 6.2 Azure ML & Data Platform Stack  
- **ML Training**: Azure Machine Learning với distributed compute
- **Data Storage**: Azure Storage (Blob/ADLS Gen2) với versioning
- **Model Registry**: Azure ML Model Registry với metadata tracking
- **Compute**: Auto-scaling compute clusters với scale-to-zero
- **ML Framework**: Support cho TensorFlow, PyTorch, scikit-learn

### 6.3 DevOps & Monitoring Stack
- **CI/CD Platform**: Azure DevOps Pipelines với multi-stage workflows
- **Monitoring**: Application Insights và Azure Monitor dashboards
- **Observability**: Log Analytics với KQL queries và custom metrics
- **Deployment**: Managed Online Endpoints với blue/green deployments

## 7. Kiến trúc MLOps chi tiết

### 7.1 Phase 1: Azure Foundation

**Resource Provisioning với Azure CLI/Bicep**
- Resource Group với comprehensive tagging strategy
- Key Vault với RBAC-enabled access policies
- Azure Container Registry với admin-disabled security
- Azure Storage với hierarchical namespace (ADLS Gen2)
- Managed Identities cho secure service-to-service communication

**Security Architecture**
- Private endpoints cho secure network communication
- Key Vault integration cho secrets và certificates
- RBAC assignments với least privilege principles
- Network Security Groups với application-specific rules

### 7.2 Phase 2: Azure ML Training & Model Management

**Azure ML Training Pipeline**
- **Data Assets**: Versioned data references với automated validation
- **Compute Clusters**: Auto-scaling compute với scale-to-zero capabilities
- **Training Jobs**: Command jobs với reproducible environments
- **Model Registry**: Versioned model artifacts với performance metrics

**Data Management Strategy**
- Raw data (Blob) → Processed data → Data Assets → Model artifacts
- Intelligent tiering cho cost optimization (Hot → Cool → Archive)
- MLflow integration cho experiment tracking và model lineage

### 7.3 Phase 3: Managed Inference Platform

**Azure ML Managed Endpoints**
- **Managed Online Endpoints**: Serverless HTTPS inference với authentication
- **Blue/Green Deployments**: Zero-downtime deployments với traffic splitting
- **Auto-scaling**: Dynamic scaling dựa trên request load
- **Container Integration**: Custom Docker images từ ACR
- **Alternative AKS**: Advanced networking và custom sidecars khi cần

**Monitoring & Observability**
- **Application Insights**: Distributed tracing và performance monitoring
- **Azure Monitor**: Custom metrics, alerts, và workbooks
- **Log Analytics**: Centralized logging với KQL queries
- **Dashboards**: Real-time visualization với Azure Monitor workbooks

### 7.5 Phase 4: CI/CD & Automation

**Automated Pipeline Flow**
```bash
 Code/Data Change → Git Webhook
2. Azure DevOps Build → Run Tests
3. Azure ML Training → Model Validation  
4. Docker Build → Push to ACR
5. Managed Endpoint Deploy → Blue/Green Update
6. Health Check → Monitor Performance
```

**DataOps Workflow**
- **Data Versioning**: Azure Storage với metadata tracking
- **Data Quality**: Automated validation và testing
- **Feature Engineering**: Reproducible pipelines
- **Model Deployment**: Blue/green deployment capabilities

## 8. Scope & Expected Outcomes

### In Scope
✅ **Complete Infrastructure**: Bicep/CLI cho toàn bộ Azure resources  
✅ **ML Training**: Azure ML distributed training với hyperparameter tuning  
✅ **Container Deployment**: Managed Endpoints với autoscaling và load balancing  
✅ **Security Best Practices**: Key Vault encryption, Activity Log audit, RBAC least privilege  
✅ **Monitoring & Alerting**: Application Insights comprehensive monitoring  
✅ **CI/CD Automation**: End-to-end pipeline từ code đến production  
✅ **Cost Optimization**: Scale-to-zero compute, lifecycle policies  

### Out of Scope
❌ Multi-region deployment (focus on Southeast Asia)  
❌ Advanced ML features (A/B testing, canary deployments)  
❌ Real-time streaming inference (batch-focused)  
❌ Custom monitoring solutions (Azure Monitor-only)  

### Expected Outcomes
🎯 **Production-Ready MLOps Platform**: Scalable, reliable, cost-effective  
🎯 **Automated ML Lifecycle**: Từ data ingestion đến model deployment  
🎯 **Infrastructure Reproducibility**: Azure Resource Manager state management  
🎯 **Operational Excellence**: Comprehensive monitoring và alerting  
🎯 **Cost Efficiency**: Optimized resource usage với scale-to-zero  

{{% notice info %}}
Kiến trúc này được thiết kế để support enterprise-grade ML workloads với khả năng scale từ proof-of-concept đến production với hàng triệu requests/day.
{{% /notice %}}

{{% notice warning %}}
**Prerequisites**: Azure Subscription với permissions cho Azure ML, ACR, Storage, Key Vault, Monitor. Azure CLI >= 2.0, Docker, Python 3.8+.
{{% /notice %}}
