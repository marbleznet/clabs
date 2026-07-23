# CCNP OSPF Deep-Dive Lab (Cisco IOL + Edgeshark)

A 5-router + 1-switch `cisco_iol` containerlab topology for learning **OSPFv2**
from the packet up. The routers boot with **only interfaces and IP addresses** —
**no OSPF, no routing config at all**. You build every concept by hand and use
**Edgeshark packet captures** at each step to *see* what OSPF is doing on the
wire.

By the end you will have configured and observed:

- OSPF process enablement and the **neighbor FSM** (Down → Init → 2-Way →
  ExStart → Exchange → Loading → Full)
- **DR/BDR election** on a real broadcast segment (and how to influence it)
- **All four network types** — broadcast, point-to-point, **nonbroadcast (NBMA)**,
  and **point-to-multipoint** — and their DR/timer/discovery differences
- **Multi-area** design, **ABRs**, and inter-area (Type-3 summary) LSAs
- **Redistribution** at an **ASBR** — Type-5 external + Type-4 ASBR-summary LSAs,
  and **E1 vs E2** metrics
- **Stub / totally-stubby / NSSA / totally-NSSA** areas (Type-7 translation) and
  ABR-injected default routes
- **Virtual links** across a **transit area** to repair a disconnected area
- **Path preference** (intra > inter > E1 > E2) and administrative distance
- **OSPFv3 for IPv6** using the address-family model (Type-8/9 LSAs)
- Cost, timers, and authentication

> 📖 **This lab has a companion:** [`CONCEPTS.md`](CONCEPTS.md) is the in-depth
> reference behind every step (the link-state model, the neighbor FSM, LSA types,
> area types, path preference, OSPFv3 internals, and more). Each lab opens with a
> **Deep dive** callout linking to the relevant sections — follow them for the
> *why*, skip them when you just want to type. It also reads well end-to-end as an
> OSPF primer.

---

## Topology

```
                       Area 1  (normal, later STUB)
        R4 (4.4.4.4)
        Lo1 172.16.40.0/24
            |
            | 10.0.14.0/30   point-to-point (no DR/BDR)
            |
        R1 (1.1.1.1)  ── ABR ──┐
            |                   |
    ┌───────┴───────┐          Area 0 backbone
    │   SW1 (L2)    │          BROADCAST LAN 10.0.0.0/24
    │  flat switch  │          (DR/BDR election happens here)
    └───┬───────┬───┘
        |       |
    R2 (2.2.2.2)  R3 (3.3.3.3)
     ABR           ASBR  Lo1 172.16.30.0/24 + static routes
      |                  (redistributed -> Type-5 externals)
      | 10.0.25.0/30  point-to-point (no DR/BDR)
      |
    R5 (5.5.5.5)
    Lo1 172.16.50.0/24
      Area 2  (normal)
```

**Why a switch?** R1, R2 and R3 all connect to **SW1**, a zero-config L2 IOL
switch, so they share one true Ethernet **broadcast domain**. DR/BDR election
only happens on broadcast/NBMA networks — you cannot demonstrate it on a
point-to-point link. The R1–R4 and R2–R5 links are point-to-point on purpose, to
contrast the two network types.

### Roles

| Router | Router-ID | Area membership        | Role                                    |
|--------|-----------|------------------------|-----------------------------------------|
| R1     | 1.1.1.1   | Area 0 + Area 1        | **ABR** (backbone ↔ Area 1)             |
| R2     | 2.2.2.2   | Area 0 + Area 2        | **ABR** (backbone ↔ Area 2)             |
| R3     | 3.3.3.3   | Area 0                 | **ASBR** (redistributes externals)      |
| R4     | 4.4.4.4   | Area 1                 | Internal router                         |
| R5     | 5.5.5.5   | Area 2                 | Internal router                         |
| SW1    | —         | —                      | L2 switch (broadcast segment for Area 0)|

### Addressing

| Link / Interface        | IPv4 Network     | IPv6 Network      | Area   | Type            |
|-------------------------|------------------|-------------------|--------|-----------------|
| R1/R2/R3 ↔ SW1 (e0/1)   | 10.0.0.0/24      | 2001:db8:0::/64   | 0      | broadcast (LAN) |
| R1 e0/2 ↔ R4 e0/1       | 10.0.14.0/30     | 2001:db8:14::/64  | 1      | point-to-point  |
| R2 e0/2 ↔ R5 e0/1       | 10.0.25.0/30     | 2001:db8:25::/64  | 2      | point-to-point  |
| R1 Lo0                  | 1.1.1.1/32       | —                 | 0      | loopback        |
| R2 Lo0                  | 2.2.2.2/32       | —                 | 0      | loopback        |
| R3 Lo0                  | 3.3.3.3/32       | —                 | 0      | loopback        |
| R3 Lo1 (external)       | 172.16.30.0/24   | 2001:db8:30::/64  | —      | redistributed   |
| R4 Lo0 / Lo1            | 4.4.4.4/32 / 172.16.40.0/24 | 2001:db8:40::/64 | 1 | loopback  |
| R5 Lo0 / Lo1            | 5.5.5.5/32 / 172.16.50.0/24 | 2001:db8:50::/64 | 2 | loopback  |

The `e0/X` interfaces are **dual-stacked** — IPv4 for OSPFv2 (Labs 1–10) and IPv6
for OSPFv3 (Lab 11). Host bits match the router number (R4 = `...::4`). Lo0 stays
IPv4-only (it is the manual 32-bit router-id source for both v2 and v3).

Interface mapping: containerlab `ethX` → IOL `Ethernet0/X`. `E0/0` is management.
So `e0/1` = `Ethernet0/1`, `e0/2` = `Ethernet0/2`.

---

## Deploy

```bash
cd /home/lab/clabs/ccnp/ospf
sudo containerlab deploy -t ospf-lab.clab.yml
```

Log in to any node with `admin` / `admin`:

```bash
ssh admin@clab-ospf-lab-r1        # or: docker exec -it clab-ospf-lab-r1 telnet localhost
```

Give the nodes ~60–90s after deploy before the CLI is fully ready. Destroy with:

```bash
sudo containerlab destroy -t ospf-lab.clab.yml --cleanup
```

### Sanity check before you start (no OSPF yet)

```
r1# show ip interface brief          ! e0/1, e0/2, Lo0 up/up
r1# show ip route                    ! only connected + local routes
r1# show ip ospf                     ! "OSPF ... not enabled" -- correct, blank slate
```

---

## Using Edgeshark for captures

Edgeshark (Ghostwire + Packetflix) lets you capture on any container interface
straight from the browser:

