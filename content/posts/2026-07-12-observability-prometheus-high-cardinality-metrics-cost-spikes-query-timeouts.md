---
title: "Observability: Prometheus High-Cardinality Metrics Causing Cost Spikes and Query Timeouts"
date: 2026-07-12
draft: true
tags: ["observability", "cloud", "monitoring", "prometheus"]
---

## Problem

High‑cardinality metrics in Prometheus—when a metric has many distinct label combinations—cause two main symptoms:

1. **Storage cost spikes** – each unique time series consumes disk space in the TSDB.
2. **Query timeouts** – Prometheus must scan many series to answer a query, which can exceed query‑time limits.

The root cause is usually a lack of aggregation or down‑sampling of metrics that are emitted with high‑cardinality dimensions (e.g., per‑request counters, per‑user counters, per‑tenant metrics).

## Architecture Decisions

### 1. Use Recording Rules for Aggregation

Recording rules pre‑aggregate metrics on the Prometheus server before they are written to the TSDB. This reduces the number of series stored while preserving the aggregated view for dashboards.

*Why*: Summarising high‑cardinality series (e.g., `sum by (job, instance)`) dramatically cuts the series count without losing the ability to view per‑job or per‑instance data.

*Trade‑off*: You lose the finest granularity; dashboards must be built around the aggregated labels.

### 2. Apply Down‑Sampling for Long‑Term Storage

For metrics that are only needed for historical analysis, use Thanos or Cortex down‑sampling to store a lower‑resolution version after a retention period.

*Why*: Reduces long‑term storage costs while still allowing trend analysis.

*Trade‑off*: Coarser data means slower drill‑down; you need to balance retention vs. cost.

### 3. Enforce Cardinality Limits via Alerting

Add alerts that fire when a metric’s cardinality (number of distinct label values) exceeds a threshold, prompting teams to review instrumentation.

*Why*: Early detection prevents runaway series growth before it becomes costly.

*Trade‑off*: Requires monitoring of metric cardinality itself, which adds some operational overhead.

## Implementation

Below is a focused Python snippet that demonstrates how to define a recording rule in a Prometheus rule file.

```python
# Define a recording rule to aggregate high‑cardinality metrics
# This example reduces `http_requests_total` (a counter with method, handler, instance labels)
# to a per‑job, per‑instance rate over 5 minutes.

record_rule_yaml = """
  - record: job:http_requests_rate5m:sum
    expr: sum by (job, instance) (rate(http_requests_total[5m]))
"""

print("Add the above recording rule to your Prometheus rules YAML file (e.g., `rules.yml`).")
print("Then reload Prometheus to activate the rule.")
```

**Explanation**

- `rate(http_requests_total[5m])` computes a per‑second rate over a 5‑minute window, turning the counter into a gauge.
- `sum by (job, instance)` aggregates across all `method` and `handler` label values, leaving only the essential dimensions.
- The resulting series (`job:http_requests_rate5m:sum`) has far fewer series, reducing storage and query load while still providing the needed visibility.

You can adapt the `expr` part to any high‑cardinality metric (e.g., `user_counter`, `tenant_requests_total`, etc.) by adjusting the `by (…)` clause to the dimensions you want to keep.

## Conclusion

By aggregating high‑cardinality metrics at ingestion time with recording rules, you can control storage costs and maintain responsive queries in Prometheus. Combine this with down‑sampling for long‑term archives and cardinality alerts for proactive monitoring, and you’ll have a sustainable observability stack.
