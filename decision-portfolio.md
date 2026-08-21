# NCX Decision Portfolio

Talking points, objection handling, and the questions that actually move the decision.
Built from the [evaluation](NCX-Service-Gateway-Site-Access-Evaluation.md) and the
[reference design](ncx-grid-access-design.html).

A note on how this is organized: a decision is *solid* when you know what would change
your mind, not when you've collected enough reasons to say yes. Sections 5 and 6 are
therefore the load-bearing ones. Everything before them is how you communicate the
decision; those two are how you know it's right.

---

## 1. The case in five sentences

Use this when someone asks "what are we doing and why" in a hallway.

1. Today, anyone on the corporate LAN can reach any device at any grid site with no
   authentication — the office is the weak trust model, and it governs most daily access.
2. NCX inverts that: site modems dial outbound to a Service Gateway, nothing routes
   inbound, and every device is unreachable until it's explicitly defined and allowed.
3. Access becomes a property of *who you are* — an AD group claim through our existing
   identity provider — not *where you're plugged in*, so the rule is identical in the
   office, at home, and in a truck.
4. It also removes the public IPs from the modems, which today present an internet-facing
   login page at every site.
5. It's scoped to grid site access only; Palo Alto keeps the corporate perimeter.

---

## 2. Talking points by audience

### Leadership and budget

- **The exposure is asymmetric to the cost.** Every grid site currently answers a login
  prompt from the open internet, and every corporate host can reach every site device.
  The remediation is a licensing change and a routing change, not a rebuild.
- **We are not replacing the security stack.** This is one narrow job — the access path to
  grid sites. The Palo Alto investment is untouched and stays where it's strong.
- **Some of it pays for itself.** Static/public IP plans are a recurring per-SIM charge.
  Retiring them offsets part of the licensing. *Get this number before the meeting — it's
  the most persuasive line item you have.*
- **It scales without headcount.** Sites are onboarded by tag, and policy is written
  against groups of sites rather than lists of addresses. Site 400 costs the same effort
  as site 4.
- **The alternative isn't "do nothing."** It's tightening Palo Alto and segmenting the
  corporate LAN, which is also real work. The honest comparison is between two projects,
  not between a project and the status quo.

### Security and compliance

- **Deny-all by default, with dark assets.** Undefined equipment isn't firewalled off,
  it's unaddressable — no DNS answer, no route. That's a materially different posture from
  a permit-list on a perimeter device.
- **Identity is inherited, not reinvented.** SAML 2.0 to our existing IdP means MFA,
  conditional access, and the joiner-mover-leaver process govern grid access automatically.
  No second identity store to audit.
- **Every session is attributable.** Today a shared local admin password on a device with a
  public IP produces no usable audit trail. Afterward, every session maps to a person or a
  named host.
- **Third-party access gets a real answer.** Isolated, agentless, brokered sessions for OEM
  and contractor support, revocable instantly — instead of handing out credentials.
- **Be straight about the gaps.** It does nothing for east/west movement inside the
  corporate LAN or inside a site LAN, and log retention is our problem to build. Say this
  before security finds it; the credibility is worth more than the point.

### Operations, SCADA, and field crews

- **Machine paths stay machine paths.** SCADA polling is a host-to-resource policy with no
  user login in front of it. Nothing about ADMS-to-RTU communication becomes interactive.
- **Failover is seconds, not minutes.** Dead-peer detection moves traffic to the second
  carrier in roughly five seconds. Confirm poll timers tolerate that gap — and say so out
  loud, because this is the question operations will actually care about.
- **The workflow gets simpler, not harder.** No VPN to remember to connect to; the client
  is always on and the experience is the same everywhere.
- **Commissioning changes.** A new device at a site is unreachable until someone defines it
  — by design. This goes on the commissioning checklist or crews will report the fabric as
  broken. Raise it early; it's the most likely source of early friction.
- **Break-glass is a real question.** Today the public IP *is* the recovery path for a
  half-broken router. We need a documented out-of-band answer before we retire it.

### The network team who owns Palo Alto

- **This is not a bake-off.** NCX wins on the things Palo Alto handles poorly at this
  scale: outbound-dialed tunnels from CGNAT cellular sites with no private APN, hundreds of
  sites sharing the same `192.168.1.0/24`, and name-based resource definition.
- **Palo Alto keeps the internet edge, corporate egress, and DC segmentation.** Framing
  this as a replacement will produce a fight the project doesn't need.
- **The endpoint is the real integration risk.** Two agents on one laptop is where parallel
  SASE deployments actually break. Test the NetCloud Client alongside GlobalProtect in week
  one, not at pilot exit.

