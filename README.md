# AWS DNS Architecture — Landing Zone Quick Reference

## High-Level View

- **Route 53 Resolver:** performs DNS resolution.
- **PHZ / Resolver rules / Profiles:** determine how DNS names become visible and where queries are answered.
- **TGW:** provides network reachability to the private endpoint IPs.

**Order of operations:** DNS resolves the name to the endpoint ENI's private IP first. Then the VPC route table → TGW → ENI path carries the traffic. DNS answers *where*; routing answers *how*. TGW does no DNS.

---

## Design Recommendations

- **Decentralised endpoints:** AWS-managed private DNS (Private DNS enabled on each workload VPC's own endpoint).
- **Centralised endpoints:** Route 53 Profiles with VPCE integration is worth considering first (endpoint keeps Private DNS enabled, shared via RAM). Custom PHZ + Alias remains a valid alternative when you need your own names, records or governance.
- **Decentralised DNS ownership:** team-owned PHZs and delegated namespaces.
- **Hybrid DNS:** Resolver rules and endpoints.

---

## Core Model

**PrivateLink = connectivity.** The VPC Endpoint (VPCE) provides a private network path to a service.  
**DNS = discovery.** It determines how a workload finds that VPCE.

These are two separate decisions. Don't conflate them.

---

## DNS Ownership: Three Patterns

| Pattern | Records owned by | Shared via | Best for |
|---|---|---|---|
| Private DNS **enabled** | AWS | Profiles (endpoint association) or Resolver rules | Simple, AWS-managed DNS |
| Private DNS **disabled** (PHZ) | You | PHZ association or Profiles | Central DNS governance |
| Delegated / team-owned PHZs | Delegated team/account | Same as any PHZ (association or Profiles) | Decentralised ownership |

**One-line rule:**
- Private DNS enabled → **AWS owns** the private DNS mapping.
- PHZ + Alias (or CNAME) record → **you own** the DNS record.
- Resolver rules → control DNS forwarding.
- Profiles → distribute DNS configuration (PHZs, Resolver rules, supported endpoints) to many VPCs.
- RAM → shares supported DNS resources across accounts.

---

## VPC+2: The Resolver Address

Every VPC has an Amazon-provided DNS resolver at the base of its CIDR **+2** (e.g. `10.0.0.2`).

This is what workloads use for normal DNS resolution, including:
- AWS-managed private DNS records.
- Customer-managed Private Hosted Zones (PHZs).
- Resolver forwarding rules.

- **Private DNS** = AWS-managed private DNS mapping.
- **PHZ** = customer-managed private DNS records.
- **Resolver rules** = DNS forwarding instructions.

**Important:** You cannot simply query another VPC's +2 address to resolve its private DNS records. Use Route 53 Resolver endpoints for cross-network DNS forwarding.

**Key constraint:** When Private DNS is enabled on an interface VPCE, AWS creates a hidden, AWS-managed PHZ associated with the endpoint's VPC. You cannot manage that zone or associate it with other VPCs yourself.

To make that endpoint's name resolve from other VPCs, you have three options:
- **Attach the endpoint to a Route 53 Profile** (keep Private DNS enabled) and share the Profile via RAM. The name then resolves from every VPC attached to the Profile (Model 2b).
- **Disable Private DNS and own the PHZ**, then share it by association or Profile (Model 3).
- **Keep Private DNS and forward queries** to the endpoint's VPC with Resolver rules (Model 2, discouraged).

This is the fork point for centralising PrivateLink DNS.

---

## PrivateLink Models

### 1. Decentralised PrivateLink

- Workload VPC owns the VPCE.
- Private DNS: **enabled**.
- AWS owns the DNS record.
- AWS-managed private DNS maps the service name to the VPCE's private IPs.
- No central DNS sharing required.
- ✅ Simple.
- ❌ No central control over endpoint ownership.

### 2. Centralised PrivateLink + AWS DNS (Resolver Forwarding)

- Network VPC owns the VPCE.
- Private DNS: **enabled**.
- AWS manages the private DNS records for the endpoint.
- AWS-managed PHZ is associated with the Network VPC only.
- Spoke DNS queries are forwarded to the central VPC's Resolver inbound endpoint.
- Resolver forwarding rules determine which DNS queries are forwarded.
- AWS RAM shares the Resolver rules across VPCs/accounts.
- Spoke VPCs associate the shared Resolver rules.
- TGW routing + security groups control network reachability to the central VPCE.
- ❌ No direct ownership of AWS-managed private DNS records.

**⚠️ AWS anti-pattern:** forwarding spoke queries to a central VPC's inbound Resolver endpoint breaks AZ isolation (spoke resolution now depends on a specific ENI/AZ in the hub instead of resolving locally), and each Resolver endpoint ENI caps at 10k QPS. AWS's hybrid-DNS whitepaper names this pattern directly and recommends sharing PHZs across VPCs instead. The whitepaper is dated Dec 2022 and predates Profiles, which are now the way to do that sharing at scale (Model 2b, Model 3 Option B). Use Model 2 only as a fallback.

| Component | Responsibility |
|---|---|
| Route 53 Resolver + Resolver rules + RAM | DNS resolution and forwarding |
| TGW + route tables + SGs | Network connectivity to central VPCE |

**DNS flow:**

Spoke workload → spoke VPC resolver (+2) → shared forwarding rule → outbound endpoint (Network VPC) → inbound endpoint (Network VPC) → AWS-managed private DNS answer.

### 2b. Centralised PrivateLink + AWS DNS (Route 53 Profiles)

Worth considering first for centralised endpoints.

- Network VPC owns the VPCE.
- Private DNS: **enabled**.
- Create a Route 53 Profile in the Network account and associate the VPCE with it.
- Share the Profile with spoke accounts using AWS RAM.
- Spoke accounts associate their VPCs with the shared Profile.
- The endpoint's DNS name then resolves from every VPC attached to the Profile. No self-managed PHZ, Alias record or Resolver rules needed for VPC-to-VPC resolution.
- Network connectivity (TGW or Cloud WAN) must already exist, and security groups must allow the spokes. The Profile shares DNS only.
- A VPC can have only one Profile associated.
- AWS's documented examples are AWS service endpoints (Lambda, S3, CloudWatch). Confirm support for your service, and check Service Quotas for the endpoints-per-Profile limit.
- ✅ AWS keeps the records in sync. No per-VPC association work.
- ❌ No ownership of the records, so no custom names or records.

### 3. Centralised PrivateLink + Customer DNS

- Network VPC owns the VPCE.
- Private DNS: **disabled**.
- Network account creates a Route 53 Private Hosted Zone (PHZ).
- Name the PHZ for the domain you are overriding: the service's own domain (e.g. `ecr.ap-southeast-2.amazonaws.com`) so existing clients resolve to the endpoint, or your own domain for a custom name.
- Create an Alias record (type A) pointing to the central VPCE's regional DNS name. VPC interface endpoints are a supported Alias target, and Alias is preferred over CNAME here: no query charges, works at the zone apex, tracks the endpoint's IPs automatically.
- Associate the PHZ with the Network VPC.
- Make the PHZ available to Dev/Prod spoke VPCs using either:
  - Option A: Direct cross-account PHZ association.
  - Option B: Route 53 Profiles shared across accounts using AWS RAM.
- Spoke VPCs resolve the PrivateLink service name through the shared PHZ.
- TGW routing + security groups control network reachability to the central VPCE.
- Network team owns DNS records and governance.
- Resolver endpoints are not required just to share a PHZ between AWS VPCs.
- Resolver endpoints are required when integrating with external DNS infrastructure, such as on-premises.
- ✅ Full DNS governance and control.

**Option A: Direct PHZ association**
- Create the PHZ in the Network account.
- Authorise cross-account VPC association.
- Associate the Dev/Prod VPCs with the PHZ.
- Network account retains ownership of the PHZ and records.
- Limit: 300 VPC associations per PHZ. AWS recommends Profiles beyond that.

**Option B: Route 53 Profiles + RAM**
- Create a Route 53 Profile in the Network account.
- Associate the PHZ with the Profile.
- Share the Profile with spoke accounts using AWS RAM.
- Spoke accounts associate their VPCs with the shared Profile.
- PHZ configuration is applied automatically to associated VPCs, and central changes propagate through the Profile.
- A VPC can have only one Profile associated. A Profile supports 1,000 VPC associations by default (adjustable).
- PHZs per Profile is a separate quota. Check Service Quotas rather than assuming it matches the VPC limit.

**Key distinction:**
- Direct association = associate each VPC with the PHZ.
- Profiles + RAM = share a central DNS configuration and associate each VPC once with the Profile.

---

## Provider-Side: PrivateLink with Custom DNS

Different actor from the models above.

This is when **you are the service provider**, exposing your own VPC Endpoint Service backed by an NLB, and want third-party consumers to access it using a friendly DNS name.

There are two approaches.

### 4. Provider-Side + AWS-Managed Custom Private DNS

- Provider creates a VPC Endpoint Service backed by an NLB.
- Provider configures a custom private DNS name on the Endpoint Service (e.g. `api.company.com`). One private DNS name per endpoint service.
- Provider must prove domain ownership through AWS TXT verification.
- The TXT record must be published in a **public** DNS zone. Verification fails if it only exists in a private zone.
- Consumer creates an Interface VPCE to the Endpoint Service.
- Consumer enables Private DNS on their VPCE (only after the provider accepts the connection request, where acceptance is required).
- AWS automatically creates a hidden PHZ associated with the consumer's VPC.
- AWS creates a CNAME in it mapping the provider's custom DNS name to the consumer's regional VPCE DNS name.
- Consumer does not need to create their own PHZ.
- ✅ Friendly custom DNS name with automatic consumer-side DNS configuration.
- ❌ No direct ownership of the AWS-managed private DNS mapping.

**Important distinction:**
- Provider owns the custom domain name and verifies domain ownership.
- AWS owns the private DNS mapping.
- The PHZ is created in the consumer's VPC, not the provider's VPC.
- Each consumer gets DNS mapping to their own VPCE.

### 5. Provider-Side + Customer-Managed PHZ

- A customer-managed Route 53 PHZ holds the records for the service name.
- Records point at consumer-specific VPCE DNS names.
- PHZ can be shared across AWS accounts using:
  - Option A: Direct cross-account PHZ association.
  - Option B: Route 53 Profiles shared through AWS RAM.
- Consumer VPCs associate with the PHZ or shared Profile.
- Consumer still creates their own Interface VPCE to the Endpoint Service.
- Consumer must have network connectivity to their VPCE ENIs.
- ✅ Full control of names and records. No public DNS footprint needed.

**Important limitation:**
- Each consumer creates their own VPCE with its own endpoint-specific DNS name and private IP addresses.
- A single shared PHZ record cannot automatically point to a different consumer-specific VPCE in each VPC.
- Each consumer therefore needs its own record. Either the consumer owns it (Model 3 applied to a partner service), or the provider maintains one record per consumer using that consumer's VPCE DNS name. Option 4 automates this.
- Sharing a PHZ does not share PrivateLink connectivity.

**4 vs 5:**
- Option 4: many consumers, one shared name, no ongoing admin.
- Option 5: per-consumer names, or no public DNS footprint, at the cost of maintaining records.

---

## Decentralised DNS Ownership (Team-Owned Zones)

- Give each team a child PHZ (e.g. `team-a.corp.internal`) in its own account, associated with the VPCs that need it.
- When a VPC is associated with overlapping PHZs (`corp.internal` and `team-a.corp.internal`), Resolver uses the most specific match.
- Subdomains can also be delegated with NS records in a PHZ.
- If the matching zone has no matching record, Resolver returns NXDOMAIN. It does not fall back to public DNS.
- Share team zones like any PHZ: direct association, or a Profile.

---

## Central DNS (No PrivateLink Involved)

Network account provides:
- Resolver endpoints (inbound/outbound).
- Private Hosted Zones (PHZs).
- DNS forwarding rules.
- DNS Firewall rules, if required.
- Route 53 Profiles for centralised DNS configuration.

All can be shared across accounts using AWS RAM where supported.

- **Resolver** = where the DNS query gets sent.
- **Resolver rules** = which DNS queries get forwarded and where.
- **PHZ** = who owns the DNS record.
- **RAM** = how supported DNS resources are shared.
- **Route 53 Profiles** = how DNS configurations (PHZs, rules, DNS Firewall rule groups, supported endpoints) are distributed across multiple VPCs.

**Hybrid note:** if a PHZ and a Resolver forwarding rule cover the same domain, the Resolver rule takes precedence, so queries go to your network instead of the PHZ.

---

## Shared Egress (Adjacent, Not DNS)

Also centralised in the Network account, but a distinct concern:

- **PrivateLink** = private access to AWS/partner services.
- **Shared Egress** = controlled internet-bound access.
- **TGW** = network connectivity between VPCs.

Keep these separate when explaining the architecture.

---

## Spoke VPC Setup — Decision Table

| You want... | Do this |
|---|---|
| AWS to manage the DNS record (endpoint in the same VPC) | Enable Private DNS on the VPCE |
| Centralise AWS-managed endpoint DNS (try first) | Keep Private DNS enabled + attach the VPCE to a Route 53 Profile + share via RAM |
| Centralise AWS-managed endpoint DNS (fallback) | Private DNS enabled + Resolver forwarding to the central VPC (discouraged) |
| Central governance/audit of DNS | Disable Private DNS + create customer-managed PHZ + Alias record |
| Share a PHZ across accounts | Direct PHZ association (300 VPC limit per PHZ) or Route 53 Profiles + RAM |
| Centralise DNS configuration at scale | Route 53 Profiles + RAM |
| Hybrid DNS integration | Resolver inbound/outbound endpoints + forwarding rules |
| Decentralised ownership per team | Team-owned child PHZs and/or NS delegation |
| Expose your own PrivateLink service with a friendly DNS name | Provider-side custom private DNS + TXT verification |
| Keep your own DNS records when exposing a PrivateLink service | Customer-managed PHZ, with consumer-specific DNS targets |

---

## Simple Summary

Think of PrivateLink DNS in terms of **who owns the endpoint, who owns DNS, and how DNS is distributed**.

| Model | Endpoint owner | DNS owner | DNS distribution |
|---|---|---|---|
| 1. Decentralised PrivateLink | Workload VPC | AWS | Automatic |
| 2. Centralised + AWS DNS (Resolver forwarding) | Network VPC | AWS | Resolver rules + RAM |
| 2b. Centralised + AWS DNS (Profiles) | Network VPC | AWS | Profile + RAM |
| 3. Centralised + Customer DNS | Network VPC | Network team | PHZ association or Profile + RAM |
| 4. Provider + AWS custom DNS | Service provider (NLB) / consumer (VPCE) | AWS manages mapping; provider owns domain | Automatic per consumer |
| 5. Provider + Customer PHZ | Service provider (NLB) / consumer (VPCE) | PHZ owner | PHZ sharing, with consumer-specific DNS considerations |

**The three things to remember:**

1. **PrivateLink** provides the private network path.
2. **Route 53** determines how workloads discover the endpoint.
3. **RAM + Route 53 Profiles** distribute DNS configuration across accounts.

AWS manages the DNS mapping when you use AWS-managed Private DNS. You manage the records when you use your own PHZ.
