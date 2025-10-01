# Azure MLOps Retail Forecast Workshop
> **End-to-End MLOps on Azure (Azure ML + ACR + Managed Online Endpoints/AKS + CI/CD + Monitoring)**

![Azure MLOps Architecture](Azure-MLOps-Architecture.png)

---

## Table of Contents
- [Overview](#overview)
- [Architecture Overview](#architecture-overview)
- [Roles & Difficulty](#roles--difficulty)
- [Project Structure](#project-structure)
- [Dataset & Problem Definition](#dataset--problem-definition)
- [MLOps Flow](#mlops-flow)
- [Reference Architecture](#reference-architecture)
- [In Scope / Out of Scope](#in-scope--out-of-scope)
- [Core Tasks (1–19)](#core-tasks-1–19)
- [Technology Stack](#technology-stack)
- [Prerequisites](#prerequisites)
- [Naming & Tagging Standards](#naming--tagging-standards)
- [Acceptance Criteria](#acceptance-criteria)
- [Monitoring & Security](#monitoring--security)
- [CI/CD & DataOps](#cicd--dataops)
- [Cost Optimization](#cost-optimization)
- [FAQ](#faq)

---

## Overview
This workshop delivers a **production-ready MLOps pipeline on Azure** for retail demand forecasting. It mirrors the AWS version but uses **Azure-native services**:

- **ML Platform**: **Azure Machine Learning (AML)** for training, model registry, and managed online endpoints  
- **Container Registry**: **Azure Container Registry (ACR)** for Docker images  
- **Data Lake**: **Azure Storage (Blob / ADLS Gen2)** for datasets and artifacts  
- **CI/CD**: **Azure DevOps Pipelines** (alternatively GitHub Actions)  
- **Observability**: **Azure Monitor + Application Insights + Log Analytics**  
- **Security**: **Azure Key Vault**, **Managed Identities**, **RBAC**, **Activity Log**  
- **Alternative Runtime**: **AKS** path when you need VNet-ingress/sidecars/custom networking

**Goal**: fully automate the lifecycle from **data ingestion → ETL → training → model registration → deployment → monitoring → scheduled retraining**.

---

## Architecture Overview

Three phases with automation-first design:

### Phase 1: Foundation & Registry
- **Resource Group, Key Vault, ACR, Storage** created with Azure CLI / IaC  
- **Azure ML Workspace** as control-plane for experiments, models, and endpoints

### Phase 2: Training & Model Management
- **Versioned Data Assets** referencing blob paths  
- **AML Compute Cluster** (auto scale-to-zero) for training jobs  
- **Reproducible Environments** via Conda/Docker images  
- **Model Registry** with metrics/metadata & quality gates

### Phase 3: Deployment & Operations
- **Managed Online Endpoint (default)** for HTTPS inference (blue/green)  
- **App Insights & Azure Monitor** dashboards and alerts  
- **CD pipeline** to automate submit-train-register-deploy with approvals

---

## Roles & Difficulty

| Member | Role | Focus (Difficulty) |
|---|---|---|
| **Nhân** | Cloud Engineer | IaC, Identity/RBAC, Endpoints/Deployment, CI/CD, Alerts, Cost (**Hard**) |
| **Nguyên** | Data Engineer | ETL (clickstream→time-series), Environments, Training, Registry, Retraining (**Hard**) |
| **Đào** | Data Analyst | Architecture diagrams, Data Asset, Smoke/Contract tests, Docs (**Easy/Medium**) |
| **Giang** | Data Analyst | RG UI, Data upload, Workbooks/KQL, Alert follow-up, Cost report (**Easy/Medium**) |

> **Note:** During bootstrapping, all 4 users temporarily have **Contributor** at RG scope for speed. Reduce privileges later.

---

## Project Structure

This site uses Hugo to render workshop docs; code/config lives under `azure/`.

```
azure-mlops-retail-forecast/
├── archetypes/                      # Hugo archetypes
├── config.toml                      # Hugo site config (theme params, menus)
├── content/                         # Workshop markdown content
│   ├── _index.md                    # Landing page
│   ├── 1-introduction/              # Architecture & objectives (Task 1 kept as-is)
│   ├── 2-resource-group/            # Identity + RG (Task 2)
│   ├── 3-data-upload/               # Upload ≥4GB clickstream (Task 3)
│   ├── 4-data-asset/                # AML Data Asset (Task 4)
│   ├── 5-etl-aggregate/             # ETL to daily time-series (Task 5)
│   ├── 6-workbooks/                 # App Insights/KQL (Task 6)
│   ├── 7-infra/                     # IaC: TF/Bicep (Task 7)
│   ├── 8-environment/               # Conda/Docker env (Task 8)
│   ├── 9-train-job/                 # Training job (Task 9)
│   ├── 10-model-registry/           # Registry + gates (Task 10)
│   ├── 11-ci-image/                 # CI build & push image (Task 11)
│   ├── 12-endpoint/                 # Managed online endpoint (Task 12)
│   ├── 13-deploy-blue/              # Blue 100% (Task 13)
│   ├── 14-smoke-contract/           # API tests (Task 14)
│   ├── 15-alerts/                   # Azure Monitor alerts (Task 15)
│   ├── 16-cicd/                     # Multi-stage pipelines (Task 16)
│   ├── 17-canary/                   # Green + traffic split (Task 17)
│   ├── 18-retraining/               # Scheduled retraining (Task 18)
│   └── 19-cost/                     # Cost & teardown (Task 19)
├── layouts/                         # Theme overrides (partials/shortcodes)
├── static/                          # Assets (images/css/fonts)
│   └── images/
├── themes/
│   └── hugo-theme-learn/
├── public/                          # Hugo build output
└── README.md                        # This document
```

---

## Dataset & Problem Definition

- **Dataset:** *E‑Commerce Behavior Data (2019)* — clickstream events `view`, `cart`, `remove_from_cart`, `purchase`.  
- **Volume:** aggregate multiple months ⇒ **≥4GB** (compressed `.csv.gz`).  
- **Store path:** `data/raw/ecommerce/2019-*.csv.gz` (Azure Storage container `data`).  
- **Forecast target:** **Daily purchases** per **key** (choose one): `product_id` | `category_code` | `brand`.  
- **Features:** `views_t`, `carts_t`, `price_mean_t`, moving averages, lags, `day_of_week`, special-day flags.  
- **Horizon:** 7 / 14 / 28 days.  
- **Metrics:** **MAPE/SMAPE** (primary), **RMSE/RMSPE** (secondary).

---

## MLOps Flow

### 1) Provision & Bootstrap
```bash
# Subscription + RG
az account set --subscription "<SUBSCRIPTION_ID>"
az group create -n rg-mlops-retail-dev -l southeastasia

# Key Vault (RBAC-enabled)
az keyvault create -n kv-retail-dev -g rg-mlops-retail-dev -l southeastasia --enable-rbac-authorization true

# ACR
az acr create -n acrretaildev -g rg-mlops-retail-dev --sku Standard --admin-enabled false

# Storage (Blob/ADLS Gen2)
az storage account create -n stretaildev -g rg-mlops-retail-dev -l southeastasia --sku Standard_LRS --kind StorageV2
az storage container create -n data --account-name stretaildev
# Upload large clickstream files (≥4GB total)
az storage blob upload-batch --account-name stretaildev -d data --destination-path raw/ecommerce/ -s ./clickstream/
```

### 2) Train & Register
```bash
# Azure ML workspace
az extension add -n ml -y
az ml workspace create -n amlws-retail-dev -g rg-mlops-retail-dev

# Data Asset (versioned)
az ml data create --name retail_data --version 1 \
  --path azureml://datastores/workspaceblobstore/paths/data/raw/ecommerce/ \
  -g rg-mlops-retail-dev -w amlws-retail-dev

# Compute cluster (scale-to-zero)
az ml compute create --name cpu-cluster --type amlcompute \
  --size Standard_DS3_v2 --min-instances 0 --max-instances 4 \
  -g rg-mlops-retail-dev -w amlws-retail-dev

# Environment (from conda.yml)
az ml environment create --name retail-train-env --version 1 \
  --file aml/environments/conda.yml -g rg-mlops-retail-dev -w amlws-retail-dev

# ETL Job (clickstream → daily aggregates)
az ml job create -f aml/jobs/etl-aggregate.yml -g rg-mlops-retail-dev -w amlws-retail-dev

# Training Job
az ml job create -f aml/jobs/train-job.yml -g rg-mlops-retail-dev -w amlws-retail-dev

# Register model (after gate, e.g., MAPE <= 20%)
az ml model create --name retail-forecast --path ./outputs/model.bin \
  --type custom_model -g rg-mlops-retail-dev -w amlws-retail-dev
```

### 3) Containerize & Deploy
```bash
# CI: build & push inference image
docker build -f aml/environments/infer.Dockerfile \
  -t acrretaildev.azurecr.io/retail/infer:$BUILD_ID .
az acr login --name acrretaildev
docker push acrretaildev.azurecr.io/retail/infer:$BUILD_ID

# Endpoint + 'blue' deployment (all traffic)
az ml online-endpoint create -f aml/deployments/endpoint.yml \
  -g rg-mlops-retail-dev -w amlws-retail-dev || true

az ml online-deployment create -f aml/deployments/blue.yml \
  -g rg-mlops-retail-dev -w amlws-retail-dev --all-traffic
```

### 4) Observe & Alert
- **Application Insights dashboards**: latency p50/p95, error rate, dependencies  
- **Azure Monitor alerts**: e.g., p95 > 600ms, 5xx > 1% over 5 minutes

### 5) CI/CD Automation
- **Stages**: *CI (build & test)* → *CD (etl → train → register → deploy DEV)* → *manual approve → deploy PROD*  
- **Gates**: model quality metric threshold; auto-rollback by swapping traffic to previous deployment

---

## Reference Architecture
```mermaid
graph TB
  subgraph DataOps["Data & Model Management"]
    Blob[Azure Storage (Blob/ADLS)\nData Lake + Versioning]
    AML[Azure Machine Learning\nETL/Training Jobs + Metrics]
    Registry[AML Model Registry\nVersioned Artifacts]
    Blob --> AML
    AML --> Registry
  end

  subgraph Foundation["Foundation Layer"]
    RG[Resource Group\nRBAC + Policy]
    KV[Key Vault\nSecrets/Keys + MI]
    ACR[Azure Container Registry\nImages + Scanning]
    WS[Azure ML Workspace\nExperiments + Endpoints]
  end

  subgraph Deploy["Inference Platform"]
    EP[Managed Online Endpoint\nHTTPS + Blue/Green]
    IMG[Inference Image\nfrom ACR]
    EP <--> IMG
  end

  subgraph Observability["Observability & Security"]
    Insights[Application Insights\nLogs + Metrics + Traces]
    Monitor[Azure Monitor\nAlerts + Workbooks]
    Activity[Activity Log\nAudit Trail]
    EP --> Insights
    AML --> Insights
  end

  subgraph CICD["CI/CD Pipeline"]
    Repo[Git Repo\nCode + Config]
    ADO[Azure DevOps Pipelines\nBuild/Test/Deploy]
    Repo --> ADO
    ADO --> AML
    ADO --> EP
  end

  Registry --> IMG
```

**Alternative:** Swap **Managed Online Endpoint** for **AKS** + Ingress (AGIC/Nginx) when you need VNet isolation/sidecars/advanced routing.

---

## In Scope / Out of Scope

**In scope**
- 19 core tasks (after removing old “README & Onboarding”) covering **RG, KV, ACR, Storage, AML WS, Data Assets, Compute, Env, ETL, Training Job, Model Registry, CI/CD, Endpoint/Deployment, Monitoring, Alerts, Scheduled Retraining**, and **AKS alternative**
- Managed identities, RBAC, and secure-by-default configurations
- End-to-end automation via Azure DevOps

**Out of scope**
- Multi-region active-active (single region `southeastasia`)
- Custom monitoring stacks beyond Azure Monitor/App Insights
- Real-time streaming ingestion (batch focus)
- Multi-tenant isolation patterns

---

## Core Tasks (1–19)

> **Note:** We removed the previous Task “README & Onboarding” and **renumbered** the rest. **Task 1 (Introduction/Architecture)** stays unchanged.

1) **Architecture Design** — HLA, flows, identity (UI tools) — *Đào/giang*  
2) **Identity & Resource Group** — RG + AAD/RBAC (equal rights for 4 users initially) — *Nhân/Giang*  
3) **Data Upload (≥4GB)** — Upload `2019-*.csv.gz` to `data/raw/ecommerce/` — *Giang*  
4) **Data Asset (AML)** — Versioned pointer to blob folder — *Đào*  
5) **ETL Aggregate** — Clickstream → daily time‑series (Parquet partitions) — *Nguyên*  
6) **Workbooks & KQL** — App Insights dashboards + KQL queries — *Giang/Đào*  
7) **Infra (TF/Bicep)** — RG/KV/ACR/Storage/WS/Compute provisioning — *Nhân*  
8) **Training Environment** — Conda/Docker env with pandas/dask + lgbm/xgb/prophet — *Nguyên*  
9) **Training Job** — TimeSeriesSplit, MAPE/SMAPE, log MLflow — *Nguyên*  
10) **Model Registry (Gated)** — Register with metrics/tags & thresholds — *Nguyên/Nhân*  
11) **CI: Build & Push Image** — FastAPI inference container to ACR — *Nhân*  
12) **Managed Online Endpoint** — Create endpoint (Key/AAD) — *Nhân*  
13) **Deployment BLUE (100%)** — Health probes, autoscale — *Nhân/Nguyên*  
14) **Smoke & Contract Tests** — Postman/Studio tests & schema checks — *Đào/Giang*  
15) **Alerts (Azure Monitor)** — p95 latency & 5xx rules + action groups — *Nhân/Giang*  
16) **CI/CD (Multi‑stage)** — CI → ETL → Train → Gate → Deploy — *Nhân/Nguyên*  
17) **Canary GREEN & Split** — 90/10 split + dashboards & rollback — *Nhân/Giang/Nguyên*  
18) **Scheduled Retraining** — Weekly + drift triggers (if any) — *Nguyên/Nhân*  
19) **Cost & Teardown** — Autoscale to zero, lifecycle, budgets — *Nhân/Giang*  

