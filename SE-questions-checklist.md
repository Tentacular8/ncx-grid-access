# SE questions, working checklist

Every question for the Ericsson and Cradlepoint SE, in one place, in the order they should
be asked. This is the working copy to take into the call and fill in.

**This file is the canonical list, and it is the only one.** New questions get added here.
The evaluation
([`NCX-Service-Gateway-Site-Access-Evaluation.md`](NCX-Service-Gateway-Site-Access-Evaluation.md))
and the decision portfolio ([`decision-portfolio.md`](decision-portfolio.md)) both point
here rather than carrying their own copies. Two question lists in one repository drift, the
diagrams here have already proved that, and a list people edit live during a call is the
worst place for it to happen. If a question belongs in the vendor conversation, it belongs
in this file, whatever document you were reading when you thought of it.

Every question carries four things. **Blocks** says what is waiting on the answer. **Why**
says where the question comes from. **Good answer** and **bad answer** are the point of the
exercise: they keep the conversation from ending in a reassuring non-answer, and they
record what changes if the answer is not the one the design assumes, so the reasoning does
not have to be rebuilt weeks later from a set of notes.

**Before the call.** Have the site inventory to hand, at least in draft, including edge
device model, WAN type, impact classification, and who administers the switch at each site.
Group A cannot be answered usefully without it. Worth having in the room: network
engineering, OT security, someone from compliance for group D, and someone who can talk
money for group E.

**Questions we have to answer ourselves are not here.** They are Section 5 of the decision
portfolio, and they matter more than anything below.

## Tracking

| # | Question | Blocks | Answered |
|---|---|---|---|
| 1 | Supported router models and minimum NCOS | The whole evaluation | [ ] |
| 2 | Non-Cradlepoint IPsec peers | Site coverage | [ ] |
| 3 | Does onboarding constrain local zone firewall config | Containment tier 1 | [ ] |
| 4 | Wildcards on named resources | Policy scaling | [ ] |
| 5 | Admin sessions to the LAN IP over the tunnel | Modem management, break-glass | [ ] |
| 6 | BGP over the overlay | Routing design | [ ] |
| 7 | Multicast roadmap | Telemetry and protection schemes | [ ] |
| 8 | Geo-redundant Service Gateways | HA design | [ ] |
| 9 | Cost and failover of a second network | HA business case | [ ] |
| 10 | Established sessions when NetCloud Manager is unreachable | Break-glass planning | [ ] |
| 11 | Clientless ZTNA on customer-hosted, roadmap | Path C, hosting choice | [ ] |
| 12 | Behavior at the 10% throughput cliff | Gateway sizing | [ ] |
| 13 | Gateway upgrade process and maintenance window | Operational acceptance | [ ] |
| 14 | Syslog and SIEM export, path to 90 day retention | Logging build | [ ] |
| 15 | NetCloud Client and GlobalProtect coexistence | Endpoint rollout | [ ] |
| 16 | Per-model Secure Connect throughput | Site-level sizing | [ ] |
| 17 | Rolling back a policy change that breaks a site | Change control | [ ] |
| 18 | CIP artifacts for cloud-delivered deployments | Medium impact plan | [ ] |
| 19 | Full SKU list for our shape | Budget | [ ] |
| 20 | Remote Connect, included or extra | Budget | [ ] |
| 21 | Early release of the 90 day ZTNA licence reservation | Contractor licensing | [ ] |
| 22 | Year four renewal and price movement | Budget | [ ] |
| 23 | Reference customers who removed the legacy L3 path | Confidence | [ ] |
| 24 | Paid or loaned PoC on two real sites | Pilot | [ ] |

---

## Group A. Ask first, these can end the evaluation

### 1. Which exact router models in our fleet support Secure Connect and ZTNA, and what minimum NCOS does each need?