1. Open the **Edgeshark / Ghostwire** web UI in your browser.
2. Find the container (e.g. `clab-ospf-lab-r1`) and expand its interfaces.
3. Click the **capture** (shark-fin) icon next to the interface you want
   (e.g. `Ethernet0/1`). A browser-based **Wireshark** session opens and streams
   live packets.

Throughout this guide, each **📸 CAPTURE** block tells you:
- **which node + interface** to capture on,
- **when** to start it (usually *before* the config step that triggers traffic),
- **what to look for** (with a Wireshark display filter).

Handy OSPF display filters:

| Filter                    | Shows                                        |
|---------------------------|----------------------------------------------|
| `ospf`                    | all OSPF packets                             |
| `ospf.msg == 1`           | **Hello**                                    |
| `ospf.msg == 2`           | **DB Description (DBD)**                      |
| `ospf.msg == 3`           | **LS Request (LSR)**                          |
| `ospf.msg == 4`           | **LS Update (LSU)** — carries LSAs           |
| `ospf.msg == 5`           | **LS Acknowledgment (LSAck)**                |
| `ospf.lsa.type == 1`      | Router LSA (Type 1)                          |
| `ospf.lsa.type == 2`      | Network LSA (Type 2, DR-originated)          |
| `ospf.lsa.type == 3`      | Summary LSA (Type 3, inter-area)             |
| `ospf.lsa.type == 4`      | ASBR-Summary LSA (Type 4)                    |
| `ospf.lsa.type == 5`      | AS-External LSA (Type 5)                     |

OSPF multicast MACs/IPs to recognize:
- **224.0.0.5** = *AllSPFRouters* (all OSPF speakers)
- **224.0.0.6** = *AllDRouters* (only DR/BDR listen)

