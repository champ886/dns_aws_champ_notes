# AWS DNS Architecture — Landing Zone Quick Reference

## Core Model

**PrivateLink = connectivity.** The VPC Endpoint (VPCE) provides a private network path to a service.  
**DNS = discovery.** It determines how a workload finds that VPCE.

These are two separate decisions. Don't conflate them.

---

## DNS Ownership: Three Patterns

| Pattern | Records owned by | Shared via RAM | Best for |
|---|---|---|---|
| Private DNS **enabled** | AWS | Resolver rules / Profiles | Simple, AWS-managed DNS |
| Private DNS **disabled** (PHZ) | You | PHZ associations / Profiles | Central DNS governance |
| DNS Delegation | Delegated team/account | Depends on design | Decentralised ownership |

**One-line rule:**
- Private DNS enabled → **AWS owns** the private DNS mapping.
- PHZ + Alias (or CNAME) record → **you own** the DNS record.
- Resolver rules → control DNS forwarding.
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

**Key constraint:** When Private DNS is enabled on an interface VPCE, AWS creates a hidden, AWS-managed PHZ associated with the endpoint's VPC. You cannot directly manage or share that hidden zone with other VPCs.

When Private DNS is disabled, you can create your own PHZ and associate it with multiple VPCs, including cross-account VPCs.

This is the actual fork point for centralising PrivateLink DNS.

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

### 2. Centralised PrivateLink + AWS DNS

- Network VPC owns the VPCE.
- Private DNS: **enabled**.
- AWS manages the private DNS records for the endpoint.
- AWS-managed PHZ is associated with the Network VPC only.
- Route 53 Resolver provides centralised DNS resolution.
- Spoke DNS queries are forwarded to the central VPC's Resolver inbound endpoint.
- Resolver forwarding rules determine which DNS queries are forwarded to central DNS.
- AWS RAM shares Resolver rules across VPCs/accounts.
- Spoke VPCs associate shared Resolver rules to use the centralised DNS configuration.
- TGW routing + security groups control network reachability to the central VPCE.
- ❌ No direct ownership of AWS-managed private DNS records.

**⚠️ AWS anti-pattern:** forwarding spoke queries to one central VPC's inbound Resolver endpoint breaks AZ isolation (spoke resolution now depends on reaching a specific ENI/AZ in the hub instead of resolving locally) and each Resolver endpoint ENI caps at 10k QPS. AWS's own hybrid-DNS whitepaper names this pattern directly and recommends cross-account PHZ association (Model 3) or Profiles (3b) instead. Use Model 2 only when Private DNS must stay enabled and you accept that ceiling.

| Component | Responsibility |
|---|---|
| Route 53 Resolver + RAM | DNS resolution and forwarding |
| TGW + route tables + SGs | Network connectivity to central VPCE |

**DNS flow:**

Spoke workload → Spoke Resolver → Outbound Resolver endpoint → Central Resolver inbound endpoint → AWS-managed private DNS resolution.

### 3. Centralised PrivateLink + Customer DNS

- Network VPC owns the VPCE.
- Private DNS: **disabled**.
- Network account creates a Route 53 Private Hosted Zone (PHZ).
- Create an Alias record (type A) pointing to the central VPCE's regional DNS name — VPC interface endpoints are a supported Alias target, and AWS recommends Alias over CNAME here: no query charges, works at the zone apex, tracks the endpoint's IPs automatically.
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

**Option B: Route 53 Profiles + RAM**
- Create a Route 53 Profile in the Network account.
- Associate the PHZ with the Profile.
- Share the Profile with spoke accounts using AWS RAM.
- Spoke accounts associate their VPCs with the shared Profile.
- PHZ configuration is automatically applied to associated VPCs.
- Central DNS changes propagate through the Profile.
- A Profile supports up to 1,000 VPC associations by default (adjustable). Don't confuse this with the separate 5,000 default for PHZ-to-Profile associations (how many hosted zones one Profile can carry) — different quota, same feature.

**Key distinction:**
- Direct association = associate each VPC with the PHZ.
- Profiles + RAM = share a central DNS configuration and associate VPCs with the Profile.

---

## Provider-Side: PrivateLink with Custom DNS

Different actor from the models above.

This is when **you are the service provider**, exposing your own VPC Endpoint Service backed by an NLB, and want third-party consumers to access it using a friendly DNS name.

There are two approaches.

### 4. Provider-Side + AWS-Managed Custom Private DNS

