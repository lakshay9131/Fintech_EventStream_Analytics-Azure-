# test_project_1 — Secure Analytics Platform on Azure

> **AZ-305 Study Project** | Akshay Moryani | March 2026  
> Hands-on implementation covering AZ-305 Domains 1 & 2

---

## 🎯 Business Scenario

A FinTech food delivery platform needs a secure, scalable analytics system that:
- Ingests 200,000+ daily order events in real time
- Routes data to fast operational storage AND deep analytics storage simultaneously  
- Enforces strict identity governance across four distinct roles with least privilege
- Maintains full audit trails for compliance
- Operates at near-zero cost during development (~₹2-3/month)

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  IDENTITY LAYER                           │
│   Microsoft Entra ID                                      │
│   4 Security Groups → RBAC → ABAC Conditions             │
│   Managed Identity (tp1-app-identity + ASA job)          │
└─────────────────────────┬────────────────────────────────┘
                          │
               ┌──────────▼──────────┐
               │     EVENT HUB        │
               │   tp1-eventhub       │
               │  tp1-orders-hub      │
               │   2 partitions       │
               │  (Kafka-compatible)  │
               └──────────┬──────────┘
                          │
               ┌──────────▼──────────┐
               │  STREAM ANALYTICS   │
               │  tp1-stream-        │
               │  analytics          │
               │  (SQL query engine) │
               └────┬───────────┬────┘
                    │           │
         ┌──────────▼──┐   ┌───▼──────────────┐
         │  COSMOS DB  │   │  DATA LAKE GEN2  │
         │ tp1cosmosdb │   │ testproject1adls  │
         │  Hot Path   │   │   Cold Path      │
         │  Completed  │   │   ALL orders     │
         │  orders only│   │  /orders/YYYY/   │
         └──────┬──────┘   └───────┬──────────┘
                │                  │
                └────────┬─────────┘
                         │  Diagnostic Settings
               ┌──────────▼──────────┐
               │   LOG ANALYTICS     │
               │  tp1-logs-workspace │
               │  Full audit trail   │
               │  KQL queryable      │
               └─────────────────────┘
```

---

## ☁️ Resources Deployed

| Resource | Type | Region | Tier |
|---|---|---|---|
| testproject1adls | Storage Account / Data Lake Gen2 | South India | Standard LRS |
| tp1cosmosdb | Azure Cosmos DB | Central India | Serverless |
| tp1-eventhub | Event Hubs Namespace | South India | Basic |
| tp1-stream-analytics | Stream Analytics Job | South India | 1 SU |
| tp1-logs-workspace | Log Analytics Workspace | South India | Pay-as-you-go |
| tp1-app-identity | Managed Identity | South India | Free |
| test-project-1-kv | Key Vault | South India | Standard |

**Total monthly cost: ~₹2-3 (development/idle)**

---

## 🔐 Identity & Governance Design

All permissions assigned to **groups, never individuals** — AZ-305 best practice.

### Resource Group RBAC

| Group | Role | Scope | Condition |
|---|---|---|---|
| platform-admins | Owner | Resource Group | ABAC: Cannot assign Owner/UAA/RBAC roles |
| Data_Engineers | Contributor | Resource Group | None |
| analytics-viewers | Reader | Resource Group | None |
| compliance-officers | Reader | Resource Group | None |

### Data Plane RBAC (Storage Account)

| Principal | Role | Why |
|---|---|---|
| Data_Engineers | Storage Blob Data Contributor | Write pipeline data |
| analytics-viewers | Storage Blob Data Reader | Read reports only |
| compliance-officers | Storage Blob Data Reader | Audit access |
| tp1-app-identity | Storage Blob Data Contributor | App authentication |
| tp1-stream-analytics | Storage Blob Data Contributor | ASA job writes |

### Key Architecture Decisions

**Control Plane ≠ Data Plane**
```
Contributor on Resource Group = manage the storage account resource
Storage Blob Data Contributor  = read/write data inside the storage account
Both are required independently — this is a classic AZ-305 exam trap
```

**ABAC Condition on Owner**
```
platform-admins has Owner role BUT with condition:
Cannot assign Owner, User Access Administrator, or RBAC Admin roles
→ Limits blast radius even if admin account is compromised
→ Principle of least privilege applied even to privileged roles
```

---

## 🔄 Data Flow

### Stream Analytics Query
```sql
-- ALL orders → Data Lake (cold path, full history)
SELECT *, System.Timestamp() AS ProcessedTime
INTO [tp1-datalake-output]
FROM [tp1-orders-input]

