================================================================================
                    DOMO ANALYTICS CONTROL PLANE ON AWS
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
│  │  AWS Application Load Balancer (ALB)                                   │ │
│  │  • SSL/TLS Termination                                                 │ │
│  │  • Path-based routing                                                  │ │
│  │  • Health checks                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Domo API Gateway Service                                              │ │
│  │  • Authentication/Authorization (OAuth 2.0, SAML, SSO)                 │ │
│  │  • Rate limiting & throttling                                          │ │
│  │  • API versioning                                                      │ │
│  │  • Request routing & transformation                                    │ │
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
│ │  • Data dictionary               │ │  │ └──────────┬─────────────────────┘│
│ │  Store: AWS RDS (PostgreSQL)     │ │  │           │                       │
│ └──────────────────────────────────┘ │  │ ┌─────────▼──────────────────────┐│
│                                      │  │ │  Agent Planner                  ││
│ ┌──────────────────────────────────┐ │  │ │  • Task orchestration          ││
│ │  Query Orchestrator              │ │  │ │  • Tool selection              ││
│ │  • Query parsing & optimization  │ │  │ │  • Amazon Bedrock API calls    ││
│ │  • Workload management           │ │  │ │  • Execution planning          ││
│ │  • Resource allocation           │ │  │ └──────────┬─────────────────────┘│
│ │  • Query routing (Tundra/Vertica)│ │  │           │                       │
│ │  • Cost-based optimizer          │ │  │ ┌─────────▼──────────────────────┐│
│ └─────────┬────────────────────────┘ │  │ │  Tool Execution Layer          ││
│           │                          │  │ │  • Vector search (embeddings)  ││
│ ┌─────────▼────────────────────────┐ │  │ │  • SQL generation              ││
│ │  Cluster Manager                 │ │  │ │  • Domo Cloud Amplifier        ││
│ │  • Adrenaline cluster lifecycle  │ │  │ │  • Knowledge base retrieval    ││
│ │  • Auto-scaling policies         │ │  │ └────────────────────────────────┘│
│ │  • Health monitoring             │ │  │           │                       │
│ │  • Hydration orchestration       │ │  │ ┌─────────▼──────────────────────┐│
│ │  • Failover management           │ │  │ │  Model Management              ││
│ │  AWS: EC2 Auto Scaling Groups    │ │  │ │  • FM selection & routing      ││
│ └──────────────────────────────────┘ │  │ │  • Performance tracking        ││
│                                      │  │ │  • Prompt engineering          ││
│ ┌──────────────────────────────────┐ │  │ │  Backend: Amazon Bedrock       ││
│ │  Security & Governance Service   │ │  │ │  - Anthropic Claude            ││
│ │  • Access control (IAM)          │ │  │ │  - Meta Llama                  ││
│ │  • Data encryption mgmt          │ │  │ │  - Cohere, AI21, Mistral       ││
│ │  • Audit logging                 │ │  │ └────────────────────────────────┘│
│ │  • Policy enforcement            │ │  │                                    │
│ │  AWS: IAM, KMS, CloudTrail       │ │  └────────────────────────────────────┘
│ └──────────────────────────────────┘ │
│                                      │
│ ┌──────────────────────────────────┐ │
│ │  Job Scheduler & Workflow Mgmt   │ │
│ │  • ETL job scheduling            │ │
│ │  • Connector refresh scheduling  │ │
│ │  • Alert & notification engine   │ │
│ │  • Workflow orchestration        │ │
│ │  AWS: Amazon EventBridge, SQS    │ │
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
│  │  • AWS: S3, Redshift, RDS, Athena, Aurora, DynamoDB, CloudWatch, IoT  │ │
│  │  • SaaS: Salesforce, Google Analytics, Marketo, etc.                  │ │
│  │  • Databases: MySQL, PostgreSQL, Oracle, SQL Server, MongoDB          │ │
│  │  • Files: CSV, Excel, JSON, Parquet, etc.                             │ │
│  │  Runtime: AWS Lambda + ECS Fargate (containerized connectors)         │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Domo Workbench (On-Prem Agent)                                        │ │
│  │  • On-premises data source connectivity                                │ │
│  │  • Secure tunneling (VPN/DirectConnect)                                │ │
│  │  • Change data capture (CDC)                                           │ │
│  │  • Data encryption in transit                                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  API & SDK Integration Layer                                           │ │
│  │  • REST API endpoints                                                  │ │
│  │  • Custom connector SDK                                                │ │
│  │  • Webhook receivers                                                   │ │
│  │  • Streaming ingestion (Kinesis integration)                           │ │
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
│  │  • SQL transforms                                                      │ │
│  │  • Upsert & partition operations                                       │ │
│  │  Runtime: Apache Spark on EMR (for complex transformations)           │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Data Science Workspace                                                │ │
│  │  • Jupyter Notebooks integration                                       │ │
│  │  • R & Python script execution                                         │ │
│  │  • ML model training & deployment                                      │ │
│  │  • Sandbox environment for experimentation                             │ │
│  │  • Code Engine for custom data processing                              │ │
│  │  Runtime: AWS SageMaker, ECS Fargate                                   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Data Quality & Validation Service                                     │ │
│  │  • Schema validation                                                   │ │
│  │  • Data profiling & statistics                                         │ │
│  │  • Anomaly detection                                                   │ │
│  │  • DQ rules engine                                                     │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└─────────────────────────────────────┬─────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                     DURABLE STORAGE LAYER (S3 BACKEND)                        │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Amazon S3 - Primary Durability Layer                                  │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Raw Data Lake (S3 Standard)                                      │ │ │
│  │  │  • Ingested data in native formats                                │ │ │
│  │  │  • Partitioned by: tenant/dataset/date                            │ │ │
│  │  │  • Compression: Parquet, ORC, Gzip                                │ │ │
│  │  │  • Encryption: SSE-S3 / SSE-KMS                                   │ │ │
│  │  │  • Lifecycle policies: Transition to Glacier after 90 days        │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                          │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Processed Data Store (S3 Standard-IA)                            │ │ │
│  │  │  • Transformed & cleansed datasets                                │ │ │
│  │  │  • Optimized columnar formats (Parquet)                           │ │ │
│  │  │  • Pre-aggregated rollup tables                                   │ │ │
│  │  │  • Source of truth for Adrenaline hydration                       │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                          │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Metadata & Config Store (S3 + RDS)                               │ │ │
│  │  │  • Dataset manifests                                              │ │ │
│  │  │  • Query result caches                                            │ │ │
│  │  │  • Checkpoint data                                                │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Write-Back Integration                                                 │ │
│  │  • S3 bucket write operations (for data exports)                       │ │
│  │  • Redshift write-back connector                                       │ │
│  │  • Aurora write-back connector                                         │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└─────────────────────────────────────┬─────────────────────────────────────────┘
                                      │
                            ┌─────────┴─────────┐
                            │  HYDRATION LAYER  │
                            │  (S3 → Adrenaline) │
                            └─────────┬─────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                   ADRENALINE QUERY ENGINE LAYER (Analytics)                   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  ADRENALINE - Hybrid Query Engine Architecture                         │ │
