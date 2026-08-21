# NCX Grid Site Access

Evaluation and reference design for making access to distribution grid sites private,
using the Ericsson Cradlepoint NetCloud Exchange (NCX) Service Gateway in place of the
current model (open access from the corporate LAN, GlobalProtect for remote users), and
for containing lateral movement inside a site once that access path exists.

## Contents

| File | What it is |
|---|---|
| [`NCX-Service-Gateway-Site-Access-Evaluation.md`](NCX-Service-Gateway-Site-Access-Evaluation.md) | Product evaluation. Components, how NCX substitutes for a VPN, containment inside a site, NERC CIP applicability, constraints and reasons it might not fit, SE questions, PoC plan. |
| [`ncx-grid-access-design.html`](ncx-grid-access-design.html) | Reference design. Ten sections, five numbered drawing sheets: topology, deployment options, site detail under a modem, user access sequence, cutover. Open it in a browser. |
| [`diagrams.md`](diagrams.md) | The same five sheets in Mermaid. Renders inline on GitHub and in VS Code, for pasting into tickets, wikis, or slides. |

The HTML file is a complete standalone document. It pulls fonts from Google Fonts, so
opened offline it falls back to system fonts and is otherwise unaffected. It prints to
PDF in landscape with the sheets kept whole.

**If you change a sheet, change both files.** The HTML carries the title blocks, revision
letter and sheet numbers, so it governs if the two disagree.

## Status

Illustrative. Site names, addresses, and group names in the design are examples.
Revision B. Reviewers still required: OT security, compliance, and the team that owns the
site switches.

Before committing to this shape, confirm with an Ericsson SE:

1. Which routers in the fleet are Secure Connect and ZTNA capable, and minimum NCOS per model.
2. Whether onboarding a router as an NCX site constrains local multi-LAN and zone firewall
   configuration, the way it already constrains tunnels, DNS and routing. The intra-site
   segmentation plan depends on the answer.
3. Whether destination wildcards apply to named resources or only to FQDNs. If only FQDNs,
   every policy has to enumerate sites and the 20-entry ceiling arrives quickly.
4. Whether NCOS accepts admin sessions to the LAN IP when traffic arrives over the NCX tunnel,
   which decides whether a jumphost can reach modem GUIs without the Remote Connect add-on.
5. Whether BGP is supported over the NCX overlay. The Administrator's Guide and the
   Technical FAQ contradict each other.
6. Whether any geo-redundant Service Gateway option exists, or if same-data-center
   active/standby is the ceiling.

## Open items on our side

- Site inventory by edge device and WAN type, plus two columns the first revision did not
  have: impact classification supplied by compliance, and who administers the site switch.
  The share of sites already on NCX-capable Cradlepoint hardware decides NCX versus
  tightening the existing Palo Alto path.
- Written commitment to withdraw the legacy corporate routes to site subnets after cutover.
  Without it the fabric is a second network, not a private one.
- Current monthly cost of static and public IP plans per SIM, which offsets licensing if retired.
- The medium impact plan. Whether existing low impact sites get reclassified or net new
  medium sites are taken on changes the Electronic Security Perimeter work and pushes the
  cloud-delivered versus customer-hosted decision toward customer-hosted. See Section 9.

## What this does not cover

NCX contains movement between sites and between the corporate network and a site. It does
nothing about movement inside a site. That is local segmentation work, most of it does not
require administrative access to the site switches, and Section 7 of the evaluation sets
out the ordering.
