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