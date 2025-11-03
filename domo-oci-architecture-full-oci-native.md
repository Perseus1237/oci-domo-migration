================================================================================
              DOMO ANALYTICS CONTROL PLANE ON ORACLE CLOUD (OCI)
                        (Target Architecture Post-Migration)
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│                          USER INTERACTION LAYER                               │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │   Web UI    │  │  Mobile App │  │  Domo Apps  │  │ Embedded BI │       │
│  │  (React)    │  │   (Native)  │  │ (Custom JS) │  │ (iFrames)   │       │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘       │
│         │                 │                 │                 │              │
│         └─────────────────┴─────────────────┴─────────────────┘              │
│                                     │                                         │
└─────────────────────────────────────┼─────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                          API GATEWAY & LOAD BALANCING                         │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  OCI Load Balancer (Flexible Shape)                                    │ │
│  │  • SSL/TLS Termination with OCI Certificates Service                   │ │
│  │  • Path-based routing & host-based routing                             │ │
│  │  • Health checks & connection draining                                 │ │
│  │  • WAF integration (OCI Web Application Firewall)                      │ │
│  │  • Regional deployment with cross-AD distribution                      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Domo API Gateway Service                                              │ │
│  │  • Authentication/Authorization (OAuth 2.0, SAML, SSO, IDCS)           │ │
│  │  • OCI API Gateway integration for rate limiting                       │ │
│  │  • Request routing & transformation                                    │ │
│  │  • API versioning & deployment strategies                              │ │
│  │  Runtime: OKE (Container Engine for Kubernetes)                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└─────────────────────────────────────┬─────────────────────────────────────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    │                                   │
                    ▼                                   ▼
