# CCNP EIGRP Deep-Dive Lab (Cisco IOL + Edgeshark)

A 5-router + 1-switch `cisco_iol` containerlab topology for learning **EIGRP**
and the **DUAL** algorithm from the packet up. The routers boot with **only
interfaces and IP addresses** — **no EIGRP, no routing config at all**. You build
every concept by hand and use **Edgeshark packet captures** at each step to *see*
what EIGRP is doing on the wire.

This is the companion lab to [`../ospf`](../ospf). Same node count, same
addressing style — but the wiring is deliberately different: EIGRP's headline
lessons (**feasible successors** and **query scoping**) need **redundant paths**,
so **R4 is dual-homed** and **R5 is a single-homed stub**. Where it's useful,
this guide points out the direct EIGRP-vs-OSPF contrast.

By the end you will have configured and observed:

- EIGRP neighbor discovery and the **reliable transport** (Hello / Update /
  Query / Reply / Ack, RTP, sequence + ACK numbers)
- The **topology table** and **DUAL**: successor, **feasible successor**, the
  **feasibility condition** (RD < FD)
- **Convergence two ways** — instant failover to a feasible successor (route
  never leaves *Passive*) vs. **going Active** and flooding **Queries** when no
  feasible successor exists (and how **Stuck-in-Active** happens)
- **Bounding the query domain** — **summarization** and **EIGRP stub** routers
- **Metric composition** (K-values, bandwidth/delay), **wide metrics**, and
  **unequal-cost load balancing** with `variance`
- **Named-mode** configuration, **authentication**, `no auto-summary`, and
  passive interfaces
- **EIGRP for IPv6** in both **classic** (`ipv6 router eigrp`) and **named-mode
  address-family** form — same DUAL, new address family

> 📖 **This lab has a companion:** [`CONCEPTS.md`](CONCEPTS.md) is the in-depth
> reference behind every step (RTP, the three tables, DUAL & the feasibility
> condition, query scoping, the composite metric, and more). Each lab opens with a
> **Deep dive** callout — follow it for the *why*, skip it when you just want to
> type. It also reads well end-to-end as an EIGRP primer.

---

## Topology

```
                          R4 (4.4.4.4)   Lo1 172.16.40.0/24
                       DUAL playground (dual-homed edge)
                        /                       \
        10.0.14.0/30   /  e0/1            e0/2   \  10.0.24.0/30
         (p2p, PRIMARY) \                        / (p2p, BACKUP = FS)
                         \                      /
                   ┌── R1 (1.1.1.1) ──── R2 (2.2.2.2) ──┐
                   │        \             /             │
     CORE LAN      │         \           /              │   Area-less:
     10.0.0.0/24   │        ┌──┴───────┴──┐             │   ONE EIGRP AS,
     via SW1 (L2)  │        │   SW1 (L2)  │             │   no DR/BDR,
     R1 .1 R2 .2   │        │ flat switch │             │   every router
     R3 .3         │        └──────┬──────┘             │   peers directly
                   └──────────── R3 (3.3.3.3) ──────────┘
                                  │  Lo1 172.16.30.0/24 (external)
                                  │  10.0.35.0/30  (p2p)
                                  │
                              R5 (5.5.5.5)   STUB / edge
                              Lo1 172.16.50.0/24
```

**Why this shape?** Three things EIGRP does that OSPF does not are all baked in:

1. **Feasible successors** need a *loop-free backup path*. **R4 is dual-homed**
   to R1 and R2, so to reach anything in the core it has a **successor** (primary)
   and — once you tune the metrics — a **feasible successor** (backup). This is
   where you watch DUAL fail over *without* ever going Active.
2. **Queries** are what EIGRP floods when a route goes Active with no backup.
   **R5 hangs single-homed off R3**, so killing the R3–R5 link gives R3 a
   destination with **no feasible successor** — it *must* go Active and Query.
3. **No DR/BDR.** R1, R2, R3 share one Ethernet segment (SW1), exactly like the
   OSPF lab — but EIGRP elects nothing. Every router forms a full peering with
   every other router on the segment. That contrast is the point of keeping SW1.

### Roles

| Router | EIGRP RID | Role                                                        |
|--------|-----------|-------------------------------------------------------------|
| R1     | 1.1.1.1   | Core; **R4's primary (successor) path**                     |
| R2     | 2.2.2.2   | Core; **R4's backup (feasible successor) path**             |
| R3     | 3.3.3.3   | Core; gateway to the stub R5; external Lo1 (redistribution) |
| R4     | 4.4.4.4   | **Dual-homed edge** — the DUAL / feasible-successor lab     |
| R5     | 5.5.5.5   | **Single-homed edge** — EIGRP **stub** / query boundary     |
| SW1    | —         | L2 switch (shared core segment; proves EIGRP has no DR)      |

### Addressing

