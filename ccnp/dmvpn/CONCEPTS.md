# VPN Technologies (DMVPN + MPLS) — In-Depth Concepts

The companion reference to the [README](README.md) lab guide. The README is the
*hands-on* path (build the tunnel, register the spoke, watch spoke-to-spoke form);
this file is the *why* — for DMVPN, and for the **describe-level MPLS** topics (§2.1,
§2.2) you won't build but must understand. Each 📖 **Deep dive** link in the README
jumps to a section here.

**Contents**
- [What DMVPN is: underlay and overlay](#what-dmvpn-is-underlay-and-overlay)
- [mGRE: one tunnel, many endpoints](#mgre-one-tunnel-many-endpoints)
- [NHRP: resolution for the overlay](#nhrp-resolution-for-the-overlay)
- [The three DMVPN phases](#the-three-dmvpn-phases)
- [Routing over DMVPN](#routing-over-dmvpn)
- [IPsec protection](#ipsec-protection)
- [MPLS operations](#mpls-operations) *(describe — §2.1)*
- [MPLS Layer 3 VPN](#mpls-layer-3-vpn) *(describe — §2.2)*

---

## What DMVPN is: underlay and overlay

**DMVPN** (Dynamic Multipoint VPN) is Cisco's answer to a hard problem: give many
sites **full any-to-any private connectivity** across a public transport, **without**
configuring a tunnel for every pair of sites and **without** touching the hub every
time you add a spoke. It stacks four technologies you already half-know: **mGRE** +
**NHRP** + a **routing protocol** + **IPsec**.

The mental model is two layers:

- **Underlay** — the real transport network (the internet, or an MPLS/broadband WAN)
  that carries packets between the sites' **public / NBMA** addresses (NBMA =
  Non-Broadcast Multi-Access, the term inherited from Frame Relay/ATM). In this lab the
  ISP router *is* the underlay; the `203.0.x` addresses are the NBMA addresses.
- **Overlay** — the private **tunnel** network (`172.16.0.0/24` here) with its own
  addressing, over which your routing protocol runs and your real traffic
  (`192.168.x`) flows. Every overlay packet is **wrapped in GRE** (and optionally
  IPsec) and sent across the underlay.

The payoff: a new spoke needs only **spoke-side** config pointing at the hub; the hub
learns it dynamically via NHRP. Spokes can sit behind **dynamic or NAT'd** public IPs
(the hub needs a stable one). And any two spokes can talk **directly**, on demand,
without pre-configuration.

[↩ back to README captures](README.md#using-edgeshark-for-captures)

---

## mGRE: one tunnel, many endpoints

A plain **GRE** tunnel is **point-to-point**: `tunnel destination x.x.x.x` hard-codes
exactly one far end. N spokes would mean N tunnels on the hub — the config explosion
DMVPN exists to avoid.

**Multipoint GRE** (`tunnel mode gre multipoint`) removes the fixed destination. One
mGRE interface can reach **any** peer; when it needs to send an overlay packet, it
asks: *"what NBMA (public) address is behind this overlay next-hop?"* and looks the
answer up in **[NHRP](#nhrp-resolution-for-the-overlay)**. So the destination is chosen
**per-packet, dynamically**, instead of being pinned in config.

That's the whole trick: **mGRE provides the multipoint tunnel, NHRP provides the
destination lookup.** The interface carries a `tunnel source` (its own public
interface) and an `ip nhrp network-id` (a locally-significant tag grouping the tunnel),
but crucially **no `tunnel destination`** — because there are many.

[↩ back to README Lab 1](README.md#lab-1--mgre--nhrp-spokes-register-with-the-hub)

---

## NHRP: resolution for the overlay

**NHRP** (Next Hop Resolution Protocol, RFC 2332) is the **"ARP of the overlay"** — it
maps an **overlay** (tunnel) address to the **NBMA** (public) address behind it, which
is exactly what mGRE needs to send a packet. Two roles:

- **NHS (Next Hop Server)** — the **hub**. It holds the authoritative table of every
  spoke's overlay↔NBMA mapping.
- **NHC (Next Hop Client)** — each **spoke**. It's configured only with the hub's
  mapping (`ip nhrp nhs … nbma …`).

Two operations do the work:

- **Registration** — on boot, a spoke sends an **NHRP Registration Request** to the
  NHS: *"my overlay 172.16.0.2 is reachable at NBMA 203.0.12.1."* The hub records it.
  This is why the hub always knows all spokes **even if their public IP is dynamic** —
  the spoke tells it. (README Lab 1.)
- **Resolution** — when spoke A needs spoke B's NBMA address to build a **direct**
  tunnel, it sends an **NHRP Resolution Request** (via the hub) and gets B's public
  address in a **Reply**, then talks to B directly. (README Lab 3.)

One subtlety that trips people up: the tunnel needs to carry **multicast** (routing
protocol hellos) even though the underlay may not support it. NHRP maps multicast too —
the hub uses **`ip nhrp map multicast dynamic`** to replicate multicast to every
registered spoke, and each spoke maps multicast to the hub (the `multicast` keyword on
its `nhs` line). Get that wrong and the tunnel is up but the routing protocol never
forms a neighbor.

[↩ back to README Lab 1](README.md#lab-1--mgre--nhrp-spokes-register-with-the-hub)

---

## The three DMVPN phases

DMVPN evolved through three **phases**, and knowing which one you're looking at is a
core troubleshooting skill:

| | **Phase 1** | **Phase 2** | **Phase 3** |
|---|---|---|---|
| Spoke tunnel | **p2p GRE** to hub | **mGRE** | **mGRE** |
| Spoke-to-spoke | ✗ — all via hub | ✓ direct | ✓ direct |
| How direct forms | n/a | spoke initiates NHRP resolution itself | hub sends **NHRP redirect**, spoke installs **shortcut** |
| Hub next-hop | rewritten to hub (fine) | **must be preserved** | preserved |
| Summarization at hub | ✓ (spokes only need a default) | ✗ (spokes need specific routes) | ✓ (redirect fixes it) |

- **Phase 1** — pure hub-and-spoke. Simple, and the hub can hand spokes a **summary /
  default** (they never talk directly, so they don't need specifics). But every
  spoke-to-spoke packet hair-pins through the hub.
- **Phase 2** — direct spoke-to-spoke via mGRE, but it demands that spokes have the
  **specific** route to the other spoke with the **originating spoke as next-hop**
  (hence `no ip next-hop-self` and **no** summarization on the hub) — which limits
  scale.
- **Phase 3** — direct spoke-to-spoke via **NHRP redirect** (hub) + **shortcut**
  (spoke). The hub can summarize again; when traffic hair-pins, the hub **redirects**
  the spoke to resolve a direct path, and CEF installs a **shortcut**. Best scale, and
  the **modern default** — it's what this lab builds (`ip nhrp redirect` on the hub,
  `ip nhrp shortcut` on spokes).

[↩ back to README Lab 3](README.md#lab-3--spoke-to-spoke-phase-3-on-demand)

---

## Routing over DMVPN

DMVPN gives you a tunnel; a **routing protocol over that tunnel** is what actually
distributes the LAN prefixes. EIGRP, OSPF, and BGP all work, but the hub-and-spoke
shape fights two default behaviours you must override (EIGRP shown, as in the lab):

- **Split horizon** — a router won't re-advertise a route back out the interface it
  learned it on. On the hub's single mGRE interface, that means **one spoke's routes
  never reach the other spokes**. Fix: **`no ip split-horizon eigrp <as>`** on the hub
  tunnel. (OSPF doesn't have this issue but has its own: pick a sane network type —
  point-to-multipoint — and force the **DR onto the hub**.)
- **Next-hop rewrite** — by default the hub advertises spoke routes with **itself** as
  the next-hop, which forces traffic through the hub. **`no ip next-hop-self eigrp
  <as>`** preserves the **originating spoke** as the next-hop, which is the
  prerequisite for a **direct** spoke-to-spoke path.

The through-line: on a hub, you're deliberately turning off the loop-prevention and
next-hop defaults that make sense on normal interfaces, because the hub's *job* here is
to reflect routes among spokes and point them at each other. (BGP is common at large
scale — dynamic peers, and the hub as a route-reflector-like reflector of spoke
prefixes.)

[↩ back to README Lab 2](README.md#lab-2--routing-over-dmvpn-eigrp)

---

## IPsec protection

GRE gives you the multipoint tunnel but **no confidentiality** — anyone on the
underlay can read the encapsulated traffic. **IPsec** adds encryption/authentication,
and DMVPN integrates it in the scalable way: an **IPsec profile** applied to the tunnel
as **`tunnel protection ipsec profile NAME`**, *not* the old per-peer crypto maps.

Why that matters: a crypto map needs an entry per peer — impossible when spoke-to-spoke
tunnels form **dynamically**. A **single profile** on the mGRE interface protects
**every** tunnel it builds, hub-spoke and dynamic spoke-to-spoke alike. The pieces:

- **IKE** (IKEv2 preferred, or IKEv1/ISAKMP) — negotiates keys and authenticates peers,
  typically with a **wildcard pre-shared key** (`peer … address 0.0.0.0 0.0.0.0`) or
  PKI, since you can't pre-list every spoke.
- **ESP transform set** — the actual encryption (AES) + integrity (SHA). DMVPN usually
  uses **transport mode**, because GRE already added an outer IP header — tunnel mode
  would add a *second* one and waste overhead.
- **IPsec profile** — bundles the transform set + IKE profile, referenced by `tunnel
  protection`.

On the wire the GRE is then wrapped in **ESP**. *(Caveat: the IOL image in this lab may
lack a working crypto engine, so the SAs may not form — the config is correct and is
what you'd deploy; the DMVPN mechanics in Labs 1–3 don't depend on it.)*

[↩ back to README Lab 4](README.md#lab-4--ipsec-protection-config--concept)

---

## MPLS operations

*(ENARSI §2.1 — describe.)* **MPLS** (Multiprotocol Label Switching) forwards packets
by a short **label** instead of the destination IP. An ingress router prepends a 20-bit
label; downstream routers forward by **swapping** labels via a simple table lookup, and
the egress router removes it — no per-hop IP route lookup in the core. It sits "between"
L2 and L3 (sometimes called "layer 2.5").

The vocabulary the exam names:

- **LSR (Label Switch Router)** — a core ("**P**", provider) router that **swaps** an
  incoming label for an outgoing one and forwards. It only ever looks at labels.
- **LER / edge LSR (PE)** — imposes the label on the way in (**push**) and removes it on
  the way out (**pop**).
- **LDP (Label Distribution Protocol)** — how neighboring LSRs **advertise label
  bindings** to each other ("for prefix X, use label 17"), building a consistent set of
  labels hop-by-hop. (TDP is the legacy predecessor; RSVP-TE is the traffic-engineering
  alternative.)
- **LSP (Label Switched Path)** — the **end-to-end path** a labeled packet follows
  through the LSRs, built from those bindings. LSPs are **unidirectional**.
- **FEC (Forwarding Equivalence Class)** — the group of packets treated the same (given
  the same label), usually "all packets to prefix X."

Two ideas to hold: the **control plane** (an IGP like OSPF/IS-IS builds reachability,
LDP maps labels to it) is separate from the **data plane** (fast label swapping); and
**PHP (Penultimate Hop Popping)** — the second-to-last LSR usually pops the label so the
egress does a single lookup. The big win is that the **core forwards on labels and
never needs customer/Internet routes** — which is exactly what makes the VPN service
below possible.

[↩ back to README §2 MPLS](README.md#the-rest-of-section-2--mpls-describe-level)

---

## MPLS Layer 3 VPN

*(ENARSI §2.2 — describe.)* An **MPLS L3VPN** is a **provider-delivered** service that
gives a customer private any-to-any connectivity between sites, keeping every
customer's routes isolated — even when two customers use the **same** (overlapping)
private address space. It's the "you buy it" counterpart to the "you build it" DMVPN.

The building blocks, most of which you've already met:

- **VRF** (Virtual Routing and Forwarding) — on each **PE** (provider edge) router, each
  customer gets its own isolated routing table/interface set, so their routes never mix
  (this is exactly **VRF-lite**, extended across the provider — the next lab).
- **PE–CE routing** — the customer edge (**CE**) exchanges routes with the PE using
  static, OSPF, EIGRP, or BGP, into that customer's VRF.
- **MP-BGP with VPNv4** — PEs advertise customer routes to each other with **MP-BGP**
  (the address-family BGP from `../bgp`). To keep overlapping customer prefixes unique,
  each is prepended with a **Route Distinguisher (RD)**, becoming a **VPNv4** route.
- **Route Targets (RT)** — extended-community tags that control **which VRFs import
  which routes** (export RT on the way out, import RT on the way in) — that's how you
  build any-to-any, hub-and-spoke, or extranet topologies.
- **Two-label stack** — a VPN packet carries an **outer** label (the LDP **LSP** to the
  remote PE, swapped by the P routers) and an **inner** VPN label (identifies the
  egress VRF/CE, understood only by the remote PE). **The P routers only swap the outer
  label — they never see or hold customer routes**, which is what lets the provider
  scale to thousands of customers.

Put together: `../bgp` gave you MP-BGP, the next lab gives you VRFs, and MPLS labels
tie them into a provider VPN — L3VPN is those three ideas combined at carrier scale.

[↩ back to README §2 MPLS](README.md#the-rest-of-section-2--mpls-describe-level)
