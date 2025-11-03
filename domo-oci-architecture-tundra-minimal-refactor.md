================================================================================
        DOMO ANALYTICS CONTROL PLANE - RAPID OCI REPLATFORM ARCHITECTURE
                    (Tundra-Only, Minimal Changes, Maximum OCI Benefits)
                            Target: 4-8 Week Migration
================================================================================

MIGRATION PHILOSOPHY: "Lift, Shift, and Enhance"
- Keep application architecture unchanged
- Tundra clusters only (Vertica stays, defer ADB migration)
- Direct AWS→OCI service mapping
- Apply OCI differentiators where they're DROP-IN replacements
- Prioritize speed over perfect optimization

┌──────────────────────────────────────────────────────────────────────────────┐
│                          USER INTERACTION LAYER                               │
│                              [NO CHANGES]                                     │
├──────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │   Web UI    │  │  Mobile App │  │  Domo Apps  │  │ Embedded BI │       │
│  │  (React)    │  │   (Native)  │  │ (Custom JS) │  │ (iFrames)   │       │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘       │
│         └─────────────────┴─────────────────┴─────────────────┘              │
└─────────────────────────────────────────┬───────────────────────────────────┘
                                          │
                                          ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                     API GATEWAY & LOAD BALANCING LAYER                        │
│                    [DIRECT AWS→OCI MAPPING + OCI ENHANCEMENTS]               │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  AWS ALB → OCI Load Balancer (Flexible Shape)                                │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  CHANGE: Replace ALB with OCI Load Balancer                            │ │
│  │  • SSL/TLS: Use existing certs OR OCI Certificates Service             │ │
│  │  • Same path-based routing rules (config migration)                    │ │
│  │  • Same health check endpoints                                         │ │
│  │  • OCI ENHANCEMENT: Built-in WAF (enable with 1 click)                 │ │
│  │  • OCI ENHANCEMENT: DDoS protection included (no extra cost)           │ │
│  │  • OCI ENHANCEMENT: Cross-AD redundancy automatic                      │ │
│  │  EFFORT: 2-3 days (config translation)                                 │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  Domo API Gateway Service                                                    │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  MINIMAL CHANGE: Keep existing service, update endpoints               │ │
│  │  • Authentication/Authorization logic UNCHANGED                         │ │
│  │  • OAuth 2.0, SAML, SSO flows UNCHANGED                                │ │
│  │  • API versioning UNCHANGED                                            │ │
│  │  • Deployment: Keep on EC2-equivalent (OCI Compute)                    │ │
│  │  EFFORT: 1 day (endpoint configuration)                                │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└─────────────────────────────────────┬─────────────────────────────────────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    │                                   │
                    ▼                                   ▼
