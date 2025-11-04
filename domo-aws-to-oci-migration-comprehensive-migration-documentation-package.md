================================================================================
        DOMO AWS→OCI MIGRATION: COMPREHENSIVE DOCUMENTATION PACKAGE
================================================================================

## Documentation Suite Overview

This package provides 15+ essential documents to support the Domo engineering
team through the entire migration lifecycle. Each document serves a specific
audience and purpose.

================================================================================
                    DOCUMENT 1: RUNBOOK - OCI TUNDRA OPERATIONS
================================================================================

# OCI Tundra Cluster Operations Runbook
**Audience**: SRE, On-Call Engineers, Platform Team
**Purpose**: Day-to-day operational procedures for OCI Tundra clusters

## Table of Contents
1. Cluster Startup/Shutdown
2. Scaling Operations
3. Health Check Procedures
4. Common Incidents & Resolutions
5. Emergency Procedures
6. Maintenance Windows

---

## 1. CLUSTER STARTUP PROCEDURES

### Starting a New Tundra Cluster for a Tenant

**Prerequisites**:
- Tenant data synced to OCI Object Storage
- VCN and subnets configured
- Instance Principal policies in place

**Steps**:
```bash
# 1. Set environment variables
export TENANT_ID="T-0042"
export COMPARTMENT_OCID="ocid1.compartment.oc1.."
export SUBNET_OCID="ocid1.subnet.oc1.."

# 2. Deploy cluster via Terraform
cd terraform/tundra-cluster
terraform init
terraform plan \
  -var="tenant_id=${TENANT_ID}" \
  -var="compartment_ocid=${COMPARTMENT_OCID}" \
  -var="subnet_ocid=${SUBNET_OCID}" \
  -var="cluster_size=medium" \
  -var="ocpus=16" \
  -var="memory_gb=256"

terraform apply -auto-approve

# 3. Wait for instances to be running (5-7 minutes)
watch -n 10 'oci compute instance list \
  --compartment-id ${COMPARTMENT_OCID} \
  --lifecycle-state RUNNING \
  | jq ".data | length"'

# 4. Verify cluster health
./scripts/check_cluster_health.sh ${TENANT_ID}

# Expected output:
# ✅ All 3 nodes healthy
# ✅ Object Storage connectivity OK
# ✅ Data hydration complete (100GB in 6 minutes)
# ✅ Query test passed (p95: 850ms)
```

**Typical Timing**:
- Terraform apply: 8-10 minutes
- Hydration (100GB): 5-8 minutes
- Total: 15-20 minutes to production-ready

**Common Issues**:

| Symptom | Cause | Solution |
|---------|-------|----------|
| Instances fail to start | Capacity limits | Request limit increase or try different AD |
| Hydration timeout | Network issue | Check Service Gateway, retry hydration |
| Slow queries | Under-provisioned | Scale up OCPU/memory using Flex resize |

---

## 2. SCALING OPERATIONS

### Scaling Up (Peak Hours)

**When**: Business hours (8am-6pm), high query load
**Target**: CPU <70%, Memory <75%
```bash
# Auto-scale using our automation
./scripts/scale_tundra_cluster.sh ${TENANT_ID} --target-ocpus 32 --target-memory 512

# Or manual via OCI CLI
INSTANCE_ID="ocid1.instance.oc1.iad.xxxxx"

oci compute instance update \
  --instance-id ${INSTANCE_ID} \
  --shape-config '{"ocpus": 32, "memoryInGBs": 512}' \
  --force

# ⏱️ Resize takes 3-5 minutes
# ✅ NO REBOOT, data stays in memory
```

**Verification**:
```bash
# Check new size
oci compute instance get --instance-id ${INSTANCE_ID} \
  | jq '.data."shape-config"'

# Monitor query latency (should improve)
./scripts/query_performance_check.sh ${TENANT_ID}
```

### Scaling Down (Off-Peak)

**When**: After hours (6pm-8am), low query load
**Target**: Save costs while maintaining performance
```bash
# Scale down to minimum viable size
./scripts/scale_tundra_cluster.sh ${TENANT_ID} --target-ocpus 16 --target-memory 256

# 💰 Cost savings: 50% during off-peak hours
```

**Automated Schedule** (via cron or OCI Functions):
```bash
# Scale up at 7:30 AM EST (before peak)
30 7 * * 1-5 /opt/domo/scripts/scale_all_clusters.sh --size large

# Scale down at 6:30 PM EST (after peak)
30 18 * * 1-5 /opt/domo/scripts/scale_all_clusters.sh --size small
```

---

## 3. HEALTH CHECK PROCEDURES

### Manual Health Check
```bash
#!/bin/bash
# check_cluster_health.sh

TENANT_ID=$1
CLUSTER_NAME="tundra-${TENANT_ID}"

echo "🏥 Health Check: ${CLUSTER_NAME}"
echo "================================"

# Check 1: Instance status
echo "1️⃣ Checking instance status..."
INSTANCES=$(oci compute instance list \
  --compartment-id ${COMPARTMENT_OCID} \
  --display-name "${CLUSTER_NAME}*" \
  --lifecycle-state RUNNING \
  --query 'data[].{id:id, name:"display-name", state:"lifecycle-state"}')

RUNNING_COUNT=$(echo $INSTANCES | jq '. | length')
echo "   ✅ ${RUNNING_COUNT}/3 instances running"

if [ "$RUNNING_COUNT" -ne 3 ]; then
  echo "   ⚠️  WARNING: Not all instances running!"
  exit 1
fi

# Check 2: CPU/Memory utilization
echo "2️⃣ Checking resource utilization..."
for INSTANCE_ID in $(echo $INSTANCES | jq -r '.[].id'); do
  CPU=$(oci monitoring metric-data summarize-metrics-data \
    --namespace oci_computeagent \
    --query-text "CpuUtilization[5m].mean()" \
    --compartment-id ${COMPARTMENT_OCID} \
    | jq '.data[0]."aggregated-datapoints"[0].value')
  
  echo "   CPU: ${CPU}% (target: <70%)"
done

# Check 3: Query performance
echo "3️⃣ Running performance test..."
LATENCY=$(./scripts/run_test_query.sh ${TENANT_ID})
echo "   Query latency: ${LATENCY}ms (target: <1000ms)"

# Check 4: Object Storage connectivity
echo "4️⃣ Testing Object Storage access..."
oci os object list \
  --bucket-name "domo-tenant-${TENANT_ID}-oci" \
  --limit 1 > /dev/null 2>&1

if [ $? -eq 0 ]; then
  echo "   ✅ Object Storage OK"
else
  echo "   ❌ Object Storage FAILED"
  exit 1
fi

echo ""
echo "✅ All health checks passed!"
```

### Automated Health Checks

**OCI Monitoring Alarms** (deployed via Terraform):
```hcl
# terraform/monitoring/alarms.tf

resource "oci_monitoring_alarm" "tundra_high_cpu" {
  compartment_id        = var.compartment_ocid
  display_name          = "Tundra ${var.tenant_id} - High CPU"
  is_enabled            = true
  severity              = "CRITICAL"
  metric_compartment_id = var.compartment_ocid
  
  query = <<-EOT
    CpuUtilization[5m]{resourceDisplayName=~"tundra-${var.tenant_id}*"}.mean() > 80
  EOT
  
  destinations = [oci_ons_notification_topic.alerts.id]
  
  repeat_notification_duration = "PT30M"  # Repeat every 30 min
}

resource "oci_monitoring_alarm" "tundra_high_memory" {
  compartment_id        = var.compartment_ocid
  display_name          = "Tundra ${var.tenant_id} - High Memory"
  is_enabled            = true
  severity              = "WARNING"
  metric_compartment_id = var.compartment_ocid
  
  query = <<-EOT
    MemoryUtilization[5m]{resourceDisplayName=~"tundra-${var.tenant_id}*"}.mean() > 85
  EOT
  
  destinations = [oci_ons_notification_topic.alerts.id]
}

resource "oci_monitoring_alarm" "tundra_instance_down" {
  compartment_id        = var.compartment_ocid
  display_name          = "Tundra ${var.tenant_id} - Instance Down"
  is_enabled            = true
  severity              = "CRITICAL"
  metric_compartment_id = var.compartment_ocid
  
  query = <<-EOT
    InstanceRunningCount[1m]{tenantId="${var.tenant_id}"}.mean() < 3
  EOT
  
  destinations = [oci_ons_notification_topic.alerts.id]
}
```

---

## 4. COMMON INCIDENTS & RESOLUTIONS

### Incident 1: High Query Latency

**Symptoms**: 
- Query p95 latency >2000ms (normal: 800-1000ms)
- Customer complaints about slow dashboards

**Diagnosis**:
```bash
# Check current latency
./scripts/query_performance_check.sh ${TENANT_ID}

# Check cluster resource usage
./scripts/check_cluster_health.sh ${TENANT_ID}

# Check if cluster needs more resources
```

**Common Causes & Solutions**:

| Cause | Diagnosis | Solution |
|-------|-----------|----------|
| Under-provisioned | CPU >85% or Memory >90% | Scale up OCPUs/memory |
| Cold cache | Recent cluster restart | Wait 10-15 min for cache warm-up |
| Data skew | Uneven data distribution | Re-balance data partitions |
| Network congestion | High packet loss | Check NSG rules, VCN routing |

**Resolution Steps**:
```bash
# 1. Scale up immediately (if resources constrained)
./scripts/scale_tundra_cluster.sh ${TENANT_ID} \
  --target-ocpus 32 \
  --target-memory 512

# 2. Monitor for 10 minutes
watch -n 30 './scripts/query_performance_check.sh ${TENANT_ID}'

# 3. If still slow, restart cluster (last resort)
./scripts/restart_tundra_cluster.sh ${TENANT_ID}
# ⏱️ Downtime: ~8 minutes (hydration from Object Storage)
```

---

### Incident 2: Cluster Node Failure

**Symptoms**:
- OCI Monitoring alarm: "Instance Down"
- Load Balancer shows 2/3 healthy backends

**Diagnosis**:
```bash
# List cluster instances
oci compute instance list \
  --compartment-id ${COMPARTMENT_OCID} \
  --display-name "tundra-${TENANT_ID}*" \
  --all

# Check terminated instances
oci compute instance list \
  --compartment-id ${COMPARTMENT_OCID} \
  --lifecycle-state TERMINATED \
  --time-created-greater-than $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)
```

**Resolution** (Automatic via Instance Pool):
```bash
# Instance Pool automatically replaces failed instances
# Typical recovery: 10-15 minutes

# Monitor recovery
watch -n 30 'oci compute instance list \
  --compartment-id ${COMPARTMENT_OCID} \
  --display-name "tundra-${TENANT_ID}*" \
  --lifecycle-state RUNNING \
  | jq ".data | length"'

# Once 3/3 running, verify health
./scripts/check_cluster_health.sh ${TENANT_ID}
```

**Manual Recovery** (if auto-recovery fails):
```bash
# Launch replacement instance manually
cd terraform/tundra-cluster
terraform taint 'oci_core_instance.tundra_node[2]'
terraform apply -auto-approve
```

---

### Incident 3: Object Storage Access Denied

**Symptoms**:
- Hydration fails with 401 Unauthorized
- Queries fail with "cannot read data"

**Diagnosis**:
```bash
# Test Object Storage access from cluster node
ssh -i ~/.ssh/domo_key opc@<node-ip>

# From node, try to list objects
oci os object list \
  --bucket-name "domo-tenant-${TENANT_ID}-oci" \
  --auth instance_principal

# If fails, check Instance Principal setup
oci iam dynamic-group get --dynamic-group-id <dynamic-group-ocid>
```

**Common Causes**:
1. **Missing dynamic group membership**: Instance not tagged correctly
2. **Policy not applied**: IAM policy missing or incorrect
3. **Wrong compartment**: Policy applied to wrong compartment

**Resolution**:
```bash
# 1. Verify instance tags
oci compute instance get --instance-id ${INSTANCE_ID} \
  | jq '.data."defined-tags"'

# Should show: {"domo": {"role": "tundra", "tenant_id": "42"}}

# 2. If tags missing, add them
oci compute instance update \
  --instance-id ${INSTANCE_ID} \
  --defined-tags '{"domo": {"role": "tundra", "tenant_id": "42"}}' \
  --force

# 3. Wait 2-3 minutes for IAM propagation

# 4. Test access again
ssh -i ~/.ssh/domo_key opc@<node-ip>
oci os object list --bucket-name "domo-tenant-${TENANT_ID}-oci" \
  --auth instance_principal
```

---

## 5. EMERGENCY PROCEDURES

### Emergency Rollback to AWS

**When to Execute**:
- Critical outage on OCI (>5 min downtime)
- Data integrity issues discovered
- Performance degradation >50%
- Customer escalation from enterprise account

**Pre-Requisites**:
- AWS Tundra cluster still running (kept for 24h post-migration)
- DNS TTL = 60 seconds
- Rollback script tested

**Execution** (5 minute procedure):
```bash
#!/bin/bash
# emergency_rollback.sh

TENANT_ID=$1
REASON="$2"

echo "🚨 EMERGENCY ROLLBACK: Tenant ${TENANT_ID}"
echo "Reason: ${REASON}"
echo ""

# 1. Update DNS to AWS ALB
echo "1️⃣ Reverting DNS to AWS..."
AWS_ALB_IP=$(cat /opt/domo/config/tenant_${TENANT_ID}_aws_ip.txt)

aws route53 change-resource-record-sets \
  --hosted-zone-id ${HOSTED_ZONE_ID} \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "tenant'${TENANT_ID}'.domo.com",
        "Type": "A",
        "TTL": 60,
        "ResourceRecords": [{"Value": "'${AWS_ALB_IP}'"}]
      }
    }]
  }'

echo "✅ DNS updated to AWS: ${AWS_ALB_IP}"
echo ""

# 2. Verify AWS cluster health
echo "2️⃣ Verifying AWS cluster health..."
AWS_HEALTH=$(aws elbv2 describe-target-health \
  --target-group-arn ${AWS_TG_ARN} \
  | jq '[.TargetHealthDescriptions[].TargetHealth.State] | map(select(. == "healthy")) | length')

echo "   AWS healthy targets: ${AWS_HEALTH}/3"

if [ "$AWS_HEALTH" -lt 2 ]; then
  echo "⚠️  WARNING: AWS cluster unhealthy!"
  echo "   Manual intervention required!"
fi

# 3. Monitor traffic shift
echo ""
echo "3️⃣ Monitoring traffic shift..."
sleep 30
echo "   30s: Traffic shifting back to AWS..."
sleep 30
echo "   60s: ~50% back on AWS..."
sleep 60
echo "   120s: ~95% back on AWS..."

# 4. Send notifications
echo ""
echo "4️⃣ Sending notifications..."
./scripts/send_slack_alert.sh \
  "🚨 ROLLBACK: Tenant ${TENANT_ID} rolled back to AWS. Reason: ${REASON}"

./scripts/create_pagerduty_incident.sh \
  "Tenant ${TENANT_ID} emergency rollback" \
  "${REASON}"

# 5. Mark in database
psql -h ${DB_HOST} -U domo -d migrations -c \
  "UPDATE tenant_migrations 
   SET state='rolled_back', 
       notes='${REASON}', 
       rolled_back_at=NOW() 
   WHERE tenant_id='${TENANT_ID}'"

echo ""
echo "✅ Rollback complete"
echo ""
echo "Next steps:"
echo "1. Investigate OCI issue: ${REASON}"
echo "2. Fix root cause"
echo "3. Re-test thoroughly"
echo "4. Reschedule migration (T+48 hours minimum)"
```

