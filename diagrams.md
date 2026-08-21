# NCX Grid Access, Mermaid diagrams

Mermaid versions of the five drawing sheets in
[`ncx-grid-access-design.html`](ncx-grid-access-design.html). These render inline on
GitHub, in VS Code's markdown preview, and anywhere else Mermaid is supported, which
makes them easy to paste into tickets, wikis, or a design review deck.

The HTML version stays the reference drawing. These are the portable copies.

> **If you change a sheet, change both files.** The two have already drifted once. The
> HTML carries the title blocks, revision letter and sheet numbers, so it is the version
> that governs if they disagree.

**Symbols used below.** A solid blue box is on the encrypted fabric. A dashed blue box is
a cloud-delivered component that is not part of the drawn deployment. A dashed grey box
is a dark asset, meaning something with no defined resource and therefore no reachability.
An amber box is an OT application. These are kept distinct on purpose: the first revision
of this document used one dashed grey style for both cloud components and dark assets,
which made the legend say two different things on two different sheets.

---

## 1. System topology

Every modem dials outbound to the Service Gateway. Nothing routes inbound. NetCloud
Manager reaches every device for configuration and telemetry and carries none of the
traffic below.

```mermaid
flowchart TB
    NCM["<b>NetCloud Manager</b><br/>cloud · sites, resources, policy, telemetry"]

    subgraph CORP["Corporate network 10.20.0.0/16"]
        LAP["<b>Engineer laptop</b><br/>10.20.41.87<br/>NetCloud Client, always-on<br/><i>no route to any site LAN</i>"]
    end

    subgraph DC["Data center 10.30.0.0/16"]
        SG["<b>NCX Service Gateway</b><br/>customer-hosted virtual appliance<br/>mgmt0 10.30.99.10 · lan0 10.30.9.10<br/>wan0 198.51.100.10<br/><i>deny-all by default · first match wins</i>"]
        ADMS["<b>ADMS / SCADA master</b><br/>10.30.12.40"]
        HIST["<b>Historian</b><br/>10.30.12.55"]
    end

    subgraph VEN["Third party · unmanaged device"]
        OEM["<b>Relay OEM engineer</b><br/>browser only, no agent"]
        POR["<b>Clientless ZTNA portal</b><br/>isolated cloud container<br/><i>NOT AVAILABLE WITH THIS GATEWAY</i>"]
    end

    SUB["<b>SUB-042 · substation</b><br/>Cradlepoint E3000<br/>WAN1 + WAN2 5G, 2 tunnels<br/>3 LAN zones · .10 / .20 / .30"]
    RCL["<b>RCL-118 · recloser cabinet</b><br/>Cradlepoint S700<br/>WAN1 LTE, 1 tunnel<br/>1 LAN zone · .20"]
    CRW["<b>CREW-07 · service truck</b><br/>Cradlepoint R1900<br/>WAN1 + WAN2 5G<br/>1 LAN zone · .10"]

    LAP ==>|ZTNA tunnel| SG
    OEM --> POR
    POR -.->|brokered session, cloud-delivered model only| SG
    ADMS --- SG
    HIST --- SG

    SUB ==>|GRE over IPsec, dialed outbound| SG
    RCL ==>|GRE over IPsec| SG
    CRW ==>|GRE over IPsec| SG

    NCM -.->|control plane only| SG
    NCM -.-> LAP
    NCM -.-> SUB
    NCM -.-> RCL
    NCM -.-> CRW

    classDef fabric fill:#DBEAF0,stroke:#0F6E8C,stroke-width:2px,color:#0b2630
    classDef ot fill:#F6EBD3,stroke:#9C6B08,color:#2e1f02
    classDef cloud fill:none,stroke:#0F6E8C,stroke-dasharray:5 3,color:#0F6E8C
    class SG,SUB,RCL,CRW fabric
    class ADMS,HIST ot
    class POR cloud
```

> **The portal is drawn dashed for a reason.** The gateway above is customer-hosted, and
> the clientless portal is available only in the cloud-delivered model. As drawn, that
> path does not exist. See sheet 2 and the note under the access policy table.

