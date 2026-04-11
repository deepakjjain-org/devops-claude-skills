# AWS Cost Spike Investigation: $45K to $63K (+40%)

## Executive Summary

Your AWS bill increased by $18,000 (40%) month-over-month, primarily in us-east-1 running EC2 and RDS. Below is a systematic investigation playbook to identify the root cause and immediate actions to contain costs.

---

## Phase 1: Immediate Triage (Do This First - 30 Minutes)

### 1.1 Check AWS Cost Explorer for the Spike

```bash
# Get daily cost breakdown for the last 60 days, grouped by service
aws ce get-cost-and-usage \
  --time-period Start=2026-02-01,End=2026-04-01 \
  --granularity DAILY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --filter '{"Dimensions":{"Key":"REGION","Values":["us-east-1"]}}' \
  --output json
```

This will show you the exact day the cost increase started and which service(s) caused it.

### 1.2 Identify Top Cost Drivers by Service

```bash
# Compare last month vs previous month by service
aws ce get-cost-and-usage \
  --time-period Start=2026-03-01,End=2026-04-01 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --output table
```

### 1.3 Check for Usage Type Anomalies

```bash
# Break down EC2 costs by usage type (on-demand vs spot vs RI, data transfer, etc.)
aws ce get-cost-and-usage \
  --time-period Start=2026-03-01,End=2026-04-01 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=USAGE_TYPE \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Elastic Compute Cloud - Compute"]}}' \
  --output json
```

---

## Phase 2: Most Likely Culprits (EC2 + RDS Focus)

Based on the $18K increase with EC2 and RDS as primary services, here are the most common causes ranked by likelihood:

### 2.1 Reserved Instance or Savings Plan Expiration (MOST COMMON)

This is the #1 cause of sudden cost jumps. If an RI or Savings Plan expired, those instances silently flip to on-demand pricing, which can be 40-60% more expensive.

```bash
# Check RI status - look for recently expired reservations
aws ec2 describe-reserved-instances \
  --filters Name=state,Values=retired \
  --query 'ReservedInstances[?End>=`2026-02-01`].[ReservedInstancesId,InstanceType,InstanceCount,End,FixedPrice]' \
  --output table

# Check active RIs
aws ec2 describe-reserved-instances \
  --filters Name=state,Values=active \
  --output table

# Check RDS RIs
aws rds describe-reserved-db-instances \
  --query 'ReservedDBInstances[?State==`retired`].[ReservedDBInstanceId,DBInstanceClass,DBInstanceCount,StartTime,Duration]' \
  --output table

# Check Savings Plans
aws savingsplans describe-savings-plans \
  --query 'SavingsPlans[?state==`retired`].[savingsPlanId,savingsPlanType,end,commitment]' \
  --output table
```

**What to look for**: Any RI or Savings Plan that expired in the last 30-60 days. A single expired RI covering 10-20 instances can easily account for $18K/month.

### 2.2 New or Upsized EC2 Instances

Someone may have launched new instances or upsized existing ones without going through a proper change process.

```bash
# Find EC2 instances launched in the last 30 days
aws ec2 describe-instances \
  --region us-east-1 \
  --query 'Reservations[].Instances[?LaunchTime>=`2026-03-01`].[InstanceId,InstanceType,LaunchTime,Tags[?Key==`Name`].Value|[0],State.Name]' \
  --output table

# Count running instances by type
aws ec2 describe-instances \
  --region us-east-1 \
  --filters Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].[InstanceType]' \
  --output text | sort | uniq -c | sort -rn

# Check for large/expensive instance types
aws ec2 describe-instances \
  --region us-east-1 \
  --filters Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[?contains(`["x1","x2","p3","p4","p5","i3","r5","r6","m5","m6"]`,to_string(InstanceType))].[InstanceId,InstanceType,LaunchTime,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

### 2.3 RDS Instance Changes

```bash
# List all RDS instances with their class and multi-AZ status
aws rds describe-db-instances \
  --region us-east-1 \
  --query 'DBInstances[].[DBInstanceIdentifier,DBInstanceClass,MultiAZ,StorageType,AllocatedStorage,InstanceCreateTime]' \
  --output table

# Check for recent modifications (look at events)
aws rds describe-events \
  --region us-east-1 \
  --start-time 2026-03-01 \
  --source-type db-instance \
  --output table
```

**What to look for**: Instance class changes (e.g., db.r5.xlarge to db.r5.4xlarge), Multi-AZ being enabled (doubles cost), storage auto-scaling events.

### 2.4 Data Transfer Costs

Data transfer is a hidden cost driver. Inter-AZ, internet egress, or cross-region transfer can spike unexpectedly.

```bash
# Check data transfer costs
aws ce get-cost-and-usage \
  --time-period Start=2026-02-01,End=2026-04-01 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --filter '{"Dimensions":{"Key":"USAGE_TYPE","Values":["USE1-DataTransfer-Out-Bytes","USE1-DataTransfer-Regional-Bytes"]}}' \
  --output table
