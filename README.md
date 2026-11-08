# AWS DNS Architecture — Landing Zone Quick Reference

## Core Model

**PrivateLink = connectivity.** The VPC Endpoint (VPCE) gives you a private network path to a service.  
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

**Key constraint:** Private DNS enabled → AWS creates its own PHZ, locked to the single VPC with the endpoint — can't be associated to any other VPC. Private DNS disabled → you create the PHZ, and you can associate it with additional VPCs, including cross-account. This is the actual fork point for centralising PrivateLink across spokes — not a style preference, a hard limit on the AWS-managed zone.

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
- AWS manages the private DNS records for the endpoint
- Route 53 Resolver provides centralised DNS resolution
- Resolver forwarding rules determine which DNS queries are forwarded to central DNS or external DNS servers
- AWS RAM shares Resolver rules across VPCs/accounts
- Spoke VPCs associate shared Resolver rules to use the centralised DNS configuration
- TGW routing + security groups control network reachability to the central VPCE
- ❌ No direct ownership of AWS-managed private DNS records

Component	Responsibility  
Route 53 Resolver + RAM	DNS resolution and forwarding  
TGW + route tables + SGs	Network connectivity to central VPCE

### 3. Centralised PrivateLink + Customer DNS
- Network VPC owns the VPCE
- Private DNS: **disabled**
- Network account creates a Route 53 Private Hosted Zone (PHZ)
- Create a CNAME record pointing to the central VPCE's DNS name
- Associate the PHZ with the Network VPC
- Make the PHZ available to Dev/Prod spoke VPCs using either:
  - Option A: Direct cross-account PHZ association
  - Option B: Route 53 Profiles shared across accounts using AWS RAM
- Spoke VPCs resolve the PrivateLink service name through the shared PHZ
- TGW routing + security groups control network reachability to the central VPCE
- Network team owns DNS records and governance
- ✅ Full DNS governance and control

### 3b. Centralised PrivateLink + Customer DNS (Scaled via Profiles)
- Network VPC owns the VPCE + PHZ
- Private DNS: **disabled**
- PHZ added as a resource in a Route 53 Profile
- Profile shared to spokes via RAM
- Spokes associate the Profile once — no per-VPC PHZ association, no 300-VPC cap
- Resolver endpoint only required for hybrid (on-prem) queries
- ✅ Full governance, scales past 300 spokes

---

## Provider-Side: Custom Domain via TXT-Verified Private DNS Name

Different actor than the models above — this is for when **you're the service provider** exposing a VPC Endpoint Service, and want third-party consumers to reach it via *your* custom domain instead of the generated `vpce-svc-xxxx` name. Still uses an AWS-managed PHZ under the hood — this is a variant of Private DNS **enabled**, not a fourth ownership type.

**Provider side:**
- Add a private DNS name to the endpoint service config (new or existing service)
- AWS issues a domain verification value/name → you publish it as a **TXT record in your public DNS zone** (must be publicly resolvable — a private zone won't verify)
- One private DNS name per endpoint service; can verify at the domain or subdomain level

**Consumer side:**
- `enableDnsHostnames` / `enableDnsSupport` must be `true` on their VPC
- Consumer can't enable Private DNS on their interface endpoint until the provider accepts the connection request
- Once enabled: AWS auto-creates a PHZ in the **consumer's** VPC with a CNAME mapping your custom domain → the endpoint's regional DNS name — zero manual PHZ work on their end

**Same constraint as above still applies:** that auto-created PHZ is locked to the consumer's own VPC. It doesn't help *you* centralise or fan out DNS — it just gives *their* single VPC a friendly name for free.

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