-- COMPLETED orders only → Cosmos DB (hot path, fast lookups)
SELECT *, System.Timestamp() AS ProcessedTime
INTO [tp1-cosmos-output]
FROM [tp1-orders-input]
WHERE orderStatus = 'completed'
```

### Hot Path vs Cold Path
| | Cosmos DB (Hot) | Data Lake Gen2 (Cold) |
|---|---|---|
| Data | Completed orders only | ALL orders |
| Latency | < 10ms per record | Batch / Synapse |
| Use case | Operational queries | Analytics, reporting |
| Partition key | /orderId | /orders/YYYY/MM/DD/ |

---

## 🧠 AZ-305 Concepts Demonstrated

| Concept | Where Encountered |
|---|---|
| Control plane vs data plane RBAC | Step 10 — Storage RBAC |
| Group-based access management | Step 2 — Group creation |
| ABAC conditions on privileged roles | Step 6 — Owner assignment |
| System vs User assigned Managed Identity | Step 16 — ASA vs tp1-app-identity |
| Hot path vs cold path architecture | Stream Analytics query design |
| Event Hub partitioning (Kafka equivalence) | tp1-orders-hub — 2 partitions |
| Streaming Units vs Partition count | ASA job sizing |
| PIM licensing reality (Entra P2 required) | Step 7 — skipped, documented |
| Free Trial subscription limitations | ASA storage dropdown issue |
| Cosmos DB partition key requirement | Container creation error |

---

## 📁 Repository Structure

```
test_project_1/
├── README.md                          ← this file
├── arm-template/
│   ├── template.json                  ← ARM template (subscription ID redacted)
│   ├── parameters.json                ← ARM parameters
│   └── main.bicep                     ← Bicep equivalent
├── docs/
│   ├── role-assignments.csv           ← exported IAM role assignments
│   ├── azure-resources.csv            ← exported resource list
│   └── HLD.docx                       ← High Level Design document
├── queries/
│   └── audit-queries.kql             ← Log Analytics KQL queries
└── screenshots/
    ├── resource-visualizer.png        ← Azure resource visualizer
    ├── iam-resource-group.png         ← Resource group IAM assignments
    ├── iam-storage-account.png        ← Storage account IAM assignments
    ├── entra-audit-logs.png           ← Entra ID group audit logs
    └── log-analytics-query.png        ← Log Analytics workspace active
```

---

## 🚀 How to Redeploy

```bash
# Using Azure CLI
az login
az group create --name test_project_1_resource_group --location southindia

# Deploy ARM template
az deployment group create \
  --resource-group test_project_1_resource_group \
  --template-file arm-template/template.json \
  --parameters arm-template/parameters.json

# Or using Bicep
az deployment group create \
  --resource-group test_project_1_resource_group \
  --template-file arm-template/main.bicep
```

> ⚠️ Replace SUBSCRIPTION_ID_REDACTED with your actual subscription ID before deploying.

---

## 💰 Cost Management

Budget alert set at $1/month to prevent accidental charges.  
Stream Analytics job must be **stopped immediately after testing** (~₹6/hour when running).

---

## 📌 Status

- [x] Phase 1 — Identity & Governance (Entra ID, RBAC, Groups)
- [x] Phase 2 — Data Storage (Data Lake, Cosmos DB, Event Hub, ASA, Log Analytics)
- [ ] Phase 3 — Pipeline Test (start ASA, send events, validate, stop)
- [ ] Phase 4 — Business Continuity (backup, geo-redundancy, RTO/RPO)
- [ ] Phase 5 — Infrastructure (VNet, NSG, Private Endpoints, Firewall)