> 📖 **Deep dive:** [The link-state model and the LSDB](CONCEPTS.md#the-link-state-model-and-the-lsdb)
> · [The LSA types](CONCEPTS.md#the-lsa-types) — what each captured LSA means

**OSPFv3 (Lab 11)** is the same `ospf` dissector but **version 3**: it rides IP
protocol **89** over IPv6, multicasts to **FF02::5 / FF02::6**, and sources every
packet from the interface **link-local** (`fe80::`) address. Add `ospf.lsa.type==8`
(Link-LSA) and `ospf.lsa.type==9` (Intra-Area-Prefix-LSA) for the v3-only LSAs.

> Tip: OSPF Hellos every 10s (broadcast) can bury the interesting packets. Start
> the capture, apply the config, and use the message-type filters above to isolate
> the adjacency-building exchange.

---

# Lab 1 — Bring up the first adjacency (Area 0 LAN)

**Goal:** enable OSPF on R1, R2, R3 (the shared LAN) and watch the neighbor FSM,
Hellos, and the database exchange (DBD/LSR/LSU/LSAck) on the wire.

> 📖 **Deep dive:** [The link-state model and the LSDB](CONCEPTS.md#the-link-state-model-and-the-lsdb)
> · [The neighbor state machine](CONCEPTS.md#the-neighbor-state-machine)
> · [Hello packets and what must match](CONCEPTS.md#hello-packets-and-what-must-match)
> · [Database synchronization](CONCEPTS.md#database-synchronization-dbd-lsr-lsu-lsack)

### 1.1 Start the capture FIRST

> **📸 CAPTURE — `r1` interface `Ethernet0/1`** (the Area 0 LAN).
> Start this *before* you configure OSPF so you catch the very first Hello and the
> entire adjacency build. Filter: `ospf`.

### 1.2 Enable OSPF on R1 only

```
r1# conf t
r1(config)# router ospf 1
r1(config-router)#  router-id 1.1.1.1
r1(config-router)#  network 10.0.0.0 0.0.0.255 area 0
r1(config-router)#  network 1.1.1.1 0.0.0.0 area 0
r1(config-router)# end
```

> The `network <addr> <wildcard> area` command is match-by-wildcard: any
> interface whose IP falls in the range is activated in that area. `1.1.1.1
> 0.0.0.0` matches exactly the loopback. (Equivalent modern syntax is per-interface
> `ip ospf 1 area 0`.)

**On the capture now:** with only R1 running you'll see R1 sending **Hellos** to
**224.0.0.5** every 10s. Open one Hello and note:
- `Active Neighbor` list is **empty** (R1 has no neighbors yet),
- `Designated Router = 10.0.0.1`, `Backup DR = 0.0.0.0` — R1 elected *itself* DR
  because it's alone,
- Hello Interval 10, Dead Interval 40, Area, and **Network Mask** — these must
  match or neighbors won't form.

### 1.3 Enable OSPF on R2, then R3

```
r2# conf t
r2(config)# router ospf 1
r2(config-router)#  router-id 2.2.2.2
r2(config-router)#  network 10.0.0.0 0.0.0.255 area 0
r2(config-router)#  network 2.2.2.2 0.0.0.0 area 0
r2(config-router)# end
```

```
r3# conf t
r3(config)# router ospf 1
r3(config-router)#  router-id 3.3.3.3
r3(config-router)#  network 10.0.0.0 0.0.0.255 area 0
r3(config-router)#  network 3.3.3.3 0.0.0.0 area 0
r3(config-router)# end
```

Watch the adjacencies form:

```
r1# show ip ospf neighbor
```

You want to see R2 and R3 reach **FULL**. On a broadcast segment the state may
settle as `FULL/DR`, `FULL/BDR`, or `2WAY/DROTHER` (see Lab 2 — DROther routers
stay **2-WAY** with each other, which is normal and *not* a fault).

### 1.4 Read the adjacency off the capture (the payoff)

Filter the capture to the moment R2/R3 came up. In order you should see:

1. **Hello (`ospf.msg==1`)** — now the neighbor's RID appears in the
   `Active Neighbor` list. Seeing your own RID listed in a neighbor's Hello is
   what moves you from **Init → 2-Way** (bidirectional communication confirmed).
2. **DBD (`ospf.msg==2`)** — **ExStart/Exchange**. Look at the flags:
   - **I (Init), M (More), MS (Master/Slave)** bits. The router with the higher
     RID becomes **Master**. First DBDs are empty/negotiation; then each side
     sends DBDs that are *headers only* (LSA summaries, not full LSAs).
   - Note the **Interface MTU** field — a mismatch here is the classic cause of
     an adjacency stuck in **ExStart/Exchange**.
3. **LSR (`ospf.msg==3`)** — **Loading**. Each router requests the specific LSAs
   it noticed it was missing from the DBD headers.
4. **LSU (`ospf.msg==4`)** — the actual **LSAs** are delivered here. Expand and
   find **Router-LSA (Type 1)** entries for 1.1.1.1 / 2.2.2.2 / 3.3.3.3.
5. **LSAck (`ospf.msg==5`)** — reliable flooding: every LSA is acknowledged.

> **Concept:** OSPF neighbors *discover* each other with Hellos, but the database
> is synchronized with the reliable DBD → LSR → LSU → LSAck exchange. Only after
> databases match do the routers declare **FULL**.

### 1.5 Verify

```
r1# show ip ospf neighbor            ! R2 & R3 present
r1# show ip route ospf               ! learn 2.2.2.2/32 and 3.3.3.3/32 (O routes)
r1# show ip ospf database            ! three Router LSAs in area 0
```

---

# Lab 2 — DR/BDR election (and how to rig it)

**Goal:** understand who becomes DR/BDR on the LAN and why, and force a
re-election.

> 📖 **Deep dive:** [DR and BDR election](CONCEPTS.md#dr-and-bdr-election)
> · [The neighbor state machine](CONCEPTS.md#the-neighbor-state-machine) (why DROthers rest at 2-Way)

### 2.1 Observe the current election

```
r1# show ip ospf neighbor
r1# show ip ospf interface Ethernet0/1
```

With default priority (1) everywhere, election is by **highest Router-ID**:
- **DR = R3 (3.3.3.3)**, **BDR = R2 (2.2.2.2)**, **R1 = DROTHER**.

`show ip ospf interface e0/1` on R1 confirms its own state (`DROTHER`) and lists
the `Designated Router (ID) 3.3.3.3` and `Backup ... 2.2.2.2`.

> **📸 CAPTURE — `r3` interface `Ethernet0/1`.** Filter `ospf.msg==1`. Open a
> Hello from R3 and confirm `Designated Router = 10.0.0.3` (R3's LAN IP) and
> `Backup Designated Router = 10.0.0.2`. Also note traffic to **224.0.0.6**
> (AllDRouters) — that address is how DROthers send updates to *just* the DR/BDR.

### 2.2 Why DROthers stay 2-WAY

```
r1# show ip ospf neighbor
```

Between the two DROthers (here R1 sees the DR R3 and BDR R2, but if you had a 4th
DROther) the state is **2WAY/DROTHER** — **by design**. DROthers form FULL
adjacencies *only* with the DR and BDR, not with each other. This is the whole
point of a DR: it reduces a full mesh of adjacencies to a star.

### 2.3 Force R1 to become DR

DR election is **non-preemptive** — a higher-priority router joining later will
*not* take over. You must clear the process (or bounce the interface) after
changing priority.

```
r1(config)# interface Ethernet0/1
r1(config-if)#  ip ospf priority 255
r1(config-if)# end
```

> **📸 CAPTURE — `r1` interface `Ethernet0/1`** *before* the next command.
> Filter `ospf`.

```
r1# clear ip ospf process          ! confirm with 'yes' -- also do on r2/r3 to speed it up
```

On the capture watch the routers renegotiate: Hellos with updated DR/BDR fields,
a fresh DBD/LSU exchange, and a **new Type-2 Network LSA** now originated by the
new DR. Verify:

```
r1# show ip ospf neighbor          ! R1 now DR (255 wins)
r1# show ip ospf database network  ! Network LSA now advertised by 1.1.1.1
```

### 2.4 The Network (Type-2) LSA — the DR's job

```
r1# show ip ospf database network
```

Only the **DR** originates a **Network LSA (Type 2)**, which lists every router on
the segment. On a broadcast network this is how the LAN is represented in the
topology graph (a "pseudonode").

> **📸 CAPTURE — `r1` interface `Ethernet0/1`.** Filter `ospf.lsa.type==2`. Trigger
> a reflood with `clear ip ospf process` and confirm the Network LSA's
> *Advertising Router* is the current DR, and its *Attached Router* list contains
> 1.1.1.1, 2.2.2.2, 3.3.3.3.

> **Optional:** set `ip ospf priority 0` on R1 to make it **ineligible** — it will
> never be DR/BDR. Re-clear and observe R3/R2 retake the roles.

Reset R1 priority back to default when done (`no ip ospf priority`), clear, and
move on.

---

# Lab 3 — Point-to-point network type (R1 ↔ R4)

**Goal:** contrast the LAN with a point-to-point link — **no DR/BDR**, faster
adjacency, different Hello timers.

> 📖 **Deep dive:** [Network types in depth](CONCEPTS.md#network-types-in-depth)

### 3.1 Enable OSPF across the R1–R4 link *in Area 0 for now*

We'll put R4 in its own area in Lab 4; first, just see the p2p behavior. Add the
link to Area 0 temporarily is *not* valid (it would be a discontiguous area), so
we go straight to Area 1 here and treat R1 as an ABR.

```
r1(config)# router ospf 1
r1(config-router)#  network 10.0.14.0 0.0.0.3 area 1
```

```
r4# conf t
r4(config)# router ospf 1
r4(config-router)#  router-id 4.4.4.4
r4(config-router)#  network 10.0.14.0 0.0.0.3 area 1
r4(config-router)#  network 4.4.4.4 0.0.0.0 area 1
r4(config-router)#  network 172.16.40.0 0.0.0.255 area 1
r4(config-router)# end
```

> **📸 CAPTURE — `r1` interface `Ethernet0/2`** (start before the R4 config).
> Filter `ospf`.

### 3.2 What's different on a point-to-point link

Open a Hello (`ospf.msg==1`) on `e0/2` and compare with the LAN Hellos:
- **Designated Router = 0.0.0.0** and **Backup = 0.0.0.0** — there is **no DR/BDR**
  on point-to-point.
- The neighbor goes to **FULL** directly (no DROther/2-Way holding pattern).

```
r1# show ip ospf neighbor          ! R4 shows FULL/  -  (no DR role after state)
r1# show ip ospf interface Ethernet0/2   ! "Network Type POINT_TO_POINT"
```

> **Why?** IOL Ethernet defaults to network type **broadcast**, but IOS treats a
> `/30`... actually it does *not* auto-detect — Ethernet is broadcast by default
> even on a /30. If your `e0/2` shows `BROADCAST` with a DR elected, that's the
> teaching point: run `ip ospf network point-to-point` on **both** R1 e0/2 and R4
> e0/1, then re-capture and watch the DR fields drop to 0.0.0.0 and the adjacency
> re-form without an election.

```
r1(config)# interface Ethernet0/2
r1(config-if)#  ip ospf network point-to-point
r4(config)# interface Ethernet0/1
r4(config-if)#  ip ospf network point-to-point
```

---

# Lab 4 — Multiple areas, ABRs, and Type-3 summary LSAs

**Goal:** see how areas partition the link-state database and how ABRs glue them
together with **Type-3 Summary LSAs**. R1 is already an ABR (Area 0 + Area 1);
now make R2 an ABR into Area 2.

> 📖 **Deep dive:** [Areas: why OSPF partitions the topology](CONCEPTS.md#areas-why-ospf-partitions-the-topology)
> · [The LSA types](CONCEPTS.md#the-lsa-types)
> · [Router types](CONCEPTS.md#router-types)

### 4.1 Bring up Area 2 behind R2

```
r2(config)# router ospf 1
r2(config-router)#  network 10.0.25.0 0.0.0.3 area 2
```

```
r5# conf t
r5(config)# router ospf 1
r5(config-router)#  router-id 5.5.5.5
r5(config-router)#  network 10.0.25.0 0.0.0.3 area 2
r5(config-router)#  network 5.5.5.5 0.0.0.0 area 2
r5(config-router)#  network 172.16.50.0 0.0.0.255 area 2
r5(config-router)# end
```

Set both ends of R2–R5 to point-to-point as in Lab 3 if they come up broadcast.

### 4.2 Confirm the ABRs

```
r1# show ip ospf                   ! "It is an area border router"
r1# show ip ospf border-routers
r2# show ip ospf                   ! also an ABR
```

### 4.3 Inter-area routes and the Type-3 Summary LSA

From R4 (deep in Area 1), you should now be able to reach R5's Area 2 LAN — the
route crosses **Area 1 → (R1) → Area 0 → (R2) → Area 2**:

```
r4# show ip route ospf
```

Look for **O IA** routes (inter-area) such as `172.16.50.0/24` and `10.0.25.0/30`.

> **How it works:** an internal router's LSDB does **not** contain other areas'
> Router/Network LSAs. Instead, each **ABR** takes the networks it learned in one
> area and re-advertises them into the adjacent area as **Type-3 Summary LSAs**
> (prefix + cost, no topology detail). This is why OSPF scales: areas hide
> topology from each other.

```
r4# show ip ospf database summary            ! Type-3 LSAs injected by ABR R1 (1.1.1.1)
r1# show ip ospf database summary             ! R1 as ABR originates these
```

> **📸 CAPTURE — `r1` interface `Ethernet0/2`** (Area 1 side). Filter
> `ospf.lsa.type==3`. Trigger a reflood (e.g. `clear ip ospf process` on R2 to
> restate Area 2 prefixes, or bounce R5's Lo1). In the LSU you'll see **Summary
> LSAs** whose *Advertising Router* is the ABR **1.1.1.1**, carrying Area 0/Area 2
> prefixes into Area 1 — with a **metric** but no per-link topology. Contrast this
> with the **Type-1 Router LSAs** you saw in Lab 1 that *did* carry link detail.

### 4.4 Prove areas hide topology

```
r4# show ip ospf database             ! Area 1: Router/Network LSAs are LOCAL only
                                      ! everything else arrives as Summary (Type-3)
r3# show ip ospf database             ! Area 0 backbone sees more
```

R4 has **no** Router LSA for R5 — only a Type-3 summary of R5's prefixes. That's
the encapsulation boundary an area provides.

---

# Lab 5 — Redistribution: making R3 an ASBR (Type-5 + Type-4 LSAs)

**Goal:** inject non-OSPF routes (connected + static) into OSPF, turning R3 into
an **ASBR**, and watch **AS-External (Type-5)** and **ASBR-Summary (Type-4)**
LSAs propagate. Learn **E1 vs E2** metrics.

> 📖 **Deep dive:** [External routes: E1 vs E2, Type-4, forwarding address](CONCEPTS.md#external-routes-e1-vs-e2-type-4-forwarding-address)
> · [The LSA types](CONCEPTS.md#the-lsa-types)

### 5.1 Create the "external" routes on R3

R3's `Lo1 172.16.30.0/24` is *not* in OSPF (it was never in a `network`
statement). Add a couple of static routes too, to represent routes from another
protocol/domain:

```
r3# conf t
r3(config)# ip route 192.168.10.0 255.255.255.0 Null0
r3(config)# ip route 192.168.11.0 255.255.255.0 Null0
r3(config)# end
```

### 5.2 Redistribute — the `subnets` keyword matters

> **📸 CAPTURE — `r3` interface `Ethernet0/1`** (start before redistributing).
> Filter `ospf.lsa.type==5`.

```
r3(config)# router ospf 1
r3(config-router)#  redistribute connected subnets      ! brings in Lo1 172.16.30.0/24
r3(config-router)#  redistribute static subnets         ! brings in the two 192.168.x nets
r3(config-router)# end
```

> **Classic gotcha:** without `subnets`, IOS only redistributes *classful*
> networks and silently drops the /24 subnets. Try it without `subnets` first,
> run `show ip ospf database external`, see nothing useful, then add `subnets`.

### 5.3 Watch the external LSAs flood

On the capture you'll see **LSU** packets carrying **AS-External LSA (Type 5)**
entries, *Advertising Router 3.3.3.3*, one per redistributed prefix. Type-5 LSAs
**flood unchanged across all normal areas** (they are not summarized by ABRs).

```
r3# show ip ospf database external
r1# show ip ospf database external          ! same Type-5s, verbatim, in Area 0
r5# show ip route ospf                       ! O E2 routes for 172.16.30/192.168.10/11
```

### 5.4 The Type-4 ASBR-Summary LSA (how far-away areas find the ASBR)

R4 and R5 aren't in R3's area. For them to use R3's externals, their ABR must tell
them *how to reach the ASBR*. That's the **Type-4 ASBR-Summary LSA**:

```
r5# show ip ospf database asbr-summary       ! ABR R2 (2.2.2.2) advertises reachability to ASBR 3.3.3.3
r4# show ip ospf database asbr-summary
```

> **📸 CAPTURE — `r2` interface `Ethernet0/2`** (Area 2 side). Filter
> `ospf.lsa.type==4`. The *Advertising Router* is ABR **2.2.2.2**; the LSA's
> content identifies the **ASBR 3.3.3.3**. Type-5 says *"here is an external
> prefix, origin ASBR 3.3.3.3"*; Type-4 says *"here's how to reach ASBR
> 3.3.3.3"*. You need both to install the external route.

### 5.5 E2 vs E1 — the metric difference

Default external metric type is **E2**: the cost is the redistributed metric
**only**, ignoring the internal OSPF cost to reach the ASBR (so the metric is the
same everywhere).

```
r5# show ip route 172.16.30.0
```

Note `O E2` and that the metric doesn't grow with distance. Now switch to **E1**:

```
r3(config)# router ospf 1
r3(config-router)#  redistribute connected subnets metric-type 1
r3(config-router)#  redistribute static subnets metric-type 1
```

```
r5# show ip route 172.16.30.0          ! now O E1, metric = external + internal path cost
r4# show ip route 172.16.30.0          ! E1 metric differs by distance from the ASBR
```

> **E1** adds the internal cost to reach the ASBR, so different routers see
> different (more accurate) metrics — useful when there are multiple ASBRs and you
> want the closest one. Confirm on the capture (`ospf.lsa.type==5`) that the LSA's
> **E-bit** / metric-type flips between the two runs.

---

# Lab 6 — Stub and totally-stubby areas

**Goal:** shrink an area's database by blocking external (and inter-area) LSAs and
replacing them with an ABR-injected default route. We'll make **Area 1** a stub.

> 📖 **Deep dive:** [Area types: stub, totally-stubby, NSSA](CONCEPTS.md#area-types-stub-totally-stubby-nssa)

### 6.1 Baseline — what R4 sees now

```
r4# show ip ospf database external      ! R4 currently HAS the Type-5 externals
r4# show ip route ospf                   ! O E2 routes present
```

### 6.2 Configure the stub (BOTH routers in the area)

Every router in a stub area must agree (the **E-bit in the Hello** must match) or
the adjacency drops.

> **📸 CAPTURE — `r1` interface `Ethernet0/2`** before configuring. Filter `ospf`.

```
r1(config)# router ospf 1
r1(config-router)#  area 1 stub
```

At this point the R1–R4 adjacency will **drop** (mismatched stub flag) until you
match it on R4 — a great thing to watch on the capture:

```
r4(config)# router ospf 1
r4(config-router)#  area 1 stub
```

The adjacency re-forms. Open a Hello on the capture and find the **Options** field
— the **E-bit is now clear (0)**, signalling "external routing capability
disabled" (stub). A mismatched E-bit is exactly why the neighbor dropped.

### 6.3 What changed in R4's database

```
r4# show ip ospf database external      ! GONE -- no Type-5 in a stub area
r4# show ip route ospf
```

R4 lost the externals but gained a single **default route** `O*IA 0.0.0.0/0`
injected automatically by ABR **R1**. Reachability to the externals is preserved
via the default, but R4's database is much smaller.

```
r4# show ip route 172.16.30.0          ! now resolved via the default route, not a specific O E2
```

### 6.4 Totally-stubby (Cisco) — block Type-3 too

Make the ABR block inter-area summaries as well (only R1 needs `no-summary`):

```
r1(config)# router ospf 1
r1(config-router)#  area 1 stub no-summary
```

```
r4# show ip ospf database summary       ! Type-3 summaries GONE too
r4# show ip route ospf                    ! only the single O*IA default remains
```

> **Now R4's entire view of the OSPF world is one default route.** That's the most
> aggressive standard area type. Mentally note the ladder:
> **normal → stub (no Type-5) → totally-stubby (no Type-5 or Type-3) → NSSA**
> (stub that still allows a *local* ASBR via Type-7). **Lab 8** builds the NSSA and
> totally-NSSA on Area 2 in full.

### 6.5 Restore

```
r1(config-router)# no area 1 stub no-summary
r4(config-router)# no area 1 stub
```

---

# Lab 7 — More network types: nonbroadcast (NBMA) & point-to-multipoint

**Goal:** cover the two OSPF network types Labs 1–3 skipped —
**nonbroadcast (NBMA)** and **point-to-multipoint** — on the Area 0 LAN
(R1/R2/R3 on SW1). ENARSI expects you to recognize **all four** types and the
DR / timer / neighbor-discovery differences between them.

> 📖 **Deep dive:** [Network types in depth](CONCEPTS.md#network-types-in-depth)
> · [DR and BDR election](CONCEPTS.md#dr-and-bdr-election) (why NBMA pins the DR to the hub)

### 7.1 The four network types at a glance

| Network type          | DR/BDR? | Neighbor discovery      | Default Hello/Dead | Set with                          |
|-----------------------|---------|-------------------------|--------------------|-----------------------------------|
| broadcast             | **yes** | automatic (multicast)   | 10 / 40            | `ip ospf network broadcast`       |
| non-broadcast (NBMA)  | **yes** | **manual `neighbor`**   | **30 / 120**       | `ip ospf network non-broadcast`   |
| point-to-point        | no      | automatic (multicast)   | 10 / 40            | `ip ospf network point-to-point`  |
| point-to-multipoint   | no      | automatic (multicast)   | **30 / 120**       | `ip ospf network point-to-multipoint` |

Two things that trip people up: the **timers differ by type** (30/120 for NBMA &
p2mp), so a type mismatch also mismatches Hello/Dead and drops the adjacency; and
**NBMA needs static `neighbor` statements** because there is no multicast to
discover peers.

### 7.2 Make the Area 0 LAN nonbroadcast (NBMA)

Clear any priority tweaks from Lab 2 first. Then on **all three** LAN routers
change the type — and because NBMA has no multicast discovery, statically define
neighbors. In NBMA the **DR must be a well-connected, always-up router**, so make
one router the "hub" (high priority) and the others ineligible (priority 0) —
this mirrors a real frame-relay hub-and-spoke.

> **📸 CAPTURE — `r1` interface `Ethernet0/1`.** Filter `ospf`. Watch the Hellos
> change from multicast (224.0.0.5) to **unicast** to each configured neighbor.

R1 (the hub):
```
r1(config)# interface Ethernet0/1
r1(config-if)#  ip ospf network non-broadcast
r1(config-if)#  ip ospf priority 10
r1(config)# router ospf 1
r1(config-router)#  neighbor 10.0.0.2
r1(config-router)#  neighbor 10.0.0.3
```

R2 and R3 (spokes — non-broadcast, never DR, point back at the hub):
```
r2(config)# interface Ethernet0/1
r2(config-if)#  ip ospf network non-broadcast
r2(config-if)#  ip ospf priority 0
r2(config)# router ospf 1
r2(config-router)#  neighbor 10.0.0.1
```
(same on R3 with `neighbor 10.0.0.1`)

On the capture: Hellos are now **unicast** to the neighbor addresses, sent every
**30s**, and adjacencies form only where you configured `neighbor`.

```
r1# show ip ospf interface Ethernet0/1    ! "Network Type NON_BROADCAST", timers 30/120
r1# show ip ospf neighbor                  ! R2/R3 Full; R1 is DR (only eligible router)
```

> **Gotcha to burn in:** forget `priority 0` on a spoke that *isn't* meshed to all
> other spokes and it can win the DR election and black-hole the segment — the
> classic NBMA DR problem, and why hub-and-spoke NBMA pins the DR to the hub.

### 7.3 Switch the LAN to point-to-multipoint

p2mp treats the segment as a bundle of point-to-point links: **no DR/BDR**,
automatic (multicast) discovery again, and it advertises each neighbor as a **/32
host route**.

```
r1(config)# router ospf 1
r1(config-router)#  no neighbor 10.0.0.2
r1(config-router)#  no neighbor 10.0.0.3
r1(config)# interface Ethernet0/1
r1(config-if)#  ip ospf network point-to-multipoint
```
(do the same on R2/R3 — remove their `neighbor` statements and `priority 0`, set
`point-to-multipoint`; timers become 30/120 everywhere)

```
r1# show ip ospf neighbor          ! all three Full, NO DR/BDR (state FULL/  -)
r1# show ip route ospf             ! /32 host routes to 10.0.0.2 and 10.0.0.3 appear
```

The **/32 host routes** are the p2mp signature — they're why p2mp "just works"
over a partial-mesh NBMA cloud without any DR headache. On the capture the Hellos
are multicast (224.0.0.5) again but with DR/BDR fields **0.0.0.0**.

### 7.4 Restore broadcast

```
r1/r2/r3(config-if)# ip ospf network broadcast     ! or: no ip ospf network
```
Remove any leftover `neighbor` and `priority 0` lines, `clear ip ospf process`,
and confirm DR/BDR election is back (Lab 2).

---

# Lab 8 — NSSA & totally-NSSA (the Type-7 LSA)

**Goal:** an NSSA is a stub area that must still **import its own externals**.
Make **Area 2** an NSSA, turn **R5 into an ASBR** inside it, and watch the
**Type-7 NSSA-External LSA** appear in Area 2 and get **translated to a Type-5**
by ABR R2 as it crosses into the backbone. This completes the area-type ladder
from Lab 6: *normal → stub → totally-stubby → **NSSA** → totally-NSSA*.

> 📖 **Deep dive:** [Area types: stub, totally-stubby, NSSA](CONCEPTS.md#area-types-stub-totally-stubby-nssa)
> · [External routes: E1 vs E2, Type-4, forwarding address](CONCEPTS.md#external-routes-e1-vs-e2-type-4-forwarding-address)
> · [The LSA types](CONCEPTS.md#the-lsa-types)

### 8.1 Give R5 an external to inject

```
r5(config)# ip route 192.168.50.0 255.255.255.0 Null0
```

### 8.2 Make Area 2 an NSSA (both R2 and R5)

Every router in the area must agree — the **N-bit in the Hello** must match or the
adjacency drops (exactly like the stub E-bit in Lab 6).

> **📸 CAPTURE — `r2` interface `Ethernet0/2`** (Area 2 side). Filter `ospf` to
> catch the adjacency drop/re-form on the N-bit change, then `ospf.lsa.type==7`.

```
r2(config)# router ospf 1
r2(config-router)#  area 2 nssa
r5(config)# router ospf 1
r5(config-router)#  area 2 nssa
r5(config-router)#  redistribute static subnets
```

### 8.3 Watch the Type-7 → Type-5 translation

```
r5# show ip ospf database nssa-external   ! Type-7 for 192.168.50.0, originated by R5 (ASBR)
r2# show ip ospf database nssa-external    ! Type-7 visible inside area 2
r2# show ip ospf database external          ! ABR R2 has TRANSLATED it to a Type-5
r1# show ip route 192.168.50.0             ! backbone sees a normal O E2
r4# show ip route 192.168.50.0             ! R4 (area 1) also sees O E2 (translated)
```

- Inside the NSSA the route is a **Type-7** (N2 by default). ABR **R2 is the NSSA
  translator** (the highest-RID ABR in the area does the job) and re-originates it
  as a **Type-5**, so the rest of the domain sees an ordinary `O E2`.
- On the capture (`ospf.lsa.type==7`) note the **P-bit (propagate)** in the LSA
  options — that's what tells the ABR to translate — and a **non-zero forwarding
  address** (unlike a Type-5, which usually carries 0.0.0.0).

### 8.4 What R5 does and doesn't see

```
r5# show ip ospf database external   ! NONE -- Type-5s are blocked in an NSSA (stub behaviour)
r5# show ip route ospf                ! O IA + its own N routes, but NO O E2 from R3's externals
```
R5 is shielded from R3's Type-5 externals (stub-like) yet can still originate its
own — the whole point of NSSA.

### 8.5 Totally-NSSA (block Type-3 too) + the default

```
r2(config-router)# area 2 nssa no-summary     ! ABR only
```
R5 loses inter-area summaries and gets a single **`O*N2 0.0.0.0/0`** default from
R2 — the NSSA analogue of totally-stubby.

> **Two NSSA gotchas:** the injected default is **N2**, not IA; and (Cisco) an NSSA
> ABR does **not** originate a default automatically unless it's `no-summary` **or**
> you add `area 2 nssa default-information-originate`. Plain `area 2 nssa` gives
> the spoke *no* default — a frequent "NSSA can't reach the internet" ticket.

### 8.6 Restore
```
r2(config-router)# no area 2 nssa no-summary
r2(config-router)# no area 2 nssa
r5(config-router)# no area 2 nssa
r5(config-router)# no redistribute static subnets
```

---

# Lab 9 — Virtual links & transit areas

**Goal:** OSPF requires every area to touch the backbone and Area 0 to be
contiguous. A **virtual link** repairs a violation by tunnelling a backbone
adjacency across a **transit area**. You'll deliberately strand an area behind R4
and heal it.

> 📖 **Deep dive:** [Virtual links and transit areas](CONCEPTS.md#virtual-links-and-transit-areas)
> · [Areas: why OSPF partitions the topology](CONCEPTS.md#areas-why-ospf-partitions-the-topology)

### 9.1 Break the rule: an area behind R4 with no backbone link

R4's Lo1 172.16.40.0/24 is in Area 1. Move it into a **new Area 3**, making R4 an
ABR between Area 1 and Area 3 — but Area 3 reaches Area 0 **only through Area 1**:

```
r4(config)# router ospf 1
r4(config-router)#  no network 172.16.40.0 0.0.0.255 area 1
r4(config-router)#  network 172.16.40.0 0.0.0.255 area 3
```

### 9.2 Observe the breakage

```
r4# show ip ospf                ! "It is an area border router"
r1# show ip route 172.16.40.0    ! MISSING from the backbone
r5# show ip route 172.16.40.0    ! unreachable from other areas
```
An ABR must have an Area 0 adjacency to inject its other areas' prefixes into the
backbone. R4's only "uplink" is Area 1, so Area 3's 172.16.40.0/24 is **stranded**.

### 9.3 Build the virtual link across transit Area 1

A virtual link runs between two ABRs (R1 and R4) **through** a non-backbone
**transit area** (Area 1). Configure both ends, each referencing the **remote
router's RID**:

> **📸 CAPTURE — `r1` interface `Ethernet0/2`** (the Area 1 transit link). Filter
> `ospf`. Virtual-link OSPF packets are **unicast** between the endpoints (not the
> usual 224.0.0.5) — you'll see them addressed R1 ↔ R4.

```
r1(config)# router ospf 1
r1(config-router)#  area 1 virtual-link 4.4.4.4
r4(config)# router ospf 1
r4(config-router)#  area 1 virtual-link 1.1.1.1
```

> Area 1 is now a **transit area**. It therefore **cannot** be a stub/NSSA — a
> virtual link can't traverse a stub area. (If you'd left Area 1 as a stub from
> Lab 6, the virtual link would refuse to come up — a classic exam trap.)

### 9.4 Verify the heal

```
r1# show ip ospf virtual-links     ! state UP, "transit area 1", cost, timers
r4# show ip ospf virtual-links
r1# show ip ospf neighbor           ! R4 now also appears as a neighbor on interface OSPF_VL0
r5# show ip route 172.16.40.0       ! now reachable as O IA -- backbone logically extended to R4
```
The virtual link gives R4 a logical Area 0 adjacency, so it acts as a true ABR and
Area 3's prefix reaches the whole domain.

### 9.5 Restore
```
r1(config-router)# no area 1 virtual-link 4.4.4.4
r4(config-router)# no area 1 virtual-link 1.1.1.1
r4(config-router)# no network 172.16.40.0 0.0.0.255 area 3
r4(config-router)# network 172.16.40.0 0.0.0.255 area 1
```

---

# Lab 10 — Path preference (intra > inter > E1 > E2)

**Goal:** when a prefix is learned multiple ways inside one OSPF process, OSPF
picks by a strict **route-type order — *before* metric**. This is ENARSI 1.10.d
and a favourite "why is traffic taking the long way?" troubleshooting scenario.

> 📖 **Deep dive:** [Path preference and route selection](CONCEPTS.md#path-preference-and-route-selection)
> · [External routes: E1 vs E2](CONCEPTS.md#external-routes-e1-vs-e2-type-4-forwarding-address)

### 10.1 The order OSPF uses

Within a single OSPF process, best-path selection is:

1. **Intra-area (`O`)** — preferred regardless of cost
2. **Inter-area (`O IA`)**
3. **External Type-1 (`O E1`)** — external metric **+** internal cost to the ASBR
4. **External Type-2 (`O E2`)** — external metric only (default)

Only *within the same type* does the lowest **cost** decide. (In an NSSA, `N1`/`N2`
rank alongside `E1`/`E2`.) An intra-area route beats an inter-area route to the
same prefix **even when the IA path is objectively shorter** — the single most
common OSPF path-preference ticket.

### 10.2 See type-beats-metric on the externals

R3's `172.16.30.0/24` is a redistributed external (Lab 5). Confirm the E1/E2
behaviour that sits at the bottom of the preference order:

```
r5# show ip route 172.16.30.0     ! O E2 -- metric is the SAME everywhere (external cost only)
r3(config-router)# redistribute connected subnets metric-type 1
r5# show ip route 172.16.30.0     ! now O E1 -- metric = external + internal cost to ASBR R3
r4# show ip route 172.16.30.0     ! E1 metric DIFFERS by distance; E2 did not
```
Because E1 folds in the internal cost, a router near the ASBR prefers a *closer*
ASBR — which is exactly why E1 matters when a prefix is redistributed at **two**
ASBRs. E2 (default) ignores internal cost, so every router sees the same metric
and can pick the wrong exit.

### 10.3 The tie-breaker below path preference: administrative distance

Route **type** decides *within* OSPF. When **two protocols** offer the same prefix,
**administrative distance** decides first (OSPF 110, EIGRP internal 90, etc.) —
that cross-protocol contest is the subject of the redistribution lab. Preview:

```
r4# show ip route 172.16.30.0     ! the "[110/..]" -- 110 is OSPF's AD, then the metric
r4# show ip ospf rib               ! OSPF's own view before it hands the winner to the RIB
```

> **Reset:** put R3 back to the default `metric-type 2` when done
> (`redistribute connected subnets` / `redistribute static subnets` without
> `metric-type 1`).

---

# Lab 11 — OSPFv3: OSPF for IPv6 with address families

The interfaces are already **dual-stacked** (see the addressing table). Now run
OSPF for IPv6. Modern IOS uses the **OSPFv3 address-family model** (`router
ospfv3`), which carries **both** the IPv6 and IPv4 address families in one process
— exactly the "address families" ENARSI 1.10.a calls out. (The older `ipv6 router
ospf` syntax still works; see 11.4.)

> 📖 **Deep dive:** [OSPFv3: what actually changed](CONCEPTS.md#ospfv3-what-actually-changed)
> · [The LSA types](CONCEPTS.md#the-lsa-types) (Type-8/9 are v3-only)

### 11.0 Enable IPv6 routing first (the #1 gotcha)

```
r1(config)# ipv6 unicast-routing
```
Do this on **every** router. Without it, the interfaces keep their IPv6 addresses
but the router won't form OSPFv3 adjacencies or install IPv6 routes — and the
failure is **silent**. (Repeat on r2–r5.)

### 11.1 Bring up OSPFv3 on the Area 0 LAN

> **📸 CAPTURE — `r1` interface `Ethernet0/1`.** Filter `ospf`. OSPFv3 is **OSPF
> version 3**: IP protocol 89, multicast **FF02::5 / FF02::6**, and every packet is
> sourced from the interface **link-local** address (`fe80::`), never the global
> IPv6 address. Confirm that in the capture.

```
r1(config)# router ospfv3 1
r1(config-router)#  router-id 1.1.1.1
r1(config-router)#  address-family ipv6 unicast
r1(config-router-af)#  exit-address-family
r1(config)# interface Ethernet0/1
r1(config-if)#  ospfv3 1 ipv6 area 0
r1(config)# interface Loopback0
r1(config-if)#  ospfv3 1 ipv6 area 0
```

OSPFv3 still needs a **32-bit router-id**; with IPv4 present it can auto-derive,
but set it explicitly. Activation is **always per-interface** (`ospfv3 1 ipv6 area
N`) — there are **no `network` statements** in OSPFv3.

Repeat on R2/R3 (e0/1 + Lo0 → area 0), the p2p links into R4 (area 1) and R5
(area 2), and the edge loopbacks (Lo1) in their areas — mirroring the IPv4 area
design exactly.

### 11.2 Verify and read the new v3-only LSAs

```
r1# show ospfv3 neighbor
r1# show ospfv3 interface brief
r4# show ipv6 route ospf           ! OI (inter-area), O (intra), OE2 (external) IPv6 prefixes
r1# show ospfv3 ipv6 database
```

OSPFv3 architecture differences to note in the LSDB / on the wire:

- **Link-LSA (Type 8)** — link-local scope; carries the router's link-local address
  and the IPv6 prefixes on that link. **New in v3.**
- **Intra-Area-Prefix-LSA (Type 9)** — carries the IPv6 prefixes. In v3 the
  **Router (1) and Network (2) LSAs are topology-only — they no longer carry
  addresses**; prefixes moved into Type 9. This **decoupling of topology from
  addressing** is *the* architectural change in OSPFv3.
- Inter-Area-Prefix (3, ≈ v2 Type-3), Inter-Area-Router (4, ≈ v2 Type-4) and
  AS-External (5) all have direct v3 analogues.
- The adjacency FSM, DR/BDR election, area types (stub/NSSA), and path preference
  behave **identically** to v2 — you already know them.

### 11.3 Everything from v2 works in v3

In the IPv6 AF, try what you built earlier: DR/BDR on the LAN (v3 uses RID and
`ospfv3 1 ipv6 priority` identically), `area 2 stub`, and summarization —
```
r1(config-router)# address-family ipv6 unicast
r1(config-router-af)#  area 1 range 2001:DB8:40::/64
```
Authentication differs: OSPFv3 uses **IPsec** (`ospfv3 1 ipv6 authentication ipsec
spi ...`) or its own auth trailer — **not** the v2 `ip ospf message-digest-key`.

### 11.4 (Reference) Traditional OSPFv3 syntax

```
r1(config)# ipv6 router ospf 1
r1(config-rtr)#  router-id 1.1.1.1
r1(config)# interface Ethernet0/1
r1(config-if)#  ipv6 ospf 1 area 0
```
Functionally equivalent for IPv6-only. The `router ospfv3` address-family form is
preferred because a single process serves both IPv4 and IPv6.

---

# Quick extras (short, high-value)

> 📖 **Deep dive:** [Cost and the reference-bandwidth trap](CONCEPTS.md#cost-and-the-reference-bandwidth-trap)
> · [Authentication](CONCEPTS.md#authentication)

### Cost / path selection
```
r1# show ip ospf interface Ethernet0/1     ! see Cost (auto = ref-bw / bw)
r1(config-if)# ip ospf cost 50              ! override; watch routes shift
r1(config-router)# auto-cost reference-bandwidth 100000   ! fix the 1990s 100Mb default (do on ALL routers)
```

### Hello / Dead timers (must match to peer)
> **📸 CAPTURE — `r1` e0/1**, filter `ospf.msg==1`. Change `ip ospf hello-interval 5`
> on R1 only and watch the adjacency to R2/R3 **drop** (Dead interval mismatch),
> then fix it. Timers are carried in and verified from the Hello.

### Authentication
> **📸 CAPTURE — `r2` e0/2**, filter `ospf`. Enable on R2 only first and watch
> adjacency drop, then match on R5.
```
r2(config-if)# ip ospf authentication message-digest
r2(config-if)# ip ospf message-digest-key 1 md5 CISCO
```
Inspect a Hello/DBD: the **Auth Type = 2 (Cryptographic)** and an MD5 digest is
appended. With `authentication` (type 1) the password is **cleartext** in the
packet — a nice security lesson to see on the wire.

### Route summarization at an ABR
```
r1(config-router)# area 1 range 172.16.40.0 255.255.255.0     ! summarize Area 1 into the backbone
```
Capture Type-3 LSAs (`ospf.lsa.type==3`) before/after to see many specifics
collapse into one summary.

---

## Suggested end-state verification

```
r4# show ip route ospf         ! O (intra), O IA (inter-area), O E1/E2 (external), O*IA default if stub
r1# show ip ospf database       ! ABR: Router, Network, Summary, ASBR-summary, External LSAs
r3# show ip ospf                ! "It is an autonomous system boundary router"
any# show ip ospf neighbor      ! all adjacencies FULL (or 2WAY between DROthers on the LAN)
any# show ospfv3 neighbor       ! (Lab 11) OSPFv3/IPv6 adjacencies
r4# show ipv6 route ospf        ! (Lab 11) O / OI / OE2 IPv6 prefixes
```

## LSA type cheat-sheet (what you captured)

| Type | Name          | Originated by | Scope                    | Seen in this lab      |
|------|---------------|---------------|--------------------------|-----------------------|
| 1    | Router        | every router  | within its area          | Lab 1 (adjacency)     |
| 2    | Network       | **DR** only   | within its area          | Lab 2 (DR's job)      |
| 3    | Summary       | **ABR**       | into adjacent areas      | Lab 4 (inter-area)    |
| 4    | ASBR-Summary  | **ABR**       | how to reach the ASBR    | Lab 5.4               |
| 5    | AS-External   | **ASBR**      | whole OSPF domain        | Lab 5.3               |
| 7    | NSSA-External | ASBR in NSSA  | within the NSSA (→ T5)   | Lab 8                 |
| 8    | Link (v3)     | every router  | **link-local** (v3 only) | Lab 11                |
| 9    | Intra-Prefix (v3)| router/DR  | within its area (v3 only)| Lab 11                |

**Router types** (ENARSI 1.10.c-iii): *internal* (all interfaces in one area — R4,
R5), *backbone* (≥1 interface in Area 0 — R1/R2/R3), *ABR* (interfaces in ≥2 areas,
one being Area 0 — R1/R2), *ASBR* (redistributes externals — R3, and R5 in Lab 8).
A router can wear several hats at once.

---

## Reset the whole lab

```bash
sudo containerlab destroy -t ospf-lab.clab.yml --cleanup
sudo containerlab deploy  -t ospf-lab.clab.yml
```

Because the startup configs contain **no OSPF**, every redeploy gives you a clean
slate to practice against.
