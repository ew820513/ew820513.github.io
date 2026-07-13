---
title: "Taming AWS Security Group Sprawl: A Practical Guide to Network Auditing"
date: 2026-07-12
draft: false
tags: ["architecture", "aws", "networking", "security"]
---

## Problem

Security Group sprawl is a common day-to-day problem that emerges in growing AWS environments. You start with a few clean Security Groups, but over time they multiply uncontrollably — dozens of groups with overlapping rules, unclear purposes, and mysterious inbound/outbound connections. When something breaks or you need to audit for compliance, you're left sifting through a tangled web of rules that nobody fully understands. Network troubleshooting becomes guesswork, and security reviews take days instead of hours.

**A practicing engineer will encounter this when:**
- Onboarding a new service and finding 57 Security Groups with unclear naming
- Debugging connectivity issues where traffic should flow but doesn't
- Security audits where you need to prove least-privilege access but rules contradict each other
- Cleanup sprints where you risk breaking production by removing "unused" rules

The core issue is that Security Groups lack inherent structure — they're just sets of IP ranges and ports. Without discipline, they become the "junk drawer" of your cloud network.

## Architecture Decisions

### 1. **Adopt a Naming Convention with Purpose Prefixes**

**Decision**: Use descriptive names that encode the Security Group's role.

Rather than generic names like `sg-123abc` or `web-server`, I recommend a structured format:

```
{environment}-{service}-{purpose}-{protocol}
```

Examples:
- `prod-api-https` – HTTPS traffic for the API service
- `prod-api-redis` – Redis access for the API service
- `prod-bastion-admin` – Administrative access via the bastion host

**Why this matters**: When you see `stg-monolith-pgsql` in a list, you immediately know it handles PostgreSQL traffic into the monolith service. This cuts investigation time dramatically because you can eyeball a rule and understand its intent.

**Trade-offs**: Slightly longer names, but the clarity payoff outweighs the typing cost. You'll spend less time in documentation and more time fixing problems.

### 2. **Tag Everything, Not Just for Cost Allocation**

**Decision**: Apply mandatory tags that support security operations.

```yaml
# Required tags for every Security Group
Environment: prod|staging|dev
Service: api-gateway|user-service|payments
Owner: team-network|team-platform
Purpose: short description
ManagedBy: terraform|cloudformation|manual
```

**Why this matters**: When you need to find all Security Groups touching the payments service across all environments, tags let you query via AWS CLI or Console filters instead of manual spelunking. Cost allocation is nice, but operational efficiency is the real win.

### 3. **Use Security Group Referencing (Not CIDR) Where Appropriate**

**Decision**: Reference Security Groups instead of IP ranges when you control both sides.

Instead of:
```hcl
# Anti-pattern: wide CIDR ranges
ingress {
  from_port   = 5432
  to_port     = 5432
  protocol    = "tcp"
  cidr_blocks = ["10.0.0.0/16"]  # Anyone in VPC can connect
}
```

Use:
```hcl
# Better: explicit SG references
ingress {
  from_port       = 5432
  to_port         = 5432
  protocol        = "tcp"
  security_groups = [aws_security_group.api.id]  # Only API instances
}
```

**Why this matters**: Security Group references create intentional coupling — you can only access this resource if you're running in an approved source Security Group. This enables least privilege without managing IP allocations, and when instances are terminated, their access disappears automatically.

### 4. **Scope Security Groups by Service Tier, Not Traffic Direction**

**Decision**: Use one Security Group per application tier or service, each containing both ingress and egress rules.

Many teams create one SG per application tier (web, app, data) that defines the full network contract for that tier. For example, a web-tier SG allows HTTPS inbound from the ALB and all necessary outbound traffic; an app-tier SG allows inbound from the web tier and outbound to the data tier. Keeping both directions in one SG per tier maintains a clear "this is how this component communicates with the world" contract.

**Why this works**: AWS Security Groups are **stateful** — response traffic for allowed connections is automatically permitted regardless of the corresponding direction's rules. Separating ingress and egress into distinct SGs adds no security benefit, doubles your SG count (exacerbating sprawl), and increases administrative overhead with no defensive gain.