---

## 2. Where the gateway lives

NetCloud Manager is always cloud. The choice is where the **Service Gateway** runs, and
it decides which features you get.

```mermaid
flowchart TB
    subgraph A["Option A, customer-hosted · NetCloud Exchange"]
        direction TB
        NCMA["NetCloud Manager · cloud"]
        SGA["<b>NCX Service Gateway</b><br/>in your data center<br/>VM or SG4000, you size it<br/>250 Mbps to 4 Gbps"]
        SITEA["SUB-042 · RCL-118 · CREW-07"]
        NCMA -.->|config + policy| SGA
        SITEA ==>|tunnels| SGA
        NOTEA["FIPS 140-3 available<br/>no clientless ZTNA<br/>active/standby, same DC only"]
    end

    subgraph B["Option B, cloud-delivered · NetCloud SASE"]
        direction TB
        NCMB["NetCloud Manager · cloud"]
        SGB["<b>Hosted gateway</b><br/>Ericsson operates it<br/>includes clientless portal"]
        SITEB["SUB-042 · RCL-118 · CREW-07"]
        DCB["Your data center<br/>joins via NetCloud Virtual Edge"]
        NCMB -.->|config + policy| SGB
        SITEB ==>|tunnels| SGB
        DCB ==> SGB
        NOTEB["clientless ZTNA available<br/>no FIPS option<br/>capacity managed for you"]
    end

    classDef fabric fill:#DBEAF0,stroke:#0F6E8C,stroke-width:2px,color:#0b2630
    classDef note fill:none,stroke:#63757E,stroke-dasharray:4 3,color:#63757E
    class SGA,SGB fabric
    class NOTEA,NOTEB note
```

**The conflict to settle before buying:** isolated agentless vendor access is
cloud-delivered only. FIPS and an on-premises data plane are customer-hosted only. If
the fleet ever carries medium impact sites, the gateway is performing electronic access
control and sits in the interactive remote access path, which makes the cloud-delivered
option considerably harder to defend.

---

## 3. Under the modem, SUB-042 detail

Two different mechanisms, and they are easy to confuse. NCX decides who may reach this
site from outside. The router zone firewall decides what may move between zones once
something is already inside the cabinet. NCX has no view of the second one.

Every zone is a separate router LAN on its own physical port, so this works with
unmanaged switches and does not require administrative access to a managed switch.

```mermaid
flowchart TB
    SG(["NCX Service Gateway"])
    RTR["<b>SUB-042 site router, the modem</b><br/>Cradlepoint E3000<br/><i>NCX owns tunnel, DNS and routing config</i><br/><i>zone firewall is local NCOS config</i>"]

    RTR ==>|WAN1 · 5G carrier A, dialed outbound| SG
    RTR ==>|WAN2 · 5G carrier B, standby| SG

    subgraph Z2["ZONE LAN-OT-CTRL · port 2 · 192.168.20.0/24"]
        RTU["<b>RTU-042</b> · .30 · DNP3<br/>resource SUB042-RTU"]
        RLA["<b>RELAY-A</b> · .31 · HTTPS<br/>resource SUB042-RELAY-A"]
        RLB["<b>RELAY-B</b> · .32 · HTTPS<br/>resource SUB042-RELAY-B"]
    end

    subgraph Z1["ZONE LAN-OT-APPS · port 3 · 192.168.10.0/24"]
        HOST["Virtualization host · .20"]
        HMI["<b>HMI-042</b> · .21<br/>Windows engineering HMI<br/>resource SUB042-HMI"]
        PGW["<b>PGW-042</b> · .22<br/>DNP3 to Modbus gateway<br/>resource SUB042-PGW"]
        HSTL["<b>HIST-042</b> · .23<br/>local historian collector<br/>NOT DEFINED, dark"]
        HOST --- HMI
        HOST --- PGW
        HOST --- HSTL
    end

    subgraph Z3["ZONE LAN-AUX · port 4 · 192.168.30.0/24"]
        CAM["<b>CAMERA</b> · .50<br/>no resource, dark"]
        SPR["<b>SPARE / UNKNOWN</b> · .64<br/>no resource, dark"]
    end

    RTR --- Z1
    RTR --- Z2
    RTR --- Z3

    Z1 ==>|allow TCP 443, TCP 20000| Z2
    Z2 x--x|deny| Z1
    Z3 x--x|deny both directions| Z1
    Z3 x--x|deny both directions| Z2

    classDef fabric fill:#DBEAF0,stroke:#0F6E8C,stroke-width:2px,color:#0b2630
    classDef allow fill:#DCEDE5,stroke:#26765A,color:#0b2b1e
    classDef dark fill:none,stroke:#63757E,stroke-dasharray:4 3,color:#63757E
    class SG,RTR fabric
    class HMI,PGW,RTU,RLA,RLB allow
    class HSTL,CAM,SPR dark
```

