# NCX Grid Site Access

Evaluation and reference design for making access to distribution grid sites private,
using the Ericsson Cradlepoint NetCloud Exchange (NCX) Service Gateway in place of the
current model (open access from the corporate LAN, GlobalProtect for remote users).

## Contents

| File | What it is |
|---|---|
| [`NCX-Service-Gateway-Site-Access-Evaluation.md`](NCX-Service-Gateway-Site-Access-Evaluation.md) | Product evaluation — components, how NCX substitutes for a VPN, constraints and reasons it might not fit, SE questions, PoC plan. |
| [`ncx-grid-access-design.html`](ncx-grid-access-design.html) | Reference design, five drawing sheets: topology, deployment options, site detail under a modem, resource/policy model, user access sequence, cutover. Open in a browser. |
| [`diagrams.md`](diagrams.md) | The same five diagrams in Mermaid — renders inline on GitHub and in VS Code, for pasting into tickets, wikis, or slides. |
| [`decision-portfolio.md`](decision-portfolio.md) | Talking points by audience, objection handling, vendor questions with good/bad answers, and explicit go/kill criteria. |

## Status

Illustrative. Site names, addresses, and group names in the design are examples.
Before committing to this shape, confirm with an Ericsson SE:

1. Which routers in the fleet are Secure Connect and ZTNA capable, and minimum NCOS per model.
2. Whether NCOS accepts admin sessions to the LAN IP when traffic arrives over the NCX tunnel
   (decides whether a jumphost can reach modem GUIs without the Remote Connect add-on).
3. Whether BGP is supported over the NCX overlay — the Administrator's Guide and the
   Technical FAQ contradict each other.
4. Whether any geo-redundant Service Gateway option exists, or if same-data-center
   active/standby is the ceiling.

## Open items on our side

- Site inventory by edge device and WAN type. The share of sites already on NCX-capable
  Cradlepoint hardware decides NCX vs. tightening the existing Palo Alto path.
- Written commitment to withdraw the legacy corporate routes to site subnets after cutover.
  Without it the fabric is a second network, not a private one.
- Current monthly cost of static/public IP plans per SIM — offsets licensing if retired.
