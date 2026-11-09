---
title: "OpenTelemetry Collector Deployment Patterns: DaemonSet vs Sidecar vs Gateway"
date: 2026-11-09
draft: true
tags: ["opentelemetry", "kubernetes", "observability", "collector", "daemonset", "sidecar", "gateway"]
---

## The Problem

You have 400+ Kubernetes pods to instrument, and the OpenTelemetry Collector is your chosen agent. The question isn't *whether* to run the collector — it's *where*.

You can run it as:

1. **DaemonSet** — one collector per node, sees every pod on that node. Minimal network hop, but one collector shares all the node's traffic.
2. **Sidecar** — one collector per application pod. Isolated telemetry per pod, but you now operate 400+ collector containers.
3. **Gateway** — a central Deployment (typically 3 replicas) behind a service. One config to manage, but every trace makes two extra network hops.

Each pattern shifts the same three variables: cost, performance, and resilience. There is no universally right answer — there is only the right answer for your telemetry mix, node sizes, and team's config-management maturity. This post walks through the trade-offs with concrete resource formulas, config strategies, failure modes, and a decision framework you can apply Monday morning.

## The Three Patterns, Side by Side

| | DaemonSet | Sidecar | Gateway |
|---|---|---|---|
| Collector instances | One per node (≈ N) | One per pod (≈ P) | Fixed replicas (≈ R, e.g. 3) |
| Network hops per trace | 1 (pod → node collector) | 1 (pod → sidecar, localhost) | 2 (pod → mesh sidecar → collector svc) |
| Config surface | One ConfigMap, templated per node pool | One ConfigMap per pod image/release | One ConfigMap, cluster-wide |
| Best telemetry fit | Logs, node-level metrics | Per-request traces, app-specific metrics | Metrics, aggregate traces |
| Failure blast radius | Lose all telemetry on a node | Lose one pod's telemetry | Lose a slice of aggregate traffic until a replica drains |

P = number of pods (400+), N = number of nodes, R = gateway replicas.

## Resource Cost: The Formulas

The most useful way to think about collector cost is to split it into two parts:

