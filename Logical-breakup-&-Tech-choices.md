# 🏗 RTB Logical Architecture + Tech Choices

## 1. Request Ingestion
- **Load Balancer**: **Envoy** → battle-tested, supports gRPC/HTTP2, very high throughput, observability built-in.  
- **Message Bus**: **Kafka** (vs Pulsar) → Kafka has stronger ecosystem, lower latency, better tooling for stream processors, perfect for auction logging.  
- **Why**: Kafka + Envoy scales horizontally to millions of RPS, provides durability, and integrates well with observability.

---

## 2. User Profile Retrieval
- **Low-latency Store**:  
  - **Redis Cluster / Aerospike** → ideal for sub-ms reads/writes, great for hot user/device context.  
  - **DynamoDB (with DAX cache)** → good for global scale, but slightly higher latency.  
- **Pick**: **Redis or Aerospike** for *real-time bidding* because latency is critical (<1ms).

---

## 3. Business Logic & Bidding Engine
- **Auction / Bidding Service**: **Go** (ultra low-latency, high concurrency) or **Node.js** (faster iteration, async IO). In production DSPs/exchanges, Go/Java dominate for throughput.  
- **ML Model Serving**: **TensorFlow Serving** or **PyTorch Serve** with gRPC → allows online CTR/CVR predictions.  
- **Campaign Config Store**: **Postgres** (for relational integrity + indexing) backed by **Redis** for caching.  
- **Why**: Go + Redis + TF Serving yields predictable p99 latency under heavy load.

---

## 4. Rule Evaluation Engine
- **Options**:  
  - **Open Policy Agent (OPA)** → extensible, declarative, integrates well with services.  
  - **Drools (Java)** → mature rules engine, but heavier footprint.  
  - **Custom Rules in Go/Node.js** → fastest but less maintainable.  
- **Pick**: **OPA** for extensibility + auditability. If ultra-low-latency required, complement with cached precompiled rules in Redis.

---

## 5. Real-Time Budget & Pacing
- **Budget Store**: **Aerospike / Redis** with atomic counters (sub-ms, high throughput).  
- **Streaming Adjustments**: **Apache Flink** → handles real-time budget rollups and pacing adjustments better than Spark due to latency. Pacing algorithms can be configured in Flink like - Performance(ROI based) or Dynamic pacing(based on availability)
- **Why**: Sub-ms consistency is required; Redis excels at high-throughput counters while Flink continuously aggregates spend. Basically Redis for micro smoothening, Flink for macro smoothening.

---

## 6. Logging & Observability
- **Event Transport**: **Kafka** → de-facto standard for append-only auction logs.  
- **OLAP for Reporting**:  
  - **BigQuery** → great for batch analytics, but not real-time.  
  - **Druid** → excellent for real-time OLAP queries (seconds latency).  
  - **ClickHouse** → best for ultra-fast analytical queries at lower cost, highly extensible with SQL.  
- **Pick**: **ClickHouse** → balances performance (sub-second query latency), cost, and scalability for auction + campaign reporting.  
- **Observability**:  
  - **Prometheus + Grafana** → metrics, dashboards.  
  - **OpenTelemetry** → tracing and distributed logging. 
  - **ELK / Loki** → for logs.

---

## 7. Delivery of Winning Bids
- **Creative Validation**: Run in **Go** with strict sandboxing.  
- **CDN**: **Cloudflare / Fastly** for edge delivery of creatives (low-latency, geo-distributed).  
- **API Gateway Response**: Envoy/Nginx → ensures winner returned to SSP <100ms.  

---

# 🔄 End-to-End Flow + Tech Highlights
1. **SSP → Envoy → Auction Service** (Go).  
2. **Profile Lookups** → Redis/Aerospike (<1ms).  
3. **ML Models** → TF Serving / PyTorch Serve (gRPC).  
4. **Rules Engine** → OPA for flexibility.  
5. **Budget Check** → Redis/Aerospike atomic counters.  
6. **Auction Logging** → Kafka → Flink → ClickHouse.  
7. **Creative Delivery** → CDN (Cloudflare/Fastly).  
8. **Observability** → Prometheus + Grafana + OpenTelemetry + Loki.  
