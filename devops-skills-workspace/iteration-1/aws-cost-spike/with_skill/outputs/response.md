# AWS Cost Spike Investigation: $45K to $63K (+40%)

Your bill jumped $18,000 in a single month -- a 40% increase. This is significant and requires immediate, systematic investigation. Below is a structured action plan following the Discover-Analyze-Prioritize-Implement-Monitor workflow, tailored to your EC2 and RDS workloads in us-east-1.

---

## Phase 1: Discover -- Find the Source of the Spike (Do This Today)

### Step 1: Run Cost Anomaly Detection

```bash
python3 scripts/cost_anomaly_detector.py --days 30 --region us-east-1
```

This script will:
- Analyze daily Cost Explorer data for spending spikes
- Identify the exact dates the cost increase occurred
- Show which specific AWS service drove the increase
- Compare the last 30 days against the prior 30 days, service by service
- Generate a 30-day forecast so you can tell the CFO what to expect next month

**What to look for in the output:**
- The "COST ANOMALIES DETECTED" section will show specific dates with unusual spending and the top service responsible for each spike
- The "PERIOD-OVER-PERIOD COMPARISON" section will show which services increased the most in dollar terms -- this is your smoking gun

### Step 2: Find Unused Resources Bleeding Money

```bash
python3 scripts/find_unused_resources.py --region us-east-1
```

This scans for:
- Unattached EBS volumes (you are paying for storage with nothing using it)
- Old EBS snapshots (>90 days, accumulating quietly)
- Unused Elastic IPs ($3.65/month each -- small individually, adds up)
- Idle NAT Gateways ($32.85/month each + data processing charges)
- Idle EC2 instances (running but with <5% CPU utilization)
- Load balancers with no healthy targets

### Step 3: Check AWS Cost Explorer Manually (Belt and Suspenders)

While scripts run, open Cost Explorer in the AWS Console:

1. **Group by Service** -- Confirm whether EC2 or RDS is the primary driver
2. **Group by Usage Type** -- Identify if the spike is compute hours, data transfer, storage, or something else
3. **Group by Linked Account** -- If you use AWS Organizations, check if a specific account caused the spike
4. **Filter to us-east-1** -- Confirm the spike is regional or global
5. **Daily granularity** -- Find the exact date the increase began

---

## Phase 2: Analyze -- Identify the Most Likely Causes

Based on the fact that you are "mainly running EC2 and RDS in us-east-1," here are the most common causes of a $18K spike, ranked by likelihood:

### Cause 1: Reserved Instances or Savings Plans Expired

**Why this is #1**: An RI expiration instantly reverts instances to on-demand pricing, which can be 40-63% more expensive. If you had $30K of RI-covered EC2/RDS and those RIs expired, the jump to on-demand pricing would add exactly $12K-$19K to your bill.

**How to check:**
```bash
# Check RI status
aws ec2 describe-reserved-instances --region us-east-1 \
  --filters "Name=state,Values=retired" \
  --query 'ReservedInstances[].{Type:InstanceType,Count:InstanceCount,End:End,State:State}'

# Check RDS RI status
aws rds describe-reserved-db-instances --region us-east-1 \
  --query 'ReservedDBInstances[?State==`retired`].{Class:DBInstanceClass,Count:DBInstanceCount,End:StartTime,State:State}'

# Check Savings Plans
aws savingsplans describe-savings-plans \
  --query 'SavingsPlans[?state==`expired`].{Type:savingsPlanType,Commitment:commitment,End:end}'
```

**If this is the cause**: Re-purchase RIs or Savings Plans immediately. Run the RI analysis script to get recommendations:

```bash
python3 scripts/analyze_ri_recommendations.py --days 60 --region us-east-1
```

### Cause 2: New Resources Were Provisioned

Someone may have launched new EC2 instances, scaled up RDS, or created resources without the team knowing.

**How to check:**
```bash
# Find EC2 instances launched in the last 30 days
aws ec2 describe-instances --region us-east-1 \
  --query 'Reservations[].Instances[?LaunchTime>=`2026-03-09`].{ID:InstanceId,Type:InstanceType,Launch:LaunchTime,Name:Tags[?Key==`Name`].Value|[0]}' \
  --output table

# Check CloudTrail for RunInstances events
aws cloudtrail lookup-events --region us-east-1 \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances \
  --start-time 2026-03-09 \
  --query 'Events[].{Time:EventTime,User:Username,Resources:Resources[0].ResourceName}'

# Check for RDS instance creation
aws cloudtrail lookup-events --region us-east-1 \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateDBInstance \
  --start-time 2026-03-09
```

### Cause 3: Auto Scaling Responded to Load (And Never Scaled Back)

