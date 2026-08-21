# Ericsson Cradlepoint NetCloud Exchange (NCX) Service Gateway
## Evaluation for Private, Zero-Trust Access to Distribution Grid Sites

| | |
|---|---|
| **Revision** | B |
| **Prepared** | 2026-08-20 |
| **Owner** | Network engineering |
| **Classification** | Internal |
| **Reviewers still required** | OT security, compliance (impact classification and CIP applicability), the team that owns the site switches |
| **Next review** | On receipt of the site inventory, or when the medium impact plan is defined, whichever comes first |
| **Sources retrieved** | 2026-08-20, see Appendix C |

**Revision history.** Rev A, first issue. Rev B, added Section 7 on containment and Section 9 on NERC CIP, corrected the regulatory framing from "distribution is out of scope" to the actual Distribution Provider applicability, and expanded Phase 0 of the proof of concept.

**Question being answered:** Can the NCX Service Gateway (Secure Connect plus ZTNA) replace the current model, which is open access to grid site gear from the corporate LAN and GlobalProtect/Palo Alto for everyone else, with a single identity-based private access path?

**Short answer:** Yes, for the grid sites themselves, *provided* every site rides a Cradlepoint NCX-capable router and you actually decommission the existing routed path from the corporate network to the site subnets. NCX does not add privacy on top of a flat network. It replaces the transport, and the flat path has to go away for the security model to mean anything. It is not a general-purpose replacement for GlobalProtect for the rest of the enterprise, and there are real constraints (single-vendor edge, active/standby HA in one data center only, licensed-throughput hard caps, IPv4 unicast only) documented below.

**One thing it does not do at all:** NCX contains movement *between* sites and between the corporate network and a site. It does nothing about movement *inside* a site. If the requirement is blast radius containment after an endpoint at a site is compromised, read Section 7 before anything else, because that part is local segmentation work and no NCX licence changes it.

---

## 1. The current problem, stated precisely

| Access path today | Authentication | Authorization | Effective blast radius |
|---|---|---|---|
| In office to site gear | None (implicit network trust) | Routing table | Any host on the corporate LAN can reach any device at any site |
| Remote, GlobalProtect, Palo Alto, site gear | Yes (GP plus IdP) | Firewall rules, largely IP and subnet based | Once on the VPN, reachability is network-shaped, not user-shaped |
| Device to device inside one site | None | None | Every device at a site can reach every other device at that site |

Two different trust models for the same assets, and the weaker one (in-office) is the one that governs the majority of daily access. The third row is not an access path anyone uses on purpose. It is the path an attacker uses. The goal, "make site access private," is really four goals:

1. **Remove implicit trust from the corporate LAN.** Being plugged in should grant nothing.
2. **Make access identity-based and per-resource,** not per-subnet, regardless of where the user sits.
3. **Keep the operational path for machine-to-machine SCADA traffic working** without wrapping it in a user-authentication model that does not fit it.
4. **Contain a compromised device at a site** so it cannot reach the rest of that site, the other sites, or the corporate network.

NCX addresses 1, 2 and 3, with different mechanisms for 3 than for 1 and 2. That distinction is the single most important design point in this document, and it is covered in Section 6. NCX addresses only part of 4, which is Section 7.

---

## 2. What NCX actually is

NetCloud Exchange is Ericsson and Cradlepoint's SASE architecture, sold in two delivery models:

- **NetCloud Exchange (customer-hosted).** You run the Service Gateway VM or appliance in your own data center or VPC.
- **NetCloud SASE (cloud-delivered).** Ericsson runs the gateway and you consume it as a service.

Both are managed from **NetCloud Manager (NCM)**, the same cloud console you already use for router management. The two models are *not* feature-identical. See Section 10.

### 2.1 Components

| Component | What it is | Role in your solution |
|---|---|---|
| **NetCloud Manager (NCM)** | Cloud control plane, single pane of glass | Defines sites, resources, users, and policy. Never carries data traffic. |
| **NCX Service Gateway** | Virtual appliance (VMware, KVM, AWS, Azure) or SG4000 hardware appliance | The **data plane and policy enforcement point**. Terminates every site tunnel, terminates every ZTNA client, decides what may talk to what. |
| **NCX-enabled routers** | Cradlepoint NCOS routers at each grid site | The spoke. Dials out to the Service Gateway, presents its LAN as defined resources. |
| **NetCloud Client** | Agent for Windows, macOS, Linux, iOS, Android | The GlobalProtect replacement for user access. Authenticates via SAML, joins the Secure Connect network. |
| **NCX Portal / Clientless ZTNA portal** | Browser-based access | Access without an agent, including full application isolation for third parties. |
| **NetCloud Virtual Edge** | Lightweight virtual router for AWS, Azure and VMware | Extends the Secure Connect fabric to where your applications live (historian, ADMS, jump hosts) so the app side is policy-governed too. |

### 2.2 Service Gateway specifications (as published)

| | Virtual appliance | SG4000 hardware |
|---|---|---|
| Licensed capacity | 250 Mbps to 4 Gbps | 500 Mbps to 4 Gbps |
| Platform | AWS, Azure, KVM, VMware ESXi 6.7 U3 and later | 1RU rack mount, Intel 13th-gen i7-13700E, 64 GB DDR5, 2x 10GbE SFP+ |
| VM sizing | 8 vCPU, 16 GB RAM (32 GB in some cloud SKUs), 16 GB disk, 2x 1 Gb/s or better NICs | Not applicable |
| Concurrent tunnels | Up to 4,000 | Up to 4,000 |
| HA | Active/standby | Active/standby |
| FIPS 140-3 | Available on customer-hosted with certified hardware | Yes |

Three interfaces are required for full function:

- **mgmt0**, SSH and local UI
- **lan0**, reaches internal resources and DNS servers
- **wan0**, tunnel termination, and the gateway's own path back to NetCloud Manager