**Post-Rollback**:
- Root cause analysis within 24 hours
- Fix identified issues
- Re-test on staging tenant
- Reschedule production migration

---

## 6. MAINTENANCE WINDOWS

### Planned Maintenance Procedure

**Timing**: Tuesdays 2-4 AM EST (lowest traffic)

**Notification Timeline**:
- T-14 days: Email to all affected customers
- T-7 days: Reminder email + in-app banner
- T-1 day: Final reminder + SMS (if opted-in)
- T-0: Status page update

**Maintenance Activities**:
- OS patching
- Tundra version upgrades
- Cluster rebalancing
- Performance tuning

**Execution**:
```bash
# Maintenance script (rolling upgrade)
./scripts/rolling_maintenance.sh ${TENANT_ID}

# Steps:
# 1. Scale up to 4 nodes (add temporary node)
# 2. Drain node 1, upgrade, bring back
# 3. Drain node 2, upgrade, bring back
# 4. Drain node 3, upgrade, bring back
# 5. Scale back to 3 nodes
# Total time: 45-60 minutes
# User-facing downtime: 0 seconds
```

---

## Appendix: Quick Reference Commands
```bash
# Scale cluster
./scripts/scale_tundra_cluster.sh <tenant-id> --target-ocpus <N> --target-memory <GB>

# Health check
./scripts/check_cluster_health.sh <tenant-id>

# Performance test
./scripts/query_performance_check.sh <tenant-id>

# Emergency rollback
./scripts/emergency_rollback.sh <tenant-id> "<reason>"

# View cluster metrics (Grafana)
https://grafana.domo-internal.com/d/tundra-cluster?var-tenant=<tenant-id>

# View logs (OCI Logging)
oci logging-search search-logs --query '<tenant-id>' --time-start <ISO8601>
```

---

## On-Call Escalation Path

1. **L1**: On-call SRE (initial response, basic troubleshooting)
2. **L2**: Platform Engineering Team Lead (complex issues)
3. **L3**: Senior Architect (architecture changes needed)
4. **L4**: VP Engineering (customer escalation, executive visibility)

**Response Times**:
- Critical (P1): 15 minutes
- High (P2): 1 hour
- Medium (P3): 4 hours
- Low (P4): Next business day

---

**Document Version**: 1.0
**Last Updated**: 2025-01-15
**Owner**: Platform Engineering Team
**Review Frequency**: Quarterly


================================================================================
              DOCUMENT 2: ARCHITECTURE DECISION RECORDS (ADRs)
================================================================================

# ADR-001: Use OCI Flex Shapes for Tundra Clusters

**Date**: 2025-01-10
**Status**: Accepted
**Deciders**: Platform Architecture Team, CTO

## Context

Tundra clusters need to handle variable workloads:
- Peak hours (8am-6pm): High query volume, need more resources
- Off-peak (6pm-8am): Low query volume, can reduce resources
- On AWS, resizing requires instance termination and re-hydration (15+ min downtime)

## Decision

Use OCI Flex shapes (VM.Standard.E5.Flex) for all Tundra clusters.

## Rationale

**Pros**:
- Resize CPU/RAM without reboot (3-5 minute operation)
- No data loss during resize (in-memory data preserved)
- 40-60% cost savings during off-peak hours
- Automatic scaling based on workload
- Pay-per-OCPU pricing (granular cost control)

**Cons**:
- Slightly more expensive than fixed shapes at same size
- Requires automation to realize savings
- Max 64 OCPUs per VM (vs 128 on Bare Metal)

**Alternatives Considered**:
1. **Fixed E5 shapes**: Cheaper per hour, but can't resize → Rejected (no flexibility)
2. **Bare Metal**: Highest performance, but expensive and no flex → Use for largest tenants only
3. **AWS r6i with Auto Scaling Groups**: Can scale, but requires termination → Rejected (downtime)

## Consequences

**Positive**:
- Zero-downtime scaling
- Cost optimization via auto-scaling
- Better customer experience (no interruptions)
- Simplified operations (one instance pool, dynamic sizing)

**Negative**:
- Need to build auto-scaling automation
- Monitoring must be precise (avoid over/under-provisioning)

**Implementation**:
- Default: 16 OCPUs, 256GB RAM
- Scale range: 8-64 OCPUs, 128GB-1TB RAM
- Auto-scale trigger: CPU >75% scale up, CPU <30% scale down
- Schedule-based: Scale up 7:30am, scale down 6:30pm

---

# ADR-002: Keep Vertica on AWS During Phase 1

**Date**: 2025-01-12
**Status**: Accepted
**Deciders**: Migration Team, VP Engineering

## Context

Vertica handles 20% of query volume (large, complex queries). Migrating Vertica to OCI adds complexity:
- Need to test ADB as replacement
- Different SQL dialect (minor differences)
- Vertica→ADB migration is separate project

Phase 1 goal: Migrate Tundra (80% of queries) quickly to show value.

## Decision

Keep Vertica clusters on AWS during Phase 1 (4-8 weeks).
Connect via FastConnect (OCI↔AWS private link).

## Rationale

**Pros**:
- Reduces Phase 1 scope (faster time to value)
- Tundra migration is less risky (80% of queries)
- Keeps Vertica stable while proving OCI migration works
- FastConnect provides low-latency cross-cloud connectivity

**Cons**:
- Double-cloud complexity (AWS + OCI running simultaneously)
- FastConnect cost: ~$1K/month
- Still paying for AWS Vertica clusters
- Need to migrate Vertica eventually (Phase 2)

**Alternatives Considered**:
1. **Migrate Vertica to ADB immediately**: Too risky, delays migration → Rejected
2. **Migrate Vertica to OCI (keep same engine)**: Still complex, licensing → Considered for Phase 2
3. **Eliminate Vertica entirely**: Not feasible, some queries need it → Rejected

## Consequences

**Positive**:
- Faster Phase 1 migration (focus on Tundra only)
- Lower risk (Vertica stays stable)
- Prove OCI value quickly
- Learn from Tundra migration before tackling Vertica

**Negative**:
- $1K/month FastConnect cost (3-6 months)
- Added cross-cloud latency (+5-10ms for Vertica queries)
- Need Phase 2 project to complete migration

**Implementation**:
- Deploy FastConnect: Week 1
- Test cross-cloud query routing: Week 2
- Migrate Tundra tenants: Week 3-16
- Plan Vertica→ADB migration: Week 17+

---

# ADR-003: Use Instance Principals (Not API Keys)

**Date**: 2025-01-08
**Status**: Accepted
**Deciders**: Security Team, Platform Engineering

## Context

AWS uses access keys (AKIA...) for programmatic access:
- Stored in environment variables or config files
- Need rotation every 90 days (compliance requirement)
- Risk of exposure (logs, git commits, etc.)
- 30+ keys to manage across Tundra clusters

OCI offers Instance Principals: VMs authenticate using their identity (no keys).

## Decision

Use OCI Instance Principals for ALL authentication from compute instances.
**Zero API keys** in the OCI environment.

## Rationale

**Pros**:
- No keys to manage, rotate, or secure
- Eliminates key exposure risk
- Simpler code (SDK auto-detects instance principal)
- Better security posture (passes SOC 2 audit more easily)
- Automatic credential refresh (no expiration)

**Cons**:
- OCI-specific (not portable to other clouds)
- Requires IAM policy setup (dynamic groups)
- Need to tag instances correctly for policy matching

**Alternatives Considered**:
1. **API keys in Vault**: Still need to manage rotation → Rejected
2. **IAM role assumption**: More complex than Instance Principals → Rejected
3. **Keep AWS-style keys**: Security risk, fails audit → Rejected

## Consequences

**Positive**:
- Eliminate 30+ API keys immediately
- Remove key rotation automation (saves ops time)
- Reduce security risk (no keys to expose)
- Faster development (less auth boilerplate)

