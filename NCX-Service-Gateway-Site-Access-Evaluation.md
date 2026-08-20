# Ericsson Cradlepoint NetCloud Exchange (NCX) Service Gateway
## Evaluation for Private, Zero-Trust Access to Distribution Grid Sites

**Prepared:** 2026-08-20
**Question being answered:** Can the NCX Service Gateway (Secure Connect + ZTNA) replace the current model — open access to grid site gear from the corporate LAN, GlobalProtect/Palo Alto for everyone else — with a single, identity-based, private access path?

**Short answer:** Yes, for the grid sites themselves, *provided* every site rides a Cradlepoint NCX-capable router and you actually decommission the existing routed path from the corporate network to the site subnets. NCX does not "add" privacy on top of a flat network — it replaces the transport, and the flat path has to go away for the security model to mean anything. It is not a general-purpose replacement for GlobalProtect for the rest of the enterprise, and there are real constraints (single-vendor edge, active/standby HA in one data center only, licensed-throughput hard caps, IPv4 unicast only) documented below.

---

## 1. The current problem, stated precisely

| Access path today | Authentication | Authorization | Effective blast radius |
|---|---|---|---|
| In office → site gear | None (implicit network trust) | Routing table | Any host on the corporate LAN can reach any device at any site |
| Remote → GlobalProtect → Palo Alto → site gear | Yes (GP + IdP) | Firewall rules, largely IP/subnet based | Once on the VPN, reachability is network-shaped, not user-shaped |

Two different trust models for the same assets, and the weaker one (in-office) is the one that governs the majority of daily access. The goal — "make site access private" — is really three goals:

1. **Remove implicit trust from the corporate LAN.** Being plugged in should grant nothing.
2. **Make access identity-based and per-resource,** not per-subnet, regardless of where the user sits.
3. **Keep the operational path for machine-to-machine SCADA traffic working** without wrapping it in a user-authentication model that doesn't fit it.

NCX addresses all three, but with different mechanisms for #3 than for #1 and #2. That distinction is the single most important design point in this document — see Section 6.

---

## 2. What NCX actually is

NetCloud Exchange is Ericsson/Cradlepoint's SASE architecture, sold in two delivery models:

- **NetCloud Exchange (customer-hosted)** — you run the Service Gateway VM or appliance in your own data center or VPC.
- **NetCloud SASE (cloud-delivered)** — Ericsson runs the gateway; you consume it as a service.

Both are managed from **NetCloud Manager (NCM)**, the same cloud console you already use for router management. The two models are *not* feature-identical; see Section 8.

### 2.1 Components

| Component | What it is | Role in your solution |
|---|---|---|
| **NetCloud Manager (NCM)** | Cloud control plane / single pane of glass | Defines sites, resources, users, and policy. Never carries data traffic. |
| **NCX Service Gateway** | Virtual appliance (VMware, KVM, AWS, Azure) or SG4000 hardware appliance | The **data plane and policy enforcement point**. Terminates every site tunnel, terminates every ZTNA client, decides what may talk to what. |
| **NCX-enabled routers** | Cradlepoint NCOS routers at each grid site | The spoke. Dials out to the Service Gateway, presents its LAN as defined "resources". |
| **NetCloud Client** | Agent for Windows, macOS, Linux, iOS, Android | The GlobalProtect replacement for user access. Authenticates via SAML, joins the Secure Connect network. |
| **NCX Portal / Clientless ZTNA portal** | Browser-based access | Access without an agent, including full application isolation for third parties. |
| **NetCloud Virtual Edge** | Lightweight virtual router for AWS / Azure / VMware | Extends the Secure Connect fabric to where your applications live (historian, ADMS, jump hosts) so the app side is policy-governed too. |

### 2.2 Service Gateway specifications (as published)

| | Virtual appliance | SG4000 hardware |
|---|---|---|
| Licensed capacity | 250 Mbps – 4 Gbps | 500 Mbps – 4 Gbps |
| Platform | AWS, Azure, KVM, VMware ESXi 6.7 U3+ | 1RU rack mount, Intel 13th-gen i7-13700E, 64 GB DDR5, 2x 10GbE SFP+ |
| VM sizing | 8 vCPU, 16 GB RAM (32 GB in some cloud SKUs), 16 GB disk, 2x 1 Gb/s or better NICs | — |
| Concurrent tunnels | Up to 4,000 | Up to 4,000 |
| HA | Active/standby | Active/standby |
| FIPS 140-3 | Available on customer-hosted with certified hardware | Yes |