| Link / Interface        | IPv4 Network     | IPv6 Network      | Type            |
|-------------------------|------------------|-------------------|-----------------|
| R1/R2/R3 ↔ SW1 (e0/1)   | 10.0.0.0/24      | 2001:db8:0::/64   | broadcast (LAN) |
| R1 e0/2 ↔ R4 e0/1       | 10.0.14.0/30     | 2001:db8:14::/64  | point-to-point  |
| R2 e0/2 ↔ R4 e0/2       | 10.0.24.0/30     | 2001:db8:24::/64  | point-to-point  |
| R3 e0/2 ↔ R5 e0/1       | 10.0.35.0/30     | 2001:db8:35::/64  | point-to-point  |
| R1 Lo0                  | 1.1.1.1/32       | —                 | loopback / RID  |
| R2 Lo0                  | 2.2.2.2/32       | —                 | loopback / RID  |
| R3 Lo0 / Lo1            | 3.3.3.3/32 / 172.16.30.0/24 | 2001:db8:30::/64 | RID / external |
| R4 Lo0 / Lo1            | 4.4.4.4/32 / 172.16.40.0/24 | 2001:db8:40::/64 | RID / edge net |
| R5 Lo0 / Lo1            | 5.5.5.5/32 / 172.16.50.0/24 | 2001:db8:50::/64 | RID / edge net |

The `e0/X` interfaces are **dual-stacked** — IPv4 for classic EIGRP (Labs 1–6) and
IPv6 for EIGRP-for-IPv6 (Lab 7). Host bits match the router number (R4 = `...::4`).
Lo0 stays IPv4-only (the manual 32-bit router-id source for both families).

Interface mapping: containerlab `ethX` → IOL `Ethernet0/X`. `E0/0` is management.
So `e0/1` = `Ethernet0/1`, `e0/2` = `Ethernet0/2`.

**We use AS 100 throughout.** All routers must agree on the AS number *and* the
K-values or they will not become neighbors.

---

## Deploy

```bash
cd /home/lab/clabs/ccnp/eigrp
sudo containerlab deploy -t eigrp-lab.clab.yml
```

Log in to any node with `admin` / `admin`:

```bash
ssh admin@clab-eigrp-lab-r1        # or: docker exec -it clab-eigrp-lab-r1 telnet localhost
```

Give the nodes ~60–90s after deploy before the CLI is fully ready. Destroy with:

```bash
sudo containerlab destroy -t eigrp-lab.clab.yml --cleanup
```

### Sanity check before you start (no EIGRP yet)

```
r1# show ip interface brief          ! e0/1, e0/2, Lo0 up/up
r1# show ip route                    ! only connected + local routes
r1# show ip eigrp neighbors          ! "EIGRP-IPv4 ...": nothing -- correct, blank slate
r1# show ip protocols                ! no eigrp -- blank slate
```

---

## Using Edgeshark for captures

Edgeshark (Ghostwire + Packetflix) lets you capture on any container interface
straight from the browser:

1. Open the **Edgeshark / Ghostwire** web UI in your browser.
2. Find the container (e.g. `clab-eigrp-lab-r1`) and expand its interfaces.
3. Click the **capture** (shark-fin) icon next to the interface you want
   (e.g. `Ethernet0/1`). A browser-based **Wireshark** session opens and streams
   live packets.

Throughout this guide, each **📸 CAPTURE** block tells you **which node +
interface** to capture on, **when** to start it, and **what to look for** (with a
Wireshark display filter).

EIGRP rides directly on **IP protocol 88** and multicasts to **224.0.0.10**
(*AllEIGRPRouters*, MAC `01:00:5e:00:00:0a`). Reliable packets (Update/Query/
Reply) are unicast-acknowledged. Handy display filters:

| Filter                       | Shows                                            |
|------------------------------|--------------------------------------------------|
| `eigrp`                      | all EIGRP packets                                |
| `eigrp.opcode == 5`          | **Hello** (and **ACK** — a Hello with an Ack num)|
| `eigrp.opcode == 1`          | **Update** (carries routes / TLVs)               |
| `eigrp.opcode == 3`          | **Query** (going Active — "does anyone have…?")  |
| `eigrp.opcode == 4`          | **Reply** (answer to a Query)                    |
| `eigrp.opcode == 10`         | **SIA-Query** (Stuck-in-Active liveness probe)   |
| `eigrp.opcode == 11`         | **SIA-Reply**                                    |
| `eigrp.flags.init`           | the **INIT** flag (start of a full table sync)   |
| `eigrp.tlv.type == 0x0102`   | **Internal** route TLV                           |
| `eigrp.tlv.type == 0x0103`   | **External** route TLV (redistributed)           |

> Tip: Hellos every 5s can bury the interesting packets. Start the capture,
> *then* apply the config that triggers the exchange, and use the opcode filters
> above to isolate Updates/Queries/Replies.

**EIGRP vs OSPF on the wire, at a glance:** EIGRP has **no DBD/LSR/LSU/LSAck**
handshake and **no LSAs**. Neighbors sync by exchanging their whole topology once
(Updates with the INIT flag), then send only **incremental** Updates on change.
There is no periodic reflooding and no DR.

