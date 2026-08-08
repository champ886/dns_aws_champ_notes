# AWS DNS Architecture — Landing Zone Quick Reference

## Core Model

**PrivateLink = connectivity.** The VPC Endpoint (VPCE) gives you a private network path to a service. /n
**DNS = discovery.** It decides how a workload actually *finds* that VPCE.

These are two separate decisions — don't conflate them.

---

## DNS Ownership: Three Patterns

| Pattern | Records owned by | Shared via RAM | Best for |
|---|---|---|---|
| Private DNS **enabled** | AWS | Resolver rules | Simple, AWS-managed DNS |
| Private DNS **disabled** (PHZ) | You | PHZ associations (+ Resolver rules if hybrid) | Central DNS governance |
| DNS Delegation | Delegated team/account | Usually none | Decentralised ownership |

**One-line rule:**
- Private DNS enabled → **AWS owns** the DNS record.
- PHZ + Alias record → **you own** the DNS record.
- The Network account owns *connectivity* either way.

---

## VPC+2: the resolver address

Every VPC has an Amazon-provided DNS resolver at the base of its CIDR **+2** (e.g. `10.0.0.2`). This is what interface endpoints and PHZs rely on for in-VPC resolution — and what spokes reach via Resolver endpoints for cross-VPC resolution.

- **Private DNS** = AWS-managed private DNS records
- **PHZ** = customer-managed private DNS records

---

## PrivateLink Models

### 1. Decentralised PrivateLink
- Workload VPC owns the VPCE
- Private DNS: **enabled**
- AWS owns the DNS record
- ✅ Simple ❌ No central control

### 2. Centralised PrivateLink + AWS DNS
- Network VPC owns the VPCE
- Private DNS: **enabled**
- AWS still owns the DNS record
- Resolution can be centralised via Resolver + RAM
- ❌ No DNS ownership on your side

### 3. Centralised PrivateLink + Customer DNS
- Network VPC owns the VPCE
- Private DNS: **disabled**
- You create a Route 53 PHZ + Alias record
- Network team owns DNS
- ✅ Full governance

---

## Central DNS (no PrivateLink involved)

Network account provides:
- Resolver endpoints (inbound/outbound)
- PHZs
- DNS forwarding rules

All shared to spokes via RAM.

- **Resolver** = where the query gets sent
- **PHZ** = who owns the record

---

## Shared Egress (adjacent, not DNS)

Also centralised in the Network account, but a distinct concern:
- **PrivateLink** = private access to AWS/partner services
- **Shared Egress** = controlled internet-bound access

Keep these separate when explaining the architecture — they get conflated a lot.

---

## Spoke VPC Setup — Decision Table

| You want... | Do this |
|---|---|
| AWS to manage the DNS record | Enable Private DNS on the VPCE → share Resolver rules via RAM |
| Central governance/audit of DNS | Disable Private DNS → create PHZ → share PHZ association (+ Resolver rules for hybrid) |
| Decentralised ownership per team | Use DNS delegation to sub-zones |
