# RTB — Component-level UML Layout

## Component Diagram
```mermaid
graph TD


subgraph PublisherSide[Publisher Side]
Browser[Browser / App]
PubAd[Publisher Ad Server]
SSP[SSP / Edge]
end


subgraph AdExchange[Ad Exchange]
API[API Gateway / Edge]
Auction[Auction Coordinator]
Adapter[Bid Adapter Pool]
Timeout[Timeout Manager]
Floor[Price Floor Service]
Fraud[Fraud / Policy Filter]
CreativeVal[Creative Validator]
Logger[Event Logger]
Metrics[Metrics / Observability]
end


subgraph DSPSide[DSP Side]
DSP[DSP Bidding Engine]
DSPStore[DSP Data Store]
ML[ML Model Service]
end


subgraph Infra[Shared Infra]
Profile[Real-Time Profile Store]
Feature[Feature Store]
Kafka[Event Stream / Kafka]
Stream[Stream Processor Flink/Beam]
OLAP[OLAP Data Warehouse]
CDN[Creative CDN]
end


Browser --> PubAd --> SSP --> API
API --> Auction
Auction --> Adapter
Auction --> Timeout
Auction --> Floor
Auction --> Fraud
Auction --> Profile
Adapter --> DSP
DSP --> DSPStore
DSP --> ML
Auction --> CreativeVal --> CDN
Auction --> Logger --> Kafka --> Stream
Stream --> Feature
Stream --> OLAP
Auction --> Metrics
```
**Notes:** Keep Edge and Adapter Pool horizontally scalable. Use sticky hashing by publisher/adslot for cache locality. Profile and Floor are fast key-value read paths — colocate near Edge/Auction for latency.

---

## Class Diagram (Exchange internals)

```mermaid
classDiagram
class AuctionCoordinator {
+runAuction(request)
+applyFilters()
+selectWinner()
}

class BidAdapter {
+sendBidRequest()
+parseBidResponse()
}

class TimeoutManager {
+startTimer()
+cancelTimer()
}

class PriceFloorService {
+getFloorPrice(slotId)
}

class FraudFilter {
+checkRequest(request)
}

class CreativeValidator {
+validateCreative()
}

class EventLogger {
+logEvent(event)
}

AuctionCoordinator --> BidAdapter
AuctionCoordinator --> TimeoutManager
AuctionCoordinator --> PriceFloorService
AuctionCoordinator --> FraudFilter
AuctionCoordinator --> CreativeValidator
AuctionCoordinator --> EventLogger
```
**Design hints:** AuctionCoordinator should be stateless for autoscaling and persist all inputs/outputs to EventLogger. BidAdapter should be asynchronous (non-blocking), with backpressure handling and per-adapter rate limits.

---

## Sequence Diagram (detailed auction flow)

```mermaid
sequenceDiagram
participant U as User Browser/App
participant P as Publisher Ad Server
participant S as SSP / Edge
participant G as API Gateway
participant A as Auction Coordinator
participant B as Bid Adapter Pool
participant D as DSP Bidding Engine
participant M as ML Model Service
participant C as Creative Validator
participant X as CDN


U->>P: Request page/ad slot
P->>S: Send ad request
S->>G: Forward normalized request
G->>A: Forward to auction service
A->>B: Broadcast bid requests
B->>D: Send bid request (with user/profile)
D->>M: Query CTR/conversion model
M-->>D: Return prediction score
D-->>B: Return bid response
B-->>A: Return all bid responses
A->>A: Apply timeout, floor, fraud checks
A->>C: Validate winning creative
C-->>A: Creative approved
A->>P: Send winning ad details
P->>U: Render creative via CDN
U->>X: Fetch creative assets
```

Operational note: ensure the TimeoutManager cancels RPCs to slow DSPs as tmax approaches, and prefer non-blocking IO.

---

## Deployment Diagram

```mermaid
graph LR


subgraph UserDevices[User Devices]
BrowserApp[Browser/App]
end


subgraph PublisherCluster[Publisher Infrastructure]
PubAdServer[Publisher Ad Server]
SSPCluster[SSP / Edge Cluster]
end


subgraph ExchangeCluster[Ad Exchange Cluster]
APIGateway[(API Gateway Pods)]
AuctionSvc[(Auction Coordinator Pods)]
AdapterPool[(Bid Adapter Pods)]
TimeoutPods[(Timeout Manager Pods)]
FloorPods[(Price Floor Service Pods)]
FraudPods[(Fraud / Policy Filter Pods)]
CreativePods[(Creative Validator Pods)]
LoggerPods[(Event Logger Pods)]
MetricsPods[(Metrics / Observability Pods)]
end


subgraph DSPCluster[DSP Infrastructure]
DSPPods[(DSP Bidding Engine Pods)]
DSPStore[(DSP Data Store)]
MLPods[(ML Model Service)]
end


subgraph SharedInfra[Shared Infrastructure]
ProfileStore[(Real-Time Profile Store / Redis Cluster)]
FeatureStore[(Feature Store)]
KafkaCluster[(Kafka Brokers)]
Flink[(Stream Processor Cluster)]
OLAP[(OLAP Data Warehouse)]
CDN[(Creative CDN)]
end


BrowserApp --> PubAdServer --> SSPCluster --> APIGateway
APIGateway --> AuctionSvc
AuctionSvc --> AdapterPool --> DSPPods
DSPPods --> DSPStore
DSPPods --> MLPods
AuctionSvc --> TimeoutPods
AuctionSvc --> FloorPods
AuctionSvc --> FraudPods
AuctionSvc --> ProfileStore
AuctionSvc --> CreativePods --> CDN
AuctionSvc --> LoggerPods --> KafkaCluster --> Flink --> FeatureStore
Flink --> OLAP
AuctionSvc --> MetricsPods
```

