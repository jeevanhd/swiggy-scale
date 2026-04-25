# Architecture Redesign - SwiftEats

## Section 1 - Current Architecture Diagram (Monolith)

[Clients]
|
v
[Single Node.js Express]
| (1 process, 1 CPU, 4GB) <-- Failure 2: Event loop saturates at ~12-15k RPS
|
+--> [PostgreSQL]
| max_connections=100 <-- Failure 1: Pool exhausts at ~340-1,455 RPS
|
+--> [Razorpay]
| sync call 200-2000ms <-- Failure 3: Payment amplification
|
+--> [Promo Codes Table]
| no locking strategy <-- Failure 4: Promo race condition
|
+--> [Static Assets]
served by Node.js <-- Failure 5: NIC saturation (no CDN)

## Section 2 - New Architecture Diagram (Handles 10M Users)

[Clients]
|
v
[CloudFront CDN] - Caches images, CSS, JS (TTL 24h) - Caches HTML shell (TTL 60s)
|
v
[ALB] - SSL termination - Health checks - Rate limiting by path
|
v
[Node.js ASG] - min 8, max 120 - scale on RequestCountPerTarget > 1,000 RPS or CPU > 60%
|
+--> [Redis Cache]
| - Menu, restaurant list, cart pricing (TTL 5m)
| - Promo apply uses atomic lock + idempotency key
|
+--> [PgBouncer]
| - Transaction pooling, protects DB connections
|
+--> [PostgreSQL Primary]
| - Writes and payments
|
+--> [Read Replicas x2]
| - Serve read-only traffic
|
+--> [SQS Payments Queue] - Checkout returns immediately with order pending
|
v
[Payment Worker ASG] - Reads SQS, executes Razorpay calls - Writes payment status back to DB

## Section 3 - Component Justification Table

| Component                  | Failure It Prevents                      | How It Prevents It                                                       |
| -------------------------- | ---------------------------------------- | ------------------------------------------------------------------------ |
| CloudFront CDN             | Failure 5: Static asset NIC saturation   | Serves static assets from edge; Node never receives image requests.      |
| ALB                        | Failure 2: Node.js event loop saturation | Distributes load across instances, health checks keep bad nodes out.     |
| Node.js Auto Scaling Group | Failure 2: Node.js event loop saturation | Adds capacity during spikes; keeps per-node RPS below saturation.        |
| Redis Cache                | Failure 1 and Failure 4                  | Reduces DB reads and uses atomic promo locks to prevent race conditions. |
| PgBouncer                  | Failure 1: PostgreSQL pool exhaustion    | Pools and multiplexes connections to keep DB under max_connections.      |
| PostgreSQL Read Replicas   | Failure 1: PostgreSQL pool exhaustion    | Offloads read traffic from primary; fewer connections on primary.        |
| SQS Payments Queue         | Failure 3: Payment amplification         | Decouples request path from payment latency; no DB hold during payment.  |
| Payment Worker Service     | Failure 3: Payment amplification         | Processes payments asynchronously with controlled concurrency.           |
