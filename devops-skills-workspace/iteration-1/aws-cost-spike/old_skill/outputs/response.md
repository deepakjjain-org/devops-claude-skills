# AWS Cost Spike Investigation: $45K to $63K (+40%)

Your bill jumped $18,000 in a single month. That is a 40% increase, which is significant and needs immediate root-cause analysis. Here is a systematic approach following the Discover, Analyze, Prioritize, Implement, Monitor workflow.

---

## Phase 1: DISCOVER -- Find What Changed

### Step 1: Detect the Anomaly Source

Run the cost anomaly detector to pinpoint which services and days drove the spike:

```bash
python3 scripts/cost_anomaly_detector.py --days 30
```

This will output:
- Daily cost trends showing exactly when the spike occurred
- Top cost drivers by service
- Period-over-period comparison (this month vs last month)
- A 30-day forecast so you can see if the trend is continuing

### Step 2: Find Unused and Wasted Resources

```bash
python3 scripts/find_unused_resources.py --region us-east-1
```

This scans for:
- Unattached EBS volumes (common hidden cost -- someone terminates an instance but the volume persists)
- Old EBS snapshots (accumulate silently)
- Unused Elastic IPs ($3.65/month each when unattached)
- Idle NAT Gateways ($32.85/month each + data processing charges)
- Idle EC2 instances (running but doing nothing)
- Unused load balancers ($16-22/month each)

### Step 3: Manual Cost Explorer Investigation

While the scripts run, open AWS Cost Explorer and do these checks immediately:

1. **Service breakdown**: Filter by service, compare March vs February. Which service jumped the most -- EC2, RDS, data transfer, or something else?
2. **Daily granularity**: Switch to daily view. Was it a sudden spike on one day (resource creation event) or a gradual ramp?
3. **Region check**: Confirm all the cost increase is in us-east-1. Sometimes someone launches resources in another region by accident.
4. **Linked accounts**: If you use AWS Organizations, check if the spike is in one specific account.

### Step 4: Check CloudTrail for Resource Creation Events

Look for resources that were created in the period the costs spiked:

```bash
# Check for large EC2 instances launched recently
aws ec2 describe-instances \
  --region us-east-1 \
  --query 'Reservations[].Instances[?LaunchTime>=`2026-03-01`].[InstanceId,InstanceType,LaunchTime,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Check for new RDS instances
aws rds describe-db-instances \
  --region us-east-1 \
  --query 'DBInstances[?InstanceCreateTime>=`2026-03-01`].[DBInstanceIdentifier,DBInstanceClass,Engine,MultiAZ]' \
  --output table
```

---

## Phase 2: ANALYZE -- Common Causes for EC2 + RDS Cost Spikes

Since you are mainly running EC2 and RDS in us-east-1, here are the most likely culprits ranked by probability:

### Most Likely Cause 1: Reserved Instances or Savings Plans Expired

This is the number one cause of sudden bill increases when nothing else changed. If RIs expired, those same instances are now being billed at on-demand rates, which can be 40-65% more expensive.

**Check immediately:**
```bash
# Check RI expiration
aws ec2 describe-reserved-instances \
  --region us-east-1 \
  --filters Name=state,Values=retired \
  --query 'ReservedInstances[?End>=`2026-02-01`].[ReservedInstancesId,InstanceType,InstanceCount,End]' \
  --output table

# Check RDS RI expiration
aws rds describe-reserved-db-instances \
  --region us-east-1 \
  --query 'ReservedDBInstances[?State==`retired`].[ReservedDBInstanceId,DBInstanceClass,DBInstanceCount]' \
  --output table
```

If RIs expired, that alone could explain the $18K jump. The fix: purchase new RIs or Savings Plans (see Phase 4).

### Most Likely Cause 2: New or Upsized Instances

Someone launched new EC2/RDS instances or scaled up existing ones. Run the rightsizing analyzer to see current utilization:

```bash
python3 scripts/rightsizing_analyzer.py --region us-east-1 --days 30
```