**Hard limit worth reading twice:** per Ericsson's NCX terms, if actual throughput exceeds purchased throughput by 10% or more, *the excess traffic is not transmitted*. This is not a soft overage bill, it is traffic loss. Size the gateway for peak, including storm and restoration events when every site is chattering at once.

---

## 3. Secure Connect, the transport layer

Secure Connect is the foundation service. ZTNA, SD-WAN, and the Hybrid Mesh Firewall are all add-ons that require it plus a Service Gateway.

**How it works.** Each site router builds an outbound tunnel to the Service Gateway using GRE over IPsec, IKEv2, pre-shared key, with liveness checks and DPD driving failover. One tunnel is built per active WAN interface, so a dual-modem router at a substation maintains two, with priority set by Connection Manager. From NCOS 7.23.10 onward the IKE ID is the GRE key rather than the router's WAN IP, which is what makes this work cleanly behind carrier NAT.

**Why that matters for grid sites:** the site dials out. There is no inbound listener, no static IP requirement, no public IP on the router, and no private APN needed. Cellular sites behind CGNAT work without carrier involvement. Ericsson positions Secure Connect explicitly as a legacy VPN and private-APN replacement.

**Security properties, as designed:**

- **Deny-all by default.** Outbound internet from a site is permitted by default. **Site-to-site and site-to-resource traffic is blocked until a policy explicitly allows it.** East and west are off unless you turn them on. This is the property that does most of the containment work in Section 7.
- **Resource obscuring, dark assets.** Devices are reachable only if defined as a resource and referenced in a policy. Undefined gear on a site LAN is not addressable across the fabric at all.
- **Name-based (domain-based) routing.** Resources are referenced by name, not IP. This hides addressing and, importantly for utilities, supports **overlapping IP ranges across sites**, which is very common when hundreds of sites were built from the same cookie-cutter template.
- **Split routing.** Direct internet access at the edge for what should not cross the fabric, with selective traffic steered to the gateway.
- **Encryption choice.** IPsec at three cipher levels, or a lighter micro-tunnel mode for constrained IoT endpoints.

### 3.1 The access policy model

This is where the private part is actually enforced.

- **Sources:** site name, tag, IP subnet, a LAN defined in the router config, or **SAML attributes** (requires the ZTNA licence).
- **Destinations:** site name, tag, subnet, FQDN (wildcards supported), named resource, and, with the Hybrid Mesh Firewall licence, web category and reputation, application and app category, and geolocation.
- **Logic:** criteria within a source are ANDed, destinations allow OR. Empty criteria means *any*.
- **Evaluation:** strict top-down, first match wins, no further evaluation.
- **Ceiling:** 20 source entries and 20 destination entries per policy.

The SAML-attribute-as-source capability is the hinge of the whole design. It lets one policy engine express *"members of the AD group `Grid-Relay-Techs` may reach resource `RECLOSER-HMI` at sites tagged `District-4`, on TCP/443 only"* with no reference to a subnet anywhere.

**One caveat that the reference design depends on.** The published documentation describes wildcards for **FQDN** destinations. It does not clearly extend them to named resources. If a destination cannot be written as `*-RELAY-A`, every policy has to enumerate resources site by site and the 20-entry ceiling arrives almost immediately. This is question 3 in Section 11 and it is more consequential than it looks.

---

## 4. ZTNA, the GlobalProtect replacement

NCX ZTNA is licensed **per user** and requires Secure Connect and a Service Gateway already in place, NCOS 7.23.31 or newer on the gateway and participating endpoints, and an **external SAML 2.0 identity provider** (Okta, Microsoft Entra ID, and AD FS are the named ones).

### 4.1 Three ways a user gets in

1. **NetCloud Client (agent).** Windows, macOS, Linux, iOS, Android. Works from anywhere, including the corporate LAN, home, or a truck. The user authenticates against your IdP. On success they receive a token that permits them to join the Secure Connect network. Policy then decides which resources resolve and respond.
2. **NCX Portal (browser, on-net).** A user sitting behind a Cradlepoint router authenticates in a browser with no software install. Aimed at regional staff and BYOD.
3. **Clientless ZTNA portal.** Announced April 2025 and included in the existing ZTNA licence. Brokers **HTTP, HTTPS, RDP, SSH, and VNC** sessions inside an **isolated cloud container**, with no agent, no enterprise browser, and no plug-in. Ericsson's stated target for this is precisely "IoT/OT assets maintained by contractors and third parties." **Cloud-delivered only**, see Section 10.

### 4.2 Policy and verification

- Access is granted **per session**, evaluated against identity plus context including device attributes and location, and re-evaluated when conditions change, with instant revocation.
- **Device posture visibility** covers antivirus state, OS version, and device type.
- **MFA is inherited from your IdP**, not implemented natively, which is the right answer. It means your existing Entra or Okta MFA, conditional access, and joiner-mover-leaver process govern grid access automatically.
- One policy engine covers Secure Connect, SD-WAN, and ZTNA. No second rulebase to reconcile.
- **Licensing detail with an operational sting:** each authenticated user *acquires and reserves a ZTNA licence for 90 days.* With high contractor churn you can consume licences far faster than headcount suggests. Model this before buying.

### 4.3 How this replaces a VPN, concretely

| | GlobalProtect today | NCX ZTNA |
|---|---|---|
| Order of operations | Connect, then secure | Secure, then connect |
| What you join | The network | A specific resource, per session |
| Grant shape | Routes plus firewall rules, IP-shaped | Policy referencing identity and named resources |
| Undefined assets | Reachable if routed | Dark, not addressable |
| On-LAN users | Bypass the VPN entirely | Same client, same policy, same enforcement |
| Site reachability | Requires routing to site subnets | Site dials out, nothing routes inbound |
| Revocation | Session teardown or rule change | Continuous re-evaluation, instant revoke |