```

### 2.5 EBS Volume and Snapshot Growth

```bash
# Total EBS volume cost by type
aws ec2 describe-volumes \
  --region us-east-1 \
  --query 'Volumes[].[VolumeId,VolumeType,Size,State,CreateTime]' \
  --output table

# Count unattached volumes (wasted money)
aws ec2 describe-volumes \
  --region us-east-1 \
  --filters Name=status,Values=available \
  --query 'Volumes[].[VolumeId,VolumeType,Size]' \
  --output table

# Check snapshot count and growth
aws ec2 describe-snapshots \
  --owner-ids self \
  --region us-east-1 \
  --query 'length(Snapshots[])'

# Recent snapshots (last 30 days)
aws ec2 describe-snapshots \
  --owner-ids self \
  --region us-east-1 \
  --query 'Snapshots[?StartTime>=`2026-03-01`].[SnapshotId,VolumeSize,StartTime,Description]' \
  --output table
```

### 2.6 Elastic IPs and NAT Gateways

```bash
# Check for unused Elastic IPs ($3.65/month each since Feb 2024 pricing change - all EIPs now cost money)
aws ec2 describe-addresses \
  --region us-east-1 \
  --query 'Addresses[?AssociationId==null].[PublicIp,AllocationId]' \
  --output table

# NAT Gateway costs ($0.045/hr + data processing = ~$32/month base + data)
aws ec2 describe-nat-gateways \
  --region us-east-1 \
  --filter Name=state,Values=available \
  --query 'NatGateways[].[NatGatewayId,SubnetId,CreateTime]' \
  --output table
```

---

## Phase 3: Deeper Analysis

### 3.1 Enable AWS Cost Anomaly Detection (If Not Already)

```bash
# Create a cost anomaly monitor for immediate alerts
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "ServiceLevelMonitor",
    "MonitorType": "DIMENSIONAL",
    "MonitorDimension": "SERVICE"
  }'
```

### 3.2 Check CloudTrail for Infrastructure Changes

```bash
# Look for EC2 RunInstances events in the last 30 days
aws cloudtrail lookup-events \
  --region us-east-1 \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances \
  --start-time 2026-03-01 \
  --end-time 2026-04-01 \
  --query 'Events[].[EventTime,Username,CloudTrailEvent]' \
  --output table

# Look for RDS modifications
aws cloudtrail lookup-events \
  --region us-east-1 \
  --lookup-attributes AttributeKey=EventName,AttributeValue=ModifyDBInstance \
  --start-time 2026-03-01 \
  --end-time 2026-04-01 \
  --output table

# Check for Auto Scaling changes
aws cloudtrail lookup-events \
  --region us-east-1 \
  --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateAutoScalingGroup \
  --start-time 2026-03-01 \
  --end-time 2026-04-01 \
  --output table
```

### 3.3 Auto Scaling Group Review

```bash
# Check if ASG desired/max capacity increased
aws autoscaling describe-auto-scaling-groups \
  --region us-east-1 \
  --query 'AutoScalingGroups[].[AutoScalingGroupName,MinSize,MaxSize,DesiredCapacity,Instances[].InstanceType|[0]]' \
  --output table

# Check scaling activity history
aws autoscaling describe-scaling-activities \
  --region us-east-1 \
  --query 'Activities[?StartTime>=`2026-03-01`].[AutoScalingGroupName,Cause,StartTime,StatusCode]' \
  --output table
