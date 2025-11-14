# random-notes

## A Typical 1-Second Observability Architecture
```
             ┌──────────────────────────────┐
             │     DSP Microservices        │
             │  (Bidder, ML, Audience...)   │
             └──────────────────────────────┘
                        │
                        ▼
     ┌─────────────────────────────────────────────────┐
     │   Local Observability Agent (sidecar or lib)    │
     │  - In-memory metric store                       │
     │  - Light tracing (sampled)                      │
     │  - Log batching                                 │
     │  - 1s async flush                               │
     └─────────────────────────────────────────────────┘
                        │
                        ▼
           ┌────────────────────────┐
           │   Streaming Ingestion  │
           │   Kinesis / MSK Kafka  │
           └────────────────────────┘
                        │
                        ▼
   ┌───────────────────────────────────────┐
   │  Real-Time Processors (Flink / KDA)   │
   │  - rollups                            │
   │  - anomaly detection                  │
   │  - per-service latency/err tracking   │
   └───────────────────────────────────────┘
                        │
                        ▼
      ┌────────────────────────────────┐
      │  Stores & Dashboards           │
      │  - CloudWatch Metrics (1s)     │
      │  - Timestream                  │
      │  - OpenSearch                  │
      │  - Grafana                     │
      └────────────────────────────────┘
```


## System design (textual diagram + components)
```
                        ┌────────────────────────────┐
                        │   External Exchanges /    │
                        │   Publishers (SSP)        │
                        └────────────┬──────────────┘
                                     │
                            Bid request / response (RTB)
                                     │
                        ┌────────────▼─────────────┐
                        │ Ingress / Edge Layer     │
                        │ - TLS termination        │
                        │ - Auth / Rate-limit      │
                        └────────────┬─────────────┘
                                     │
                        ┌────────────▼─────────────┐
                        │ DSP Microservice Cluster │
                        │ (bidder, model, pacing,  │
                        │  audience lookup, fraud) │
                        └────────────┬─────────────┘
                                     │
            ┌────────────────────────┼────────────────────────┐
            │                        │                        │
            ▼                        ▼                        ▼
  ┌────────────────┐        ┌────────────────┐        ┌─────────────────┐
  │ Local Obs SDK  │        │ Trace Headers  │        │ Sidecar Agent   │
  │ (in-process)   │        │ (W3C trace)    │        │ or "agent" for  │
  │ - in-memory    │        │ propagated     │        │ batching & APM  │
  │   counters     │        │ across calls   │        │ - ring buffer   │
  │ - histograms   │        │ - request-id   │        │ - async flush   │
  └──────┬─────────┘        └──────┬─────────┘        └─────┬──────────┘
         │                         │                        │
         └─────────────┬───────────┴──────────────┬─────────┘
                       ▼                          ▼
               Local flush every 1s         Trace & log batching
               (delta metrics)              (sampled + buffered)
                       │                          │
                       ▼                          ▼
              ┌────────────────────────────────────────────┐
              │   Streaming Ingestion (durable, ordered)   │
              │   - Amazon Kinesis (or MSK Kafka)          │
              │   - Enhanced fan-out / multiple shards     │
              └─────────────────┬──────────────────────────┘
                                │
                   ┌────────────▼────────────┐
                   │  Real-time Processing   │
                   │  (Flink / KDA / Flink) │
                   │  - per-1s rollups       │
                   │  - cardinality control  │
                   │  - anomaly detection    │
                   └──────┬─────────┬────────┘
                          │         │
        ┌─────────────────┘         └─────────────────┐
        ▼                                             ▼
┌────────────────────┐                       ┌─────────────────────┐
│ Short-term stores  │                       │ Long-term stores    │
│ - DynamoDB / Redis │                       │ - S3 (parquet)      │
│ - Timestream       │                       │ - Data lake         │
└────────┬───────────┘                       └────────┬────────────┘
         │                                            │
         ▼                                            ▼
 ┌────────────────┐                          ┌────────────────────┐
 │ Dashboards &   │                          │ Backfill & ML      │
 │ Alerting       │                          │ (batch jobs)       │
 │ - Grafana      │                          │ - model training   │
 │ - CloudWatch   │                          │ - offline metrics  │
 └────────────────┘                          └────────────────────┘
```

### Key component notes / design choices
- **Local Obs SDK / Sidecar:** keep hot path instrumentation in-process and non-blocking. Update counters/histograms in memory and flush deltas every 1s to streaming. Use a tiny, optimized API (increment, observe, tag-limited).
- **Traces:** use W3C Trace Context propagation + very low-overhead OpenTelemetry (or a custom ultralight lib). Sample traces aggressively (e.g., adaptive sampling: high error-rate gets higher sample). Always emit a 1-line span event for every win/loss with request-id so you can link events even when the full trace wasn’t sampled.
- **Streaming backbone:** Amazon Kinesis or Kafka (MSK) — durable, ordered, partitions/shards. Use enhanced fan-out or consumers per pipeline. Keep partition key choices consistent (e.g., region/service/request-id mod shards).
- **Realtime processing:** Flink / Kinesis Data Analytics for 1s sliding/tumbling windows, anomaly detection, and cardinality-aware aggregations. Emit to short-term store for dashboards with TTL.
- **Short-term store:** low-latency DB (DynamoDB, Timestream, Redis) for 1s-visibility metrics; long retention and analytics live in S3 + data lake.
- **Backpressure & graceful degradation:** SDKs must drop or sample telemetry if local queues/backups exceed threshold to avoid affecting bidding.
- **High-cardinality control:** limit labels/tags, use hashing buckets, or use approximate structures (HLL, t-digest) for heavy cardinalities.