The in-office row is the important one. Because the NetCloud Client works on any network, the same policy applies whether the technician is in the operations center or in a bucket truck. That is what eliminates the two-trust-model problem, and it is the part your current architecture cannot fix without segmenting the corporate LAN itself.

---

## 5. The other two services (context, not core to this decision)

- **Advanced SD-WAN** (per device): DPI-based traffic classification into four priority classes, application-aware steering, real-time latency and loss measurement, intelligent link bonding with flow duplication, forward error correction, and 5G network-slice steering. Genuinely useful for dual-modem substation routers where SCADA must survive a degraded link. FEC and flow duplication are the relevant features.
- **Hybrid Mesh Firewall** (per device, Premium tier): application governance via DPI, IDS and IPS on both north/south and east/west traffic, and web filtering, with signatures curated for 5G and LTE environments. This is what would give you inspection *at the substation edge* rather than only at the Palo Alto perimeter. Note the limit described in Section 7: east-west inspection only sees traffic that traverses the router, so it cannot see two devices talking on the same LAN segment.

---

## 6. Recommended target design

The critical modeling decision: **human access and machine access are different problems.** Do not force SCADA polling through a user-authentication product, and do not let machine-path policies quietly recreate implicit trust for humans.

**Path A, machine to machine** (SCADA master, ADMS or historian to RTUs, reclosers, IEDs). The data center or VPC side joins the Secure Connect fabric via the Service Gateway's `lan0` or a **NetCloud Virtual Edge** instance. Policy source is the specific SCADA host or its small subnet. Destination is the exact named resource and port at sites carrying the right tag. No user identity is involved, because there is no user. This path is narrow, static, auditable, and always on. Give the historian its own rule: it is a second collector with a different source address, and it is easy to forget until collection silently stops.

**Path B, human access** (engineers, relay techs, field crews, NOC). NetCloud Client on every managed endpoint, always on, on-net and off-net alike. Policy source is a SAML group attribute from Entra or Okta. Destination is named resources, scoped by site tag. Nothing is addressable outside the grant. MFA and lifecycle are inherited from the IdP.

**Path C, vendors and third parties** (OEM relay support, meter vendors, integrators). Clientless ZTNA portal, brokering RDP, SSH and HTTPS in an isolated container. The vendor's unmanaged laptop never touches your network and never receives an IP on it. Sessions are revocable instantly, which maps directly onto vendor remote access control expectations. **Requires the cloud-delivered model**, which conflicts with an on-premises data plane. If the design settles on customer-hosted, Path C becomes a jump host inside the site application zone, reached with the NetCloud Client, and the isolation argument for NCX weakens considerably.

**And the step that makes it real:** once A, B and C are proven, **withdraw the corporate routes to the site subnets.** If the old L3 path survives, you have bought a second network, not a private one. This is the make-or-break item, and it is entirely on your side of the line. No product decision changes it.

**Keep Palo Alto** for the internet edge, corporate egress, data-center segmentation, and any non-grid remote access. NCX is not a corporate NGFW replacement, and framing it as one will produce a bad bake-off.

---

## 7. Containment inside a site

This section exists because "stop a compromised endpoint spreading" is the stated driver, and it is four separate problems rather than one. NCX answers three of them and does nothing at all about the fourth.

| Spread path | Contained by NCX | Mechanism | Depends on |
|---|---|---|---|
| Site A device to Site B device | **Yes, by default** | Secure Connect denies site to site until a policy allows it | Nothing. This is the shipped default. |
| Site device into corporate or the data center | **Yes** | Deny-all access policy, resources are one-directional grants | Withdrawing the legacy corporate route |
| Corporate device into a site | **Yes** | Undefined gear is not addressable and does not resolve | The same route withdrawal |
| Device to device **inside one site** | **No. Nothing.** | None. This traffic never leaves the site LAN and the gateway never sees it. | Local segmentation |

The first row is the strongest single thing NCX does for this requirement and it costs nothing extra. The last row is the reason the site detail sheet in the reference design is drawn as three LAN zones rather than one flat subnet. Before zoning, a camera on the substation LAN had an unfiltered layer 2 path to two protective relays. NCX makes that camera dark to the outside world and does nothing whatsoever about the path to the relays.

### 7.1 An ordering that puts the switch dependency last

Site switches are not currently under our administrative control. That constrains tier 3 below and nothing else, so most of the containment value can land while the access conversation is still in progress.

| Tier | What | Needs | Closes |
|---|---|---|---|
| 0 | NCX deny-all between sites, plus withdraw the legacy corporate routes | Nothing new | Cross-site spread, site to corporate, corporate to site |
| 1 | OT and non-OT on separate router LANs and physical ports, zone firewall default deny between them | Re-cabling and NCOS config, no switch administration | A corporate or contractor device at a site pivoting onto control gear |
| 2 | Hypervisor: separate port groups per zone, no bridged path between the OT virtual network and anything else | Already ours | VM to VM movement inside the virtualization host |
| 3 | Per-function VLANs, port isolation for cameras and unidentified gear, ACLs between OT function groups | Administrative access to the site switches | Movement inside a single zone |

NCOS supports multiple LANs, binding them to specific Ethernet ports, and a zone firewall where filter policies are one-way and applied per zone-forwarding pair. Two things to confirm on the deployed NCOS version before designing around tier 1: a LAN has to be placed in its own zone rather than sharing the primary LAN zone, and the stock forwarding policies are permissive in the LAN-to-WAN direction, so they get replaced rather than inherited.

### 7.2 What zoning still does not fix

Devices sharing a zone reach each other at layer 2, and that traffic never reaches the router. Neither the zone firewall nor the Hybrid Mesh Firewall east-west inspection can see it, because inspection only applies to traffic that traverses the router. Zoning contains movement between function groups, not inside one. Until tier 3 is available, the residue inside a zone is covered by host controls and physical access control, and that should be stated rather than assumed. If a vendor presents east-west IDS or IPS as the answer to lateral movement inside a substation, that is the question to put to them.