Three interfaces are required for full function:

- **mgmt0** — SSH / local UI
- **lan0** — reaches internal resources and DNS servers
- **wan0** — tunnel termination, and the gateway's own path back to NetCloud Manager

**Hard limit worth reading twice:** per Ericsson's NCX terms, if actual throughput exceeds purchased throughput by 10% or more, *the excess traffic is not transmitted*. This is not a soft overage bill — it is traffic loss. Size the gateway for peak, including storm and restoration events when every site is chattering at once.

---

## 3. Secure Connect — the transport layer

Secure Connect is the foundation service; ZTNA, SD-WAN, and the Hybrid Mesh Firewall are all add-ons that require it plus a Service Gateway.

**How it works.** Each site router builds an outbound tunnel to the Service Gateway — **GRE over IPsec**, IKEv2, pre-shared key, with liveness checks / DPD driving failover. One tunnel is built per active WAN interface, so a dual-modem router at a substation maintains two, with priority set by Connection Manager. From NCOS 7.23.10 onward the IKE ID is the GRE key rather than the router's WAN IP, which is what makes this work cleanly behind carrier NAT.

**Why that matters for grid sites:** the site dials out. There is no inbound listener, no static IP requirement, no public IP on the router, and no private APN needed. Cellular sites behind CGNAT work without carrier involvement. Ericsson positions Secure Connect explicitly as a legacy VPN and private-APN replacement.

**Security properties, as designed:**

- **Deny-all by default.** Outbound internet from a site is permitted by default; **site-to-site and site-to-resource traffic is blocked until a policy explicitly allows it.** East/west is off unless you turn it on.
- **Resource obscuring / dark assets.** Devices are reachable only if defined as a "resource" and referenced in a policy. Undefined gear on a site LAN is not addressable across the fabric at all.
- **Name-based (domain-based) routing.** Resources are referenced by name, not IP. This hides addressing and — importantly for utilities — supports **overlapping IP ranges across sites**, which is very common when hundreds of sites were built from the same cookie-cutter 192.168.x template.
- **Split routing.** Direct internet access at the edge for what should not cross the fabric, with selective traffic steered to the gateway.
- **Encryption choice.** IPsec at three cipher levels, or a lighter "micro-tunnel" mode for constrained IoT endpoints.

### 3.1 The access policy model

This is where the "private" part is actually enforced.

- **Sources:** site name, tag, IP subnet, a LAN defined in the router config, or **SAML attributes** (requires the ZTNA license).
- **Destinations:** site name, tag, subnet, FQDN (wildcards supported), named resource, and — with the Hybrid Mesh Firewall license — web category/reputation, application/app category, and geolocation.
- **Logic:** criteria within a source are ANDed; destinations allow OR. Empty criteria means *any*.
- **Evaluation:** strict top-down, first match wins, no further evaluation.
- **Ceiling:** 20 source entries and 20 destination entries per policy.

The SAML-attribute-as-source capability is the hinge of the whole design: it lets one policy engine express *"members of the AD group `Grid-Relay-Techs` may reach resource `RECLOSER-HMI` at sites tagged `District-4`, on TCP/443 only"* — with no reference to a subnet anywhere.

---

## 4. ZTNA — the GlobalProtect replacement

NCX ZTNA is licensed **per user** and requires Secure Connect and a Service Gateway already in place, NCOS 7.23.31 or newer on the gateway and participating endpoints, and an **external SAML 2.0 identity provider** (Okta, Microsoft Entra ID, and AD FS are the named ones).

### 4.1 Three ways a user gets in

1. **NetCloud Client (agent).** Windows, macOS, Linux, iOS, Android. Works from anywhere — corporate LAN, home, or a truck. The user authenticates against your IdP; on success they receive a token that permits them to join the Secure Connect network. Policy then decides which resources resolve and respond.
2. **NCX Portal (browser, on-net).** A user sitting behind a Cradlepoint router authenticates in a browser with no software install. Aimed at regional staff and BYOD.
3. **Clientless ZTNA portal.** Announced April 2025 and included in the existing ZTNA license. Brokers **HTTP, HTTPS, RDP, SSH, and VNC** sessions inside an **isolated cloud container** — no agent, no enterprise browser, no plug-in. Ericsson's stated target for this is precisely "IoT/OT assets maintained by contractors and third parties." **Cloud-delivered only** (see Section 8).

