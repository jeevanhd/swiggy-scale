# Failure Cascade Analysis - SwiftEats Monolith

## Step 2 - System Understanding (Before Documenting)

- The system is a single Node.js Express process (1 CPU, 4GB) with a single PostgreSQL instance (max_connections = 100), no cache, no CDN, and synchronous Razorpay payment calls that hold a DB connection for 200-2000ms.
- Weakest component: PostgreSQL connection pool. Hard limit is 100 concurrent connections. It fails first at roughly 700 RPS given the payment hold time.
- Hard compute limit: single Node.js event loop saturates around 12,000-15,000 RPS, with OOM risk at ~15,000 queued requests.
- The system has no load balancer and no auto-scaling, so any spike hits the single process directly and permanently until manual intervention.

## Section 1 - Traffic Simulation Math

Assumptions for the first 60 seconds after the push:

- Total users notified: 180,000,000
- Click-through rate: 8%
- Active users in spike window: 180,000,000 \* 0.08 = 14,400,000 users
- API calls per active user in first 60s: 8
  - App open, home feed, restaurant list, menu, cart, apply promo, place order, payment

Peak API RPS:

- Peak RPS = (active users \* calls per user) / 60
- Peak RPS = (14,400,000 \* 8) / 60 = 1,920,000 RPS

Static asset requests (no CDN) in the same window:

- Average static requests per user: 4 images
- Static asset RPS = (14,400,000 \* 4) / 60 = 960,000 RPS

## Section 2 - Component Capacity Numbers

Hard limits and key numbers:

- PostgreSQL max_connections = 100
- Node.js event loop saturation ~12,000-15,000 RPS on a single t3.medium
- Synchronous payment call hold time: 200-2,000ms (0.2-2.0s) holding a DB connection
- Node.js heap limit: 4GB -> OOM at ~15,000 queued requests

DB pool exhaustion math:

- Total RPS = R
- Payment fraction = 1/8 = 0.125
- Non-payment fraction = 7/8 = 0.875
- Non-payment query time = 50ms = 0.05s
- Payment hold time (average) = 0.8s (range 0.2-2.0s)

Connections held at any moment:

- Held = (0.875 _ R _ 0.05) + (0.125 _ R _ 0.8)
- Held = R _ (0.04375 + 0.1) = R _ 0.14375

Pool exhaustion when Held = 100:

- R = 100 / 0.14375 = 696 RPS

Range using payment hold time bounds:

- If hold time = 0.2s, Held = R \* 0.06875 -> R = 1,455 RPS
- If hold time = 2.0s, Held = R \* 0.29375 -> R = 340 RPS

Conclusion: DB pool exhausts between ~340 and ~1,455 RPS, well below the expected 1,920,000 RPS.

## Section 3 - The Cascade (Failures and Triggers)

Failure 1: PostgreSQL connection pool exhaustion (CRITICAL)

- Trigger RPS: ~340-1,455 RPS (using payment hold time range), ~696 RPS at 0.8s average
- User impact: API calls hang, then return 500/timeout errors. Checkout and feeds fail.
- Next: timeouts and retries pile up, pushing Node.js into event loop saturation and amplifying payment delays.

Failure 2: Node.js event loop saturation (CRITICAL)

- Trigger RPS: ~12,000-15,000 RPS; effectively earlier due to blocked callbacks from DB timeouts
- User impact: request latency explodes, connections pile up, 502/504 from clients.
- Next: queued requests blow up heap, and retries intensify DB pressure and payment failures.

Failure 3: Synchronous payment call amplification (HIGH)

- Trigger RPS: payment RPS = 0.125 \* total RPS (at peak, ~240,000 payment RPS)
- User impact: payment calls stall for 200-2,000ms, causing repeated taps and duplicate attempts.
- Next: DB pool holds longer, pushing Failure 1 earlier, and more retries saturate the event loop.

Failure 4: Promo code race condition (HIGH)

- Trigger RPS: promo apply RPS = 0.125 \* total RPS (same as payment call rate)
- User impact: duplicate discounts, lock contention, or 409/500 errors on promo apply.
- Next: user retries increase load on DB and app, worsening Failures 1 and 2.

Failure 5: Static asset NIC saturation (HIGH)

- Trigger RPS: ~960,000 asset RPS with images served from the Node.js process
- User impact: images and static files time out, app UI breaks, perceived outages even before API failures.
- Next: slower asset responses tie up Node.js sockets, increasing event loop pressure and cascading into Failures 1 and 2.

## Section 4 - Timeline (T+0s to T+2h)

- T+0s: Push notification delivered to 180M users.
- T+5s: ~14.4M users open the app; requests surge toward 1.9M RPS.
- T+10s: Static asset NIC saturates; image and JS loads stall.
- T+15s: DB connections hit 100; request timeouts begin.
- T+20s: Retry storms begin from mobile clients; p95 latency > 10s.
- T+30s: Node.js event loop saturates; heap grows rapidly.
- T+45s: Payment calls stall; duplicate payment attempts spike.
- T+60s: Promo code locks and conflicts trigger; promo apply failures rise.
- T+2m: 5xx rate > 80%; app effectively down for new sessions.
- T+10m: Manual intervention begins; on-call tries to scale vertically (limited).
- T+30m: Partial recovery as traffic drops; DB connections still pegged.
- T+45m: Payments stabilize, but order placement remains unreliable.
- T+1h: Traffic declines; system recovers gradually with high error rates.
- T+2h: Services return to baseline with residual failed orders and refunds required.