### 7.3 Put the boundary on the router, not the switch

Whatever device enforces the boundary is the device that inherits the scrutiny that comes with the role, and later the compliance requirement load if a site is ever categorized above low impact. If a managed switch enforces zone separation through VLANs, that switch is in scope. If the Cradlepoint is the single access point between zones and out to the fabric, the count stays at one box per site, on hardware already managed through NetCloud Manager. Tiers 0 to 2 build that shape deliberately. Tier 3 then adds depth inside a zone without moving the boundary.

---

## 8. Reasons it might not work for you, the honest list

Ordered by how likely each is to kill or reshape the project.

1. **Cradlepoint-only edge.** Secure Connect requires an NCX-capable Cradlepoint router at every participating site. Sites on fiber with a third-party router, sites behind a partner-managed WAN, or serial-only gear with no IP router are **not covered** without adding hardware. Terminating a non-Cradlepoint IPsec peer on the Service Gateway is not a documented, supported design. **Inventory your sites by edge device before anything else.** Note also that Ericsson's own terms state not all NetCloud Edge routers are supported for Secure Connect, so verification is model by model, not brand level.
2. **The corporate LAN itself is untouched, and so is the inside of each site.** NCX makes the *site* private. It does nothing about east and west movement inside the corporate network, and nothing about device-to-device traffic *within* a single site LAN. Two IEDs on the same substation VLAN still talk freely. Section 7 is the whole answer to this and most of it is local work.
3. **Active/standby HA in a single data center.** The documentation is explicit that the standby gateway must sit in the same data center as the active. There is no documented geo-redundant gateway pair. For a utility where the access path to grid assets is operationally significant, that concentrates risk in one facility. Achieving site-level redundancy appears to mean a second, separately licensed network. Get this priced and confirmed in writing.
4. **Two agents on one laptop.** Running the NetCloud Client alongside GlobalProtect invites split-tunnel and route-priority conflicts, doubles the endpoint support burden, and splits your access logs across two systems. Test this combination early. It is the most common practical failure in parallel SASE deployments.
5. **You already own an identity-aware enforcement point.** Palo Alto with User-ID, always-on GlobalProtect including internal gateways, and site subnets placed behind the NGFW can also deliver "no implicit access from the corporate LAN," with no new vendor, no new agent, and no new licence stack. NCX wins on the parts Palo Alto handles poorly: outbound-dialed tunnels from CGNAT cellular sites without a private APN, overlapping site IP space, name-based resource definition, and zero-touch scaling to hundreds of small sites. **If the fleet is already Cradlepoint and already cellular, NCX is the stronger fit. If most grid sites are on managed fiber, the Palo Alto path is likely cheaper and less disruptive.** That inventory answer decides the whole evaluation.
6. **NCOS feature conflicts once a router becomes an NCX site.** Custom IPsec and GRE tunnels can no longer be added or modified, because NCX owns them. CP Secure web filter and Umbrella or OpenDNS conflict with NCX's required DNS configuration. The router's primary DNS is forced to `100.127.255.254`, reachable only through the tunnel, with split DNS sending NetCloud Manager domains to `8.8.8.8`, an external resolver dependency that some OT security teams will object to and that becomes a compliance conversation at any site categorized above low impact. Changing a LAN IP after a router is added as an NCX site breaks gateway-side NAT bonding. **Any existing site-to-SCADA IPsec must be migrated into NCX, not run alongside it.** Confirm separately whether onboarding also constrains local multi-LAN and zone firewall configuration, because Section 7 tier 1 depends on it not doing so.
7. **Wildcards on named resources are not clearly supported.** The scaling story for policy depends on writing `*-RTU` rather than listing sites. Published documentation describes wildcards for FQDN destinations only. If they do not apply to resource names, the 20-entry ceiling per policy binds quickly.
8. **IPv4 unicast only. No multicast.** If anything in your telemetry or protection scheme relies on multicast across the WAN, it will not traverse the fabric.
9. **Routing documentation is inconsistent.** The Administrator's Guide lists BGP as supported from NCOS 7.24.80. The NCX Technical FAQ states only static routing is supported and that policy routing steers traffic across the GRE and IPsec tunnels. Get a definitive answer from the SE before designing around dynamic routing.
10. **Published known issues.** The R980 achieves only 30 to 40 Mbps through Secure Connect versus roughly 200 Mbps on plain NCOS IPsec, with no workaround. Starlink WAN upload degrades roughly 50%. The gateway can falsely report degraded under heavy CPU load. Overlapping FQDN and subnet resources return wrong DNS answers for direct-internet rules. The clientless portal fails to open external links embedded in HTTP resources. The Linux client has restart and token-refresh race conditions.
11. **Per-router throughput ceilings.** Secure Connect throughput ranges roughly 10 to 720 Mbps depending on router model, and 10 to 400 Mbps with the Hybrid Mesh Firewall enabled, with 10 to 20 concurrent tunnels per router. Fine for SCADA and telemetry. Verify against any video, LiDAR, or bulk-transfer workloads at sites.
12. **Licensing stack complexity.** Service Gateway capacity licence, plus HA add-on, plus per-site Secure Connect, plus per-device SD-WAN, plus per-device HMF, plus per-user ZTNA. Five dimensions to model, and both the 10% throughput cliff and the 90-day ZTNA licence reservation punish under-modeling.
13. **Logging and retention.** ZTNA logs "may be exported and stored in a location determined by the user." There is no built-in long-term retention. Plan the syslog and SIEM export path as part of the build, not after. At medium impact CIP-007 R4.3 puts a number on it, 90 days.
14. **Regulatory scope.** Covered in Section 9. The short version is that "distribution is out of scope for NERC CIP" is not a safe default, the current fleet is registered low impact, and CIP-003-9 vendor remote access obligations are already enforceable.

---

## 9. Where NERC CIP touches this

