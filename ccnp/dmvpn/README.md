# CCNP DMVPN Lab (Cisco IOL + Edgeshark)

A single-hub **Dynamic Multipoint VPN** built from scratch — **mGRE**, **NHRP**,
**dynamic spoke registration**, **spoke-to-spoke** tunnels, and **IPsec** protection.
One hub and two spokes ride over a simulated internet underlay. The routers boot with
**only physical interfaces and "public" IPs**: no tunnels, no NHRP, no routing, no
IPsec. You build the entire DMVPN overlay by hand and use **Edgeshark** to watch NHRP
registration, GRE encapsulation, and on-demand spoke-to-spoke resolution on the wire.

This lab is the **VPN section (§2.0)** of ENARSI: it *configures and verifies* **DMVPN
(§2.3)** hands-on, and its [`CONCEPTS.md`](CONCEPTS.md) also covers the **describe**
topics — **MPLS operations (§2.1)** and **MPLS Layer 3 VPN (§2.2)** — so the whole
section is here.

By the end you will have configured and observed:

- **mGRE** — a single multipoint tunnel that reaches every peer
- **NHRP** — spokes dynamically **register** their public (NBMA) address with the hub,
  and **resolve** each other on demand
- **Routing over DMVPN** (EIGRP) and the split-horizon / next-hop-self knobs it needs
- **Spoke-to-spoke** (Phase 3) — a direct tunnel forming on demand, seen on the wire
- **IPsec** tunnel protection (config + concept; see the image caveat)

> 📖 **This lab has a companion:** [`CONCEPTS.md`](CONCEPTS.md) — underlay/overlay,
> mGRE, NHRP, the three DMVPN phases, IPsec, **and the MPLS describe topics**. Each
> lab opens with a **Deep dive** callout.

---

## Topology

```
        hub LAN 192.168.1.0/24 (Lo1)          overlay: mGRE Tunnel0 172.16.0.0/24
                 R1  HUB (NHS)  172.16.0.1
                 │ e0/1  203.0.11.1  (public / NBMA)
                 │
          ┌──────┴───────┐
          │  ISP underlay │   routes the "public" /30s to each other
          └───┬───────┬───┘
       e0/2   │       │   e0/3
       203.0.12.1   203.0.13.1
          R2 SPOKE      R3 SPOKE
          172.16.0.2    172.16.0.3
      192.168.2.0/24    192.168.3.0/24
```

| Node | Public (NBMA) | Tunnel (overlay) | LAN            | Role            |
|------|---------------|------------------|----------------|-----------------|
| R1   | 203.0.11.1    | 172.16.0.1       | 192.168.1.0/24 | **Hub / NHS**   |
| R2   | 203.0.12.1    | 172.16.0.2       | 192.168.2.0/24 | Spoke           |
| R3   | 203.0.13.1    | 172.16.0.3       | 192.168.3.0/24 | Spoke           |
| ISP  | .2 on each /30| —                | —              | Underlay fixture |

Interface mapping: containerlab `ethX` → IOL `Ethernet0/X`; `E0/0` is management.

---

## Deploy

```bash
cd /home/lab/clabs/ccnp/dmvpn
sudo containerlab deploy -t dmvpn.clab.yml
```
Destroy with `sudo containerlab destroy -t dmvpn.clab.yml --cleanup`.