If Auto Scaling Groups scaled out during a traffic spike and the scale-in policy is misconfigured, instances keep running at peak capacity.

**How to check:**
```bash
# List Auto Scaling Groups and their current vs desired capacity
aws autoscaling describe-auto-scaling-groups --region us-east-1 \
  --query 'AutoScalingGroups[].{Name:AutoScalingGroupName,Min:MinSize,Max:MaxSize,Desired:DesiredCapacity,Instances:Instances|length(@)}' \
  --output table

# Check scaling activities in the last 30 days
aws autoscaling describe-scaling-activities --region us-east-1 \
  --query 'Activities[?StartTime>=`2026-03-09`].{ASG:AutoScalingGroupName,Cause:Cause,Time:StartTime}' \
  --output table
```

### Cause 4: RDS Instances Were Scaled Up or Multi-AZ Was Enabled

RDS instance class changes or enabling Multi-AZ doubles the cost.

**How to check:**
```bash
# List all RDS instances with their current configuration
aws rds describe-db-instances --region us-east-1 \
  --query 'DBInstances[].{ID:DBInstanceIdentifier,Class:DBInstanceClass,Engine:Engine,MultiAZ:MultiAZ,Storage:AllocatedStorage,StorageType:StorageType}' \
  --output table

# Check for RDS modification events
aws cloudtrail lookup-events --region us-east-1 \
  --lookup-attributes AttributeKey=EventName,AttributeValue=ModifyDBInstance \
  --start-time 2026-03-09
```

### Cause 5: Data Transfer Costs Spiked

Data transfer is often an overlooked cost driver. Cross-region replication, NAT Gateway data processing, or increased egress traffic can add up fast.

**How to check:**
- In Cost Explorer, filter by Usage Type Group = "Data Transfer"
- Look for `DataTransfer-Out-Bytes`, `NatGateway-Bytes`, `DataTransfer-Regional-Bytes`

### Cause 6: EBS Volume or Snapshot Accumulation

Unmanaged snapshots and orphaned volumes accumulate silently.

**How to check** (already covered by `find_unused_resources.py` above).

---

## Phase 3: Prioritize -- Quick Wins for Immediate Savings

Once you have identified the cause, here are actions prioritized by impact and speed:

### Tier 1: Immediate Actions (This Week) -- Low Risk, High Impact

| Action | Expected Savings | Effort | Risk |
|--------|-----------------|--------|------|
| Re-purchase expired RIs/Savings Plans | $7K-12K/month | Low | None |
| Delete unattached EBS volumes | Varies | Low | Low (verify first) |
| Release unused Elastic IPs | $3.65/each/month | Low | Low |
| Terminate idle EC2 instances (<5% CPU) | Varies | Low | Verify with owners |
| Delete old snapshots (>90 days) | Varies | Low | Verify backups exist |

### Tier 2: This Month -- Medium Effort

| Action | Expected Savings | Effort | Risk |
|--------|-----------------|--------|------|
| Right-size oversized EC2 instances | 20-50% per instance | Medium | Test first |
| Right-size oversized RDS instances | 20-50% per instance | Medium | Maintenance window |
| Convert gp2 volumes to gp3 | 20% on EBS costs | Low | No downtime |
| Replace NAT Gateways with VPC Endpoints for S3/DynamoDB | $25-30/month each | Medium | Low |

Run the rightsizing analyzer to identify candidates:

```bash
python3 scripts/rightsizing_analyzer.py --days 30 --region us-east-1
```

### Tier 3: Next Quarter -- Strategic

| Action | Expected Savings | Effort | Risk |
|--------|-----------------|--------|------|
| Migrate old-gen instances (t2 to t3, m4 to m5) | 5-10% | Medium | Low |
| Evaluate Graviton/ARM64 migration | 20% | High | Test required |
| Implement Spot instances for fault-tolerant workloads | Up to 70% | High | Architecture change |
| Implement dev/test resource scheduling (stop nights/weekends) | 65% on non-prod | Medium | Low |

Run the generation detection script:

```bash
python3 scripts/detect_old_generations.py --region us-east-1
```

---

## Phase 4: Respond to the CFO -- What to Say

Here is a framework for the executive summary:

### Talking Points for CFO

1. **Acknowledge the increase**: "Our AWS bill increased from $45K to $63K, a 40% increase. We have identified the root cause(s) and are taking corrective action."

2. **Root cause** (fill in after investigation): "The primary driver was [expired Reserved Instances / new infrastructure provisioned for Project X / Auto Scaling that did not scale back / data transfer increase]."

3. **Immediate actions taken**: "We are [re-purchasing commitments / terminating unused resources / rightsizing over-provisioned instances] which will reduce costs by an estimated $X,XXX/month."