Every site in the current fleet is a **low impact** BES Cyber System. Medium impact sites are planned, but whether that means reclassifying existing sites or taking on net new ones is not yet decided, and the answer changes the design. This section states what is true now and marks the rest as open.

### 9.1 Distribution is not automatically out of scope

A Distribution Provider is a registered function and is in scope for specific systems under CIP-002 Applicability 4.2.1: UFLS and UVLS programs meeting the criteria, each RAS subject to a NERC or Regional standard, Protection Systems applying to Transmission, and Cranking Path and blackstart elements. The correct framing for a distribution grid access design is not "out of scope," it is "in scope at any site carrying one of those." The site inventory therefore needs an impact classification column supplied by the compliance group, not an assumption supplied by network engineering.

### 9.2 Low impact and medium impact are different standards

| | Low impact, the fleet today | Medium impact, planned but not scoped |
|---|---|---|
| Governing standard | CIP-003 R2 Attachment 1 | CIP-005, plus the rest of the medium impact set |
| ESP required | **No** | **Yes**, for every applicable Cyber Asset connected by a routable protocol |
| Electronic access controls | Attachment 1 Section 3. At assets with external routable connectivity, permit only necessary inbound and outbound access, authenticate remote users, protect authentication in transit. | ESP with identified Electronic Access Points, inbound and outbound permissions documented by reason, everything else denied |
| Remote access for people | Covered by Section 3. No Intermediate System requirement. | Interactive Remote Access must terminate at an **Intermediate System**, be encrypted, and use multi-factor, where the site has external routable connectivity |
| Vendor remote access | Attachment 1 Section 6. Determine active vendor remote access, be able to disable it, detect malicious inbound and outbound communication tied to it. **Enforceable since 1 April 2026.** | CIP-005 R3, determine and disable |
| Log retention | Not specified at this level | CIP-007 R4.3, **90 days** |

Two consequences worth stating plainly. First, the lack of switch access does not block compliance today, because nothing at low impact requires an ESP or internal segmentation. Second, an ESP is a logical boundary, not a VLAN. Separate router LANs on separate physical ports with an enforced default deny satisfy the concept without a managed switch, which is why Section 7 recommends putting the boundary on the router.

### 9.3 Low impact obligations mapped to this design

| Obligation | What this design provides | Evidence still owed |
|---|---|---|
| Section 3, necessary inbound and outbound only | Deny-all by default with an ordered, first-match policy. Undefined gear is not addressable and does not resolve. | An exported policy set showing the reason for each rule, and evidence the legacy corporate route is actually withdrawn. Until it is, the flat path is the real access control and the policy set is decoration. |
| Section 3, authenticate remote users | SAML to Entra ID or Okta. MFA, conditional access and the leaver process are inherited rather than reimplemented. | IdP configuration, group-to-policy mapping, and evidence that removal from a group actually removes access |
| Section 6, determine active vendor remote access | Vendor sessions are brokered and per-session, so they are enumerable rather than inferred from firewall logs | A named report or console view, and a named person who checks it. If the design settles on customer-hosted there is no clientless portal, so this needs a different answer. |
| Section 6, disable vendor remote access | Per-session evaluation with instant revocation, and group removal at the IdP as the second lever | A written procedure with an owner and a target time, and one tested revocation per year. Model the 90-day ZTNA licence reservation against contractor churn at the same time. |
| Section 6, detect malicious vendor communication | **Not provided by default.** Hybrid Mesh Firewall adds IDS and IPS at extra licence cost, and only for traffic that crosses the router. | The detection method and the SIEM export path. ZTNA logs have no built-in retention, so the export is part of the build. |

### 9.4 What changes if medium impact arrives

- **CIP-005 R1.1 requires an ESP** for every applicable Cyber Asset connected by a routable protocol, at medium impact regardless of external routable connectivity. A medium impact system that is serial only, with no routable connection, does not pull an ESP, which in a substation usually helps for a subset of the relays and no more.
- **Anything inside the ESP that is not part of the highest impact BES Cyber System becomes a Protected Cyber Asset** and inherits most of the medium impact requirement load, including patching, ports and services, malware prevention, logging, baseline configuration and change management, and personnel access management. Keeping corporate and non-OT devices out of the OT zone is therefore scope control as much as it is security. This is the argument to take to whoever owns the switches, because it is a budget argument rather than a security argument.
- **The Service Gateway performs electronic access control**, and at medium impact the ZTNA broker also sits in the Interactive Remote Access path as an Intermediate System. Placing those functions in a vendor cloud is the open question NERC has not resolved and is actively drafting. Customer-hosted avoids it and cannot do clientless vendor access. The medium impact plan therefore decides Section 10, not the other way round.
- **CIP-010 R1 baseline and change management** sits awkwardly with NCX owning the tunnel, DNS and routing configuration on an onboarded router, because part of the configuration evidence would live in a vendor cloud console.
- **CIP-015-1 internal network security monitoring** became effective 2 September 2025, with high impact and medium impact with external routable connectivity due 1 October 2028 and the remainder due 1 October 2030. CIP-015-2 extends the scope toward EACMS, PACS and shared infrastructure. This turns the intra-site visibility gap in Section 7.2 from an architectural note into a dated obligation.
- **CIP-013 supply chain risk management** applies to procuring this, and adding a vendor cloud to the access path is a CIP-013 R1 item.
- **The virtualization CIP package** introduces Shared Cyber Infrastructure as a category distinct from a BES Cyber System, which changes how a site virtualization host is categorized. Effective dating runs on a long implementation clock. Confirm it with compliance rather than assuming.

**None of the containment work in Section 7 waits on any of this.** Segmentation for blast radius is a security control we own outright, and building it at the router now is also the cheapest path to an ESP later.

---

## 10. Cloud-delivered versus customer-hosted, a decision you cannot defer

