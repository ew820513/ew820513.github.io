---
title: "Postgres Connection-Pool Exhaustion Under Burst Load: Sizing, PgBouncer, and a 3-Command Runbook"
date: 2026-07-12
draft: true
tags: ["database", "postgres", "pgbouncer", "scaling", "incident-response"]
---

## Problem

You scaled the app out to 20 replicas for a launch. CPU on the database box is sitting at 25%. Yet new requests start hanging, and eventually you see:

```
FATAL: remaining connection slots are reserved for non-replication superuser connections
```

or client-side timeouts like `HikariPool-1 - Connection is not available, request timed out after 30000ms`.

Nothing is CPU-bound. The database is alive. The problem is that Postgres `max_connections` (default 100 on many managed offerings) is exhausted because **every app instance opens its own pool of connections**, and they don't share.

The brutal arithmetic:

```
20 app pods  x  10 pooled connections each  =  200 open connections
Postgres max_connections (default)          =  100
=> 100 connections rejected. Service degrades for everyone.
```

This is the single most common cloud-DB incident a mid-level engineer hits the first time they scale out. It feels like a database problem; it's really a **topology and capacity-planning** problem. This post gives you the mental model, a sizing formula, a minimal PgBouncer config, and a 3-command runbook you can use tomorrow.

## Why app-side pools alone don't fix this

