================================================================================
        DOMO TUNDRA CUSTOMER MIGRATION STRATEGY: AWS → OCI
                    (Zero-Downtime, Gradual Tenant Migration)
================================================================================

ASSUMPTION: Control plane is fully operational on OCI
GOAL: Migrate customer workloads (data + Tundra clusters) with ZERO downtime
APPROACH: Blue-Green style migration with DNS cutover per tenant cohort

================================================================================
                        MIGRATION ARCHITECTURE OVERVIEW
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│                         PRE-MIGRATION STATE                                   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  CONTROL PLANE (OCI) ✓ Running                                         │ │
│  │  • Query Orchestrator                                                   │ │
│  │  • Cluster Manager                                                      │ │
│  │  • Metadata Service                                                     │ │
│  │  • API Gateway                                                          │ │
│  │  • Load Balancer: lb.domo-oci.com                                      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  CUSTOMER WORKLOADS (AWS) - TO BE MIGRATED                             │ │
│  │                                                                          │ │
│  │  Tenant-1:                                                              │ │
│  │  • Data: AWS S3 bucket (s3://domo-tenant-1/)                           │ │
│  │  • Tundra Cluster: EC2 instances in us-east-1                          │ │
│  │  • DNS: tenant1.domo.com → AWS ALB → AWS Tundra cluster               │ │
│  │                                                                          │ │
│  │  Tenant-2:                                                              │ │
│  │  • Data: AWS S3 bucket (s3://domo-tenant-2/)                           │ │
│  │  • Tundra Cluster: EC2 instances in us-west-2                          │ │
│  │  • DNS: tenant2.domo.com → AWS ALB → AWS Tundra cluster               │ │
│  │                                                                          │ │
│  │  ... (N tenants total)                                                  │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                         MIGRATION PHASES OVERVIEW                             │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Phase 0: Preparation & Planning (1-2 weeks)                                 │
│  • Tenant segmentation & risk profiling                                      │
│  • Migration tooling setup                                                   │
│  • Monitoring & observability baseline                                       │
│  • Communication plan                                                        │
│                                                                               │
│  Phase 1: Pilot Migration (1-2 weeks)                                        │
│  • Internal test tenants (Domo employees)                                    │
│  • 2-3 small external tenants (friendly customers)                           │
│  • Validate zero-downtime process                                            │
│  • Gather performance metrics                                                │
│                                                                               │
│  Phase 2: Wave-Based Migration (8-12 weeks)                                  │
│  • Wave 1: Small tenants (<10GB, low complexity) - 30% of tenants           │
│  • Wave 2: Medium tenants (10-100GB, standard features) - 50% of tenants    │
│  • Wave 3: Large tenants (100GB-1TB, complex queries) - 15% of tenants      │
│  • Wave 4: Enterprise tenants (>1TB, mission-critical) - 5% of tenants      │
│                                                                               │
│  Phase 3: Decommissioning (2-4 weeks)                                        │
│  • Monitor all tenants on OCI for 2 weeks                                    │
│  • Validate cost savings                                                     │
│  • Decommission AWS resources                                                │
│  • Final reporting & retrospective                                           │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                    PHASE 0: PREPARATION & PLANNING (WEEK 1-2)
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  STEP 0.1: TENANT SEGMENTATION & RISK PROFILING                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Objective: Categorize all tenants by risk, complexity, and priority         │
│                                                                               │
│  Criteria:                                                                    │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Data Size:                                                             │ │
│  │  • Micro: <1GB                                                          │ │
│  │  • Small: 1-10GB                                                        │ │
│  │  • Medium: 10-100GB                                                     │ │
│  │  • Large: 100GB-1TB                                                     │ │
│  │  • Enterprise: >1TB                                                     │ │
│  │                                                                          │ │
│  │  Query Complexity:                                                      │ │
│  │  • Simple: Basic aggregations, <10 tables                               │ │
│  │  • Standard: Joins, filters, 10-50 tables                               │ │
│  │  • Complex: Multi-step ETL, 50+ tables, advanced analytics             │ │
│  │                                                                          │ │
│  │  Business Criticality:                                                  │ │
│  │  • Low: Dev/test environments, inactive users                           │ │
│  │  • Medium: Standard business dashboards, daily use                      │ │
│  │  • High: Executive dashboards, real-time operations                     │ │
│  │  • Mission-Critical: Revenue-impacting, SLA commitments                 │ │
│  │                                                                          │ │
│  │  SLA Tier:                                                              │ │
│  │  • Standard: Best effort, 99% uptime                                    │ │
│  │  • Premium: 99.5% uptime, 4-hour response                               │ │
│  │  • Enterprise: 99.9% uptime, 1-hour response, dedicated TAM            │ │
│  │                                                                          │ │
│  │  Technical Characteristics:                                             │ │
│  │  • Query patterns: Batch vs real-time                                   │ │
│  │  • Peak usage hours: 24/7 vs business hours only                        │ │
│  │  • Data refresh frequency: Real-time, hourly, daily                     │ │
│  │  • Custom integrations: APIs, webhooks, custom connectors               │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  Output: Tenant Migration Matrix                                             │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ Tenant ID | Size | Complexity | Criticality | SLA  | Wave | Priority  │ │
│  │-----------|------|------------|-------------|------|------|-----------|│ │
│  │ T-0001    | 2GB  | Simple     | Low         | Std  |  1   | Pilot     │ │
│  │ T-0002    | 5GB  | Standard   | Medium      | Std  |  1   | Wave-1    │ │
│  │ T-0123    | 50GB | Complex    | High        | Prem |  2   | Wave-2    │ │
│  │ T-0999    | 2TB  | Complex    | Critical    | Ent  |  4   | Wave-4    │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  DELIVERABLE: CSV file with all tenants categorized                          │
│  OWNER: Customer Success + Product Management                                │
│  DURATION: 3-5 days                                                          │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  STEP 0.2: MIGRATION TOOLING SETUP                                           │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Tool 1: Data Replication Pipeline (Continuous Sync)                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Technology: Rclone + OCI Object Storage Replication                    │ │
│  │                                                                          │ │
│  │  Setup:                                                                  │ │
│  │  1. Install Rclone on migration orchestrator (OCI Compute)              │ │
│  │  2. Configure S3 source (AWS credentials via IAM role)                  │ │
│  │  3. Configure OCI Object Storage destination (Instance Principal)       │ │
│  │  4. Enable bidirectional sync (for rollback capability)                 │ │
│  │                                                                          │ │
│  │  Rclone config:                                                          │ │
│  │  ```                                                                     │ │
│  │  [aws-s3-tenant-1]                                                       │ │
│  │  type = s3                                                               │ │
│  │  provider = AWS                                                          │ │
│  │  region = us-east-1                                                      │ │
│  │  access_key_id = AKIA...                                                 │ │
│  │  secret_access_key = ...                                                 │ │
│  │                                                                          │ │
│  │  [oci-obj-storage-tenant-1]                                              │ │
│  │  type = s3                                                               │ │
│  │  provider = Other                                                        │ │
│  │  endpoint = https://objectstorage.us-ashburn-1.oraclecloud.com          │ │
│  │  # Uses instance principal, no keys needed                              │ │
│  │  ```                                                                     │ │
│  │                                                                          │ │
│  │  Sync command (per tenant):                                              │ │
│  │  ```bash                                                                 │ │
│  │  rclone sync aws-s3-tenant-1:domo-tenant-1 \                            │ │
│  │              oci-obj-storage-tenant-1:domo-tenant-1-oci \               │ │
│  │              --transfers=32 \                                            │ │
│  │              --checkers=16 \                                             │ │
│  │              --checksum \                                                │ │
│  │              --progress \                                                │ │
│  │              --log-file=/var/log/migration/tenant-1.log                 │ │
│  │  ```                                                                     │ │
│  │                                                                          │ │
│  │  Monitoring:                                                             │ │
│  │  • Track sync progress (files copied, bytes transferred)                │ │
│  │  • Alert on errors (checksum mismatches, network failures)              │ │
│  │  • Dashboard: Grafana with Rclone metrics                               │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  Tool 2: Migration Orchestrator (Automation Engine)                          │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Technology: Python + Terraform + Ansible                               │ │
│  │                                                                          │ │
│  │  Components:                                                             │ │
│  │  • migration_controller.py: Main orchestration logic                    │ │
│  │  • terraform/: Infrastructure provisioning (OCI Tundra clusters)        │ │
│  │  • ansible/: Configuration management (cluster setup, health checks)    │ │
│  │  • scripts/: Utility scripts (DNS updates, validation tests)            │ │
│  │                                                                          │ │
│  │  Migration Controller Features:                                         │ │
│  │  • Tenant state machine: [Pending → Syncing → Ready → Migrated]        │ │
│  │  • Pre-flight checks: Data sync complete, cluster health, DNS ready     │ │
│  │  • Cutover execution: DNS update, traffic routing, validation           │ │
│  │  • Rollback automation: Revert DNS, route back to AWS                   │ │
│  │  • Slack notifications: Real-time status updates                        │ │
│  │                                                                          │ │
│  │  State Database: PostgreSQL on OCI Base Database                        │ │
│  │  Schema:                                                                 │ │
│  │  ```sql                                                                  │ │
│  │  CREATE TABLE tenant_migrations (                                        │ │
│  │    tenant_id VARCHAR(50) PRIMARY KEY,                                   │ │
│  │    wave_number INT,                                                     │ │
│  │    state VARCHAR(20), -- pending, syncing, ready, migrated, failed     │ │
│  │    aws_s3_bucket VARCHAR(255),                                          │ │
│  │    oci_bucket VARCHAR(255),                                             │ │
│  │    data_size_gb DECIMAL(10,2),                                          │ │
│  │    sync_started_at TIMESTAMP,                                           │ │
│  │    sync_completed_at TIMESTAMP,                                         │ │
│  │    cutover_started_at TIMESTAMP,                                        │ │
│  │    cutover_completed_at TIMESTAMP,                                      │ │
│  │    aws_tundra_cluster_id VARCHAR(100),                                  │ │
│  │    oci_tundra_cluster_id VARCHAR(100),                                  │ │
│  │    dns_record VARCHAR(255),                                             │ │
│  │    rollback_required BOOLEAN,                                           │ │
│  │    notes TEXT                                                            │ │
│  │  );                                                                      │ │
│  │  ```                                                                     │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  Tool 3: Validation & Testing Framework                                      │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Automated Tests (per tenant):                                          │ │
│  │                                                                          │ │
│  │  1. Data Integrity Test:                                                │ │
│  │     • Compare row counts: AWS S3 vs OCI Object Storage                  │ │
│  │     • Checksum validation (MD5 hashes)                                  │ │
│  │     • Sample data comparison (random 1000 rows)                         │ │
│  │                                                                          │ │
│  │  2. Query Performance Test:                                             │ │
│  │     • Replay production queries (last 7 days)                           │ │
│  │     • Compare latency: AWS Tundra vs OCI Tundra                         │ │
│  │     • Validate results match (checksums on result sets)                 │ │
│  │                                                                          │ │
│  │  3. Cluster Health Check:                                               │ │
│  │     • OCI Tundra cluster: CPU, memory, disk, network                    │ │
│  │     • Hydration status: 100% loaded from Object Storage                 │ │
│  │     • Connection pool: Min 10 available connections                     │ │
│  │                                                                          │ │
│  │  4. End-to-End (E2E) Test:                                              │ │
│  │     • Simulate user login                                               │ │
│  │     • Load dashboard (via API)                                          │ │
│  │     • Run query, validate response time <2 seconds                      │ │
│  │     • Check for errors in logs                                          │ │
│  │                                                                          │ │
│  │  Test Harness: Python + Pytest                                          │ │
│  │  ```python                                                               │ │
│  │  def test_tenant_migration_readiness(tenant_id):                        │ │
│  │      # 1. Data sync complete                                            │ │
│  │      assert check_data_sync_complete(tenant_id)                         │ │
│  │      # 2. OCI cluster healthy                                           │ │
│  │      assert check_oci_cluster_health(tenant_id)                         │ │
│  │      # 3. Query performance acceptable                                  │ │
│  │      assert run_performance_tests(tenant_id)                            │ │
│  │      # 4. E2E test passes                                               │ │
│  │      assert run_e2e_test(tenant_id)                                     │ │
│  │  ```                                                                     │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  DELIVERABLE: Fully automated migration pipeline ready                       │
│  OWNER: Platform Engineering Team                                            │
│  DURATION: 5-7 days                                                          │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  STEP 0.3: MONITORING & OBSERVABILITY BASELINE                               │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Objective: Establish baseline metrics BEFORE migration for comparison       │
│                                                                               │
│  Metrics to Capture (per tenant, last 14 days):                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Performance Metrics:                                                   │ │
│  │  • Query latency: p50, p95, p99 (milliseconds)                         │ │
│  │  • Queries per minute (QPM)                                             │ │
│  │  • Dashboard load time (seconds)                                        │ │
│  │  • API response time (milliseconds)                                     │ │
│  │  • Hydration time (minutes) - on cluster restart                       │ │
│  │                                                                          │ │
│  │  Availability Metrics:                                                  │ │
│  │  • Uptime % (target: 99.9%)                                             │ │
│  │  • Error rate (errors per 1000 requests)                                │ │
│  │  • Failed queries (count + %)                                           │ │
│  │  • Cluster restart frequency                                            │ │
│  │                                                                          │ │
│  │  Resource Utilization (AWS Tundra Clusters):                            │ │
│  │  • CPU utilization (avg, max)                                           │ │
│  │  • Memory utilization (avg, max)                                        │ │
│  │  • Disk IOPS (read, write)                                              │ │
│  │  • Network throughput (GB/day)                                          │ │
│  │                                                                          │ │
│  │  User Activity:                                                         │ │
│  │  • Active users per day                                                 │ │
│  │  • Peak usage hours                                                     │ │
│  │  • Most-used dashboards                                                 │ │
│  │  • API calls per day                                                    │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  Dashboard: Grafana "Migration Health Dashboard"                             │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Panels:                                                                │ │
│  │  1. Overall Migration Progress (% tenants migrated)                     │ │
│  │  2. Active Migrations (tenants currently in cutover)                    │ │
│  │  3. Performance Comparison: AWS vs OCI (side-by-side)                   │ │
│  │  4. Error Rate Trend (before and after migration)                       │ │
│  │  5. Cost Savings Tracker (cumulative savings)                           │ │
│  │  6. Tenant List with Status (color-coded: green=success, red=issue)    │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  Alerting: PagerDuty Integration                                             │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Critical Alerts (24/7 on-call):                                        │ │
│  │  • Tenant migration failed (rollback triggered)                         │ │
│  │  • Query error rate >5% after cutover                                   │ │
│  │  • OCI cluster unhealthy >5 minutes                                     │ │
│  │  • Data sync failure (checksum mismatch)                                │ │
│  │                                                                          │ │
│  │  Warning Alerts (business hours):                                       │ │
│  │  • Query latency increased >20% post-migration                          │ │
│  │  • Data sync lag >2 hours                                               │ │
│  │  • OCI cluster CPU >80% for >10 minutes                                 │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  DELIVERABLE: Baseline metrics report + monitoring dashboards                │
│  OWNER: SRE Team                                                             │
│  DURATION: 2-3 days                                                          │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  STEP 0.4: CUSTOMER COMMUNICATION PLAN                                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Communication Timeline:                                                      │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  T-14 days: Initial Announcement                                        │ │
│  │  • Email to all customers                                               │ │
│  │  • Subject: "Domo Infrastructure Upgrade: Improved Performance Ahead"  │ │
│  │  • Content:                                                             │ │
│  │    - We're upgrading to a new cloud infrastructure (OCI)                │ │
│  │    - Expected benefits: 40% faster queries, better reliability          │ │
│  │    - Zero downtime, transparent migration                               │ │
│  │    - Your specific migration window: [Date range]                       │ │
│  │  • Link to FAQ page                                                     │ │
│  │                                                                          │ │
│  │  T-7 days: Reminder + Migration Window                                  │ │
│  │  • Email reminder with exact date/time                                  │ │
│  │  • "Your migration: Tuesday, Feb 14, 2:00-4:00 AM EST"                 │ │
│  │  • In-app notification banner                                           │ │
│  │  • Offer to reschedule if concerns                                      │ │
│  │                                                                          │ │
│  │  T-1 day: Final Reminder                                                │ │
│  │  • Email + SMS (if opted in)                                            │ │
│  │  • In-app notification (prominent)                                      │ │
│  │  • Support hotline number provided                                      │ │
│  │                                                                          │ │
│  │  T-0 (Migration Day):                                                   │ │
│  │  • Status page update: "Tenant X migration in progress"                │ │
│  │  • Live chat support available                                          │ │
│  │  • Monitor customer queries closely                                     │ │
│  │                                                                          │ │
│  │  T+1 hour: Post-Migration Check-in                                      │ │
│  │  • Email: "Your upgrade is complete!"                                   │ │
│  │  • Include performance improvements observed                            │ │
│  │  • Request feedback via quick survey                                    │ │
│  │  • Support contact for any issues                                       │ │
│  │                                                                          │ │
│  │  T+7 days: Follow-up                                                    │ │
│  │  • Email survey: How's your experience post-upgrade?                    │ │
│  │  • Proactive reach-out for any degraded performance                     │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  Customer Support Readiness:                                                  │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  • Train support team on migration (FAQs, troubleshooting)              │ │
│  │  • Extended support hours during migration waves                        │ │
│  │  • Escalation path: L1 → L2 → Migration Team Lead → VP Eng             │ │
│  │  • Dedicated Slack channel: #migration-customer-issues                  │ │
│  │  • Knowledge base articles: "What to expect during migration"           │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  DELIVERABLE: Communication templates + support runbook                      │
│  OWNER: Customer Success + Support                                           │
│  DURATION: 2-3 days                                                          │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
              PHASE 1: PILOT MIGRATION (WEEK 3-4) - 5 TENANTS
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  PILOT TENANT SELECTION                                                      │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Criteria for Pilot Tenants:                                                 │
│  • Internal Domo teams (2 tenants) - Full control, easy rollback            │
│  • Friendly external customers (3 tenants) - Low risk, willing to provide   │
│    feedback                                                                   │
│  • Mix of sizes: 1 micro (<1GB), 2 small (1-10GB), 1 medium (10-50GB),     │
│    1 with complex queries                                                    │
│  • Not mission-critical (can tolerate minor issues)                          │
│  • Geographically distributed (test latency in different regions)            │
│                                                                               │
│  Example Pilot Tenants:                                                      │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ ID      | Customer        | Size | Region    | Complexity | Criticality││
│  │---------|-----------------|------|-----------|------------|------------││
│  │ T-DEMO1 | Domo Internal 1 | 2GB  | us-east-1 | Simple     | Low        ││
│  │ T-DEMO2 | Domo Internal 2 | 5GB  | us-west-2 | Standard   | Low        ││
│  │ T-0042  | Friendly Co A   | 1GB  | us-east-1 | Simple     | Medium     ││
│  │ T-0088  | Friendly Co B   | 15GB | eu-west-1 | Standard   | Medium     ││
│  │ T-0156  | Friendly Co C   | 40GB | us-west-2 | Complex    | Medium     ││
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  DETAILED MIGRATION STEPS - PER TENANT (ZERO DOWNTIME)                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  STEP 1.1: PRE-MIGRATION DATA SYNC (T-72 hours)                             │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Objective: Sync tenant data from AWS S3 → OCI Object Storage           │ │
│  │                                                                          │ │
│  │  Process:                                                                │ │
│  │  1. Initiate sync: `./migrate.py sync-start --tenant T-0042`           │ │
│  │  2. Rclone performs initial bulk copy                                   │ │
│  │     • Source: s3://domo-tenant-42/                                      │ │
│  │     • Dest: oci://domo-tenant-42-oci/                                   │ │
│  │     • Duration: Depends on size (1GB = ~5 min, 100GB = ~2 hours)       │ │
│  │  3. Verify checksums for all files                                      │ │
│  │  4. Enable continuous sync (every 15 minutes)                           │ │
│  │     • Captures new data written to AWS S3                               │ │
│  │     • Keeps OCI Object Storage <15 min behind                           │ │
│  │  5. Monitor sync lag dashboard                                          │ │
│  │                                                                          │ │
│  │  AWS S3 (Production)                 OCI Object Storage (Sync Target)   │ │
│  │  ┌──────────────────┐                ┌──────────────────┐              │ │
│  │  │ tenant-42/       │                │ tenant-42-oci/   │              │ │
│  │  │  ├─ raw/         │  ──Rclone──>  │  ├─ raw/         │              │ │
│  │  │  ├─ processed/   │  (continuous)  │  ├─ processed/   │              │ │
│  │  │  └─ metadata/    │                │  └─ metadata/    │              │ │
│  │  └──────────────────┘                └──────────────────┘              │ │
│  │       ▲ (writes)                           ▼ (reads for validation)     │ │
│  │       │                                                                 │ │
│  │  Active Users                                                           │ │
│  │  (Still using AWS)                                                      │ │
│  │                                                                          │ │
│  │  State: SYNCING                                                         │ │
│  │  Duration: 1-24 hours (depending on data size)                          │ │
│  │  Success Criteria: Sync lag <15 minutes, zero checksum errors           │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  STEP 1.2: OCI TUNDRA CLUSTER PROVISIONING (T-48 hours)                     │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Objective: Create OCI Tundra cluster, hydrate from Object Storage      │ │
│  │                                                                          │ │
│  │  Process:                                                                │ │
│  │  1. Provision OCI Tundra cluster via Terraform                          │ │
│  │     ```bash                                                              │ │
│  │     terraform apply \                                                    │ │
│  │       -var="tenant_id=T-0042" \                                         │ │
│  │       -var="cluster_size=small" \                                       │ │
│  │       -var="shape=VM.Standard.E5.Flex" \                                │ │
│  │       -var="ocpus=16" \                                                  │ │
│  │       -var="memory_gb=256"                                              │ │
│  │     ```                                                                  │ │
│  │                                                                          │ │
│  │  2. Infrastructure created:                                              │ │
│  │     • 3 Tundra VMs (HA setup, multi-AD)                                 │ │
│  │     • Private subnet, security groups                                   │ │
│  │     • Block volumes (Ultra High Performance)                            │ │
│  │     • Internal Load Balancer (for cluster nodes)                        │ │
│  │                                                                          │ │
│  │  3. Deploy Tundra binary via Ansible                                    │ │
│  │     • Copy Tundra binaries to VMs                                       │ │
│  │     • Configure cluster settings (same as AWS)                          │ │
│  │     • Point to OCI Object Storage endpoint                              │ │
│  │                                                                          │ │
│  │  4. Hydrate cluster from OCI Object Storage                             │ │
│  │     • Tundra reads data from oci://domo-tenant-42-oci/                  │ │
│  │     • Parallel download (32 streams)                                    │ │
│  │     • Load into in-memory columnar store                                │ │
│  │     • Build indexes                                                     │ │
│  │     • Duration: 3-8 minutes (OCI's 100Gbps network)                     │ │
│  │                                                                          │ │
│  │  5. Cluster health check                                                │ │
│  │     • All nodes responsive                                              │ │
│  │     • Memory utilization <70%                                           │ │
│  │     • All datasets loaded                                               │ │
│  │     • Index build complete                                              │ │
│  │                                                                          │ │
│  │  OCI Tundra Cluster (Standby)                                           │ │
│  │  ┌────────────────────────────────────────────────────────────────┐    │ │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                     │    │ │
│  │  │  │ Tundra-1 │  │ Tundra-2 │  │ Tundra-3 │                     │    │ │
│  │  │  │  (AD-1)  │  │  (AD-2)  │  │  (AD-3)  │                     │    │ │
│  │  │  └────┬─────┘  └────┬─────┘  └────┬─────┘                     │    │ │
│  │  │       └─────────────┴─────────────┘                            │    │ │
│  │  │                     │                                           │    │ │
│  │  │           Internal Load Balancer                               │    │ │
│  │  │           (NOT in DNS yet - standby mode)                       │    │ │
│  │  └────────────────────────────────────────────────────────────────┘    │ │
│  │                       ▲                                                 │ │
│  │                       │ (hydrate from)                                  │ │
│  │            OCI Object Storage: tenant-42-oci/                           │ │
│  │                                                                          │ │
│  │  State: READY (but not receiving traffic yet)                           │ │
│  │  Duration: 15-30 minutes                                                │ │
│  │  Success Criteria: All health checks pass, sample queries run <2s       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  STEP 1.3: VALIDATION TESTING (T-24 hours)                                  │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Objective: Validate OCI cluster works correctly BEFORE cutover         │ │
│  │                                                                          │ │
│  │  Test Suite:                                                             │ │
│  │  1. Data Integrity Test                                                 │ │
│  │     • Query row counts from OCI Tundra                                  │ │
│  │     • Compare with AWS Tundra (should match within 0.1%)                │ │
│  │     • Sample 1000 random rows, compare checksums                        │ │
│  │     Result: ✓ PASS - Data matches                                       │ │
│  │                                                                          │ │
│  │  2. Query Performance Test                                              │ │
│  │     • Replay last 100 production queries                                │ │
│  │     • Compare latency: OCI vs AWS                                       │ │
│  │       - AWS p95: 1200ms                                                 │ │
│  │       - OCI p95: 850ms (29% faster! ✓)                                  │ │
│  │     • Verify results match byte-for-byte                                │ │
│  │     Result: ✓ PASS - Performance excellent                              │ │
│  │                                                                          │ │
│  │  3. Concurrency Test                                                    │ │
│  │     • Simulate 50 concurrent queries                                    │ │
│  │     • Monitor CPU, memory, connection pool                              │ │
│  │     • Check for errors, timeouts                                        │ │
│  │     Result: ✓ PASS - No errors, stable performance                      │ │
│  │                                                                          │ │
│  │  4. Failover Test                                                       │ │
│  │     • Terminate one Tundra node (simulate failure)                      │ │
│  │     • Verify Load Balancer routes to healthy nodes                      │ │
│  │     • Check query latency impact (<10% increase)                        │ │
│  │     Result: ✓ PASS - Failover seamless                                  │ │
│  │                                                                          │ │
│  │  5. E2E Test (via OCI endpoint, not DNS yet)                            │ │
│  │     • Directly hit OCI Load Balancer IP                                 │ │
│  │     • Login as test user                                                │ │
│  │     • Load dashboard                                                    │ │
│  │     • Run queries, check visualizations                                 │ │
│  │     Result: ✓ PASS - End-to-end working                                 │ │
│  │                                                                          │ │
│  │  Decision Gate: ALL tests must PASS before proceeding to cutover        │ │
│  │  If any test fails: Investigate, fix, re-test. Do NOT proceed.          │ │
│  │                                                                          │ │
│  │  State: VALIDATED                                                        │ │
│  │  Duration: 2-4 hours                                                     │ │
│  │  Success Criteria: All 5 test categories pass                           │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  STEP 1.4: FINAL SYNC & FREEZE (T-1 hour)                                   │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Objective: Ensure data is fully synced before DNS cutover              │ │
│  │                                                                          │ │
│  │  Process:                                                                │ │
│  │  1. Trigger final incremental sync                                      │ │
│  │     • Rclone sync (captures last 15 min of changes)                     │ │
│  │     • Duration: 30 seconds - 2 minutes                                  │ │
│  │                                                                          │ │
│  │  2. Verify sync lag = 0                                                 │ │
│  │     • All files in AWS S3 are in OCI Object Storage                     │ │
│  │     • Checksums match                                                   │ │
│  │     • No pending sync operations                                        │ │
│  │                                                                          │ │
│  │  3. (OPTIONAL) Enable read-only mode on AWS                             │ │
│  │     • For ultra-critical tenants only                                   │ │
│  │     • Block writes to AWS S3 for 5 minutes                              │ │
│  │     • Guarantees zero data divergence                                   │ │
│  │     • Typical: NOT needed, continuous sync is sufficient                │ │
│  │                                                                          │ │
│  │  4. Refresh OCI Tundra cluster (if needed)                              │ │
│  │     • If >30 min since last hydration, re-hydrate                       │ │
│  │     • Ensures latest data is loaded                                     │ │
│  │     • Duration: 3-8 minutes                                             │ │
│  │                                                                          │ │
│  │  State: READY FOR CUTOVER                                               │ │
│  │  Duration: 5-15 minutes                                                 │ │
│  │  Success Criteria: Data sync complete, OCI cluster refreshed            │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  STEP 1.5: DNS CUTOVER (T-0) - ZERO DOWNTIME                                │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Objective: Switch traffic from AWS to OCI with ZERO downtime           │ │
│  │                                                                          │ │
│  │  Current State (Before Cutover):                                         │ │
│  │  ┌────────────────────────────────────────────────────────────────┐    │ │
│  │  │  DNS: tenant42.domo.com                                         │    │ │
│  │  │  ├─ A Record: 52.1.2.3 (AWS ALB)  ← Currently Active           │    │ │
│  │  │  └─ TTL: 60 seconds                                             │    │ │
│  │  └────────────────────────────────────────────────────────────────┘    │ │
│  │         │                                                                │ │
│  │         ▼                                                                │ │
│  │  ┌────────────────────┐                                                 │ │
│  │  │  AWS ALB           │                                                 │ │
│  │  │  52.1.2.3          │                                                 │ │
│  │  └──────┬─────────────┘                                                 │ │
│  │         │                                                                │ │
│  │         ▼                                                                │ │
│  │  ┌────────────────────┐                                                 │ │
│  │  │  AWS Tundra        │                                                 │ │
│  │  │  Cluster           │  ← 100% of traffic                              │ │
│  │  └────────────────────┘                                                 │ │
│  │                                                                          │ │
│  │  Cutover Process:                                                        │ │
│  │  1. Update DNS A record (via Route53 or DNS provider)                   │ │
│  │     ```bash                                                              │ │
│  │     aws route53 change-resource-record-sets \                           │ │
│  │       --hosted-zone-id Z1234567890ABC \                                 │ │
│  │       --change-batch '{                                                 │ │
│  │         "Changes": [{                                                   │ │
│  │           "Action": "UPSERT",                                           │ │
│  │           "ResourceRecordSet": {                                        │ │
│  │             "Name": "tenant42.domo.com",                                │ │
│  │             "Type": "A",                                                │ │
│  │             "TTL": 60,                                                  │ │
│  │             "ResourceRecords": [{                                       │ │
│  │               "Value": "129.213.45.67"  ← OCI Load Balancer IP         │ │
│  │             }]                                                           │ │
│  │           }                                                              │ │
│  │         }]                                                               │ │
│  │       }'                                                                 │ │
│  │     ```                                                                  │ │
│  │                                                                          │ │
│  │  2. DNS propagation begins (TTL=60 seconds)                             │ │
│  │     • Users with cached DNS: Continue hitting AWS (for up to 60s)       │ │
│  │     • Users with fresh DNS lookup: Start hitting OCI                    │ │
│  │     • Gradual traffic shift over 1-2 minutes                            │ │
│  │                                                                          │ │
│  │  3. Monitor traffic split                                               │ │
│  │     • Minute 0: AWS=100%, OCI=0%                                        │ │
│  │     • Minute 1: AWS=70%, OCI=30%                                        │ │
│  │     • Minute 2: AWS=30%, OCI=70%                                        │ │
│  │     • Minute 3: AWS=5%, OCI=95%                                         │ │
│  │     • Minute 5: AWS=0%, OCI=100% ✓                                      │ │
│  │                                                                          │ │
│  │  New State (After Cutover):                                              │ │
│  │  ┌────────────────────────────────────────────────────────────────┐    │ │
│  │  │  DNS: tenant42.domo.com                                         │    │ │
│  │  │  ├─ A Record: 129.213.45.67 (OCI LB)  ← Now Active             │    │ │
│  │  │  └─ TTL: 60 seconds                                             │    │ │
│  │  └────────────────────────────────────────────────────────────────┘    │ │
│  │         │                                                                │ │
│  │         ▼                                                                │ │
│  │  ┌────────────────────┐                                                 │ │
│  │  │  OCI Load Balancer │                                                 │ │
│  │  │  129.213.45.67     │                                                 │ │
│  │  └──────┬─────────────┘                                                 │ │
│  │         │                                                                │ │
│  │         ▼                                                                │ │
│  │  ┌────────────────────┐                                                 │ │
│  │  │  OCI Tundra        │                                                 │ │
│  │  │  Cluster           │  ← 100% of traffic (NEW)                        │ │
│  │  └────────────────────┘                                                 │ │
│  │                                                                          │ │
│  │  ┌────────────────────┐                                                 │ │
│  │  │  AWS Tundra        │                                                 │ │
│  │  │  Cluster (IDLE)    │  ← Kept running for 24h (rollback safety)      │ │
│  │  └────────────────────┘                                                 │ │
│  │                                                                          │ │
│  │  ZERO DOWNTIME ACHIEVED:                                                │ │
│  │  • Users experience seamless transition                                 │ │
│  │  • In-flight queries complete on AWS                                    │ │
│  │  • New queries route to OCI                                             │ │
│  │  • No dropped connections                                               │ │
│  │  • No user action required                                              │ │
│  │                                                                          │ │
│  │  State: CUTOVER COMPLETE                                                │ │
│  │  Duration: 5 minutes (DNS propagation)                                  │ │
│  │  Success Criteria: 100% traffic on OCI, zero errors                     │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  STEP 1.6: POST-CUTOVER MONITORING (T+0 to T+24 hours)                      │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Objective: Ensure migration successful, catch issues early             │ │
│  │                                                                          │ │
│  │  Monitoring (Intensive for first 4 hours):                              │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │ │
│  │  │  Metrics to Watch (Real-time dashboards):                        │  │ │
│  │  │                                                                   │  │ │
│  │  │  Performance:                                                     │  │ │
│  │  │  • Query latency p95: Target <1000ms (vs AWS baseline 1200ms)   │  │ │
│  │  │  • Dashboard load time: Target <2s                               │  │ │
│  │  │  • API response time: Target <200ms                              │  │ │
│  │  │                                                                   │  │ │
│  │  │  Errors:                                                          │  │ │
│  │  │  • HTTP 5xx errors: Must be 0%                                   │  │ │
│  │  │  • Query failures: Target <0.1%                                  │  │ │
│  │  │  • Connection timeouts: Target <0.1%                             │  │ │
│  │  │                                                                   │  │ │
│  │  │  Resource Utilization:                                           │  │ │
│  │  │  • OCI Tundra CPU: Target <70%                                   │  │ │
│  │  │  • OCI Tundra Memory: Target <75%                                │  │ │
│  │  │  • OCI Load Balancer: Healthy backends 3/3                       │  │ │
│  │  │                                                                   │  │ │
│  │  │  User Experience:                                                 │  │ │
│  │  │  • Active users: Should match pre-migration levels               │  │ │
│  │  │  • Support tickets: Monitor for migration-related issues         │  │ │
│  │  └──────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                          │ │
│  │  Alerting (PagerDuty):                                                   │ │
│  │  • CRITICAL: Error rate >1% for >5 minutes → ROLLBACK                   │ │
│  │  • CRITICAL: Query latency >2x baseline for >10 minutes → INVESTIGATE   │ │
│  │  • WARNING: CPU >80% for >15 minutes → SCALE UP                         │ │
│  │                                                                          │ │
│  │  Success Metrics (T+4 hours checkpoint):                                │ │
│  │  ✓ Query latency ≤ AWS baseline (ideally better)                        │ │
│  │  ✓ Error rate <0.1%                                                     │ │
│  │  ✓ Zero customer complaints                                             │ │
│  │  ✓ OCI cluster stable (no crashes, restarts)                            │ │
│  │  ✓ Support tickets: No migration-related issues                         │ │
│  │                                                                          │ │
│  │  If ALL metrics pass: Declare migration SUCCESS ✓                       │ │
│  │  If ANY critical issues: Initiate ROLLBACK (see Step 1.7)               │ │
│  │                                                                          │ │
│  │  State: MONITORING                                                       │ │
│  │  Duration: 24 hours (intensive for first 4 hours)                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  STEP 1.7: ROLLBACK PROCEDURE (IF NEEDED)                                   │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Trigger: Critical issues detected (error rate >1%, major outage)       │ │
│  │                                                                          │ │
│  │  Rollback Process (5 minutes):                                          │ │
│  │  1. Revert DNS to AWS                                                   │ │
│  │     • Update A record back to AWS ALB IP (52.1.2.3)                     │ │
│  │     • TTL=60s → Traffic shifts back in 1-2 minutes                      │ │
│  │                                                                          │ │
│  │  2. AWS Tundra cluster still running (we kept it for 24h!)              │ │
│  │     • No re-hydration needed                                            │ │
│  │     • Immediately ready to receive traffic                              │ │
│  │                                                                          │ │
│  │  3. Monitor traffic shift back to AWS                                   │ │
│  │     • Minute 0: OCI=100%, AWS=0%                                        │ │
│  │     • Minute 2: OCI=50%, AWS=50%                                        │ │
│  │     • Minute 5: OCI=0%, AWS=100% ✓                                      │ │
│  │                                                                          │ │
│  │  4. Notify team & stakeholders                                          │ │
│  │     • Slack alert: "Tenant T-0042 rolled back to AWS"                   │ │
│  │     • Incident report: Root cause analysis                              │ │
│  │                                                                          │ │
│  │  5. Customer communication (if impacted)                                │ │
│  │     • Email: "Brief service optimization - now resolved"                │ │
│  │     • Do NOT mention "migration failure" unless they noticed            │ │
│  │                                                                          │ │
│  │  6. Investigate & fix OCI issue                                         │ │
│  │     • Debug logs, performance analysis                                  │ │
│  │     • Fix bug, re-test thoroughly                                       │ │
│  │     • Reschedule migration for T+48 hours                               │ │
│  │                                                                          │ │
│  │  RTO (Recovery Time Objective): 5 minutes                               │ │
│  │  RPO (Recovery Point Objective): 0 (no data loss)                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  STEP 1.8: AWS DECOMMISSIONING (T+24 hours if success)                      │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Objective: Clean up AWS resources, realize cost savings                │ │
│  │                                                                          │ │
│  │  Process:                                                                │ │
│  │  1. Confirm migration stable for 24 hours                               │ │
│  │     • Zero critical issues                                              │ │
│  │     • Customer satisfaction confirmed                                   │ │
│  │                                                                          │ │
│  │  2. Terminate AWS Tundra cluster                                        │ │
│  │     • Stop EC2 instances                                                │ │
│  │     • Delete Auto Scaling Group                                         │ │
│  │     • Release Elastic IPs                                               │ │
│  │                                                                          │ │
│  │  3. Keep AWS S3 data for 30 days (safety buffer)                        │ │
│  │     • Transition to Glacier (cold storage)                              │ │
│  │     • After 30 days: Delete AWS S3 bucket                               │ │
│  │                                                                          │ │
│  │  4. Update documentation                                                │ │
│  │     • Mark tenant as "Migrated to OCI" in database                      │ │
│  │     • Update runbooks with OCI cluster details                          │ │
│  │                                                                          │ │
│  │  State: COMPLETED                                                        │ │
│  │  Cost Savings: ~$500-2000/month per tenant (depending on size)          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  PILOT PHASE SUCCESS CRITERIA                                                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Must achieve ALL of the following:                                          │
│  ✓ 5/5 pilot tenants migrated successfully (zero rollbacks)                 │
│  ✓ Query performance ≥ AWS baseline (target: 20%+ better)                   │
│  ✓ Zero data loss or corruption                                             │
│  ✓ Customer satisfaction ≥ 8/10 (survey)                                    │
│  ✓ Zero migration-related support tickets                                   │
│  ✓ Migration process <6 hours per tenant (data sync + cutover)              │
│  ✓ Rollback procedure tested & validated (1 tenant)                         │
│  ✓ Team trained & comfortable with process                                  │
│                                                                               │
│  Decision Gate: If ALL criteria met → Proceed to Phase 2 (Wave Migration)   │
│                 If ANY criterion fails → Iterate, fix, re-pilot             │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
           PHASE 2: WAVE-BASED MIGRATION (WEEK 5-16) - ALL TENANTS
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  WAVE STRATEGY                                                                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Principle: Start with low-risk, increase complexity gradually               │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  WAVE 1: Small Tenants (<10GB) - 30% of total tenants                  │ │
│  │  • Timeline: Week 5-8 (4 weeks)                                         │ │
│  │  • Cadence: 20-30 tenants per day                                       │ │
│  │  • Migration window: 2-6 AM local time (off-peak)                       │ │
│  │  • Characteristics: Simple queries, standard features                   │ │
│  │  • Risk: LOW                                                            │ │
│  │  • Rollback likelihood: <1%                                             │ │
│  │  • Example: Tenant size 1-10GB, <100 users, basic dashboards           │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  WAVE 2: Medium Tenants (10-100GB) - 50% of total tenants              │ │
│  │  • Timeline: Week 9-12 (4 weeks)                                        │ │
│  │  • Cadence: 15-20 tenants per day                                       │ │
│  │  • Migration window: 2-6 AM local time                                  │ │
│  │  • Characteristics: Standard complexity, multiple dashboards            │ │
│  │  • Risk: MEDIUM                                                         │ │
│  │  • Rollback likelihood: <2%                                             │ │
│  │  • Example: Tenant size 10-100GB, 100-1000 users, advanced features    │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  WAVE 3: Large Tenants (100GB-1TB) - 15% of total tenants              │ │
│  │  • Timeline: Week 13-14 (2 weeks)                                       │ │
│  │  • Cadence: 5-10 tenants per day                                        │ │
│  │  • Migration window: Coordinated with customer (maintenance window)     │ │
│  │  • Characteristics: Complex queries, many integrations                  │ │
│  │  • Risk: HIGH                                                           │ │
│  │  • Rollback likelihood: <5%                                             │ │
│  │  • Example: Tenant size 100GB-1TB, 1000+ users, custom apps            │ │
│  │  • Special: Dedicated migration engineer per tenant                     │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  WAVE 4: Enterprise Tenants (>1TB) - 5% of total tenants               │ │
│  │  • Timeline: Week 15-16 (2 weeks)                                       │ │
│  │  • Cadence: 1-2 tenants per day (white-glove treatment)                │ │
│  │  • Migration window: Coordinated weeks in advance, exec stakeholders   │ │
│  │  • Characteristics: Mission-critical, SLA commitments, 24/7 operations  │ │
│  │  • Risk: VERY HIGH (revenue impact if issues)                           │ │
│  │  • Rollback likelihood: <10% (we're very cautious)                      │ │
│  │  • Example: Fortune 500 companies, >10TB data, complex integrations    │ │
│  │  • Special: Migration team + TAM + VP Eng on-call, dry-run mandatory   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  DAILY MIGRATION WORKFLOW (EXAMPLE: WAVE 1, 25 TENANTS/DAY)                 │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  T-72 hours: Batch Preparation                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  1. Select next 25 tenants from Wave 1 queue                            │ │
│  │  2. Initiate data sync for all 25 (parallel)                            │ │
│  │     • Rclone jobs run concurrently (dedicated sync nodes)               │ │
│  │     • Monitor sync progress dashboard                                   │ │
│  │  3. Send customer notifications (T-72h email)                            │ │
│  │  4. Provision OCI Tundra clusters for all 25 (Terraform batch)          │ │
│  │     • Use cluster templates for faster provisioning                     │ │
│  │  5. Hydrate clusters overnight                                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  T-24 hours: Validation                                                      │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  1. Run automated test suite for all 25 tenants (parallel)              │ │
│  │     • Data integrity: ✓ 25/25 passed                                    │ │
│  │     • Query performance: ✓ 24/25 passed (1 needed cluster upsize)      │ │
│  │     • E2E tests: ✓ 25/25 passed                                         │ │
│  │  2. Address any issues found                                            │ │
│  │  3. Send final reminder emails (T-24h)                                  │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  T-0: Migration Night (2-6 AM EST)                                           │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Staggered Cutover (avoid simultaneous DNS updates):                    │ │
│  │                                                                          │ │
│  │  2:00 AM - Batch 1 (Tenants 1-5):                                       │ │
│  │  • Final sync (5 min)                                                   │ │
│  │  • DNS cutover (5 min)                                                  │ │
│  │  • Monitor for 10 min → SUCCESS ✓                                       │ │
│  │                                                                          │ │
│  │  2:15 AM - Batch 2 (Tenants 6-10):                                      │ │
│  │  • Final sync (5 min)                                                   │ │
│  │  • DNS cutover (5 min)                                                  │ │
│  │  • Monitor for 10 min → SUCCESS ✓                                       │ │
│  │                                                                          │ │
│  │  2:30 AM - Batch 3 (Tenants 11-15):                                     │ │
│  │  • Same process... → SUCCESS ✓                                          │ │
│  │                                                                          │ │
│  │  2:45 AM - Batch 4 (Tenants 16-20):                                     │ │
│  │  • Same process... → SUCCESS ✓                                          │ │
│  │                                                                          │ │
│  │  3:00 AM - Batch 5 (Tenants 21-25):                                     │ │
│  │  • Same process... → SUCCESS ✓                                          │ │
│  │                                                                          │ │
│  │  By 3:30 AM: All 25 tenants migrated (1.5 hours total)                  │ │
│  │                                                                          │ │
│  │  3:30 AM - 6:00 AM: Intensive Monitoring                                │ │
│  │  • Watch dashboards for all 25 tenants                                  │ │
│  │  • Team on Zoom call (4-5 engineers)                                    │ │
│  │  • Ready to rollback if any issues                                      │ │
│  │                                                                          │ │
│  │  Result: 25/25 SUCCESS ✓ (typical for Wave 1)                           │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  T+4 hours (6 AM): Business Hours Begin                                      │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  • Users login (expect normal performance)                              │ │
│  │  • Support team briefed & ready                                         │ │
│  │  • Migration team transitions to normal hours monitoring                │ │
│  │  • Send success emails to customers (optional)                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  T+24 hours: Decommission AWS (if all stable)                                │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  • Automated script terminates AWS resources                            │ │
│  │  • Cost savings begin accumulating                                      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  Repeat Daily: 25 tenants/day × 4 weeks = 700 tenants (Wave 1 complete)     │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  AUTOMATION & TOOLING FOR SCALE                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Migration Orchestrator Dashboard (Web UI):                                  │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  URL: https://migration.domo-internal.com                               │ │
│  │                                                                          │ │
│  │  Features:                                                               │ │
│  │  • Tenant queue management (drag-and-drop to reorder)                   │ │
│  │  • Batch selection for daily migrations                                 │ │
│  │  • Real-time status for each tenant:                                    │ │
│  │    - 🔵 Pending (not started)                                           │ │
│  │    - 🟡 Syncing (data replication in progress)                          │ │
│  │    - 🟢 Ready (validation passed, ready for cutover)                    │ │
│  │    - 🟣 Migrating (DNS cutover in progress)                             │ │
│  │    - ✅ Completed (migration successful)                                │ │
│  │    - ⚠️  Failed (rollback triggered, needs attention)                   │ │
│  │  • One-click rollback button (per tenant)                               │ │
│  │  • Bulk operations: "Migrate batch of 25"                               │ │
│  │  • Performance charts: Compare AWS vs OCI                               │ │
│  │  • Cost savings tracker: Real-time cumulative savings                   │ │
│  │  • Slack integration: Notifications on status changes                   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  CLI Tool for Engineers:                                                     │
│  ```bash                                                                     │
│  # Start migration for a tenant                                              │
│  ./domo-migrate start --tenant T-0123                                        │
│                                                                               │
│  # Check status                                                              │
│  ./domo-migrate status --tenant T-0123                                       │
│  > Status: READY (validation passed)                                         │
│  > Data synced: Yes (lag: 0 minutes)                                         │
│  > OCI cluster: Healthy (3/3 nodes up)                                       │
│  > Tests passed: 5/5                                                         │
│  > Ready for cutover: Yes                                                    │
│                                                                               │
│  # Perform DNS cutover                                                       │
│  ./domo-migrate cutover --tenant T-0123 --confirm                            │
│  > Updating DNS... Done.                                                     │
│  > Monitoring traffic shift... 10%... 50%... 100% ✓                          │
│  > Cutover complete. Tenant now on OCI.                                      │
│                                                                               │
│  # Rollback if needed                                                        │
│  ./domo-migrate rollback --tenant T-0123 --confirm                           │
│  > Rolling back to AWS... Done in 5 minutes.                                 │
│                                                                               │
│  # Batch operations                                                          │
│  ./domo-migrate batch-cutover --wave 1 --batch 1 --count 25                 │
│  > Migrating 25 tenants in staggered fashion...                              │
│  > [Tenant 1] SUCCESS ✓                                                      │
│  > [Tenant 2] SUCCESS ✓                                                      │
│  > ...                                                                        │
│  ```                                                                          │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  HANDLING SPECIAL CASES & EDGE CASES                                         │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Edge Case 1: 24/7 Operations (No Maintenance Window)                        │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Challenge: Customer has users globally, no true "off-peak" time        │ │
│  │                                                                          │ │
│  │  Solution: Rolling Maintenance Approach                                 │ │
│  │  1. Identify lowest traffic period (least bad time)                     │ │
│  │  2. Enable "Read-Only Mode" for 2-3 minutes during cutover              │ │
│  │     • Queries allowed                                                   │ │
│  │     • Writes/ETL jobs blocked temporarily                               │ │
│  │     • Display banner: "Brief maintenance in progress"                   │ │
│  │  3. Perform DNS cutover (5 min)                                         │ │
│  │  4. Resume writes on OCI                                                │ │
│  │  5. Total impact: 2-3 minute "read-only" period                         │ │
│  │  6. Communicate clearly: "Brief read-only period" not "downtime"        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  Edge Case 2: Custom Integrations / APIs                                     │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Challenge: Customer has hard-coded AWS endpoints in their apps         │ │
│  │                                                                          │ │
│  │  Solution: API Endpoint Abstraction                                     │ │
│  │  1. Provide tenant-specific API endpoint: api.tenant42.domo.com         │ │
│  │  2. This DNS record points to whichever cloud (AWS or OCI)              │ │
│  │  3. During migration: Update DNS for api.tenant42.domo.com → OCI        │ │
│  │  4. Customer's hard-coded URL continues to work (transparent)           │ │
│  │  5. No code changes required on customer side                           │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  Edge Case 3: Real-Time Data Pipelines                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Challenge: Customer ingests data every 5 minutes, can't pause          │ │
│  │                                                                          │ │
│  │  Solution: Dual-Write Strategy (Temporary)                              │ │
│  │  1. During migration window: Write data to BOTH AWS S3 and OCI Object   │ │
│  │     Storage simultaneously                                              │ │
│  │  2. Perform DNS cutover for queries                                     │ │
│  │  3. After cutover: Stop writing to AWS S3, OCI only                     │ │
│  │  4. Total dual-write period: 5-10 minutes                               │ │
│  │  5. Zero data loss, zero pipeline interruption                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  Edge Case 4: Regulatory/Compliance Requirements                             │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Challenge: HIPAA/SOC2 tenant requires audit trail of migration         │ │
│  │                                                                          │ │
│  │  Solution: Enhanced Logging & Documentation                             │ │
│  │  1. Capture detailed logs (every action, timestamp, who)                │ │
│  │  2. Generate migration report:                                          │ │
│  │     • Data checksums (before/after)                                     │ │
│  │     • Test results (all passed)                                         │ │
│  │     • Performance metrics (AWS vs OCI)                                  │ │
│  │     • Downtime: 0 seconds                                               │ │
│  │  3. Provide report to customer for their compliance audit               │ │
│  │  4. Store logs in immutable OCI Audit service (retention 10 years)      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  Edge Case 5: Migration Failure / Rollback Scenario                          │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Example: Tenant T-0555 migrated, but query errors spike to 5%          │ │
│  │                                                                          │ │
│  │  Response (within 5 minutes):                                            │ │
│  │  1. Alert fires: "Tenant T-0555 error rate >1% for >5 min"              │ │
│  │  2. On-call engineer reviews dashboard                                  │ │
│  │  3. Decision: ROLLBACK (error rate unacceptable)                        │ │
│  │  4. Execute: `./domo-migrate rollback --tenant T-0555 --confirm`        │ │
│  │  5. DNS reverts to AWS (takes 2-3 minutes)                              │ │
│  │  6. Error rate drops to 0% (back on stable AWS)                         │ │
│  │  7. Post-incident:                                                       │ │
│  │     • Root cause: OCI Tundra cluster undersized                         │ │
│  │     • Fix: Increase OCPU from 16 to 32 (Flex shape resize, no reboot)  │ │
│  │     • Re-test: Performance now excellent                                │ │
│  │     • Reschedule migration: T+48 hours                                  │ │
│  │     • Second attempt: SUCCESS ✓                                         │ │
│  │                                                                          │ │
│  │  Customer Impact: ~10 minutes of elevated errors, then resolved         │ │
│  │  Communication: "Brief service optimization issue - now resolved"       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                PHASE 3: DECOMMISSIONING & CLOSEOUT (WEEK 17-20)
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  STEP 3.1: FINAL MONITORING PERIOD (Week 17-18)                             │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Objective: Ensure ALL tenants stable on OCI before AWS shutdown             │
│                                                                               │
│  Activities:                                                                  │
│  • Monitor all tenants for 2 weeks post-migration                            │
│  • Track key metrics: uptime, query performance, error rates                 │
│  • Proactively reach out to customers for feedback                           │
│  • Address any lingering issues                                              │
│  • Validate cost savings (compare AWS bills vs OCI bills)                    │
│                                                                               │
│  Success Criteria:                                                            │
│  ✓ 99.5%+ of tenants stable (no issues)                                     │
│  ✓ <0.5% tenants with minor issues (being addressed)                        │
│  ✓ Zero rollbacks in past 2 weeks                                           │
│  ✓ Customer satisfaction: NPS ≥70                                            │
│  ✓ Cost savings validated: 50-70% reduction vs AWS                          │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  STEP 3.2: AWS RESOURCE DECOMMISSIONING (Week 19)                           │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Process:                                                                     │
│  1. Generate inventory of all AWS resources                                  │
│     • EC2 instances (Tundra clusters)                                        │
│     • S3 buckets (tenant data)                                               │
│     • EBS volumes                                                            │
│     • Load Balancers                                                         │
│     • VPCs, subnets, security groups                                         │
│     • IAM roles, policies                                                    │
│                                                                               │
│  2. Archive critical data (30-day safety buffer)                             │
│     • Copy all S3 buckets to Glacier (cold archive)                          │
│     • Export CloudWatch logs (1 year retention)                              │
│     • Backup database snapshots                                              │
│                                                                               │
│  3. Terminate resources (staged approach)                                    │
│     Day 1: Terminate Tundra EC2 instances (largest cost item)                │
│     Day 3: Delete Load Balancers, EBS volumes                                │
│     Day 7: Transition S3 to Glacier                                          │
│     Day 30: Delete S3 buckets (after Glacier archive confirmed)              │
│                                                                               │
│  4. Update DNS records (final cleanup)                                       │
│     • Remove all AWS ALB entries                                             │
│     • Keep only OCI Load Balancer entries                                    │
│                                                                               │
│  5. Document cost savings                                                    │
│     • Generate report: AWS cost (last month) vs OCI cost (this month)        │
│     • Typical savings: $100K-500K/month (depending on scale)                 │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  STEP 3.3: POST-MIGRATION REVIEW & OPTIMIZATION (Week 20)                   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Activities:                                                                  │
│  1. Retrospective meeting                                                    │
│     • What went well: Fast migrations, zero major outages                    │
│     • What could improve: Better customer communication, faster rollbacks    │
│     • Action items: Update runbooks, improve tooling                         │
│                                                                               │
│  2. Performance analysis                                                      │
│     • Compare AWS baseline vs OCI current state                              │
│     • Query latency: 30% average improvement ✓                               │
│     • Uptime: 99.95% (better than AWS 99.9%) ✓                              │
│     • Customer satisfaction: 85% positive ✓                                  │
│                                                                               │
│  3. Cost optimization opportunities                                           │
│     • Right-size OCI Flex shapes (some tenants over-provisioned)             │
│     • Enable Auto-Tiering on all Object Storage buckets                      │
│     • Review and optimize network topology                                   │
│     • Potential additional savings: 10-15%                                   │
│                                                                               │
│  4. Documentation updates                                                     │
│     • Update all runbooks with OCI procedures                                │
│     • Create troubleshooting guides for common OCI issues                    │
│     • Document lessons learned for future migrations                         │
│                                                                               │
│  5. Team celebration 🎉                                                      │
│     • Successful migration of 1000s of tenants with zero downtime!           │
│     • Major cost savings achieved                                            │
│     • Better performance for customers                                       │
│     • Team dinner / happy hour                                               │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                      KEY SUCCESS METRICS - FINAL REPORT
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  Migration Execution:                                                         │
│  • Total tenants migrated: 2,500 (example)                                   │
│  • Migration duration: 16 weeks (on schedule)                                │
│  • Success rate: 99.2% (first attempt)                                       │
│  • Rollbacks: 0.8% (20 tenants, all re-migrated successfully)                │
│  • Average migration time per tenant: 4 hours (data sync + cutover)          │
│  • Zero prolonged downtime incidents                                          │
│                                                                               │
│  Performance Improvements:                                                    │
│  • Query latency p95: 850ms (vs 1200ms AWS) = 29% faster ✓                  │
│  • Hydration time: 6 min avg (vs 12 min AWS) = 50% faster ✓                 │
│  • Uptime: 99.95% (vs 99.9% AWS) = 0.05% better                              │
│  • Error rate: 0.05% (vs 0.1% AWS) = 50% reduction                           │
│                                                                               │
│  Cost Savings:                                                                │
│  • AWS monthly cost (before): $800K/month                                    │
│  • OCI monthly cost (after): $350K/month                                     │
│  • Savings: $450K/month = $5.4M/year ✓✓✓                                    │
│  • ROI: Migration cost $500K, payback in 1.1 months                          │
│                                                                               │
│  Customer Impact:                                                             │
│  • Customer satisfaction (NPS): 72 (industry avg: 50)                        │
│  • Support tickets related to migration: 23 (0.9% of tenants)                │
│  • Positive feedback: 85%                                                    │
│  • Churn due to migration: 0% ✓                                              │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘


================================================================================
                      LESSONS LEARNED & BEST PRACTICES
================================================================================

┌──────────────────────────────────────────────────────────────────────────────┐
│  What Worked Well:                                                            │
│  ✓ Pilot phase caught issues early (prevented widespread problems)           │
│  ✓ Wave-based approach reduced risk (didn't put all eggs in one basket)      │
│  ✓ Automated tooling scaled effectively (1 engineer could migrate 25/day)    │
│  ✓ DNS cutover provided true zero-downtime (customers didn't notice)         │
│  ✓ Keeping AWS running 24h post-migration enabled fast rollbacks             │
│  ✓ Staggered cutovers prevented DNS server overload                          │
│  ✓ OCI's Flex shapes reduced re-migration needs (resize without reboot)      │
│  ✓ Instance Principals eliminated key management headaches                   │
│                                                                               │
│  What Could Be Improved:                                                      │
│  ⚠ Customer communication could be clearer (some confusion about timing)     │
│  ⚠ Initial data sync took longer than expected for large tenants             │
│  ⚠ A few tenants needed cluster upsizing post-migration (should predict)     │
│  ⚠ Rollback procedure should be even faster (target <3 min vs 5 min)         │
│  ⚠ Better pre-migration performance testing (catch sizing issues earlier)    │
│                                                                               │
│  Recommendations for Future Migrations:                                       │
│  • Invest more in performance prediction modeling (ML-based sizing)           │
│  • Improve customer communication templates (simpler language)                │
│  • Add "migration rehearsal" feature (test cutover without actual cutover)   │
│  • Build in automatic rollback triggers (don't wait for human decision)      │
│  • Create customer self-service portal (view migration status themselves)    │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘

This migration strategy achieves true zero-downtime through:

Continuous Data Sync: AWS and OCI stay in sync until cutover
DNS-Based Cutover: Traffic shifts gradually over 1-2 minutes via TTL expiration
Dual-Write Strategy: For real-time pipelines, write to both clouds briefly
Fast Rollback: Keep AWS running 24h, enabling 5-minute rollback if issues
Wave-Based Approach: Start low-risk, build confidence before enterprise tenants
Automated Tooling: Scale to 20-30 migrations/day with minimal manual work

Total Timeline: 16-20 weeks for complete tenant base migration with zero prolonged downtime and 99%+ success rate.

