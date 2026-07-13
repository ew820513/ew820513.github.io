---
title: "Taming AWS Security Group Sprawl: A Practical Guide to Network Auditing"
date: 2026-07-12
draft: true
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
    "SourceIdentifier": "SECURITY_GROUPS_RESTRICTED_INBOUND"
  }
}
```

Set up custom rules that alert when:
- Security Groups have >10 rules (sign of sprawl)
- Overlapping CIDR ranges are detected
- Unused Security Groups accumulate (no ENIs attached)

## Implementation

Here's a focused Python script that audits your Security Groups and identifies sprawl patterns. This is the kind of tool you'd run weekly to maintain visibility:

```python
#!/usr/bin/env python3
"""
Security Group Sprawl Detector
Run this to identify problematic patterns in your AWS Security Groups.
"""

import boto3
from collections import defaultdict

def detect_sprawl():
    ec2 = boto3.client('ec2')

    # Get all SGs with their rules
    response = ec2.describe_security_groups()
    sgs = response['SecurityGroups']

    # Find sprawl patterns
    large_sgs = []        # >10 rules
    unnamed_sgs = []      # Missing tags
    wide_open_sgs = []    # 0.0.0.0/0 or ::/0

    for sg in sgs:
        rule_count = len(sg.get('IpPermissions', [])) + len(sg.get('IpPermissionsEgress', []))

        # Pattern 1: Too many rules
        if rule_count > 10:
            large_sgs.append({
                'GroupId': sg['GroupId'],
                'GroupName': sg.get('GroupName', 'unnamed'),
                'RuleCount': rule_count
            })

        # Pattern 2: Missing purpose tags
        tag_dict = {t['Key']: t['Value'] for t in sg.get('Tags', [])}
        if 'Purpose' not in tag_dict and 'Service' not in tag_dict:
            unnamed_sgs.append(sg['GroupId'])

        # Pattern 3: Wide open access
        for perm in sg.get('IpPermissions', []):
            for ip_range in perm.get('IpRanges', []):
                if ip_range.get('CidrIp') == '0.0.0.0/0':
                    wide_open_sgs.append({
                        'GroupId': sg['GroupId'],
                        'Port': perm.get('FromPort'),
                        'Protocol': perm.get('IpProtocol')
                    })

    # Report findings
    print("=== Security Group Sprawl Report ===\n")

    if large_sgs:
        print(f"Large Security Groups (>10 rules): {len(large_sgs)} found")
        for sg in large_sgs[:5]:  # Show first 5
            print(f"  - {sg['GroupName']} ({sg['GroupId']}): {sg['RuleCount']} rules")

    if unnamed_sgs:
        print(f"\nUnnamed Security Groups: {len(unnamed_sgs)} found")
        print(f"  First 5: {', '.join(unnamed_sgs[:5])}")

    if wide_open_sgs:
        print(f"\nWide-open Security Groups (0.0.0.0/0): {len(wide_open_sgs)} found")
        for sg in wide_open_sgs[:5]:
            print(f"  - {sg['GroupId']}: {sg['Protocol']}/{sg['Port']}")

    return {
        'large_sgs': len(large_sgs),
        'unnamed_sgs': len(unnamed_sgs),
        'wide_open_sgs': len(wide_open_sgs)
    }

if __name__ == '__main__':
    results = detect_sprawl()
    print(f"\nTotal sprawl indicators: {sum(results.values())}")
```

Run this weekly via cron:
```bash
# Add to crontab
0 9 * * 1 python3 /path/to/security_group_audit.py
```

## Next Steps

1. **Day 1**: Implement the naming convention on all new Security Groups
2. **Week 1**: Backfill tags on existing SGs using the audit script
3. **Month 1**: Convert CIDR-based rules to SG references where possible
4. **Ongoing**: Weekly automated audits to prevent sprawl creep

The goal isn't perfection — it's making your network legible enough that the next person (or future you) can debug an issue without spending a day untangling invisible dependencies.