This will identify:
- EC2 instances with low CPU utilization (candidates for downsizing)
- RDS instances with low CPU/connection utilization
- Recommended smaller instance types
- Estimated savings from rightsizing

### Most Likely Cause 3: Auto Scaling Event or Runaway Instances

If you use Auto Scaling Groups, check if the group scaled out and never scaled back in:

```bash
aws autoscaling describe-auto-scaling-groups \
  --region us-east-1 \
  --query 'AutoScalingGroups[].[AutoScalingGroupName,DesiredCapacity,MinSize,MaxSize]' \
  --output table
```

### Most Likely Cause 4: RDS Multi-AZ Enabled or Storage Growth

RDS Multi-AZ deployments cost 2x. If someone enabled Multi-AZ on a large instance, that could be a big jump. Also check if RDS storage auto-scaling triggered:

```bash
aws rds describe-db-instances \
  --region us-east-1 \
  --query 'DBInstances[].[DBInstanceIdentifier,DBInstanceClass,MultiAZ,AllocatedStorage,StorageType]' \
  --output table
```

### Most Likely Cause 5: Data Transfer Costs

Data transfer between AZs, regions, or out to the internet can spike unexpectedly. Check Cost Explorer filtered to "Data Transfer" service.

### Most Likely Cause 6: Old-Generation Instances Costing More Than Necessary

```bash
python3 scripts/detect_old_generations.py --region us-east-1
```

This identifies instances running on old generations (t2, m4, c4) that cost more per unit of performance. While this is not the cause of the spike, fixing it reduces your baseline cost.

---

## Phase 3: PRIORITIZE -- Quick Wins to Present to the CFO

Once you have identified the root cause, here is how to prioritize the response:

### Tier 1: Immediate Actions (This Week) -- Low Risk, High Impact

| Action | Estimated Monthly Savings | Effort |
|--------|--------------------------|--------|
| Delete unattached EBS volumes | $50-500 depending on count | 1 hour |
| Release unused Elastic IPs | $3.65 each | 30 min |
| Delete old snapshots (>90 days) | $100-1,000+ | 1 hour |
| Stop/terminate confirmed idle EC2 instances | Varies widely | 2 hours |
| Convert gp2 volumes to gp3 | 20% of EBS costs, no downtime | 2 hours |

### Tier 2: This Month -- Medium Effort

| Action | Estimated Monthly Savings | Effort |
|--------|--------------------------|--------|
| Purchase RIs/Savings Plans for stable workloads | 30-65% of compute costs | 1 day analysis |
| Rightsize oversized EC2 instances | 20-50% per instance | 1-2 days |
| Rightsize oversized RDS instances | 20-50% per instance | 1 day |
| Migrate t2 instances to t3 | ~10% savings each | 1 day |
| Convert RDS gp2 storage to gp3 | 20% of RDS storage costs | 1 day |

### Tier 3: Strategic (This Quarter)

| Action | Estimated Monthly Savings | Effort |
|--------|--------------------------|--------|
| Migrate to Graviton (ARM64) instances | 20% of compute | 1-2 weeks |
| Implement Spot instances for fault-tolerant workloads | Up to 70% | 1-2 weeks |
| Replace NAT Gateways with VPC Endpoints | $25-30/month each | 1 week |
| Implement scheduled shutdown for dev/test | 65% of non-prod compute | 1 week |

---

## Phase 4: IMPLEMENT -- Specific Recommendations for Your Situation

### If Reserved Instances Expired (Most Likely Scenario)

Run the RI analysis to determine what to purchase:

```bash
python3 scripts/analyze_ri_recommendations.py --days 60
```

**Decision framework for new commitments:**
- Instances running 24/7 for 60+ days with stable workloads --> Standard RI (up to 63% savings)
- Workloads that may change instance types --> Convertible RI or Compute Savings Plan
- Variable workloads across instance families --> Compute Savings Plan (66% savings)
- Maximum flexibility needed --> Compute Savings Plan

For a $63K monthly bill mainly on EC2 and RDS, getting 60-70% of your compute covered by RIs or Savings Plans could save $10-15K/month.

