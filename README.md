# SwiftEats Scale Analysis

A failure analysis and redesign plan for a food delivery monolith under an extreme traffic spike.

## Scenario

India vs Pakistan World Cup Final, 8 PM IST. A 50% off promo is sent to 180 million users. About 14.4 million users open the app in 60 seconds. The legacy system is a single Node.js Express process with a single PostgreSQL database, no cache, no CDN, synchronous payments, and no auto-scaling.

## Document Summary

| Document                | What it contains                                               |
| ----------------------- | -------------------------------------------------------------- |
| docs/FAILURE-CASCADE.md | RPS math, hard limits, and the failure cascade timeline        |
| docs/ARCHITECTURE.md    | Current vs redesigned architecture and component justification |
| docs/COST-ESTIMATE.md   | Baseline and peak AWS cost calculations with instance types    |
| docs/RUNBOOK.md         | 5-step incident response guide and postmortem template         |

## Key Findings

- Peak API traffic is ~1,920,000 RPS in the first 60 seconds.
- PostgreSQL pool exhausts at roughly 340-1,455 RPS (100 connections), far below peak.
- Static assets alone hit ~960,000 RPS and saturate a single NIC without a CDN.
- Node.js event loop saturation occurs around 12,000-15,000 RPS, making the single process a hard cap.

## Architecture Overview

The redesigned system adds CloudFront, an ALB, a horizontally scaled Node.js tier, Redis caching, PgBouncer, read replicas, and an SQS-driven payment worker service. The design decouples payment latency and protects the database with pooling and cache offload to survive a 10 million user spike.

## Tech Stack Context

Node.js, PostgreSQL, Redis, and AWS (EC2, RDS, ElastiCache, ALB, CloudFront, SQS).