**Quality thresholds:** MAPE overall ≤ **20%**, P90-by-key ≤ **25%**; p95 latency ≤ **600ms (CPU)** or **250ms (GPU)**; 5xx < **1%**.

---

## Technology Stack

### Platform & Infra
- **Azure Machine Learning (AML)**, **Azure Container Registry (ACR)**, **Azure Storage (Blob/ADLS Gen2)**, **Azure Key Vault**  
- **Managed Online Endpoints** *(default)* or **AKS** *(alternative)*

### Monitoring & Security
- **Application Insights**, **Azure Monitor**, **Log Analytics**  
- **RBAC**, **Managed Identities**, **Activity Log**, **Private Link** *(optional)*

### CI/CD
- **Azure DevOps Pipelines** (or GitHub Actions) with multi-stage YAML

---

## Prerequisites
- Azure subscription with permissions for AML, ACR, Storage, Key Vault, Monitor
- Azure CLI + AML extension (`az extension add -n ml`)
- Docker for image builds
- Python 3.8+ for ML code
- Region: `southeastasia` for all resources

---

## Naming & Tagging Standards

**Naming**
- Resource Group: `rg-mlops-retail-{env}`  
- Key Vault: `kv-retail-{env}`  
- ACR: `acrretail{env}` (global-unique)  
- AML Workspace: `amlws-retail-{env}`  
- AML Compute: `cpu-cluster` / `gpu-cluster`