### 4.2 Policy and verification

- Access is granted **per session**, evaluated against identity plus context — device attributes and location included — and re-evaluated when conditions change, with instant revocation.
- **Device posture visibility** covers antivirus state, OS version, and device type.
- **MFA is inherited from your IdP**, not implemented natively — which is the right answer, since it means your existing Entra/Okta MFA, conditional access, and joiner-mover-leaver process govern grid access automatically.
- One policy engine covers Secure Connect, SD-WAN, and ZTNA. No second rulebase to reconcile.
- **Licensing detail with an operational sting:** each authenticated user *acquires and reserves a ZTNA license for 90 days.* With high contractor churn you can consume licenses far faster than headcount suggests. Model this before buying.

### 4.3 How this replaces a VPN, concretely

| | GlobalProtect today | NCX ZTNA |
|---|---|---|
| Order of operations | Connect, then secure | Secure, then connect |
| What you join | The network | A specific resource, per session |
| Grant shape | Routes plus firewall rules, IP-shaped | Policy referencing identity and named resources |
| Undefined assets | Reachable if routed | Dark — not addressable |
| On-LAN users | Bypass the VPN entirely | Same client, same policy, same enforcement |
| Site reachability | Requires routing to site subnets | Site dials out; nothing routes inbound |
| Revocation | Session teardown / rule change | Continuous re-evaluation, instant revoke |

The in-office row is the important one. Because the NetCloud Client works on any network, the same policy applies whether the technician is in the operations center or in a bucket truck. That is what eliminates the two-trust-model problem, and it is the part your current architecture cannot fix without segmenting the corporate LAN itself.

---

## 5. The other two services (context, not core to this decision)

- **Advanced SD-WAN** (per device): DPI-based traffic classification into four priority classes, application-aware steering, real-time latency/loss measurement, intelligent link bonding with flow duplication, forward error correction, and 5G network-slice steering. Genuinely useful for dual-modem substation routers where SCADA must survive a degraded link — FEC and flow duplication are the relevant features.
- **Hybrid Mesh Firewall** (per device, Premium tier): application governance via DPI, IDS/IPS on both north/south and east/west traffic, and web filtering, with signatures curated for 5G/LTE environments. This is what would give you inspection *at the substation edge* rather than only at the Palo Alto perimeter.

---

## 6. Recommended target design

The critical modeling decision: **human access and machine access are different problems.** Do not force SCADA polling through a user-authentication product, and do not let machine-path policies quietly recreate implicit trust for humans.

**Path A — Machine to machine (SCADA master / ADMS / historian to RTUs, reclosers, IEDs)**
The data center or VPC side joins the Secure Connect fabric via the Service Gateway's `lan0` or a **NetCloud Virtual Edge** instance. Policy source = the specific SCADA host or its small subnet. Destination = the exact named resource and port at sites carrying the right tag. No user identity involved, because there is no user. This path is narrow, static, auditable, and always on.

**Path B — Human access (engineers, relay techs, field crews, NOC)**
NetCloud Client on every managed endpoint, always on, on-net and off-net alike. Policy source = SAML group attribute from Entra/Okta. Destination = named resources, scoped by site tag. Nothing addressable outside the grant. MFA and lifecycle inherited from the IdP.

**Path C — Vendors and third parties (OEM relay support, meter vendors, integrators)**
Clientless ZTNA portal, brokering RDP/SSH/HTTPS in an isolated container. The vendor's unmanaged laptop never touches your network and never receives an IP on it. Sessions are revocable instantly, which maps directly to vendor-remote-access control expectations. **Requires the cloud-delivered model.**

**And the step that makes it real:** once A/B/C are proven, **withdraw the corporate routes to the site subnets.** If the old L3 path survives, you have bought a second network, not a private one. This is the make-or-break item, and it is entirely on your side of the line — no product decision changes it.

