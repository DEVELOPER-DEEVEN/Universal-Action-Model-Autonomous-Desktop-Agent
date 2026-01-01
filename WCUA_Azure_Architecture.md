# WCUA v2.0 - Azure-First Architecture & Deployment Guide

| **Category** | **Details** |
| :--- | :--- |
| **Version** | 2.0 Azure Edition |
| **Date** | December 2025 |
| **Target** | Enterprise-grade production (99.99% SLA) |
| **Goal** | Microsoft Imagine Cup Showcase |

---

## Strategic Rationale: Why Azure?

**Strategic Alignment**
The choice of Azure is strategic for the Imagine Cup to align with judge expectations:
*   **Judges**: Microsoft employees who prioritize native ecosystem usage.
*   **Expectation**: Professional implementation using Microsoft's own services.
*   **Result**: Demonstrates seriousness about production-grade architecture.

**Technical Advantages**
*   **Integration**: Seamless compatibility with Windows and .NET/Python ecosystems.
*   **AI/ML**: Direct access to Cognitive Services, Language, and Vision APIs.
*   **Identity**: Robust security via Azure Active Directory (Entra ID).
*   **Security**: Enterprise-grade compliance and protection built-in.
*   **Cost**: Optimization via reserved instances and serverless tiers.
*   **Reliability**: Global redundancy ensuring high availability.

---

## Part 1: Complete Azure Architecture

### 1.1 High-Level Azure Services Map

```text
┌──────────────────────────────────────────────────────────────┐
│                    INTERNET / CDN LAYER                       │
│                  (Azure Front Door + CDN)                     │
└────────┬──────────────────────────────────────────────┬───────┘
         │                                              │
┌────────▼──────────────────────────────┐   ┌─────────▼────────────┐
│    API GATEWAY LAYER                  │   │   STATIC WEB APP     │
├───────────────────────────────────────┤   ├─────────────────────┤
│ • Azure API Management                │   │ • Next.js Web App   │
│ • Rate limiting & throttling          │   │ • Azure Static Web  │
│ • Request validation                  │   │ • Azure Blob (CDN)  │
│ • OAuth 2.0 / OpenID Connect          │   │ • Global distribution
└────────┬──────────────────────────────┘   └─────────────────────┘
         │
┌────────▼──────────────────────────────────────────────────────┐
│              BACKEND COMPUTE LAYER (MULTI-OPTION)             │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────┐  ┌──────────────────────────────┐  │
│  │ Azure Container     │  │ Azure App Service            │  │
│  │ Instances (ACI)     │  │ (Docker support)             │  │
│  ├─────────────────────┤  ├──────────────────────────────┤  │
│  │ • Serverless        │  │ • Managed Python runtime     │  │
│  │ • FastAPI containers│  │ • Auto-scaling               │  │
│  │ • Pay-per-second    │  │ • Kubernetes-ready  │        │  │
│  └─────────────────────┘  └──────────────────────────────┘  │
│                                                               │
│  RECOMMENDED FOR WCUA: Azure App Service (Tier: Premium)    │
│  - Predictable performance                                   │
│  - Supports Python/FastAPI natively                          │
│  - Auto-scaling included                                     │
│  - Deployment slots for zero-downtime                        │
│                                                               │
└──────────────────────────┬───────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
┌───────▼──────┐  ┌────────▼────────┐  ┌────▼──────────────┐
│  DATA LAYER  │  │  MESSAGE QUEUE  │  │  AI/ML LAYER      │
├──────────────┤  ├─────────────────┤  ├──────────────────┤
│ • PostgreSQL │  │ • Azure Service │  │ • Cognitive Svcs │
│ • Azure SQL  │  │   Bus           │  │ • Language (OCR) │
│ • Cosmos DB  │  │ • Azure Queue   │  │ • Vision API     │
│ • Redis      │  │ • Event Grid    │  │ • Speech Services│
└──────────────┘  └─────────────────┘  └──────────────────┘

Additional Services:
* Azure Monitor (Monitoring)
* Application Insights (APM)
* Azure Key Vault (Secrets)
* Azure DevOps (CI/CD)
* Azure Container Registry (ACR)
* Azure Backup (Disaster Recovery)
```

---

## Part 2: Detailed Azure Services Breakdown

### 2.1 Frontend Hosting

#### Azure Static Web Apps (Next.js)
*   **Purpose**: Host the Next.js web dashboard.
*   **Cost**: Free tier available; Production tier ~$99/month.
*   **Key Features**:
    *   Global CDN distribution.
    *   Automatic SSL/TLS.
    *   CI/CD integration via GitHub.
    *   Staging environments and API backend integration.