┌──────────────────────────────────────┐  ┌────────────────────────────────────┐
│    CONTROL PLANE SERVICES LAYER      │  │   AI SERVICE LAYER                │
│    [MINIMAL CHANGE - REHOST ONLY]    │  │   [DEFER OR KEEP ON AWS]          │
├──────────────────────────────────────┤  ├────────────────────────────────────┤
│                                      │  │                                    │
│ AWS RDS → OCI Base Database          │  │ **PHASE 1 DECISION:**             │
│ ┌──────────────────────────────────┐ │  │ Option A: Keep AI services on AWS │
│ │  Metadata Service                │ │  │ (Amazon Bedrock via FastConnect)  │
│ │  CHANGE: Minimal                 │ │  │ • Zero code changes               │
│ │  • Dump AWS RDS PostgreSQL       │ │  │ • Add FastConnect link            │
│ │  • Restore to OCI Base Database  │ │  │ • Defer migration to Phase 2      │
│ │  • Keep same schema/queries      │ │  │ EFFORT: 2 days                    │
│ │  • Connection string update only │ │  │                                    │
│ │  OCI ENHANCEMENT:                │ │  │ Option B: Migrate to OCI Gen AI   │
│ │  • Immutable backups (30 days)   │ │  │ • Replace Bedrock API calls       │
│ │  • Automatic cross-AD standby    │ │  │ • Test all AI prompts             │
│ │  EFFORT: 1-2 days                │ │  │ • Validate performance            │
│ └──────────────────────────────────┘ │  │ EFFORT: 2-3 weeks                 │
│                                      │  │                                    │
│ Query Orchestrator                   │  │ **RECOMMENDATION:**               │
│ ┌──────────────────────────────────┐ │  │ Phase 1 = Option A (keep on AWS)  │
│ │  NO CODE CHANGES                 │ │  │ Phase 2 = Migrate to OCI Gen AI   │
│ │  • Existing routing logic        │ │  └────────────────────────────────────┘
│ │  • Routes to: Tundra (OCI)       │
│ │  • Routes to: Vertica (keep AWS) │
│ │  OCI CHANGE:                     │
│ │  • Update Tundra endpoints to    │
│ │    OCI private IPs               │
│ │  • Vertica: Cross-cloud via      │
│ │    FastConnect (temporary)       │
│ │  EFFORT: 1 day                   │
│ └──────────────────────────────────┘ │
│                                      │
│ Cluster Manager                      │
│ ┌──────────────────────────────────┐ │
│ │  MINOR CHANGES                   │ │
│ │  • AWS Auto Scaling Groups →     │ │
│ │    OCI Instance Pools            │ │
│ │  • Same lifecycle logic          │ │
│ │  • Same health checks            │ │
│ │  • Same hydration triggers       │ │
│ │  OCI API calls (SDK update)      │ │
│ │  EFFORT: 3-5 days                │ │
│ └──────────────────────────────────┘ │
│                                      │
│ Security & Governance                │
│ ┌──────────────────────────────────┐ │
│ │  AWS IAM → OCI IAM               │ │
│ │  CHANGE: Policy translation      │ │
│ │  • Map IAM roles → OCI policies  │ │
│ │  • Keep same access patterns     │ │
│ │                                  │ │
│ │  🔥 OCI DIFFERENTIATOR:          │ │
│ │  INSTANCE PRINCIPALS (ZERO-KEY)  │ │
│ │  • NO AWS ACCESS KEYS needed!    │ │
│ │  • Compute→Object Storage: auth  │ │
│ │    via instance identity         │ │
│ │  • Eliminate key rotation        │ │
│ │  • Better security posture       │ │
│ │  • Code change: Remove AWS creds,│ │
│ │    use default OCI SDK auth      │ │
│ │  EFFORT: 2-3 days                │ │
│ │  VALUE: High security, low ops   │ │
│ │                                  │ │
│ │  AWS KMS → OCI Vault (HSM)       │ │
│ │  🔥 OCI DIFFERENTIATOR:          │ │
│ │  • FIPS 140-2 Level 3 HSM        │ │
│ │    (AWS KMS is Level 2)          │ │
│ │  • Better compliance (PCI-DSS)   │ │
│ │  • Bring Your Own Key (BYOK)     │ │
│ │  • Key versions immutable        │ │
│ │  EFFORT: 1-2 days (key migration)│ │
│ │                                  │ │
│ │  AWS CloudTrail → OCI Audit      │ │
│ │  🔥 OCI DIFFERENTIATOR:          │ │
│ │  • Immutable audit logs (WORM)   │ │
│ │  • Cannot be deleted/modified    │ │
│ │  • Better compliance posture     │ │
│ │  • 365 days retention (free)     │ │
│ │  EFFORT: 1 day (enable service)  │ │
│ └──────────────────────────────────┘ │
│                                      │
│ Job Scheduler                        │
│ ┌──────────────────────────────────┐ │
│ │  AWS EventBridge → OCI Events    │ │
│ │  CHANGE: Event rule migration    │ │
│ │  • Same cron expressions         │ │
│ │  • Same event patterns           │ │
│ │  • Update target ARNs → OCIDs    │ │
│ │  EFFORT: 2-3 days                │ │
│ │                                  │ │
│ │  AWS SQS → OCI Queue             │ │
│ │  CHANGE: Minimal (same API)      │ │
│ │  • Queue creation scripts        │ │
│ │  • Keep message formats          │ │
│ │  EFFORT: 1-2 days                │ │
│ └──────────────────────────────────┘ │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                      DATA INGESTION & INTEGRATION LAYER                       │
│                         [REHOST + OCI CONNECTOR ADDITIONS]                    │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Connector Service                                                            │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  AWS Lambda + ECS Fargate → OCI Functions + Container Instances        │ │
│  │                                                                          │ │
│  │  CHANGE: Containerized connectors easy to migrate                       │ │
│  │  • Docker images: Push to OCI Container Registry (OCIR)                │ │
│  │  • Lambda functions → OCI Functions (same code, diff deployment)       │ │
│  │  • ECS Fargate → OCI Container Instances (similar service)             │ │
│  │  • Keep all 1000+ connector logic UNCHANGED                            │ │
│  │  • Update endpoints to OCI Object Storage                              │ │
│  │                                                                          │ │
│  │  NEW: Add OCI-specific connectors                                       │ │
│  │  • OCI Object Storage (native)                                          │ │
│  │  • OCI Autonomous Database                                             │ │
│  │  • OCI MySQL HeatWave                                                   │ │
│  │                                                                          │ │
│  │  🔥 OCI DIFFERENTIATOR:                                                 │ │
│  │  INSTANCE PRINCIPALS for connectors                                     │ │
│  │  • Functions automatically auth to Object Storage, DB, etc.            │ │
│  │  • Zero secrets management!                                            │ │
│  │                                                                          │ │
│  │  EFFORT: 5-7 days (container migration + testing)                      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  Domo Workbench                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  NO CHANGES to agent software                                           │ │
│  │  • Workbench agents stay on customer premises                           │ │
│  │  • Update server endpoints to OCI Load Balancer                         │ │
│  │  • Tunneling: Site-to-Site VPN or FastConnect                          │ │
│  │  EFFORT: 1 day (endpoint config)                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└─────────────────────────────────────────┬─────────────────────────────────────┘
                                          │
                                          ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    DATA PROCESSING & TRANSFORMATION LAYER                     │
│                         [REHOST + MINOR ENHANCEMENTS]                         │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Magic ETL Engine                                                             │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  AWS EMR → OCI Data Flow (Apache Spark)                                │ │
│  │                                                                          │ │
│  │  CHANGE: Infrastructure only, NOT logic                                 │ │
│  │  • Same Spark jobs (PySpark, Scala)                                    │ │
│  │  • Same transformation code                                            │ │
│  │  • Update S3 paths → Object Storage paths                              │ │
│  │  • OCI Data Flow is serverless (easier than EMR!)                      │ │
│  │                                                                          │ │
│  │  🔥 OCI DIFFERENTIATOR:                                                 │ │
│  │  OCI Data Flow = TRUE SERVERLESS                                        │ │
│  │  • No cluster management (EMR requires cluster setup)                  │ │
│  │  • Pay per execution time only                                         │ │
│  │  • Auto-scaling built-in                                               │ │
│  │  • Faster startup than EMR                                             │ │
│  │                                                                          │ │
│  │  EFFORT: 3-5 days (job migration + testing)                            │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  Data Science Workspace                                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  **PHASE 1: DEFER MIGRATION**                                           │ │
│  │  • Keep Jupyter, R, Python on AWS initially                            │ │
│  │  • Cross-cloud access via FastConnect                                  │ │
│  │  • Migrate in Phase 2 when bandwidth allows                            │ │
│  │  EFFORT: 0 days (deferred)                                              │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└─────────────────────────────────────────┬─────────────────────────────────────┘
                                          │
                                          ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│              🔥🔥🔥 DURABLE STORAGE LAYER - KEY OCI WINS 🔥🔥🔥                │
