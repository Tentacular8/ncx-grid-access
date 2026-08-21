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
   authentication. The office is the weak trust model, and it governs most daily access.
2. NCX inverts that: site modems dial outbound to a Service Gateway, nothing routes
   inbound, and every device is unreachable until it's explicitly defined and allowed.
3. Access becomes a property of *who you are*, an AD group claim through our existing
   identity provider, not *where you are plugged in*, so the rule is identical in the
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
- **We are not replacing the security stack.** This is one narrow job, the access path to
  grid sites. The Palo Alto investment is untouched and stays where it's strong.
- **Some of it pays for itself.** Static/public IP plans are a recurring per-SIM charge.
  Retiring them offsets part of the licensing. *Get this number before the meeting, it is
  the most persuasive line item you have.*
- **It scales without headcount.** Sites are onboarded by tag, and policy is written
  against groups of sites rather than lists of addresses. Site 400 costs the same effort
  as site 4.
- **The alternative isn't "do nothing."** It's tightening Palo Alto and segmenting the
  corporate LAN, which is also real work. The honest comparison is between two projects,
  not between a project and the status quo.

### Security and compliance

- **Deny-all by default, with dark assets.** Undefined equipment isn't firewalled off,
  it is unaddressable, with no DNS answer and no route. That's a materially different posture from
  a permit-list on a perimeter device.
- **Identity is inherited, not reinvented.** SAML 2.0 to our existing IdP means MFA,
  conditional access, and the joiner-mover-leaver process govern grid access automatically.
  No second identity store to audit.
- **Every session is attributable.** Today a shared local admin password on a device with a
  public IP produces no usable audit trail. Afterward, every session maps to a person or a
  named host.
- **Third-party access gets a real answer.** Isolated, agentless, brokered sessions for OEM
  and contractor support, revocable instantly, instead of handing out credentials.
- **Be straight about the gaps.** It does nothing for east and west movement inside the
  corporate LAN or inside a site LAN, and log retention is our problem to build. Say this
  before security finds it, because the credibility is worth more than the point. Have the
  answer ready: Section 7 of the evaluation sets out the four spread paths, which three NCX
  closes, and a four tier plan for the fourth that does not wait on switch access.

### Operations, SCADA, and field crews

- **Machine paths stay machine paths.** SCADA polling is a host-to-resource policy with no
  user login in front of it. Nothing about ADMS-to-RTU communication becomes interactive.
- **Failover is seconds, not minutes.** Dead-peer detection moves traffic to the second
  carrier in roughly five seconds. Confirm poll timers tolerate that gap, and say so out
  loud, because this is the question operations will actually care about.
- **The workflow gets simpler, not harder.** No VPN to remember to connect to; the client
  is always on and the experience is the same everywhere.
- **Commissioning changes.** A new device at a site is unreachable until someone defines it.
  That is by design. This goes on the commissioning checklist or crews will report the fabric as
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
| "We already own Palo Alto, use what we have." | True, and it can deliver no-implicit-trust with User-ID and internal gateways. It handles CGNAT cellular sites, overlapping site subnets, and per-site onboarding at scale poorly. | The site inventory. If most sites are cellular Cradlepoint, NCX wins on merit. If most are managed fiber, Palo Alto is genuinely the cheaper path, and we should say so. |
| "Another vendor, another agent, another console." | Fair. It is a second stack, scoped to one job. | Confirming the NetCloud Client coexists cleanly with GlobalProtect, and that logs export to our SIEM so there's one place to look. |
| "OT can't depend on a cloud service." | NetCloud Manager is cloud and there's no on-prem option. But it is the control plane. It pushes config and policy and carries none of the traffic. The data plane can sit in our data center. | Choosing customer-hosted, and testing what actually happens to established sessions when NCM is unreachable. |
| "What happens when the gateway dies?" | Active/standby only, and the standby must sit in the *same data center*. This is the weakest part of the architecture for us. | A written answer from Ericsson on geo-redundancy, with the licensing cost of a second network. Do not hand-wave this one. |
| "We'll lose access when we need it most." | Storm restoration is exactly when the identity provider being unreachable or the gateway being saturated would hurt. | A tested break-glass path, and sizing the gateway for a restoration event rather than an average Tuesday. Traffic above 10% over the licensed tier is dropped, not billed. |
| "This is a big change for field crews." | The client is less work than a VPN. Commissioning is more work, because new devices must be defined. | Putting resource definition on the commissioning checklist and piloting with the crews who'll be loudest. |
| "Can't we just leave the public IPs?" | Yes, NCX works either way. But if the modem still answers a login prompt from the internet, we secured the gear behind the modem and not the modem. | Separating the two: keep the IP if it's useful for carrier diagnostics, disable or ACL the WAN-side admin. |

