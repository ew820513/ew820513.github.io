---
title: "VPC Peering vs Transit Gateway: Cost vs Complexity vs Scale in Multi-Account AWS"
date: 2026-11-09
draft: true
tags: ["architecture", "cloud"]
---

# VPC peering vs Transit Gateway: cost vs complexity vs scale in multi-account AWS

You have 15 AWS accounts — prod, staging per team, shared services — and they all need to talk to each other. Two accounts? A single VPC peering connection and you're done. Fifteen accounts? Suddenly you're looking at 105 peering connections, route tables that span multiple pages, and DNS resolution that fails silently across peering links.

This post walks through the real trade-offs between VPC Peering and Transit Gateway (TGW) at multi-account scale, with a calculator you can use to pick the right one for your environment, plus a migration path that doesn't require a big-bang cutover.

## The Problem: "Works in Dev" Breaks in Production

A peering mesh scales quadratically. With N accounts, you need N × (N−1) / 2 peering connections if every account talks to every other account. For 15 accounts, that's 105 connections. Each one needs:

- A peering connection request and acceptance (cross-account, if accounts differ)
- Route table entries on both sides
- Security group rules referencing the peer's CIDR (or vice versa)
- DNS configuration so private hostnames resolve across the link

The breaking point isn't the connection count itself. It's the ops surface area. Adding a new account means touching N existing accounts' route tables and security groups. Transitive routing doesn't exist: if A peers with B, and B peers with C, A cannot reach C without an explicit A↔C peering connection. And overlapping CIDR ranges kill peering outright — you can't peer two VPCs with overlapping CIDRs.

This is exactly why the mesh feels manageable at 2–3 accounts and unsustainable at 15.

## The Alternative: Transit Gateway

Transit Gateway turns the mesh into a hub-and-spoke. Each account attaches its VPC(s) to a single TGW in a shared-services account. One attachment per account, centralized routing, consistent DNS, and built-in route propagation.

The trade-off is cost: TGW charges per-GB data processing plus an hourly fee per attachment. For 15 attachments at roughly $0.02/hour each, that's about $21/month just for attachments, before you move a single byte of traffic.

## Cost Model at Scale

Here's a side-by-side for 15 accounts with ~100 GB/month of cross-account traffic:

| Cost component | VPC Peering | Transit Gateway |
|---|---|---|
| Data processing | $0.01/GB → $1.00 | $0.06/GB → $6.00 |
| Cross-region transfer | $0.02/GB (if applicable) | $0.02/GB (if applicable) |
| Attachment fees | $0 | ~$21.00 (15 × $0.02/hr × 720 hrs) |
| Total (100 GB/mo) | ~$1.00 | ~$27.30 |

At first glance, peering wins on pure data cost. But the calculator below shows where the curves cross — and why peering's "free" connections become expensive in engineer-hours once you pass a handful of accounts.

### Cost & Complexity Calculator

Use this rule of thumb for N accounts and T GB/month of cross-account traffic:

```
peering_monthly   = T × $0.01 + (N × (N−1) / 2) × $0.10  # $0.10 = rough ops overhead per connection
tgw_monthly       = T × $0.06 + N × $14.40              # $14.40 = $0.02/hr × 720 hrs per attachment
```

| N accounts | 10 GB/mo | 50 GB/mo | 100 GB/mo | 500 GB/mo |
|---|---|---|---|---|
| 3 (3 conns) | $0.33 | $1.65 | $3.30 | $16.50 |
| 5 (10 conns) | $1.00 | $5.00 | $10.00 | $50.00 |
| 8 (28 conns) | $2.80 | $14.00 | $28.00 | $140.00 |
| 15 (105 conns) | $10.50 | $52.50 | $105.00 | $525.00 |
| TGW (any N) | $0.60 + N×$14.40 | $3.00 + N×$14.40 | $6.00 + N×$14.40 | $30.00 + N×$14.40 |

For 15 accounts at 100 GB/month, peering's connection overhead alone (~$10.50) starts approaching TGW's attachment fees (~$21.60), before you count the engineer time spent touching 105 route tables every time you add an account.

## DNS and Service Discovery

Peering DNS is link-by-link. If account A peers with B and C, A needs separate Route 53 Resolver rules (or conditional forwarders) for each peering target. Private hosted zones don't automatically propagate across peering, so service discovery (e.g., `api.internal.sharedsvc.aws`) resolves inconsistently depending on which path a request takes.

TGW centralizes this. Route 53 Resolver endpoints attach to the TGW, and private hosted zones resolve consistently across every spoke attachment. One DNS story, not N.

## Transitive Routing

This is the architectural difference that matters most:

- Peering: point-to-point only. A ↔ B, B ↔ C, but A cannot reach C through B. Every pair that needs to talk gets its own peering connection.
- TGW: hub-and-spoke. A → TGW → C works out of the box via route tables on the TGW. Add a new account and it can reach every existing spoke with one attachment, not N new connections.

If your architecture has a shared-services pattern (all accounts reach a central logging/security/account VPC), TGW's transitive routing is a natural fit. If it's strictly pairwise (prod talks to staging, nothing else), peering is simpler.

## Security and Isolation

Peering gives you granular isolation: each connection is its own boundary, and security group rules are explicit per peer. The downside is you manage N×(N−1)/2 rule sets, and it's easy to forget one when onboarding a new team.

TGW centralizes governance — security groups and network ACLs propagate across attachments — but that central model can be too permissive if you're not careful. A misconfigured TGW route table gives an account reach into every spoke. The mitigations:

- Use TGW route tables per environment (prod, staging, shared services) rather than one giant table.
- Attach VPC resource policies (S3, etc.) so even network reachability doesn't imply data access.
- Keep security groups least-privilege at the destination VPC, not just at the network boundary.

## Migration Path

You don't have to choose once and never change. A pragmatic rollout:

1. N ≤ 5 accounts, low traffic (< 50 GB/month): Start with VPC peering. It's free operationally at this size and you learn your actual traffic patterns before paying for TGW.
2. N ≥ 8 accounts or traffic > 100 GB/month: Move to TGW. The attachment fee is cheaper than the engineer time you'd spend on the peering mesh.
3. Hybrid transition: Keep peering for latency-sensitive or isolated workloads (e.g., database replication between two specific accounts), and route general cross-account traffic through TGW. TGW supports inter-peering attachments so the two can coexist during migration.

### Decision Guidance

| Scenario | Recommendation |
|---|---|
| N ≤ 5 and traffic < 50 GB/month | VPC Peering |
| N ≥ 8 or traffic > 100 GB/month | Transit Gateway |
| 5 < N < 8 with growing traffic | Evaluate now; TGW likely pays for itself in ops time |
| Strictly pairwise, no transitive needs | Peering is fine |
| Shared-services / hub-and-spoke pattern | TGW |
| Overlapping CIDRs across accounts | TGW (peering is impossible) |

## Bottom Line

Peering is free per connection but expensive in complexity: 15 accounts means 105 connections to provision, route, secure, and DNS-enable. Transit Gateway costs ~$21/month in attachment fees plus $0.06/GB, but buys you one attachment per account, centralized routing, consistent DNS, and transitive routing that doesn't break when you add account sixteen.

Use peering when you're small and pairwise. Use TGW when you're multi-account and growing. The crossover is roughly N = 5–8 accounts or 50–100 GB/month of cross-account traffic — whichever you hit first.
