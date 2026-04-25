# AWS Cost Estimate (Monthly)

Assumptions:

- Region: us-east-1, on-demand pricing as of 2026-04-25 (rounded)
- Month length: 720 hours
- Prices shown in USD
- Architecture from docs/ARCHITECTURE.md

## Baseline (Normal Traffic)

EC2 - Node.js app servers:

- 8 x t3.large @ $0.0832/hr
- Cost = 8 _ 0.0832 _ 720 = $479.23

EC2 - Payment workers:

- 4 x t3.medium @ $0.0416/hr
- Cost = 4 _ 0.0416 _ 720 = $119.81

RDS PostgreSQL primary:

- 1 x db.m6g.large @ $0.154/hr
- Cost = 1 _ 0.154 _ 720 = $110.88

RDS PostgreSQL read replicas:

- 2 x db.m6g.large @ $0.154/hr
- Cost = 2 _ 0.154 _ 720 = $221.76

ElastiCache Redis:

- 2 x cache.r6g.large @ $0.185/hr
- Cost = 2 _ 0.185 _ 720 = $266.40

ALB:

- Base cost = 0.0225/hr \* 720 = $16.20
- LCU cost (avg 5 LCUs) = 5 _ 0.008/hr _ 720 = $28.80

CloudFront:

- Data transfer: 30,000 GB \* $0.085/GB = $2,550.00
- Requests: 600,000,000 / 10,000 \* $0.0075 = $450.00

SQS:

- Messages: 300,000,000 / 1,000,000 \* $0.40 = $120.00

Baseline total: $4,363.08 per month

## Peak Event (World Cup Night, 4-hour window)

Additional EC2 - Node.js app servers:

- Scale to 120 total, +112 above baseline
- Cost = 112 _ 0.0832 _ 4 = $37.27

Additional EC2 - Payment workers:

- Scale to 40 total, +36 above baseline
- Cost = 36 _ 0.0416 _ 4 = $5.99

ALB LCU surge:

- 50 LCUs during peak (45 above baseline)
- Cost = 45 _ 0.008 _ 4 = $1.44

CloudFront surge transfer:

- Extra 15,000 GB \* $0.085/GB = $1,275.00
- Extra requests: 2,000,000,000 / 10,000 \* $0.0075 = $1,500.00

SQS surge:

- Extra 2,000,000 messages / 1,000,000 \* $0.40 = $0.80

Peak event add-on cost (4 hours): $2,820.50

## Business Justification (ROI)

Estimated revenue loss in a 45-minute outage:

- Loss rate = INR 4.2 crore per minute
- Total loss = 4.2 \* 45 = INR 189 crore

Even if the total monthly infrastructure cost is under $7,200 (baseline + one major peak event), it is negligible compared to a single outage loss of INR 189 crore. The ROI of the redesigned architecture is clearly positive.