| | NetCloud SASE (cloud-delivered) | NetCloud Exchange (customer-hosted) |
|---|---|---|
| Gateway operated by | Ericsson | You |
| **Clientless ZTNA** | **Yes** | **No** |
| Firewall-as-a-Service | Yes | No |
| FIPS 140-3 | Not available | Yes, with certified hardware |
| Gateway capacity choice | Managed for you, 500 GB shared data pool for mobile and branch routers, unlimited for IoT | You size it, 250 Mbps to 4 Gbps |
| Data plane location | Ericsson cloud | Your DC or VPC |
| Best for | Lean IT teams, third-party access | Data-plane control, compliance posture |

**This is a genuine conflict for this use case.** The strongest argument for NCX in a utility is clientless, isolated third-party access for OEM and contractor support (Path C), and that is cloud-delivered only. The strongest argument for customer-hosted is keeping the grid data plane and FIPS posture in your own facility, and that argument gets much stronger the moment any site is categorized above low impact, for the reasons in Section 9.4. You will likely have to pick, run both models, or push the SE on a roadmap commitment for clientless on customer-hosted. **Raise this in the first vendor call.**

---

## 11. Questions for the Ericsson and Cradlepoint SE

1. Which exact router models in our fleet are Secure Connect and ZTNA capable, and what minimum NCOS does each need?
2. Once a router is onboarded as an NCX site, does NCX also constrain local multi-LAN and zone firewall configuration, the way it already owns tunnels, DNS and routing? Our intra-site segmentation plan depends on the answer.
3. Do destination wildcards apply to named resources, or only to FQDNs? If only FQDNs, how is a policy meant to scale past 20 destinations?
4. Is BGP supported over the NCX overlay today, or is it static routing only? The docs conflict.
5. Is there any supported path for geo-redundant Service Gateways, or is same-data-center active/standby the ceiling? What is the licensing impact of a second network for redundancy?
6. What is the roadmap for clientless ZTNA on customer-hosted deployments?
7. What exactly happens at the 10% throughput cliff: which traffic is dropped, is it signaled, and how is it alarmed in NCM?
8. Can the 90-day ZTNA licence reservation be released early for terminated contractors?
9. What are the supported syslog and SIEM export formats and destinations for ZTNA authentication and policy-hit logs, and what is the shortest path to 90 days of retained records?
10. Is there any supported way to terminate a non-Cradlepoint IPsec peer on the Service Gateway for sites we cannot re-platform?
11. What is the interoperability guidance for the NetCloud Client coexisting with GlobalProtect on the same Windows endpoint?
12. Reference customers in electric distribution, specifically ones who removed the legacy L3 path rather than running NCX in parallel.
13. For customers with NERC CIP obligations, what artifacts does Ericsson provide for a cloud-delivered deployment, and are there reference customers using the cloud-delivered model for anything categorized above low impact?

---

## 12. Suggested proof of concept

**Phase 0, inventory. Do this first, it may end the evaluation.** Every grid site by edge device, WAN type, IP scheme, and what protocols actually cross the WAN. Add two columns that the first revision of this document did not have:

- **Impact classification**, supplied by compliance, not inferred.
- **Who administers the site switch**, because that decides how far tier 3 in Section 7.1 can go and how long the access request will take.

The percentage of sites already on NCX-capable Cradlepoint hardware is the single number that decides NCX versus tightening Palo Alto.

**Phase 1, gateway and two sites.** Stand up the Service Gateway (virtual, non-HA is fine for a PoC). Onboard two representative sites, ideally one cellular behind CGNAT and one on the current fiber path. Confirm tunnel establishment, failover timing on dual-WAN, and end-to-end SCADA polling through a narrow Path A policy. Confirm the historian collects, not just the ADMS, because they are two different sources and need two rules.

**Phase 2, identity.** Wire the SAML app to Entra or Okta. Deploy the NetCloud Client to a small engineering group. Build one Path B policy keyed to a group attribute. **Explicitly test the in-office case**, because the whole point is that being on the corporate LAN grants nothing extra.

**Phase 3, negative testing.** From an authenticated client, attempt to reach a device at the site that is *not* a defined resource. It must be unreachable and, ideally, unresolvable. Repeat from an unauthenticated corporate host. This is the test that proves or disproves "private."

**Phase 4, containment testing.** Separate from Phase 3, and the one that answers the actual driver. Put a host in the auxiliary zone at the pilot site and attempt to reach the control zone. It must fail at the router. Then put a host in the control zone and attempt to reach a second device in the same zone. **It will succeed**, and that result should be recorded rather than quietly skipped, because it is the residue that only managed switch access can close.

**Phase 5, third party.** If clientless is in scope, broker one vendor RDP or SSH session through the isolated portal and validate the revocation path. Time the revocation and write the number down, because Section 6 of CIP-003 Attachment 1 asks for the capability and an auditor will ask how long it takes.

**Phase 6, cutover proof.** On one pilot site only, remove the legacy corporate route and confirm operations are unaffected for a full monthly cycle including a planned maintenance window. Until this passes, no site is actually private.

---

## 13. Verdict

**NCX Secure Connect plus ZTNA is a legitimate fit for this problem**, and a better conceptual fit than extending GlobalProtect. It closes the in-office implicit-trust gap because the client enforces the same policy on-net, it handles CGNAT cellular sites and overlapping IP space without private APNs, it makes undefined gear unaddressable rather than merely firewalled, and it lets one policy engine express access in terms of AD groups and named grid resources instead of subnets. On the containment requirement specifically, it closes three of the four spread paths and the cross-site one is free.

**The two things most likely to sink it are not security features.** They are how many sites are already on NCX-capable Cradlepoint hardware, and whether the organization will actually retire the legacy routed path once the fabric is live. Answer the first with an inventory this week. Get the second committed in writing before signing anything.

**Do not expect it to segment the inside of a site.** That is local work, it is mostly available without switch access, and it should proceed on its own schedule regardless of how the NCX decision lands.