│  │  (Non-Durable, Hydrated from S3, Multi-Tenant)                         │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │      TUNDRA CLUSTERS            │  │     VERTICA CLUSTERS            │  │
│  │   (Custom-Built Engine)         │  │   (Commercial RDBMS)            │  │
│  ├─────────────────────────────────┤  ├─────────────────────────────────┤  │
│  │                                 │  │                                 │  │
│  │ ┌─────────────────────────────┐ │  │ ┌─────────────────────────────┐ │  │
│  │ │  Tundra Cluster A (Tenant1) │ │  │ │ Vertica Cluster A (TenantX) │ │  │
│  │ │  • In-memory columnar store │ │  │ │ • Columnar compressed store │ │  │
│  │ │  • MPP query execution      │ │  │ │ • MPP query execution       │ │  │
│  │ │  • 8x faster than Snowflake │ │  │ │ • Proven enterprise DBMS    │ │  │
│  │ │  • Optimized for analytics  │ │  │ │ • Complex query support     │ │  │
│  │ │  • Query result caching     │ │  │ │ • Transaction support       │ │  │
│  │ │                             │ │  │ │                             │ │  │
│  │ │  Instance: EC2 r6i/r7i      │ │  │ │  Instance: EC2 i3/i4i       │ │  │
│  │ │  • Memory-optimized         │ │  │ │  • SSD local attached       │ │  │
│  │ │  • EBS gp3 for temp storage │ │  │ │  • NVMe local storage       │ │  │
│  │ │  • Enhanced networking      │ │  │ │  • High IOPS optimization   │ │  │
│  │ └─────────────────────────────┘ │  │ └─────────────────────────────┘ │  │
│  │           ▲                     │  │           ▲                     │  │
│  │           │ Hydrate from S3     │  │           │ Hydrate from S3     │  │
│  │           │ (on-demand/         │  │           │ (on-demand/         │  │
│  │           │  scheduled)         │  │           │  scheduled)         │  │
│  │           │                     │  │           │                     │  │
│  │ ┌─────────┴───────────────────┐ │  │ ┌─────────┴───────────────────┐ │  │
│  │ │  Tundra Cluster B (Tenant2) │ │  │ │ Vertica Cluster B (TenantY) │ │  │
│  │ │  (Auto-scaled replicas)     │ │  │ │ (Auto-scaled replicas)      │ │  │
│  │ └─────────────────────────────┘ │  │ └─────────────────────────────┘ │  │
│  │           ▲                     │  │           ▲                     │  │
│  │           │                     │  │           │                     │  │
│  │ ┌─────────┴───────────────────┐ │  │ ┌─────────┴───────────────────┐ │  │
│  │ │  Tundra Cluster C (Tenant3) │ │  │ │ Vertica Cluster C (TenantZ) │ │  │
│  │ └─────────────────────────────┘ │  │ └─────────────────────────────┘ │  │
│  │                                 │  │                                 │  │
│  │ Routing Logic:                  │  │ Routing Logic:                  │  │
│  │ • Small datasets (<100GB)       │  │ • Large datasets (>500GB)       │  │
│  │ • High query frequency          │  │ • Complex queries               │  │
│  │ • Real-time dashboards          │  │ • Historical analysis           │  │
│  │ • Low latency requirements      │  │ • Batch workloads               │  │
│  └─────────────────────────────────┘  └─────────────────────────────────┘  │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Query Routing Intelligence (Part of Query Orchestrator)               │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Decision Engine - Selects Tundra vs Vertica based on:           │ │ │
│  │  │  • Dataset size & characteristics                                │ │ │
│  │  │  • Query complexity & patterns                                   │ │ │
│  │  │  • Tenant tier & SLA                                             │ │ │
│  │  │  • Current cluster load & availability                           │ │ │
│  │  │  • Cost optimization policies                                    │ │ │
│  │  │  • Historical query performance metrics                          │ │ │
│  │  │  NOTE: Customer never knows which engine is used                 │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Cluster Lifecycle Management                                          │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Hydration Process (S3 → Adrenaline Clusters):                   │ │ │
│  │  │  1. Cluster Launch (EC2 Auto Scaling)                            │ │ │
│  │  │  2. S3 data fetch (parallel streams, multi-part downloads)       │ │ │
│  │  │  3. Data loading into cluster memory/local storage               │ │ │
│  │  │  4. Index building & statistics computation                      │ │ │
│  │  │  5. Cluster warm-up queries                                      │ │ │
│  │  │  6. Ready for production queries (5-15 minutes typical)          │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                          │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Cluster Failure & Recovery:                                      │ │ │
│  │  │  1. Health check failure detected (CloudWatch alarms)            │ │ │
│  │  │  2. Traffic redirected to healthy cluster nodes                  │ │ │
│  │  │  3. Failed cluster terminated (no data loss - S3 is source)      │ │ │
│  │  │  4. New cluster launched automatically                           │ │ │
│  │  │  5. Re-hydration from S3 (leveraging EBS snapshots if available) │ │ │
│  │  │  6. Cluster back online (RTO: 10-20 minutes)                     │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Query Execution Path:                                                 │ │
│  │  1. Query arrives at Adrenaline layer                                  │ │
│  │  2. Query parsed & optimized by routing engine                         │ │
│  │  3. Execution plan generated (MPP aware)                               │ │
│  │  4. Query distributed across cluster nodes                             │ │
│  │  5. Parallel execution on data shards                                  │ │
│  │  6. Results aggregated & cached (ElastiCache Redis)                    │ │
│  │  7. Results returned to application layer                              │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    EXTERNAL DATA WAREHOUSE INTEGRATION                        │
│                       (Domo Cloud Amplifier / Federated)                      │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Snowflake Integration (Growing Backend Partnership)                   │ │
│  │  • Federated query pushdown (query-in-place, no data movement)         │ │
│  │  • Native connector with optimized drivers                             │ │
│  │  • Customer data remains in Snowflake                                  │ │
│  │  • Domo queries via Snowflake Connector API                            │ │
│  │  • Future: More backend workloads moving to Snowflake                  │ │
│  │  • NOTE: Tundra remains 8x faster, won't be replaced by Snowflake     │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Other Cloud Data Warehouse Adapters                                   │ │
│  │  • Amazon Redshift (direct JDBC, optimized queries)                    │ │
│  │  • Google BigQuery (federated adapter, cross-cloud)                    │ │
│  │  • Databricks (Delta Lake connector)                                   │ │
│  │  • Amazon Athena (S3 query-in-place)                                   │ │
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
│  │  Amazon ElastiCache (Redis)                                            │ │
│  │  • Query result caching (TTL-based invalidation)                       │ │
│  │  • Session state management                                            │ │
│  │  • Dashboard metadata caching                                          │ │
│  │  • Real-time aggregation cache                                         │ │
│  │  Cluster: Multi-AZ, cluster mode enabled                               │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  CloudFront CDN                                                         │ │
│  │  • Static asset delivery (JS, CSS, images)                             │ │
│  │  • Dashboard thumbnail caching                                         │ │
│  │  • Global edge locations for low latency                               │ │
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
│  │  Domo Analyzer (Visualization Engine)                                  │ │
│  │  • 150+ chart types (proprietary viz engine)                           │ │
│  │  • Real-time dashboard rendering                                       │ │
│  │  • Drill-down & filtering                                              │ │
│  │  • Trellis/small multiples support                                     │ │
│  │  • Custom maps & geospatial viz                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Beast Mode (Calculated Fields Engine)                                 │ │
│  │  • User-defined metrics & calculations                                 │ │
│  │  • SQL-like expressions                                                │ │
│  │  • Aggregation functions                                               │ │
│  │  • Computed at query time                                              │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Domo Everywhere (Embedded Analytics)                                  │ │
│  │  • White-label embedding                                               │ │
│  │  • iFrame & JavaScript SDK                                             │ │
│  │  • Programmable filters                                                │ │
│  │  • SSO pass-through authentication                                     │ │
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
│  │  Domo App Dev Framework                                                │ │
│  │  • Custom app builder (HTML/CSS/JS)                                    │ │
│  │  • Domo.js SDK & APIs                                                  │ │
│  │  • AppDB (per-app database, NoSQL)                                     │ │
│  │  • Files API (media storage, S3-backed)                                │ │
│  │  • Serverless runtime (Lambda + ECS Fargate)                           │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Domo Workflows (Automation Engine)                                    │ │
│  │  • Visual workflow designer                                            │ │
│  │  • Event-driven triggers (data changes, schedules, webhooks)           │ │
│  │  • Action connectors (email, Slack, Jira, etc.)                        │ │
│  │  • Integration with external APIs                                      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                         MONITORING & OPERATIONS LAYER                          
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  AWS CloudWatch                                                               │
│  • Metrics: Query latency, cluster health, API response times                │
│  • Logs: Application logs, query logs, audit trails                          │
│  • Alarms: Auto-scaling triggers, failure notifications                      │
│  • Dashboards: Real-time operational metrics                                 │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  AWS X-Ray & CloudTrail                                                       │
│  • Distributed tracing across microservices                                   │
│  • API call auditing & compliance logging                                     │
│  • Security event monitoring                                                  │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  Custom Domo Operations Platform                                              │
│  • Tenant health monitoring dashboards                                        │
│  • Cluster utilization & optimization                                         │
│  • Cost tracking & chargeback                                                 │
│  • Performance benchmarking (Tundra vs Vertica vs Snowflake)                 │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                    NETWORK & CONNECTIVITY ARCHITECTURE                         
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  Multi-AZ VPC Architecture                                                    │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Public Subnets (Multi-AZ)                                             │ │
│  │  • ALB/NLB endpoints                                                   │ │
│  │  • NAT Gateways                                                        │ │
│  │  • Bastion hosts                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Private Subnets - Application Tier (Multi-AZ)                         │ │
│  │  • API Gateway services                                                │ │
│  │  • Control plane microservices (ECS/EKS)                               │ │
│  │  • ETL processing nodes                                                │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Private Subnets - Data Tier (Multi-AZ)                                │ │
│  │  • Tundra clusters                                                     │ │
│  │  • Vertica clusters                                                    │ │
│  │  • RDS instances (metadata)                                            │ │
│  │  • ElastiCache clusters                                                │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  VPC Endpoints & PrivateLink                                           │ │
│  │  • S3 Gateway Endpoint (no internet egress for S3)                     │ │
│  │  • Bedrock Interface Endpoint (AI services)                            │ │
│  │  • SageMaker Interface Endpoint                                        │ │
│  │  • Redshift Interface Endpoint                                         │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Connectivity Options                                                  │ │
│  │  • AWS Direct Connect (customer VPC peering)                           │ │
│  │  • VPN tunnels (Workbench agent connectivity)                          │ │
│  │  • Transit Gateway (multi-account, multi-region routing)               │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                    KEY ARCHITECTURAL CHARACTERISTICS                           
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  DURABILITY MODEL:                                                            │
│  • S3 is single source of truth (99.999999999% durability)                   │
│  • Adrenaline clusters are EPHEMERAL & NON-DURABLE                           │
│  • Cluster failures = Terminate & Re-hydrate from S3                         │
│  • RTO: 10-20 minutes | RPO: 0 (no data loss, S3 has everything)            │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  QUERY ENGINE SELECTION (Transparent to Customer):                           │
│  • Tundra: 8x faster than Snowflake, built from ground up                   │
│  • Vertica: Legacy workloads, complex queries, being phased out              │
│  • Snowflake: Backend partnership growing, federated access                  │
│  • Tundra will NOT be replaced by Snowflake                                  │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  MULTI-TENANCY:                                                               │
│  • Logical isolation (shared infrastructure, separate clusters)              │
│  • Tenant-specific Adrenaline clusters                                       │
│  • Dedicated S3 prefixes per tenant                                          │
│  • IAM role-based tenant isolation                                           │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  HYDRATION STRATEGY:                                                          │
│  • On-demand: New cluster launch, dataset refresh                            │
│  • Scheduled: Nightly full refresh, incremental during day                   │
│  • Triggered: Data pipeline completion, ETL job finish                       │
│  • Optimized: Parallel S3 downloads, compression-aware transfers             │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  FUTURE ARCHITECTURE (Oracle Cloud Infrastructure Migration):                │
│  • Evaluating OCI for new infrastructure deployments                         │
│  • Concern: Double-payment period during AWS→OCI migration                   │
│  • Dependency: Snowflake availability on OCI                                 │
│  • Potential: Replace Vertica with ADB (Autonomous Database) on OCI          │
│  • Timeline: Couple weeks for certification, then gradual migration          │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                            DATA FLOW SUMMARY                                   
================================================================================