---

## 4. Questions for Ericsson and Cradlepoint

The questions live in [`SE-questions-checklist.md`](SE-questions-checklist.md), which is the
canonical list. It is a working copy with space to record what was actually said, grouped by
what each question blocks, and every question carries a good answer and a bad answer,
because that framing is what keeps the conversation from ending in a reassuring non-answer.

Take that file into the call. **New questions go there, not here.** What follows is only the
shape of it, so this document reads end to end without switching files. If the two ever
disagree, the checklist is right.

| Group | Covers | Why it is in this position |
|---|---|---|
| A, ask first | Supported router models, non-Cradlepoint IPsec peers | Either answer can end the evaluation before money is spent |
| B, design shape | Zone firewall config after onboarding, resource wildcards, admin sessions over the tunnel, BGP, multicast, geo-redundancy and its cost, sessions when NetCloud Manager is unreachable, clientless on customer-hosted | These decide what we build, not merely how we run it |
| C, operational | Throughput cliff, gateway upgrades, log export and retention, agent coexistence, per-model throughput, policy rollback | Needed before the build rather than before the purchase |
| D, compliance | CIP artifacts for cloud-delivered deployments | Bring compliance to this part |
| E, commercial | Full SKU list, Remote Connect, the 90 day ZTNA reservation, year four renewal | Five licensing dimensions, and under-modeling is punished twice |
| F, proof | References who removed the legacy path, a PoC on two real sites | The reference who completed the withdrawal is worth more than any feature answer |

Three of them decide the shape of this design rather than merely informing it, and they are
the ones to protect if the call runs short:

1. **Which exact models in our fleet are Secure Connect and ZTNA capable.** The single
   number that decides NCX versus tightening Palo Alto.
2. **Whether onboarding a router as an NCX site constrains local zone firewall
   configuration.** The intra-site containment plan puts the boundary on the router. If NCX
   owns that configuration the way it already owns tunnels, DNS and routing, the plan needs
   a different mechanism at every site.
3. **Whether same-data-center active/standby is the ceiling, and what a second network
   costs.** The weakest part of the architecture for us, and the one most likely to get a
   reassuring non-answer.

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
6. **Which sites sit at which impact level, and what is the plan for medium?** The
   current fleet is registered low impact, so CIP-003 governs, no Electronic Security
   Perimeter is required, and the vendor remote access obligations in Attachment 1 Section 6
   have been enforceable since 1 April 2026. "Distribution is out of scope" is not a safe
   default: a Distribution Provider is in scope for UFLS and UVLS programs meeting the
   criteria, RAS subject to a standard, Transmission Protection Systems, and blackstart
   cranking path elements. Medium impact sites are planned. Whether that means reclassifying
   existing sites or taking on new ones changes the design, because medium impact requires an
   ESP and pushes the hosting decision toward customer-hosted. Section 9 of the evaluation
   has the detail. This is a compliance answer, not an engineering one.
7. **Who administers the switch at each site?** We do not have that access today. It does
   not block anything at low impact, and it does not block the first three tiers of the
   containment plan, but it is the long pole on the fourth and the request should be in
   flight rather than waiting on a trigger.
8. **Who owns policy changes after go-live?** Network, security, or OT. Unowned policy is
   how deny-all quietly becomes allow-all over two years.
9. **What actually depends on the flat path that nobody has documented?** Nobody knows.
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
- Intra-site zoning is proven at the pilot site: a host in the auxiliary zone cannot reach
  the control zone, and the intra-zone residue is recorded rather than quietly skipped

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
| Weeks 3 and 4 | Gateway plus two sites, machine path only | Prove transport before involving people |
| Weeks 5 and 6 | Identity wired up, small pilot group, in-office test | The in-office case is the whole point |
| Week 6 | Negative testing | Try to reach the undefined device. This is the test that proves "private" |
| Week 6 | Containment testing | Try to cross zones inside the pilot site. This is the test that proves "contained", and it is a different test |
| Week 7+ | One site cut over, legacy route withdrawn, full month | The only test that counts |

---

*Prepared from the NCX evaluation and reference design in this repository. Figures and
constraints cited here come from published Ericsson Cradlepoint documentation as of
August 2026 and should be re-confirmed with an SE before commitment.*