Scaling guidance: Separate adapter-service from auction-service to scale differently: adapters scale with outbound fan-out and socket counts; auction-service scales for CPU & decision latency. Use HPA on CPU + custom metrics (socket usage, p99 latency, request queue length).

---

## Interfaces & API sketches

* **Edge API** (incoming): `POST /bid` body=OpenRTB JSON, headers: `x-publisher-id`, `x-request-id`. Response: `200 OK` with `served` HTML or `204 No-Content` if no winner.
* **Adapter interface** (internal gRPC):

  * `Bid(req: BidRequest) returns (stream BidResponse)` — supports streaming responses for VAST-like multi-creative.
  * Backoff & Retry semantics are handled by adapter layer.
* **Event Logger**: asynchronous append-only API to Kafka; require ack semantics `at-least-once` and idempotency keys for deduping.

---

# Detailed Everview of major classes

---

## 1. `AuctionCoordinator`

### Functionality
- Orchestrates the real-time auction flow:
  - Receives bid requests from SSP/Exchange.
  - Triggers price floor checks, fraud/policy filters.
  - Invokes BidAdapters and TimeoutManager.
  - Aggregates bids and selects the winning bid.
  - Calls CreativeValidator and EventLogger.

### Tech Stack & Rationale
- **Go microservice** for low-latency and high concurrency.
- **fasthttp** library: high-performance HTTP handling (better than standard `net/http` for RTB rates).
- Lightweight, compiled, and optimized for millisecond-level response times.

### Throughput / Latency Targets
- Latency: 10–20 ms per auction orchestration (excluding ML scoring or network delays).
- Throughput: 1K–10K auctions/sec per node, scales linearly with replicas.

### Scaling
- Horizontal scaling: multiple Go service instances behind a load balancer.
- Stateless design: state (e.g., temporary auction info) can be stored in in-memory cache or passed in requests.
- Supports sharding by request hash or publisher ID for affinity.

### Fault Tolerance
- Graceful shutdown and request retry for failed bid requests.
- Circuit breakers on BidAdapters.
- Leader election or consensus (if some coordination is needed across multiple AuctionCoordinator instances).

---

## 2. `BidAdapter`

### Functionality
- Connects to DSP endpoints to send bid requests and receive bids.
- Encapsulates DSP-specific logic and protocols.
- Handles concurrency and bid response aggregation.

### Tech Stack & Rationale
- **Go service** leveraging goroutines for concurrent requests.
- Uses async I/O to avoid blocking on slow DSPs.
- Can run multiple replicas for redundancy.

### Throughput / Latency Targets
- Latency: 5–10 ms per DSP request (network-dependent).
- Throughput: 10K+ bid requests/sec per adapter instance (depending on concurrency and network).

### Scaling
- Horizontal replicas: multiple adapter instances per DSP endpoint.
- Sharding by DSP or request hash to distribute load evenly.
- Goroutines handle thousands of concurrent DSP requests per instance.

### Fault Tolerance
- Timeouts and retries per DSP request.
- Circuit breakers to prevent cascading failures from slow/unhealthy DSPs.
- Idempotent handling of duplicate responses.

---

## 3. `TimeoutManager`

### Functionality
- Enforces strict deadlines for bid responses.
- Cancels late bid requests and ensures auction finishes on time.
- Tracks per-bidder response times.

### Tech Stack & Rationale
- Internal **Go library** inside AuctionCoordinator.
- Uses **high-resolution timers**.
- Optional use of **eBPF/tuned sockets** for precise network event timing.

### Throughput / Latency Targets
- Overhead: sub-millisecond for timer checks.
- Can handle thousands of concurrent timers per AuctionCoordinator instance.

### Scaling
- Runs as part of AuctionCoordinator; scales with replicas.
- Timer management is lightweight and in-memory, so no additional sharding needed.

### Fault Tolerance
- Timers reset on service restart (state is ephemeral).
- AuctionCoordinator handles missing bids gracefully if timeout triggers.

---

## 4. `EventLogger`

### Functionality
- Logs all auction events, impressions, clicks, and errors.
- Feeds downstream analytics, billing, and ML pipelines.

### Tech Stack & Rationale
- **Kafka producer client** in Go.
- Use **idempotent producers** to prevent duplicate events during retries.
- Guarantees at-least-once delivery to Kafka topic.

### Throughput / Latency Targets
- Latency: <2 ms per log event.
- Throughput: millions of events/sec per Kafka cluster.

### Scaling
- Horizontal: multiple producer instances.
- Partitioning in Kafka by `publisherID`, `campaignID`, or `auctionID` for load distribution.
- Producers can buffer asynchronously to smooth spikes.

### Fault Tolerance
- Retries on Kafka broker failures.
- Local buffer to prevent data loss during short outages.
- Idempotent writes ensure no duplication.

---

## 5. `Profile Store`

### Functionality
- Stores real-time user/device profiles and features.
- Provides **sub-ms lookups** for auction decisions.
- Supports TTL-based expiration and compaction for memory efficiency.

### Tech Stack & Rationale
- **Redis Cluster** (in-memory) for ultra-low latency.
- TTLs remove stale data automatically.
- Compaction merges redundant features and reduces memory footprint.

### Throughput / Latency Targets
- Latency: 0.2–1 ms per key lookup.
- Throughput: millions of lookups/sec with clustering.

### Scaling
- Horizontal sharding by `userID` or `deviceID`.
- Replication for read availability.
- Automatic resharding to balance memory across nodes.

### Fault Tolerance
- Master-replica failover for high availability.
- Persistence (AOF or RDB) optional for recovery.
- TTLs and compaction prevent stale data from affecting decisions.