**Setup Steps**:
1.  Create Static Web App resource.
2.  Connect GitHub repository.
3.  Configure build output directory (`out/`).
4.  Deploy on push to `main`.
5.  Configure custom domain (e.g., `wcua-web.azurestaticapps.com`).

#### Azure Blob Storage (Asset CDN)
*   **Purpose**: Storage for app icons, screenshots, and media.
*   **Cost**: ~$0.024/GB/month.
*   **Features**: 11-nines durability, tiered storage (Hot/Cool/Archive), and replication.

**Setup Steps**:
1.  Create Storage Account (General-purpose v2).
2.  Create public "assets" container and private "backups" container.
3.  Enable Azure CDN.
4.  Configure CORS.

#### Azure CDN (Azure Front Door)
*   **Purpose**: Global content delivery, load balancing, and DDoS protection.
*   **Cost**: ~$100/month.
*   **Features**: Global edge locations, WAF (Web Application Firewall), SSL/TLS termination.

---

### 2.2 Backend Hosting

#### Azure App Service (FastAPI Backend)
*   **Purpose**: Host the FastAPI application.
*   **Cost**: $13–$240/month. **Recommended**: Premium (P1V2) for production (~$100/mo).
*   **Features**: Auto-scaling, deployment slots, built-in monitoring, Linux support (Python 3.11).

**Configuration Details**:
*   **Runtime**: Python 3.11.
*   **Startup Command**: `gunicorn --workers 4 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000 main:app`

**Environment Variables**:
*   `CLAUDE_API_KEY`: [Managed via Key Vault]
*   `DATABASE_URL`: [PostgreSQL Connection String]
*   `REDIS_URL`: [Azure Cache for Redis Connection String]
*   `ENVIRONMENT`: `production`

**Auto-Scaling Rules**:
1.  **Scale Out**: Trigger when CPU > 70%. Max 10 instances.
2.  **Scale In**: Trigger when CPU < 30% for 10 minutes. Min 2 instances.
3.  **Memory**: Scale up if Memory > 80%.

---

### 2.3 Database Layer

#### Azure Database for PostgreSQL (Flexible Server)
*   **Purpose**: Primary storage for user data, task history, and analytics.
*   **Cost**: $40–$150/month.
*   **Configuration**: General Purpose (2 vCores, 8 GB RAM), 32 GB Storage (Auto-scale).
*   **Backup**: 7-day retention (Point-in-Time Restore), Geo-redundant.

**Connection String Format**:
```text
postgresql://admin:password@wcua-db.postgres.database.azure.com:5432/wcua_prod?sslmode=require
```

#### Azure Cache for Redis
*   **Purpose**: Caching, session storage, and real-time state management.
*   **Cost**: $20–$150/month. **Recommended**: Premium (6GB) for persistence and clustering.
*   **Use Cases**:
    *   **LLM Caching**: Store model responses for 1 hour.
    *   **Session Storage**: User sessions (24h TTL).
    *   **Real-time**: WebSocket state for task execution.

**Python Integration**:
```python
import redis
# ... setup code ...
redis_client = redis.from_url("rediss://wcua-redis.redis.cache.windows.net:6380", ssl_certfile=None, decode_responses=True)
```

---

### 2.4 AI/ML Services

#### Azure Cognitive Services - Vision
*   **Purpose**: OCR (Text extraction) and UI element detection from screenshots.
*   **Cost**: Pay-per-use (approx $1–4/month).
*   **Features**: Serverless, high accuracy, supports 70+ languages.

#### Azure Speech Services
*   **Purpose**: Speech-to-Text (Voice Commands) and Text-to-Speech (Responses).
*   **Cost**: Pay-per-use or Subscription (~$49/month).
*   **Features**: Natural neural voices, real-time processing.

---

### 2.5 Identity & Security

#### Microsoft Entra ID (Azure AD)
*   **Purpose**: Authentication, SSO, and user management.
*   **Cost**: Free tier is sufficient for basic auth; Premium available for advanced features.
*   **Flow**: OAuth 2.0 / OpenID Connect with Microsoft Graph permissions.

#### Azure Key Vault
*   **Purpose**: Secure management of keys, secrets, and certificates.
*   **Cost**: Minimal ($1–2/month).
*   **Best Practice**: Use Managed Identities to access Key Vault without hardcoding credentials.

---

### 2.6 Monitoring & Observability