│                    AWS S3 → OCI OBJECT STORAGE                                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  OCI Object Storage - THE DURABILITY LAYER                             │ │
│  │  (Direct S3 replacement with MAJOR OCI advantages)                      │ │
│  │                                                                          │ │
│  │  CHANGE: Minimal code changes (S3 SDK → OCI SDK)                        │ │
│  │  • Bucket names stay same (if available)                               │ │
│  │  • Object keys/paths stay same                                         │ │
│  │  • Same partitioning scheme                                            │ │
│  │  • SDK change: boto3 (S3) → oci.object_storage (OCI)                   │ │
│  │  • API compatible with S3 (can use S3 compatibility layer)             │ │
│  │                                                                          │ │
│  │  🔥🔥🔥 OCI DIFFERENTIATORS (IMMEDIATE VALUE): 🔥🔥🔥                    │ │
│  │                                                                          │ │
│  │  1. FREE 10TB/MONTH EGRESS (vs AWS $922/TB after 100GB)                │ │
│  │     • MASSIVE cost savings for data hydration                           │ │
│  │     • Object Storage → Tundra clusters: FREE                           │ │
│  │     • AWS charges ~$1000/TB, OCI = $0 for first 10TB                   │ │
│  │     • If Domo pulls 50TB/month: AWS = $45K, OCI = $0-4K               │ │
│  │     VALUE: $40K+/month savings immediately                              │ │
│  │                                                                          │ │
│  │  2. IMMUTABLE OBJECT STORAGE (WORM - Write Once Read Many)             │ │
│  │     • Enable retention rules on buckets                                │ │
│  │     • Objects CANNOT be deleted/modified during retention              │ │
│  │     • Perfect for compliance (SOC 2, GDPR, HIPAA)                      │ │
│  │     • Better than S3 Object Lock (easier to configure)                 │ │
│  │     CONFIG: 1-click enable on bucket                                   │ │
│  │     VALUE: Compliance without extra cost                               │ │
│  │                                                                          │ │
│  │  3. AUTO-TIERING (Free, Automatic)                                      │ │
│  │     • Standard → Infrequent Access → Archive                           │ │
│  │     • OCI auto-moves objects based on access patterns                  │ │
│  │     • No lifecycle policies needed (AWS requires config)               │ │
│  │     • Intelligent cost optimization                                    │ │
│  │     CONFIG: Enable "Auto-Tiering" on bucket                            │ │
│  │     VALUE: 30-60% storage cost reduction (automated)                   │ │
│  │                                                                          │ │
│  │  4. CROSS-REGION REPLICATION (Built-in, No Extra Cost)                 │ │
│  │     • Built into service, not add-on like AWS                          │ │
│  │     • Automatic multi-region durability                                │ │
│  │     • Geo-redundancy for DR                                            │ │
│  │     CONFIG: Enable replication policy on bucket                        │ │
│  │     VALUE: DR without paying for cross-region data transfer            │ │
│  │                                                                          │ │
│  │  5. STORAGE PERFORMANCE AUTO-TUNING                                     │ │
│  │     • OCI automatically optimizes IOPS/throughput                      │ │
│  │     • No need to choose storage classes upfront                        │ │
│  │     • Adapts to workload patterns                                      │ │
│  │     • Faster parallel downloads for hydration                          │ │
│  │     VALUE: Faster Tundra hydration (3-8 min vs 5-15 min AWS)          │ │
│  │                                                                          │ │
│  │  6. PRE-AUTHENTICATED REQUESTS (PAR) - Better than S3 Pre-Signed URLs  │ │
│  │     • More granular control                                            │ │
│  │     • Longer expiration windows                                        │ │
│  │     • Bucket-level or object-level                                     │ │
│  │     • Easier to manage than S3 pre-signed URLs                         │ │
│  │                                                                          │ │
│  │  MIGRATION STRATEGY:                                                    │ │
│  │  1. Create OCI buckets (same names as S3)                              │ │
│  │  2. Enable WORM, Auto-Tiering, Cross-Region Replication                │ │
│  │  3. Data migration options:                                            │ │
│  │     Option A: OCI Data Transfer Service (physical appliance, 100TB+)   │ │
│  │     Option B: Rclone over FastConnect (10-50TB)                        │ │
│  │     Option C: OCI Object Storage Replication (continuous sync)         │ │
│  │  4. Update SDK calls: boto3 → oci.object_storage                       │ │
│  │  5. Test hydration performance                                         │ │
│  │                                                                          │ │
│  │  EFFORT: 5-10 days (depending on data size)                            │ │
│  │  VALUE: Immediate 40-60% cost savings + better compliance              │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Bucket Structure (Same as AWS S3):                                     │ │
│  │  • domo-raw-data-{region}/                                             │ │
│  │    - tenant-1/dataset-xyz/2025/01/15/data.parquet                      │ │
│  │    - tenant-2/dataset-abc/2025/01/15/data.parquet                      │ │
│  │  • domo-processed-data-{region}/                                        │ │
│  │    - tenant-1/dataset-xyz-transformed/2025/01/15/data.parquet          │ │
│  │  • domo-metadata-{region}/                                              │ │
│  │    - manifests/dataset-xyz.json                                        │ │
│  │  • domo-archive-{region}/                                               │ │
│  │    - Historical data > 1 year                                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└─────────────────────────────────────────┬─────────────────────────────────────┘
                                          │
                                          ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│            🔥 ADRENALINE - TUNDRA CLUSTERS ON OCI (OPTIMIZED) 🔥              │