**Keep Palo Alto** for the internet edge, corporate egress, data-center segmentation, and any non-grid remote access. NCX is not a corporate NGFW replacement, and framing it as one will produce a bad bake-off.

---

## 7. Reasons it might not work for you — the honest list

Ordered by how likely each is to kill or reshape the project.

1. **Cradlepoint-only edge.** Secure Connect requires an NCX-capable Cradlepoint router at every participating site. Sites on fiber with a third-party router, sites behind a partner-managed WAN, or serial-only gear with no IP router are **not covered** without adding hardware. Terminating a non-Cradlepoint IPsec peer on the Service Gateway is not a documented, supported design. **Inventory your sites by edge device before anything else.** Note also that Ericsson's own terms state not all NetCloud Edge routers are supported for Secure Connect — model-by-model verification is required, not brand-level.
2. **The corporate LAN itself is untouched.** NCX makes the *site* private. It does nothing about east/west movement inside the corporate network, and nothing about device-to-device traffic *within* a single site LAN — two IEDs on the same substation VLAN still talk freely. Intra-site segmentation still needs VLANs and local firewalling.
3. **Active/standby HA in a single data center.** The documentation is explicit that the standby gateway must sit in the same data center as the active. There is no documented geo-redundant gateway pair. For a utility where the access path to grid assets is operationally significant, that concentrates risk in one facility. Achieving site-level redundancy appears to mean a second, separately licensed network — get this priced and confirmed in writing.
4. **Two agents on one laptop.** Running the NetCloud Client alongside GlobalProtect invites split-tunnel and route-priority conflicts, doubles the endpoint support burden, and splits your access logs across two systems. Test this combination early; it is the most common practical failure in parallel-SASE deployments.
5. **You already own an identity-aware enforcement point.** Palo Alto with User-ID, always-on GlobalProtect including internal gateways, and site subnets placed behind the NGFW can also deliver "no implicit access from the corporate LAN" — with no new vendor, no new agent, and no new license stack. NCX wins on the parts Palo Alto handles poorly: outbound-dialed tunnels from CGNAT cellular sites without a private APN, overlapping site IP space, name-based resource definition, and zero-touch scaling to hundreds of small sites. **If the fleet is already Cradlepoint and already cellular, NCX is the stronger fit. If most grid sites are on managed fiber, the Palo Alto path is likely cheaper and less disruptive.** That inventory answer decides the whole evaluation.
6. **NCOS feature conflicts once a router becomes an NCX site.** Custom IPsec and GRE tunnels can no longer be added or modified — NCX owns them. CP Secure web filter and Umbrella/OpenDNS conflict with NCX's required DNS configuration. The router's primary DNS is forced to `100.127.255.254`, reachable only through the tunnel, with split DNS sending NetCloud Manager domains to `8.8.8.8` — an external resolver dependency that some OT security teams will object to on principle. Changing a LAN IP after a router is added as an NCX site breaks gateway-side NAT bonding. **Any existing site-to-SCADA IPsec must be migrated into NCX, not run alongside it.**
7. **IPv4 unicast only. No multicast.** If anything in your telemetry or protection scheme relies on multicast across the WAN, it will not traverse the fabric.
8. **Routing documentation is inconsistent.** The Administrator's Guide lists BGP as supported from NCOS 7.24.80; the NCX Technical FAQ states only static routing is supported and that policy routing steers traffic across the GRE/IPsec tunnels. Get a definitive answer from the SE before designing around dynamic routing.
9. **Published known issues.** The R980 achieves only 30–40 Mbps through Secure Connect versus roughly 200 Mbps on plain NCOS IPsec, with no workaround. Starlink WAN upload degrades roughly 50%. The gateway can falsely report "degraded" under heavy CPU load. Overlapping FQDN and subnet resources return wrong DNS answers for direct-internet rules. The clientless portal fails to open external links embedded in HTTP resources. The Linux client has restart and token-refresh race conditions.
10. **Per-router throughput ceilings.** Secure Connect throughput ranges roughly 10–720 Mbps depending on router model, and 10–400 Mbps with the Hybrid Mesh Firewall enabled, with 10–20 concurrent tunnels per router. Fine for SCADA and telemetry; verify against any video, LiDAR, or bulk-transfer workloads at sites.
11. **Licensing stack complexity.** Service Gateway capacity license, plus HA add-on, plus per-site Secure Connect, plus per-device SD-WAN, plus per-device HMF, plus per-user ZTNA. Five dimensions to model, and both the 10% throughput cliff and the 90-day ZTNA license reservation punish under-modeling.
12. **Logging and retention.** ZTNA logs "may be exported and stored in a location determined by the user." There is no built-in long-term retention. Plan the syslog/SIEM export path as part of the build, not after — especially if access records are ever audit evidence.
13. **Regulatory scope.** Distribution assets typically fall outside NERC CIP, which centers on the Bulk Electric System. **Confirm this with your compliance group anyway** — if any site in scope touches a BES Cyber System, Interactive Remote Access requirements (Intermediate System, MFA, encryption, session termination) apply and the design becomes an audit artifact, not just an engineering one. NCX ZTNA plus IdP MFA plausibly satisfies the intent, but that is a conversation to have before purchase, not after. IEC 62443 and NIST CSF framing will apply regardless.