- [ ] Asked
- **Blocks:** the whole evaluation.
- **Why:** Secure Connect requires an NCX-capable Cradlepoint at every participating site.
  The share of our sites already on capable hardware is the single number that decides NCX
  versus tightening the existing Palo Alto path. Ericsson's own terms state that not all
  NetCloud Edge routers are supported, so this has to be model by model, not brand level.
- **Good answer:** a model-by-model list checked against our serial numbers, in writing.
- **Bad answer:** "most current models." Also bad: a large share of the fleet needs
  replacing, which turns this from a licensing decision into a capital programme and
  probably hands the win to Palo Alto.

**Answer:**

**Date / source:**

---

### 2. Is there any supported way to terminate a non-Cradlepoint IPsec peer on the Service Gateway, for sites we cannot re-platform?

- [ ] Asked
- **Blocks:** site coverage, and therefore the completeness of the route withdrawal.
- **Why:** sites on fiber with a third party router, sites behind a partner-managed WAN, and
  serial-only gear with no IP router are not covered otherwise.
- **Good answer:** a clear no, with the Virtual Edge alternative explained properly.
- **Bad answer:** a vague "we can look into it." If the real answer is no, those sites keep
  the legacy path, and if the legacy path survives anywhere the route withdrawal becomes
  partial and the security model is weakened everywhere it does not reach.

**Answer:**

**Date / source:**

---

## Group B. These decide the shape of the design

### 3. Once a router is onboarded as an NCX site, does NCX also constrain local multi-LAN and zone firewall configuration, the way it already owns tunnels, DNS and routing?

- [ ] Asked
- **Blocks:** tier 1 of the containment plan, which is the tier that does not need switch access.
- **Why:** the highest priority question in this group. Our intra-site containment plan puts
  the security boundary on the router: separate LAN zones on separate physical ports with a
  default deny between them. That only works if NCX leaves local zone firewall and
  multi-LAN configuration alone. NCX already takes ownership of tunnels, DNS and routing on
  an onboarded site, so the question is whether the ownership stops there.
- **Good answer:** a documented statement of the supported configuration surface after
  onboarding, naming zone firewall and multi-LAN explicitly.
- **Bad answer:** a yes or no with no reference behind it. If NCX does constrain it, tier 1
  needs a different mechanism, most likely a separate firewall at every site, and the cost
  model moves from the gateway to every site.

**Answer:**

**Date / source:**

---

### 4. Do destination wildcards apply to named resources, or only to FQDNs? If only FQDNs, how is a policy meant to scale past 20 destinations?

- [ ] Asked
- **Blocks:** the entire policy scaling story.
- **Why:** the design depends on writing `*-RTU` and `*-RELAY-A` rather than enumerating
  sites. The published documentation describes wildcards for FQDN destinations and does not
  clearly extend them to resource names.
- **Good answer:** wildcards work on resource names, with a config reference.
- **Bad answer:** FQDN only. Then every policy enumerates resources site by site, the 20
  source and 20 destination ceiling per policy binds almost immediately, and the tag model
  stops being the answer to scale.

**Answer:**

**Date / source:**

---

### 5. Will NCOS accept an admin session to the router's LAN IP when the traffic arrives over the NCX tunnel rather than the physical LAN port?

- [ ] Asked
- **Blocks:** modem management, and part of the break-glass answer.
- **Why:** decides whether a jump host on the fabric can reach modem GUIs directly, or
  whether the Remote Connect add-on is required for routine modem management. It matters
  more than it looks, because today the public IP on the modem is the recovery path for a
  half-broken router, and we are proposing to retire that.
- **Good answer:** a definitive yes with a config reference.
- **Bad answer:** "you would want Remote Connect for that," with no explanation of why. That
  is another per-device licence line and it belongs in the cost model before signing, not
  during the pilot. See question 20.

**Answer:**

**Date / source:**

---

### 6. Is BGP supported over the NCX overlay today, or is it static routing only?