**Do not scope this as an enterprise-wide GlobalProtect replacement.** Scope it as *the private access path to grid sites*, keep Palo Alto for the corporate perimeter, and judge NCX on that narrower job, where it is strong, rather than on a comparison it was never built to win.

---

## Appendix A, terms

### NCX vocabulary

| Term | Meaning here |
|---|---|
| **Secure Connect** | The foundational zero-trust transport service. GRE-over-IPsec tunnels from site routers to the Service Gateway, deny-all by default. Prerequisite for every other NCX service. |
| **Service Gateway** | Data plane plus policy enforcement point. VM or SG4000 appliance. |
| **Resource** | An explicitly defined destination, a subnet, an IP, or an FQDN, behind a site router. Undefined equals dark. |
| **Site** | A router onboarded into the Secure Connect fabric. |
| **Tag** | A label on sites, used to write policy against groups of sites without listing them. |
| **Access policy** | Ordered first-match rule. Source (site, tag, subnet, LAN or SAML attribute) to destination (site, tag, subnet, FQDN, resource or app). |
| **Split routing** | Selective steering. Some traffic through the gateway, direct internet at the edge for the rest. |
| **Micro-tunnel** | Lighter-weight encryption mode as an alternative to full IPsec, for constrained endpoints. |
| **Name-based routing** | Referencing resources by name rather than IP. Enables overlapping site address space. |
| **NetCloud Client** | The endpoint agent. Windows, macOS, Linux, iOS, Android. |
| **Clientless ZTNA** | Browser-brokered HTTP, HTTPS, RDP, SSH and VNC in an isolated cloud container. Cloud-delivered only. |
| **Virtual Edge** | Virtual router extending the fabric into AWS, Azure or VMware. |
| **HMF** | Hybrid Mesh Firewall. DPI application control, IDS and IPS, web filtering. Premium tier. |
| **Zone firewall** | Local NCOS feature, unrelated to NCX. One-way filter policies applied per zone-forwarding pair between router LANs. |

### Grid and compliance vocabulary

| Term | Meaning here |
|---|---|
| **ADMS** | Advanced Distribution Management System. The operational application that supervises and controls distribution assets. |
| **RTU / IED** | Remote Terminal Unit and Intelligent Electronic Device. The field controllers and protective relays at a site. |
| **DNP3** | The SCADA protocol most of this telemetry runs on, typically TCP 20000. |
| **BES** | Bulk Electric System. The scope NERC CIP is written around. |
| **BES Cyber System** | The categorized unit CIP requirements attach to, rated high, medium or low impact. |
| **ESP** | Electronic Security Perimeter. A logical border around applicable Cyber Assets connected by a routable protocol. Required at medium and high impact, not at low. Not a synonym for VLAN. |
| **EAP** | Electronic Access Point. The identified interface where routable communication crosses an ESP. |
| **PCA** | Protected Cyber Asset. Anything inside an ESP on a routable connection that is not part of the highest impact system there. Inherits most requirements, which is why non-OT devices must stay out. |
| **EACMS** | Electronic Access Control or Monitoring System. Whatever enforces or monitors access at the boundary, including Intermediate Systems. |
| **IRA** | Interactive Remote Access. User-initiated remote access to an applicable system. At medium and high impact it must terminate at an Intermediate System. |
| **Intermediate System** | The broker an IRA session must terminate at, so the initiating device never reaches the applicable Cyber Asset directly. |
| **INSM** | Internal Network Security Monitoring, the subject of CIP-015. |
| **CGNAT** | Carrier-grade NAT. Why cellular sites have no reachable public address, and why outbound-dialed tunnels matter. |
| **DPD** | Dead Peer Detection. The liveness check that drives tunnel failover. |

---

## Appendix B, key numbers

| Item | Value |
|---|---|
| Service Gateway VM sizing | 8 vCPU, 16 GB RAM, 16 GB disk, 2 NICs at 1 Gb/s or better |
| Gateway capacity tiers | 250 Mbps, 500 Mbps, 1, 2, 4 Gbps (250 Mbps virtual only) |
| Concurrent tunnels per gateway | 4,000 |
| Throughput overage behavior | 10% or more over purchased, excess traffic dropped |
| Per-router Secure Connect throughput | Roughly 10 to 720 Mbps, model dependent |
| Per-router throughput with HMF | Roughly 10 to 400 Mbps |
| Concurrent tunnels per router | 10 to 20 |
| Minimum NCOS, Secure Connect IPsec and GRE | 7.23.60 |
| Minimum NCOS, BGP (per Admin Guide) | 7.24.80 |
| Minimum NCOS, ZTNA | 7.23.31 |
| Minimum NCOS, Service Gateway (ESXi guide) | 7.23.80 |
| ZTNA licence reservation | 90 days per authenticated user |
| DPD interval (idle router) | 30 s, tunnel teardown roughly 5 s after failed DPD |
| Policy entry limit | 20 sources, 20 destinations per policy |
| Traffic support | IPv4 unicast only, no multicast |
| IdP requirement | External SAML 2.0 (Okta, Entra ID, AD FS) |
| CIP-003-9 vendor remote access | Enforceable 1 April 2026 |
| CIP-007 R4.3 log retention (medium and high) | 90 days |
| CIP-015-1 INSM deadlines | 1 October 2028 (high, and medium with ERC), 1 October 2030 (remainder) |

Per-router throughput figures and concurrent tunnel counts are drawn from Ericsson product pages and the Administrator's Guide and vary by model. Treat them as indicative until the SE confirms per model.

---

## Appendix C, sources

All URLs retrieved 2026-08-20 and confirmed reachable on that date.

### Ericsson and Cradlepoint