```

---

## Phase 4: Common Root Causes Checklist

Use this checklist to systematically rule in/out each cause:

| # | Potential Cause | Likelihood | Typical Impact | Check |
|---|----------------|-----------|----------------|-------|
| 1 | **RI/Savings Plan expiration** | HIGH | 30-60% increase on covered instances | Check RI/SP status |
| 2 | **New EC2 instances launched** | HIGH | Variable, depends on instance type | CloudTrail + instance list |
| 3 | **RDS instance upsized or Multi-AZ enabled** | MEDIUM | 2x on that specific instance | RDS events + describe |
| 4 | **Auto Scaling group scaled up and stayed up** | MEDIUM | Proportional to new instances | ASG scaling activities |
| 5 | **Data transfer spike** | MEDIUM | $0.09/GB egress adds up fast | Cost Explorer by usage type |
| 6 | **EBS volumes/snapshots accumulated** | LOW-MED | gp3 = $0.08/GB/month | Volume and snapshot counts |
| 7 | **Forgotten dev/test instances running** | MEDIUM | Full on-demand price 24/7 | Instances without proper tags |
| 8 | **RDS storage auto-scaling** | LOW | Depends on growth rate | RDS describe + events |
| 9 | **Elastic IP charges (idle)** | LOW | $3.65/IP/month | Describe addresses |
| 10 | **Cross-region replication enabled** | LOW | Data transfer + storage | Check S3/RDS replication |

---

## Phase 5: Immediate Cost Reduction Actions

Once you identify the cause, here are quick wins:

### If RIs/Savings Plans Expired
1. Purchase replacement RIs or Savings Plans immediately -- every day at on-demand pricing costs money
2. Consider Compute Savings Plans for flexibility (up to 66% savings)
3. Use the AWS RI Recommendations in Cost Explorer (Cost Explorer > Recommendations)

### If New Instances Were Launched
1. Identify who launched them and why (CloudTrail)
2. Right-size or terminate if not needed
3. Move to Spot if they are fault-tolerant workloads (up to 90% savings)

### If RDS Was Upsized
1. Verify the upsize was necessary (check CloudWatch CPUUtilization, FreeableMemory)
2. Consider Aurora Serverless v2 if usage is variable
3. Ensure read replicas are used instead of upsizing for read-heavy workloads

### Quick Wins for Any Situation
```bash
# Find stopped instances still incurring EBS costs
aws ec2 describe-instances \
  --region us-east-1 \
  --filters Name=instance-state-name,Values=stopped \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType,Tags[?Key==`Name`].Value|[0],StateTransitionReason]' \
  --output table

# Find unattached EBS volumes
aws ec2 describe-volumes \
  --region us-east-1 \
  --filters Name=status,Values=available \
  --query 'sum(Volumes[].Size)'

# Find old snapshots (older than 90 days)
aws ec2 describe-snapshots \
  --owner-ids self \
  --region us-east-1 \
  --query 'Snapshots[?StartTime<=`2026-01-01`] | length(@)'
```

---

## Phase 6: CFO Response Template

Once you have identified the root cause, here is a template for responding to the CFO:

> **Subject: AWS Cost Increase Analysis - March 2026**
>
> Our AWS costs increased from $45,000 to $63,000 (+$18,000 / +40%) in March. After investigation, the root cause was identified as **[ROOT CAUSE]**.
>
> **Root Cause**: [e.g., "Three Reserved Instance commitments covering 15 EC2 instances expired on March 1st, causing those instances to revert to on-demand pricing at approximately 2.5x the reserved rate."]
>
> **Immediate Actions Taken**:
> - [e.g., "Purchased replacement 1-year Compute Savings Plans covering our baseline compute usage"]
> - [e.g., "Identified and terminated 4 unused development instances ($2,100/month savings)"]
> - [e.g., "Right-sized 3 over-provisioned RDS instances ($1,800/month savings)"]
>
> **Expected Impact**: We expect the bill to return to approximately $[X] next month.
>
> **Preventive Measures**:
> - Set up AWS Cost Anomaly Detection for automatic alerts
> - Created budget alerts at $50K and $55K thresholds
> - Added RI/SP expiration tracking to our monthly review process
> - Implemented mandatory tagging policy for cost allocation

---

## Phase 7: Prevention Going Forward

### Set Up Budget Alerts
```bash
# Create a budget with alert at 90% of $50K
aws budgets create-budget \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget '{
    "BudgetName": "Monthly-AWS-Budget",
    "BudgetLimit": {"Amount": "50000", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
  }' \
  --notifications-with-subscribers '[{
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 90,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [{
      "SubscriptionType": "EMAIL",
      "Address": "your-team@company.com"
    }]
  }]'
```

### Enable Cost Allocation Tags
```bash
# Activate cost allocation tags for tracking
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status '[
    {"TagKey": "Environment", "Status": "Active"},
    {"TagKey": "Team", "Status": "Active"},
    {"TagKey": "Project", "Status": "Active"}
  ]'
```

### Monthly Review Process
1. **Week 1**: Review Cost Explorer for anomalies
2. **Week 2**: Check RI/SP utilization and upcoming expirations
3. **Week 3**: Review right-sizing recommendations
4. **Week 4**: Audit unused resources (volumes, snapshots, IPs, idle instances)

---

## Summary of Investigation Steps

1. **Start with Cost Explorer** to pinpoint the exact day and service causing the spike
2. **Check RI/Savings Plan expirations** -- this is the most common cause of sudden jumps
3. **Review CloudTrail** for infrastructure changes (new instances, upsizes, config changes)
4. **Audit running resources** for waste (unused instances, unattached volumes, idle resources)
5. **Check data transfer** costs which are often overlooked
6. **Take immediate action** on the root cause
7. **Set up prevention** (budgets, anomaly detection, tagging, monthly reviews)

The 40% increase ($18K) on a $45K base is significant but very common and almost always traceable to one of the causes above. Reserved Instance expiration is the single most frequent cause of this pattern -- a fleet of instances silently moving to on-demand pricing produces exactly this kind of sudden, unexplained jump.
