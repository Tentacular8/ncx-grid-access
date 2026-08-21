# SE questions, working checklist

Questions for the Ericsson and Cradlepoint SE, in the order they should be asked. This is
the working copy to take into the call and fill in. The evaluation
([`NCX-Service-Gateway-Site-Access-Evaluation.md`](NCX-Service-Gateway-Site-Access-Evaluation.md))
points here rather than carrying its own list, so there is one place to update.

Every question below has a **why** line and a **bad answer** line. The bad answer line is
the point: it says what changes if the answer is not the one the design assumes, so nobody
has to reconstruct the reasoning weeks later.

**Before the call.** Have the site inventory to hand, at least in draft, including edge
device model, WAN type, impact classification, and who administers the switch at each site.
Group A cannot be answered usefully without it. Worth having on the call: network
engineering, OT security, and someone from compliance for group D.

## Tracking

| # | Question | Blocks | Answered |
|---|---|---|---|
| 1 | Supported router models and minimum NCOS | The whole evaluation | [ ] |
| 2 | Non-Cradlepoint IPsec peers | Site coverage | [ ] |
| 3 | Does onboarding constrain local zone firewall config | Containment tier 1 | [ ] |
| 4 | Wildcards on named resources | Policy scaling | [ ] |
| 5 | Admin sessions to the LAN IP over the tunnel | Modem management | [ ] |
| 6 | BGP over the overlay | Routing design | [ ] |
| 7 | Geo-redundant Service Gateways | HA design | [ ] |
| 8 | Licensing cost of a second network for redundancy | HA business case | [ ] |
| 9 | Clientless ZTNA on customer-hosted, roadmap | Path C | [ ] |
| 10 | Behavior at the 10% throughput cliff | Gateway sizing | [ ] |
| 11 | Syslog and SIEM export, path to 90 day retention | Logging build | [ ] |
| 12 | NetCloud Client and GlobalProtect coexistence | Endpoint rollout | [ ] |
| 13 | CIP artifacts for cloud-delivered deployments | Medium impact plan | [ ] |
| 14 | Early release of the 90 day ZTNA licence reservation | Contractor licensing | [ ] |
| 15 | Reference customers who removed the legacy L3 path | Confidence | [ ] |

---

## Group A. Ask first, these can end the evaluation

### 1. Which exact router models in our fleet are Secure Connect and ZTNA capable, and what minimum NCOS does each need?

- [ ] Asked
- **Why:** Secure Connect requires an NCX-capable Cradlepoint at every participating site.
  The percentage of our sites already on capable hardware is the single number that decides
  NCX versus tightening the existing Palo Alto path. Ericsson's own terms state that not all
  NetCloud Edge routers are supported, so this has to be model by model, not brand level.
- **Bad answer:** a large share of the fleet needs replacing. That turns this from a
  licensing decision into a capital programme and the Palo Alto route probably wins.
- **Ask for it in writing,** as a model list rather than a verbal confirmation.

**Answer:**

**Date / source:**

---

### 2. Is there any supported way to terminate a non-Cradlepoint IPsec peer on the Service Gateway, for sites we cannot re-platform?

- [ ] Asked
- **Why:** sites on fiber with a third party router, sites behind a partner-managed WAN, and
  serial-only gear with no IP router are not covered otherwise.
- **Bad answer:** no. Then those sites keep the legacy path, and if the legacy path survives
  anywhere, the route withdrawal in the cutover plan becomes partial and the security model
  is weakened everywhere it does not reach.

**Answer:**

**Date / source:**

---

## Group B. These decide the shape of the design

### 3. Once a router is onboarded as an NCX site, does NCX also constrain local multi-LAN and zone firewall configuration, the way it already owns tunnels, DNS and routing?