USER QUERY PATH:
Web/Mobile → ALB → API Gateway → Query Orchestrator → [Tundra|Vertica] 
→ Query Execution (MPP) → Result Aggregation → ElastiCache → Response

DATA INGESTION PATH:
Source System → Connector Service (Lambda/ECS) → Magic ETL (Spark/EMR) 
→ Data Quality → S3 (Durable Storage) → Hydration → Adrenaline Clusters

AI CHAT PATH:
User Question → AI Chat Agent → Agent Planner (Bedrock API) → Tool Execution
→ [Vector Search | SQL Generation | Cloud Amplifier] → LLM Response → User

FEDERATED QUERY PATH:
Dashboard → Query Orchestrator → Cloud Amplifier → Snowflake/Redshift
→ Result Streaming → Presentation Layer → User

================================================================================




================================================================================
            DOMO ANALYTICS CONTROL PLANE - AWS CURRENT STATE
                    (Based on Domo provided Billing Data)
================================================================================

                                    ┌─────────────────────────────────────┐
                                    │      AWS REGIONS (Multi-Region)     │
                                    │   Primary: US-EAST-1 (Virginia)     │
                                    │   Secondary: AP-NORTHEAST-1 (Tokyo) │
                                    └─────────────────────────────────────┘
                                                    │
                    ┌───────────────────────────────┴───────────────────────────────┐
                    │                                                               │
        ┌───────────▼──────────┐                                      ┌────────────▼─────────┐
        │   TUNDRA COMPUTE     │                                      │    VERTICA COMPUTE   │
        │   (Analytics Engine) │                                      │  (Complex Queries)   │
        └──────────────────────┘                                      └──────────────────────┘
                    │                                                               │
    ┌───────────────┴───────────────┐                         ┌────────────────────┴────────┐
    │                               │                         │                             │
    │  EC2 INSTANCES (BY TYPE):     │                         │  RDS INSTANCES (BY TYPE):   │
    │                               │                         │                             │
    │  Memory-Optimized (r-series): │                         │  Memory-Optimized (r):      │
    │  • r7gd.medium: 1,171 inst    │                         │  • db.r6g.2xlarge: 135 inst │
    │    - 9,370 vCPUs (ARM)        │                         │    - 1,080 vCPU, 8,640 GB   │
    │    - 69.5 TB local NVMe       │                         │  • db.r6g.xlarge: 85 inst   │
    │  • r6gd.medium: 1,039 inst    │                         │    - 340 vCPU, 2,720 GB     │
    │    - 8,315 vCPUs (ARM)        │                         │  • db.r7g.xlarge: 49 inst   │
    │    - 61.7 TB local NVMe       │                         │    - 196 vCPU, 1,568 GB     │
    │  • r6a.large: 235 inst        │                         │  • db.r7g.2xlarge: 47 inst  │
    │    - 1,884 vCPUs (AMD)        │                         │    - 376 vCPU, 3,008 GB     │
    │  • r7a.medium: 1,290 inst     │                         │  • db.r7g.4xlarge: 10 inst  │
    │    - 10,320 vCPUs (AMD)       │                         │    - 160 vCPU, 1,280 GB     │
    │  • r6i.large: 1,405 inst      │                         │  • db.r6g.8xlarge: 10 inst  │
    │    - 11,245 vCPUs (Intel)     │                         │    - 20 vCPU, 160 GB        │
    │                               │                         │  • db.t4g.small: 3 inst     │
    │  Compute-Optimized (c):       │                         │  • db.m6g.xlarge: 1 inst    │
    │  • c6gd.medium: 3,742 inst    │                         │                             │
    │    - 7,484 vCPUs (ARM)        │                         │  General Purpose (m/t):     │
    │  • c7gd.medium: 822 inst      │                         │  • db.m5.xlarge: 9 inst     │
    │    - 1,645 vCPUs (ARM)        │                         │  • db.t3.large: 53 inst     │
    │  • c6i.large: 94 inst         │                         │  • db.t3.micro: 17 inst     │
    │    - 188 vCPUs (Intel)        │                         │  • db.t3.medium: 11 inst    │
    │                               │                         │  • db.t3.small: 11 inst     │
    │  General Purpose (m):         │                         │  • db.m5.4xlarge: 1 inst    │
    │  • m6gd.medium: 5,254 inst    │                         │  • db.r5.large: 2 inst      │
    │    - 21,016 vCPUs (ARM)       │                         │  • db.r5.2xlarge: 1 inst    │
    │  • m6a.large: 56 inst         │                         │                             │
    │    - 224 vCPUs (AMD)          │                         │  ──────────────────────────  │
    │  • m7a.medium: 6 inst         │                         │  TOTAL RDS:                 │
    │    - 24 vCPUs (AMD)           │                         │  • 2,606 instances          │
    │  • m5.large: 1,028 inst       │                         │  • 19,361 GB total memory   │
    │    - 4,115 vCPUs (Intel)      │                         │  • Multi-AZ: Yes            │
    │                               │                         │  • ARM: 93% (Graviton2/3)   │
    │  Storage-Optimized (i):       │                         │  • AMD: 4%                  │
    │  • i4g.large: 100 inst        │                         │  • Intel: 3%                │
    │    - 400 vCPUs (ARM)          │                         └─────────────────────────────┘
    │    - 468.75 TB local NVMe     │
    │  • i3en.large: 82 inst        │
    │    - 512.5 TB local NVMe      │
    │                               │
    │  ──────────────────────────   │
    │  TOTAL EC2:                   │
    │  • ~18,000+ instances         │
    │  • ~85,000+ vCPUs             │
    │  • 80% ARM (Graviton2/3)      │
    │  • 15% AMD (EPYC)             │
    │  • 5% Intel (Xeon)            │
    └───────────────────────────────┘
                    │
                    ├─────────────────────────────┐
                    │                             │
        ┌───────────▼──────────┐      ┌──────────▼─────────────┐
        │   ELASTIC LOAD       │      │   AUTO SCALING GROUPS  │
        │   BALANCING (ALB)    │      │                        │
        │                      │      │  • Tundra Clusters     │
        │  • Application LBs   │      │  • Target: 70% CPU     │
        │  • 100+ Load Balancers│     │  • Scale: 1-50 nodes   │
        │  • SSL Termination   │      │  • Health checks: 30s  │
        │  • Path-based routing│      │  • Cooldown: 5 min     │
        └──────────────────────┘      └────────────────────────┘
                    │
                    │
        ┌───────────▼───────────────────────────────────────────────────┐
        │                   AMAZON S3 (PRIMARY DATA LAYER)              │
        │                                                               │
        │  BLOB STORAGE BREAKDOWN:                                      │
        │  ┌────────────────────────────────────────────────────────┐  │
        │  │  Standard (Hot Data):                                  │  │
        │  │  • 10.22 PB (10,718,009 GB)                           │  │
        │  │  • Active query data, recent ETL outputs              │  │
        │  │  • Accessed daily by Tundra clusters                  │  │
        │  └────────────────────────────────────────────────────────┘  │
        │                                                               │
        │  ┌────────────────────────────────────────────────────────┐  │
        │  │  Infrequent Access (Warm Data):                        │  │
        │  │  • 26.93 PB (28,242,519 GB)                           │  │
        │  │  • Historical data, accessed weekly/monthly            │  │
        │  │  • Auto-transitioned after 30 days                     │  │
        │  └────────────────────────────────────────────────────────┘  │
        │                                                               │
        │  ┌────────────────────────────────────────────────────────┐  │
        │  │  Archive (Cold Data):                                  │  │
        │  │  • 0.26 PB (270,549 GB)                               │  │
        │  │  • Compliance archives, rarely accessed                │  │
        │  │  • Retrieval time: 1-5 hours                          │  │
        │  └────────────────────────────────────────────────────────┘  │
        │                                                               │
        │  ═══════════════════════════════════════════════════════════  │
        │  TOTAL S3 STORAGE: 37.41 PB (38,311,596 GB)                  │
        │                                                               │
        │  S3 API OPERATIONS (DAILY):                                   │
        │  • GET requests: 332,447,939 (~3.85K/sec)                    │
        │  • PUT requests: 246,653,666 (~2.85K/sec)                    │
        │  • Metadata operations: 43,670,125                            │
        │  • DELETE operations: 6,510                                   │
        │  • Lifecycle transitions: 4,348,014                           │
        │                                                               │
        │  S3 FEATURES ENABLED:                                         │
        │  • Versioning: Enabled (30 days retention)                   │
        │  • Encryption: SSE-S3 (AES-256)                              │
        │  • Access Logging: Enabled → CloudWatch Logs                 │
        │  • Lifecycle Policies: 3 policies (Standard→IA→Archive)      │
        │  • Cross-Region Replication: Tokyo (disaster recovery)        │
        │  • Intelligent Tiering: Enabled on select buckets            │
        └───────────────────────────────────────────────────────────────┘
                    │
        ┌───────────▼──────────┐
        │   AMAZON EBS         │
        │   (Attached Storage) │
        │                      │
        │  • 524.38 TB total   │
        │  • 33 volumes        │
        │  • ~16 TB per volume │
        │  • Type: gp3 (SSD)   │
        │  • IOPS: 16,000      │
        │  • Throughput: 1000MB│
        │  • Snapshots: Daily  │
        └──────────────────────┘