**Tags**
```
Project=MLOpsRetailForecast
Environment=dev|staging|prod
Component=infra|ml|app
Owner=DataTeam
CostCenter=ML-Platform
```

---

## Acceptance Criteria
- **Provisioning**: All baseline resources created via CLI/IaC with RBAC  
- **ETL/Training**: AML jobs complete; metrics tracked in MLflow  
- **Model**: Registered with thresholds and metadata (versioned)  
- **Deployment**: Endpoint healthy; autoscale configured; p95 latency within SLO  
- **Security**: Data encrypted at rest and in transit; MI over static secrets  
- **Monitoring**: Dashboards and actionable alerts in place  
- **CI/CD**: Change → build/test → etl/train → gate → deploy → monitor

---

## Monitoring & Security

**App Insights (KQL examples)**
```kusto
requests
| summarize p50_duration=percentile(duration, 50), p95_duration=percentile(duration, 95) by bin(timestamp, 5m)

requests
| where success == false
| summarize failures=count() by bin(timestamp, 5m), resultCode
```

**Azure Monitor Alerts (examples)**
- P95 latency > **600ms** (CPU) or **250ms** (GPU) over 5 minutes (Severity 2)  
- Failure rate > **1%** over 5 minutes (Severity 2)

---

## CI/CD & DataOps