- [ ] Asked
- **Why:** this is the highest priority question in group B. Our intra-site containment plan
  puts the security boundary on the router: separate LAN zones on separate physical ports
  with a default deny between them. That plan only works if NCX leaves local zone firewall
  and multi-LAN configuration alone. NCX already takes ownership of tunnels, DNS and routing
  on an onboarded site, so the question is whether the ownership stops there.
- **Bad answer:** NCX constrains it. Then tier 1 of the containment plan needs a different
  mechanism, most likely a separate firewall at each site, and the cost model changes at
  every site rather than at the gateway.
- **Follow-up if the answer is vague:** ask for the supported configuration surface after
  onboarding, in writing, rather than a yes or no.

**Answer:**

**Date / source:**

---

### 4. Do destination wildcards apply to named resources, or only to FQDNs? If only FQDNs, how is a policy meant to scale past 20 destinations?

- [ ] Asked
- **Why:** the whole scaling story for policy depends on writing `*-RTU` and `*-RELAY-A`
  rather than enumerating sites. The published documentation describes wildcards for FQDN
  destinations and does not clearly extend them to resource names.
- **Bad answer:** FQDN only. Then every policy enumerates resources site by site, the
  20 source and 20 destination ceiling per policy binds almost immediately, and the tag
  model stops being the answer to scale.

**Answer:**

**Date / source:**

---

### 5. Does NCOS accept admin sessions to the LAN IP when traffic arrives over the NCX tunnel?

- [ ] Asked
- **Why:** decides whether a jump host on the fabric can reach modem GUIs directly, or
  whether the Remote Connect add-on is required for routine modem management.
- **Bad answer:** no. That is another per-device licence line, and it should be in the cost
  model before signing rather than discovered during the pilot.

**Answer:**

**Date / source:**

---

### 6. Is BGP supported over the NCX overlay today, or is it static routing only?

- [ ] Asked
- **Why:** the documentation contradicts itself. The Administrator's Guide lists BGP as
  supported from NCOS 7.24.80. The NCX Technical FAQ states only static routing is supported
  and that policy routing steers traffic across the GRE and IPsec tunnels.
- **Bad answer:** static only. Not fatal at our scale, but it has to be known before anyone
  designs around dynamic routing.
- **Note:** put the contradiction to them directly and cite both documents. A confident
  answer that does not acknowledge the conflict is not a reliable answer.

**Answer:**

**Date / source:**

---

### 7. Is there any supported path for geo-redundant Service Gateways, or is same-data-center active/standby the ceiling?

- [ ] Asked
- **Why:** the documentation is explicit that the standby must sit in the same data center as
  the active. For a utility where the access path to grid assets is operationally
  significant, that concentrates risk in one facility.
- **Bad answer:** same data center is the ceiling, which is what the documentation implies.
  Then site-level redundancy means a second separately licensed network, which is question 8.

**Answer:**

**Date / source:**

---

### 8. What is the licensing impact of standing up a second network purely for redundancy?

- [ ] Asked
- **Why:** if question 7 confirms the single data center ceiling, this is the actual cost of
  resilience and it needs a number, not a gesture.
- **Bad answer:** full duplicate licensing across all five dimensions. That may make the
  resilient design unaffordable, which is a finding worth having early rather than late.
- **Get this in writing.**

**Answer:**

**Date / source:**

---

### 9. What is the roadmap for clientless ZTNA on customer-hosted deployments?

- [ ] Asked
- **Why:** the strongest argument for NCX in a utility is isolated agentless third party
  access, and that is cloud-delivered only. The strongest argument for customer-hosted is
  keeping the grid data plane and the FIPS option in our own facility, and that argument gets
  stronger the moment any site is categorized above low impact. Right now we have to pick.
- **Bad answer:** no roadmap, or a roadmap with no date. Then Path C for vendors becomes a
  jump host inside the site application zone reached with the NetCloud Client, and the
  isolation argument for NCX weakens considerably.

**Answer:**

**Date / source:**

---

## Group C. Operational, needed before the build rather than before the purchase