**Anti-pattern**: Creating separate `-ingress-` and `-egress-` SGs for the same service. This doubles the number of SGs you manage, makes it harder to reason about a component's network footprint, and runs counter to AWS best practices.

### 5. **Implement Automated Auditing with AWS Config**

**Decision**: Use AWS Config managed rules to detect sprawl early.

```json
{
  "ConfigRuleName": "security-group-restricted-common-ports",
  "Description": "Checks that Security Groups don't allow unrestricted access to common ports",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "restricted-common-ports"
  }
}
```

Set up custom rules that alert when:
- Security Groups have >10 rules (sign of sprawl)
- Overlapping CIDR ranges are detected
- Unused Security Groups accumulate (no ENIs attached)

## Script vs. AWS Config: When to Use Which

Both the audit script and AWS Config rules detect Security Group problems, but they serve different purposes:

| | Audit Script | AWS Config Rule |
|---|---|---|
| Trigger | Manual or scheduled cron | Continuous, event-driven |
| Remediation | None (output only) | Auto-remediate via SSM |
| Scope | Single region, current credentials | All regions, all accounts (via aggregator) |
| Custom logic | Arbitrary Python | Lambda-backed for custom rules |
| Results | stdout / logs | Config dashboard + SNS / EventBridge |
| Cost | Free | ~$0.001 per evaluation |

**Use the script** for ad-hoc investigation, one-off audits during cleanup sprints, or when you need custom detection logic that isn't worth the overhead of a Lambda-backed Config rule.

**Use AWS Config** for continuous compliance monitoring, alerting, cross-account visibility, and producing audit trails for security reviews.

The two approaches are complementary, not competing. As noted in the implementation section below, the script is also a natural candidate for wrapping as a Lambda-backed custom Config rule — giving you custom detection logic with the operational benefits of Config's dashboard and alerting.

## Implementation

Here's a focused Python script that audits your Security Groups and identifies sprawl patterns. This is the kind of tool you'd run weekly to maintain visibility.

Key behaviors:
- **Paginated**: handles accounts with more than 1,000 Security Groups without silently truncating results
- **IPv6-aware**: checks both `0.0.0.0/0` and `::/0` for wide-open ingress
- **Unused SG detection**: cross-references ENI attachments to find Security Groups nothing is actually using
- **Typed**: annotated function signatures for editor support and static analysis