A connection pool inside your app (HikariCP for Java, pgxpool for Go, SQLAlchemy's pool for Python) does one job well: it avoids the cost of opening a new TCP + TLS + auth handshake on every request. That's a *per-process* optimization.

What it does NOT do is coordinate across processes. Each pod has its own pool and its own idea of how many connections it "needs." There is no global scheduler that says "the cluster is only allowed 100 total." So the aggregate is simply:

```
total_connections ≈ instances x pool_size
```

And as you scale out (more pods) or up (bigger pool per pod), the product blows past `max_connections` regardless of how well each individual pool is tuned. A pool that is perfectly sized for one instance becomes catastrophic at 20 instances.

### The three-way trade-off

Pool sizing is always a balance among three competing failures:

| If the pool is... | You risk... |
|---|---|
| **Too small** | Requests queue and time out waiting for a free connection; throughput capped below what the DB can handle. |
| **Too large** | DB memory pressure (each backend ~5–15 MB of RAM), connection storms on restart, lock/wait contention; the DB dies *before* CPU is the bottleneck. |
| **Wrong place** | Per-instance pools that don't share state across replicas — the exact exhaustion bug above. |

The fix is not "pick a bigger number." It's **collocate the pool so it's shared**, and **size it from DB capacity**, not gut feel.

## Architecture Decision 1: Put a consolidation layer in front of Postgres

Instead of every app pod talking directly to Postgres, route them all through a single consol

## Architecture Decision 2: Shared connection pools via PgBouncer/RDS Proxy

### Why transaction pooling (not session pooling) for most apps

PgBouncer can operate in two modes:

- **Session pooling**: One dedicated backend connection per app session (keeps session state closéd)
- **Transaction pooling**: Frontend connection is returned to the pool after each SQL statement (stateless)

Most stateless web services (HTTP APIs) work fine with transaction pooling because they don't rely on server-side session state between requests. It dramatically reduces the number of backend connections needed.

### Minimal PgBouncer config (`/etc/postgresql/pgbouncer.ini`)

```ini
[databases]
# map logical DB names to Postgres backends
mydb = host=localhost port=5432

[settings]
listen_addr = 127.0.0.1
listen_port = 6432
auth_type = scram-sha-256
# IMPORTANT: use transaction pooling for stateless apps
default_pool_mode = transaction

# tuning knobs (see below)
idle_timeout = 600
max_client_conn = 2000
max_db_conn = 200
client_conn_idle_timeout = 300
server_reset_query = DISCARD ALL
```

Key knobs explained:

| Parameter | Why it matters |
|---|---|
| `default_pool_mode = transaction` | Returns connections to the pool after each statement; perfect for HTTP statelessness. |
| `idle_timeout` | How long (seconds) a backend connection can sit idle before being closed. Set based on typical query latency. |
| `max_db_conn` | Upper bound on backend connections a single database can have. This should stay **under `max_connections`** (e.g., 200 < 1000). |
| `max_client_conn` | How many client connections PgBouncer will allow. Can be larger than `max_connections` because most clients won't need full concurrency. |
| `server_reset_query` | Clears session state before reusing a backend connection. |

### How sizing works with a proxy

Assume:
- You have a max of 200 DB connections available (`max_connections = 200` on your managed Postgres)
- You run PgBouncer on each app node, routing all app traffic to a **single** Postgres instance (or a read/write pair)

Then:
```
Backend connections (Postgres)   ≈ number of active mysql clients × (avg queries per minute / queries per backend)
PgBouncer client pool            ≤ max_client_conn per node
```

A good starting point:
- `max_db_conn = 200` (or maybe 150 to leave headroom)
- `max_client_conn = 2000` (many clients can multiplex on one backend)
- App pool size = `ceil( max_db_conn / instances )` only if you **want** direct connections, but using PgBouncer you can have a single backend pool served to all apps.

### Trade-off: Raise max_connections vs add a proxy?

| Approach | Pros | Cons |
|---|---|---|
| Raise `max_connections` on Postgres | Simple, no extra component | Increases memory pressure; connections cost RAM; the underlying problem (non-scaling pools) remains; expensive on managed services |
| Add PgBouncer/RDS Proxy | Consolidates connections, reduces DB memory pressure, gives better view of usage, enables advanced routing (read/write split, failover) | Requires deployment and maintenance; adds another hop; needs tuning; transaction pooling can't handle session variables |

Most teams find that **adding a proxy gives better long-term stability** than blindly raising `max_connections`. It also surfaces connection usage in metrics (PgBouncer logs show peak usage), helping you size future work.

## Runbook: Confirming Connection Exhaustion in <3 Commands

When you see timeouts or connection limits, run these **immediately**:

```bash
# 1. Check current connections
psql -h $DB_HOST -U $DB_USER -c "SELECT pid, usename, application_name, client_addr FROM pg_stat_activity;"

# 2. Count how many connections are active vs allowed
psql -h $DB_HOST -U $DB_USER -c "SELECT count(*) AS active_connections FROM pg_stat_activity;"

# 3. Show PgBouncer (if running) connection distribution
psql -h localhost -p 6432 -U $DB_USER -c "SELECT PoolState, count(*) FROM pgbouncer.pool_status GROUP BY PoolState;"
```

If `active_connections` is close to or exceeding your `max_connections` while CPU is low, you've got a connection exhaustion incident.

Later, the full mitigation:
- Size your connection pool per-instance based on `max_db_conn / instances`
- Deploy/stitch PgBouncer in front
- Monitor `pg_stat_activity` and `pgbouncer.pool_status` metrics
- Create an alert on connection count vs capacity

## Key Takeaways

1. **Per-instance connection pools are not composable** – they multiply across replicas.
2. **Use a consolidation layer** (PgBouncer or RDS Proxy) with `transaction_pooling` for stateless HTTP services.
3. **Size pools from DB capacity** – keep backend connections under `max_connections` *and* leave headroom.
4. **Raise max_connections only as a short-term fix**; plan to add a proxy for sustainable scaling.
5. **Monitor connection counts** before you get paged – a few lines of SQL are your early-warning system.

---

*Next steps?* If you're hitting this today, spin up a simple PgBouncer container and point your app at it. Then watch `SELECT count(*) FROM pg_stat_activity;` before and after scaling. That will make the abstract math tangible.

Found this helpful? I write practical infrastructure patterns for engineers scaling cloud databases. Subscribe to get posts like this in your inbox.