> **What zoning still does not fix.** Devices sharing a zone reach each other at layer 2,
> and that traffic never touches the router, so neither the zone firewall nor the Hybrid
> Mesh Firewall east-west inspection can see it. Zoning contains movement between groups,
> not inside one. Closing that last gap needs managed switch access.

---

## 4. Desk to substation HMI

The sequence that replaces "it just works because we're in the office." Steps 1 to 7 are
identical from the office, from home, and from a truck.

```mermaid
sequenceDiagram
    autonumber
    participant L as Laptop<br/>NetCloud Client
    participant I as Entra ID / Okta
    participant N as NetCloud Manager
    participant G as Service Gateway
    participant M as SUB-042 modem
    participant H as HMI-042<br/>192.168.10.21

    L->>I: SAML authentication request
    I-->>L: assertion · MFA passed · group = Grid-Engineering
    L->>N: present assertion
    N-->>L: session token + resource list this user may see
    L->>G: encrypted client tunnel comes up
    Note over L,G: laptop joins the Secure Connect network.<br/>Membership, not reachability.
    L->>G: opens https://sub042-hmi.grid.internal
    Note over G: name resolves inside the fabric only.<br/>POLICY 1 MATCHES, allow, session logged
    G->>M: into the GRE / IPsec tunnel
    M->>H: TCP 443 to 192.168.10.21, zone LAN-OT-APPS
    H-->>L: HMI loads · return traffic reverses the same path

    rect rgb(245, 226, 221)
    L->>G: same user tries the camera at 192.168.30.50
    Note over G: no policy match, dropped.<br/>The name never resolved either.
    end
```

---

## 5. Before and after cutover

Standing up the fabric changes nothing on its own. Until the legacy route is withdrawn,
both doors are open and you have a second network rather than a private one.

```mermaid
flowchart LR
    subgraph T["Today, implicit trust"]
        direction LR
        H1["any corporate host"] --> R1["core routing + WAN"] --> S1["site LAN<br/>every device"]
        N1["No authentication.<br/>No per-user authorization.<br/>Reachability is a property<br/>of the routing table."]
    end

    subgraph AF["After cutover, explicit access"]
        direction LR
        H2["authenticated<br/>client session"] --> G2["Service Gateway<br/>policy match"] --> S2["one resource<br/>one session"]
        X2["corporate routes to the<br/>site subnets, withdrawn<br/><br/>Being on the corporate LAN<br/>grants nothing."]
    end

    classDef ot fill:#F6EBD3,stroke:#9C6B08,color:#2e1f02
    classDef fabric fill:#DBEAF0,stroke:#0F6E8C,stroke-width:2px,color:#0b2630
    classDef allow fill:#DCEDE5,stroke:#26765A,color:#0b2b1e
    classDef note fill:none,stroke:#63757E,stroke-dasharray:4 3,color:#63757E
    class S1 ot
    class G2 fabric
    class S2 allow
    class N1,X2 note
```

**Prove it one site at a time.** Withdraw the legacy route for SUB-042 only and run a
full monthly cycle, including a planned outage and an after-hours callout, before
touching the second site. The test that matters is not "can the engineer get in," it is
"did anything quietly depend on the flat path that nobody documented."