#### Azure Application Insights
*   **Purpose**: APM (Application Performance Monitoring), error tracking, and diagnostics.
*   **Integration**: OpenTelemetry SDK for FastAPI.
*   **Capabilities**:
    *   **Tracing**: Distributed tracing for API calls.
    *   **Metrics**: Custom metrics (e.g., `wcua.tasks.executed`).
    *   **Logging**: Centralized log aggregation.

**Alert Configuration**:
*   **Error Rate**: Critical alert if > 5% in 5 min.
*   **Latency**: Warning if P95 > 5s.
*   **Success Rate**: Warning if < 90%.

---

### 2.7 DevOps

#### CI/CD Pipelines
We recommend **GitHub Actions** for native integration with the repository.

*   **Backend Workflow**:
    1.  Test (Pytest).
    2.  Build Docker Image.
    3.  Push to Azure Container Registry (ACR).
    4.  Deploy to Azure App Service (Deployment Slots).
*   **Frontend Workflow**:
    1.  Build Next.js app.
    2.  Deploy to Azure Static Web Apps.

#### Azure Container Registry (ACR)
*   **Purpose**: Private Docker registry for backend images.
*   **Cost**: ~$5/month (Standard).
*   **Features**: Vulnerability scanning, geo-replication.

---

## Part 3: Production Deployment Checklist

### Phase 1: Infrastructure Setup
- [ ] Create Azure Resource Group (`wcua-prod`).
- [ ] Deploy data services (PostgreSQL, Redis).
- [ ] Provision Compute (App Service, Static Web App).
- [ ] Configure Networking (VNets, NSGs).
- [ ] Initialize Key Vault and store secrets.
- [ ] Setup Entra ID application registration.

### Phase 2: Security Hardening
- [ ] Enable SSL/TLS enforcement.
- [ ] Configure Firewall rules (whitelist IPs).
- [ ] Enable Managed Identities for services.
- [ ] Turn on Azure Defender for Cloud.
- [ ] Verify encryption at rest.

### Phase 3: Monitoring & CI/CD
- [ ] Configure Application Insights.
- [ ] Set up Alert Rules (Latency, Errors, CPU).
- [ ] Create GitHub Actions workflows.
- [ ] Verify Deployment Slots functionality.

### Phase 4: Production Launch
- [ ] Run full test suite (Unit/Integration).
- [ ] Perform Load Testing (1000+ concurrent).
- [ ] Verify Backup/Restore procedures.
- [ ] Deploy to Production Slot.
- [ ] Post-deployment Health Check.

---

## Part 4: Cost Analysis

### Monthly Cost Estimation (Production)

| **Service Component** | **Service Name** | **Est. Cost (Monthly)** |
| :--- | :--- | :--- |
| **Frontend** | Azure Static Web Apps | $99 |
| **Frontend** | Azure Front Door (CDN) | $100 |
| **Frontend** | Blob Storage | $1 |
| **Backend** | App Service (Premium) | $100 |
| **Data** | PostgreSQL Flexible | $70 |
| **Data** | Azure Cache for Redis | $80 |
| **AI/ML** | Cognitive Services | $15 |
| **DevOps** | Container Registry | $5 |
| **Monitoring** | Application Insights | $10 |
| **Security** | Key Vault | $1 |
| **Total** | | **~$481 / month** |

### Optimization Opportunities
*   **Reserved Instances**: Commit to 1-3 years for App Service/DB to save ~30% ($144 savings).
*   **Auto-scaling**: Scale down during off-hours ($30 savings).
*   **Optimization Total**: Potential reduced cost to **~$272 / month**.

---

## Part 5: Disaster Recovery & Continuity

**Recovery Targets**
*   **RTO (Recovery Time)**: < 5 minutes for Database; < 1 minute for App Service.
*   **RPO (Recovery Point)**: < 1 minute (transactions).
*   **Uptime Goal**: 99.99%.

**Procedures**
*   **Database Failure**: Automatic failover to geo-replica.
*   **Region Failure**: Manual trigger to secondary region (West US).
*   **Data Corruption**: Point-in-time restore from backup (1 hour window).

---

## Summary

The proposed architecture leverages the full power of the Azure ecosystem to provide a secure, scalable, and enterprise-ready solution for WCUA.

**Key Deliverables**:
1.  **Professionalism**: Full usage of managed services.
2.  **Performance**: Global CDN and Redis caching.
3.  **Reliability**: Multi-region redundancy and auto-scaling.
4.  **Security**: Zero-trust model with Entra ID and Key Vault.

**Status**: Ready for Implementation.