```python
#!/usr/bin/env python3
# /// script
# requires-python = ">=3.11"
# dependencies = [
#   "boto3",
# ]
# ///
"""
Security Group Sprawl Detector
Run this to identify problematic patterns in your AWS Security Groups.
"""

import boto3


def fetch_all_security_groups(ec2) -> list[dict]:
    paginator = ec2.get_paginator("describe_security_groups")
    return [sg for page in paginator.paginate() for sg in page["SecurityGroups"]]


def fetch_attached_sg_ids(ec2) -> set[str]:
    paginator = ec2.get_paginator("describe_network_interfaces")
    attached: set[str] = set()
    for page in paginator.paginate():
        for eni in page["NetworkInterfaces"]:
            for group in eni.get("Groups", []):
                attached.add(group["GroupId"])
    return attached


def find_large_sgs(sgs: list[dict], threshold: int = 10) -> list[dict]:
    results = []
    for sg in sgs:
        rule_count = len(sg.get("IpPermissions", [])) + len(sg.get("IpPermissionsEgress", []))
        if rule_count > threshold:
            results.append({
                "GroupId": sg["GroupId"],
                "GroupName": sg.get("GroupName", "unnamed"),
                "RuleCount": rule_count,
            })
    return results


def find_untagged_sgs(sgs: list[dict]) -> list[str]:
    results = []
    for sg in sgs:
        tag_keys = {t["Key"] for t in sg.get("Tags", [])}
        if "Purpose" not in tag_keys and "Service" not in tag_keys:
            results.append(sg["GroupId"])
    return results


def find_wide_open_sgs(sgs: list[dict]) -> list[dict]:
    results = []
    for sg in sgs:
        for perm in sg.get("IpPermissions", []):
            for ip_range in perm.get("IpRanges", []):
                if ip_range.get("CidrIp") == "0.0.0.0/0":
                    results.append({
                        "GroupId": sg["GroupId"],
                        "Port": perm.get("FromPort"),
                        "Protocol": perm.get("IpProtocol"),
                        "Cidr": "0.0.0.0/0",
                    })
            for ip_range in perm.get("Ipv6Ranges", []):
                if ip_range.get("CidrIpv6") == "::/0":
                    results.append({
                        "GroupId": sg["GroupId"],
                        "Port": perm.get("FromPort"),
                        "Protocol": perm.get("IpProtocol"),
                        "Cidr": "::/0",
                    })
    return results


def find_unused_sgs(sgs: list[dict], attached_ids: set[str]) -> list[dict]:
    return [
        {"GroupId": sg["GroupId"], "GroupName": sg.get("GroupName", "unnamed")}
        for sg in sgs
        if sg["GroupId"] not in attached_ids and sg.get("GroupName") != "default"
    ]


def detect_sprawl() -> dict[str, int]:
    ec2 = boto3.client("ec2")

    sgs = fetch_all_security_groups(ec2)
    attached_ids = fetch_attached_sg_ids(ec2)

    large_sgs = find_large_sgs(sgs)
    untagged_sgs = find_untagged_sgs(sgs)
    wide_open_sgs = find_wide_open_sgs(sgs)
    unused_sgs = find_unused_sgs(sgs, attached_ids)

    print("=== Security Group Sprawl Report ===\n")

    if large_sgs:
        print(f"Large Security Groups (>10 rules): {len(large_sgs)} found")
        for sg in large_sgs[:5]:
            print(f"  - {sg['GroupName']} ({sg['GroupId']}): {sg['RuleCount']} rules")

    if untagged_sgs:
        print(f"\nUntagged Security Groups: {len(untagged_sgs)} found")
        print(f"  First 5: {', '.join(untagged_sgs[:5])}")

    if wide_open_sgs:
        print(f"\nWide-open Security Groups (0.0.0.0/0 or ::/0): {len(wide_open_sgs)} found")
        for sg in wide_open_sgs[:5]:
            print(f"  - {sg['GroupId']}: {sg['Protocol']}/{sg['Port']} from {sg['Cidr']}")

    if unused_sgs:
        print(f"\nUnused Security Groups (no ENI attachment): {len(unused_sgs)} found")
        for sg in unused_sgs[:5]:
            print(f"  - {sg['GroupName']} ({sg['GroupId']})")

    return {
        "large_sgs": len(large_sgs),
        "untagged_sgs": len(untagged_sgs),
        "wide_open_sgs": len(wide_open_sgs),
        "unused_sgs": len(unused_sgs),
    }


if __name__ == "__main__":
    results = detect_sprawl()
    print(f"\nTotal sprawl indicators: {sum(results.values())}")
```

Run this weekly via cron:

```bash
0 9 * * 1 uv run /path/to/security_group_audit.py
```

**A note on unused SG detection accuracy**: The ENI-based approach above covers EC2 instances, ECS tasks, Lambda functions in VPCs, and most other compute. It won't catch SGs that are only referenced by other SGs (as a source) but have no active ENI attachments themselves. For those edge cases, you'd need an additional pass with `describe_security_groups` filtered by `ip-permission.group-id`. For a weekly sprawl audit, ENI coverage is sufficient.

**For production environments**, consider deploying this audit as a **custom Lambda function** triggered on a schedule via EventBridge (CloudWatch Events). This removes the need for a dedicated server and integrates with incident response workflows. You can also wrap it as a **custom AWS Config rule** (a Lambda-backed Config rule) so violations appear alongside managed rules in the Config dashboard — giving you a single pane of glass for both built-in and custom compliance checks.

## Next Steps

1. **Day 1**: Implement the naming convention on all new Security Groups
2. **Week 1**: Backfill tags on existing SGs using the audit script
3. **Month 1**: Convert CIDR-based rules to SG references where possible
4. **Ongoing**: Weekly automated audits to prevent sprawl creep

The goal isn't perfection — it's making your network legible enough that the next person (or future you) can debug an issue without spending a day untangling invisible dependencies.