┌──────────────────────────────────────┐  ┌────────────────────────────────────┐
│    CONTROL PLANE SERVICES LAYER      │  │   AI SERVICE LAYER (Domo.AI)      │
├──────────────────────────────────────┤  ├────────────────────────────────────┤
│                                      │  │                                    │
│ ┌──────────────────────────────────┐ │  │ ┌────────────────────────────────┐│
│ │  Metadata Service                │ │  │ │  AI Chat Agent                 ││
│ │  • Dataset catalog               │ │  │ │  • Natural language processing ││
│ │  • Schema registry               │ │  │ │  • Guardrails enforcement      ││
│ │  • Lineage tracking              │ │  │ │  • Context management          ││
│ │  • Data dictionary               │ │  │ │  Runtime: OKE pods             ││
│ │  Store: OCI Base Database /      │ │  │ └──────────┬─────────────────────┘│
│ │         Autonomous Database      │ │  │           │                       │
│ │  Backup: OCI Object Storage      │ │  │ ┌─────────▼──────────────────────┐│
│ └──────────────────────────────────┘ │  │ │  Agent Planner                  ││
│                                      │  │ │  • Task orchestration          ││
│ ┌──────────────────────────────────┐ │  │ │  • Tool selection              ││
│ │  Query Orchestrator              │ │  │ │  • OCI Generative AI API calls ││
│ │  • Query parsing & optimization  │ │  │ │  • Execution planning          ││
│ │  • Workload management           │ │  │ │  • Multi-model routing         ││
│ │  • Resource allocation           │ │  │ └──────────┬─────────────────────┘│
│ │  • Query routing:                │ │  │           │                       │
│ │    - Tundra (OCI Compute)        │ │  │ ┌─────────▼──────────────────────┐│
│ │    - ADB (replaces Vertica)      │ │  │ │  Tool Execution Layer          ││
│ │    - Snowflake on OCI            │ │  │ │  • Vector search (OCI OpenSearch│
│ │  • Cost-based optimizer          │ │  │ │    or OCI Database 23ai vectors)│
│ │  Runtime: OKE microservices      │ │  │ │  • SQL generation              ││
│ └─────────┬────────────────────────┘ │  │ │  • Domo Cloud Amplifier        ││
│           │                          │  │ │  • Knowledge base (Object Store)││
│ ┌─────────▼────────────────────────┐ │  │ └────────────────────────────────┘│
│ │  Cluster Manager                 │ │  │           │                       │
│ │  • Adrenaline cluster lifecycle  │ │  │ ┌─────────▼──────────────────────┐│
│ │  • Auto-scaling (OCI Autoscaling)│ │  │ │  Model Management              ││
│ │  • Health monitoring (OCI Mon.)  │ │  │ │  • FM selection & routing      ││
│ │  • Hydration orchestration       │ │  │ │  • Performance tracking        ││
│ │  • Failover management           │ │  │ │  • Prompt engineering          ││
│ │  Backend: OCI Compute Pool       │ │  │ │  Backend: OCI Generative AI    ││
│ └──────────────────────────────────┘ │  │ │  Service:                      ││
│                                      │  │ │  - Cohere Command R/R+         ││
│ ┌──────────────────────────────────┐ │  │ │  - Meta Llama 3/3.1/3.2        ││
│ │  Security & Governance Service   │ │  │ │  OR Amazon Bedrock (if hybrid) ││
│ │  • Access control (OCI IAM)      │ │  │ └────────────────────────────────┘│
│ │  • IDCS integration (SSO/SAML)   │ │  │                                    │
│ │  • Data encryption (OCI Vault)   │ │  └────────────────────────────────────┘
│ │  • Audit logging (OCI Audit)     │ │
│ │  • Policy enforcement (OCI Pol.) │ │
│ │  • Compliance reporting          │ │
│ └──────────────────────────────────┘ │
│                                      │
│ ┌──────────────────────────────────┐ │
│ │  Job Scheduler & Workflow Mgmt   │ │
│ │  • ETL job scheduling            │ │
│ │  • Connector refresh scheduling  │ │
│ │  • Alert & notification engine   │ │
│ │  • Workflow orchestration        │ │
│ │  OCI: Events Service, Streaming  │ │
│ │  • OCI Queue integration         │ │
│ └──────────────────────────────────┘ │
│                                      │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                      DATA INGESTION & INTEGRATION LAYER                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Connector Service (1000+ Pre-built Connectors)                        │ │
│  │  • OCI: Object Storage, Autonomous Database, Base Database, MySQL     │ │
│  │  • Multi-cloud: AWS, Azure, GCP services (cross-cloud)                │ │
│  │  • SaaS: Salesforce, Google Analytics, Marketo, etc.                  │ │
│  │  • Databases: Oracle, MySQL, PostgreSQL, SQL Server, MongoDB          │ │
│  │  • Files: CSV, Excel, JSON, Parquet, ORC, Avro                        │ │
│  │  Runtime: OCI Functions + OCI Container Instances                     │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Domo Workbench (On-Prem/Multi-Cloud Agent)                            │ │
│  │  • On-premises data source connectivity                                │ │
│  │  • Secure tunneling (FastConnect/VPN/Site-to-Site IPSec)              │ │
│  │  • Change data capture (CDC)                                           │ │
│  │  • Data encryption in transit (TLS 1.3)                                │ │
│  │  • Multi-cloud bridge (AWS/Azure/GCP → OCI)                           │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  API & SDK Integration Layer                                           │ │
│  │  • REST API endpoints (OCI API Gateway)                                │ │
│  │  • Custom connector SDK                                                │ │
│  │  • Webhook receivers (OCI Functions)                                   │ │
│  │  • Streaming ingestion (OCI Streaming Service - Kafka compatible)     │ │
│  │  • GoldenGate integration for Oracle DB CDC                            │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                         │
└─────────────────────────────────────┼─────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    DATA PROCESSING & TRANSFORMATION LAYER                     │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Magic ETL Engine                                                      │ │
│  │  • Visual drag-and-drop transformation designer                        │ │
│  │  • Pre-built transformation tiles (join, filter, aggregate, etc.)     │ │
│  │  • SQL transforms (Oracle SQL, PL/SQL support)                         │ │
│  │  • Upsert & partition operations                                       │ │
│  │  Runtime: OCI Data Flow (Apache Spark) - Serverless                   │ │
│  │  • Autoscaling Spark clusters                                          │ │
│  │  • Support for Spark 3.x with Delta Lake                               │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Data Science Workspace                                                │ │
│  │  • Jupyter Notebooks (OCI Data Science service)                        │ │
│  │  • R & Python script execution (conda environments)                    │ │
│  │  • ML model training (OCI Data Science Jobs)                           │ │
│  │  • Model deployment (OCI Model Deployment)                             │ │
│  │  • Sandbox environment for experimentation                             │ │
│  │  • Code Engine for custom processing                                   │ │
│  │  • GPU shapes for ML workloads (A10, H100)                             │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Data Quality & Validation Service                                     │ │
│  │  • Schema validation                                                   │ │
│  │  • Data profiling & statistics                                         │ │
│  │  • Anomaly detection (OCI Anomaly Detection service)                   │ │
│  │  • DQ rules engine                                                     │ │
│  │  • Integration with OCI Data Catalog                                   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└─────────────────────────────────────┬─────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│              DURABLE STORAGE LAYER (OCI OBJECT STORAGE BACKEND)               │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  OCI Object Storage - Primary Durability Layer                         │ │
│  │  (11 nines durability, locally redundant or cross-region replication)  │ │
│  │                                                                          │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Raw Data Lake (Standard Tier)                                    │ │ │
│  │  │  • Ingested data in native formats                                │ │ │
│  │  │  • Buckets organized by: tenant/dataset/date                      │ │ │
│  │  │  • Partitioning: Hive-style partitions                            │ │ │
│  │  │  • Compression: Parquet, ORC, Snappy, Gzip                        │ │ │
│  │  │  • Encryption: Server-side with OCI Vault keys (AES-256)          │ │ │
│  │  │  • Lifecycle policy: Archive tier after 90 days                   │ │ │
│  │  │  • Object versioning enabled                                      │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                          │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Processed Data Store (Infrequent Access Tier)                    │ │ │
│  │  │  • Transformed & cleansed datasets                                │ │ │
│  │  │  • Optimized columnar formats (Parquet with Bloom filters)        │ │ │
│  │  │  • Pre-aggregated rollup tables                                   │ │ │
│  │  │  • Source of truth for Adrenaline hydration                       │ │ │
│  │  │  • Multi-part upload optimization                                 │ │ │
│  │  │  • Pre-authenticated request (PAR) for secure access              │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                          │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Metadata & Config Store                                          │ │ │
│  │  │  • Dataset manifests (JSON)                                       │ │ │
│  │  │  • Query result caches                                            │ │ │
│  │  │  • Checkpoint data                                                │ │ │
│  │  │  • Backup: Autonomous Database backups                            │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                          │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Archive Tier (Cold Storage)                                      │ │ │
│  │  │  • Historical data > 1 year                                       │ │ │
│  │  │  • Compliance & audit logs                                        │ │ │
│  │  │  • 90% cost savings vs Standard tier                              │ │ │
│  │  │  • Restore time: ~4 hours                                         │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Write-Back Integration                                                 │ │
│  │  • Object Storage bucket write operations                              │ │
│  │  • Autonomous Database write-back connector                            │ │
│  │  • Cross-cloud write-back (to AWS S3, Azure Blob, GCP GCS)            │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└─────────────────────────────────────┬─────────────────────────────────────────┘
                                      │
                            ┌─────────┴─────────┐
                            │  HYDRATION LAYER  │
                            │ (Object Storage → │
                            │    Adrenaline)    │
                            └─────────┬─────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│            ADRENALINE QUERY ENGINE LAYER (Analytics) - OCI EDITION            │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  ADRENALINE - Hybrid Query Engine Architecture on OCI                  │ │
