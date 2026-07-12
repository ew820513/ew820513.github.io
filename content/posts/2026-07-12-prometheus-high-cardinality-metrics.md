---
title: "Observability: Prometheus High-Cardinality Metrics Causing Cost Spikes and Query Timeouts"
date: 2026-07-12
draft: true
tags: ["observability", "prometheus", "cost-optimization"]
---

## Problem

Prometheus is a powerful monitoring system, but when metrics have high cardinality (too many unique label combinations), it can cause:
- Exponential growth in storage costs
- Query latency issues and timeouts
- Memory pressure on Prometheus servers
- Difficulty in troubleshooting due to massive time series

Common sources of high cardinality:
- Auto-generated metrics from applications (e.g., request IDs, user IDs)
- Untemplated service discovery
- Dynamic labels without proper aggregation strategies

## Architecture Decisions

### 1. Implement Label Hygiene Practices
**Why:** Reduce unique label combinations before metrics are scraped
- Use constant values for auto-generated labels
- Apply label normalization (e.g., strip special characters)
- Limit cardinality through aggregation at source

### 2. Use Recording Rules for Pre-Aggregation
**Why:** Reduce cardinality by pre-computing common aggregations
- Create recording rules for high-cardinality metrics
- Store aggregated results in lower-cardinality series
- Query aggregated results instead of raw high-cardinality data

### 3. Configure Prometheus for Cost Control
**Why:** Prevent runaway costs from excessive series
- Set `max_concurrency` to limit scrape parallelism
- Use `query_cache_ttl` to cache expensive queries
- Implement retention policies based on business needs
- Use remote_write to offload data to cheaper storage

## Implementation

### Label Hygiene in Application Code (Python Example)

```python
from prometheus_client import Counter, Histogram

# BAD: High cardinality - using request ID as label
request_counter = Counter('http_requests_total', 'Total HTTP requests', ['request_id'])

# GOOD: Use constant label with request count as metric value
request_counter = Counter('http_requests_total', 'Total HTTP requests')

# Even better: Use bucketed counters for rate limiting
request_counter = Counter('http_requests_total', 'Total HTTP requests', ['status_code'])
```

### Recording Rules Configuration (prometheus.yml)

```yaml
groups:
- name: high_cardinality_metrics
  interval: 1m
  rules:
  - record: job:request_count:sum
    expr: sum by (job) (rate(http_requests_total[5m]))
  - record: job:error_rate:avg
    expr: 100 * sum by (job) (rate(http_requests_total{status_code=~"5.."}[5m]))
```

### Query Optimization Example

```promql
# BAD: High cardinality query causing timeouts
sum(http_requests_total{job="web-service"})

# GOOD: Use pre-aggregated recording rule
sum(job:request_count:sum{job="web-service"})
```

## Best Practices Summary

1. **Audit your metrics** - Identify high cardinality metrics using `prometheus_tsdb_head` or Grafana
2. **Implement label hygiene** - Normalize and constrain labels at source
3. **Use recording rules** - Pre-aggregate metrics to reduce cardinality
4. **Monitor your monitoring** - Track series counts and memory usage
5. **Set cost controls** - Configure Prometheus to prevent runaway costs

## Next Steps

- [ ] Audit current metrics for high cardinality
- [ ] Implement label normalization in applications
- [ ] Create recording rules for key high-cardinality metrics
- [ ] Review and adjust Prometheus configuration for cost control

This approach has helped teams reduce Prometheus costs by 60-80% while improving query performance.