- Provider creates a VPC Endpoint Service backed by an NLB.
- Provider configures a custom private DNS name on the Endpoint Service (e.g. `api.company.com`).
- Provider must prove domain ownership through AWS TXT verification.
- TXT record is published in the provider's public DNS zone.
- Consumer creates an Interface VPCE to the Endpoint Service.
- Consumer enables Private DNS on their VPCE (after provider acceptance, where required).
- AWS automatically creates a hidden PHZ in the consumer's VPC.
- AWS creates a CNAME mapping the provider's custom DNS name to the consumer's regional VPCE DNS name.
- AWS manages the private DNS mapping.
- Consumer does not need to create their own PHZ.
- ✅ Friendly custom DNS name with automatic consumer-side DNS configuration.
- ❌ No direct ownership of the AWS-managed private DNS mapping.

**Important distinction:**
- Provider owns the custom domain name and verifies domain ownership.
- AWS owns the private DNS mapping.
- The PHZ is created in the consumer's VPC, not the provider's VPC.
- Each consumer gets DNS mapping to their own VPCE.

### 5. Provider-Side + Customer-Managed PHZ

- Provider creates and owns a customer-managed Route 53 PHZ.
- Provider creates DNS records within the PHZ.
- PHZ can be shared across AWS accounts using:
  - Option A: Direct cross-account PHZ association.
  - Option B: Route 53 Profiles shared through AWS RAM.
- Consumer VPCs associate with the PHZ or shared Profile.
- Provider retains ownership and governance of the DNS zone.
- Consumer still creates their own Interface VPCE to the Endpoint Service.
- Consumer must have network connectivity to their VPCE ENIs.
- ✅ Provider retains DNS ownership and governance.

**Important limitation:**
- Each consumer creates their own VPCE with its own endpoint-specific DNS name and private IP addresses.
- A single shared PHZ record cannot automatically point to a different consumer-specific VPCE in each VPC.
- If consumers have different VPCE DNS targets, they need individual DNS records in their own PHZs, or use the AWS-managed custom private DNS feature in Option 4.
- Sharing a PHZ does not share PrivateLink connectivity.

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
- **Route 53 Profiles** = how DNS configurations are distributed across multiple VPCs.

---

## Shared Egress (Adjacent, Not DNS)

Also centralised in the Network account, but a distinct concern:

- **PrivateLink** = private access to AWS/partner services.
- **Shared Egress** = controlled internet-bound access.
- **TGW** = network connectivity between VPCs.
- **Route 53 Resolver** = DNS resolution across network boundaries.

Keep these separate when explaining the architecture.

---

## Spoke VPC Setup — Decision Table

| You want... | Do this |
|---|---|
| AWS to manage the DNS record | Enable Private DNS on the VPCE |
| Centralise AWS-managed PrivateLink DNS | Enable Private DNS on central VPCE + Resolver forwarding to central VPC |
| Central governance/audit of DNS | Disable Private DNS + create customer-managed PHZ |
| Share PHZ across multiple accounts | Use direct PHZ association or Route 53 Profiles + RAM |
| Centralise DNS configuration at scale | Route 53 Profiles + RAM |
| Hybrid DNS integration | Resolver inbound/outbound endpoints + forwarding rules |
| Decentralised ownership per team | Use DNS delegation to sub-zones |
| Expose your own PrivateLink service with a friendly DNS name | Configure provider-side custom private DNS + TXT verification |
| Retain your own DNS records when exposing a PrivateLink service | Customer-managed PHZ, with consumer-specific DNS targets where necessary |

---

## Simple Summary

Think of PrivateLink DNS in terms of **who owns the endpoint, who owns DNS, and how DNS is distributed**.

| Model | Endpoint owner | DNS owner | DNS distribution |
|---|---|---|---|
| Decentralised PrivateLink | Workload VPC | AWS | Automatic |
| Centralised + AWS DNS | Network VPC | AWS | Resolver forwarding + RAM |
| Centralised + Customer DNS | Network VPC | Network team | PHZ association or Profiles + RAM |
| Provider + AWS custom DNS | Service provider (NLB) / consumer (VPCE) | AWS manages mapping; provider owns domain | Automatic per consumer |
| Provider + Customer PHZ | Service provider (NLB) / consumer (VPCE) | Service provider | PHZ sharing, with consumer-specific DNS considerations |

**The three things to remember:**

1. **PrivateLink** provides the private network path.
2. **Route 53** determines how workloads discover the endpoint.
3. **RAM + Route 53 Profiles** distribute DNS configuration across accounts.

AWS manages the DNS mapping when you use AWS-managed Private DNS. You manage the records when you use your own PHZ.