> ⚠️ **IPsec caveat.** The **mGRE + NHRP + routing overlay (Labs 1–3) is fully
> hands-on** and is where all the DMVPN learning is. **Lab 4 (IPsec `tunnel
> protection`)** needs a crypto engine the IOL image may not include — if `show
> crypto ikev2 sa` stays empty, treat that section as **config + concept**. DMVPN
> runs fine *without* IPsec (it's just unencrypted then), so Labs 1–3 are unaffected.

---

## Using Edgeshark for captures

Capture on the **ISP-facing** links (the NBMA/public side) to see the real magic —
GRE tunnels and NHRP over the underlay.

| Filter                         | Shows                                                  |
|--------------------------------|--------------------------------------------------------|
| `gre`                          | GRE-encapsulated tunnel traffic (IP protocol 47)       |
| `nhrp`                         | NHRP registration / resolution requests & replies      |
| `ip.addr == 203.0.12.1 && ip.addr == 203.0.13.1` | **spoke-to-spoke** traffic direct between spokes (Lab 3) |
| `esp`                          | IPsec-encrypted tunnel traffic (Lab 4, if supported)   |

> 📖 **Deep dive:** [What DMVPN is: underlay and overlay](CONCEPTS.md#what-dmvpn-is-underlay-and-overlay)

---

# Lab 0 — Underlay reachability

**Goal:** DMVPN is an **overlay** — it needs the **underlay** (the "internet") to
carry tunnel packets between public addresses first. Give the hub and spokes a default
route toward the ISP and confirm the publics can reach each other.

```
r1(config)# ip route 0.0.0.0 0.0.0.0 203.0.11.2
r2(config)# ip route 0.0.0.0 0.0.0.0 203.0.12.2
r3(config)# ip route 0.0.0.0 0.0.0.0 203.0.13.2
```
```
r2# ping 203.0.11.1              ! spoke's public -> hub's public, through the ISP
r3# ping 203.0.11.1
```
If those pings work, the NBMA underlay is ready. The tunnels ride on top of it.

---

# Lab 1 — mGRE + NHRP (spokes register with the hub)

**Goal:** build the multipoint tunnel and NHRP so spokes **register** their public
address with the hub — the DMVPN overlay comes alive.

> 📖 **Deep dive:** [mGRE: one tunnel, many endpoints](CONCEPTS.md#mgre-one-tunnel-many-endpoints)
> · [NHRP: resolution for the overlay](CONCEPTS.md#nhrp-resolution-for-the-overlay)

### 1.1 Hub tunnel (the NHS)

```
r1(config)# interface Tunnel0
r1(config-if)#  ip address 172.16.0.1 255.255.255.0
r1(config-if)#  tunnel source Ethernet0/1
r1(config-if)#  tunnel mode gre multipoint          ! mGRE -- one interface, many peers
r1(config-if)#  ip nhrp network-id 1
r1(config-if)#  ip nhrp map multicast dynamic        ! learn spokes' multicast destinations as they register
r1(config-if)#  ip nhrp redirect                     ! Phase 3: tell spokes to talk directly
r1(config-if)#  ip mtu 1400
r1(config-if)#  ip tcp adjust-mss 1360
```

### 1.2 Spoke tunnels (point at the NHS)

```
r2(config)# interface Tunnel0
r2(config-if)#  ip address 172.16.0.2 255.255.255.0
r2(config-if)#  tunnel source Ethernet0/1
r2(config-if)#  tunnel mode gre multipoint
r2(config-if)#  ip nhrp network-id 1
r2(config-if)#  ip nhrp nhs 172.16.0.1 nbma 203.0.11.1 multicast   ! hub = my Next Hop Server
r2(config-if)#  ip nhrp shortcut                                     ! Phase 3: install spoke-to-spoke shortcuts
r2(config-if)#  ip mtu 1400
r2(config-if)#  ip tcp adjust-mss 1360
```
(R3 identical, tunnel IP `172.16.0.3`, same NHS line.)

> **📸 CAPTURE — `r2` `Ethernet0/1`** (public side), filter `nhrp`. Watch the spoke
> send an **NHRP Registration Request** to the hub and get a **Reply** — that's the
> spoke telling the hub "my overlay 172.16.0.2 lives at public 203.0.12.1."

```
r1# show ip nhrp                 ! hub's NHRP table: 172.16.0.2->203.0.12.1, 172.16.0.3->203.0.13.1 (dynamic)
r1# show dmvpn                   ! hub sees 2 spokes
r2# show dmvpn                   ! spoke sees the hub (static NHS)
r2# ping 172.16.0.1              ! spoke -> hub across the tunnel
```

On a GRE capture you'll see the ping **encapsulated in GRE** (IP proto 47) between the
public addresses — the overlay packet wrapped inside an underlay packet.

> **Why `ip mtu`/`adjust-mss`:** GRE (and later IPsec) add headers, so the overlay MTU
> is smaller. Setting `ip mtu 1400` and clamping TCP MSS avoids fragmentation/PMTUD
> black-holes — a classic DMVPN "big transfers hang" fix.

---

# Lab 2 — Routing over DMVPN (EIGRP)

**Goal:** run a routing protocol *over* the tunnel so the LANs behind hub and spokes
reach each other — and meet the two DMVPN routing knobs.

> 📖 **Deep dive:** [Routing over DMVPN](CONCEPTS.md#routing-over-dmvpn)

```
r1(config)# router eigrp 100
r1(config-router)#  network 172.16.0.0 0.0.0.255
r1(config-router)#  network 192.168.1.0 0.0.0.255
r1(config)# interface Tunnel0
r1(config-if)#  no ip split-horizon eigrp 100       ! hub MUST re-advertise one spoke's routes to the others
r1(config-if)#  no ip next-hop-self eigrp 100        ! preserve the ORIGINATING spoke as next-hop (spoke-to-spoke)
```
```
r2(config)# router eigrp 100
r2(config-router)#  network 172.16.0.0 0.0.0.255
r2(config-router)#  network 192.168.2.0 0.0.0.255
```
(R3 identical with `192.168.3.0`.)

```
r2# show ip route eigrp          ! learns 192.168.1.0 (hub) AND 192.168.3.0 (the OTHER spoke)
r2# ping 192.168.1.1 source 192.168.2.1     ! spoke LAN -> hub LAN over DMVPN
r2# ping 192.168.3.1 source 192.168.2.1     ! spoke LAN -> spoke LAN (via hub for now)
```

> **The two knobs, and why:** a hub receives spoke routes on Tunnel0 and would
> normally **not** re-advertise them back out the same interface (**split horizon**) —
> so spokes would never learn each other's LANs. Disable it on the hub. And by default
> the hub would rewrite the **next-hop to itself**; `no ip next-hop-self` keeps the
> *originating spoke* as the next-hop, which is what lets Phase 2/3 build a **direct**
> spoke-to-spoke path instead of hair-pinning through the hub.

---

# Lab 3 — Spoke-to-spoke (Phase 3 on demand)

**Goal:** watch a **direct** spoke-to-spoke tunnel form the moment two spokes need to
talk — the feature that makes DMVPN "dynamic."

> 📖 **Deep dive:** [The three DMVPN phases](CONCEPTS.md#the-three-dmvpn-phases)
> · [NHRP: resolution for the overlay](CONCEPTS.md#nhrp-resolution-for-the-overlay)

### 3.1 Before: traffic hair-pins through the hub

```
r2# show dmvpn                   ! only the hub entry so far
```

### 3.2 Trigger it and watch the resolution

> **📸 CAPTURE — `r2` `Ethernet0/1`**, filter `nhrp || gre`. Then generate
> spoke-to-spoke traffic:

```
r2# ping 192.168.3.1 source 192.168.2.1 repeat 20
```

In order on the capture:
1. The **first** packets go R2 → **hub** → R3 (the only path R2 knows).
2. The hub sends R2 an **NHRP Redirect** ("you should talk to R3 directly").
3. R2 sends an **NHRP Resolution Request** for R3; R3 answers with its public address
   (203.0.13.1) in a **Resolution Reply**.
4. R2 installs a **shortcut** and subsequent packets go **directly** R2 (203.0.12.1) ↔
   R3 (203.0.13.1), GRE, **not through the hub**.

```
r2# show dmvpn                   ! now a DYNAMIC spoke-to-spoke entry to R3 (203.0.13.1)
r2# show ip nhrp                 ! resolved 172.16.0.3 -> 203.0.13.1
r2# traceroute 192.168.3.1 source 192.168.2.1    ! one overlay hop -- direct, hub no longer in the path
```

> **Dynamic neighbor:** neither spoke was pre-configured to know the other — the direct
> tunnel was created **on demand** from NHRP resolution and **torn down** when idle.
> That's the whole point of DMVPN: full-mesh connectivity with only hub-facing config
> on each spoke.

---

# Lab 4 — IPsec protection (config + concept)

**Goal:** encrypt the tunnels. DMVPN protects mGRE with an **IPsec profile** applied
as **`tunnel protection`** — one profile secures *all* tunnels (hub-spoke and the
dynamic spoke-to-spoke), which is why DMVPN scales where classic crypto maps don't.

> 📖 **Deep dive:** [IPsec protection](CONCEPTS.md#ipsec-protection)

```
r1(config)# crypto ikev2 keyring KR
r1(config-ikev2-keyring)#  peer ANY
r1(config-ikev2-keyring-peer)#   address 0.0.0.0 0.0.0.0
r1(config-ikev2-keyring-peer)#   pre-shared-key DMVPNKEY123
!
r1(config)# crypto ikev2 profile IKEV2-PROF
r1(config-ikev2-profile)#  match identity remote address 0.0.0.0
r1(config-ikev2-profile)#  authentication local pre-share
r1(config-ikev2-profile)#  authentication remote pre-share
r1(config-ikev2-profile)#  keyring local KR
!
r1(config)# crypto ipsec transform-set TS esp-aes 256 esp-sha256-hmac
r1(config-crypto-trans)#  mode transport
!
r1(config)# crypto ipsec profile IPSEC-PROF
r1(config-ipsec-profile)#  set transform-set TS
r1(config-ipsec-profile)#  set ikev2-profile IKEV2-PROF
!
r1(config)# interface Tunnel0
r1(config-if)#  tunnel protection ipsec profile IPSEC-PROF
```
(apply the **same** profile + `tunnel protection` on R2 and R3.)

```
r1# show crypto ikev2 sa          ! IKEv2 SAs to each spoke (if the image supports crypto)
r1# show crypto ipsec sa           ! encrypt/decrypt counters climbing
r1# show dmvpn detail              ! per-peer tunnel + crypto state
```

> **📸 CAPTURE — `r2` `Ethernet0/1`**, filter `esp`. With protection on, the GRE is now
> inside **ESP** — the overlay payload is encrypted on the wire.
>
> ⚠️ If IKEv2/IPsec SAs never form on your IOL image, that's the crypto-engine
> limitation — the **config above is exam-correct**, and Labs 1–3 already proved the
> DMVPN mechanics. In production the transport-mode ESP wraps every DMVPN tunnel with
> a single profile.

---

# The rest of Section 2 — MPLS (describe-level)

ENARSI §2.1 and §2.2 are **describe** topics — you won't build MPLS here (it's a
provider-scale technology), but you must understand it. Both are covered in depth in
the companion:

> 📖 **Deep dive:** [MPLS operations (LSR, LDP, LSP, label switching)](CONCEPTS.md#mpls-operations)
> · [MPLS Layer 3 VPN](CONCEPTS.md#mpls-layer-3-vpn)

The one-line contrast to anchor them: **DMVPN** is a **customer-built** overlay across
*any* IP transport (often the internet); **MPLS L3VPN** is a **provider-built** service
where the carrier's network keeps each customer's routes separate in **VRFs** and
carries them with **MP-BGP + labels**. Both give you private any-to-any connectivity
between sites — one you run, one you buy.

---

## Suggested end-state verification

```
r1# show dmvpn                    ! hub: 2 spokes, dynamic
r2# show dmvpn                    ! spoke: hub (static) + any spoke-to-spoke (dynamic)
r1# show ip nhrp                  ! NBMA<->overlay mappings
r2# show ip route eigrp           ! other spoke's LAN, next-hop = originating spoke
r2# traceroute 192.168.3.1 source 192.168.2.1   ! direct spoke-to-spoke (no hub hop)
r1# show crypto ipsec sa          ! (Lab 4) encryption, if supported
```

## DMVPN phase cheat-sheet

| | Spoke tunnel | Spoke-to-spoke | Hub next-hop | Summarization |
|---|---|---|---|---|
| **Phase 1** | p2p GRE | ✗ (all via hub) | rewrites to hub | ✓ |
| **Phase 2** | mGRE | ✓ (direct) | must preserve | ✗ (spokes need specifics) |
| **Phase 3** | mGRE | ✓ (via NHRP redirect/shortcut) | preserved | ✓ (redirect fixes it) |

This lab builds **Phase 3** (mGRE + `nhrp redirect` on the hub + `nhrp shortcut` on
spokes) — the modern default.

---

## Reset the whole lab

```bash
sudo containerlab destroy -t dmvpn.clab.yml --cleanup
sudo containerlab deploy  -t dmvpn.clab.yml
```

Because the routers hold no overlay config, every redeploy is a clean slate.

---

## Where this fits

This is the **VPN section (§2.0)**: DMVPN (§2.3) hands-on, plus MPLS (§2.1/§2.2)
described in CONCEPTS. It leans on tunnels + NHRP (new here) and EIGRP (from
`../eigrp`). With this, only **PBR + IP SLA + tracking + BFD** and **VRF-Lite** remain
before the **enterprise capstone**, where DMVPN branches hang off the core.
