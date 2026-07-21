# CloudTrail Lake: Cost Optimization Through Architecture

## Context

Our AWS environment generates massive audit log volume across multiple accounts. CloudTrail Lake was enabled for security investigations and compliance queries, but monthly costs were climbing steadily.

## The Problem

CloudTrail Lake charges per event ingested and per query scan. With all events going into the event data store, costs were growing linearly with API activity — not with security value.

Key cost drivers:
- **Ingestion**: Every API call across all accounts stored in Lake
- **Query scans**: Full scans across all data for simple lookups
- **Redundancy**: Many events already existed in S3-based CloudTrail logs

## Architecture Analysis

### What We Had

```
All AWS Accounts
    │
    ▼
CloudTrail (org-wide)
    │
    ├── S3 (raw logs, cheap, 90-day retention)
    │
    └── Lake (event data store, expensive, 7-year retention)
            │
            └── All events, no filtering
```

### What We Needed

```
All AWS Accounts
    │
    ▼
CloudTrail (org-wide)
    │
    ├── S3 (raw logs, cheap, 90-day retention)
    │
    └── Lake (event data store, selective ingestion)
            │
            └── High-value events only
                - IAM changes
                - Security group modifications
                - Console login failures
                - KMS key operations
                - Resource creation/deletion
```

## Implementation

### Step 1: Event Selection

Analyzed 30 days of query patterns to identify which event sources were actually queried:

```python
# Pseudo-code for event analysis
events_by_source = analyze_query_logs(days=30)
# Result: 85% of queries targeted <15% of event sources
# Key sources: iam, ec2, s3, kms, sts, cloudtrail, organizations
```

### Step 2: Selective Ingestion

Created a new event data store with event selectors:

```hcl
resource "aws_cloudtrail_event_data_store" "selective" {
  name = "security-critical-events"
  
  ingestion_configuration {
    event_source_selector {
      event_source = "aws.ec2"
      event_type   = "AwsCloudTrailApiEvent"
    }
    event_source_selector {
      event_source = "aws.iam"
      event_type   = "AwsCloudTrailApiEvent"
    }
    event_source_selector {
      event_source = "aws.s3"
      event_type   = "AwsCloudTrailApiEvent"
    }
    event_source_selector {
      event_source = "aws.kms"
      event_type   = "AwsCloudTrailApiEvent"
    }
    event_source_selector {
      event_source = "aws.sts"
      event_type   = "AwsCloudTrailApiEvent"
    }
  }
  
  retention_period = 2555  # 7 years for compliance
}
```

### Step 3: Query Optimization

Added partition keys and time-based filtering to existing queries:

```sql
-- Before: Full scan
SELECT * FROM events WHERE userIdentity.arn LIKE '%admin%'

-- After: Partitioned scan
SELECT * FROM events 
WHERE eventTime > '2024-01-01'
  AND eventSource = 'iam.amazonaws.com'
  AND userIdentity.arn LIKE '%admin%'
```

## Results

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Events ingested/month | ~2.1B | ~340M | 84% reduction |
| Monthly cost | $4,200 | $680 | 84% savings |
| Query performance | 15-45s avg | 3-8s avg | 70% faster |
| Compliance coverage | 100% | 100% | No change |

The 16% of events still in Lake cover 100% of compliance requirements and 95% of actual investigation use cases.

## Trade-offs

**What we gained:**
- 84% cost reduction ($42K/year savings)
- Faster queries (less data to scan)
- Clearer data model (security-relevant events only)

**What we lost:**
- Ability to query low-priority events in Lake (still available in S3)
- Some flexibility for ad-hoc investigations (mitigated by S3 Athena fallback)

**Risk mitigation:**
- S3-based CloudTrail logs remain available for 90 days
- Athena can query S3 logs for deep investigations
- Event selection reviewed quarterly with security team

## Lessons Learned

1. **Default configurations are expensive.** CloudTrail Lake's default is to ingest everything. Always start with selective ingestion.

2. **Query patterns drive architecture.** Analyze actual usage before designing storage. 85% of queries targeted 15% of data.

3. **Compliance doesn't require everything.** Security teams needed IAM, network, and crypto events — not every S3 GetObject.

4. **Cost optimization is architectural.** You can't tune your way out of ingesting everything. The fix required rethinking the data model.

## Why This Matters

This demonstrates the ability to analyze cloud costs, understand service pricing models, design selective architectures, and deliver measurable savings — while maintaining compliance and security coverage. It's infrastructure cost engineering at the intersection of security, compliance, and FinOps.