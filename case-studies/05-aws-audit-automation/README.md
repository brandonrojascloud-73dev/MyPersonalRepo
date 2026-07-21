# AWS Audit Automation: From Manual to Self-Service

## Context

Our security team needed to perform regular audits across 50+ AWS accounts. The process was entirely manual: log into each account, run specific queries, compile results into spreadsheets. Each audit cycle took 2-3 weeks of analyst time.

## The Problem

Manual audits had three critical issues:

1. **Time**: 2-3 weeks per audit cycle across all accounts
2. **Consistency**: Different analysts ran different queries, missed edge cases
3. **Coverage**: Some accounts skipped due to time pressure

The security team needed a way to automate routine audit queries and produce consistent, repeatable reports.

## Solution: Parameterized Audit Scripts

### Architecture

```
audit-scripts/
├── scripts/
│   ├── iam-unused-users.sh
│   ├── s3-public-buckets.sh
│   ├── ec2-unencrypted-volumes.sh
│   ├── security-group-audit.sh
│   ├── kms-key-rotation.sh
│   └── ...
├── config/
│   ├── accounts.txt          # List of account IDs
│   ├── regions.txt           # Regions to scan
│   └── thresholds.yaml       # Configurable thresholds
├── lib/
│   ├── aws_helpers.sh        # Shared functions
│   └── report_formatter.sh   # CSV/JSON output
└── README.md
```

### Design Principles

1. **Idempotent**: Running the same script twice produces the same result
2. **Non-destructive**: Read-only operations only (no modifications)
3. **Parameterized**: Thresholds and scope configurable, not hardcoded
4. **Machine-readable**: Output in CSV/JSON for downstream processing
5. **Human-readable**: Clear headers and descriptions in output

### Example Script

```bash
#!/usr/bin/env bash
# iam-unused-users.sh
# Find IAM users who haven't logged in for N days

set -euo pipefail
source "$(dirname "$0")/../lib/aws_helpers.sh"

THRESHOLD_DAYS="${IAM_UNUSED_THRESHOLD:-90}"
ACCOUNT_ID="${1:-$(aws sts get-caller-identity --query Account --output text)}"

echo "account_id,username,last_login,days_inactive,access_keys,mfa_enabled"

aws iam list-users --output json | \
  jq -r '.Users[] | [.UserName, .CreateDate] | @tsv' | \
  while IFS=$'\t' read -r username created; do
    # Get last login activity
    last_login=$(aws iam get-login-profile \
      --user-name "$username" 2>/dev/null | \
      jq -r '.LoginProfile.LastUsedDate // "never"' || echo "never")
    
    # Calculate days inactive
    if [[ "$last_login" == "never" ]]; then
      days_inactive="N/A"
    else
      days_inactive=$(( ($(date +%s) - $(date -d "$last_login" +%s)) / 86400 ))
    fi
    
    # Check access keys
    key_count=$(aws iam list-access-keys \
      --user-name "$username" | \
      jq '.AccessKeyMetadata | length')
    
    # Check MFA
    mfa=$(aws iam list-mfa-devices \
      --user-name "$username" | \
      jq '.MFADevices | length')
    mfa_enabled=$([[ $mfa -gt 0 ]] && echo "true" || echo "false")
    
    echo "$ACCOUNT_ID,$username,$last_login,$days_inactive,$key_count,$mfa_enabled"
  done
```

### Multi-Account Orchestration

```bash
#!/usr/bin/env bash
# run-all-audits.sh
# Execute all audit scripts across all accounts

ACCOUNTS=$(cat config/accounts.txt)
REGIONS=$(cat config/regions.txt)
DATE=$(date +%Y%m%d)
OUTPUT_DIR="reports/${DATE}"

mkdir -p "$OUTPUT_DIR"

for account in $ACCOUNTS; do
  echo "=== Auditing account: $account ==="
  
  # Assume role in target account
  export AWS_PROFILE="audit-${account}"
  
  # Run each audit script
  for script in scripts/*.sh; do
    name=$(basename "$script" .sh)
    echo "  Running $name..."
    
    # IAM audits are global (no region)
    if [[ "$name" == iam-* ]]; then
      bash "$script" "$account" > "${OUTPUT_DIR}/${account}_${name}.csv"
    else
      # Resource audits run per-region
      for region in $REGIONS; do
        AWS_DEFAULT_REGION="$region" \
          bash "$script" "$account" > "${OUTPUT_DIR}/${account}_${name}_${region}.csv"
      done
    fi
  done
done

# Generate summary
python3 lib/summarize.py "$OUTPUT_DIR" > "${OUTPUT_DIR}/summary.html"
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Audit cycle time | 2-3 weeks | 4 hours (automated) + 2 hours review |
| Account coverage | ~70% (time-limited) | 100% |
| Consistency | Variable | Identical queries every cycle |
| Analyst time | 80-120 hours | 6 hours |
| Findings missed | Unknown | Zero (comprehensive) |

**95% reduction in analyst time** while improving coverage and consistency.

## Key Design Decisions

### Why Bash + AWS CLI (not Python/Boto3)?

- **Lower barrier to entry**: Security analysts can read and modify bash scripts
- **No dependency management**: AWS CLI available everywhere
- **Pipe-friendly**: Easy to compose with `jq`, `grep`, `sort`
- **Transparency**: Every AWS API call is visible in the script

### Why CSV output (not JSON)?

- **Spreadsheet compatibility**: Analysts open directly in Excel
- **Diff-friendly**: Line-by-line comparison between audit cycles
- **Database import**: Easy to load into audit tracking systems

### Why parameterized thresholds?

```yaml
# config/thresholds.yaml
iam_unused_days: 90
ec2_volume_unattached_days: 30
s3_bucket_age_review_days: 180
security_group_max_rules: 50
```

Different environments have different risk tolerances. Dev can use 180-day thresholds; prod uses 30-day. Same scripts, different config.

## Evolution

The scripts evolved through three phases:

1. **Phase 1**: Individual scripts, run manually per account
2. **Phase 2**: Orchestration script, multi-account, CSV output
3. **Phase 3** (planned): Lambda-based continuous audit, results to Security Hub

Phase 3 would convert each script into a Lambda function triggered on schedule, with results flowing to AWS Security Hub for centralized dashboarding.

## Why This Matters

This project demonstrates:
- **Operational automation**: Turning manual processes into repeatable scripts
- **Security tooling**: Building tools that security teams actually use
- **Multi-account patterns**: Cross-account role assumption and orchestration
- **Pragmatic engineering**: Choosing bash over Python because the users (security analysts) could maintain it themselves

The scripts are simple by design. That's the point — they're maintained by the security team, not the platform team. The best automation is the kind your users can modify themselves.