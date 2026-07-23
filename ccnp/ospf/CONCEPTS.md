# OSPF — In-Depth Concepts

The companion reference to the [README](README.md) lab guide. The README is the
*hands-on* path (configure this, capture that); this file is the *why* behind each
step. Each 📖 **Deep dive** link in the README jumps to a section here. You can
also read it straight through as an OSPF primer.

**Contents**
- [The link-state model and the LSDB](#the-link-state-model-and-the-lsdb)
- [The neighbor state machine](#the-neighbor-state-machine)
- [Hello packets and what must match](#hello-packets-and-what-must-match)
- [Database synchronization: DBD, LSR, LSU, LSAck](#database-synchronization-dbd-lsr-lsu-lsack)
- [DR and BDR election](#dr-and-bdr-election)
- [Network types in depth](#network-types-in-depth)
- [Areas: why OSPF partitions the topology](#areas-why-ospf-partitions-the-topology)
- [The LSA types](#the-lsa-types)
- [Router types](#router-types)
- [Area types: stub, totally-stubby, NSSA](#area-types-stub-totally-stubby-nssa)
- [External routes: E1 vs E2, Type-4, forwarding address](#external-routes-e1-vs-e2-type-4-forwarding-address)
- [Path preference and route selection](#path-preference-and-route-selection)
- [Cost and the reference-bandwidth trap](#cost-and-the-reference-bandwidth-trap)
- [Virtual links and transit areas](#virtual-links-and-transit-areas)
- [Authentication](#authentication)
- [OSPFv3: what actually changed](#ospfv3-what-actually-changed)

---

## The link-state model and the LSDB

OSPF is a **link-state** protocol, and every behaviour you'll capture flows from
one idea: **every router in an area builds an identical map of that area**, then
each computes its *own* shortest-path tree over that map. Contrast with
distance-vector (EIGRP/RIP), where a router only knows what neighbors *tell* it
about distances and never sees the whole graph.

The map is the **Link-State Database (LSDB)**. It's a collection of **LSAs**
(Link-State Advertisements) — each router describes its own links in a Router-LSA,
the DR describes a multi-access segment in a Network-LSA, and so on (see [the LSA
types](#the-lsa-types)). Routers **flood** LSAs reliably to every neighbor in the
area until all LSDBs are identical (they have *converged*).

Then each router runs **Dijkstra's SPF algorithm** with *itself as the root*,
producing the shortest path to every prefix. Three consequences worth holding onto:

- **All routers in an area must have the same LSDB**, or you get routing loops /
  black holes. This is why mismatches (MTU, area type, auth) *break* things loudly
  rather than silently degrading.
- **SPF is CPU work.** Anything that shrinks the LSDB (areas, stub areas,
  summarization) makes SPF cheaper and convergence faster — that's the entire
  motivation behind areas.
- **Cost is directional and additive.** SPF sums the *outbound* interface [cost](#cost-and-the-reference-bandwidth-trap)
  along a path; the lowest total wins.

[↩ back to README captures](README.md#using-edgeshark-for-captures)

---

## The neighbor state machine

Two OSPF routers don't jump straight to exchanging routes — they walk a **finite
state machine**, and knowing which state a stuck adjacency is in tells you exactly
what's wrong.

| State        | What's happening                                                        | Stuck here usually means…                    |
|--------------|-------------------------------------------------------------------------|----------------------------------------------|
| **Down**     | No Hellos heard yet                                                      | L1/L2 down, wrong subnet, ACL blocking OSPF  |
| **Init**     | Heard a Hello, but *my* RID isn't in it yet                             | one-way Hellos (unidirectional link/filter)  |
| **2-Way**    | Both routers see each other; DR/BDR decided                            | **normal resting state between two DROthers**|
| **ExStart**  | Negotiating master/slave + initial DBD sequence number                 | **MTU mismatch**, or unicast blocked         |
| **Exchange** | Swapping DBD packets (LSA *headers* only)                              | MTU mismatch (again), packet loss            |
| **Loading**  | Sending LSRs for LSAs I'm missing; receiving LSUs                      | rare; loss or a corrupt LSA                   |
| **Full**     | LSDBs synchronized — adjacency complete                                | (this is the goal)                            |

Two things people misread:

- **2-Way is not a failure.** On a multi-access LAN, two **DROther** routers
  deliberately *stay* at 2-Way with each other — they only go Full with the DR and
  BDR. Seeing `2WAY/DROTHER` between two non-DR routers is correct (see [DR and BDR
  election](#dr-and-bdr-election)).
- **ExStart/Exchange stuck ⇒ think MTU.** The master/slave DBD exchange fails if
  the two sides disagree on interface MTU, because OSPF checks it. This is the
  single most common "neighbors won't come up but Hellos are fine" cause.

[↩ back to README Lab 1](README.md#lab-1--bring-up-the-first-adjacency-area-0-lan)

---

## Hello packets and what must match

Hellos (multicast to **224.0.0.5** every 10s on broadcast/p2p) do three jobs:
**discover** neighbors, **agree** on parameters, and act as a **keepalive** (miss
them for the Dead interval — default 4× Hello = 40s — and the neighbor is torn
down). A Hello carries the fields that **must match** for an adjacency to form:

| Hello field                 | Must match? | Notes                                                    |
|-----------------------------|-------------|----------------------------------------------------------|
| **Area ID**                 | ✅          | both interfaces must be in the same area                 |
| **Hello / Dead interval**   | ✅          | mismatched timers → no adjacency                         |
| **Network mask**            | ✅ (mostly) | on broadcast links; catches /24-vs-/30 typos             |
| **Authentication**          | ✅          | type + key must match                                    |
| **Stub/NSSA flags (E/N bit)** | ✅        | area-type flags in the Options field                     |
| **MTU**                     | ✅ (in DBD) | not in the Hello, but checked at [ExStart/Exchange](#the-neighbor-state-machine) |
| **Router-ID**               | must be *unique* | two routers with the same RID is a distinct failure |

The **RID** is not matched — it must be *different*. It's chosen as: manual
`router-id` > highest **loopback** IP > highest active physical IP, and it's
**sticky** (set at process start; change it and you must `clear ip ospf process`).
That's why every lab pins `router-id x.x.x.x` on a loopback — deterministic and
stable.

The Hello also carries the current **DR/BDR** (as IP addresses) and the sender's
**list of known neighbors** — seeing your own RID in a neighbor's Hello is the
event that moves you [Init → 2-Way](#the-neighbor-state-machine).

[↩ back to README Lab 1.2](README.md#12-enable-ospf-on-r1-only)

---

## Database synchronization: DBD, LSR, LSU, LSAck

Once two routers reach [ExStart](#the-neighbor-state-machine), they synchronize
databases with a reliable four-message exchange. This is OSPF's equivalent of "let
me show you my table of contents, you tell me what you're missing, I'll send it,
you acknowledge."

| Msg   | Type | Role                                                                 |
|-------|------|---------------------------------------------------------------------|
| **DBD**   | 2 | **Database Description** — LSA *headers only* (a table of contents). Master/slave is negotiated here; the I/M/MS bits sequence it. |
| **LSR**   | 3 | **Link-State Request** — "send me the full copy of these LSAs I don't have / that are newer." |
| **LSU**   | 4 | **Link-State Update** — carries the actual **LSAs**. Also used for all ongoing flooding. |
| **LSAck** | 5 | **Link-State Acknowledgment** — every LSA is explicitly acked; this is what makes flooding *reliable*. |

Key ideas:

- **DBDs are headers, not routes.** The receiver compares headers against its own
  LSDB and only *requests* (LSR) what it lacks or what's newer — efficient.
- **Master/slave:** the higher RID becomes master and controls the DBD sequence
  numbers. The negotiation is what ExStart *is*.
- **Flooding never really stops.** After initial sync, any topology change
  generates a new LSU that floods area-wide, each hop re-acked. LSAs also refresh
  every 30 minutes (LSRefreshTime) even with no change, and age out at 60 minutes
  (MaxAge) if the originator disappears.

[↩ back to README Lab 1.4](README.md#14-read-the-adjacency-off-the-capture-the-payoff)

---

## DR and BDR election

On a **multi-access** segment (broadcast or NBMA), N routers would form N×(N−1)/2
full adjacencies and flood redundantly — an O(N²) mess. OSPF fixes this by electing
a **Designated Router**: everyone forms a full adjacency with the **DR** (and a
**Backup DR** for resilience) and *only* the DR. The DR represents the segment to
the rest of the area via a [Network-LSA](#the-lsa-types).

**How the election works:**

1. **Highest interface priority wins** (`ip ospf priority`, default 1; **0 = never
   eligible**).
2. **Tie → highest RID wins.**
3. **It is non-preemptive.** A better router showing up *later* does **not** take
   over — the roles stick until the DR/BDR fails or you `clear ip ospf process`.
   This surprises people constantly: raise a priority and nothing happens until you
   clear.

**Traffic patterns you'll capture:**

- DROthers send updates to **224.0.0.6** (AllDRouters) — only the DR/BDR listen.
- The DR floods back out to **224.0.0.5** (AllSPFRouters).
- DROther-to-DROther adjacencies rest at **2-Way** on purpose (see [state
  machine](#the-neighbor-state-machine)).

**Design note:** on a well-run network you *pin* the DR to a stable, well-connected
router with priority and make edge routers ineligible (priority 0) — exactly the
NBMA hub-and-spoke pattern in README Lab 7. And DR/BDR is **irrelevant on
point-to-point** links — there's nothing to represent, so there's no election.

[↩ back to README Lab 2](README.md#lab-2--drbdr-election-and-how-to-rig-it)

---

## Network types in depth

The interface **network type** decides three things at once: whether there's a
**DR/BDR**, how neighbors are **discovered**, and the default **timers**. IOS
picks a default from the media, but you override with `ip ospf network …`.

| Type                    | DR/BDR? | Discovery              | Hello/Dead | Default on         |
|-------------------------|---------|------------------------|------------|--------------------|
| **broadcast**           | **yes** | automatic (multicast)  | 10 / 40    | Ethernet           |
| **non-broadcast (NBMA)**| **yes** | **manual `neighbor`**  | **30 / 120**| Frame-Relay main   |
| **point-to-point**      | no      | automatic (multicast)  | 10 / 40    | HDLC/PPP, sub-if p2p|
| **point-to-multipoint** | no      | automatic (multicast)  | **30 / 120**| (manual)           |

The subtle exam points:

- **Timers differ by type**, so switching type can *also* break an adjacency via a
  Hello/Dead mismatch — even though "the link is up."
- **NBMA has DR but no multicast**, so you must configure static `neighbor`
  statements *and* control the election (spokes get priority 0) because a spoke
  that isn't fully meshed must never become DR.
- **point-to-multipoint has no DR and advertises /32 host routes** for each
  neighbor. That host-route behaviour is what lets it work correctly over a
  partial-mesh cloud where NBMA's single-DR assumption would black-hole traffic.
- **Both ends must agree** on a type where DR presence differs (broadcast ↔ p2p),
  or one side elects a DR the other ignores.

[↩ back to README Lab 3](README.md#lab-3--point-to-point-network-type-r1--r4)

---

## Areas: why OSPF partitions the topology

A single-area OSPF domain means **every** router holds **every** LSA and reruns SPF
on **every** change anywhere. That doesn't scale — LSDB size and SPF cost grow with
the network. **Areas** partition the topology so detailed link-state information
stays *local*:

- Inside an area, routers exchange full topology (Router/Network LSAs) and run SPF.
- At the boundary, an **ABR** does *not* leak that topology. It **summarizes** each
  area's reachable prefixes into the neighbouring area as **Type-3 Summary LSAs** —
  just *prefix + cost*, no per-link detail.
- So an internal router sees its own area in full detail and every other area as a
  flat list of prefixes. A change *inside* another area doesn't trigger SPF for
  you — only a summary update.

**The rules that follow from this design (and generate most OSPF tickets):**

- **Area 0 is the backbone.** *All* inter-area traffic transits it.
- **Every non-backbone area must connect to Area 0** (directly or via a [virtual
  link](#virtual-links-and-transit-areas)). Areas can't chain area1→area2→area0.
- **Area 0 must be contiguous.** Split it and half your network disappears.

This is also why summarization only happens at **ABRs** (inter-area) and **ASBRs**
(external) — the boundary is the only place that has both the specifics and a
reason to hide them.

[↩ back to README Lab 4](README.md#lab-4--multiple-areas-abrs-and-type-3-summary-lsas)

---

## The LSA types

LSAs are the *content* of the LSDB. Each type answers a specific question, has a
specific originator, and a specific flooding scope. The ones this lab produces:

| Type | Name             | Originated by      | Flooding scope            | Answers…                         |
|------|------------------|--------------------|---------------------------|----------------------------------|
| **1** | Router           | every router       | its area only             | "what links do I have, at what cost?" |
| **2** | Network          | **the DR** only    | its area only             | "who is attached to this multi-access segment?" |
| **3** | Summary          | **an ABR**         | into adjacent areas       | "here's a prefix in another area + cost" |
| **4** | ASBR-Summary     | **an ABR**         | into other areas          | "here's how to reach the ASBR"   |
| **5** | AS-External      | **an ASBR**        | whole domain (normal areas)| "here's a redistributed external prefix" |
| **7** | NSSA-External    | an ASBR in an NSSA | within the NSSA only       | Type-5's stand-in *inside* an NSSA (translated to T5 at the ABR) |
| **8** | Link *(v3)*      | every router       | one link only             | (OSPFv3) link-local addr + prefixes on a link |
| **9** | Intra-Area-Prefix *(v3)* | router/DR  | its area only             | (OSPFv3) the IPv6 prefixes, split out of Type-1/2 |

Mental hooks:

- **Type-1 and Type-2 carry topology; Type-3/4/5 carry reachability.** SPF runs on
  1/2 (the graph); 3/4/5 are prefixes hung onto the tree afterward.
- **Only the DR makes a Type-2.** No DR (p2p) → no Type-2.
- **Type-4 exists only because Type-5 needs it.** A far-away router has the external
  prefix (Type-5, "origin = ASBR X") but no idea how to reach ASBR X across areas —
  the ABR supplies that with a Type-4. You need *both* to install the route.
- **Type-8/9 are OSPFv3-only** and are the mechanism behind [what changed in
  v3](#ospfv3-what-actually-changed).

[↩ back to README Lab 4.3](README.md#43-inter-area-routes-and-the-type-3-summary-lsa)

---

## Router types

A router's "type" is just a description of *where its interfaces sit*, and a single
router can be several at once:

| Type                       | Definition                                             | In this lab       |
|----------------------------|--------------------------------------------------------|-------------------|
| **Internal**               | all interfaces in one area                              | R4, R5            |
| **Backbone**               | ≥1 interface in Area 0                                  | R1, R2, R3        |
| **ABR** (Area Border)      | interfaces in ≥2 areas, **one of them Area 0**          | R1, R2            |
| **ASBR** (AS Boundary)     | injects external routes (redistribution)               | R3 (and R5 in NSSA lab) |

Why it matters: **ABRs originate Type-3/4**, **ASBRs originate Type-5/7**, and the
"must connect to Area 0" rule is really a rule about ABRs. An ABR with *no* Area 0
adjacency (README Lab 9) fails to inject its areas into the backbone — the problem a
[virtual link](#virtual-links-and-transit-areas) solves.

[↩ back to README end-state](README.md#suggested-end-state-verification)

---

## Area types: stub, totally-stubby, NSSA

Beyond partitioning, you can make an area carry *less* by having its ABR filter
certain LSAs and substitute a **default route**. The knob is what the ABR blocks:

| Area type            | Blocks Type-5? | Blocks Type-3? | Local ASBR (Type-7)? | Default route in       |
|----------------------|:--------------:|:--------------:|:--------------------:|------------------------|
| **normal**           | no             | no             | n/a                  | —                      |
| **stub**             | ✅             | no             | ❌ (not allowed)     | `O*IA 0.0.0.0/0`       |
| **totally-stubby**   | ✅             | ✅             | ❌                   | `O*IA 0.0.0.0/0`       |
| **NSSA**             | ✅             | no             | ✅ (via Type-7)      | optional (`O*N2`)      |
| **totally-NSSA**     | ✅             | ✅             | ✅                   | `O*N2 0.0.0.0/0`       |

The rules that trip people up:

- **Every router in the area must agree** on the type — it's carried as the **E-bit
  (stub) / N-bit (NSSA)** in the Hello Options. A mismatch drops the adjacency
  (which is *why* README Lab 6/8 have you watch the neighbor bounce).
- **`no-summary` (totally-stubby / totally-NSSA) goes on the ABR only.** Internal
  routers just see the result (Type-3 gone, one default remains).
- **NSSA default is not automatic.** Plain `area x nssa` injects **no** default;
  you need `no-summary` *or* `default-information-originate`. This is a classic
  "NSSA spoke can't reach the internet" bug.
- **Backbone can't be stub**, and a **stub/NSSA can't be a transit area** — so a
  [virtual link](#virtual-links-and-transit-areas) can't cross one.

[↩ back to README Lab 6](README.md#lab-6--stub-and-totally-stubby-areas)

---

## External routes: E1 vs E2, Type-4, forwarding address

When an ASBR redistributes a non-OSPF route it becomes a **Type-5 AS-External LSA**
(or [Type-7](#area-types-stub-totally-stubby-nssa) inside an NSSA). The metric can
be one of two types, and the difference is a favourite exam and troubleshooting
topic:

- **E2 (default):** the metric is the **redistributed cost only** — the internal
  cost to *reach* the ASBR is ignored. So every router in the domain sees the
  **same** metric for the prefix, no matter how far from the ASBR it is. Simple,
  but with **two ASBRs** advertising the same prefix, E2 can't tell which is
  closer.
- **E1:** the metric is **external cost + internal cost to the ASBR**, so it
  **grows with distance** and each router independently picks the *nearest* ASBR.
  Use E1 when a prefix has multiple exit points and you want closest-exit routing.

Two supporting mechanisms:

- **Type-4 ASBR-Summary** tells routers in *other* areas how to reach the ASBR (the
  Type-5 says "origin = ASBR X" but not the path to X). Same-area routers don't need
  it — they see the ASBR in their own Router-LSAs.
- **Forwarding address:** a Type-5 can name a next-hop other than the ASBR itself
  (often 0.0.0.0 = "use the ASBR"). In an NSSA the Type-7 carries a **non-zero**
  forwarding address, which matters when the ABR translates it to a Type-5.

Externals sort **below** all internal routes regardless of cost — see [path
preference](#path-preference-and-route-selection).

[↩ back to README Lab 5](README.md#lab-5--redistribution-making-r3-an-asbr-type-5--type-4-lsas)

---

## Path preference and route selection

When one OSPF process learns the same prefix multiple ways, it chooses by **route
type first, cost second** — cost only breaks ties *within* a type:

1. **Intra-area (`O`)** — always wins, even over a lower-cost inter-area path
2. **Inter-area (`O IA`)**
3. **External / NSSA Type-1 (`O E1` / `O N1`)**
4. **External / NSSA Type-2 (`O E2` / `O N2`)**

The counterintuitive part: an **`O` beats an `O IA` to the same prefix even if the
IA route has a lower cost.** "Traffic is taking the long way" is very often exactly
this — a suboptimal intra-area path winning on *type*. Within externals, **E1 beats
E2** (an E1 route is preferred over an E2 for the same prefix), and only then does
cost decide.

Step above OSPF: when **different protocols** offer the same prefix, **administrative
distance** decides before any of this — OSPF is **110**, EIGRP internal **90**,
static **1**, etc. That cross-protocol contest is the redistribution lab's territory;
`show ip route` shows it as the first number in `[110/20]`.

[↩ back to README Lab 10](README.md#lab-10--path-preference-intra--inter--e1--e2)

---

## Cost and the reference-bandwidth trap

OSPF's metric is **cost**, and by default `cost = reference-bandwidth ÷
interface-bandwidth`, with reference-bandwidth stuck at its 1988 value of **100
Mbps**. Integer division, minimum 1. The trap:

| Interface | Bandwidth | Default cost (ref 100M) |
|-----------|-----------|-------------------------|
| 10 Mbps   | 10 Mbps   | 10                      |
| 100 Mbps  | 100 Mbps  | 1                       |
| 1 Gbps    | 1 Gbps    | **1** (rounds down)     |
| 10 Gbps   | 10 Gbps   | **1** (rounds down)     |

Everything ≥100 Mbps costs **1** — OSPF can't tell a FastEthernet from a 10-Gig
link. The fix is `auto-cost reference-bandwidth 100000` (100 Gbps) on **every**
router (it must match domain-wide, or costs are inconsistent). You can also just
pin `ip ospf cost N` on an interface to override entirely.

Cost is **summed along the path in the outbound direction** during SPF, so tuning
one interface's cost shifts which path SPF prefers — the clean way to steer traffic
without touching bandwidth (which affects QoS and other features).

[↩ back to README quick extras](README.md#quick-extras-short-high-value)

---

## Virtual links and transit areas

Two OSPF rules can be violated by real-world growth: *every area must touch Area 0*,
and *Area 0 must be contiguous*. A **virtual link** is the repair — it stitches a
backbone adjacency **through** a non-backbone **transit area**.

Concretely, it's a logical link between two **ABRs** that runs *across* an
intermediate area (the transit area), giving an otherwise-stranded ABR a **logical
Area 0 connection**. Two uses:

- **Attach a detached area:** an area that only reaches the backbone through another
  area (README Lab 9 — Area 3 behind R4, reachable only via Area 1).
- **Repair a partitioned backbone:** two halves of Area 0 joined through a
  neighbouring area.

The constraints (all exam traps):

- Endpoints are the two **ABRs**, referenced by **RID**, and *both* must sit in the
  **transit area**.
- The **transit area cannot be a stub/NSSA** — a virtual link can't cross one, so
  you can't have made it stub earlier.
- Virtual-link OSPF packets are **unicast** between the endpoints (not the usual
  224.0.0.5 multicast) — visible on a capture of the transit link.
- It's a *fix*, not a *design*: the right long-term answer is usually to re-cable so
  the area touches Area 0 directly.

[↩ back to README Lab 9](README.md#lab-9--virtual-links--transit-areas)

---

## Authentication

OSPFv2 authenticates per-interface (or per-area), and you can watch the difference
on the wire:

| Type | Name            | On the wire                                              |
|------|-----------------|---------------------------------------------------------|
| 0    | Null            | no authentication (default)                             |
| 1    | Plain-text      | password sent **in cleartext** in every packet — visible in the capture |
| 2    | Cryptographic (MD5/SHA) | a **key-ID + digest** is appended; the key itself never travels |

Points to internalize:

- **Auth must match to peer** — it's verified from the Hello, so a mismatch drops
  the adjacency like any other Hello-field mismatch.
- **Type 1 is a security lesson, not a security control** — capturing it shows the
  password in plain sight. Always prefer type 2.
- **OSPFv3 is different:** it drops the built-in auth field and relies on **IPsec**
  (or a modern auth-trailer), because in v3 the address family and security moved
  to the IPv6 layer — see [OSPFv3](#ospfv3-what-actually-changed).

[↩ back to README quick extras](README.md#quick-extras-short-high-value)

---

## OSPFv3: what actually changed

OSPFv3 is **the same protocol** — same FSM, same DR/BDR, same areas, same SPF, same
LSA-flooding — retargeted at IPv6. What actually changed:

- **Runs over IPv6, per-link.** Protocol 89 over IPv6, multicast **FF02::5 /
  FF02::6**, sourced from the interface **link-local** address. Enable **per
  interface** (`ospfv3 1 ipv6 area N`) — there are **no `network` statements**.
- **Topology and addressing are decoupled.** In v2 the Router/Network LSAs carried
  both the graph *and* the prefixes. In v3 they carry **topology only**; IPv6
  prefixes moved into the new **[Type-9 Intra-Area-Prefix-LSA](#the-lsa-types)**,
  and the **Type-8 Link-LSA** carries per-link link-local + prefix info. Result: you
  can re-address a link without re-running SPF.
- **Still needs a 32-bit RID.** OSPFv3 keeps the IPv4-style router-id; with no IPv4
  interfaces you *must* set it manually or the process won't start.
- **Address-family model.** Modern IOS `router ospfv3` holds **both** an IPv6 and an
  IPv4 address family in one process — the "address families" ENARSI calls out. The
  older `ipv6 router ospf` syntax is IPv6-only.
- **Auth via IPsec**, not the v2 key fields (see [Authentication](#authentication)).

Everything you learned for v2 — [state machine](#the-neighbor-state-machine), [DR
election](#dr-and-bdr-election), [area types](#area-types-stub-totally-stubby-nssa),
[path preference](#path-preference-and-route-selection) — transfers unchanged.

[↩ back to README Lab 11](README.md#lab-11--ospfv3-ospf-for-ipv6-with-address-families)
