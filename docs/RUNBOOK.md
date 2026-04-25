# Incident Runbook - SwiftEats

## STEP 1 - DETECT (CloudWatch Alarms)

| Alarm           | Metric                                  | Warning          | Critical         |
| --------------- | --------------------------------------- | ---------------- | ---------------- |
| RDS connections | RDS: DatabaseConnections                | > 85             | > 95             |
| RDS CPU         | RDS: CPUUtilization                     | > 70%            | > 85%            |
| ALB latency     | ALB: TargetResponseTime p95             | > 1s             | > 3s             |
| ALB 5xx         | ALB: HTTPCode_Target_5XX_Count          | > 1% of requests | > 5% of requests |
| App CPU         | EC2: CPUUtilization (app ASG)           | > 70%            | > 90%            |
| App RPS         | ALB: RequestCountPerTarget              | > 1,000 RPS      | > 2,000 RPS      |
| SQS backlog     | SQS: ApproximateNumberOfMessagesVisible | > 10,000         | > 50,000         |
| SQS age         | SQS: ApproximateAgeOfOldestMessage      | > 60s            | > 300s           |
| Redis miss rate | ElastiCache: CacheMissRate              | > 30%            | > 60%            |
| CloudFront 5xx  | CloudFront: 5xxErrorRate                | > 1%             | > 5%             |

## STEP 2 - TRIAGE (30-Second Decision Tree)

1. Check ALB 5xx and TargetResponseTime.
   - If ALB 5xx > 1% and RDS DatabaseConnections > 90 -> go to Step 3a (DB pool exhaustion).
   - If ALB 5xx > 1% and App CPU > 85% or RequestCountPerTarget > 2,000 -> go to Step 3b (compute saturation).
2. If ALB 5xx is normal but checkout delays are reported:
   - If SQS AgeOfOldestMessage > 60s or backlog > 10,000 -> go to Step 3c (payment queue backup).
3. If reads are slow but writes are OK:
   - If Redis CacheMissRate > 60% -> go to Step 3d (Redis cache miss spike).
4. If UI assets fail but APIs are OK:
   - If CloudFront 5xxErrorRate > 1% or OriginLatency spikes -> CDN/origin issue (escalate to Platform On-Call).

## STEP 3 - RESPOND (Component Actions)

### Step 3a - DB Pool Exhaustion

Owner: Database On-Call (#oncall-db)

Actions:

1. Confirm saturation:
   - RDS DatabaseConnections > 95
2. Scale primary DB vertically:
   - aws rds modify-db-instance --db-instance-identifier swifteats-primary --db-instance-class db.r6g.xlarge --apply-immediately
3. Reduce pressure by shifting reads:
   - aws rds create-db-instance-read-replica --db-instance-identifier swifteats-replica-3 --source-db-instance-identifier swifteats-primary

Success criteria:

- DatabaseConnections < 80 within 10-15 minutes
- ALB 5xx < 1% within 15 minutes

### Step 3b - Compute Saturation (Node.js ASG)

Owner: Platform On-Call (#oncall-platform)

Actions:

1. Scale out app instances:
   - aws autoscaling set-desired-capacity --auto-scaling-group-name swifteats-app-asg --desired-capacity 40
2. If still saturated after 5 minutes, raise max size:
   - aws autoscaling update-auto-scaling-group --auto-scaling-group-name swifteats-app-asg --max-size 160

Success criteria:

- App CPU < 60% within 10 minutes
- RequestCountPerTarget < 1,000 RPS within 10 minutes

### Step 3c - Payment Queue Backup (SQS)

Owner: Payments On-Call (#oncall-payments)

Actions:

1. Scale payment workers:
   - aws autoscaling set-desired-capacity --auto-scaling-group-name swifteats-payment-asg --desired-capacity 80
2. Increase visibility timeout if retries are excessive:
   - aws sqs set-queue-attributes --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/swe-payments --attributes VisibilityTimeout=120

Success criteria:

- AgeOfOldestMessage < 60s within 10 minutes
- ApproximateNumberOfMessagesVisible trending down

### Step 3d - Redis Cache Miss Spike

Owner: Cache On-Call (#oncall-cache)

Actions:

1. Scale Redis vertically:
   - aws elasticache modify-replication-group --replication-group-id swifteats-redis --cache-node-type cache.r6g.xlarge --apply-immediately
2. Increase TTLs for hot keys via config (example in SSM):
   - aws ssm put-parameter --name /swifteats/cache/ttl/menu --value 900 --type String --overwrite
   - aws ssm put-parameter --name /swifteats/cache/ttl/restaurants --value 900 --type String --overwrite

Success criteria:

- CacheMissRate < 20% within 15 minutes
- p95 read latency < 200ms

## STEP 4 - ROLLBACK (Application Only)

Rollback criteria:

- ALB 5xx > 5% for 15 minutes after scaling actions
- Data integrity issues (duplicate charges, corrupted orders)

Rollback command (previous stable release from S3):

- aws deploy create-deployment --application-name swifteats-app --deployment-group-name prod --description "rollback" --revision revisionType=S3,s3Location={bucket=swifteats-deploy,key=app/releases/previous.zip,bundleType=zip}

WARNING: Never roll back database schema. Roll back application code only.

## STEP 5 - POSTMORTEM TEMPLATE

Title:

- Incident name, date, and duration

Summary:

- One paragraph on what happened, when, and how it was resolved

Impact:

- Affected users, error rate, revenue impact, and regions impacted

Timeline (fill with precise timestamps):

- T+0m: What triggered the incident
- T+Xm: Key escalations and actions
- T+Ym: Stabilization time

Root Cause:

- Direct cause (technical)
- Contributing factors (process/system)

What Worked:

- Actions or systems that reduced impact

What Did Not Work:

- Missed alarms, slow escalations, or incorrect assumptions

Action Items:

- [ ] Action item description - Owner - Due date
- [ ] Action item description - Owner - Due date

Links:

- Dashboards, logs, tickets, and deploy IDs used during the incident