┌───────────────────────────────────────────────────────────────────────────┐
│                          NETWORKING LAYER                                 │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  VPC (Virtual Private Cloud)                                    │    │
│  │  • CIDR: 10.0.0.0/8 (16M IPs)                                  │    │
│  │  • 6 Availability Zones across US-EAST-1                       │    │
│  │  • Private subnets: 10.0.0.0/16 - 10.5.0.0/16                 │    │
│  │  • Public subnets: 10.100.0.0/16 (ALBs only)                  │    │
│  │  • Database subnets: 10.200.0.0/16 (RDS, isolated)            │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  DATA TRANSFER (MONTHLY):                                       │    │
│  │  ┌──────────────────────────────────────────────────────────┐  │    │
│  │  │  Regional (Internal): 492,185 GB                         │  │    │
│  │  │  • VPC-to-VPC within region (FREE)                       │  │    │
│  │  │  • EC2 ↔ RDS: ~200 TB                                   │  │    │
│  │  │  • EC2 ↔ S3: ~292 TB                                    │  │    │
│  │  └──────────────────────────────────────────────────────────┘  │    │
│  │                                                                 │    │
│  │  ┌──────────────────────────────────────────────────────────┐  │    │
│  │  │  Internet Inbound: 166,706 GB (FREE)                     │  │    │
│  │  │  • Customer API calls to ALBs                            │  │    │
│  │  │  • Data uploads from customers                           │  │    │
│  │  └──────────────────────────────────────────────────────────┘  │    │
│  │                                                                 │    │
│  │  ┌──────────────────────────────────────────────────────────┐  │    │
│  │  │  Internet Outbound: 41,463 GB                            │  │    │
│  │  │  • US-EAST: 41,463 GB × $0.09/GB = $3,575/month         │  │    │
│  │  │  • TOKYO: Similar egress costs                           │  │    │
│  │  │  • Dashboard responses, API results to customers         │  │    │
│  │  └──────────────────────────────────────────────────────────┘  │    │
│  │                                                                 │    │
│  │  ┌──────────────────────────────────────────────────────────┐  │    │
│  │  │  Direct Connect:                                         │  │    │
│  │  │  • Inbound: 161 GB (customer private connections)        │  │    │
│  │  │  • Outbound: 24 GB                                       │  │    │
│  │  │  • 10 Gbps dedicated link                                │  │    │
│  │  │  • Cost: $6,570/month per region                         │  │    │
│  │  └──────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  NAT GATEWAYS:                                                  │    │
│  │  • 6 NAT Gateways (1 per AZ)                                   │    │
│  │  • $0.045/hour + $0.045/GB processed                           │    │
│  │  • ~500 TB/month through NAT                                   │    │
│  │  • Cost: ~$23K/month                                           │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  SECURITY GROUPS & NACLs:                                       │    │
│  │  • 500+ Security Groups                                        │    │
│  │  • Rules: Allow 443 (HTTPS), 5432 (PostgreSQL), 3306 (MySQL)  │    │
│  │  • Deny all inbound by default                                 │    │
│  │  • Stateful filtering                                          │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────────┐
│                        SUPPORTING AWS SERVICES                            │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────────┐     │
│  │  CLOUDWATCH      │  │  CLOUDTRAIL      │  │  AWS KMS          │     │
│  │                  │  │                  │  │                   │     │
│  │  • 100K+ metrics │  │  • Audit logs    │  │  • Encryption keys│     │
│  │  • 10K+ alarms   │  │  • 90 day retain │  │  • S3 encryption  │     │
│  │  • Log groups:   │  │  • Compliance    │  │  • EBS encryption │     │
│  │    - Tundra logs │  │                  │  │  • RDS encryption │     │
│  │    - RDS logs    │  │                  │  │                   │     │
│  │    - VPC logs    │  │                  │  │                   │     │
│  │  • Retention: 30d│  │                  │  │                   │     │
│  └──────────────────┘  └──────────────────┘  └───────────────────┘     │
│                                                                           │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────────┐     │
│  │  ROUTE 53        │  │  AWS IAM         │  │  SECRETS MANAGER  │     │
│  │                  │  │                  │  │                   │     │
│  │  • DNS hosting   │  │  • 500+ roles    │  │  • DB credentials │     │
│  │  • Health checks │  │  • 2000+ policies│  │  • API keys       │     │
│  │  • Failover      │  │  • MFA enforced  │  │  • Rotation: 90d  │     │
│  │  • Latency-based │  │  • SAML SSO      │  │  • Encrypted: KMS │     │
│  └──────────────────┘  └──────────────────┘  └───────────────────┘     │
│                                                                           │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────────┐     │
│  │  ELASTICACHE     │  │  KINESIS         │  │  SQS              │     │
│  │  (Redis)         │  │                  │  │                   │     │
│  │                  │  │  • Data streams  │  │  • Message queues │     │
│  │  • Cache tier    │  │  • ETL pipelines │  │  • Async jobs     │     │
│  │  • 50+ clusters  │  │  • Real-time     │  │  • 100M+ msgs/day │     │
│  │  • r6g.large     │  │    analytics     │  │  • FIFO queues    │     │
│  │  • 6 shards/node │  │  • 500 shards    │  │                   │     │
│  └──────────────────┘  └──────────────────┘  └───────────────────┘     │
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────┐       │
│  │  AWS LAMBDA (Serverless):                                    │       │
│  │  • ETL processing: 10,000+ functions                         │       │
│  │  • API Gateway backends                                       │       │
│  │  • S3 event triggers (file uploads → processing)             │       │
│  │  • CloudWatch event processing                                │       │
│  │  • Cost: ~$5K/month (1B invocations)                         │       │
│  └──────────────────────────────────────────────────────────────┘       │
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────┐       │
│  │  ELASTIC KUBERNETES SERVICE (EKS) - Optional/Partial:        │       │
│  │  • Used for: Microservices, API layer                        │       │
│  │  • Nodes: m6g.xlarge (ARM Graviton2)                         │       │
│  │  • Pods: ~5,000 running                                       │       │
│  │  • CNI: AWS VPC CNI plugin                                    │       │
│  │  • Load Balancer: AWS Load Balancer Controller               │       │
│  └──────────────────────────────────────────────────────────────┘       │
└───────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────────┐
│                         COST BREAKDOWN SUMMARY                            │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  MONTHLY AWS COSTS (APPROXIMATE):                                         │
│  ┌────────────────────────────────────────────────────────────┐          │
│  │  EC2 Compute:                          $2,309,266/month    │          │
│  │  • ~18,000 instances (mostly Graviton ARM)                │          │
│  │  • Mix of on-demand + reserved instances (70% reserved)   │          │
│  └────────────────────────────────────────────────────────────┘          │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────┐          │
│  │  RDS Database:                         $243,403/month      │          │
│  │  • 2,606 RDS instances                                     │          │
│  │  • Multi-AZ deployments                                    │          │
│  │  • Automated backups (7 days)                             │          │
│  └────────────────────────────────────────────────────────────┘          │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────┐          │
│  │  S3 Storage:                           $590,226/month      │          │
│  │  • 37.41 PB total                                          │          │
│  │  • Standard: $10.22PB × $23/TB = $235K                    │          │
│  │  • Infrequent: $26.93PB × $12.5/TB = $336K               │          │
│  │  • Archive: $0.26PB × $4/TB = $1K                         │          │
│  │  • API requests: $18K/month                                │          │
│  └────────────────────────────────────────────────────────────┘          │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────┐          │
│  │  EBS Storage:                          $43,253/month       │          │
│  │  • 524.38 TB × $80/TB                                      │          │
│  └────────────────────────────────────────────────────────────┘          │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────┐          │
│  │  Data Transfer:                        $19,990/month       │          │
│  │  • Internet egress: 41 TB × $0.09 = $3.6K (US-East)      │          │
│  │  • Direct Connect: $13K (2 regions)                        │          │
│  │  • Inter-region: $3.4K                                     │          │
│  └────────────────────────────────────────────────────────────┘          │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────┐          │
│  │  Other Services:                       $94,000/month       │          │
│  │  • Load Balancers: $3K                                     │          │
│  │  • NAT Gateways: $23K                                      │          │
│  │  • ElastiCache: $15K                                       │          │
│  │  • CloudWatch: $8K                                         │          │
│  │  • Lambda: $5K                                             │          │
│  │  • EKS: $10K                                               │          │
│  │  • Route53, KMS, Secrets Manager, etc: $30K              │          │
│  └────────────────────────────────────────────────────────────┘          │
│                                                                           │
│  ═══════════════════════════════════════════════════════════════          │
│  TOTAL MONTHLY AWS COST: ~$3,300,000                                      │
│  ANNUAL AWS COST: ~$39,600,000                                            │
│  ═══════════════════════════════════════════════════════════════          │
│                                                                           │
│  OCI ESTIMATED MONTHLY COST: ~$1,190,000 (64% SAVINGS)                   │
│  ANNUAL OCI COST: ~$14,280,000                                            │
│  ANNUAL SAVINGS: ~$25,320,000                                             │
│                                                                           │
│  PRIMARY SAVINGS DRIVERS:                                                 │
│  • Compute: 40-50% cheaper (OCI Flex shapes)                             │
│  • Egress: FREE first 10TB/month (vs $0.09/GB AWS)                      │
│  • Storage: 30% cheaper with auto-tiering                                │
│  • No NAT Gateway costs (OCI Service Gateway is free)                    │
└───────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────────┐
│                        MIGRATION PRIORITIES                               │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  PHASE 1 (Months 1-3): TUNDRA MIGRATION                                  │
│  ─────────────────────────────────────────────────────────────────        │
│  Target: ~18,000 EC2 instances → OCI Compute                             │
│  • r7gd/r6gd → OCI VM.Standard.E5.Flex (ARM-based)                       │
│  • m6gd → OCI VM.Optimized3.Flex                                          │
│  • c6gd → OCI VM.Standard.A1.Flex                                         │
│  • Intel instances → OCI VM.Standard3.Flex                                │
│  • S3 37PB → OCI Object Storage (parallel sync)                          │
│  • Keep Vertica on AWS (Phase 2)                                         │
│  Expected Savings: $1.8M/month                                            │
│                                                                           │
│  PHASE 2 (Months 4-6): DATABASE MIGRATION                                │
│  ─────────────────────────────────────────────────────────────────────    │
│  Target: 2,606 RDS instances → OCI Autonomous Database or Base DB        │
│  • Evaluate: Vertica on OCI vs migrate to ADB                           │
│  • PostgreSQL RDS → OCI Base Database Service                            │
│  • Test data sync, validate query compatibility                          │
│  Expected Additional Savings: $300K/month                                 │
│                                                                           │
│  DECOMMISSION AWS (Month 7): Final cutover, validate, terminate AWS      │
└───────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────────┐
│                            KEY INSIGHTS                                   │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  1. GRAVITON DOMINANCE: 80% of compute on ARM (AWS Graviton2/3)          │
│     → Perfect fit for OCI Ampere (ARM) instances                         │
│     → Drop-in replacement, similar performance                            │
│                                                                           │
│  2. MASSIVE STORAGE: 37+ PB in S3, growing 15%/year                      │
│     → OCI auto-tiering will save 30-40% on storage costs                 │
│     → No S3 API changes needed (S3-compatible)                            │
│                                                                           │
│  3. EGRESS NIGHTMARE: 41TB/month × $0.09 = $3,575/month (US-East only)  │
│     → OCI: First 10TB FREE, then $0.0085/GB (10x cheaper)                │
│     → This alone saves $40K/month                                         │
│                                                                           │
│  4. NAT GATEWAY WASTE: $23K/month for accessing S3                        │
│     → OCI Service Gateway is FREE for Object Storage access              │
│     → Eliminate all NAT costs                                             │
│                                                                           │
│  5. INSTANCE SPRAWL: 18,000+ EC2 instances is complex to manage          │
│     → OCI Flex shapes reduce this (dynamic sizing)                       │
│     → Fewer instances, same capacity                                      │
│                                                                           │
│  6. NO KUBERNETES DEPENDENCY: Minimal EKS usage                           │
│     → Simple lift-and-shift for most workloads                           │
│     → Can evaluate OKE later if needed                                    │
└───────────────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════════════
                         ARCHITECTURE UPDATED: 2025-01-15
                    Based on Domo provided AWS billing data 
═══════════════════════════════════════════════════════════════════════════