### 10. What exactly happens at the 10% throughput cliff? Which traffic is dropped, is it signaled, and how is it alarmed in NetCloud Manager?

- [ ] Asked
- **Why:** per the NCX terms, exceeding purchased throughput by 10% or more means the excess
  traffic is not transmitted. That is traffic loss, not an overage invoice, and it would
  arrive during exactly the storm or restoration event when every site is talking at once.
- **Bad answer:** no alarm, or silent drop. Then gateway sizing has to carry a large margin
  and the monitoring for it has to be built by us.

**Answer:**

**Date / source:**

---

### 11. What are the supported syslog and SIEM export formats and destinations for ZTNA authentication and policy-hit logs, and what is the shortest supported path to 90 days of retained records?

- [ ] Asked
- **Why:** ZTNA logs may be exported and stored in a location determined by the user, and
  there is no built-in long-term retention. At medium impact, CIP-007 R4.3 puts a number on
  it: 90 days.
- **Bad answer:** export is manual, or the format needs custom parsing. Either way the
  logging path is part of the build, not a later improvement.

**Answer:**

**Date / source:**

---

### 12. What is the interoperability guidance for the NetCloud Client coexisting with GlobalProtect on the same Windows endpoint?

- [ ] Asked
- **Why:** we will run both during migration whether we plan to or not. Two agents invites
  split-tunnel and route-priority conflicts, doubles the endpoint support burden, and splits
  access logs across two systems. This is the most common practical failure in parallel SASE
  deployments.
- **Bad answer:** untested, or a list of caveats. Then this gets tested in phase 2 of the
  pilot on real hardware before any wider rollout.

**Answer:**

**Date / source:**

---

## Group D. Compliance, bring someone from compliance to this part

### 13. For customers with NERC CIP obligations, what artifacts does Ericsson provide for a cloud-delivered deployment, and are there reference customers using the cloud-delivered model for anything categorized above low impact?

- [ ] Asked
- **Why:** at medium impact the Service Gateway is performing electronic access control and
  the ZTNA broker sits in the interactive remote access path. Placing those functions in a
  vendor cloud is the open question NERC has not resolved. If the fleet takes on medium
  impact sites, this decides whether cloud-delivered stays on the table at all.
- **Bad answer:** no artifacts, or reference customers only at low impact. That is not a
  failing on Ericsson's part, it is where the industry currently is, but it settles the
  hosting decision for us.

**Answer:**

**Date / source:**

---

### 14. Can the 90 day ZTNA licence reservation be released early for terminated contractors?

- [ ] Asked
- **Why:** each authenticated user acquires and reserves a licence for 90 days. With
  contractor churn we can consume licences far faster than headcount suggests. It is also a
  compliance question, because being unable to release a licence is adjacent to being unable
  to demonstrate that access was removed.
- **Bad answer:** no early release. Then the licence count is modelled on churn rather than
  headcount, and the number should be built before purchase.

**Answer:**

**Date / source:**

---

### 15. Reference customers in electric distribution, specifically ones who removed the legacy L3 path rather than running NCX in parallel.

- [ ] Asked
- **Why:** the make-or-break item in this whole evaluation is organizational, not technical.
  If the old routed path survives, we have bought a second network rather than a private one.
  A reference who actually completed the withdrawal is worth more than any feature answer in
  this document.
- **Bad answer:** everyone runs it in parallel. That is a signal about how hard the last step
  is, and it should shape the cutover plan and the written commitment we ask for internally.

**Answer:**

**Date / source:**

---

## After the call

- [ ] Answers transcribed into this file with dates and sources
- [ ] Anything marked "get in writing" actually received in writing
- [ ] Evaluation updated where an answer changes a conclusion, with the revision history entry
- [ ] Reference design updated where an answer changes a drawing, both the HTML and the Mermaid copies
- [ ] Questions that were deflected rather than answered re-listed for the follow-up call