### If New Resources Were Created

Identify who launched them and why. Check tags:
```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType,Tags[?Key==`Owner`].Value|[0],Tags[?Key==`Project`].Value|[0],Tags[?Key==`Environment`].Value|[0]]' \
  --output table
```

If resources are untagged, that is itself a governance problem to fix.

### If Auto Scaling Over-Provisioned

Review and tighten your Auto Scaling Group configurations:
- Set appropriate maximum capacity limits
- Review scaling policies (thresholds may be too aggressive)
- Implement scheduled scaling for known traffic patterns

---

## Phase 5: MONITOR -- Prevent This From Happening Again

### Set Up AWS Budgets (Do This Today)

```yaml
Budget Name: Monthly-AWS-Budget
Amount: $50,000/month (set to your target)
Alerts:
  - 50% actual spend: Email finance + engineering leads
  - 80% actual spend: Email CFO + CTO + engineering leads
  - 100% forecasted: Email everyone + Slack alert
  - 100% actual: Email everyone + automated investigation
```

### Enable AWS Cost Anomaly Detection

This is a free AWS service that uses ML to detect anomalies automatically:
1. Go to AWS Cost Management --> Cost Anomaly Detection
2. Create a monitor for your AWS services
3. Set alert thresholds (e.g., alert when daily cost exceeds $500 above normal)
4. Configure SNS notifications to Slack/email

### Implement Monthly Cost Reviews

Use the monthly report template to establish a recurring review process:

```bash
# Run all optimization scripts monthly
python3 scripts/find_unused_resources.py --region us-east-1
python3 scripts/cost_anomaly_detector.py --days 30
python3 scripts/rightsizing_analyzer.py --region us-east-1 --days 30
```

Schedule these for the first week of each month and review findings with the engineering team.

### Implement Tagging Governance

Enforce required tags on all resources so future cost spikes can be attributed:
- **Environment**: prod, staging, dev, test
- **Owner**: team or person responsible
- **Project**: business initiative
- **CostCenter**: for chargeback

Use AWS Organizations Service Control Policies to deny resource creation without required tags.

---

## What to Tell the CFO Right Now

Before you have completed the full analysis, here is what you can communicate:

> "We have identified that our AWS bill increased from $45K to $63K last month, a 40% increase concentrated in EC2 and RDS in us-east-1. We are running automated diagnostics to pinpoint the exact cause -- the most likely explanation is [expired Reserved Instances / new resource provisioning / auto-scaling event]. We expect to have root cause identified within 24-48 hours and a remediation plan within the week. We are also implementing budget alerts and monthly cost reviews to prevent undetected cost increases going forward."

**Key metrics to share:**
- The $18K increase represents a 40% month-over-month jump
- Root cause investigation is underway
- Budget alerts are being implemented immediately
- Monthly cost review process will be established
- Target: bring costs back to $45-50K range within 30 days (or provide a justified explanation for the higher baseline)

---

## Scripts Reference

All scripts are located in the `scripts/` directory and require `boto3` and `tabulate`:

```bash
pip install boto3 tabulate
```

| Script | Purpose | Command |
|--------|---------|---------|
| `cost_anomaly_detector.py` | Find what changed and when | `python3 scripts/cost_anomaly_detector.py --days 30` |
| `find_unused_resources.py` | Find wasted resources | `python3 scripts/find_unused_resources.py --region us-east-1` |
| `rightsizing_analyzer.py` | Find oversized instances | `python3 scripts/rightsizing_analyzer.py --region us-east-1 --days 30` |
| `detect_old_generations.py` | Find old instance types | `python3 scripts/detect_old_generations.py --region us-east-1` |
| `analyze_ri_recommendations.py` | RI purchase analysis | `python3 scripts/analyze_ri_recommendations.py --days 60` |
| `spot_recommendations.py` | Spot instance candidates | `python3 scripts/spot_recommendations.py` |

Run these in order: anomaly detector first (to understand what changed), then unused resources (quick wins), then rightsizing (medium-term savings), then RI analysis (long-term savings).