- [ ] Asked
- **Blocks:** routing design.
- **Why:** the documentation contradicts itself. The Administrator's Guide lists BGP as
  supported from NCOS 7.24.80. The NCX Technical FAQ states only static routing is supported
  and that policy routing steers traffic across the GRE and IPsec tunnels.
- **Good answer:** acknowledges the documentation conflict and names the current state.
- **Bad answer:** reads the Administrator's Guide back to us. Put the contradiction to them
  directly and cite both documents. A confident answer that does not acknowledge the
  conflict is not a reliable answer.

**Answer:**

**Date / source:**

---

### 7. Multicast: is there a roadmap, or is IPv4 unicast permanent?

- [ ] Asked
- **Blocks:** anything in telemetry or a protection scheme that relies on multicast crossing the WAN.
- **Why:** the fabric is IPv4 unicast only today. If something depends on multicast across
  the WAN, it will not traverse.
- **Good answer:** a dated roadmap position, including "no plans," which is a usable answer.
- **Bad answer:** "what do you need it for?"

**Answer:**

**Date / source:**

---

### 8. Is same-data-center active/standby the ceiling, or is there a supported geo-redundant Service Gateway option?

- [ ] Asked
- **Blocks:** the HA design, and it is the weakest part of the architecture for us.
- **Why:** the documentation is explicit that the standby must sit in the same data center
  as the active. For a utility where the access path to grid assets is operationally
  significant, that concentrates risk in one facility.
- **Good answer:** a named design, even if the design is "two separate networks."
- **Bad answer:** "you can put the standby anywhere." That contradicts the documentation, so
  push back and ask for it in writing.

**Answer:**

**Date / source:**

---

### 9. If we build a second gateway in a second data center for site redundancy, what does that cost in licensing, and can a site fail between them automatically?

- [ ] Asked
- **Blocks:** the business case for resilience.
- **Why:** if question 8 confirms the single data center ceiling, this is the actual price of
  resilience and it needs a number rather than a gesture.
- **Good answer:** a concrete SKU list and a described failover behavior.
- **Bad answer:** a quote with no failover explanation. Full duplicate licensing across all
  five dimensions may make the resilient design unaffordable, which is a finding worth
  having early rather than late.

**Answer:**

**Date / source:**

---

### 10. What happens to established sessions when NetCloud Manager is unreachable?

- [ ] Asked
- **Blocks:** break-glass planning, and the answer to the "OT cannot depend on a cloud
  service" objection.
- **Why:** NetCloud Manager is cloud only, with no on-premises option. The architecture says
  it is a control plane carrying no traffic, and that claim needs testing rather than
  repeating.
- **Good answer:** established sessions persist, new authentication fails, with a stated
  duration before anything degrades.
- **Bad answer:** "that does not happen." Storm restoration is exactly when this would hurt.

**Answer:**

**Date / source:**

---

### 11. What is the roadmap for clientless ZTNA on customer-hosted deployments?

- [ ] Asked
- **Blocks:** Path C, and by extension the hosting decision.
- **Why:** the strongest argument for NCX in a utility is isolated agentless third party
  access, and that is cloud-delivered only. The strongest argument for customer-hosted is
  keeping the grid data plane and the FIPS option in our own facility, and that argument
  gets considerably stronger if any site is ever categorized above low impact. Right now we
  have to pick.
- **Good answer:** a dated roadmap commitment.
- **Bad answer:** no roadmap, or a roadmap with no date. Then Path C for vendors becomes a
  jump host inside the site application zone reached with the NetCloud Client, and the
  isolation argument for NCX weakens considerably.

**Answer:**

**Date / source:**

---

## Group C. Operational, needed before the build rather than before the purchase

### 12. What exactly happens at the 10% throughput cliff? Which traffic is dropped, is it signaled, and how is it alarmed in NetCloud Manager?