4. **Going forward**: "We are implementing cost monitoring safeguards including:
   - AWS Budgets with alerts at 80% and 100% thresholds
   - Monthly cost review process
   - Tag enforcement so every resource is traceable to a team and project
   - Cost anomaly detection to catch spikes within 24 hours"

5. **Forecast**: "Based on our corrective actions, we expect next month's bill to be approximately $[X]K."

### Set Up Immediate Guardrails

```bash
# Create a budget alert so this never surprises you again
aws budgets create-budget --account-id YOUR_ACCOUNT_ID --budget '{
  "BudgetName": "Monthly-AWS-Budget",
  "BudgetLimit": {"Amount": "50000", "Unit": "USD"},
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}' --notifications-with-subscribers '[{
  "Notification": {
    "NotificationType": "ACTUAL",
    "ComparisonOperator": "GREATER_THAN",
    "Threshold": 80
  },
  "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "cfo@company.com"}]
}, {
  "Notification": {
    "NotificationType": "FORECASTED",
    "ComparisonOperator": "GREATER_THAN",
    "Threshold": 100
  },
  "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "engineering-leads@company.com"}]
}]'
```

---

## Phase 5: Monitor -- Prevent Future Surprises

### Establish a Monthly Cost Review Process

Following the FinOps governance framework:

- **Week 1**: Run all optimization scripts, export Cost & Usage Reports
- **Week 2**: Analyze trends, identify new optimization opportunities
- **Week 3**: Present to engineering teams, assign action items
- **Week 4**: Executive reporting with forecast

Use the monthly report template to standardize reporting:
```bash
cp assets/templates/monthly_cost_report.md reports/2026-04-cost-report.md
```

### Enable AWS-Native Cost Monitoring

1. **AWS Cost Anomaly Detection** -- Enable in the Billing Console. It uses ML to automatically detect unusual spending and sends alerts.
2. **AWS Budgets** -- Set up per-environment and per-team budgets (see above).
3. **Cost Allocation Tags** -- Activate tags in Billing > Cost Allocation Tags. At minimum, enforce: `Environment`, `Owner`, `Project`, `CostCenter`.
4. **AWS Compute Optimizer** -- Enable for ML-based rightsizing recommendations that complement the scripts.

### Implement Tagging Enforcement

Without tags, you cannot attribute costs to teams or projects. Implement this Service Control Policy to require tags on all new EC2 instances:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "RequireTagsOnEC2",
    "Effect": "Deny",
    "Action": ["ec2:RunInstances"],
    "Resource": ["arn:aws:ec2:*:*:instance/*"],
    "Condition": {
      "StringNotLike": {
        "aws:RequestTag/Environment": ["prod", "staging", "dev", "test"],
        "aws:RequestTag/Owner": "*",
        "aws:RequestTag/Project": "*"
      }
    }
  }]
}
```

---

## Complete Script Execution Checklist

Run these in order for a complete cost investigation:

```bash
# 1. Detect anomalies and identify the spike source
python3 scripts/cost_anomaly_detector.py --days 30 --region us-east-1

# 2. Find unused resources bleeding money
python3 scripts/find_unused_resources.py --region us-east-1

# 3. Find oversized instances
python3 scripts/rightsizing_analyzer.py --days 30 --region us-east-1

# 4. Find old generation instances to migrate
python3 scripts/detect_old_generations.py --region us-east-1

# 5. Evaluate RI/Savings Plan opportunities
python3 scripts/analyze_ri_recommendations.py --days 60 --region us-east-1

# 6. Evaluate Spot instance candidates
python3 scripts/spot_recommendations.py --region us-east-1
```

All scripts support `--profile PROFILE` if you use named AWS profiles. Install dependencies first: `pip install boto3 tabulate`.

---

## Summary of Expected Savings Potential

| Optimization Category | Typical Savings Range | Timeline |
|-----------------------|----------------------|----------|
| Re-purchase expired RIs/Savings Plans | 40-63% on committed instances | Immediate |
| Delete unused resources | $500-5,000/month | This week |
| Rightsize EC2/RDS instances | 20-50% per instance | This month |
| Convert gp2 to gp3 EBS volumes | 20% on EBS spend | This week |
| Migrate to newer instance generations | 5-10% per instance | This month |
| Graviton migration (where compatible) | 20% per instance | Next quarter |
| Dev/test scheduling (stop off-hours) | 65% on non-prod compute | This month |
| Spot instances for fault-tolerant workloads | Up to 70% | Next quarter |

The $18K/month spike is very likely recoverable. Based on typical findings from these analyses, organizations at your spend level commonly find $8K-15K/month in combined savings from expired commitments, unused resources, and rightsizing alone.