---

## 8. Cloud-delivered vs customer-hosted — a decision you cannot defer

| | NetCloud SASE (cloud-delivered) | NetCloud Exchange (customer-hosted) |
|---|---|---|
| Gateway operated by | Ericsson | You |
| **Clientless ZTNA** | **Yes** | **No** |
| Firewall-as-a-Service | Yes | No |
| FIPS 140-3 | — | Yes, with certified hardware |
| Gateway capacity choice | Managed for you; 500 GB shared data pool for mobile/branch routers, unlimited for IoT | You size it, 250 Mbps – 4 Gbps |
| Data plane location | Ericsson cloud | Your DC or VPC |
| Best for | Lean IT teams, third-party access | Data-plane control, compliance posture |

**This is a genuine conflict for your use case.** The strongest argument for NCX in a utility is clientless, isolated third-party access for OEM and contractor support (Path C) — and that is cloud-delivered only. The strongest argument for customer-hosted is keeping the grid data plane and FIPS posture in your own facility. You will likely have to pick, run both models, or push the SE on a roadmap commitment for clientless on customer-hosted. **Raise this in the first vendor call.**

---

## 9. Questions for the Ericsson/Cradlepoint SE

1. Which exact router models in our fleet are Secure Connect and ZTNA capable, and what minimum NCOS does each need?
2. Is BGP supported over the NCX overlay today, or is it static routing only? The docs conflict.
3. Is there any supported path for geo-redundant Service Gateways, or is same-data-center active/standby the ceiling? What is the licensing impact of a second network for redundancy?
4. What is the roadmap for clientless ZTNA on customer-hosted deployments?
5. What exactly happens at the 10% throughput cliff — which traffic is dropped, is it signaled, and how is it alarmed in NCM?
6. Can the 90-day ZTNA license reservation be released early for terminated contractors?
7. What are the supported syslog/SIEM export formats and destinations for ZTNA authentication and policy-hit logs?
8. Is there any supported way to terminate a non-Cradlepoint IPsec peer on the Service Gateway for sites we cannot re-platform?
9. What is the interoperability guidance for the NetCloud Client coexisting with GlobalProtect on the same Windows endpoint?
10. Reference customers in electric distribution — specifically ones who removed the legacy L3 path rather than running NCX in parallel.

---

## 10. Suggested proof-of-concept

**Phase 0 — Inventory (do this first; it may end the evaluation).** Every grid site by edge device, WAN type, IP scheme, and what protocols actually cross the WAN. The percentage of sites already on NCX-capable Cradlepoint hardware is the single number that decides NCX versus tightening Palo Alto.

**Phase 1 — Gateway and two sites.** Stand up the Service Gateway (virtual, non-HA is fine for a PoC). Onboard two representative sites — ideally one cellular/CGNAT and one on the current fiber path. Confirm tunnel establishment, failover timing on dual-WAN, and end-to-end SCADA polling through a narrow Path A policy.

**Phase 2 — Identity.** Wire the SAML app to Entra/Okta. Deploy the NetCloud Client to a small engineering group. Build one Path B policy keyed to a group attribute. **Explicitly test the in-office case** — the whole point is that being on the corporate LAN grants nothing extra.