- [ ] Asked
- **Blocks:** gateway sizing.
- **Why:** per the NCX terms, exceeding purchased throughput by 10% or more means the excess
  traffic is not transmitted. That is traffic loss, not an overage invoice, and it would
  arrive during exactly the storm or restoration event when every site is talking at once.
- **Good answer:** a described drop policy and an alert we can subscribe to.
- **Bad answer:** "you would just buy more capacity," or silent drop with no alarm. Either
  means sizing carries a large margin and we build the monitoring ourselves.

**Answer:**

**Date / source:**

---

### 13. What is the upgrade process for the Service Gateway, and what is the maintenance window?

- [ ] Asked
- **Blocks:** operational acceptance.
- **Why:** the gateway becomes the single path to every grid site. Its upgrade behavior is
  an operational commitment we are taking on, not a footnote.
- **Good answer:** a documented procedure with a rollback path and a realistic window.
- **Bad answer:** "it is seamless."

**Answer:**

**Date / source:**

---

### 14. What are the supported syslog and SIEM export formats and destinations for ZTNA authentication and policy-hit logs, and what is the shortest supported path to 90 days of retained records?

- [ ] Asked
- **Blocks:** the logging build, which is part of the project rather than a later improvement.
- **Why:** ZTNA logs may be exported and stored in a location determined by the user, and
  there is no built-in long-term retention. At medium impact, CIP-007 R4.3 puts a number on
  it: 90 days.
- **Good answer:** a named format and a working example.
- **Bad answer:** "logs can be exported." If export is manual or needs custom parsing, that
  is scope we are absorbing and it should be sized now.

**Answer:**

**Date / source:**

---

### 15. What is the interoperability guidance for the NetCloud Client alongside GlobalProtect on the same Windows endpoint?

- [ ] Asked
- **Blocks:** the endpoint rollout.
- **Why:** we will run both during migration whether we plan to or not. Two agents invites
  split-tunnel and route-priority conflicts, doubles the endpoint support burden, and splits
  access logs across two systems. This is where parallel SASE deployments actually break.
- **Good answer:** a tested statement, or an honest known-issues note.
- **Bad answer:** "should be fine." Either way this gets tested on a real corporate laptop
  image in week one, not at pilot exit.

**Answer:**

**Date / source:**

---

### 16. What is the actual Secure Connect throughput on the specific models we run?

- [ ] Asked
- **Blocks:** site-level sizing.
- **Why:** published figures range roughly 10 to 720 Mbps depending on model, and drop to
  roughly 10 to 400 Mbps with the Hybrid Mesh Firewall enabled. The R980 is documented at
  30 to 40 Mbps through Secure Connect against roughly 200 Mbps on plain NCOS IPsec, with no
  workaround.
- **Good answer:** per-model figures including the weak ones.
- **Bad answer:** a single headline number.

**Answer:**

**Date / source:**

---

### 17. How do we roll back a policy change that breaks a site?

- [ ] Asked
- **Blocks:** change control, and how comfortable anyone is editing policy after go-live.
- **Why:** a first-match ordered policy set with a deny-all at the bottom is easy to break
  from the top. The recovery path needs to exist before someone needs it at 2am.
- **Good answer:** policy versioning, or a supported export and restore.
- **Bad answer:** "you just change it back."

**Answer:**

**Date / source:**

---

## Group D. Compliance, bring someone from compliance to this part

### 18. For customers with NERC CIP obligations, what artifacts does Ericsson provide for a cloud-delivered deployment, and are there reference customers using the cloud-delivered model for anything categorized above low impact?

- [ ] Asked
- **Blocks:** the medium impact plan, and with it the hosting decision.
- **Why:** our sites are registered low impact today, where CIP-003 governs and no
  Electronic Security Perimeter is required. At medium impact the Service Gateway is
  performing electronic access control and the ZTNA broker sits in the interactive remote
  access path. Placing those functions in a vendor cloud is the open question NERC has not
  resolved.