> 📖 **Deep dive:** [Neighbors and reliable transport (RTP)](CONCEPTS.md#neighbors-and-reliable-transport-rtp)
> · [The three tables](CONCEPTS.md#the-three-tables)
> · [What EIGRP is and how it differs from OSPF](CONCEPTS.md#what-eigrp-is-and-how-it-differs-from-ospf)

---

# Lab 1 — Bring up EIGRP neighbors (the reliable transport)

**Goal:** enable EIGRP on the three core routers (the shared LAN) and watch
neighbor discovery, the K-value check, and the initial reliable table sync
(Update+INIT → ACK) on the wire.

> 📖 **Deep dive:** [Neighbors and reliable transport (RTP)](CONCEPTS.md#neighbors-and-reliable-transport-rtp)
> · [What EIGRP is and how it differs from OSPF](CONCEPTS.md#what-eigrp-is-and-how-it-differs-from-ospf)

### 1.1 Start the capture FIRST

> **📸 CAPTURE — `r1` interface `Ethernet0/1`** (the core LAN). Start this
> *before* you configure EIGRP so you catch the very first Hello and the entire
> neighbor bring-up. Filter: `eigrp`.

### 1.2 Enable EIGRP on R1 only

```
r1# conf t
r1(config)# router eigrp 100
r1(config-router)#  eigrp router-id 1.1.1.1
r1(config-router)#  network 10.0.0.0 0.0.0.255       ! core LAN
r1(config-router)#  network 10.0.14.0 0.0.0.3        ! p2p to R4
r1(config-router)#  network 1.1.1.1 0.0.0.0          ! Lo0
r1(config-router)#  no auto-summary
r1(config-router)# end
```

> `network` in EIGRP is **not** "advertise this network" — it's "**activate EIGRP
> on any interface whose IP falls in this wildcard range**." The interface's
> connected subnet is what actually gets advertised. `1.1.1.1 0.0.0.0` matches
> exactly the loopback.

**On the capture now:** with only R1 running you'll see R1 sending **Hellos**
(`eigrp.opcode==5`) to **224.0.0.10** every 5s. Open one Hello and note:
- the **Autonomous System = 100** in the EIGRP header,
- the **Parameters TLV** carrying **K1..K5** (default `K1=K3=1`, others `0`) and
  the **Hold Time** (15s). Both the AS and the K-values must match a neighbor or
  no adjacency forms.
- No routes are attached — Hellos only discover neighbors.

### 1.3 Enable EIGRP on R2, then R3

```
r2# conf t
r2(config)# router eigrp 100
r2(config-router)#  eigrp router-id 2.2.2.2
r2(config-router)#  network 10.0.0.0 0.0.0.255
r2(config-router)#  network 10.0.24.0 0.0.0.3
r2(config-router)#  network 2.2.2.2 0.0.0.0
r2(config-router)#  no auto-summary
r2(config-router)# end
```

```
r3# conf t
r3(config)# router eigrp 100
r3(config-router)#  eigrp router-id 3.3.3.3
r3(config-router)#  network 10.0.0.0 0.0.0.255
r3(config-router)#  network 10.0.35.0 0.0.0.3
r3(config-router)#  network 3.3.3.3 0.0.0.0
r3(config-router)#  no auto-summary
r3(config-router)# end
```

Watch the adjacencies form (you'll get console logs `%DUAL-5-NBRCHANGE ... is up:
new adjacency`):

```
r1# show ip eigrp neighbors
```

You want R2 and R3 in the neighbor table with a valid **Hold** time and an
**SRTT/RTO**. On a shared LAN **all three peer with each other** — no DR, no
"2-WAY" holding pattern. Contrast this with OSPF, where DROthers stay 2-WAY.

### 1.4 Read the neighbor bring-up off the capture (the payoff)

Filter the capture to the moment R2 came up. In order you should see:

1. **Hello (`opcode==5`)** — discovery. Once R1 and R2 have exchanged Hellos with
   matching AS + K-values, they become neighbors.
2. **Update with the INIT flag (`eigrp.flags.init`)** — each side sends a full
   copy of its topology table. This is the initial database synchronization.
   *(In OSPF this was the DBD/LSR/LSU dance; EIGRP just ships the tables.)*
3. **ACK (`opcode==5` with a non-zero Acknowledge number)** — every reliable
   packet (Update/Query/Reply) is explicitly acknowledged. This is **RTP**, EIGRP's
   Reliable Transport Protocol: note the **Sequence** and **Acknowledge** numbers
   in the header and how an ACK's Ack number matches the Update's Seq number.
4. Follow-up **Updates** (no INIT) carry the actual route TLVs
   (`eigrp.tlv.type==0x0102`, internal routes) — expand one and find the
   **Delay**, **Bandwidth**, **Hop count**, and **Prefix** fields. These metric
   components are what DUAL feeds on.

> **Concept:** EIGRP neighbors *discover* with unreliable multicast Hellos, then
> *synchronize* with reliable, acknowledged Updates. After the initial sync, only
> **changes** are sent (incremental updates) — there is no periodic reflood.

### 1.5 Verify full reachability

```
r1# show ip eigrp neighbors           ! R2 & R3 present
r1# show ip route eigrp               ! D routes: 2.2.2.2/32, 3.3.3.3/32, 10.0.24.0/30, 10.0.35.0/30 ...
r1# show ip eigrp topology            ! every prefix, each with a successor (Passive/"P")
r1# show ip protocols                 ! AS 100, "Automatic Summarization: disabled", RID
```

Everything in `show ip eigrp topology` should be **P** (Passive = stable). We'll
make things go **A** (Active) on purpose in Lab 3.

---

# Lab 2 — The topology table & DUAL: successor + feasible successor

**Goal:** understand the numbers DUAL uses — **FD** (feasible distance), **RD/AD**
(reported/advertised distance) — and create a clean **successor + feasible
successor** pair on R4 by tuning delay.

> 📖 **Deep dive:** [DUAL and loop-free path selection](CONCEPTS.md#dual-and-loop-free-path-selection)
> · [The three tables](CONCEPTS.md#the-three-tables)
> · [The composite metric](CONCEPTS.md#the-composite-metric)

### 2.1 Bring R4 up (dual-homed)

```
r4# conf t
r4(config)# router eigrp 100
r4(config-router)#  eigrp router-id 4.4.4.4
r4(config-router)#  network 10.0.14.0 0.0.0.3        ! uplink to R1
r4(config-router)#  network 10.0.24.0 0.0.0.3        ! uplink to R2
r4(config-router)#  network 4.4.4.4 0.0.0.0          ! Lo0
r4(config-router)#  network 172.16.40.0 0.0.0.255    ! edge net
r4(config-router)#  no auto-summary
r4(config-router)# end
```

R4 now has **two** neighbors (R1 and R2) and therefore two ways to reach every
core prefix. By default both links are identical Ethernet, so the two paths tie:

```
r4# show ip route eigrp               ! note EQUAL-COST paths installed (two next-hops)
r4# show ip eigrp topology
```

With equal metrics EIGRP installs **both** (equal-cost load balancing, up to 4
paths by default). Pick a specific destination to study — say R3's loopback
`3.3.3.3/32`:

```
r4# show ip eigrp topology 3.3.3.3/32
```

Read the entry: **FD** is R4's best total metric; each successor line shows
`(FD/RD)` — the total metric via that neighbor and the neighbor's **Reported
Distance** (its own metric to the destination).

### 2.2 Make R1 the successor and R2 the feasible successor

Raise the **delay** on R4's link to R2 so the path via R1 wins. Delay is in tens
of microseconds; this only changes R4's *own* metric, not R2's reported distance:

> **📸 CAPTURE — `r4` interface `Ethernet0/2`** (the R2 uplink). Filter `eigrp`.
> You'll see R4 send updated Updates as its metrics change.

```
r4(config)# interface Ethernet0/2
r4(config-if)#  delay 2000
r4(config-if)# end
```

Now re-read the destination:

```
r4# show ip eigrp topology 3.3.3.3/32
```

You should see:
- **one successor** (via **R1 / 10.0.14.1**) with the lower FD,
- **R2 (10.0.24.1) listed as a feasible successor** — its `RD` is **less than**
  R4's `FD`. That inequality, **RD < FD**, is the **feasibility condition**, and
  it's what guarantees R2's path is loop-free (R2 is closer to the destination
  than R4 currently is, so R2 can't be routing *through* R4).

```
r4# show ip route 3.3.3.3               ! now a SINGLE best path (via R1)
r4# show ip eigrp topology all-links    ! shows ALL neighbors incl. non-feasible ones
```

> **Key distinction:** `show ip eigrp topology` lists only successors + **feasible
> successors** (the loop-free set DUAL will use instantly). `show ip eigrp
> topology all-links` also shows neighbors whose path exists but **fails** the
> feasibility condition — those are *not* usable without going Active.

### 2.3 The three numbers, defined

| Term | Meaning |
|------|---------|
| **FD** — Feasible Distance | *My* lowest metric to the destination (historically lowest). |
| **RD / AD** — Reported / Advertised Distance | A **neighbor's** metric to the destination, as it reported to me. |
| **Successor** | The neighbor providing the current lowest-metric (FD) path. |
| **Feasible Successor (FS)** | A backup neighbor whose **RD < FD** → guaranteed loop-free, installed in the topology table ready to use with **zero reconvergence**. |

Everything in Lab 3 hinges on whether an FS **exists**.

---

# Lab 3 — Convergence two ways: feasible successor vs. going Active

**Goal:** see the two very different things that happen when a successor dies —
**(A)** an FS exists → instant local switchover, route never leaves *Passive*, **no
query**; **(B)** no FS exists → the route goes **Active** and EIGRP floods
**Queries**.

> 📖 **Deep dive:** [Convergence: passive, active, and queries](CONCEPTS.md#convergence-passive-active-and-queries)
> · [Stuck-in-Active (SIA)](CONCEPTS.md#stuck-in-active-sia)

### 3A — Feasible successor = silent, instant failover

R4 currently reaches the core via R1 (successor) with **R2 as a feasible
successor** (Lab 2). Kill the primary and watch DUAL switch locally.

> **📸 CAPTURE — `r4` interface `Ethernet0/2`** (the R2/backup uplink). Filter
> `eigrp`. Keep it running across the failure.

```
r4# debug eigrp fsm                     ! optional: watch DUAL pick the FS in the log
r4# conf t
r4(config)# interface Ethernet0/1        ! the PRIMARY uplink to R1
r4(config-if)#  shutdown
r4(config-if)# end
```

Observe:

```
r4# show ip route 3.3.3.3                ! instantly via R2 now (10.0.24.1)
r4# show ip eigrp topology 3.3.3.3/32    ! stayed PASSIVE the whole time
```

**On the capture:** you will **not** see a Query (`opcode==3`). Because R2 was
already a *feasible* successor, DUAL promotes it locally and the route **never
goes Active** — at most you'll see a routine Update. This is EIGRP's famous
sub-second convergence: the backup was pre-computed and proven loop-free.

Re-enable the link when done:

```
r4(config)# interface Ethernet0/1
r4(config-if)#  no shutdown
```

### 3B — No feasible successor = go Active + flood Queries

Now pick a destination with **only one path** so there is no possible FS: R5's
edge net **172.16.50.0/24**, which hangs single-homed off R3. First bring R5 up:

```
r5# conf t
r5(config)# router eigrp 100
r5(config-router)#  eigrp router-id 5.5.5.5
r5(config-router)#  network 10.0.35.0 0.0.0.3
r5(config-router)#  network 5.5.5.5 0.0.0.0
r5(config-router)#  network 172.16.50.0 0.0.0.255
r5(config-router)#  no auto-summary
r5(config-router)# end
```

Confirm the core learned it, then look at R3 — R3 is the **only** router with a
path to R5, so it has **no feasible successor** for R5's prefix:

```
r1# show ip route 172.16.50.0
r3# show ip eigrp topology 172.16.50.0/24   ! one successor (R5), NO feasible successor
```

> **📸 CAPTURE — `r3` interface `Ethernet0/1`** (R3's LAN side, facing R1/R2).
> Filter `eigrp`. Start it *before* the next command.

Kill R3's only path to R5:

```
r3# conf t
r3(config)# interface Ethernet0/2        ! R3's only link to R5
r3(config-if)#  shutdown
r3(config-if)# end
```

**On the capture**, in order:
1. R3 loses its successor for 172.16.50.0/24 and has **no FS**, so the route goes
   **Active** — R3 multicasts a **Query (`opcode==3`)** onto the LAN: *"does
   anyone have a path to 172.16.50.0/24?"*
2. R1 and R2 receive the Query. They only knew that prefix *through R3*, so they
   have nothing better — they send **Reply (`opcode==4`)** = unreachable. (If they
   had other neighbors, they'd query *those* first — this is how queries
   **propagate**, and why an unbounded query domain is dangerous.)
3. When R3 has collected a Reply from **every** queried neighbor, the route goes
   back **Passive** (now as unreachable). Check:

```
r3# show ip eigrp topology active         ! catch it Active if you're quick
r3# show ip route 172.16.50.0             ! gone once all replies are in
```

> **Stuck-in-Active (SIA):** if a queried neighbor doesn't Reply within the active
> timer (default **3 minutes**), R3 would send an **SIA-Query (`opcode==10`)** to
> check it's alive; still no answer → R3 **resets the neighbor**. SIA is almost
> always caused by an oversized query domain, a slow/flapping link, or a stuck
> neighbor. **Lab 4 is the fix:** shrink the query domain.

Restore R3–R5 before moving on:

```
r3(config)# interface Ethernet0/2
r3(config-if)#  no shutdown
```

---

# Lab 4 — Bounding the query domain: stub + summarization

**Goal:** stop queries from propagating across the whole AS. Two tools: **EIGRP
stub** routers (never queried) and **summarization** (queries for a component are
answered at the summary boundary). Both are core ENARSI troubleshooting knobs.

> 📖 **Deep dive:** [Bounding the query domain: stub and summarization](CONCEPTS.md#bounding-the-query-domain-stub-and-summarization)
> · [Stuck-in-Active (SIA)](CONCEPTS.md#stuck-in-active-sia)

### 4.1 Make R5 an EIGRP stub

R5 is an edge router — nothing should ever route *through* it, so R3 should never
waste a Query on it.

> **📸 CAPTURE — `r3` interface `Ethernet0/2`** (toward R5). Filter `eigrp`.

```
r5(config)# router eigrp 100
r5(config-router)#  eigrp stub connected summary
r5(config-router)# end
```

The R3–R5 adjacency briefly resets and re-forms. **On the capture**, open R5's
**Hello** — the **Parameters/Stub TLV** now advertises R5 as a **stub** (flags for
*connected* + *summary*). Verify from R3:

```
r3# show ip eigrp neighbors detail        ! R5 shown with "Stub Peer ... (connected summary)"
```

> **What changed:** R3 now knows R5 is a stub and will **never send it a Query**.
> A stub also **won't advertise transit routes** (it only originates its own
> connected/summary routes), so it can't become an accidental transit path.
> `eigrp stub` is *the* fix for SIA in hub-and-spoke designs — spokes stop being
> queried. Re-run the Lab 3B failure and confirm R3 no longer queries toward R5.

### 4.2 Summarize toward R4 to cut the query at the boundary

Now bound queries heading the *other* way — into R4. Configure an **interface
summary** on R1 and R2 facing R4 that covers the 172.16.0.0 edge nets:

> **📸 CAPTURE — `r1` interface `Ethernet0/2`** (toward R4). Filter `eigrp`.

```
r1(config)# interface Ethernet0/2
r1(config-if)#  ip summary-address eigrp 100 172.16.0.0 255.255.0.0
r2(config)# interface Ethernet0/2
r2(config-if)#  ip summary-address eigrp 100 172.16.0.0 255.255.0.0
```

Effects to observe:

```
r4# show ip route eigrp                   ! specific 172.16.30/40/50 replaced by ONE 172.16.0.0/16 (D)
r1# show ip route 172.16.0.0              ! a "D ... 172.16.0.0/16 ... Null0" summary appears locally
```

- R4 now sees a **single** `172.16.0.0/16` route instead of the specifics — a
  smaller table.
- The `Null0` discard route on R1 is normal: it backstops the summary so R1 can't
  loop packets for a component it doesn't actually have.
- **The query lesson:** if a component like `172.16.50.0/24` goes Active in the
  core, R1 will **not** propagate the Query to R4 — R1 *is* the summary boundary
  for that block, so it answers on R4's behalf. **Summarization = automatic query
  boundary.** Combined with the R5 stub, the query domain for the edge prefixes
  has collapsed to just the core.

> **Rule of thumb (ENARSI):** every SIA fix is really "make the query domain
> smaller" — **summarize** on distribution/core boundaries and make edge routers
> **stubs**.

---

# Lab 5 — Metric composition, wide metrics, and unequal-cost load balancing

**Goal:** understand what the metric is actually made of, and use `variance` to
load-balance over a feasible successor (something OSPF cannot do without equal
cost).

> 📖 **Deep dive:** [The composite metric](CONCEPTS.md#the-composite-metric)
> · [Load balancing: equal and unequal cost](CONCEPTS.md#load-balancing-equal-and-unequal-cost)
> · [Classic vs named mode](CONCEPTS.md#classic-vs-named-mode)

### 5.1 What the metric is made of

```
r4# show interfaces Ethernet0/1           ! read BW (Kbit) and DLY (usec)
r4# show ip eigrp topology 3.3.3.3/32      ! the composite metric
```

Classic EIGRP metric uses **bandwidth** and **delay** by default (the K-values you
saw in the Hello: `K1=1` weights bandwidth, `K3=1` weights delay; `K2/K4/K5=0`).
Roughly:

```
metric = 256 * ( 10^7 / min_bandwidth_along_path  +  sum_of_delays_along_path )
```

That's why in Lab 2 raising **delay** on R4 e0/2 cleanly demoted that path — you
increased the delay term without touching bandwidth. Prefer tuning **delay** over
**bandwidth** for path selection (bandwidth affects other features like QoS).

> **K-value mismatch demo (optional):** on R4 only, `metric weights 0 1 0 1 0 0`
> (turns on K2). The R4–R1/R4–R2 adjacencies **drop** with a
> `%DUAL-5-NBRCHANGE ... K-value mismatch` log. Capture a Hello and see the
> changed K-values in the Parameters TLV. Set it back with `no metric weights`.

### 5.2 Unequal-cost load balancing with `variance`

R4 has a successor (via R1) and a feasible successor (via R2) with *different*
metrics. `variance N` tells EIGRP to also install any **feasible successor** whose
metric is within **N ×** the successor's FD:

```
r4# show ip eigrp topology 3.3.3.3/32     ! note FD (via R1) and the FS metric (via R2)
r4(config)# router eigrp 100
r4(config-router)#  variance 4
r4(config-router)# end
```

```
r4# show ip route 3.3.3.3                 ! now BOTH R1 and R2 installed, unequally weighted
r4# show ip route 172.16.30.0
```

Traffic is shared **in inverse proportion to metric** across the two paths.

> **Two things to burn in:** (1) `variance` only ever uses **feasible successors**
> — it will *never* install a path that fails the feasibility condition, so it
> **cannot** create a loop. (2) OSPF has no equivalent; it load-balances only over
> *equal*-cost paths.

### 5.3 (Optional) Named mode + wide metrics

Modern EIGRP uses **named configuration** and **64-bit wide metrics** (so it can
represent interfaces faster than ~4 Gbps, which classic 32-bit metrics saturate
at). Same protocol, cleaner config that keeps all address-families/interfaces in
one place:

```
r5(config)# router eigrp CCNP
r5(config-router)#  address-family ipv4 unicast autonomous-system 100
r5(config-router-af)#   network 5.5.5.5 0.0.0.0
r5(config-router-af)#   network 10.0.35.0 0.0.0.3
r5(config-router-af)#   network 172.16.50.0 0.0.0.255
r5(config-router-af)#   eigrp stub connected summary
r5(config-router-af)#   af-interface Ethernet0/1
r5(config-router-af-interface)#     hello-interval 5
r5(config-router-af)#   exit-af-interface
r5(config-router-af)# end
```

Named and classic mode with the **same AS interoperate** as neighbors. Compare
`show ip eigrp topology` metric magnitudes — wide metrics are much larger numbers.

---

# Lab 6 — Authentication, auto-summary gotcha, passive interfaces

> 📖 **Deep dive:** [Authentication](CONCEPTS.md#authentication)
> · [auto-summary and discontiguous networks](CONCEPTS.md#auto-summary-and-discontiguous-networks)
> · [External routes and redistribution](CONCEPTS.md#external-routes-and-redistribution)

### 6.1 MD5 authentication (watch the adjacency drop, then match)

> **📸 CAPTURE — `r3` interface `Ethernet0/2`** (R3↔R5). Filter `eigrp`.

Enable on R3's interface first — the R3–R5 adjacency **drops** (one side
authenticating, the other not), then re-forms once you match R5:

```
r3(config)# key chain EIGRP-KC
r3(config-keychain)#  key 1
r3(config-keychain-key)#   key-string CISCO
r3(config)# interface Ethernet0/2
r3(config-if)#  ip authentication mode eigrp 100 md5
r3(config-if)#  ip authentication key-chain eigrp 100 EIGRP-KC
```

```
r5(config)# key chain EIGRP-KC
r5(config-keychain)#  key 1
r5(config-keychain-key)#   key-string CISCO
r5(config)# interface Ethernet0/1
r5(config-if)#  ip authentication mode eigrp 100 md5
r5(config-if)#  ip authentication key-chain eigrp 100 EIGRP-KC
```

On the capture, open a Hello after both sides match: it now carries an
**Authentication TLV** with the MD5 digest. A mismatched/absent key is exactly why
the neighbor dropped — a great `%DUAL-5-NBRCHANGE ... Auth failure` lesson.

### 6.2 The `auto-summary` gotcha

Our topology crosses a classful boundary: **10.0.0.0/8** and **172.16.0.0/16**.
Turn auto-summary **on** somewhere and watch discontiguous-network breakage:

```
r3(config)# router eigrp 100
r3(config-router)#  auto-summary
```

```
r1# show ip route 172.16.0.0              ! R3 now advertises the classful 172.16.0.0/16 at the boundary
```

With multiple routers owning pieces of 172.16.0.0/16, auto-summary creates
**overlapping /16 advertisements** and black-holes some subnets. This is why
**`no auto-summary` is the modern default** and why you type it by reflex. Turn it
back off:

```
r3(config-router)#  no auto-summary
```

### 6.3 Passive interfaces (advertise the subnet, don't peer)

You want R4's and R5's **edge LANs** in EIGRP but there are no EIGRP speakers out
there — so stop sending Hellos out those interfaces (and stop accepting neighbors)
while still advertising the subnet:

```
r5(config)# router eigrp 100
r5(config-router)#  passive-interface Loopback1
```

> On a real access switchport you'd `passive-interface` the user VLAN SVIs — it
> advertises the subnet but prevents an adjacency (and blocks a spoofed-neighbor
> attack). Confirm with `show ip eigrp interfaces` (passive interfaces are not
> listed as active) while the subnet still appears in neighbors' routes.

### 6.4 (Bridge to the next lab) Redistribution teaser

R3's `Lo1 172.16.30.0/24` was never in a `network` statement. Inject it (and a
static) to make R3 a redistribution point and preview **external** EIGRP routes:

```
r3(config)# ip route 192.168.30.0 255.255.255.0 Null0
r3(config)# router eigrp 100
r3(config-router)#  redistribute connected metric 100000 100 255 1 1500
r3(config-router)#  redistribute static  metric 100000 100 255 1 1500
```

> **📸 CAPTURE — `r3` interface `Ethernet0/1`.** Filter `eigrp.tlv.type==0x0103`.
> Redistributed routes ride in an **External** route TLV (vs. `0x0102` internal) —
> expand one and find the **originating router ID**, **external protocol**, and
> **external metric/tag** fields (all absent from internal routes). In the routing
> table these appear as **D EX** with AD **170** (vs **90** for internal D). The
> **router-id** field is how EIGRP prevents redistribution loops — the deep dive
> is the next lab.

---

# Lab 7 — EIGRP for IPv6 (address families: named mode + classic)

Topic **1.9.a** wants EIGRP in **both** address families. The interfaces are
already **dual-stacked** (see the addressing table); everything you learned about
DUAL, feasible successors, stubs, and query scoping applies to IPv6 **unchanged**
— only the config wrapper and the on-the-wire multicast differ.

> 📖 **Deep dive:** [EIGRP for IPv6](CONCEPTS.md#eigrp-for-ipv6)
> · [Classic vs named mode](CONCEPTS.md#classic-vs-named-mode)

### 7.0 Enable IPv6 routing first (the #1 gotcha)

```
r1(config)# ipv6 unicast-routing
```
On **every** router. Without it EIGRP for IPv6 won't run and IPv6 routes won't
install — silently.

### 7.1 Classic mode: `ipv6 router eigrp` (mind the shutdown!)

Classic EIGRP for IPv6 has **no `network` statements** — you enable it
**per-interface**. And the process starts **administratively shut down** by
default: forgetting `no shutdown` under the process is *the* classic EIGRPv6
gotcha.

> **📸 CAPTURE — `r1` interface `Ethernet0/1`.** Filter `eigrp`. EIGRP for IPv6
> rides IP protocol **88** over IPv6, multicasts to **FF02::A** (not 224.0.0.10),
> and sources packets from the interface **link-local** (`fe80::`) address — same
> opcodes (Hello/Update/Query/Reply/Ack) you already know.

On R1 (repeat the pattern on R2, R3, R4):
```
r1(config)# ipv6 router eigrp 100
r1(config-rtr)#  eigrp router-id 1.1.1.1        ! still a 32-bit RID
r1(config-rtr)#  no shutdown                     ! <-- the gotcha
r1(config)# interface Ethernet0/1
r1(config-if)#  ipv6 eigrp 100
r1(config)# interface Ethernet0/2
r1(config-if)#  ipv6 eigrp 100
```
(R4 enables `ipv6 eigrp 100` on both uplinks + Lo1; R3 on e0/1, e0/2 + Lo1.)

### 7.2 Verify — same DUAL, new address family

```
r1# show ipv6 eigrp neighbors
r4# show ipv6 route eigrp                          ! D (internal) IPv6 prefixes
r4# show ipv6 eigrp topology 2001:DB8:30::/64      ! FD/RD, successor + FS -- identical semantics
r1# show ipv6 protocols                            ! EIGRP-IPv6 AS 100, RID, K-values
```
Because R4 is dual-homed, **repeat Labs 2–3 in IPv6**: raise `delay` on R4 e0/2,
confirm R1 is the successor and R2 the feasible successor for an IPv6 prefix, then
`shutdown` R4 e0/1 and watch it fail over **without going Active**. DUAL is
address-family-agnostic.

### 7.3 Named mode: one process, both address families

Named mode is the modern face and holds IPv4 and IPv6 **in the same process**.
Extend R5 (its IPv4 named-mode config was Lab 5.3) with an IPv6 address family —
note there is **no `network` command** for IPv6; EIGRP includes every
IPv6-enabled interface automatically (scope it with `af-interface ... shutdown`):

```
r5(config)# router eigrp CCNP
r5(config-router)#  address-family ipv6 unicast autonomous-system 100
r5(config-router-af)#   eigrp router-id 5.5.5.5
r5(config-router-af)#   eigrp stub connected summary       ! stub works the same in v6
```

```
r5# show ipv6 eigrp neighbors
r3# show ipv6 eigrp neighbors detail       ! R5 flagged as a Stub Peer in the v6 AF too
```

> **Interop note:** classic `ipv6 router eigrp 100` and named-mode `address-family
> ipv6 ... autonomous-system 100` are the **same protocol instance** — R5 in named
> mode peers happily with R3 in classic mode as long as the AS (100), K-values and
> authentication match.

### 7.4 What's identical vs different (v4 → v6)

| Aspect                      | EIGRP IPv4                | EIGRP for IPv6                    |
|-----------------------------|---------------------------|----------------------------------|
| Transport / multicast       | IP proto 88, 224.0.0.10   | IP proto 88, **FF02::A**         |
| Packet source               | interface IPv4            | interface **link-local** fe80::  |
| Enable on interface         | `network` wildcard        | **per-interface** `ipv6 eigrp`   |
| Process default state       | up                        | classic mode starts **shutdown** |
| Router-ID                   | 32-bit (auto/manual)      | 32-bit (**manual** if no IPv4)   |
| DUAL / FS / stub / variance | ✅                        | ✅ **identical**                 |
| auto-summary                | exists                    | **n/a** (no classful IPv6)       |

---

## Suggested end-state verification

```
r4# show ip route eigrp        ! D (internal, AD 90) and D EX (external, AD 170)
r4# show ip eigrp topology     ! successors + feasible successors, all Passive (P)
r3# show ip eigrp neighbors detail   ! R5 flagged as a Stub Peer
any# show ip protocols         ! AS 100, no auto-summary, K-values, RID, variance
any# show ip eigrp traffic     ! Hello/Update/Query/Reply/Ack counters
any# show ipv6 eigrp neighbors ! (Lab 7) EIGRP-for-IPv6 adjacencies
r4# show ipv6 route eigrp      ! (Lab 7) D IPv6 prefixes -- same DUAL, new AF
```

## EIGRP packet / concept cheat-sheet (what you captured)

| Opcode | Packet | Reliable? | Seen in this lab                        |
|--------|--------|-----------|-----------------------------------------|
| 5      | Hello / ACK  | no    | Lab 1 (discovery, K-values, stub flag)  |
| 1      | Update       | yes   | Lab 1 (INIT sync), Lab 2 (metric change)|
| 3      | Query        | yes   | Lab 3B (going Active)                    |
| 4      | Reply        | yes   | Lab 3B (answering a Query)              |
| 10/11  | SIA-Query/Reply | yes | Lab 3B (Stuck-in-Active discussion)    |

| Term | One-liner |
|------|-----------|
| **FD** | my best metric to the destination |
| **RD/AD** | a neighbor's own metric, as reported to me |
| **Successor** | lowest-metric next hop (installed in RIB) |
| **Feasible Successor** | backup with **RD < FD** → loop-free, instant failover |
| **Active** | no successor *and* no FS → must Query |
| **Stub** | edge router that is never queried and never transits |

---

## Reset the whole lab

```bash
sudo containerlab destroy -t eigrp-lab.clab.yml --cleanup
sudo containerlab deploy  -t eigrp-lab.clab.yml
```

Because the startup configs contain **no EIGRP**, every redeploy gives you a clean
slate to practice against.

---

## How this compares to the OSPF lab

| Concept                | OSPF lab (`../ospf`)              | EIGRP lab (here)                          |
|------------------------|-----------------------------------|-------------------------------------------|
| Shared LAN role        | DR/BDR **election**               | **No DR** — every router peers directly   |
| Adjacency sync         | DBD → LSR → LSU → LSAck            | Update with **INIT** flag, then ACK       |
| Loop-free backup       | full SPF recompute                | **Feasible successor** (pre-computed)     |
| Reconvergence on fail  | run SPF                           | FS = instant; else **go Active + Query**  |
| Scaling / boundaries   | **areas** (Type-3/5 LSAs)         | **stub** routers + **summarization**      |
| Unequal-cost balancing | not supported                    | **`variance`**                            |
| External routes        | Type-5 LSA, E1/E2                  | **D EX** (AD 170), external route TLV      |

Next up in the ENARSI track: **route redistribution & filtering**, which reuses
both this EIGRP domain and the OSPF domain — mutual redistribution, route tags to
break feedback loops, and administrative-distance games.