│                    [REHOST + OCI COMPUTE ENHANCEMENTS]                        │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  TUNDRA CLUSTER ARCHITECTURE (Same Logic, Better Infrastructure)       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌─────────────────────────────────┐                                         │
│  │      TUNDRA CLUSTERS            │                                         │
│  │   (Same Custom Engine)          │                                         │
│  ├─────────────────────────────────┤                                         │
│  │                                 │                                         │
│  │ ┌─────────────────────────────┐ │                                         │
│  │ │  Tundra Cluster A (Tenant1) │ │                                         │
│  │ │  • In-memory columnar store │ │  NO CODE CHANGES                       │
│  │ │  • MPP query execution      │ │  • Same Tundra binary                  │
│  │ │  • 8x faster than Snowflake │ │  • Same query engine                   │
│  │ │  • Query result caching     │ │  • Same data structures                │
│  │ │                             │ │                                         │
│  │ │  🔥 OCI COMPUTE SHAPES:     │ │  INFRASTRUCTURE CHANGES ONLY:          │
│  │ │                             │ │  ┌──────────────────────────────────┐  │
│  │ │  Option 1: FLEX SHAPES      │ │  │ 🔥🔥🔥 OCI DIFFERENTIATOR:       │  │
│  │ │  (Dynamic CPU/Memory)       │ │  │ FLEX SHAPES (Dynamic Sizing)    │  │
│  │ │  • Start: 16 OCPUs, 256GB   │ │  │                                  │  │
│  │ │  • Scale UP: 64 OCPUs, 1TB  │ │  │ • Resize CPU/RAM WITHOUT reboot │  │
│  │ │  • Scale DOWN: 8 OCPUs,128GB│ │  │ • Adjust to workload needs      │  │
│  │ │  • NO REBOOT required!      │ │  │ • Pay only for what you use     │  │
│  │ │  • Adjust in minutes        │ │  │ • AWS requires instance change  │  │
│  │ │                             │ │  │   (reboot, downtime, re-hydrate)│  │
│  │ │  WHY THIS MATTERS:          │ │  │                                  │  │
│  │ │  • Daily pattern: Scale up  │ │  │ Example Scenario:                │  │
│  │ │    8am-6pm (business hours) │ │  │ - 8am: 32 OCPUs, 512GB          │  │
│  │ │  • Scale down 6pm-8am       │ │  │ - 6pm: 16 OCPUs, 256GB          │  │
│  │ │  • 40% cost savings!        │ │  │ - NO cluster termination        │  │
│  │ │                             │ │  │ - NO re-hydration needed        │  │
│  │ │  Option 2: E5 (Memory)      │ │  │ - Data stays in memory          │  │
│  │ │  • Fixed: 64 OCPUs, 1TB RAM │ │  │ - 5-minute resize operation     │  │
│  │ │  • For predictable workloads│ │  │                                  │  │
│  │ │                             │ │  │ AWS Equivalent:                  │  │
│  │ │  Option 3: BARE METAL       │ │  │ - Must terminate r6i.16xl       │  │
│  │ │  • BM.Standard.E5.192       │ │  │ - Launch smaller instance       │  │
│  │ │  • 192 OCPUs, 2.8TB RAM     │ │  │ - Re-hydrate from S3 (10+ min)  │  │
│  │ │  • NO hypervisor overhead   │ │  │ - Data movement cost            │  │
│  │ │  • +10-15% performance      │ │  │                                  │  │
│  │ │  • For largest tenants      │ │  │ VALUE: Faster ops, lower cost   │  │
│  │ │                             │ │  └──────────────────────────────────┘  │
│  │ │  Storage:                   │ │                                         │
│  │ │  🔥 BLOCK VOLUMES (Auto)    │ │  ┌──────────────────────────────────┐  │
│  │ │  • Ultra High Performance   │ │  │ 🔥 OCI DIFFERENTIATOR:           │  │
│  │ │  • 200K IOPS, 480MB/s       │ │  │ AUTO-TUNED BLOCK STORAGE         │  │
│  │ │  • Auto-tuning enabled      │ │  │                                  │  │
│  │ │  • Adapts to workload       │ │  │ • OCI monitors I/O patterns     │  │
│  │ │  • No manual perf tuning!   │ │  │ • Automatically optimizes IOPS  │  │
│  │ │  • AWS: Must choose gp3 vs  │ │  │ • No need to pick gp2/gp3/io2   │  │
│  │ │    io2 manually, provision  │ │  │ • Eliminates guesswork           │  │
│  │ │    IOPS separately          │ │  │ • Better performance, less ops  │  │
│  │ │                             │ │  │                                  │  │
│  │ │  Network:                   │ │  │ VALUE: "Set and forget" storage │  │
│  │ │  • 2x50Gbps per VM (E5)     │ │  └──────────────────────────────────┘  │
│  │ │  • 100Gbps backbone         │ │                                         │
│  │ │  • Faster than AWS (25Gbps) │ │  ┌──────────────────────────────────┐  │
│  │ │  • FREE inter-VM traffic    │ │  │ 🔥 OCI DIFFERENTIATOR:           │  │
│  │ │                             │ │  │ INSTANCE PRINCIPALS              │  │
│  │ │  Authentication:            │ │  │                                  │  │
│  │ │  🔥 INSTANCE PRINCIPALS     │ │  │ • NO AWS access keys in code!   │  │
│  │ │  • Zero credentials in code │ │  │ • Tundra VM auto-auths to:      │  │
│  │ │  • Auto-auth to Obj Storage │ │  │   - Object Storage (reads)      │  │
│  │ │  • Auto-auth to Vault, etc. │ │  │   - Vault (encryption keys)     │  │
│  │ │  • Eliminate key rotation!  │ │  │   - OCI Cache (query cache)     │  │
│  │ └─────────────────────────────┘ │  │   - Monitoring (push metrics)   │  │
│  │           ▲                     │  │ • Instance identity = credential│  │
│  │           │ Hydrate from        │  │ • Managed by OCI IAM            │  │
│  │           │ Object Storage      │  │                                  │  │
│  │           │ (FAST: 100Gbps      │  │ Code change:                     │  │
│  │           │  network + FREE     │  │ AWS: boto3.client('s3',         │  │
│  │           │  egress)            │  │   aws_access_key_id=X,          │  │
│  │           │                     │  │   aws_secret_access_key=Y)      │  │
│  │ ┌─────────┴───────────────────┐ │  │ OCI: object_storage_client =    │  │
│  │ │  Tundra Cluster B (Tenant2) │ │  │   oci.object_storage.Client(    │  │
│  │ │  • OCI Instance Pool        │ │  │     config)  # Auto-auth!       │  │
│  │ │  • Auto-scaling (same logic)│ │  │                                  │  │
│  │ │  • Flex shapes: 8-64 OCPUs  │ │  │ VALUE: Security best practice   │  │
│  │ └─────────────────────────────┘ │  └──────────────────────────────────┘  │
│  │           ▲                     │                                         │
│  │           │                     │  ┌──────────────────────────────────┐  │
│  │ ┌─────────┴───────────────────┐ │  │ HYDRATION PERFORMANCE:           │  │
│  │ │  Tundra Cluster C (Tenant3) │ │  │                                  │  │
│  │ │  • Bare Metal (if needed)   │ │  │ AWS S3 → Tundra: 5-15 minutes   │  │
│  │ │  • Maximum performance      │ │  │ • 25Gbps network bottleneck     │  │
│  │ └─────────────────────────────┘ │  │ • S3 egress costs ($922/TB)     │  │
│  │                                 │  │ • gp3 EBS: 250MB/s baseline     │  │
│  │ Routing Logic: UNCHANGED        │  │                                  │  │
│  │ • Small datasets → Tundra       │  │ OCI Object Storage → Tundra:    │  │
│  │ • High query frequency → Tundra │  │ 3-8 minutes (40-60% faster!)    │  │
│  │ • Real-time dashboards → Tundra │  │ • 100Gbps backbone network      │  │
│  │ • Large datasets → Vertica (AWS)│  │ • FREE egress (first 10TB/mo)   │  │
│  │   via FastConnect               │  │ • Auto-tuned storage: 200K IOPS │  │
│  └─────────────────────────────────┘  │ • Parallel streaming downloads  │  │
│                                        │                                  │  │
│  Cluster Lifecycle: MINIMAL CHANGES    │ VALUE: Faster recovery, lower   │  │
│  ┌──────────────────────────────────┐ │        cost, better UX           │  │
│  │ 1. Launch: Instance Pool (OCI)  │ │ └──────────────────────────────────┘  │
│  │ 2. Fetch: Object Storage (OCI)  │ │                                         │
│  │ 3. Load: Same Tundra process    │ │  ┌──────────────────────────────────┐  │
│  │ 4. Index: Same logic            │ │  │ FAILURE RECOVERY (Better on OCI):│  │
│  │ 5. Ready: 3-8 min (vs 5-15 AWS) │ │  │                                  │  │
│  │                                  │ │  │ 1. Health check fails (OCI Mon.)│  │
│  │ Failure Recovery: UNCHANGED      │ │  │ 2. Traffic redirect (LB)        │  │
│  │ • Terminate failed cluster      │ │  │ 3. Launch new cluster (pool)    │  │
│  │ • Launch new from pool          │ │  │ 4. Hydrate from Object Storage  │  │
│  │ • Re-hydrate from Object Storage│ │  │ 5. Ready: 3-8 min (faster!)     │  │
│  │ • RTO: 5-10 min (faster on OCI!)│ │  │                                  │  │
│  └──────────────────────────────────┘ │  │ RTO: 5-10 min (vs 10-20 AWS)    │  │
│                                        │ RPO: 0 (Object Storage has all) │  │
│                                        └──────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                    VERTICA CLUSTERS (TEMPORARY - KEEP ON AWS)                 │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  **PHASE 1 DECISION: KEEP VERTICA ON AWS**                             │ │
│  │                                                                          │ │
│  │  Rationale:                                                             │ │
│  │  • Focus Phase 1 on Tundra (80% of queries)                            │ │
│  │  • Vertica migration complex (20% of queries, but stable)              │ │
│  │  • Defer to Phase 2: Vertica → ADB migration                           │ │
│  │                                                                          │ │
│  │  Cross-Cloud Connectivity (Temporary):                                  │ │
│  │  • OCI FastConnect + AWS Direct Connect                                │ │
│  │  • Private 10Gbps link: OCI VCN ↔ AWS VPC                              │ │
│  │  • Cost: ~$1K/month (FastConnect port)                                 │ │
│  │  • Query Orchestrator routes Vertica queries to AWS                    │ │
│  │  • Latency: +5-10ms (acceptable for batch workloads)                   │ │
│  │                                                                          │ │
│  │  EFFORT: 0 days (no migration)                                          │ │
│  │  COST: $1K/month FastConnect (eliminate in Phase 2)                    │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                EXTERNAL DATA WAREHOUSE INTEGRATION (UNCHANGED)                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Snowflake, Redshift, BigQuery, Databricks - ALL UNCHANGED                   │
│  • Domo Cloud Amplifier works across clouds                                  │
│  • Customers on AWS Snowflake: Connect via FastConnect                      │
│  • No changes to federated query logic                                       │
│  EFFORT: 0 days                                                              │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                     CACHING & ACCELERATION LAYER                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  AWS ElastiCache → OCI Cache (Redis)                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  CHANGE: Minimal (same Redis API)                                       │ │
│  │  • Export/import Redis data (RDB snapshot)                              │ │
│  │  • Update connection endpoints in app config                            │ │
│  │  • Keep same key structure                                              │ │
│  │  • OCI Cache: Multi-AD HA built-in                                      │ │
│  │  EFFORT: 1-2 days                                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  CloudFront CDN                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  **PHASE 1: KEEP CLOUDFRONT**                                           │ │
│  │  • Works with OCI Load Balancer as origin                               │ │
│  │  • Update origin to OCI LB endpoint                                     │ │
│  │  • No code changes needed                                               │ │
│  │  • Defer building OCI CDN to Phase 2                                    │ │
│  │  EFFORT: 1 day (config change)                                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                  PRESENTATION & APPLICATION LAYERS                            │
│                         [MINIMAL TO NO CHANGES]                               │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Domo Analyzer, Beast Mode, Domo Everywhere, App Dev Framework               │
│  • ALL stay as-is (containerized apps)                                       │
│  • Deploy containers to OCI Container Instances or OKE                       │
│  • Update backend API endpoints                                              │
│  • Zero UI/UX changes                                                        │
│  EFFORT: 3-5 days (container deployment)                                     │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                    🔥🔥🔥 MONITORING & OPERATIONS - OCI EDITION 🔥🔥🔥
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  AWS CloudWatch → OCI Monitoring + Logging                                    │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  CHANGE: Migrate dashboards and alarms                                  │ │
│  │  • Same metrics: CPU, memory, disk, network                             │ │
│  │  • Same custom metrics (query latency, etc.)                            │ │
│  │  • Recreate CloudWatch alarms as OCI Alarms                             │ │
│  │  • Update metric publishing in app code                                 │ │
│  │                                                                          │ │
│  │  OCI Logging:                                                            │ │
│  │  • Application logs → OCI Logging service                               │ │
│  │  • Query logs → OCI Logging Analytics                                   │ │
│  │  • Better search than CloudWatch Insights                               │ │
│  │                                                                          │ │
│  │  EFFORT: 3-5 days                                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  🔥 OCI Audit Service (Immutable Logs)                                        │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  • Every API call logged (like CloudTrail)                              │ │
│  │  • 🔥 IMMUTABLE: Cannot be deleted/modified (WORM)                       │ │
│  │  • Better compliance than CloudTrail                                    │ │
│  │  • 365 days retention FREE                                              │ │
│  │  • Export to Object Storage for longer retention                        │ │
│  │  EFFORT: 0 days (automatically enabled)                                 │ │
│  │  VALUE: Better audit trail for compliance                               │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                    🔥 NETWORK ARCHITECTURE - OCI EDITION 🔥
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  AWS VPC → OCI VCN (Virtual Cloud Network)                                    │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  CHANGE: Recreate network topology (same architecture)                  │ │
│  │                                                                          │ │
│  │  Public Subnets:                                                         │ │
│  │  • Load Balancer endpoints                                              │ │
│  │  • NAT Gateways (for private subnet outbound)                           │ │
│  │  • Bastion hosts                                                        │ │
│  │  • DRG (Dynamic Routing Gateway) - like AWS TGW                         │ │
│  │                                                                          │ │
│  │  Private Subnets - Application Tier:                                    │ │
│  │  • API Gateway services                                                 │ │
│  │  • Control plane microservices                                          │ │
│  │  • Container Instances / OKE nodes                                      │ │
│  │                                                                          │ │
│  │  Private Subnets - Data Tier:                                           │ │
│  │  • Tundra cluster VMs                                                   │ │
│  │  • Base Database (metadata)                                             │ │
│  │  • OCI Cache (Redis)                                                    │ │
│  │                                                                          │ │
│  │  Service Gateway (NO internet for OCI services):                        │ │
│  │  🔥 OCI DIFFERENTIATOR:                                                 │ │
│  │  • Object Storage: Private connection (no NAT, no egress cost)          │ │
│  │  • All OCI services: Regional service gateway (free)                    │ │
│  │  • AWS equivalent requires VPC endpoints ($$)                           │ │
│  │                                                                          │ │
│  │  Cross-Cloud Connectivity:                                              │ │
│  │  • OCI FastConnect port (10Gbps): $1,275/month                         │ │
│  │  • Connects to AWS Direct Connect                                       │ │
│  │  • Private link: OCI VCN ↔ AWS VPC                                      │ │
│  │  • For Vertica access (temporary, Phase 1 only)                         │ │
│  │                                                                          │ │
│  │  🔥 OCI DIFFERENTIATOR: CROSS-AD REPLICATION (FREE)                     │ │
│  │  • Built-in redundancy across Availability Domains                      │ │
│  │  • No cross-AZ data transfer charges (AWS charges)                      │ │
│  │  • Better HA at lower cost                                              │ │
│  │                                                                          │ │
│  │  EFFORT: 5-7 days (network setup)                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                    🔥🔥🔥 IMMEDIATE OCI VALUE SUMMARY 🔥🔥🔥
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  1. COST SAVINGS (Day 1):                                                     │
│     • FREE 10TB/month egress: $40K+/month savings                            │
│     • Auto-tiering Object Storage: 30-50% storage cost reduction             │
│     • Flex Shapes: 30-40% compute savings (auto-scale without reboot)        │
│     • No cross-AD transfer fees: $5-10K/month savings                        │
│     • Better CPUs (AMD EPYC): Same performance, 30% lower cost               │
│     TOTAL ESTIMATED: $50-70K/month = $600K-840K/year                         │
│                                                                               │
│  2. PERFORMANCE (Day 1):                                                      │
│     • Faster hydration: 3-8 min (vs 5-15 min AWS) = 40-60% faster           │
│     • 100Gbps inter-VM network (vs 25Gbps AWS) = 4x bandwidth                │
│     • Auto-tuned Block Storage: No performance tuning needed                 │
│     • Bare Metal option: +10-15% performance (no hypervisor)                 │
│                                                                               │
│  3. SECURITY (Day 1):                                                         │
│     • Instance Principals: Eliminate AWS access keys                         │
│     • OCI Vault: FIPS 140-2 Level 3 HSM (vs AWS Level 2)                    │
│     • Immutable Audit Logs: Better compliance (SOC 2, HIPAA, GDPR)          │
│     • Immutable Object Storage: WORM for data governance                     │
│                                                                               │
│  4. OPERATIONS (Day 1):                                                       │
│     • Flex Shapes: Resize without reboot/re-hydrate                          │
│     • Auto-tuned Storage: Zero performance tuning ops                        │
│     • Serverless Data Flow: No Spark cluster management                      │
│     • Better monitoring: OCI Ops Insights                                    │
│                                                                               │
│  5. COMPLIANCE (Day 1):                                                       │
│     • Immutable logs (OCI Audit)                                             │
│     • Immutable storage (Object Storage WORM)                                │
│     • Better HSM (Vault FIPS 140-2 Level 3)                                  │
│     • Easier to pass SOC 2, HIPAA, PCI-DSS audits                            │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                    MIGRATION TIMELINE - RAPID REPLATFORM
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  WEEK 1-2: Infrastructure Setup                                               │
│  • Create OCI tenancy structure (compartments, IAM)                           │
│  • Set up VCN with public/private subnets (multi-AD)                          │
│  • Deploy Load Balancer, NAT Gateway, DRG                                     │
│  • Enable OCI Vault, Audit, Monitoring                                        │
│  • Order FastConnect (10Gbps port to AWS)                                     │
│  TEAM: 2-3 cloud engineers                                                    │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  WEEK 2-3: Data Migration (Parallel with Week 1-2)                            │
│  • Create Object Storage buckets (WORM, auto-tier, replication)              │
│  • Data transfer:                                                             │
│    - Small (<10TB): Rclone over FastConnect                                  │
│    - Large (>50TB): OCI Data Transfer Service (appliance)                    │
│  • Validate data integrity (checksums)                                        │
│  • Set up continuous sync (AWS S3 → OCI Object Storage)                      │
│  TEAM: 2 data engineers                                                       │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  WEEK 3-4: Application Migration                                              │
│  • Deploy metadata DB (Base Database, restore from RDS dump)                  │
│  • Deploy OCI Cache (migrate Redis data)                                      │
│  • Update SDK code:                                                           │
│    - boto3 (S3) → oci.object_storage                                         │
│    - AWS IAM credentials → Instance Principals                               │
│  • Deploy control plane services (containers to OCI)                          │
│  • Update Query Orchestrator endpoints                                        │
│  • Deploy Cluster Manager (OCI Instance Pools)                                │
│  TEAM: 4-5 backend engineers                                                  │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  WEEK 4-5: Tundra Cluster Deployment                                          │
│  • Create Tundra compute images (same binary, OCI-optimized)                 │
│  • Set up Instance Pools (Flex shapes: 16-64 OCPUs)                          │
│  • Configure auto-scaling policies                                            │
│  • Test hydration from Object Storage                                         │
│  • Benchmark performance vs AWS                                               │
│  • Test failover/recovery                                                     │
│  TEAM: 3 platform engineers                                                   │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  WEEK 5-6: Testing & Validation                                               │
│  • Run test tenant (internal Domo team)                                       │
│  • Query performance testing                                                  │
│  • Load testing (simulate production traffic)                                 │
│  • Disaster recovery testing                                                  │
│  • Security validation (penetration testing)                                  │
│  • Compliance review                                                          │
│  TEAM: 2 QA engineers, 1 security engineer                                    │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  WEEK 6-7: Pilot Migration (5% of tenants)                                    │
│  • Select low-risk pilot tenants                                              │
│  • DNS cutover for pilot (Route 53 → OCI DNS)                                │
│  • Monitor closely (24/7 on-call)                                             │
│  • Fix bugs found                                                             │
│  • Gather performance metrics                                                 │
│  TEAM: Full team on standby                                                   │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  WEEK 7-8: Full Cutover (Remaining 95%)                                       │
│  • Gradual tenant migration (10-20%/day)                                      │
│  • DNS updates in waves                                                       │
│  • Monitor customer impact                                                    │
│  • Communication plan (notify customers)                                      │
│  • Rollback plan ready (keep AWS warm for 2 weeks)                            │
│  TEAM: Full team + SRE + customer success                                     │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  WEEK 9-10: Stabilization                                                     │
│  • Monitor production (all tenants on OCI)                                    │
│  • Fine-tune performance                                                      │
│  • Cost optimization (rightsize Flex shapes)                                  │
│  • Begin AWS decommissioning (except Vertica)                                │
│  TEAM: Reduced team (2-3 engineers on-call)                                   │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                        DEFERRED TO PHASE 2 (Month 3+)
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  Items NOT in Phase 1 (Rapid Replatform):                                    │
│  • Vertica → ADB migration (keep Vertica on AWS temporarily)                 │
│  • AI services migration (keep Bedrock on AWS, cross-cloud)                  │
│  • Data Science workspace migration (keep on AWS)                             │
│  • OKE/Kubernetes migration (use Container Instances for now)                │
│  • Custom CDN on OCI (keep CloudFront)                                        │
│  • Advanced OCI features: Service Mesh, API Gateway managed service          │
│  • Snowflake on OCI integration (wait for availability)                      │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                        RISK MITIGATION STRATEGIES
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  1. Data Integrity Risks:                                                     │
│     • Mitigation: Checksum validation (S3 ETags → OCI MD5)                   │
│     • Continuous sync during migration (bidirectional for 2 weeks)            │
│     • Automated testing: Compare query results AWS vs OCI                     │
│                                                                               │
│  2. Performance Risks:                                                         │
│     • Mitigation: Benchmark before cutover (should be faster on OCI)         │
│     • Pilot tenant performance monitoring                                     │
│     • Rollback plan: DNS revert to AWS (< 5 min)                             │
│                                                                               │
│  3. Cross-Cloud Latency (Vertica on AWS):                                     │
│     • Mitigation: FastConnect 10Gbps (low latency <10ms)                     │
│     • Monitor query performance for cross-cloud queries                       │
│     • Prioritize Vertica→ADB migration in Phase 2                            │
│                                                                               │
│  4. OCI Learning Curve:                                                        │
│     • Mitigation: Training for ops team (OCI certifications)                 │
│     • OCI support engagement (TAM assigned)                                   │
│     • Keep AWS expertise for 3-6 months during transition                     │
│                                                                               │
│  5. Customer Impact:                                                           │
│     • Mitigation: Transparent migration (zero downtime cutover via DNS)      │
│     • Customer communication plan (advance notice)                            │
│     • 24/7 support during migration period                                    │
│     • Rollback plan tested and ready                                          │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                    SUCCESS CRITERIA - PHASE 1 COMPLETE
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  ✓ 95%+ of tenants running on OCI (Tundra clusters only)                     │
│  ✓ Query performance ≥ AWS performance (target: 20-40% faster)               │
│  ✓ Hydration time < 10 minutes (target: 3-8 minutes)                         │
│  ✓ Zero data loss (RPO = 0)                                                  │
│  ✓ RTO < 15 minutes (target: 5-10 minutes)                                   │
│  ✓ Customer satisfaction ≥ pre-migration baseline                            │
│  ✓ Cost savings validated: $50K+/month reduction                             │
│  ✓ Security/compliance: Pass internal audit                                  │
│  ✓ Ops team trained on OCI (certifications completed)                        │
│  ✓ AWS costs reduced by 60-70% (Vertica still on AWS = 30-40% of costs)     │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                        TOTAL EFFORT ESTIMATE
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  Team Composition:                                                            │
│  • 2-3 Cloud/Infrastructure Engineers                                         │
│  • 4-5 Backend/Platform Engineers                                             │
│  • 2 Data Engineers                                                           │
│  • 2 QA/Test Engineers                                                        │
│  • 1 Security Engineer                                                        │
│  • 1 Project Manager                                                          │
│  • OCI TAM (Technical Account Manager from Oracle)                           │
│                                                                               │
│  Timeline: 8-10 weeks (Phase 1 - Tundra only)                                │
│                                                                               │
│  Total Engineering Hours: ~3,000-4,000 hours                                 │
│  • Infrastructure: 600 hours                                                  │
│  • Data migration: 400 hours                                                  │
│  • Application code: 1,000 hours                                              │
│  • Testing/QA: 600 hours                                                      │
│  • Security/compliance: 300 hours                                             │
│  • Documentation/training: 200 hours                                          │
│  • Project management: 200 hours                                              │
│  • Buffer (unknowns): 700-1,700 hours                                         │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                    KEY DECISION: TUNDRA-ONLY FIRST
================================================================================