│  │  (Non-Durable, Hydrated from Object Storage, Multi-Tenant)             │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │      TUNDRA CLUSTERS            │  │  AUTONOMOUS DATABASE (ADB)      │  │
│  │   (Custom-Built Engine)         │  │   (Replaces Vertica)            │  │
│  ├─────────────────────────────────┤  ├─────────────────────────────────┤  │
│  │                                 │  │                                 │  │
│  │ ┌─────────────────────────────┐ │  │ ┌─────────────────────────────┐ │  │
│  │ │  Tundra Cluster A (Tenant1) │ │  │ │ ADB-DW Cluster A (TenantX)  │ │  │
│  │ │  • In-memory columnar store │ │  │ │ • Autonomous Data Warehouse │ │  │
│  │ │  • MPP query execution      │ │  │ │ • Self-tuning, self-healing │ │  │
│  │ │  • 8x faster than Snowflake │ │  │ │ • Columnar storage (IMCU)   │ │  │
│  │ │  • Optimized for analytics  │ │  │ │ • In-Memory (IM column store│ │  │
│  │ │  • Query result caching     │ │  │ │ • Parallel query execution  │ │  │
│  │ │  • OCI-native architecture  │ │  │ │ • HeatWave acceleration     │ │  │
│  │ │                             │ │  │ │ • Automatic indexing        │ │  │
│  │ │  Compute Shapes:            │ │  │ │                             │ │  │
│  │ │  • E5/E6 (Memory-optimized) │ │  │ │  Serverless or Dedicated    │ │  │
│  │ │  • HM (High Memory) shapes  │ │  │ │  • Auto-scaling ECPUs       │ │  │
│  │ │  • 256GB-2TB RAM per node   │ │  │ │  • Storage auto-scaling     │ │  │
│  │ │  • 100Gbps network backbone │ │  │ │  • Integrated backups       │ │  │
│  │ │  • NVMe SSD for temp data   │ │  │ │  • Cross-region DR          │ │  │
│  │ └─────────────────────────────┘ │  │ └─────────────────────────────┘ │  │
│  │           ▲                     │  │           ▲                     │  │
│  │           │ Hydrate from        │  │           │ Bulk load from      │  │
│  │           │ Object Storage      │  │           │ Object Storage      │  │
│  │           │ (high-throughput    │  │           │ (OCI Data Pump,     │  │
│  │           │  parallel download) │  │           │  DBMS_CLOUD)        │  │
│  │           │                     │  │           │                     │  │
│  │ ┌─────────┴───────────────────┐ │  │ ┌─────────┴───────────────────┐ │  │
│  │ │  Tundra Cluster B (Tenant2) │ │  │ │ ADB-DW Cluster B (TenantY)  │ │  │
│  │ │  • Cluster autoscaling      │ │  │ │ • Elastic compute scaling   │ │  │
│  │ │  • Instance pools           │ │  │ │ • Auto resource mgmt        │ │  │
│  │ └─────────────────────────────┘ │  │ └─────────────────────────────┘ │  │
│  │           ▲                     │  │           ▲                     │  │
│  │           │                     │  │           │                     │  │
│  │ ┌─────────┴───────────────────┐ │  │ ┌─────────┴───────────────────┐ │  │
│  │ │  Tundra Cluster C (Tenant3) │ │  │ │ ADB-DW Cluster C (TenantZ)  │ │  │
│  │ │  • DenseIO shapes (optional)│ │  │ │ • Dedicated Exadata infra   │ │  │
│  │ │  • 51TB-408TB NVMe per node │ │  │ │   (for largest tenants)     │ │  │
│  │ └─────────────────────────────┘ │  │ └─────────────────────────────┘ │  │
│  │                                 │  │                                 │  │
│  │ Routing Logic:                  │  │ Routing Logic:                  │  │
│  │ • Small datasets (<100GB)       │  │ • Large datasets (>500GB)       │  │
│  │ • High query frequency          │  │ • Complex queries               │  │
│  │ • Real-time dashboards          │  │ • Advanced analytics (ML)       │  │
│  │ • Sub-second latency needs      │  │ • Graph queries                 │  │
│  │ • Hot data workloads            │  │ • Spatial analytics             │  │
│  │                                 │  │ • JSON, XML processing          │  │
│  │ Storage:                        │  │                                 │  │
│  │ • OCI Block Volumes (Ultra     │  │ Benefits vs Vertica:            │  │
│  │   High Perf) for temp storage  │  │ • No cluster mgmt overhead      │  │
│  │ • 200K+ IOPS per volume        │  │ • Automatic patching/upgrades   │  │
│  │                                 │  │ • Autonomous operations         │  │
│  │                                 │  │ • Pay-per-use pricing           │  │
│  │                                 │  │ • Oracle SQL compatibility      │  │
│  └─────────────────────────────────┘  └─────────────────────────────────┘  │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Query Routing Intelligence (Enhanced for OCI)                         │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Decision Engine - Routes between: Tundra | ADB | Snowflake-OCI  │ │ │
│  │  │  Based on:                                                        │ │ │
│  │  │  • Dataset size & characteristics                                │ │ │
│  │  │  • Query complexity & SQL features needed                        │ │ │
│  │  │  • Oracle SQL features (JSON, Spatial, Graph, ML)                │ │ │
│  │  │  • Tenant tier & SLA requirements                                │ │ │
│  │  │  • Current cluster load & availability                           │ │ │
│  │  │  • Cost optimization (ADB auto-scaling vs fixed Tundra)          │ │ │
│  │  │  • OCI region capacity & proximity                               │ │ │
│  │  │  • Historical query performance metrics                          │ │ │
│  │  │  • Exadata utilization (for ADB on Exadata)                      │ │ │
│  │  │  NOTE: Customer never knows which engine is used                 │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Cluster Lifecycle Management (OCI Edition)                            │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Hydration Process (Object Storage → Tundra):                    │ │ │
│  │  │  1. Cluster Launch (OCI Compute Auto Scaling / Instance Pool)    │ │ │
│  │  │  2. Object Storage data fetch via private endpoint               │ │ │
│  │  │     • Parallel downloads (100+ concurrent streams)               │ │ │
│  │  │     • OCI backbone network (100Gbps inter-VM)                    │ │ │
│  │  │     • Pre-authenticated requests for security                    │ │ │
│  │  │  3. Data loading into cluster memory/NVMe                        │ │ │
│  │  │  4. Index building & statistics computation                      │ │ │
│  │  │  5. Cluster warm-up queries                                      │ │ │
│  │  │  6. Ready for production (3-12 minutes on OCI - faster than AWS) │ │ │
│  │  │     • OCI's high-throughput network accelerates hydration        │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                          │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │  ADB Provisioning (Object Storage → ADB):                         │ │ │
│  │  │  1. Autonomous Database spins up (3-5 minutes)                   │ │ │
│  │  │  2. DBMS_CLOUD package used for bulk load                        │ │ │
│  │  │  3. External tables over Object Storage (no data movement)       │ │ │
│  │  │  4. Bulk load into ADB with parallel DML                         │ │ │
│  │  │  5. Automatic indexing kicks in                                  │ │ │
│  │  │  6. Query-ready in minutes                                       │ │ │
│  │  │  OR: Use ADB External Tables for query-in-place on Object Storage│ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                          │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Cluster Failure & Recovery (OCI):                                │ │ │
│  │  │  1. Health check failure (OCI Monitoring alarms)                 │ │ │
│  │  │  2. Traffic redirected via Load Balancer to healthy nodes        │ │ │
│  │  │  3. Failed Tundra cluster terminated (no data loss - Obj Storage)│ │ │
│  │  │  4. New cluster from instance pool launched                      │ │ │
│  │  │  5. Re-hydration from Object Storage (faster on OCI)             │ │ │
│  │  │  6. Cluster back online (RTO: 5-15 minutes on OCI)               │ │ │
│  │  │                                                                   │ │ │
│  │  │  ADB Failure & Recovery:                                          │ │ │
│  │  │  • ADB handles failover automatically (RAC on Exadata)           │ │ │
│  │  │  • Zero data loss (RPO=0) with Fast-Start Failover               │ │ │
│  │  │  • RTO: <2 minutes for ADB autonomous failover                   │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Query Execution Path (OCI-Optimized):                                │ │
│  │  1. Query arrives at Adrenaline orchestrator (OKE service)             │ │
│  │  2. Query parsed & optimized (cost-based optimizer)                    │ │
│  │  3. Execution plan generated (Tundra MPP or ADB optimizer)             │ │
│  │  4. Query distributed across cluster nodes / ADB nodes                 │ │
│  │  5. Parallel execution (OCI RDMA network for low latency)              │ │
│  │  6. Results aggregated & cached (OCI Cache / Redis)                    │ │
│  │  7. Results returned to app layer                                      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    EXTERNAL DATA WAREHOUSE INTEGRATION                        │
│                    (Domo Cloud Amplifier / Federated - OCI)                   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Snowflake on OCI Integration (KEY MIGRATION DEPENDENCY)               │ │
│  │  • Snowflake availability on OCI is critical for migration             │ │
│  │  • Federated query pushdown (query-in-place)                           │ │
│  │  • Native connector with OCI FastConnect for high-throughput           │ │
│  │  • Customer data stays in Snowflake                                    │ │
│  │  • Cross-cloud: Snowflake on AWS/Azure can still connect to Domo-OCI  │ │
│  │  • Private connectivity via FastConnect + cross-cloud peering          │ │
│  │  NOTE: Domo is verifying Snowflake-OCI readiness with Snowflake       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Autonomous Database External Tables                                   │ │
│  │  • Query OCI Object Storage in-place (no data movement)                │ │
│  │  • Supports Parquet, ORC, Avro, JSON, CSV                              │ │
│  │  • Predicate pushdown to Object Storage                                │ │
│  │  • Automatic statistics gathering                                      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Other Cloud Data Warehouse Adapters (Multi-Cloud)                     │ │
│  │  • Amazon Redshift (via VPN/FastConnect to AWS)                        │ │
│  │  • Google BigQuery (via VPN/FastConnect to GCP)                        │ │
│  │  • Databricks (on AWS/Azure/OCI)                                       │ │
│  │  • Azure Synapse (via ExpressRoute peering)                            │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Oracle Database Integration (Enhanced on OCI)                         │ │
│  │  • Direct connection to Oracle DB (on-prem or cloud)                   │ │
│  │  • GoldenGate for real-time replication                                │ │
│  │  • Oracle Data Integrator (ODI) integration                            │ │
│  │  • Oracle Analytics Cloud (OAC) interoperability                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                     CACHING & ACCELERATION LAYER                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  OCI Cache (Redis-Compatible Managed Service)                          │ │
│  │  • Query result caching (TTL-based, LRU eviction)                      │ │
│  │  • Session state management                                            │ │
│  │  • Dashboard metadata caching                                          │ │
│  │  • Real-time aggregation cache                                         │ │
│  │  • High availability: Multi-node clusters across ADs                   │ │
│  │  • In-memory performance: <1ms latency                                 │ │
│  │  • Backup to Object Storage                                            │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Content Delivery                                                       │ │
│  │  Option 1: Keep CloudFront (multi-cloud hybrid)                        │ │
│  │  Option 2: Build custom CDN on OCI with edge locations                 │ │
│  │  • Static asset delivery (JS, CSS, images)                             │ │
│  │  • Dashboard thumbnails                                                │ │
│  │  • Global distribution for low latency                                 │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  ADB In-Memory Column Store (Bonus Caching Layer)                      │ │
│  │  • Automatic population of hot data in IM column store                 │ │
│  │  • 100x-1000x faster scans for analytics queries                       │ │
│  │  • No application changes needed                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                      PRESENTATION & VISUALIZATION LAYER                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Domo Analyzer (Visualization Engine) - OCI Edition                    │ │
│  │  • 150+ chart types (proprietary viz engine)                           │ │
│  │  • Real-time dashboard rendering                                       │ │
│  │  • Drill-down & filtering                                              │ │
│  │  • Trellis/small multiples support                                     │ │
│  │  • Custom maps & geospatial viz (Oracle Spatial integration)           │ │
│  │  Runtime: OKE pods with auto-scaling                                   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Beast Mode (Calculated Fields Engine)                                 │ │
│  │  • User-defined metrics & calculations                                 │ │
│  │  • SQL-like expressions (Oracle SQL dialect support)                   │ │
│  │  • Aggregation functions                                               │ │
│  │  • Computed at query time (pushed to ADB when possible)                │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Domo Everywhere (Embedded Analytics)                                  │ │
│  │  • White-label embedding                                               │ │
│  │  • iFrame & JavaScript SDK                                             │ │
│  │  • Programmable filters                                                │ │
│  │  • SSO via IDCS (Oracle Identity Cloud Service)                        │ │
│  │  • Cross-cloud embedding (works with apps on AWS/Azure/GCP)            │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                      APPLICATION DEVELOPMENT PLATFORM                         │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Domo App Dev Framework (OCI Edition)                                  │ │
│  │  • Custom app builder (HTML/CSS/JS)                                    │ │
│  │  • Domo.js SDK & REST APIs                                             │ │
│  │  • AppDB: Oracle NoSQL Database Cloud Service or ADB JSON              │ │
│  │  • Files API: Object Storage backed                                    │ │
│  │  • Serverless runtime: OCI Functions + Container Instances             │ │
│  │  • Auto-scaling: OKE horizontal pod autoscaling                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Domo Workflows (Automation Engine)                                    │ │
│  │  • Visual workflow designer                                            │ │
│  │  • Event-driven triggers: OCI Events, Streaming, Queue                 │ │
│  │  • Action connectors (email, Slack, Jira, OCI services)                │ │
│  │  • Integration with OCI Functions                                      │ │
│  │  • Orchestration: OCI Resource Manager, Terraform                      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                         MONITORING & OPERATIONS LAYER                          
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  OCI Monitoring & Logging                                                     │
│  • Metrics: Query latency, cluster health, API response times                │
│  • Logs: Application logs (Logging service), query logs, audit trails        │
│  • Alarms: Auto-scaling triggers, failure notifications, SNS/PagerDuty       │
│  • Dashboards: Real-time operational metrics                                 │
│  • Log Analytics: Advanced log search, pattern detection, anomaly detection  │
│  • Service Connector Hub: Stream logs/metrics to other services              │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  OCI Observability & Management Platform                                      │
│  • Application Performance Monitoring (APM)                                   │
│  • Distributed tracing across microservices                                   │
│  • Stack Monitoring for infrastructure health                                 │
│  • Database Management for ADB performance insights                           │
│  • Ops Insights for predictive analytics on resource usage                    │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  OCI Audit Service                                                            │
│  • API call auditing & compliance logging (immutable logs)                    │
│  • Security event monitoring                                                  │
│  • Compliance reporting (GDPR, HIPAA, SOC 2)                                  │
│  • Retention: Up to 10 years in Archive Storage                               │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  Custom Domo Operations Platform (OCI-Enhanced)                               │
│  • Tenant health monitoring dashboards                                        │
│  • Cluster utilization & optimization                                         │
│  • Cost tracking (OCI Cost Analysis, chargeback by tenant)                    │
│  • Performance benchmarking:                                                  │
│    - Tundra vs ADB vs Snowflake-OCI                                           │
│    - OCI vs AWS performance comparison                                        │
│  • Capacity planning with OCI Ops Insights                                    │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                    NETWORK & CONNECTIVITY ARCHITECTURE                         
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  Multi-AD VCN (Virtual Cloud Network) Architecture                            │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Public Subnets (Multi-AD for HA)                                      │ │
│  │  • Load Balancer endpoints                                             │ │
│  │  • NAT Gateways (for private subnet outbound)                          │ │
│  │  • Bastion hosts (with session recording)                              │ │
│  │  • DRG (Dynamic Routing Gateway) for external connectivity             │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Private Subnets - Application Tier (Multi-AD)                         │ │
│  │  • OKE worker nodes (API Gateway, control plane services)              │ │
│  │  • Container Instances                                                 │ │
│  │  • OCI Functions subnets                                               │ │
│  │  • Data Flow (Spark) execution subnets                                 │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Private Subnets - Data Tier (Multi-AD)                                │ │
│  │  • Tundra cluster compute instances                                    │ │
│  │  • Autonomous Database endpoints (private)                             │ │
│  │  • OCI Cache clusters                                                  │ │
│  │  • Base Database (for metadata if needed)                              │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Service Gateway & Private Endpoints                                   │ │
│  │  • Object Storage Service Gateway (no internet for Object Storage)     │ │
│  │  • Autonomous Database private endpoint                                │ │
│  │  • OCI Functions service endpoint                                      │ │
│  │  • OCI Generative AI service endpoint                                  │ │
│  │  • OCI Data Science endpoint                                           │ │
│  │  • All OCI services via regional service gateway                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Cross-Cloud & Hybrid Connectivity                                     │ │
│  │  • OCI FastConnect (dedicated 1-100Gbps connection)                    │ │
│  │    - To AWS via FastConnect + Direct Connect                           │ │
│  │    - To Azure via FastConnect + ExpressRoute                           │ │
│  │    - To GCP via FastConnect + Interconnect                             │ │
│  │  • Site-to-Site VPN (IPSec) for customer data centers                  │ │
│  │  • Remote VCN Peering (cross-region OCI)                               │ │
│  │  • Local VCN Peering (same-region OCI)                                 │ │
│  │  • Transit Routing via DRG (hub-and-spoke model)                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Security & Network Policies                                           │ │
│  │  • Network Security Groups (NSGs) - instance-level firewall            │ │
│  │  • Security Lists - subnet-level firewall                              │ │
│  │  • Web Application Firewall (WAF) on Load Balancer                     │ │
│  │  • DDoS protection (Network Firewall service)                          │ │
│  │  • Private DNS (OCI DNS with private views)                            │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                    KEY ARCHITECTURAL CHARACTERISTICS (OCI)                     
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  DURABILITY MODEL (Same as AWS):                                              │
│  • Object Storage is single source of truth (11 nines durability)            │
│  • Adrenaline clusters (Tundra) are EPHEMERAL & NON-DURABLE                  │
│  • ADB has automatic backups (retained 60 days, cross-region replication)    │
│  • Cluster failures = Terminate & Re-hydrate from Object Storage             │
│  • RTO: 5-15 minutes (faster than AWS due to OCI network) | RPO: 0           │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  QUERY ENGINE SELECTION (OCI Enhanced):                                      │
│  • Tundra: 8x faster than Snowflake, OCI-optimized, NOT replaced             │
│  • ADB: REPLACES Vertica, autonomous operations, Oracle SQL features         │
│  • Snowflake on OCI: Federated access, migration dependency                  │
│  • Customer transparency: Routing logic decides engine                       │
│  • NEW: ADB brings autonomous tuning, ML in-database, JSON, Spatial          │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  MULTI-TENANCY (OCI Native):                                                 │
│  • Logical isolation via compartments (OCI IAM hierarchy)                    │
│  • Tenant-specific Tundra clusters in separate instance pools                │
│  • Tenant-specific ADB instances (or schemas within shared ADB)              │
│  • Object Storage buckets per tenant with compartment isolation              │
│  • Network isolation via separate VCNs or NSGs per tenant tier               │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  HYDRATION STRATEGY (OCI Optimized):                                         │
│  • FASTER than AWS: OCI's 100Gbps backbone between VMs and Object Storage    │
│  • Parallel downloads from Object Storage (pre-authenticated requests)       │
│  • On-demand: New cluster launch, dataset refresh                            │
│  • Scheduled: Nightly full refresh, incremental during day                   │
│  • Triggered: Data Flow job completion, OCI Events-driven                    │
│  • ADB: DBMS_CLOUD.COPY_DATA for parallel bulk load (10GB+/sec)             │
│  • ADB External Tables: Query-in-place on Object Storage (zero copy)         │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  MIGRATION STRATEGY (AWS → OCI):                                             │
│  • Phase 1 (Weeks 1-4): OCI certification & infrastructure setup             │
│    - Deploy pilot tenant on OCI                                              │
│    - Test application compatibility                                          │
│    - Verify Snowflake-OCI connectivity                                       │
│    - Benchmark Tundra on OCI vs AWS                                          │
│    - Test ADB as Vertica replacement                                         │
│                                                                               │
│  • Phase 2 (Months 2-6): Hybrid operation (DOUBLE PAYMENT PERIOD)            │
│    - Run Domo on BOTH AWS and OCI                                            │
│    - Gradual tenant migration (10-20% per month)                             │
│    - Data replication: AWS S3 → OCI Object Storage                           │
│    - DNS-based traffic shifting                                              │
│    - Cross-cloud VPN for data synchronization                                │
│    - Monitor costs carefully (paying for both clouds)                        │
│                                                                               │
│  • Phase 3 (Months 7-12): OCI primary, AWS decommission                      │
│    - 80%+ tenants on OCI                                                     │
│    - AWS infrastructure downsized                                            │
│    - Final tenant migrations                                                 │
│    - AWS resources deprovisioned                                             │
│                                                                               │
│  • Considerations:                                                            │
│    - Snowflake availability on OCI: CRITICAL PATH                            │
│    - Domo + Snowflake coordination needed                                    │
│    - Cost analysis: OCI savings vs double-payment period                     │
│    - Customer impact: Transparent migration (zero downtime)                  │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  OCI ADVANTAGES OVER AWS FOR DOMO:                                           │
│  • Network Performance: 100Gbps VM-to-VM, faster hydration (3-12 min vs 5-15)│
│  • ADB Replaces Vertica: Zero cluster management, autonomous operations      │
│  • Cost Savings: 30-50% lower compute costs (especially Bare Metal/DenseIO)  │
│  • Oracle SQL Features: JSON, Spatial, Graph, ML in-database (ADB)           │
│  • Integrated Stack: OCI Gen AI, Data Science, Streaming - native services   │
│  • Bare Metal Shapes: Higher performance for Tundra (no hypervisor overhead) │
│  • DenseIO Shapes: Up to 408TB NVMe per node (vs 60TB on AWS i4i)           │
│  • HeatWave: Optional MySQL analytics acceleration                           │
│  • Exadata Cloud: Dedicated hardware for largest tenants (ADB on Exadata)    │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  POTENTIAL CHALLENGES IN OCI MIGRATION:                                      │
│  • Snowflake on OCI availability & timing                                    │
│  • Double-payment period (AWS + OCI) - need to minimize duration             │
│  • Application bugs specific to OCI (Steve's concern)                        │
│  • Cross-cloud data transfer costs during migration                          │
│  • Customer communication about potential performance differences            │
│  • Retraining operations teams on OCI tools                                  │
│  • Re-certification of security/compliance on OCI                            │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  ADB BENEFITS AS VERTICA REPLACEMENT:                                        │
│  • Autonomous: Self-tuning, self-patching, self-scaling                      │
│  • Zero Administration: No cluster management overhead                       │
│  • Advanced Analytics: Machine Learning in-database (OML)                    │
│  • Modern Features: JSON, graph, spatial, text analytics                     │
│  • Cost Model: Pay-per-use ECPUs (auto-scale up/down)                        │
│  • Performance: IM column store, parallel query, result cache                │
│  • Reliability: RAC on Exadata, <2 min RTO                                   │
│  • Compatibility: Oracle SQL standard (easier for developers)                │
│  • Integration: Native with OCI services (Object Storage, Data Science)      │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                            DATA FLOW SUMMARY (OCI)                             
================================================================================

USER QUERY PATH:
Web/Mobile → OCI Load Balancer → API Gateway → Query Orchestrator 
→ [Tundra|ADB|Snowflake-OCI] → Query Execution (MPP) → Result Aggregation 
→ OCI Cache → Response

DATA INGESTION PATH:
Source System → Connector Service (OCI Functions/Container Instances) 
→ Magic ETL (OCI Data Flow - Spark) → Data Quality → Object Storage (Durable) 
→ Hydration → Adrenaline Clusters (Tundra) or Bulk Load → ADB

AI CHAT PATH:
User Question → AI Chat Agent → Agent Planner (OCI Generative AI API or Bedrock) 
→ Tool Execution → [Vector Search | SQL Generation | Cloud Amplifier] 
→ LLM Response → User

FEDERATED QUERY PATH:
Dashboard → Query Orchestrator → Cloud Amplifier → Snowflake-OCI/ADB External Tables
→ Result Streaming → Presentation Layer → User

CROSS-CLOUD PATH (During Migration):
Domo-OCI → FastConnect → AWS Direct Connect → Domo-AWS
→ S3 (sync to Object Storage via OCI Object Storage Replication)

================================================================================

Key OCI Migration Notes:

Vertica → ADB Migration: Major architectural shift - ADB is autonomous, eliminating cluster management overhead
Snowflake Dependency: OCI migration blocked until Snowflake-OCI availability confirmed (Domo reaching out to Snowflake)
Timeline:

Certification: 2-4 weeks
Pilot: 1-2 months
Full migration: 12-18 months with double-payment period


Cost Consideration: Need to minimize double-payment period (running on both AWS and OCI simultaneously)
Tundra Remains: 8x faster than Snowflake, will be optimized for OCI but not replaced
Network Performance: OCI's 100Gbps backbone will accelerate hydration (3-12 min vs 5-15 min on AWS)
ADB Advantages: Autonomous operations, Oracle SQL features (JSON, Spatial, ML), better integration with OCI native services

This architecture represents the target state post-migration to OCI with ADB replacing Vertica as the key differentiator.