---

## 3. Objections you will get, and honest answers

| Objection | The honest answer | What makes it go away |
|---|---|---|
| "We already own Palo Alto — use what we have." | True, and it can deliver no-implicit-trust with User-ID and internal gateways. It handles CGNAT cellular sites, overlapping site subnets, and per-site onboarding at scale poorly. | The site inventory. If most sites are cellular Cradlepoint, NCX wins on merit. If most are managed fiber, Palo Alto is genuinely the cheaper path — and we should say so. |
| "Another vendor, another agent, another console." | Fair. It is a second stack, scoped to one job. | Confirming the NetCloud Client coexists cleanly with GlobalProtect, and that logs export to our SIEM so there's one place to look. |
| "OT can't depend on a cloud service." | NetCloud Manager is cloud and there's no on-prem option. But it's the control plane — it pushes config and policy and carries none of the traffic. The data plane can sit in our data center. | Choosing customer-hosted, and testing what actually happens to established sessions when NCM is unreachable. |
| "What happens when the gateway dies?" | Active/standby only, and the standby must sit in the *same data center*. This is the weakest part of the architecture for us. | A written answer from Ericsson on geo-redundancy, with the licensing cost of a second network. Do not hand-wave this one. |
| "We'll lose access when we need it most." | Storm restoration is exactly when the identity provider being unreachable or the gateway being saturated would hurt. | A tested break-glass path, and sizing the gateway for a restoration event rather than an average Tuesday — remember traffic above 10% over the licensed tier is dropped, not billed. |
| "This is a big change for field crews." | The client is less work than a VPN. Commissioning is more work, because new devices must be defined. | Putting resource definition on the commissioning checklist and piloting with the crews who'll be loudest. |
| "Can't we just leave the public IPs?" | Yes — NCX works either way. But if the modem still answers a login prompt from the internet, we secured the gear behind the modem and not the modem. | Separating the two: keep the IP if it's useful for carrier diagnostics, disable or ACL the WAN-side admin. |

---

## 4. Questions for Ericsson / Cradlepoint

Grouped by what they decide. The "good answer / bad answer" column is the point — it keeps
the conversation from ending in a reassuring non-answer.

### Architecture and fit

| # | Question | Good answer | Bad answer |
|---|---|---|---|
| 1 | Which exact models in our fleet support Secure Connect and ZTNA, and what minimum NCOS does each need? | A model-by-model list against our serial numbers | "Most current models" |
| 2 | Will NCOS accept an admin session to the router's LAN IP when the traffic arrives over the NCX tunnel rather than the physical LAN port? | A definitive yes with a config reference | "You'd want Remote Connect for that" without explaining why |
| 3 | Is BGP supported over the NCX overlay today? The Administrator's Guide says yes from 7.24.80; the Technical FAQ says static only. | Acknowledges the doc conflict and names the current state | Reads the Administrator's Guide back to us |
| 4 | Is there any supported way to terminate a non-Cradlepoint IPsec peer on the Service Gateway? | A clear no, with the Virtual Edge alternative explained | A vague "we can look into it" |
| 5 | Multicast: any roadmap, or is IPv4 unicast permanent? | A dated roadmap position | "What do you need it for?" |

### Resilience — the section that matters most

| # | Question | Good answer | Bad answer |
|---|---|---|---|
| 6 | Is same-data-center active/standby the ceiling, or is there a supported geo-redundant option? | A named design, even if it's "two networks" | "You can put the standby anywhere" — this contradicts the documentation, so push back |
| 7 | If we build a second gateway in a second data center for site redundancy, what does that cost in licensing, and can a site fail between them automatically? | A concrete SKU list and a failover description | A quote with no failover explanation |
| 8 | What happens to established sessions when NetCloud Manager is unreachable? | Sessions persist, new auth fails, with a stated duration | "That doesn't happen" |
| 9 | Exactly what happens at the 10% throughput cliff — which traffic is dropped, is it signaled, how is it alarmed in NCM? | A described drop policy and an alert we can subscribe to | "You'd just buy more capacity" |
| 10 | What is the upgrade process for the gateway, and what is the maintenance window? | A documented procedure with rollback | "It's seamless" |

### Commercial

| # | Question | Good answer | Bad answer |
|---|---|---|---|
| 11 | Full SKU list for our shape: gateway capacity, HA add-on, per-site Secure Connect, per-user ZTNA, and anything else required. | An itemized quote naming every dimension | A single blended number |
| 12 | Is Remote Connect included, and if not, what does it cost across the fleet? | A clear line item | Silence — we already know it's separate |
| 13 | Can the 90-day ZTNA license reservation be released early for a terminated contractor? | A described process | "It expires eventually" |
| 14 | What does the renewal look like in year four, and what has price movement been on this product line? | Honest numbers | Deflection to "we can discuss at renewal" |