- **Base overhead**: the collector binary itself, idle (~100–250m CPU, ~128–256 MiB memory per instance).
- **Per-span cost**: receiving, batching, and exporting telemetry (~20–40m CPU and ~40–80 MiB per pod's worth of spans, depending on volume).

The per-span work has to happen regardless of pattern — you still process every span. What changes is how many times you pay the *base overhead*.

Let:
- `P` = total pods (e.g., 400)
- `N` = total nodes
- `R` = gateway replicas (e.g., 3)
- `pb` = pods per node on average (`P / N`)
- `c_base` = collector base footprint (~150m CPU / 256 MiB)
- `c_pod` = per-pod span-processing footprint (~25m CPU / 64 MiB)

### DaemonSet

One collector per node, sized for everything on that node:

```
collector count      = N
resources per node   = c_base + pb × c_pod
total CPU            = N × c_base + P × c_pod
total memory         = N × c_base_mem + P × c_pod_mem
```

### Sidecar

One collector per pod, processing only that pod's spans:

```
collector count      = P
resources per pod    = c_base + c_pod
total CPU            = P × c_base + P × c_pod
total memory         = P × c_base_mem + P × c_pod_mem
```

### Gateway

R replicas sharing the aggregate load:

```
collector count      = R
resources per replica= c_base + (P / R) × c_pod
total CPU            = R × c_base + P × c_pod
total memory         = R × c_base_mem + P × c_pod_mem
```

### Worked example: 400 pods, 20 nodes, 3 gateway replicas

| Pattern | Collector count | Approx. CPU | Approx. memory | Notes |
|---|---|---|---|---|
| DaemonSet | 20 | 20×150m + 400×25m ≈ **13 CPU** | 20×256Mi + 400×64Mi ≈ **30 GiB** | Base overhead paid 20× |
| Sidecar | 400 | 400×150m + 400×25m ≈ **70 CPU** | 400×256Mi + 400×64Mi ≈ **128 GiB** | Base overhead paid 400× — ~5× DaemonSet CPU |
| Gateway | 3 | 3×150m + 400×25m ≈ **10.4 CPU** | 3×256Mi + 400×64Mi ≈ **26 GiB** | Base overhead paid 3×, but all spans funnel through 3 pods |

The sidecar pattern roughly quintuples collector CPU in this example because you pay the collector base footprint once per pod. The daemon set and gateway both pay it far fewer times, but the gateway concentrates all of that traffic on just three pods — which brings us to the performance and resilience trade-offs.

## Config Management: One Config vs 400

### DaemonSet: template per node pool

A DaemonSet uses a single ConfigMap, but you often need per-node-pool variations — for example, different resource limits on GPU nodes versus general-purpose nodes, or different exporters for nodes in different availability zones.

Strategy: manage node-pool variants with Helm values or Kustomize patches, and bind each DaemonSet to its nodes with `nodeSelector` or pod affinity. One ConfigMap per node-pool variant, not one per node. Do not bake node-specific exporter endpoints into the collector config; pass the backend endpoint as an environment variable (`OTEL_EXPORTER_OTLP_ENDPOINT`) so the same ConfigMap works everywhere.

```yaml
# values.yaml snippet
daemonsets:
  general:
    nodeSelector: { node-type: general }
    resources:
      limits: { cpu: "500m", memory: "512Mi" }
  gpu:
    nodeSelector: { node-type: gpu }
    resources:
      limits: { cpu: "1000m", memory: "1Gi" }
```

A config change rolls out per DaemonSet — fast, bounded, and easy to test on one node pool first.

### Sidecar: 400 configs, 400 rollouts

The sidecar pattern's real cost isn't CPU — it's operational surface area. Each pod carries its own collector container with its own config. A config change means rolling out every application pod that embeds it. With 400+ pods, that's a slow, noisy rollout and a large blast radius if the new config is wrong.

Strategy: keep the sidecar collector config as thin as possible — receiver, one exporter pointing at an endpoint, and basic batching. Put anything that varies (backend endpoint, auth token, sampling rate) into environment variables injected via the downward API or Helm, not into the config file. Better yet, don't run a full collector in the sidecar; use the OTel SDK or a minimal agent that forwards to a DaemonSet or Gateway, so config changes happen centrally.

If you must run sidecar collectors, gate config changes behind canary deployments: roll to 5% of pods, verify span flow, then expand.

### Gateway: one ConfigMap, one deployment

The gateway pattern wins on config management. One ConfigMap, one Deployment, one rollout. Everyone sends telemetry to `opentelemetry-collector.monitoring.svc.cluster.local` and the central collector handles exporters, sampling, and routing.

The catch: the application pods need to know the collector endpoint. Inject it dynamically via a service mesh sidecar (e.g., Istio/Linkerd egress), a headless service, or a downward-API environment variable so you don't hardcode IPs. Keep the gateway config focused on aggregation and export — do not put per-application processing rules in the central pipeline unless you really need them; that couples every app to the central collector's release cycle.

## Telemetry Type: Match Pattern to Signal

| Signal | Best pattern | Why |
|---|---|---|
| **Metrics** (Prometheus scrape, host metrics) | DaemonSet or Gateway | Scrape targets are stable; either pattern works. DaemonSet can scrape node-local cadvisor/kubelet directly without crossing the network. |
| **Traces** (per-request spans) | Sidecar for lowest latency; DaemonSet acceptable | Sidecar sends spans over localhost — one hop, no mesh contention. DaemonSet is fine for node-level traces but all spans on a node compete for one collector. Gateway adds two hops and is the slowest option for trace latency. |
| **Logs** (stdout/file tailing) | DaemonSet | The filelog receiver with hostPath log mounts is the mature pattern. One collector per node tails all container logs on that node and forwards them. Sidecar log tailing requires shared log volumes or sidecar-specific scrapers, which is more complex. |

If your workload is trace-heavy and latency-sensitive, lean toward sidecar (or a DaemonSet+Gateway hybrid — see below). If it's log-heavy, DaemonSet is the natural fit. If it's metrics-heavy with occasional traces, Gateway keeps things simple.

## Failure Modes

| Pattern | What breaks | Blast radius | Mitigation |
|---|---|---|---|
| DaemonSet | Node dies | All pod telemetry on that node is lost until the collector pod reschedules elsewhere | Logs/spans buffered at the application or agent side; don't rely on the collector as the only buffer |
| Sidecar | Pod dies | Only that pod's telemetry is lost | Minimal blast radius — this is sidecar's resilience advantage |
| Gateway | Collector replica dies | A fraction of aggregate traffic is affected; spans may buffer in the collector's memory queue or be retried by the sender | PodDisruptionBudgets, readiness/liveness probes, graceful draining, sender-side retries with exponential backoff |

The gateway concentrates risk: three replicas carry *all* cluster telemetry. If one is unhealthy and your mesh doesn't route around it, you drop or delay spans. Set `terminationGracePeriodSeconds` and pre-stop hooks so replicas drain their queues before shutting down, and size replicas so that losing one still leaves headroom.

## The Recommendation Framework

**Choose DaemonSet if:**

- You collect lots of logs (container stdout/file tailing) — the hostPath + filelog receiver pattern is proven.
- You scrape node-level metrics (kubelet, cadvisor, node exporter) and want to stay on-node.
- You have large, uniform nodes and want to minimize collector base overhead (N instances, not P).
- Your team is comfortable with Helm/Kustomize per node pool and wants bounded, node-pool-scoped rollouts.

**Choose Sidecar if:**

- You need per-request trace latency and want localhost between your app and its collector.
- You need hard telemetry isolation (e.g., one team's misbehaving exporter shouldn't affect another's spans).
- You run few pods per node (low P/N ratio), so the base-overhead multiplication is small.
- You can gate config changes behind canary rollouts and tolerate slower config propagation.

**Choose Gateway if:**

- You want one ConfigMap and one rollout for the entire cluster.
- Your telemetry is metrics-heavy and traces can tolerate an extra network hop.
- You already run a service mesh and can inject the collector endpoint dynamically.
- You have operational bandwidth to run a highly available central pipeline (PDBs, draining, replica sizing for failure of any single node).

## A Practical Hybrid: DaemonSet + Gateway

Most mature clusters don't pick just one. A common production pattern is a **DaemonSet collector per node** (tails logs, scrapes node metrics, receives spans over the local network) that forwards to a **Gateway Deployment** (handles sampling, exporter routing, auth, and central aggregation). Applications send to the node DaemonSet; the DaemonSet sends to the gateway. You get local collection with centralized control, and a single gateway config change doesn't touch 400 pods.

If you adopt this, size the DaemonSet for local intake only (light processing, quick forward) and size the gateway for aggregate export load.

## The Master Checklist

If you're deciding this week:

1. **Count your signals.** Logs → DaemonSet. Ultra-low-latency traces → Sidecar. Metrics + simple traces → Gateway.
2. **Run the formula.** Estimate `N × c_base + P × c_pod` (DaemonSet), `P × c_base + P × c_pod` (Sidecar), `R × c_base + P × c_pod` (Gateway). If sidecar CPU looks unaffordable, rule it out.
3. **Pick config ownership.** One team managing all telemetry → Gateway. Per-team autonomy with bounded rollouts → DaemonSet per node pool.
4. **Plan for failure.** Add sender-side retries, receiver timeouts, and (for gateway) PDBs plus graceful draining.
5. **Start small.** Deploy one pattern to a single namespace or node pool, measure span throughput and collector CPU, then expand. Don't migrate 400 pods in one rollout.

## Summary

The OTel Collector deployment pattern is a cost-vs-performance-vs-resilience triangle, and you get to pick which corner you sharpen:

- **DaemonSet** minimizes collector base overhead and fits logs/node metrics, but one collector can become a bottleneck on busy nodes.
- **Sidecar** gives per-pod isolation and the lowest trace latency, but multiplies base overhead and config surface area by pod count — expensive at 400+ pods.
- **Gateway** centralizes config and cost into a few replicas, but adds network hops and concentrates failure risk on a small set of collector pods.

There's no single right answer — there's the right answer for your telemetry mix, node sizes, and how much config-management pain your team wants to sign up for. Run the formulas, match the pattern to your signals, and start with one node pool before you roll out to all 400 pods.

