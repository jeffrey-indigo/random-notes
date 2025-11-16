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

---
Here are some of the most relevant resources you can use this week to ramp up quickly for building 1-second observability + traceability for a DSP. I grouped them by topic so you can pick what you need most first:

### ✅ Observability / Telemetry Pipelines
- [“Observability Pipeline: An Easy‐to-Follow Guide for Engineers” (by Last9) — covers how to build & optimise observability pipelines. Last9](https://last9.io/blog/observability-pipeline/?utm_source=chatgpt.com)
- [“Understanding Observability Pipelines – A Practical Guide” (by SigNoz) — walks you through core components, challenges & best practices. SigNoz](https://signoz.io/guides/observability-pipeline/)
- “What you need in a telemetry pipeline” (by Chronosphere) — explains how to control, route, enrich telemetry data at scale. Chronosphere
- “How to Maximise Telemetry Data Value With Observability Pipelines” (by DevOps.com) — shows how to reduce noise, filter, sample telemetry so you focus on the important stuff. DevOps.com

### ✅ Distributed Tracing & High-Throughput Systems
- “Distributed Tracing Logs: How They Work, Benefits & Best Practices” (by Groundcover) — strong on trace-log correlation + instrumentation advice. Groundcover
- “Optimizing Distributed Tracing: Best practices for remaining within budget and capturing critical traces” (by Datadog) — good guidance on sampling, prioritisation. Datadog
- “Distributed Tracing: Concepts, Pros/Cons & Best Practices” (by Coralogix) — conceptual foundation you can use to anchor your tracing strategy. Coralogix
- “Investigating Performance Overhead of Distributed Tracing” (academic paper) — useful for understanding cost/overhead trade-offs in high-traffic environments. @Large Research

### ✅ AdTech / DSP / Real-Time Bidding Context
- “Implementing High-Performance Ad Tech Demand-Side Platforms (DSPs)” (by The New Stack) — architecture & per-millisecond latency case-study relevant for your domain. The New Stack
- “Build a reference architecture for a demand-side platform” (by Redpanda) — DSP-specific architecture reference you can tie into your observability project. Redpanda
- “Guidance for capturing Advertising OpenRTB real-time bidding events for analytics on AWS” (by Amazon Web Services) — shows how to capture high throughput real-time bidding events, which overlaps with observability. Amazon Web Services, Inc.
- “AdTech data pipelines: Best practices for architecting efficient AdTech platforms” (by Xenoss) — more on the data pipeline side of the ad-tech world. Xenoss

### 📋 How to Use These in < 1 Week
- Day 1 – Read the observability pipeline guides (Last9 + SigNoz) to get a mental model of telemetry systems.
- Day 2 – Dive tracing best practices (Groundcover + Datadog) so you’re comfortable with trace/log/metric correlation and sampling.
- Day 3 – Review DSP/RTB architecture stuff (New Stack + Redpanda + AWS) so you understand the domain constraints you’ll be working within.
- Day 4 – Focus on data pipeline observability (Chronosphere + DevOps.com + Xenoss) because you'll be building streaming/ingestion pipelines.
- Day 5 – Review the academic/performance overhead papers to prepare for trade-offs with instrumentation in high throughput systems.
- Day 6–7 – Create a mini “observability strategy” document for your first sprint: pick 2–3 metrics/traces to instrument first, decide sampling strategy, decide streaming backbone approach, sketch local flush & ingestion path. Use the resources to justify technical choices.

Optional – Bookmark these for reference when you’re coding: batch size/settings, sampling rates, cardinality limits, backpressure strategies.