**Phase 3 — Negative testing.** From an authenticated client, attempt to reach a device at the site that is *not* a defined resource. It must be unreachable and, ideally, unresolvable. Repeat from an unauthenticated corporate host. This is the test that proves or disproves "private."

**Phase 4 — Third party.** If clientless is in scope, broker one vendor RDP/SSH session through the isolated portal and validate the revocation path.

**Phase 5 — Cutover proof.** On one pilot site only, remove the legacy corporate route and confirm operations are unaffected for a full monthly cycle including a planned maintenance window. Until this passes, no site is actually private.

---

## 11. Verdict

**NCX Secure Connect plus ZTNA is a legitimate fit for this problem**, and a better conceptual fit than extending GlobalProtect: it closes the in-office implicit-trust gap (the client enforces the same policy on-net), it handles CGNAT cellular sites and overlapping IP space without private APNs, it makes undefined gear unaddressable rather than merely firewalled, and it lets one policy engine express access in terms of AD groups and named grid resources instead of subnets.

**The two things most likely to sink it are not security features.** They are (a) how many of your sites are already on NCX-capable Cradlepoint hardware, and (b) whether the organization will actually retire the legacy routed path once the fabric is live. Answer (a) with an inventory this week; get (b) committed to in writing before signing anything.

**Do not scope this as an enterprise-wide GlobalProtect replacement.** Scope it as *the private access path to grid sites*, keep Palo Alto for the corporate perimeter, and judge NCX on that narrower job — where it is strong — rather than on a comparison it was never built to win.

---

## Appendix A — Terms

| Term | Meaning here |
|---|---|
| **Secure Connect** | The foundational zero-trust transport service. GRE-over-IPsec tunnels from site routers to the Service Gateway, deny-all by default. Prerequisite for every other NCX service. |
| **Service Gateway** | Data plane plus policy enforcement point. VM or SG4000 appliance. |
| **Resource** | An explicitly defined destination — a subnet, an IP, or an FQDN — behind a site router. Undefined equals dark. |
| **Site** | A router onboarded into the Secure Connect fabric. |
| **Tag** | A label on sites, used to write policy against groups of sites without listing them. |
| **Access policy** | Ordered first-match rule: source (site/tag/subnet/LAN/SAML attribute) to destination (site/tag/subnet/FQDN/resource/app). |
| **Split routing** | Selective steering — some traffic through the gateway, direct internet at the edge for the rest. |
| **Micro-tunnel** | Lighter-weight encryption mode as an alternative to full IPsec, for constrained endpoints. |
| **Name-based routing** | Referencing resources by name rather than IP; enables overlapping site address space. |
| **NetCloud Client** | The endpoint agent. Windows, macOS, Linux, iOS, Android. |
| **Clientless ZTNA** | Browser-brokered HTTP/HTTPS/RDP/SSH/VNC in an isolated cloud container. Cloud-delivered only. |
| **Virtual Edge** | Virtual router extending the fabric into AWS/Azure/VMware. |
| **HMF** | Hybrid Mesh Firewall — DPI application control, IDS/IPS, web filtering. Premium tier. |

## Appendix B — Key numbers

| Item | Value |
|---|---|
| Service Gateway VM sizing | 8 vCPU, 16 GB RAM, 16 GB disk, 2 NICs at 1 Gb/s or better |
| Gateway capacity tiers | 250 Mbps / 500 Mbps / 1 / 2 / 4 Gbps (250 Mbps virtual only) |
| Concurrent tunnels per gateway | 4,000 |
| Throughput overage behavior | 10% or more over purchased, excess traffic dropped |
| Per-router Secure Connect throughput | ~10–720 Mbps, model dependent |
| Per-router throughput with HMF | ~10–400 Mbps |
| Concurrent tunnels per router | 10–20 |
| Minimum NCOS — Secure Connect IPsec/GRE | 7.23.60 |
| Minimum NCOS — BGP (per Admin Guide) | 7.24.80 |
| Minimum NCOS — ZTNA | 7.23.31 |
| Minimum NCOS — Service Gateway (ESXi guide) | 7.23.80 |
| ZTNA license reservation | 90 days per authenticated user |
| DPD interval (idle router) | 30 s; tunnel teardown roughly 5 s after failed DPD |
| Policy entry limit | 20 sources / 20 destinations per policy |
| Traffic support | IPv4 unicast only — no multicast |
| IdP requirement | External SAML 2.0 (Okta, Entra ID, AD FS) |