This architecture is optimized for SPEED and IMMEDIATE VALUE:

✓ Tundra clusters (80% of query volume) → OCI (fast migration)
✗ Vertica (20% of queries) → Stay on AWS temporarily (defer complexity)
✓ Focus on OCI differentiators that require minimal code changes
✓ Achieve 60-70% cost savings immediately
✓ Demonstrate OCI value quickly (8-10 weeks)
✓ Build confidence for Phase 2 (Vertica→ADB, AI services, etc.)

This is a PRAGMATIC approach: Maximum value, minimum risk, fastest timeline.

SUMMARY: Minimum Changes + Maximum OCI Value
What Changes:

Infrastructure → OCI (Load Balancer, Compute, Object Storage)
SDK calls → OCI SDK (boto3 → oci)
IAM → Instance Principals (zero-key auth)
Network → VCN + Service Gateway

What Stays the Same:

Tundra binary/logic (unchanged)
Query orchestration logic
Application code (95%)
Vertica (on AWS temporarily)
AI services (on AWS via FastConnect)

Immediate OCI Wins:

$50-70K/month savings (free egress, Flex shapes)
40-60% faster hydration (100Gbps network)
Zero-key security (Instance Principals)
Better compliance (WORM storage, immutable logs)
Easier ops (auto-tuned storage, Flex shapes)

Timeline: 8-10 weeks to production with Tundra on OCI, Vertica still on AWS.