- [NetCloud Exchange Administrator's Guide, Overview](https://docs.cradlepoint.com/r/NetCloud-Exchange-Administrator-s-Guide/NetCloud-Exchange-Overview)
- [NetCloud Exchange Administrator's Guide, Architecture](https://docs.cradlepoint.com/r/NetCloud-Exchange-Administrator-s-Guide/NetCloud-Exchange-Architecture)
- [NetCloud Exchange Administrator's Guide, Service Gateway](https://docs.cradlepoint.com/r/NetCloud-Exchange-Administrator-s-Guide/NetCloud-Exchange-Service-Gateway)
- [Configuration Conflicts Between NetCloud OS and NetCloud Exchange](https://docs.cradlepoint.com/r/NetCloud-Exchange-Administrator-s-Guide/Configuration-Conflicts-Between-NetCloud-OS-and-NetCloud-Exchange)
- [NCX Technical FAQ, Tunneling](https://docs.cradlepoint.com/r/NCX-Technical-FAQ/Tunneling)
- [NCX Technical FAQ, DNS and Routing](https://docs.cradlepoint.com/r/NCX-Technical-FAQ/DNS-and-Routing)
- [NCX Technical FAQ, Administration and Design](https://docs.cradlepoint.com/r/NCX-Technical-FAQ/Administration-and-Design)
- [Configuring NCX Secure Connect, Using Access Policies](https://docs.cradlepoint.com/r/Configuring-NetCloud-Exchange-Secure-Connect/Using-Access-Policies)
- [Configuring NCX Secure Connect, Creating a Group](https://docs.cradlepoint.com/r/Configuring-NetCloud-Exchange-Secure-Connect/Creating-a-Group)
- [Configuring NCX ZTNA, Introduction](https://docs.cradlepoint.com/r/Configuring-NetCloud-Exchange-Zero-Trust-Network-Access/Zero-Trust-Network-Access-Introduction)
- [Configuring NCX ZTNA, Requirements](https://docs.cradlepoint.com/r/Configuring-NetCloud-Exchange-Zero-Trust-Network-Access/Zero-Trust-Network-Access-Requirements)
- [Configuring NCX ZTNA, Authenticating Users](https://docs.cradlepoint.com/r/Configuring-NetCloud-Exchange-Zero-Trust-Network-Access/Authenticating-Users)
- [Service Gateway Deployment Guide (VMware ESXi), Requirements](https://docs.cradlepoint.com/r/NetCloud-Exchange-Service-Gateway-Deployment-Guide-VMware-ESXi/Requirements)
- [NetCloud SASE Release Notes, Known Issues](https://docs.cradlepoint.com/r/NetCloud-SASE-Release-Notes/Known-Issues)
- [NCOS, Configuring a Zone Firewall, Creating Filter Policies](https://docs.cradlepoint.com/r/Configuring-a-Zone-Firewall/Creating-Filter-Policies)
- [NCOS, Enabling Multiple LANs](https://customer.cradlepoint.com/s/article/NCOS-Enabling-Multiple-LANs)
- [NCOS, Change the LAN/WAN Ethernet Port Mode](https://docs.cradlepoint.com/r/NCOS-Change-the-LAN-WAN-Ethernet-Port-Mode-of-a-Cradlepoint-with-Multiple-Ethernet-Ports/Overview-of-Changing-Ports)
- [Ericsson NetCloud SASE and NetCloud Exchange datasheet](https://cradlepoint.com/datasheet/netcloud-sase/)
- [Ericsson NetCloud Service Gateway product page](https://cradlepoint.ericsson.com/products/endpoints/netcloud-service-gateway/)
- [Ericsson NetCloud ZTNA product page](https://cradlepoint.ericsson.com/products/netcloud/ztna/)
- [Ericsson Secure Connect product page](https://cradlepoint.ericsson.com/products/netcloud/secure-connect/)
- [NetCloud Virtual Edge product page](https://cradlepoint.com/product/endpoints/netcloud-exchange-virtual-edge/)
- [NetCloud Exchange Terms and Conditions](https://cradlepoint.com/legal/end-user-agreement-0724/netcloud-exchange-terms-and-conditions/)
- [Ericsson press release, clientless ZTNA for wireless WAN (April 2025)](https://www.ericsson.com/en/news/2025/4/ericsson-boosts-netcloud-sase-with-industrys-first-fully-integrated-clientless-ztna-solution-for-wireless-wan)
- [Cradlepoint blog, Introducing NetCloud ZTNA for secure, clientless remote access](https://cradlepoint.com/resources/blog/introducing-netcloud-ztna-for-secure-clientless-remote-access/)

### NERC CIP

- [CIP-002-5.1a, BES Cyber System Categorization](https://www.nerc.com/globalassets/standards/reliability-standards/cip/cip-002-5.1a.pdf)
- [CIP-002-5.1 Standard Application Guide](https://www.nerc.com/globalassets/programs/compliance/compliance-guidance/implementation/cip-002-5.1_standard_application_guide.pdf)
- [CIP-005-7, Electronic Security Perimeters](https://www.nerc.com/globalassets/standards/reliability-standards/cip/cip-005-7.pdf)
- [CIP-015-1, Internal Network Security Monitoring](https://www.nerc.com/globalassets/standards/reliability-standards/cip/cip-015-1.pdf)
- [FERC order approving CIP-015-1, Federal Register, 2 July 2025](https://www.federalregister.gov/documents/2025/07/02/2025-12309/critical-infrastructure-protection-reliability-standard-cip-015-1-cyber-security-internal-network)
- [Technical Rationale for CIP-003-9](https://www.nerc.com/globalassets/standards/projects/2023-04/cip-003-a-technical-rationale-1.4_102423.pdf)
- [MRO, Planning for CIP-003-9 and Vendor Electronic Remote Access](https://www.mro.net/planning-for-cip-003-9-and-vendor-electronic-remote-access/)
- [NERCipedia, Protected Cyber Assets definition](https://nercipedia.com/glossary/protected-cyber-assets/)
- [Dragos, CIP-015-2 EACMS, PACS and SCI monitoring](https://www.dragos.com/blog/nerc-cip-015-eacms-pacs-sci-monitoring-explained)