**azure-pipelines.yml (snippets)**
```yaml
# CI
- stage: CI
  jobs:
  - job: build_test
    steps:
    - task: UsePythonVersion@0
      inputs: { versionSpec: '3.10' }
    - script: pip install -r requirements.txt && pytest -q
    - script: |
        docker build -f aml/environments/infer.Dockerfile -t acrretaildev.azurecr.io/retail/infer:$(Build.BuildId) .
        az acr login --name acrretaildev
        docker push acrretaildev.azurecr.io/retail/infer:$(Build.BuildId)

# CD (ETL + train + register + deploy DEV)
- stage: CD_Train_Register
  dependsOn: CI
  jobs:
  - job: aml_jobs
    steps:
    - task: AzureCLI@2
      inputs:
        azureSubscription: 'SERVICE_CONNECTION_NAME'
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          az extension add -n ml -y
          az ml job create -f aml/jobs/etl-aggregate.yml -g $(RG) -w $(WS)
          az ml job create -f aml/jobs/train-job.yml -g $(RG) -w $(WS)
          # TODO: parse metrics.json, apply gate, and register the model if passed

- stage: Deploy_DEV
  dependsOn: CD_Train_Register
  jobs:
  - job: deploy
    steps:
    - task: AzureCLI@2
      inputs:
        azureSubscription: 'SERVICE_CONNECTION_NAME'
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          az extension add -n ml -y
          az ml online-endpoint create -f aml/deployments/endpoint.yml -g $(RG) -w $(WS) || true
          az ml online-deployment create -f aml/deployments/blue.yml -g $(RG) -w $(WS) --all-traffic
```

---

## Cost Optimization
- **Scale-to-zero** AML compute clusters in dev/test  
- **Right-size** deployment instances; enable autoscaling  
- **Lifecycle policies** for blob tiers (Hot → Cool → Archive)  
- **Use MI** instead of secrets; centralize secrets in Key Vault  
- **Schedule retraining** during off-peak; use low-priority compute where applicable

---

## FAQ
**Why Managed Online Endpoints instead of AKS by default?**  
Managed Online Endpoints provide turnkey HTTPS, scaling, auth, logging and blue/green — ideal for most teams. Use AKS when you require advanced networking/sidecars.

**Can I use GitHub Actions instead of Azure DevOps?**  
Yes. The YAML concepts are similar: stages, jobs, and gates before deploy.

**How do we handle model drift?**  
Track metrics in App Insights/MLflow. Trigger scheduled retraining; use gates to block regressions and auto-rollback via traffic split.

**What about batch inference?**  
Use **AML Batch Endpoints** or scheduled AML jobs that write results back to Blob.

---

> © Azure MLOps Retail Forecast Workshop — Production-ready ML pipeline on Azure.