### Operations

| # | Question | Good answer | Bad answer |
|---|---|---|---|
| 15 | Supported syslog/SIEM export format and destinations for authentication and policy-hit logs? | A named format and a working example | "Logs can be exported" |
| 16 | Interoperability guidance for the NetCloud Client alongside GlobalProtect on Windows? | A tested statement or a known-issues note | "Should be fine" |
| 17 | What's the actual Secure Connect throughput on the specific models we run? | Per-model figures, including the weak ones | A single headline number |
| 18 | How do we roll back a policy change that breaks a site? | Versioning or export/restore | "You just change it back" |

### Proof

| # | Question |
|---|---|
| 19 | Reference customers in electric distribution who **removed** the legacy L3 path, not ones running NCX in parallel. Can we talk to them directly? |
| 20 | Will you support a paid or loaned PoC on two of our real sites, with an SE on the cutover call? |

---

## 5. Questions we have to answer ourselves

The vendor cannot answer these, and the decision isn't solid until we have.

1. **What share of grid sites are already on NCX-capable Cradlepoint hardware?** This
   single number decides NCX versus tightening Palo Alto. Nothing else in this portfolio
   matters more.
2. **Will the organization actually withdraw the corporate routes to site subnets?** If
   that commitment isn't in writing from whoever owns routing, we are buying a second
   network and calling it private.
3. **What is the current monthly spend on static/public IP plans?** Funds part of the
   licensing and is the strongest budget argument available.
4. **What does an ADMS poll timeout tolerate?** Decides whether a five-second failover is a
   non-event or an operational objection.
5. **What is the break-glass path for a router that's up but has no tunnel?** Today it's the
   public IP. If we retire that, what replaces it, and has anyone tested it?
6. **Are any sites in NERC CIP scope?** Distribution usually isn't, but confirm with
   compliance rather than assuming. It changes the design from an engineering artifact into
   an audit artifact.
7. **Who owns policy changes after go-live?** Network, security, or OT. Unowned policy is
   how deny-all quietly becomes allow-all over two years.
8. **What actually depends on the flat path that nobody has documented?** Nobody knows.
   This is what the single-site cutover is for.

---

## 6. Go and kill criteria

Write these down *before* the vendor meeting, so the decision is measured against
something rather than against enthusiasm.

**Proceed if all of these hold**

- A clear majority of grid sites are on NCX-capable Cradlepoint hardware, or the cost of
  getting them there is already budgeted
- Route withdrawal is committed in writing by the routing owner
- Ericsson gives a workable answer on gateway redundancy, or we accept the single-DC risk
  with our eyes open and it's documented as an accepted risk
- The NetCloud Client and GlobalProtect coexist cleanly on a real corporate laptop image
- Logs export to our SIEM in a format we can actually use
- A single-site cutover survives a full month including a planned outage and an
  after-hours callout

**Stop, or re-scope to the Palo Alto path, if any of these are true**

- Most grid sites aren't Cradlepoint and re-platforming isn't funded
- Nobody will commit to withdrawing the legacy routes
- The endpoint agents conflict and there's no supported workaround
- Total licensing exceeds the cost of tightening Palo Alto and segmenting the LAN, with no
  capability we actually need to justify the difference
- Operations rejects the failover behavior after seeing it in the pilot

---

## 7. Sequence

| When | What | Why it's in this order |
|---|---|---|
| Week 1 | Site inventory by edge device and WAN type | May end the evaluation before any money is spent |
| Week 1 | Pull the static-IP monthly spend | Needed for the budget conversation, not after it |
| Week 2 | Vendor call using Section 4 | Go in with the inventory already in hand |
| Week 2 | Test the two agents on one laptop | Cheapest way to find the most common failure |
| Week 3–4 | Gateway plus two sites, machine path only | Prove transport before involving people |
| Week 5–6 | Identity wired up, small pilot group, in-office test | The in-office case is the whole point |
| Week 6 | Negative testing | Try to reach the undefined device. This is the test that proves "private" |
| Week 7+ | One site cut over, legacy route withdrawn, full month | The only test that counts |

---

*Prepared from the NCX evaluation and reference design in this repository. Figures and
constraints cited here come from published Ericsson Cradlepoint documentation as of
August 2026 and should be re-confirmed with an SE before commitment.*