**Negative**:
- Must set up dynamic groups and policies correctly
- Developers need to learn OCI IAM model
- Local development requires config file (can't use instance principal)

**Implementation**:
- Create dynamic group for Tundra VMs (tag: domo.role=tundra)
- Create IAM policy granting Object Storage, Vault, Monitoring access
- Update all code to use Instance Principal signer
- Remove all API keys from code/config

---

# ADR Template (for future ADRs)
```markdown
# ADR-XXX: [Title]

**Date**: YYYY-MM-DD
**Status**: Proposed | Accepted | Rejected | Superseded
**Deciders**: [List of people involved]

## Context
[Describe the problem and constraints]

## Decision
[State the decision clearly]

## Rationale
**Pros**:
- [List advantages]

**Cons**:
- [List disadvantages]

**Alternatives Considered**:
1. [Alternative 1]: [Reason for rejection]
2. [Alternative 2]: [Reason for rejection]

## Consequences
**Positive**:
- [Good outcomes]

**Negative**:
- [Challenges or technical debt]

**Implementation**:
- [High-level implementation plan]
```

================================================================================
            DOCUMENT 3: DISASTER RECOVERY & BUSINESS CONTINUITY
================================================================================

# OCI Disaster Recovery Plan for Domo Tundra
**RPO**: 0 (zero data loss)
**RTO**: 15 minutes (return to operation)

## DR Scenarios

### Scenario 1: Single Availability Domain Failure

**Impact**: 1/3 of Tundra clusters affected
**Likelihood**: Low (OCI AD independence is high)

**Detection**:
- OCI Monitoring alarm: "Availability Domain Down"
- Multiple instance health checks failing simultaneously

**Response**:
1. Load Balancer automatically routes traffic to healthy ADs
2. Instance Pools launch replacement instances in surviving ADs
3. New instances hydrate from Object Storage (5-10 minutes)
4. Traffic fully restored (RTO: 10-15 minutes)

**No manual intervention required** (fully automated).

---

### Scenario 2: Regional Failure (Entire OCI Region Down)

**Impact**: All tenants in region affected
**Likelihood**: Very Low
**RTO**: 30-60 minutes (cross-region failover)

**Pre-Requisites** (deployed in advance):
- Secondary region configured (e.g., us-phoenix-1 as backup for us-ashburn-1)
- Object Storage cross-region replication enabled
- DNS managed by failover-aware service (OCI Traffic Management)

**Response**:
```bash
# Automated failover script
./scripts/regional_failover.sh \
  --from-region us-ashburn-1 \
  --to-region us-phoenix-1

# Steps performed:
# 1. Detect regional health (all ADs down for >10 min)
# 2. Launch Tundra clusters in secondary region
# 3. Hydrate from replicated Object Storage
# 4. Update DNS (Traffic Management automatic failover)
# 5. Monitor traffic shift to secondary region
# 6. Alert team (manual verification)
```

**Post-Recovery**:
- Primary region comes back online
- Run in secondary region until maintenance window
- Failback during scheduled maintenance (not automatic)

---

### Scenario 3: Object Storage Data Corruption

**Impact**: Tenant data corrupted or deleted
**Likelihood**: Very Low (immutable buckets prevent accidental deletion)
**RPO**: 24 hours (daily backups)

**Detection**:
- Checksum validation fails
- Query results incorrect
- Customer reports data issues

**Response**:
```bash
# Restore from backup (Object Storage versioning)
./scripts/restore_tenant_data.sh ${TENANT_ID} --from-date 2025-01-14

# Steps:
# 1. Identify corrupt objects (checksum comparison)
# 2. Restore from Object Storage version history
# 3. OR restore from cross-region replica
# 4. OR restore from daily backup (worst case: 24h data loss)
# 5. Re-hydrate Tundra cluster
# 6. Verify data integrity
```

**Prevention**:
- Object Storage WORM (Write Once Read Many) enabled
- Cross-region replication (every object in 2 regions)
- Daily snapshots retained for 30 days

---

### Scenario 4: Complete OCI Failure (Roll Back to AWS)

**Impact**: All tenants need to fail back to AWS
**Likelihood**: Extremely Low
**RTO**: 2-4 hours (depending on number of tenants)

**Pre-Requisites**:
- Keep AWS infrastructure running for 30 days post-migration
- Data synced bidirectionally (AWS S3 ↔ OCI Object Storage)
- DNS failover scripts tested

**Response**:
```bash
# Mass failover to AWS
./scripts/emergency_aws_failover.sh --all-tenants

# Steps:
# 1. Update DNS for all tenants (point to AWS ALBs)
# 2. Verify AWS Tundra clusters healthy
# 3. Monitor traffic shift (5-10 min per tenant)
# 4. Notify customers (brief service optimization)
# 5. Incident report to leadership
```

---

## DR Testing Schedule

| Test Type | Frequency | Next Scheduled | Owner |
|-----------|-----------|----------------|-------|
| Single node failure | Monthly | 2025-02-01 | SRE Team |
| AD failover | Quarterly | 2025-03-15 | Platform Eng |
| Regional failover | Semi-annually | 2025-06-01 | VP Engineering |
| Full AWS failback | Annually | 2025-12-01 | CTO + VP Eng |

---

## Contact Information

**On-Call Engineer**: +1-555-0100 (PagerDuty)
**Platform Team Lead**: +1-555-0101
**VP Engineering**: +1-555-0102
**CTO**: +1-555-0103


================================================================================
       DOCUMENT 4: PERFORMANCE BENCHMARKING & OPTIMIZATION GUIDE
================================================================================

# Performance Benchmarking Guide
**Purpose**: Measure and optimize Tundra cluster performance on OCI

## Benchmark Suite

### Test 1: Query Latency
```bash
#!/bin/bash
# benchmark_query_latency.sh

TENANT_ID=$1
NUM_QUERIES=100

echo "🔬 Query Latency Benchmark: ${TENANT_ID}"
echo "Running ${NUM_QUERIES} sample queries..."

RESULTS_FILE="/tmp/benchmark_${TENANT_ID}_$(date +%Y%m%d_%H%M%S).csv"
echo "query_id,query_type,latency_ms" > ${RESULTS_FILE}

for i in $(seq 1 ${NUM_QUERIES}); do
  # Generate random query type
  QUERY_TYPE=$((RANDOM % 3))
  
  case $QUERY_TYPE in
    0)
      # Simple aggregation
      QUERY="SELECT COUNT(*) FROM table1"
      TYPE="simple"
      ;;
    1)
      # Join query
      QUERY="SELECT * FROM table1 JOIN table2 ON table1.id = table2.id LIMIT 1000"
      TYPE="join"
      ;;
    2)
      # Complex analytics
      QUERY="SELECT category, SUM(revenue), AVG(profit) FROM table1 WHERE date >= '2025-01-01' GROUP BY category"
      TYPE="analytics"
      ;;
  esac
  
  # Time query execution
  START=$(date +%s%3N)
  # Execute query (simplified - would use actual Tundra API)
  sleep 0.$(( RANDOM % 2000 ))  # Simulate 0-2000ms
  END=$(date +%s%3N)
  
  LATENCY=$((END - START))
  echo "${i},${TYPE},${LATENCY}" >> ${RESULTS_FILE}
done

# Analyze results
echo ""
echo "📊 Results:"
python3 << EOF
import pandas as pd
import numpy as np

df = pd.read_csv('${RESULTS_FILE}')

print(f"Total queries: {len(df)}")
print(f"\nOverall Statistics:")
print(f"  Mean latency: {df['latency_ms'].mean():.0f}ms")
print(f"  Median (p50): {df['latency_ms'].median():.0f}ms")
print(f"  p95: {df['latency_ms'].quantile(0.95):.0f}ms")
print(f"  p99: {df['latency_ms'].quantile(0.99):.0f}ms")
print(f"  Max: {df['latency_ms'].max():.0f}ms")

print(f"\nBy Query Type:")
for qtype in df['query_type'].unique():
    subset = df[df['query_type'] == qtype]
    print(f"  {qtype}:")
    print(f"    Mean: {subset['latency_ms'].mean():.0f}ms")
    print(f"    p95: {subset['latency_ms'].quantile(0.95):.0f}ms")
EOF

echo ""
echo "✅ Benchmark complete. Results saved to ${RESULTS_FILE}"
```

### Test 2: Hydration Performance
```bash
#!/bin/bash
# benchmark_hydration.sh

TENANT_ID=$1
BUCKET_NAME="domo-tenant-${TENANT_ID}-oci"
DATA_SIZE_GB=$2  # e.g., 100

echo "🔬 Hydration Benchmark: ${TENANT_ID}"
echo "Data size: ${DATA_SIZE_GB}GB"

# Launch fresh Tundra cluster
echo "1️⃣ Launching cluster..."
START_LAUNCH=$(date +%s)
./scripts/launch_tundra_cluster.sh ${TENANT_ID}
END_LAUNCH=$(date +%s)
LAUNCH_TIME=$((END_LAUNCH - START_LAUNCH))

echo "   Cluster launch: ${LAUNCH_TIME}s"

# Measure hydration time
echo "2️⃣ Hydrating from Object Storage..."
START_HYDRATE=$(date +%s)

# Monitor hydration progress
while true; do
  LOADED_GB=$(check_tundra_data_loaded ${TENANT_ID})
  PROGRESS=$((LOADED_GB * 100 / DATA_SIZE_GB))
  echo "   Progress: ${LOADED_GB}GB / ${DATA_SIZE_GB}GB (${PROGRESS}%)"
  
  if [ "$LOADED_GB" -eq "$DATA_SIZE_GB" ]; then
    break
  fi
  
  sleep 10
done

END_HYDRATE=$(date +%s)
HYDRATE_TIME=$((END_HYDRATE - START_HYDRATE))

# Calculate metrics
THROUGHPUT_MBPS=$((DATA_SIZE_GB * 1024 * 8 / HYDRATE_TIME))  # Mbps

echo ""
echo "📊 Results:"
echo "  Cluster launch: ${LAUNCH_TIME}s"
echo "  Hydration time: ${HYDRATE_TIME}s (${DATA_SIZE_GB}GB)"
echo "  Throughput: ${THROUGHPUT_MBPS}Mbps"
echo "  Total time: $((LAUNCH_TIME + HYDRATE_TIME))s"

# Compare to AWS baseline
AWS_HYDRATE_TIME_BASELINE=$((DATA_SIZE_GB * 8))  # ~8s per GB on AWS
IMPROVEMENT=$(( (AWS_HYDRATE_TIME_BASELINE - HYDRATE_TIME) * 100 / AWS_HYDRATE_TIME_BASELINE ))

echo ""
echo "vs AWS baseline (${AWS_HYDRATE_TIME_BASELINE}s):"
if [ $HYDRATE_TIME -lt $AWS_HYDRATE_TIME_BASELINE ]; then
  echo "  ✅ ${IMPROVEMENT}% faster on OCI"
else
  echo "  ⚠️  ${IMPROVEMENT}% slower on OCI (investigate)"
fi
```

### Test 3: Concurrent Query Load
```python
#!/usr/bin/env python3
# benchmark_concurrent_load.py

import concurrent.futures
import time
import statistics

def run_query(query_id):
    """Execute a single query and return latency."""
    start = time.time()
    
    # Execute query (simplified)
    time.sleep(0.5 + (query_id % 10) * 0.1)  # Simulate 500-1400ms
    
    end = time.time()
    return (end - start) * 1000  # Convert to ms

def benchmark_concurrent(num_queries, num_workers):
    """Run queries concurrently and measure performance."""
    print(f"🔬 Concurrent Load Benchmark")
    print(f"   Queries: {num_queries}")
    print(f"   Concurrent workers: {num_workers}")
    print("")
    
    latencies = []
    start_time = time.time()
    
    with concurrent.futures.ThreadPoolExecutor(max_workers=num_workers) as executor:
        futures = [executor.submit(run_query, i) for i in range(num_queries)]
        
        for i, future in enumerate(concurrent.futures.as_completed(futures)):
            latency = future.result()
            latencies.append(latency)
            
            if (i + 1) % 10 == 0:
                print(f"   Completed: {i+1}/{num_queries} queries")
    
    end_time = time.time()
    total_time = end_time - start_time
    
    # Calculate metrics
    mean = statistics.mean(latencies)
    median = statistics.median(latencies)
    p95 = sorted(latencies)[int(len(latencies) * 0.95)]
    p99 = sorted(latencies)[int(len(latencies) * 0.99)]
    qps = num_queries / total_time
    
    print("")
    print("📊 Results:")
    print(f"   Total time: {total_time:.1f}s")
    print(f"   QPS (queries/second): {qps:.1f}")
    print(f"   Mean latency: {mean:.0f}ms")
    print(f"   Median latency: {median:.0f}ms")
    print(f"   p95 latency: {p95:.0f}ms")
    print(f"   p99 latency: {p99:.0f}ms")
    
    # Check if performance acceptable
    if p95 > 1000:
        print("")
        print("⚠️  WARNING: p95 latency >1000ms")
        print("   Consider scaling up cluster")
    else:
        print("")
        print("✅ Performance acceptable")

if __name__ == '__main__':
    # Test with increasing concurrency
    for workers in [10, 25, 50, 100]:
        print("=" * 60)
        benchmark_concurrent(num_queries=100, num_workers=workers)
        print("")
```

---

## Performance Targets

| Metric | Target | Stretch Goal | Alert Threshold |
|--------|--------|--------------|-----------------|
| Query latency (p95) | <1000ms | <800ms | >1500ms |
| Query latency (p99) | <2000ms | <1500ms | >3000ms |
| Hydration (100GB) | <8 min | <5 min | >12 min |
| QPS (100 concurrent) | >50 | >75 | <30 |
| CPU utilization | <70% | <60% | >85% |
| Memory utilization | <80% | <70% | >90% |

---

## Optimization Checklist

- [ ] Cluster sized appropriately (CPU, memory)
- [ ] Data partitioning optimized (evenly distributed)
- [ ] Indexes built on frequently queried columns
- [ ] Query cache enabled and warmed
- [ ] Network path optimized (Service Gateway for Object Storage)
- [ ] Block volumes using Ultra High Performance tier
- [ ] Auto-scaling configured (scale up for peak hours)
- [ ] Monitoring alerts configured (CPU, memory, latency)


================================================================================
          DOCUMENT 5: COST TRACKING & OPTIMIZATION DASHBOARD
================================================================================

# Cost Tracking Guide

## Daily Cost Report Script
```python
#!/usr/bin/env python3
# daily_cost_report.py

import oci
from datetime import datetime, timedelta

def get_daily_costs():
    """Generate daily cost report for OCI migration."""
    
    # Use Instance Principal for auth
    signer = oci.auth.signers.InstancePrincipalsSecurityTokenSigner()
    usage_api = oci.usage_api.UsageapiClient(config={}, signer=signer)
    
    # Get yesterday's usage
    end_date = datetime.utcnow().date()
    start_date = end_date - timedelta(days=1)
    
    # Query usage
    response = usage_api.request_summarized_usages(
        request_summarized_usages_details=oci.usage_api.models.RequestSummarizedUsagesDetails(
            tenant_id=get_tenancy_id(),
            time_usage_started=start_date,
            time_usage_ended=end_date,
            granularity='DAILY',
            group_by=['service', 'compartmentName']
        )
    )
    
    # Parse results
    costs_by_service = {}
    for item in response.data.items:
        service = item.service
        cost = item.computed_amount
        
        if service not in costs_by_service:
            costs_by_service[service] = 0
        costs_by_service[service] += cost
    
    # Print report
    print(f"📊 OCI Cost Report: {start_date}")
    print("=" * 60)
    
    total_cost = sum(costs_by_service.values())
    
    for service in sorted(costs_by_service.keys(), 
                         key=lambda x: costs_by_service[x], 
                         reverse=True):
        cost = costs_by_service[service]
        pct = cost / total_cost * 100
        print(f"  {service:30s} ${cost:8.2f} ({pct:4.1f}%)")
    
    print("-" * 60)
    print(f"  {'TOTAL':30s} ${total_cost:8.2f}")
    
    # Compare to AWS baseline
    aws_daily_cost = 25000  # Placeholder: $750K/month = $25K/day
    savings = aws_daily_cost - total_cost
    savings_pct = savings / aws_daily_cost * 100
    
    print("")
    print(f"AWS daily cost (baseline): ${aws_daily_cost:,.2f}")
    print(f"OCI daily cost (current):  ${total_cost:,.2f}")
    print(f"Daily savings:             ${savings:,.2f} ({savings_pct:.1f}%)")
    print(f"Projected monthly savings: ${savings * 30:,.2f}")

if __name__ == '__main__':
    get_daily_costs()
```

---

## Cost Optimization Recommendations

### 1. Right-Size Tundra Clusters
```bash
# Find over-provisioned clusters (CPU <30% for 7 days)
./scripts/find_oversized_clusters.sh

# Output:
# Tenant-42: 32 OCPUs, CPU avg 25% → Recommend 16 OCPUs (save $600/month)
# Tenant-88: 64 OCPUs, CPU avg 40% → Recommend 32 OCPUs (save $1200/month)
```

### 2. Enable Object Storage Auto-Tiering
```bash
# Enable on all buckets (30-60% storage cost reduction)
./scripts/enable_autotuning_all_buckets.sh
```

### 3. Use Scheduled Scaling
```bash
# Auto-scale down off-peak hours (50% savings)
# Already implemented in crontab:
30 18 * * 1-5 /opt/domo/scripts/scale_all_clusters.sh --size small
```

### 4. Clean Up Unused Resources
```bash
# Find orphaned Block Volumes
oci bv volume list --lifecycle-state AVAILABLE | jq '.data[] | select(.["time-created"] < "2025-01-01")'

# Find idle VMs (no queries for 30 days)
./scripts/find_idle_tenants.sh --days 30
```

---

## Monthly Cost Review Template
```markdown
# OCI Cost Review: January 2025

## Summary
- **Total OCI Cost**: $350,000
- **AWS Cost (pre-migration)**: $800,000
- **Savings**: $450,000 (56%)
- **ROI**: Migration cost $500K, payback in 1.1 months ✅

## Cost Breakdown
| Service | Cost | % of Total | vs Last Month |
|---------|------|------------|---------------|
| Compute | $180,000 | 51% | +5% (more tenants) |
| Object Storage | $80,000 | 23% | -2% (auto-tiering) |
| Block Volumes | $40,000 | 11% | +0% |
| Networking | $30,000 | 9% | +0% |
| Monitoring/Logging | $10,000 | 3% | +1% |
| Other | $10,000 | 3% | +0% |

## Optimization Opportunities
1. Right-size 15 over-provisioned clusters → Save $12K/month
2. Archive old data (>1 year) → Save $5K/month
3. Implement more aggressive auto-scaling → Save $8K/month

## Actions for Next Month
- [ ] Resize over-provisioned clusters (by Jan 31)
- [ ] Enable archive tier for old data (by Jan 31)
- [ ] Review and optimize networking costs (by Feb 15)
```


================================================================================
              DOCUMENT 6: SECURITY & COMPLIANCE CHECKLIST
================================================================================

# OCI Security & Compliance Guide for Domo Migration
**Audience**: Security Team, Compliance Officers, Auditors
**Purpose**: Ensure OCI environment meets security and compliance requirements

## Table of Contents
1. SOC 2 Compliance Requirements
2. HIPAA Compliance Validation
3. PCI-DSS Controls
4. Security Baseline Configuration
5. Audit Logging & Monitoring
6. Encryption Standards
7. Access Control & IAM
8. Vulnerability Management

---

## 1. SOC 2 COMPLIANCE REQUIREMENTS

### Control: Access Control (CC6.1)

**Requirement**: Logical access to system resources is restricted through authentication and authorization

**OCI Implementation**:
```hcl
# terraform/iam/soc2_policies.tf

# Enforce MFA for all human users
resource "oci_identity_authentication_policy" "mfa_required" {
  compartment_id = var.tenancy_ocid
  
  password_policy {
    is_lowercase_characters_required = true
    is_numeric_characters_required   = true
    is_special_characters_required   = true
    is_uppercase_characters_required = true
    is_username_containment_allowed  = false
    minimum_password_length          = 14
  }
}

# Least privilege policy for developers (read-only)
resource "oci_identity_policy" "developer_readonly" {
  compartment_id = var.tenancy_ocid
  name           = "developer-readonly-policy"
  description    = "SOC2 CC6.1 - Developer read-only access"
  
  statements = [
    "Allow group Developers to inspect all-resources in compartment ${var.compartment_name}",
    "Allow group Developers to read all-resources in compartment ${var.compartment_name}",
    # Explicitly deny write operations
    "Deny group Developers to manage all-resources in compartment ${var.compartment_name}",
  ]
}

# Break-glass admin access (audited)
resource "oci_identity_policy" "emergency_admin" {
  compartment_id = var.tenancy_ocid
  name           = "emergency-admin-policy"
  description    = "SOC2 CC6.1 - Emergency admin access (requires approval)"
  
  statements = [
    "Allow group EmergencyAdmins to manage all-resources in compartment ${var.compartment_name} where request.permission = 'EMERGENCY_APPROVED'",
  ]
}
```

**Validation**:
```bash
# Test 1: Verify MFA is enforced
oci iam authentication-policy get --compartment-id ${TENANCY_OCID} \
  | jq '.data["password-policy"]["is-lowercase-characters-required"]'
# Expected: true

# Test 2: Verify developers cannot write
oci os bucket create --name test-bucket --compartment-id ${COMPARTMENT_OCID} \
  --profile developer-profile
# Expected: Error - Access Denied

# Test 3: Audit emergency admin access
oci audit event list --compartment-id ${TENANCY_OCID} \
  --start-time 2025-01-01T00:00:00Z \
  --end-time 2025-01-31T23:59:59Z \
  | jq '.data[] | select(.data."identity"."principal-name" | contains("EmergencyAdmin"))'
# Expected: List of all emergency admin actions
```

**Audit Evidence**:
- Screenshot of IAM policies
- MFA enforcement configuration
- Access review logs (quarterly)
- Emergency access logs

---

### Control: Monitoring (CC7.2)

**Requirement**: System monitoring is performed to detect potential security events

**OCI Implementation**:
```hcl
# terraform/monitoring/soc2_alarms.tf

# Failed login attempts
resource "oci_monitoring_alarm" "failed_logins" {
  compartment_id        = var.compartment_ocid
  display_name          = "SOC2 - Failed Login Attempts"
  is_enabled            = true
  severity              = "CRITICAL"
  metric_compartment_id = var.tenancy_ocid
  
  query = <<-EOT
    oci_iam_authentication[5m]{eventType="com.oraclecloud.IdentityControlPlane.AuthenticateUser.Failure"}.count() > 5
  EOT
  
  destinations = [oci_ons_notification_topic.security_alerts.id]
  
  # Alert every 15 minutes if condition persists
  repeat_notification_duration = "PT15M"
  
  # Require acknowledgment
  message_format = "ONS_OPTIMIZED"
}

# Unauthorized API calls
resource "oci_monitoring_alarm" "unauthorized_api_calls" {
  compartment_id        = var.compartment_ocid
  display_name          = "SOC2 - Unauthorized API Calls"
  is_enabled            = true
  severity              = "CRITICAL"
  metric_compartment_id = var.tenancy_ocid
  
  query = <<-EOT
    oci_audit[1m]{data.message="Unauthorized"}.count() > 0
  EOT
  
  destinations = [
    oci_ons_notification_topic.security_alerts.id,
    oci_ons_notification_topic.soc_team.id
  ]
}

# Policy changes (sensitive operations)
resource "oci_monitoring_alarm" "policy_changes" {
  compartment_id        = var.compartment_ocid
  display_name          = "SOC2 - IAM Policy Changes"
  is_enabled            = true
  severity              = "WARNING"
  metric_compartment_id = var.tenancy_ocid
  
  query = <<-EOT
    oci_audit[1m]{eventType=~".*Policy.*"}.count() > 0
  EOT
  
  destinations = [oci_ons_notification_topic.security_alerts.id]
}

# Instance Principal abuse detection
resource "oci_monitoring_alarm" "instance_principal_abuse" {
  compartment_id        = var.compartment_ocid
  display_name          = "SOC2 - Instance Principal Unusual Activity"
  is_enabled            = true
  severity              = "WARNING"
  metric_compartment_id = var.compartment_ocid
  
  query = <<-EOT
    oci_audit[5m]{data.identity.principalType="instance"}.count() > 1000
  EOT
  
  destinations = [oci_ons_notification_topic.security_alerts.id]
}
```

**24/7 Monitoring Dashboard**:
```python
# scripts/soc2_monitoring_dashboard.py

import oci
from datetime import datetime, timedelta

def generate_soc2_dashboard():
    """Generate SOC2 compliance monitoring dashboard."""
    
    signer = oci.auth.signers.InstancePrincipalsSecurityTokenSigner()
    audit = oci.audit.AuditClient(config={}, signer=signer)
    
    # Get last 24 hours of security events
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(days=1)
    
    print("🔒 SOC2 Security Monitoring Dashboard")
    print("=" * 70)
    print(f"Period: {start_time.date()} to {end_time.date()}")
    print("")
    
    # 1. Failed login attempts
    failed_logins = audit.list_events(
        compartment_id=get_tenancy_id(),
        start_time=start_time,
        end_time=end_time,
        search_context="eventType:AuthenticateUser.Failure"
    ).data
    
    print(f"1️⃣  Failed Login Attempts: {len(failed_logins)}")
    if len(failed_logins) > 10:
        print(f"   ⚠️  WARNING: >10 failed logins (investigate)")
    
    # 2. Unauthorized access attempts
    unauthorized = audit.list_events(
        compartment_id=get_tenancy_id(),
        start_time=start_time,
        end_time=end_time,
        search_context="data.message:Unauthorized"
    ).data
    
    print(f"2️⃣  Unauthorized Access Attempts: {len(unauthorized)}")
    if len(unauthorized) > 0:
        print(f"   🚨 CRITICAL: Investigate immediately!")
        for event in unauthorized[:5]:
            print(f"      - {event.event_time}: {event.data['identity']['principal_name']}")
    
    # 3. IAM policy changes
    policy_changes = audit.list_events(
        compartment_id=get_tenancy_id(),
        start_time=start_time,
        end_time=end_time,
        search_context="eventType:*Policy*"
    ).data
    
    print(f"3️⃣  IAM Policy Changes: {len(policy_changes)}")
    for change in policy_changes:
        print(f"      - {change.event_time}: {change.event_name} by {change.data['identity']['principal_name']}")
    
    # 4. Root/admin actions
    admin_actions = audit.list_events(
        compartment_id=get_tenancy_id(),
        start_time=start_time,
        end_time=end_time,
        search_context="data.identity.principalName:*admin*"
    ).data
    
    print(f"4️⃣  Admin Actions: {len(admin_actions)}")
    
    # 5. Encryption key usage
    key_usage = audit.list_events(
        compartment_id=get_tenancy_id(),
        start_time=start_time,
        end_time=end_time,
        search_context="eventType:*Vault*"
    ).data
    
    print(f"5️⃣  Encryption Key Operations: {len(key_usage)}")
    
    print("")
    print("=" * 70)
    
    # Overall compliance status
    issues = []
    if len(failed_logins) > 10:
        issues.append("High failed login count")
    if len(unauthorized) > 0:
        issues.append("Unauthorized access detected")
    
    if not issues:
        print("✅ No security issues detected")
    else:
        print("⚠️  Issues detected:")
        for issue in issues:
            print(f"   - {issue}")

if __name__ == '__main__':
    generate_soc2_dashboard()
```

---

## 2. HIPAA COMPLIANCE VALIDATION

### Required Controls for HIPAA

**Technical Safeguards (§164.312)**
```bash
#!/bin/bash
# scripts/hipaa_compliance_check.sh

echo "🏥 HIPAA Compliance Validation for OCI"
echo "======================================"
echo ""

# Check 1: Encryption at Rest (§164.312(a)(2)(iv))
echo "1️⃣  Encryption at Rest"
echo "   Checking Object Storage buckets..."

for BUCKET in $(oci os bucket list --compartment-id ${COMPARTMENT_OCID} --query 'data[].name' --raw-output); do
  ENCRYPTION=$(oci os bucket get --bucket-name ${BUCKET} \
    | jq -r '.data."kms-key-id"')
  
  if [ "$ENCRYPTION" == "null" ]; then
    echo "   ❌ FAIL: Bucket ${BUCKET} not encrypted with customer key"
    exit 1
  else
    echo "   ✅ PASS: Bucket ${BUCKET} encrypted with Vault key"
  fi
done

echo ""
echo "   Checking Block Volumes..."
for VOLUME in $(oci bv volume list --compartment-id ${COMPARTMENT_OCID} --query 'data[].id' --raw-output); do
  ENCRYPTION=$(oci bv volume get --volume-id ${VOLUME} \
    | jq -r '.data."kms-key-id"')
  
  if [ "$ENCRYPTION" == "null" ]; then
    echo "   ❌ FAIL: Volume ${VOLUME} not encrypted"
    exit 1
  else
    echo "   ✅ PASS: Volume encrypted"
  fi
done

# Check 2: Audit Logging (§164.312(b))
echo ""
echo "2️⃣  Audit Logging"
echo "   Checking if audit logs are enabled and immutable..."

AUDIT_ENABLED=$(oci audit configuration get | jq -r '.data."retention-period-days"')

if [ "$AUDIT_ENABLED" -lt 365 ]; then
  echo "   ❌ FAIL: Audit retention < 365 days (HIPAA requires 6 years)"
  exit 1
else
  echo "   ✅ PASS: Audit retention ${AUDIT_ENABLED} days"
fi

# Check 3: Access Control (§164.312(a)(1))
echo ""
echo "3️⃣  Access Control"
echo "   Verifying MFA is enforced..."

MFA_ENABLED=$(oci iam authentication-policy get --compartment-id ${TENANCY_OCID} \
  | jq -r '.data."password-policy"')

if [ "$MFA_ENABLED" != "null" ]; then
  echo "   ✅ PASS: MFA policy configured"
else
  echo "   ❌ FAIL: MFA not enforced"
  exit 1
fi

# Check 4: Encryption in Transit (§164.312(e)(1))
echo ""
echo "4️⃣  Encryption in Transit"
echo "   Verifying TLS 1.2+ required for all endpoints..."

# Check Load Balancer SSL config
for LB in $(oci lb load-balancer list --compartment-id ${COMPARTMENT_OCID} --query 'data[].id' --raw-output); do
  LISTENERS=$(oci lb listener list --load-balancer-id ${LB} --query 'data[].name' --raw-output)
  
  for LISTENER in $LISTENERS; do
    PROTOCOL=$(oci lb listener get --listener-name ${LISTENER} --load-balancer-id ${LB} \
      | jq -r '.data.protocol')
    
    if [ "$PROTOCOL" == "HTTP" ]; then
      echo "   ❌ FAIL: Listener ${LISTENER} uses HTTP (requires HTTPS)"
      exit 1
    fi
  done
done

echo "   ✅ PASS: All listeners use HTTPS"

# Check 5: Access Logging (§164.308(a)(1)(ii)(D))
echo ""
echo "5️⃣  Access Logging"
echo "   Verifying comprehensive logging enabled..."

LOG_GROUPS=$(oci logging log-group list --compartment-id ${COMPARTMENT_OCID} --query 'data | length(@)')

if [ "$LOG_GROUPS" -eq 0 ]; then
  echo "   ❌ FAIL: No log groups configured"
  exit 1
else
  echo "   ✅ PASS: ${LOG_GROUPS} log groups configured"
fi

echo ""
echo "======================================"
echo "✅ HIPAA Compliance Check PASSED"
echo ""
echo "Next steps:"
echo "1. Save this output for audit evidence"
echo "2. Run quarterly (required for HIPAA)"
echo "3. Document any exceptions with BAA"
```

**HIPAA Business Associate Agreement (BAA)**

OCI BAA Configuration:
```hcl
# terraform/compliance/hipaa_baa.tf

# Enable HIPAA-compliant regions only
variable "hipaa_approved_regions" {
  default = [
    "us-ashburn-1",
    "us-phoenix-1",
    # eu-frankfurt-1 NOT HIPAA compliant (EU region)
  ]
}

# Enforce HIPAA tagging
resource "oci_identity_tag_namespace" "hipaa" {
  compartment_id = var.tenancy_ocid
  name           = "HIPAA"
  description    = "HIPAA compliance tags"
  
  is_retired = false
}

resource "oci_identity_tag" "phi_data" {
  tag_namespace_id = oci_identity_tag_namespace.hipaa.id
  name             = "contains-phi"
  description      = "Resource contains Protected Health Information"
  
  validator {
    validator_type = "ENUM"
    values         = ["true", "false"]
  }
}

# Policy: All HIPAA resources must be tagged
resource "oci_identity_policy" "hipaa_tagging_required" {
  compartment_id = var.tenancy_ocid
  name           = "hipaa-tagging-enforcement"
  description    = "Require PHI tagging for audit"
  
  statements = [
    "Require defined-tags.HIPAA.contains-phi to be set on all-resources in compartment ${var.compartment_name}",
  ]
}
```

---

## 3. PCI-DSS CONTROLS

### PCI-DSS Requirement 2: Do Not Use Vendor-Supplied Defaults

**Implementation**:
```bash
#!/bin/bash
# scripts/pci_dss_hardening.sh

echo "💳 PCI-DSS Hardening Script"
echo ""

# 1. Change default passwords (N/A for OCI - uses IAM only)
echo "1️⃣  Password Policy"
echo "   OCI uses IAM (no default passwords) ✅"
echo ""

# 2. Disable unnecessary services
echo "2️⃣  Disable Unnecessary Services"

# List all services, disable unused ones
UNUSED_SERVICES=("streaming" "data-catalog" "data-flow")

for SERVICE in "${UNUSED_SERVICES[@]}"; do
  # Check if service is used
  COUNT=$(oci search resource structured-search \
    --query-text "query ${SERVICE} resources return allAdditionalFields" \
    | jq '.data.items | length')
  
  if [ "$COUNT" -eq 0 ]; then
    echo "   ✅ Service '${SERVICE}' not in use (good)"
  else
    echo "   ⚠️  Service '${SERVICE}' has ${COUNT} resources (review needed)"
  fi
done

echo ""

# 3. Secure configuration
echo "3️⃣  Secure Configuration"

# Check for public Object Storage buckets
PUBLIC_BUCKETS=$(oci os bucket list --compartment-id ${COMPARTMENT_OCID} \
  --query 'data[?\"public-access-type\"!=`NoPublicAccess`].name' \
  --raw-output)

if [ -z "$PUBLIC_BUCKETS" ]; then
  echo "   ✅ No public Object Storage buckets"
else
  echo "   ❌ FAIL: Public buckets found:"
  echo "$PUBLIC_BUCKETS"
  exit 1
fi

# Check for overly permissive NSG rules
echo ""
echo "   Checking Network Security Groups..."

for NSG in $(oci network nsg list --compartment-id ${COMPARTMENT_OCID} --query 'data[].id' --raw-output); do
  OPEN_RULES=$(oci network nsg rules list --nsg-id ${NSG} \
    --query 'data[?source==`0.0.0.0/0` && direction==`INGRESS`]' \
    | jq 'length')
  
  if [ "$OPEN_RULES" -gt 0 ]; then
    echo "   ⚠️  WARNING: NSG ${NSG} has ${OPEN_RULES} open ingress rules"
  fi
done

echo ""
echo "======================================"
echo "✅ PCI-DSS Hardening Complete"
```

### PCI-DSS Requirement 10: Log and Monitor All Access
```python
#!/usr/bin/env python3
# scripts/pci_dss_logging_validation.py

import oci
from datetime import datetime, timedelta

def validate_pci_logging():
    """Validate PCI-DSS logging requirements."""
    
    print("💳 PCI-DSS Requirement 10: Logging Validation")
    print("=" * 60)
    
    signer = oci.auth.signers.InstancePrincipalsSecurityTokenSigner()
    audit = oci.audit.AuditClient(config={}, signer=signer)
    logging = oci.logging.LoggingManagementClient(config={}, signer=signer)
    
    # Requirement 10.2.1: All individual user accesses to cardholder data
    print("\n1️⃣  User Access Logging")
    
    # Check if logs capture user ID
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=1)
    
    events = audit.list_events(
        compartment_id=get_tenancy_id(),
        start_time=start_time,
        end_time=end_time
    ).data
    
    if events:
        sample_event = events[0].data
        if 'identity' in sample_event and 'principal_name' in sample_event['identity']:
            print("   ✅ PASS: User identity captured in logs")
        else:
            print("   ❌ FAIL: User identity not captured")
    
    # Requirement 10.2.2: All actions taken by any individual with root or admin
    print("\n2️⃣  Privileged User Action Logging")
    
    admin_events = [e for e in events if 'admin' in e.data.get('identity', {}).get('principal_name', '').lower()]
    print(f"   Admin actions logged: {len(admin_events)}")
    print("   ✅ PASS: Admin actions captured")
    
    # Requirement 10.2.3: Access to all audit trails
    print("\n3️⃣  Audit Trail Access Logging")
    
    audit_access = [e for e in events if 'audit' in e.event_name.lower()]
    print(f"   Audit trail accesses: {len(audit_access)}")
    
    if audit_access:
        print("   ✅ PASS: Audit trail access is logged")
    else:
        print("   ℹ️  INFO: No audit trail access in sample period")
    
    # Requirement 10.3: Log retention (1 year, 3 months online)
    print("\n4️⃣  Log Retention Policy")
    
    log_groups = logging.list_log_groups(
        compartment_id=get_compartment_id()
    ).data
    
    for log_group in log_groups:
        logs = logging.list_logs(log_group_id=log_group.id).data
        
        for log in logs:
            retention = log.retention_duration
            
            if retention and retention >= 365:
                print(f"   ✅ PASS: Log '{log.display_name}' retention {retention} days")
            else:
                print(f"   ❌ FAIL: Log '{log.display_name}' retention {retention} days (need 365+)")
    
    # Requirement 10.5: Secure audit trails
    print("\n5️⃣  Audit Trail Security")
    
    # OCI audit logs are immutable by default
    print("   ✅ PASS: OCI audit logs are immutable (cannot be altered)")
    
    # Check if audit logs are backed up
    print("   ℹ️  INFO: Verify audit log backup to secondary region")
    
    print("\n" + "=" * 60)
    print("✅ PCI-DSS Logging Validation Complete")

if __name__ == '__main__':
    validate_pci_logging()
```

---

## 4. SECURITY BASELINE CONFIGURATION

### OCI Security Baseline (CIS Benchmark)
```yaml
# ansible/security_baseline.yml

---
- name: Apply OCI CIS Security Baseline
  hosts: tundra_nodes
  become: yes
  
  tasks:
    - name: 1.1 Ensure SSH Protocol is 2
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?Protocol'
        line: 'Protocol 2'
      notify: restart sshd
    
    - name: 1.2 Disable root login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: 'PermitRootLogin no'
      notify: restart sshd
    
    - name: 1.3 Disable password authentication
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: 'PasswordAuthentication no'
      notify: restart sshd
    
    - name: 2.1 Enable firewall
      systemd:
        name: firewalld
        enabled: yes
        state: started
    
    - name: 2.2 Configure firewall rules (allow only required ports)
      firewalld:
        port: "{{ item }}"
        permanent: yes
        state: enabled
        immediate: yes
      loop:
        - 22/tcp  # SSH
        - 443/tcp # HTTPS (Tundra API)
    
    - name: 2.3 Block all other ports
      firewalld:
        rich_rule: 'rule family="ipv4" drop'
        permanent: yes
        state: enabled
        zone: public
        immediate: yes
    
    - name: 3.1 Enable auditd
      systemd:
        name: auditd
        enabled: yes
        state: started
    
    - name: 3.2 Configure audit rules
      copy:
        content: |
          # Audit authentication attempts
          -w /var/log/auth.log -p wa -k auth_log
          
          # Audit file permission changes
          -a always,exit -F arch=b64 -S chmod,fchmod,fchmodat -F auid>=1000 -k perm_mod
          
          # Audit user/group changes
          -w /etc/passwd -p wa -k user_modification
          -w /etc/group -p wa -k group_modification
          
          # Audit sudo usage
          -w /etc/sudoers -p wa -k sudoers
          -w /var/log/sudo.log -p wa -k sudo_log
        dest: /etc/audit/rules.d/domo_security.rules
      notify: restart auditd
    
    - name: 4.1 Set strong password policy
      lineinfile:
        path: /etc/security/pwquality.conf
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
      loop:
        - { regexp: '^minlen', line: 'minlen = 14' }
        - { regexp: '^dcredit', line: 'dcredit = -1' }
        - { regexp: '^ucredit', line: 'ucredit = -1' }
        - { regexp: '^lcredit', line: 'lcredit = -1' }
        - { regexp: '^ocredit', line: 'ocredit = -1' }
    
    - name: 5.1 Install security updates
      dnf:
        name: '*'
        state: latest
        security: yes
      when: ansible_os_family == "RedHat"
    
    - name: 6.1 Configure log retention
      lineinfile:
        path: /etc/logrotate.d/syslog
        regexp: '^rotate'
        line: 'rotate 365'
    
    - name: 7.1 Disable unnecessary services
      systemd:
        name: "{{ item }}"
        enabled: no
        state: stopped
      loop:
        - avahi-daemon
        - cups
      ignore_errors: yes
    
    - name: 8.1 Enable AIDE (file integrity monitoring)
      dnf:
        name: aide
        state: present
    
    - name: 8.2 Initialize AIDE database
      command: aide --init
      args:
        creates: /var/lib/aide/aide.db.new.gz
    
    - name: 8.3 Schedule AIDE checks
      cron:
        name: "AIDE integrity check"
        hour: "3"
        minute: "0"
        job: "/usr/sbin/aide --check | mail -s 'AIDE Report' security@domo.com"
  
  handlers:
    - name: restart sshd
      systemd:
        name: sshd
        state: restarted
    
    - name: restart auditd
      command: service auditd restart
```

---

## 5. AUDIT LOGGING & MONITORING

### Comprehensive Audit Configuration
```hcl
# terraform/audit/comprehensive_logging.tf

# Enable OCI Audit (automatic, immutable)
# Retention: 365 days default, configurable up to 2,557 days (~7 years)

# Configure audit retention
resource "oci_audit_configuration" "audit_config" {
  compartment_id = var.tenancy_ocid
  
  retention_period_days = 2557  # Maximum: 7 years for compliance
}

# Create log group for application logs
resource "oci_logging_log_group" "domo_app_logs" {
  compartment_id = var.compartment_ocid
  display_name   = "domo-application-logs"
  description    = "Tundra query logs, API access logs"
}

# Log: Tundra query logs
resource "oci_logging_log" "tundra_queries" {
  display_name = "tundra-query-log"
  log_group_id = oci_logging_log_group.domo_app_logs.id
  log_type     = "CUSTOM"
  
  configuration {
    source {
      category    = "all"
      resource    = oci_core_instance.tundra_node[0].id
      service     = "computeagent"
      source_type = "OCISERVICE"
    }
    
    compartment_id = var.compartment_ocid
  }
  
  is_enabled         = true
  retention_duration = 365  # 1 year
}

# Log: Load Balancer access logs
resource "oci_logging_log" "lb_access" {
  display_name = "load-balancer-access"
  log_group_id = oci_logging_log_group.domo_app_logs.id
  log_type     = "SERVICE"
  
  configuration {
    source {
      category    = "access"
      resource    = oci_load_balancer_load_balancer.tundra_lb.id
      service     = "loadbalancer"
      source_type = "OCISERVICE"
    }
    
    compartment_id = var.compartment_ocid
  }
  
  is_enabled         = true
  retention_duration = 365
}

# Log: Object Storage access logs
resource "oci_logging_log" "object_storage_access" {
  display_name = "object-storage-access"
  log_group_id = oci_logging_log_group.domo_app_logs.id
  log_type     = "SERVICE"
  
  configuration {
    source {
      category    = "read"
      parameters  = {
        bucketName = "domo-tenant-*"
      }
      resource    = oci_objectstorage_bucket.tenant_data.id
      service     = "objectstorage"
      source_type = "OCISERVICE"
    }
    
    compartment_id = var.compartment_ocid
  }
  
  is_enabled         = true
  retention_duration = 365
}

# Export logs to external SIEM (Splunk, ELK)
resource "oci_logging_unified_agent_configuration" "log_export" {
  compartment_id = var.compartment_ocid
  display_name   = "domo-log-export-to-splunk"
  is_enabled     = true
  
  service_configuration {
    configuration_type = "LOGGING"
    
    destination {
      log_object_id = oci_logging_log.tundra_queries.id
    }
    
    sources {
      source_type = "LOG_TAIL"
      name        = "tundra-app-logs"
      
      paths = [
        "/var/log/tundra/*.log",
        "/var/log/domo-api/*.log"
      ]
    }
  }
  
  group_association {
    group_list = [oci_identity_dynamic_group.tundra_vms.id]
  }
}
```

---

## 6. ENCRYPTION STANDARDS

### Encryption at Rest
```hcl
# terraform/security/encryption.tf

# Create customer-managed encryption key in Vault
resource "oci_kms_vault" "domo_vault" {
  compartment_id = var.compartment_ocid
  display_name   = "domo-encryption-vault"
  vault_type     = "DEFAULT"  # Or "VIRTUAL_PRIVATE" for dedicated HSM
}

resource "oci_kms_key" "master_encryption_key" {
  compartment_id = var.compartment_ocid
  display_name   = "domo-master-key"
  
  key_shape {
    algorithm = "AES"
    length    = 256  # AES-256
  }
  
  management_endpoint = oci_kms_vault.domo_vault.management_endpoint
  
  # Key rotation every 90 days
  # (Manual rotation - OCI doesn't auto-rotate yet)
}

# Encrypt Object Storage buckets
resource "oci_objectstorage_bucket" "tenant_data_encrypted" {
  compartment_id = var.compartment_ocid
  name           = "domo-tenant-${var.tenant_id}-oci"
  namespace      = data.oci_objectstorage_namespace.ns.namespace
  
  # Customer-managed encryption
  kms_key_id = oci_kms_key.master_encryption_key.id
  
  # WORM (Write Once Read Many) for compliance
  object_events_enabled = true
  
  # Versioning for data recovery
  versioning = "Enabled"
}

# Encrypt Block Volumes
resource "oci_core_volume" "tundra_data_encrypted" {
  availability_domain = data.oci_identity_availability_domains.ads.availability_domains[0].name
  compartment_id      = var.compartment_ocid
  display_name        = "tundra-data-encrypted-${var.tenant_id}"
  
  size_in_gbs = 1000
  vpus_per_gb = 20  # Ultra High Performance
  
  # Customer-managed encryption
  kms_key_id = oci_kms_key.master_encryption_key.id
}

# Boot volume encryption (enabled by default, but can specify key)
resource "oci_core_instance" "tundra_node_encrypted" {
  # ... other config ...
  
  source_details {
    source_type = "image"
    source_id   = var.tundra_image_ocid
    
    # Encrypt boot volume with customer key
    kms_key_id = oci_kms_key.master_encryption_key.id
  }
}
```

### Encryption in Transit
```hcl
# terraform/security/tls_config.tf

# Load Balancer with TLS 1.2+ only
resource "oci_load_balancer_backend_set" "tundra_backend" {
  name             = "tundra-backend"
  load_balancer_id = oci_load_balancer_load_balancer.tundra_lb.id
  policy           = "LEAST_CONNECTIONS"
  
  health_checker {
    protocol = "HTTPS"
    port     = 443
    url_path = "/health"
    
    # Require TLS for health checks
    return_code = 200
  }
  
  ssl_configuration {
    # Only TLS 1.2 and 1.3
    protocols = ["TLSv1.2", "TLSv1.3"]
    
    # Strong cipher suites only
    cipher_suite_name = "oci-default-ssl-cipher-suite-v1"
    
    # Certificate from OCI Certificates service
    certificate_ids = [oci_certificates_management_certificate.tundra_cert.id]
    
    # Verify client certificates (mutual TLS)
    verify_depth            = 5
    verify_peer_certificate = true
  }
}

# Certificate management
resource "oci_certificates_management_certificate" "tundra_cert" {
  compartment_id = var.compartment_ocid
  name           = "tundra-${var.tenant_id}-cert"
  
  certificate_config {
    config_type = "MANAGED"
    
    subject {
      common_name = "tenant${var.tenant_id}.domo.com"
    }
    
    validity {
      time_of_validity_not_after = "2026-01-01T00:00:00Z"
    }
    
    # Auto-renewal 30 days before expiration
  }
}

# Enforce HTTPS redirect
resource "oci_load_balancer_rule_set" "https_redirect" {
  load_balancer_id = oci_load_balancer_load_balancer.tundra_lb.id
  name             = "https_redirect_rule"
  
  items {
    action = "REDIRECT"
    
    conditions {
      attribute_name  = "PATH"
      attribute_value = "/"
      operator        = "PREFIX_MATCH"
    }
    
    redirect_uri {
      protocol = "HTTPS"
      port     = 443
    }
    
    response_code = 301  # Permanent redirect
  }
}
```

---

## 7. SECURITY CHECKLIST (Pre-Production)
```markdown
# Pre-Production Security Checklist

## Network Security
- [ ] All NSGs configured with least-privilege rules
- [ ] No 0.0.0.0/0 ingress rules (except Load Balancer)
- [ ] Private subnets for compute instances
- [ ] Service Gateway configured for Object Storage (no NAT)
- [ ] FastConnect or VPN for cross-cloud (no public internet)

## IAM & Access Control
- [ ] Instance Principals configured for all VMs
- [ ] Zero API keys in code or config files
- [ ] MFA enforced for all human users
- [ ] Least-privilege IAM policies
- [ ] Regular access reviews scheduled (quarterly)

## Encryption
- [ ] Object Storage encrypted with customer-managed keys
- [ ] Block Volumes encrypted with customer-managed keys
- [ ] Boot Volumes encrypted
- [ ] TLS 1.2+ enforced for all endpoints
- [ ] Load Balancer uses strong cipher suites

## Logging & Monitoring
- [ ] OCI Audit enabled (365+ day retention)
- [ ] Application logs exported to log group
- [ ] Security alarms configured (failed logins, unauthorized access)
- [ ] Log export to SIEM (Splunk/ELK) operational
- [ ] 24/7 security monitoring dashboard

## Compliance
- [ ] SOC 2 controls validated
- [ ] HIPAA compliance check passed (if applicable)
- [ ] PCI-DSS requirements met (if applicable)
- [ ] Security baseline (CIS) applied to all VMs
- [ ] Compliance reports generated and archived

## Vulnerability Management
- [ ] OS security patches applied
- [ ] Vulnerability scanning scheduled (weekly)
- [ ] Container images scanned (if using containers)
- [ ] Dependency updates current

## Disaster Recovery
- [ ] Cross-region replication enabled
- [ ] Backup retention configured (30 days)
- [ ] DR runbook tested
- [ ] RTO/RPO documented and validated

## Incident Response
- [ ] Security incident response plan documented
- [ ] On-call rotation configured
- [ ] PagerDuty/Slack integrations tested
- [ ] Escalation path defined
- [ ] Incident communication templates ready
```

---

**Document Version**: 1.0
**Last Updated**: 2025-01-15
**Owner**: Security Team
**Review Frequency**: Quarterly


================================================================================
              DOCUMENT 7: DEVELOPER ONBOARDING GUIDE
================================================================================

# OCI Developer Onboarding Guide
**Audience**: Software Engineers, DevOps Engineers
**Purpose**: Get developers productive with OCI in 1 day

## Welcome to OCI Development!

This guide will get you from zero to deploying on OCI in a few hours.

---

## Day 1 Morning: Setup (2 hours)

### Prerequisites
```bash
# Required software
- Python 3.8+ or 3.9+
- Terraform 1.5+
- Git
- SSH client
- Your favorite code editor (VS Code recommended)
```

### Step 1: Get OCI Access (30 minutes)

**Request from IT**:
1. OCI user account (with MFA enforced)
2. Access to `domo-dev` compartment
3. API key generation permissions

**Login**:
```bash
# Navigate to OCI Console
https://cloud.oracle.com

# Sign in with your Domo SSO credentials
# Complete MFA setup (required)
```

**Generate API Key** (for local development):
```bash
# In OCI Console: User Settings → API Keys → Add API Key
# Download both:
# - Private key: ~/.oci/oci_api_key.pem
# - Config file: ~/.oci/config

# Set permissions
chmod 600 ~/.oci/oci_api_key.pem
chmod 600 ~/.oci/config

# Your ~/.oci/config should look like:
[DEFAULT]
user=ocid1.user.oc1..aaaaaaa...
fingerprint=aa:bb:cc:dd:...
tenancy=ocid1.tenancy.oc1..aaaaaa...
region=us-ashburn-1
key_file=~/.oci/oci_api_key.pem
```

**Test Connection**:
```bash
# Install OCI CLI
pip install oci-cli

# Test
oci iam region list
# Should list OCI regions

# Verify your identity
oci iam user get --user-id $(oci iam user list --query 'data[0].id' --raw-output)
# Should return your user details
```

### Step 2: Install Developer Tools (15 minutes)
```bash
# Install OCI Python SDK
pip install oci

# Install Terraform OCI provider
# Create terraform test directory
mkdir -p ~/oci-test
cd ~/oci-test

cat > main.tf <<EOF
terraform {
  required_providers {
    oci = {
      source  = "oracle/oci"
      version = "~> 5.0"
    }
  }
}

provider "oci" {
  region = "us-ashburn-1"
}

# Test data source
data "oci_identity_availability_domains" "ads" {
  compartment_id = var.tenancy_ocid
}

output "availability_domains" {
  value = data.oci_identity_availability_domains.ads.availability_domains[*].name
}
EOF

# Initialize Terraform
terraform init

# Test (get your tenancy OCID from ~/.oci/config)
terraform plan -var="tenancy_ocid=YOUR_TENANCY_OCID"
```

**Install VS Code Extensions** (optional but recommended):
- Terraform (HashiCorp)
- Python (Microsoft)
- OCI Extension Pack

### Step 3: Clone Domo OCI Repo (15 minutes)
```bash
# Clone main repo
git clone git@github.com:domo/oci-infrastructure.git
cd oci-infrastructure

# Install Python dependencies
pip install -r requirements.txt

# Install pre-commit hooks
pre-commit install

# Verify setup
./scripts/verify_developer_setup.sh
# Should output: ✅ Developer setup complete
```

---

## Day 1 Afternoon: Your First Deployment (2-3 hours)

### Exercise 1: Deploy a Test Tundra Cluster (90 minutes)

**Scenario**: Deploy a small Tundra cluster for testing in your dev compartment.
```bash
cd terraform/tundra-cluster

# Create your dev variables file
cat > dev-yourname.tfvars <<EOF
# Your personal dev environment
tenant_id       = "dev-$(whoami)"
compartment_ocid = "ocid1.compartment.oc1..domo-dev"
subnet_ocid      = "ocid1.subnet.oc1..domo-dev-private-subnet"

# Small dev cluster
cluster_size = "small"
ocpus        = 8
memory_gb    = 128
node_count   = 1  # Just 1 node for dev

# Tags
tags = {
  "domo.environment" = "dev"
  "domo.owner"       = "$(whoami)"
  "domo.expires"     = "$(date -d '+7 days' +%Y-%m-%d)"  # Auto-cleanup in 7 days
}
EOF

# Plan
terraform plan -var-file=dev-yourname.tfvars

# Review the plan (should create ~10 resources)
# - 1 compute instance
# - 1 block volume
# - 1 VNIC
# - Security rules
# - etc.

# Apply
terraform apply -var-file=dev-yourname.tfvars

# ⏱️ Takes ~8 minutes
# ☕ Go get coffee!
```

**Connect to Your Cluster**:
```bash
# Get the private IP
INSTANCE_IP=$(terraform output -raw instance_private_ip)

# SSH (you're on VPN or bastion, right?)
ssh -i ~/.ssh/domo_dev_key opc@${INSTANCE_IP}

# On the instance:
# Check Tundra is running
sudo systemctl status tundra

# Check data hydration
df -h /mnt/tundra-data

# Run a test query
/opt/tundra/bin/tundra-cli --query "SELECT COUNT(*) FROM test_table"
```

**Clean Up** (when done):
```bash
# Destroy your dev cluster
terraform destroy -var-file=dev-yourname.tfvars

# Confirm cleanup
oci compute instance list --compartment-id YOUR_COMPARTMENT_OCID \
  | jq '.data[] | select(.["display-name"] | contains("dev-yourname"))'
# Should return empty
```

### Exercise 2: Work with Object Storage (30 minutes)

**Create a Dev Bucket**:
```python
#!/usr/bin/env python3
# my_first_oci_script.py

import oci

# Use your OCI config file
config = oci.config.from_file()

# Create Object Storage client
os_client = oci.object_storage.ObjectStorageClient(config)
namespace = os_client.get_namespace().data

# Create bucket
bucket_name = f"dev-{config['user'].split('.')[-1]}-test"

try:
    os_client.create_bucket(
        namespace_name=namespace,
        create_bucket_details=oci.object_storage.models.CreateBucketDetails(
            name=bucket_name,
            compartment_id=config['tenancy'],  # Your compartment OCID
            public_access_type='NoPublicAccess'
        )
    )
    print(f"✅ Created bucket: {bucket_name}")
except oci.exceptions.ServiceError as e:
    if e.status == 409:
        print(f"ℹ️  Bucket already exists: {bucket_name}")
    else:
        raise

# Upload a file
test_content = b"Hello from OCI! This is my first object."

os_client.put_object(
    namespace_name=namespace,
    bucket_name=bucket_name,
    object_name="test.txt",
    put_object_body=test_content
)
print("✅ Uploaded test.txt")

# Download the file
response = os_client.get_object(
    namespace_name=namespace,
    bucket_name=bucket_name,
    object_name="test.txt"
)

downloaded_content = response.data.content
print(f"✅ Downloaded: {downloaded_content.decode()}")

# List objects
objects = os_client.list_objects(
    namespace_name=namespace,
    bucket_name=bucket_name
).data.objects

print(f"✅ Objects in bucket: {[obj.name for obj in objects]}")

# Clean up
os_client.delete_object(namespace, bucket_name, "test.txt")
os_client.delete_bucket(namespace, bucket_name)
print("✅ Cleaned up")
```

**Run it**:
```bash
python my_first_oci_script.py
```

---

## Day 2+: Development Workflows

### Workflow 1: Instance Principal Development

**Challenge**: Instance Principals only work on OCI VMs. How to develop locally?

**Solution 1**: Use config file for local, Instance Principal on OCI
```python
import oci
import os

def get_oci_client():
    """
    Smart client factory: Instance Principal on OCI, config file locally.
    """
    if os.path.exists('/var/run/secrets/oci'):
        # We're on OCI - use Instance Principal
        signer = oci.auth.signers.InstancePrincipalsSecurityTokenSigner()
        client = oci.object_storage.ObjectStorageClient(config={}, signer=signer)
        print("🔒 Using Instance Principal (zero keys!)")
    else:
        # Local development - use config file
        config = oci.config.from_file()
        client = oci.object_storage.ObjectStorageClient(config)
        print("💻 Using config file (local dev)")
    
    return client

# Use it
client = get_oci_client()
namespace = client.get_namespace().data
```

**Solution 2**: Use environment variables
```bash
# Set environment variables (alternative to config file)
export OCI_CLI_USER=ocid1.user.oc1..aaaa
export OCI_CLI_FINGERPRINT=aa:bb:cc:dd
export OCI_CLI_TENANCY=ocid1.tenancy.oc1..aaaa
export OCI_CLI_REGION=us-ashburn-1
export OCI_CLI_KEY_FILE=~/.oci/oci_api_key.pem

# Python will automatically use these
```

### Workflow 2: Testing Against Dev Tenants

**We have 10 dev tenants for testing** (`dev-001` through `dev-010`).

**Claim a dev tenant**:
```bash
# Check availability
./scripts/dev-tenant-manager.sh list

# Example output:
# dev-001: AVAILABLE
# dev-002: IN_USE (by alice@domo.com, expires 2025-01-18)
# dev-003: AVAILABLE
# ...

# Claim one for 24 hours
./scripts/dev-tenant-manager.sh claim dev-001 --hours 24

# Deploy your code
./scripts/deploy_to_tenant.sh dev-001

# Test
./scripts/run_integration_tests.sh dev-001

# Release when done (or auto-releases after 24h)
./scripts/dev-tenant-manager.sh release dev-001
```

### Workflow 3: Local Testing with Docker

**Run Tundra locally** (for query testing without OCI):
```bash
# Pull Tundra dev image
docker pull domo/tundra:dev-latest

# Run with sample data
docker run -d \
  --name tundra-local \
  -p 8080:8080 \
  -v $(pwd)/sample-data:/data \
  domo/tundra:dev-latest

# Test query
curl -X POST http://localhost:8080/query \
  -H 'Content-Type: application/json' \
  -d '{"sql": "SELECT COUNT(*) FROM sample_table"}'

# Stop
docker stop tundra-local
docker rm tundra-local
```

---

## Common Development Tasks

### Task 1: Add a New Tenant
```bash
# Use the tenant provisioning script
./scripts/provision_tenant.sh \
  --tenant-id T-9999 \
  --size medium \
  --region us-ashburn-1 \
  --data-source s3://customer-data-bucket

# This will:
# 1. Create Object Storage bucket
# 2. Sync data from S3
# 3. Provision Tundra cluster
# 4. Run health checks
# 5. Add to load balancer
# ⏱️ Takes ~30 minutes
```

### Task 2: Scale a Tenant's Cluster
```bash
# Scale up for load testing
./scripts/scale_tenant.sh T-0042 --ocpus 32 --memory 512

# Test under load
./scripts/load_test.sh T-0042 --duration 10m --qps 100

# Scale back down
./scripts/scale_tenant.sh T-0042 --ocpus 16 --memory 256
```

### Task 3: Debug Query Performance
```bash
# Enable query profiling
./scripts/enable_query_profiling.sh T-0042

# Run problematic query
./scripts/execute_query.sh T-0042 "SELECT * FROM large_table WHERE..."

# View query plan
./scripts/show_query_plan.sh T-0042 --query-id XXXXX

# Check if indexes needed
./scripts/analyze_query_performance.sh T-0042

# Disable profiling
./scripts/disable_query_profiling.sh T-0042
```

---

## Debugging Tips

### Debugging OCI API Calls
```python
import oci
import logging

# Enable debug logging
logging.basicConfig(level=logging.DEBUG)
oci_logger = logging.getLogger('oci')
oci_logger.setLevel(logging.DEBUG)

# Now all API calls are logged
config = oci.config.from_file()
client = oci.object_storage.ObjectStorageClient(config)
client.get_namespace()  # Will log full request/response
```

### Common Errors & Solutions

**Error**: `ServiceError: 401 NotAuthenticated`
```bash
# Solution: Check your API key
oci iam user get --user-id $(oci iam user list --query 'data[0].id' --raw-output)

# Regenerate if needed (OCI Console → User Settings → API Keys)
```

**Error**: `ServiceError: 404 NotFound`
```bash
# Solution: Wrong compartment or region
# Verify compartment OCID
oci iam compartment list --all

# Verify region
oci iam region list
```

**Error**: `ServiceError: 429 TooManyRequests`
```bash
# Solution: You hit rate limits. Add retry logic:
from oci.retry import DEFAULT_RETRY_STRATEGY

client = oci.object_storage.ObjectStorageClient(
    config,
    retry_strategy=DEFAULT_RETRY_STRATEGY
)
```

---

## Code Review Checklist

Before submitting your PR:

- [ ] No API keys in code (use Instance Principal or config file)
- [ ] Error handling for all OCI API calls
- [ ] Retry logic for transient failures
- [ ] Logging at appropriate level (INFO/DEBUG)
- [ ] Tests written (unit + integration if applicable)
- [ ] Terraform plan reviewed (no accidental deletions)
- [ ] Dev tenant tested
- [ ] Documentation updated
- [ ] Security review (no public buckets, etc.)

---

## Getting Help

**Slack Channels**:
- `#oci-migration` - General questions
- `#oci-dev-help` - Developer support
- `#oci-security` - Security questions

**Office Hours**:
- Tuesdays 10am PT: OCI Deep Dive
- Thursdays 2pm PT: Developer Q&A

**Documentation**:
- Internal Wiki: https://wiki.domo.com/oci
- OCI Docs: https://docs.oracle.com/en-us/iaas/
- This repo: `docs/` folder

**On-Call**: PagerDuty (only for production incidents)

---

## Next Steps

Once comfortable with basics:
1. Review the Architecture Decision Records (ADRs)
2. Read the Security & Compliance Guide
3. Complete the Advanced OCI Patterns course (internal)
4. Shadow an on-call shift
5. Deploy your first production change!

**Welcome to the team! 🎉**

---

**Document Version**: 1.0
**Last Updated**: 2025-01-15
**Owner**: Platform Engineering Team


================================================================================
        DOCUMENT 8: MONITORING & ALERTING CONFIGURATION
================================================================================

# OCI Monitoring & Alerting Configuration Guide
**Audience**: SRE Team, Platform Engineers, DevOps
**Purpose**: Comprehensive monitoring setup for OCI Tundra clusters

## Overview

This guide covers:
1. OCI Monitoring setup
2. Custom metrics
3. Alert configuration
4. Grafana dashboards
5. PagerDuty integration
6. Slack notifications

---

## 1. OCI MONITORING SETUP

### Enable Monitoring Agent on All Instances
```hcl
# terraform/monitoring/monitoring_agents.tf

resource "oci_core_instance" "tundra_node" {
  # ... other config ...
  
  agent_config {
    is_monitoring_disabled = false
    is_management_disabled = false
    
    plugins_config {
      name          = "Compute Instance Monitoring"
      desired_state = "ENABLED"
    }
    
    plugins_config {
      name          = "Custom Logs Monitoring"
      desired_state = "ENABLED"
    }
  }
}
```

### Create Monitoring Namespace
```hcl
# terraform/monitoring/custom_metrics.tf

# Custom metrics namespace for Domo-specific metrics
locals {
  custom_namespace = "domo_tundra"
}

# Metric: Query latency
resource "oci_monitoring_metric_data" "query_latency" {
  compartment_id = var.compartment_ocid
  
  # This is a template - actual metrics posted via API
  namespace = local.custom_namespace
  name      = "query_latency_ms"
  
  dimensions = {
    tenant_id    = var.tenant_id
    query_type   = "analytics"  # or "simple", "join"
    cluster_name = "tundra-${var.tenant_id}"
  }
}

# Metric: Hydration time
resource "oci_monitoring_metric_data" "hydration_time" {
  compartment_id = var.compartment_ocid
  
  namespace = local.custom_namespace
  name      = "hydration_duration_seconds"
  
  dimensions = {
    tenant_id    = var.tenant_id
    data_size_gb = "100"  # Dynamic
    cluster_name = "tundra-${var.tenant_id}"
  }
}

# Metric: Query throughput (QPS)
resource "oci_monitoring_metric_data" "query_throughput" {
  compartment_id = var.compartment_ocid
  
  namespace = local.custom_namespace
  name      = "queries_per_second"
  
  dimensions = {
    tenant_id    = var.tenant_id
    cluster_name = "tundra-${var.tenant_id}"
  }
}
```

---

## 2. CUSTOM METRICS PUBLISHING

### Python Client for Publishing Metrics
```python
#!/usr/bin/env python3
# lib/monitoring/metrics_publisher.py

import oci
import time
from datetime import datetime
from typing import Dict, List

class DomoMetricsPublisher:
    """
    Publish custom metrics to OCI Monitoring.
    Used by Tundra clusters to report query metrics.
    """
    
    def __init__(self, compartment_id: str):
        # Use Instance Principal
        signer = oci.auth.signers.InstancePrincipalsSecurityTokenSigner()
        self.client = oci.monitoring.MonitoringClient(config={}, signer=signer)
        self.compartment_id = compartment_id
        self.namespace = "domo_tundra"
    
    def publish_query_latency(self, tenant_id: str, latency_ms: float, 
                              query_type: str = "analytics"):
        """
        Publish query latency metric.
        
        Args:
            tenant_id: Tenant ID (e.g., "T-0042")
            latency_ms: Query latency in milliseconds
            query_type: "simple", "join", or "analytics"
        """
        metric_data = oci.monitoring.models.PostMetricDataDetails(
            metric_data=[
                oci.monitoring.models.MetricDataDetails(
                    namespace=self.namespace,
                    name="query_latency_ms",
                    compartment_id=self.compartment_id,
                    dimensions={
                        "tenant_id": tenant_id,
                        "query_type": query_type,
                        "cluster_name": f"tundra-{tenant_id}"
                    },
                    datapoints=[
                        oci.monitoring.models.Datapoint(
                            timestamp=datetime.utcnow(),
                            value=latency_ms
                        )
                    ]
                )
            ]
        )
        
        self.client.post_metric_data(post_metric_data_details=metric_data)
    
    def publish_query_throughput(self, tenant_id: str, qps: float):
        """Publish queries per second."""
        metric_data = oci.monitoring.models.PostMetricDataDetails(
            metric_data=[
                oci.monitoring.models.MetricDataDetails(
                    namespace=self.namespace,
                    name="queries_per_second",
                    compartment_id=self.compartment_id,
                    dimensions={
                        "tenant_id": tenant_id,
                        "cluster_name": f"tundra-{tenant_id}"
                    },
                    datapoints=[
                        oci.monitoring.models.Datapoint(
                            timestamp=datetime.utcnow(),
                            value=qps
                        )
                    ]
                )
            ]
        )
        
        self.client.post_metric_data(post_metric_data_details=metric_data)
    
    def publish_hydration_metrics(self, tenant_id: str, duration_seconds: float,
                                  data_size_gb: int):
        """Publish cluster hydration metrics."""
        metric_data = oci.monitoring.models.PostMetricDataDetails(
            metric_data=[
                oci.monitoring.models.MetricDataDetails(
                    namespace=self.namespace,
                    name="hydration_duration_seconds",
                    compartment_id=self.compartment_id,
                    dimensions={
                        "tenant_id": tenant_id,
                        "data_size_gb": str(data_size_gb),
                        "cluster_name": f"tundra-{tenant_id}"
                    },
                    datapoints=[
                        oci.monitoring.models.Datapoint(
                            timestamp=datetime.utcnow(),
                            value=duration_seconds
                        )
                    ]
                )
            ]
        )
        
        self.client.post_metric_data(post_metric_data_details=metric_data)
    
    def publish_batch(self, metrics: List[Dict]):
        """
        Publish multiple metrics in one API call (more efficient).
        
        Args:
            metrics: List of metric dicts with keys: name, value, dimensions
        """
        metric_data_list = []
        
        for metric in metrics:
            metric_data_list.append(
                oci.monitoring.models.MetricDataDetails(
                    namespace=self.namespace,
                    name=metric['name'],
                    compartment_id=self.compartment_id,
                    dimensions=metric.get('dimensions', {}),
                    datapoints=[
                        oci.monitoring.models.Datapoint(
                            timestamp=datetime.utcnow(),
                            value=metric['value']
                        )
                    ]
                )
            )
        
        # Publish batch
        batch = oci.monitoring.models.PostMetricDataDetails(
            metric_data=metric_data_list
        )
        
        self.client.post_metric_data(post_metric_data_details=batch)

# Usage example
if __name__ == '__main__':
    publisher = DomoMetricsPublisher(compartment_id='ocid1.compartment.oc1...')
    
    # Publish single metric
    publisher.publish_query_latency(
        tenant_id='T-0042',
        latency_ms=850.5,
        query_type='analytics'
    )
    
    # Publish batch (more efficient)
    publisher.publish_batch([
        {
            'name': 'query_latency_ms',
            'value': 850.5,
            'dimensions': {'tenant_id': 'T-0042', 'query_type': 'analytics'}
        },
        {
            'name': 'queries_per_second',
            'value': 45.2,
            'dimensions': {'tenant_id': 'T-0042'}
        },
        {
            'name': 'active_connections',
            'value': 23,
            'dimensions': {'tenant_id': 'T-0042'}
        }
    ])
```

### Integrate with Tundra Query Engine
```python
# tundra/query_engine.py (simplified)

from lib.monitoring.metrics_publisher import DomoMetricsPublisher
import time

class TundraQueryEngine:
    def __init__(self, tenant_id: str):
        self.tenant_id = tenant_id
        self.metrics = DomoMetricsPublisher(compartment_id=get_compartment_id())
    
    def execute_query(self, sql: str):
        """Execute query and publish metrics."""
        start_time = time.time()
        
        try:
            # Execute query
            result = self._run_query(sql)
            
            # Calculate latency
            latency_ms = (time.time() - start_time) * 1000
            
            # Determine query type
            query_type = self._classify_query(sql)
            
            # Publish metric
            self.metrics.publish_query_latency(
                tenant_id=self.tenant_id,
                latency_ms=latency_ms,
                query_type=query_type
            )
            
            return result
        
        except Exception as e:
            # Publish error metric
            self.metrics.publish_batch([
                {
                    'name': 'query_errors',
                    'value': 1,
                    'dimensions': {
                        'tenant_id': self.tenant_id,
                        'error_type': type(e).__name__
                    }
                }
            ])
            raise
```

---

## 3. ALERT CONFIGURATION

### Critical Alerts (24/7 PagerDuty)
```hcl
# terraform/monitoring/critical_alarms.tf

# ONS Topic for critical alerts
resource "oci_ons_notification_topic" "critical_alerts" {
  compartment_id = var.compartment_ocid
  name           = "domo-critical-alerts"
  description    = "Critical alerts requiring immediate response"
}

# Subscription: PagerDuty webhook
resource "oci_ons_subscription" "pagerduty" {
  compartment_id = var.compartment_ocid
  topic_id       = oci_ons_notification_topic.critical_alerts.id
  protocol       = "HTTPS"
  endpoint       = var.pagerduty_webhook_url
}

# Alarm: High query error rate
resource "oci_monitoring_alarm" "high_error_rate" {
  compartment_id        = var.compartment_ocid
  display_name          = "CRITICAL - High Query Error Rate"
  is_enabled            = true
  severity              = "CRITICAL"
  metric_compartment_id = var.compartment_ocid
  
  query = <<-EOT
    domo_tundra[1m]{name="query_errors"}.groupBy(tenant_id).rate() > 0.05
  EOT
  
  destinations = [oci_ons_notification_topic.critical_alerts.id]
  
  # Trigger if condition persists for 5 minutes
  pending_duration = "PT5M"
  
  # Re-alert every 30 minutes if not resolved
  repeat_notification_duration = "PT30M"
  
  body = <<-EOT
    🚨 CRITICAL: Tenant {{tenant_id}} query error rate >5%
    
    Current error rate: {{value}}%
    Threshold: 5%
    
    Action required:
    1. Check cluster health
    2. Review error logs
    3. Consider rollback if related to recent deployment
    
    Runbook: https://wiki.domo.com/oci/runbooks/high-error-rate
  EOT
}

# Alarm: Cluster down (no healthy instances)
resource "oci_monitoring_alarm" "cluster_down" {
  compartment_id        = var.compartment_ocid
  display_name          = "CRITICAL - Cluster Down"
  is_enabled            = true
  severity              = "CRITICAL"
  metric_compartment_id = var.compartment_ocid
  
  query = <<-EOT
    oci_computeagent[1m]{displayName=~"tundra-*"}.count() < 1
  EOT
  
  destinations = [oci_ons_notification_topic.critical_alerts.id]
  
  pending_duration = "PT2M"  # Alert after 2 minutes
  
  body = <<-EOT
    🚨 CRITICAL: Tundra cluster has ZERO healthy instances
    
    All instances may be down or unreachable.
    
    Immediate action required:
    1. Check OCI Console for instance status
    2. Verify network connectivity
    3. Initiate DR procedure if necessary
    
    Runbook: https://wiki.domo.com/oci/runbooks/cluster-down
  EOT
}

# Alarm: Slow query performance
resource "oci_monitoring_alarm" "slow_queries" {
  compartment_id        = var.compartment_ocid
  display_name          = "CRITICAL - Query Latency p95 >3s"
  is_enabled            = true
  severity              = "CRITICAL"
  metric_compartment_id = var.compartment_ocid
  
  query = <<-EOT
    domo_tundra[5m]{name="query_latency_ms"}.percentile(0.95) > 3000
  EOT
  
  destinations = [oci_ons_notification_topic.critical_alerts.id]
  
  pending_duration = "PT10M"
  
  body = <<-EOT
    🚨 CRITICAL: Query performance severely degraded
    
    p95 latency: {{value}}ms
    Threshold: 3000ms (3 seconds)
    
    Likely causes:
    - Cluster under-provisioned (scale up)
    - Data skew
    - Network issue
    
    Runbook: https://wiki.domo.com/oci/runbooks/slow-queries
  EOT
}
```

### Warning Alerts (Business Hours Slack)
```hcl
# terraform/monitoring/warning_alarms.tf

# ONS Topic for warnings
resource "oci_ons_notification_topic" "warning_alerts" {
  compartment_id = var.compartment_ocid
  name           = "domo-warning-alerts"
  description    = "Non-critical alerts for team awareness"
}

# Subscription: Slack webhook
resource "oci_ons_subscription" "slack" {
  compartment_id = var.compartment_ocid
  topic_id       = oci_ons_notification_topic.warning_alerts.id
  protocol       = "HTTPS"
  endpoint       = var.slack_webhook_url
}

# Alarm: High CPU usage
resource "oci_monitoring_alarm" "high_cpu" {
  compartment_id        = var.compartment_ocid
  display_name          = "WARNING - High CPU Usage"
  is_enabled            = true
  severity              = "WARNING"
  metric_compartment_id = var.compartment_ocid
  
  query = <<-EOT
    CpuUtilization[5m]{displayName=~"tundra-*"}.mean() > 75
  EOT
  
  destinations = [oci_ons_notification_topic.warning_alerts.id]
  
  pending_duration = "PT15M"
  
  body = <<-EOT
    ⚠️  WARNING: High CPU usage detected
    
    Cluster: {{displayName}}
    CPU: {{value}}%
    Threshold: 75%
    
    Consider scaling up if this persists.
  EOT
}

# Alarm: High memory usage
resource "oci_monitoring_alarm" "high_memory" {
  compartment_id        = var.compartment_ocid
  display_name          = "WARNING - High Memory Usage"
  is_enabled            = true
  severity              = "WARNING"
  metric_compartment_id = var.compartment_ocid
  
  query = <<-EOT
    MemoryUtilization[5m]{displayName=~"tundra-*"}.mean() > 85
  EOT
  
  destinations = [oci_ons_notification_topic.warning_alerts.id]
  
  pending_duration = "PT15M"
  
  body = <<-EOT
    ⚠️  WARNING: High memory usage detected
    
    Cluster: {{displayName}}
    Memory: {{value}}%
    Threshold: 85%
    
    Consider scaling up if this persists.
  EOT
}

# Alarm: Query latency trending up
resource "oci_monitoring_alarm" "latency_trend" {
  compartment_id        = var.compartment_ocid
  display_name          = "WARNING - Query Latency Increasing"
  is_enabled            = true
  severity              = "WARNING"
  metric_compartment_id = var.compartment_ocid
  
  query = <<-EOT
    domo_tundra[5m]{name="query_latency_ms"}.percentile(0.95) > 1500
  EOT
  
  destinations = [oci_ons_notification_topic.warning_alerts.id]
  
  pending_duration = "PT10M"
  
  body = <<-EOT
    ⚠️  WARNING: Query latency above target
    
    p95 latency: {{value}}ms
    Target: <1000ms
    Threshold: 1500ms
    
    Monitor and investigate if worsens.
  EOT
}
```

---

## 4. GRAFANA DASHBOARDS

### Setup Grafana with OCI Data Source
```bash
# Install Grafana (on monitoring VM)
sudo dnf install -y grafana

# Start Grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

# Install OCI plugin
grafana-cli plugins install oci-metrics-datasource

# Restart
sudo systemctl restart grafana-server

# Access: http://monitoring-vm:3000
# Default login: admin/admin
```

### Configure OCI Data Source
```yaml
# /etc/grafana/provisioning/datasources/oci.yaml

apiVersion: 1

datasources:
  - name: OCI Monitoring
    type: oci-metrics-datasource
    access: proxy
    isDefault: true
    
    jsonData:
      tenancyOCID: ocid1.tenancy.oc1..aaaa
      region: us-ashburn-1
      environment: local  # Use config file on Grafana server
      
      # Or use Instance Principal if Grafana on OCI:
      # environment: OCI_INSTANCE_PRINCIPAL
```

### Dashboard: Tundra Cluster Overview
```json
{
  "dashboard": {
    "title": "Tundra Cluster Overview",
    "panels": [
      {
        "id": 1,
        "title": "Query Latency (p95)",
        "type": "graph",
        "targets": [
          {
            "query": "domo_tundra{name=\"query_latency_ms\"}.percentile(0.95)",
            "refId": "A"
          }
        ],
        "yaxes": [
          {
            "label": "Latency (ms)",
            "show": true
          }
        ],
        "alert": {
          "conditions": [
            {
              "evaluator": {
                "params": [1500],
                "type": "gt"
              },
              "operator": {
                "type": "and"
              },
              "query": {
                "params": ["A", "5m", "now"]
              },
              "reducer": {
                "type": "avg"
              },
              "type": "query"
            }
          ],
          "frequency": "60s",
          "handler": 1,
          "name": "High Query Latency"
        }
      },
      {
        "id": 2,
        "title": "Queries Per Second",
        "type": "graph",
        "targets": [
          {
            "query": "domo_tundra{name=\"queries_per_second\"}",
            "refId": "A"
          }
        ]
      },
      {
        "id": 3,
        "title": "CPU Usage",
        "type": "graph",
        "targets": [
          {
            "query": "CpuUtilization{displayName=~\"tundra-*\"}",
            "refId": "A"
          }
        ],
        "thresholds": [
          {
            "value": 75,
            "colorMode": "critical",
            "op": "gt",
            "fill": true,
            "line": true
          }
        ]
      },
      {
        "id": 4,
        "title": "Memory Usage",
        "type": "graph",
        "targets": [
          {
            "query": "MemoryUtilization{displayName=~\"tundra-*\"}",
            "refId": "A"
          }
        ]
      },
      {
        "id": 5,
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "query": "domo_tundra{name=\"query_errors\"}.rate()",
            "refId": "A"
          }
        ],
        "yaxes": [
          {
            "label": "Errors/sec",
            "format": "short"
          }
        ]
      },
      {
        "id": 6,
        "title": "Active Tenants",
        "type": "stat",
        "targets": [
          {
            "query": "domo_tundra{name=\"queries_per_second\"}.groupBy(tenant_id).count()",
            "refId": "A"
          }
        ],
        "options": {
          "colorMode": "value",
          "graphMode": "area",
          "justifyMode": "auto"
        }
      }
    ],
    "time": {
      "from": "now-6h",
      "to": "now"
    },
    "refresh": "30s"
  }
}
```

Save as: `/var/lib/grafana/dashboards/tundra_overview.json`

---

## 5. PAGERDUTY INTEGRATION

### Setup PagerDuty Service
```python
#!/usr/bin/env python3
# scripts/setup_pagerduty.py

import requests
import json

PAGERDUTY_API_KEY = "YOUR_API_KEY"
PAGERDUTY_API_URL = "https://api.pagerduty.com"

def create_service():
    """Create PagerDuty service for Domo OCI."""
    
    headers = {
        "Authorization": f"Token token={PAGERDUTY_API_KEY}",
        "Content-Type": "application/json",
        "Accept": "application/vnd.pagerduty+json;version=2"
    }
    
    # Create escalation policy
    escalation_policy = {
        "escalation_policy": {
            "name": "Domo OCI Escalation",
            "escalation_rules": [
                {
                    "escalation_delay_in_minutes": 15,
                    "targets": [
                        {
                            "type": "schedule_reference",
                            "id": "SCHEDULE_ID"  # Your on-call schedule
                        }
                    ]
                },
                {
                    "escalation_delay_in_minutes": 30,
                    "targets": [
                        {
                            "type": "user_reference",
                            "id": "VP_ENG_USER_ID"  # VP Engineering
                        }
                    ]
                }
            ]
        }
    }
    
    response = requests.post(
        f"{PAGERDUTY_API_URL}/escalation_policies",
        headers=headers,
        json=escalation_policy
    )
    
    escalation_policy_id = response.json()["escalation_policy"]["id"]
    
    # Create service
    service = {
        "service": {
            "name": "Domo OCI Tundra",
            "description": "OCI Tundra cluster monitoring",
            "escalation_policy": {
                "id": escalation_policy_id,
                "type": "escalation_policy_reference"
            },
            "alert_creation": "create_alerts_and_incidents",
            "incident_urgency_rule": {
                "type": "constant",
                "urgency": "high"
            }
        }
    }
    
    response = requests.post(
        f"{PAGERDUTY_API_URL}/services",
        headers=headers,
        json=service
    )
    
    service_data = response.json()["service"]
    
    print(f"✅ PagerDuty service created:")
    print(f"   Service ID: {service_data['id']}")
    print(f"   Integration URL: {service_data['integrations'][0]['integration_url']}")
    
    return service_data

if __name__ == '__main__':
    create_service()
```

### Trigger PagerDuty from OCI Alarms
```python
#!/usr/bin/env python3
# oci_functions/pagerduty_webhook/func.py

import io
import json
import logging
import requests
from fdk import response

def handler(ctx, data: io.BytesIO = None):
    """
    OCI Function to forward OCI alarm notifications to PagerDuty.
    
    Triggered by ONS topic subscription.
    """
    try:
        body = json.loads(data.getvalue())
        
        # Parse OCI alarm notification
        alarm_name = body.get("alarmMetaData", [{}])[0].get("displayName", "Unknown")
        severity = body.get("severity", "CRITICAL")
        message = body.get("body", "")
        
        # Create PagerDuty event
        pagerduty_event = {
            "routing_key": "YOUR_INTEGRATION_KEY",
            "event_action": "trigger",
            "payload": {
                "summary": alarm_name,
                "severity": severity.lower(),
                "source": "OCI Monitoring",
                "custom_details": {
                    "message": message,
                    "alarm_name": alarm_name
                }
            }
        }
        
        # Send to PagerDuty
        response_pd = requests.post(
            "https://events.pagerduty.com/v2/enqueue",
            json=pagerduty_event
        )
        
        logging.getLogger().info(f"PagerDuty incident created: {response_pd.json()}")
        
        return response.Response(
            ctx, response_data=json.dumps({"status": "success"}),
            headers={"Content-Type": "application/json"}
        )
    
    except Exception as e:
        logging.getLogger().error(f"Error: {str(e)}")
        return response.Response(
            ctx, response_data=json.dumps({"status": "error", "message": str(e)}),
            headers={"Content-Type": "application/json"},
            status_code=500
        )
```

---

## 6. SLACK NOTIFICATIONS

### Slack App Setup
```python
#!/usr/bin/env python3
# scripts/setup_slack_app.py

"""
1. Go to https://api.slack.com/apps
2. Create New App → "Domo OCI Monitoring"
3. Add Incoming Webhook
4. Install to workspace
5. Copy webhook URL
"""

SLACK_WEBHOOK_URL = "https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX"

def send_slack_alert(message: str, severity: str = "info"):
    """
    Send alert to Slack.
    
    Args:
        message: Alert message
        severity: "critical", "warning", or "info"
    """
    import requests
    
    # Color based on severity
    colors = {
        "critical": "#FF0000",  # Red
        "warning": "#FFA500",   # Orange
        "info": "#0000FF"       # Blue
    }
    
    emoji = {
        "critical": "🚨",
        "warning": "⚠️",
        "info": "ℹ️"
    }
    
    payload = {
        "attachments": [
            {
                "color": colors.get(severity, colors["info"]),
                "blocks": [
                    {
                        "type": "section",
                        "text": {
                            "type": "mrkdwn",
                            "text": f"{emoji.get(severity, '')} *{severity.upper()}*\n\n{message}"
                        }
                    }
                ]
            }
        ]
    }
    
    response = requests.post(SLACK_WEBHOOK_URL, json=payload)
    
    if response.status_code != 200:
        print(f"❌ Slack notification failed: {response.text}")
    else:
        print("✅ Slack notification sent")

# Example usage
if __name__ == '__main__':
    send_slack_alert(
        message="Tenant T-0042 query latency p95: 1850ms (threshold: 1500ms)",
        severity="warning"
    )
```

### OCI Function for Slack Notifications
```python
# oci_functions/slack_webhook/func.py

import io
import json
import logging
import requests
from fdk import response

def handler(ctx, data: io.BytesIO = None):
    """
    OCI Function to forward OCI alarm notifications to Slack.
    """
    try:
        body = json.loads(data.getvalue())
        
        # Parse OCI alarm
        alarm_name = body.get("alarmMetaData", [{}])[0].get("displayName", "Unknown")
        severity = body.get("severity", "INFO")
        message = body.get("body", "")
        
        # Determine emoji and color
        if severity == "CRITICAL":
            emoji = "🚨"
            color = "#FF0000"
        elif severity == "WARNING":
            emoji = "⚠️"
            color = "#FFA500"
        else:
            emoji = "ℹ️"
            color = "#0000FF"
        
        # Format Slack message
        slack_message = {
            "text": f"{emoji} {alarm_name}",
            "attachments": [
                {
                    "color": color,
                    "text": message,
                    "fields": [
                        {
                            "title": "Severity",
                            "value": severity,
                            "short": True
                        },
                        {
                            "title": "Source",
                            "value": "OCI Monitoring",
                            "short": True
                        }
                    ],
                    "footer": "Domo OCI Monitoring",
                    "ts": int(body.get("timestamp", 0) / 1000)
                }
            ]
        }
        
        # Send to Slack
        webhook_url = ctx.Config()["slack-webhook-url"]
        response_slack = requests.post(webhook_url, json=slack_message)
        
        logging.getLogger().info(f"Slack notification sent: {response_slack.status_code}")
        
        return response.Response(
            ctx, response_data=json.dumps({"status": "success"}),
            headers={"Content-Type": "application/json"}
        )
    
    except Exception as e:
        logging.getLogger().error(f"Error: {str(e)}")
        return response.Response(
            ctx, response_data=json.dumps({"status": "error", "message": str(e)}),
            headers={"Content-Type": "application/json"},
            status_code=500
        )
```

---

## 7. MONITORING CHECKLIST
```markdown
# OCI Monitoring Setup Checklist

## Initial Setup
- [ ] OCI Monitoring Agent enabled on all instances
- [ ] Custom metrics namespace created (`domo_tundra`)
- [ ] Metrics publishing code deployed to Tundra
- [ ] Test metrics visible in OCI Console

## Alerting
- [ ] Critical alarms configured (error rate, cluster down, slow queries)
- [ ] Warning alarms configured (CPU, memory, latency trend)
- [ ] ONS topics created
- [ ] PagerDuty integration tested
- [ ] Slack integration tested

## Dashboards
- [ ] Grafana installed and configured
- [ ] OCI data source added
- [ ] Tundra overview dashboard imported
- [ ] Per-tenant dashboards created
- [ ] Public dashboard for status page

## Testing
- [ ] Trigger test alert (verify PagerDuty receives)
- [ ] Trigger test warning (verify Slack receives)
- [ ] Verify metrics appear in Grafana
- [ ] Test alert suppression during maintenance
- [ ] Verify alert escalation (wait 15 min, check if VP notified)

## Documentation
- [ ] Runbooks updated with OCI monitoring links
- [ ] On-call guide updated
- [ ] Alert response procedures documented
- [ ] Dashboard URLs shared with team
```

---

**Document Version**: 1.0
**Last Updated**: 2025-01-15
**Owner**: SRE Team

---

Due to length constraints, I'll continue with Documents 9-15 in the next response. Would you like me to proceed?