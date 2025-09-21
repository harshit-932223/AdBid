# Real-Time Bidding (RTB) System — Expected Performance

This section details the **performance expectations for each component** of a high-scale RTB system.

---

## 1. Request Ingestion
- **Components**: Envoy/NGINX Load Balancers + Kafka.
- **Latency**: <1–2 ms for load balancing and routing.  
- **Throughput**: Millions of bid requests/sec per cluster.  
- **Notes**: Kafka provides high-throughput, ordered, and durable message streaming. Tools like Kafka Streams, Connect, and Schema Registry simplify event processing, monitoring, and integration.

---

## 2. User Profile Retrieval
- **Components**: Redis Cluster / Aerospike / DynamoDB (with DAX).  
- **Latency**:
  - Redis/Aerospike: 0.2–1 ms for lookups per key.  
  - DynamoDB+DAX: 2–5 ms per read (stronger global consistency but higher latency).  
- **Throughput**: Millions of lookups/sec with clustering and horizontal scaling.  
- **Notes**: Sub-ms latency is critical as each auction request requires fetching user/device profiles.

---

## 3. Business Logic & Bidding Engine
- **Components**: Go / Node.js service + ML model serving (TF Serving / PyTorch Serve) + Campaign Config Store (Postgres + Redis cache).  
- **Latency**:
  - Auction orchestration & rules application: 10–20 ms.  
  - ML scoring (CTR/CVR prediction): 5–20 ms per request (can batch).  
  - Campaign config retrieval from Redis cache: <1 ms.  
- **Throughput**: 1K–10K auctions/sec per engine node; scales linearly with more nodes.  
- **Notes**: Go is preferred for low p99 latency under high concurrency.

---

## 4. Rule Evaluation Engine
- **Components**: OPA (Open Policy Agent) + optional precompiled Redis cache for fast rules.  
- **Latency**: ~1–2 ms per bid request.  
- **Throughput**: Tens of thousands of bids/sec per node, depending on complexity.  
- **Notes**: Ensures campaigns, targeting, frequency caps, and creative eligibility are enforced without blocking the main auction path.

---

## 5. Real-Time Budget & Pacing
- **Components**: Redis/Aerospike for atomic spend counters + Apache Flink for pacing adjustments.  
- **Latency**:
  - Budget check: 0.2–1 ms per bid.  
  - Pacing updates (Flink, sub-second windows): <100 ms to adjust multipliers.  
- **Throughput**: Millions of events/sec processed by Flink.  
- **Notes**: Critical for preventing overspend and ensuring smooth delivery throughout the day.

---

## 6. Logging & Observability
- **Components**: Kafka (event stream) → Flink (aggregation) → ClickHouse / Druid (OLAP) + Prometheus/Grafana/OpenTelemetry.  
- **Latency**:
  - Event ingestion into Kafka: <2 ms.  
  - Aggregation via Flink: sub-second.  
  - OLAP queries for dashboards: sub-second (ClickHouse) to seconds (BigQuery).  
- **Throughput**: Millions of logs/events/sec.  
- **Notes**: Real-time analytics allows monitoring spend, CTR, pacing, and alerts.

---

## 7. Delivery of Winning Bids
- **Components**: Creative Validator + CDN (Cloudflare/Fastly) + API Gateway.  
- **Latency**:
  - Creative validation: 1–2 ms (sandboxed).  
  - CDN edge delivery: 10–50 ms depending on geography.  
  - Response to SSP: <100 ms total from auction request to response.  
- **Throughput**: Tens of thousands of concurrent bid responses per node, scales horizontally.  

---

## 8. End-to-End RTB Auction Performance

| Component | Latency | Throughput | Notes |
|-----------|---------|-----------|-------|
| Request Ingestion (Envoy + Kafka) | 1–2 ms | Millions/sec | Load balancing + message enqueue |
| Profile Retrieval (Redis/Aerospike) | 0.2–1 ms | Millions/sec | Sub-ms lookup critical for targeting |
| Bidding Engine + ML scoring | 15–40 ms | 1K–10K auctions/sec/node | Includes ML inference & campaign logic |
| Rule Evaluation (OPA) | 1–2 ms | 10K–50K bids/sec/node | Frequency caps, targeting, compliance |
| Budget Check (Redis/Aerospike) | 0.2–1 ms | 1M+ ops/sec | Atomic spend enforcement |
| Pacing Updates (Flink) | <100 ms | Millions of events/sec | Async adjustment of bid multipliers |
| Logging & Analytics | <1 s ingestion, sub-second OLAP | Millions of events/sec | Real-time monitoring |
| Ad Delivery (CDN) | 10–50 ms | 10K–100K responses/sec/node | Edge delivery to user devices |

---

### Key Observations
1. **Critical path latency** (auction request → winning bid decision) is ~50–100 ms.  
2. **Budget checks, profile fetches, and rule evaluation** contribute <5 ms combined.  
3. **ML scoring** is the largest contributor (5–20 ms, depending on model).  
4. **End-to-end throughput** scales linearly with horizontal nodes for bidding engines, Redis/Aerospike, and Kafka/Flink clusters.  
5. **Sub-ms atomic operations + real-time streaming** ensure correctness (no overspend, consistent pacing) without slowing auctions.  

---