- **Good answer:** named artifacts, and reference customers above low impact.
- **Bad answer:** no artifacts, or references only at low impact. That is not a failing on
  Ericsson's part, it is where the industry currently is, but it settles the hosting
  decision for us.

**Answer:**

**Date / source:**

---

## Group E. Commercial

### 19. Full SKU list for our shape: gateway capacity, HA add-on, per-site Secure Connect, per-device SD-WAN, per-device Hybrid Mesh Firewall, per-user ZTNA, and anything else required.

- [ ] Asked
- **Blocks:** the budget.
- **Why:** the licensing stack has five dimensions and both the 10% throughput cliff and the
  90 day ZTNA reservation punish under-modeling.
- **Good answer:** an itemized quote naming every dimension.
- **Bad answer:** a single blended number.

**Answer:**

**Date / source:**

---

### 20. Is Remote Connect included, and if not, what does it cost across the fleet?

- [ ] Asked
- **Blocks:** the budget, and it is tied to question 5.
- **Why:** if NCOS will not accept an admin session to the LAN IP over the tunnel, Remote
  Connect is not optional, it is a per-device line we have not budgeted.
- **Good answer:** a clear line item.
- **Bad answer:** silence. We already know it is separate.

**Answer:**

**Date / source:**

---

### 21. Can the 90 day ZTNA licence reservation be released early for a terminated contractor?

- [ ] Asked
- **Blocks:** contractor licence modelling, and it touches compliance.
- **Why:** each authenticated user acquires and reserves a licence for 90 days. With
  contractor churn we can consume licences far faster than headcount suggests. It is also a
  compliance question, because being unable to release a licence sits uncomfortably close to
  being unable to demonstrate that access was removed.
- **Good answer:** a described process.
- **Bad answer:** "it expires eventually." Then the licence count is modelled on churn
  rather than headcount, and that number gets built before purchase.

**Answer:**

**Date / source:**

---

### 22. What does renewal look like in year four, and what has price movement been on this product line?

- [ ] Asked
- **Blocks:** the budget, and the honest comparison against tightening Palo Alto.
- **Why:** a three year quote against a five year commitment is not a comparison.
- **Good answer:** honest numbers.
- **Bad answer:** deflection to "we can discuss at renewal."

**Answer:**

**Date / source:**

---

## Group F. Proof

### 23. Reference customers in electric distribution who **removed** the legacy L3 path, rather than running NCX in parallel. Can we talk to them directly?

- [ ] Asked
- **Blocks:** confidence in the one step that decides whether this project succeeds.
- **Why:** the make-or-break item in this whole evaluation is organizational, not technical.
  If the old routed path survives, we have bought a second network rather than a private one.
  A reference who actually completed the withdrawal is worth more than any feature answer in
  this document.
- **Good answer:** a named customer and an introduction.
- **Bad answer:** everyone runs it in parallel. That is a signal about how hard the last step
  is, and it should shape both the cutover plan and the written commitment we ask for
  internally.

**Answer:**

**Date / source:**

---

### 24. Will you support a paid or loaned proof of concept on two of our real sites, with an SE on the cutover call?

- [ ] Asked
- **Blocks:** the pilot.
- **Why:** the PoC plan needs one cellular site behind CGNAT and one on the current fiber
  path. Vendor presence on the cutover call is the cheapest insurance available.
- **Good answer:** yes, with named terms.
- **Bad answer:** a lab demonstration offered instead.

**Answer:**

**Date / source:**

---

## After the call

- [ ] Answers transcribed into this file with dates and sources
- [ ] Anything marked "in writing" actually received in writing, specifically questions 1, 8 and 9
- [ ] Evaluation updated where an answer changes a conclusion, with a revision history entry
- [ ] Reference design updated where an answer changes a drawing, both the HTML and the Mermaid copies
- [ ] Go and kill criteria in the decision portfolio re-checked against what we heard
- [ ] Questions that were deflected rather than answered re-listed for the follow-up call