## Appendix C — Sources

- [NetCloud Exchange Administrator's Guide — Overview](https://docs.cradlepoint.com/r/NetCloud-Exchange-Administrator-s-Guide/NetCloud-Exchange-Overview)
- [NetCloud Exchange Administrator's Guide — Architecture](https://docs.cradlepoint.com/r/NetCloud-Exchange-Administrator-s-Guide/NetCloud-Exchange-Architecture)
- [NetCloud Exchange Administrator's Guide — Service Gateway](https://docs.cradlepoint.com/r/NetCloud-Exchange-Administrator-s-Guide/NetCloud-Exchange-Service-Gateway)
- [Configuration Conflicts Between NetCloud OS and NetCloud Exchange](https://docs.cradlepoint.com/r/NetCloud-Exchange-Administrator-s-Guide/Configuration-Conflicts-Between-NetCloud-OS-and-NetCloud-Exchange)
- [NCX Technical FAQ — Tunneling](https://docs.cradlepoint.com/r/NCX-Technical-FAQ/Tunneling)
- [NCX Technical FAQ — DNS and Routing](https://docs.cradlepoint.com/r/NCX-Technical-FAQ/DNS-and-Routing)
- [NCX Technical FAQ — Administration and Design](https://docs.cradlepoint.com/r/NCX-Technical-FAQ/Administration-and-Design)
- [Configuring NCX Secure Connect — Using Access Policies](https://docs.cradlepoint.com/r/Configuring-NetCloud-Exchange-Secure-Connect/Using-Access-Policies)
- [Configuring NCX Secure Connect — Creating a Group](https://docs.cradlepoint.com/r/Configuring-NetCloud-Exchange-Secure-Connect/Creating-a-Group)
- [Configuring NCX ZTNA — Introduction](https://docs.cradlepoint.com/r/Configuring-NetCloud-Exchange-Zero-Trust-Network-Access/Zero-Trust-Network-Access-Introduction)
- [Configuring NCX ZTNA — Requirements](https://docs.cradlepoint.com/r/Configuring-NetCloud-Exchange-Zero-Trust-Network-Access/Zero-Trust-Network-Access-Requirements)
- [Configuring NCX ZTNA — Authenticating Users](https://docs.cradlepoint.com/r/Configuring-NetCloud-Exchange-Zero-Trust-Network-Access/Authenticating-Users)
- [Service Gateway Deployment Guide (VMware ESXi) — Requirements](https://docs.cradlepoint.com/r/NetCloud-Exchange-Service-Gateway-Deployment-Guide-VMware-ESXi/Requirements)
- [NetCloud SASE Release Notes — Known Issues](https://docs.cradlepoint.com/r/NetCloud-SASE-Release-Notes/Known-Issues)
- [Ericsson NetCloud SASE and NetCloud Exchange datasheet](https://cradlepoint.com/datasheet/netcloud-sase/)
- [Ericsson NetCloud Service Gateway product page](https://cradlepoint.ericsson.com/products/endpoints/netcloud-service-gateway/)
- [Ericsson NetCloud ZTNA product page](https://cradlepoint.ericsson.com/products/netcloud/ztna/)
- [Ericsson Secure Connect product page](https://cradlepoint.ericsson.com/products/netcloud/secure-connect/)
- [NetCloud Virtual Edge product page](https://cradlepoint.com/product/endpoints/netcloud-exchange-virtual-edge/)
- [NetCloud Exchange Terms and Conditions](https://cradlepoint.com/legal/end-user-agreement-0724/netcloud-exchange-terms-and-conditions/)
- [Ericsson press release — clientless ZTNA for wireless WAN (April 2025)](https://www.ericsson.com/en/news/2025/4/ericsson-boosts-netcloud-sase-with-industrys-first-fully-integrated-clientless-ztna-solution-for-wireless-wan)
- [Cradlepoint blog — Introducing NetCloud ZTNA for secure, clientless remote access](https://cradlepoint.com/resources/blog/introducing-netcloud-ztna-for-secure-clientless-remote-access/)
