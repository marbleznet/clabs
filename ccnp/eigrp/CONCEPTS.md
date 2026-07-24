# EIGRP — In-Depth Concepts

The companion reference to the [README](README.md) lab guide. The README is the
*hands-on* path (configure this, capture that); this file is the *why* behind each
step. Each 📖 **Deep dive** link in the README jumps to a section here. You can
also read it straight through as an EIGRP primer.

**Contents**
- [What EIGRP is and how it differs from OSPF](#what-eigrp-is-and-how-it-differs-from-ospf)
- [Neighbors and reliable transport (RTP)](#neighbors-and-reliable-transport-rtp)
- [The three tables](#the-three-tables)
- [The composite metric](#the-composite-metric)
- [DUAL and loop-free path selection](#dual-and-loop-free-path-selection)
- [Convergence: passive, active, and queries](#convergence-passive-active-and-queries)
- [Stuck-in-Active (SIA)](#stuck-in-active-sia)
- [Bounding the query domain: stub and summarization](#bounding-the-query-domain-stub-and-summarization)
- [Load balancing: equal and unequal cost](#load-balancing-equal-and-unequal-cost)
- [Classic vs named mode](#classic-vs-named-mode)
- [Authentication](#authentication)
- [auto-summary and discontiguous networks](#auto-summary-and-discontiguous-networks)
- [External routes and redistribution](#external-routes-and-redistribution)
- [EIGRP for IPv6](#eigrp-for-ipv6)

---

## What EIGRP is and how it differs from OSPF

EIGRP is Cisco's **advanced distance-vector** protocol (once marketed as "hybrid").
That label matters: unlike a link-state protocol, **no router ever builds a full map
of the network**. Each router only knows what its **neighbors report** — a distance
to each destination — and it picks the best. What makes it "advanced" is the
algorithm on top of those reported distances: **DUAL** (Diffusing Update Algorithm),
which pre-computes a guaranteed **loop-free backup** so most failures reconverge in
*milliseconds* without any recomputation.

The contrasts you'll feel throughout this lab versus [`../ospf`](../ospf):

| | OSPF (link-state) | EIGRP (advanced distance-vector) |
|---|---|---|
| What each router knows | full **map** of the area (LSDB) | only neighbors' **reported distances** |
| Best-path math | Dijkstra SPF over the map | **DUAL** over reported distances |
| Backup path | recompute SPF on failure | **feasible successor** — pre-computed |
| Multi-access segment | elects a **DR/BDR** | **no DR** — every router peers directly |
| Metric | cost (bandwidth only) | composite **bandwidth + delay** (K-values) |
| Unequal-cost load-balance | ✗ | ✓ (`variance`) |

EIGRP rides directly on **IP protocol 88**, multicasts to **224.0.0.10**, and has an
administrative distance of **90 (internal)** / **170 (external)** — lower than OSPF's
110, so an EIGRP internal route wins if both protocols offer the same prefix.

[↩ back to README Lab 1](README.md#lab-1--bring-up-eigrp-neighbors-the-reliable-transport)

---

## Neighbors and reliable transport (RTP)

EIGRP discovers neighbors with **Hellos** (multicast, every 5s on LAN/fast links,
60s on slow ≤T1 links; **hold time** = 3× hello). But Hellos are unreliable — the
routing information itself is carried reliably by **RTP (Reliable Transport
Protocol)**, EIGRP's own acknowledged-delivery layer on top of IP.

Five packet types (opcodes) do everything:

| Opcode | Packet         | Reliable? | Job                                              |
|--------|----------------|-----------|--------------------------------------------------|
| 5      | **Hello / ACK**| no        | discover/maintain neighbors; ACK = Hello w/ ack# |
| 1      | **Update**     | **yes**   | advertise routes (INIT flag = full table sync)   |
| 3      | **Query**      | **yes**   | "I lost my path — does anyone have one?"          |
| 4      | **Reply**      | **yes**   | answer to a Query                                |
| 10/11  | **SIA-Query/Reply** | **yes** | liveness probe during a long Active state     |

Reliable packets (Update/Query/Reply) carry a **sequence number** and must be
**ACKed**; RTP tracks each with an **SRTT** (smooth round-trip time) and **RTO**
(retransmit timeout), and holds a per-neighbor queue. That's why EIGRP needs no
periodic reflooding — it sends the whole table **once** at neighbor-up (Updates with
the **INIT** flag), then only **incremental** changes.

**To *become* neighbors, two routers must agree on:** the **AS number**, the
**K-values**, be on the **same subnet**, and pass **authentication** if configured.
Note what is *not* required: the **hello/hold timers may differ** between neighbors
(each advertises its own hold time), and there's no MTU check like OSPF's.

[↩ back to README captures](README.md#using-edgeshark-for-captures)

---

## The three tables

EIGRP keeps three tables, and knowing which is which makes every `show` command
obvious:

1. **Neighbor table** (`show ip eigrp neighbors`) — who I'm adjacent to, plus each
   neighbor's SRTT/RTO/hold and the RTP queue. The input to everything else.
2. **Topology table** (`show ip eigrp topology`) — *every* destination with its
   **successor(s)** and any **feasible successors** — i.e. the full loop-free set
   DUAL is willing to use. This is EIGRP's "working memory." `show ip eigrp topology
   all-links` additionally shows paths that **fail** the feasibility test.
3. **Routing table** (`show ip route eigrp`) — only the **successors** (the winners)
   get installed as `D` routes for forwarding.

The distinction that trips people up: a **feasible successor lives in the topology
table, not the routing table.** It's a pre-qualified backup sitting ready; it only
reaches the RIB if the successor fails (or if [`variance`](#load-balancing-equal-and-unequal-cost)
installs it for load-balancing).

[↩ back to README Lab 2](README.md#lab-2--the-topology-table--dual-successor--feasible-successor)

---

## The composite metric

EIGRP's metric is **composite** — built from up to five weighted components via the
**K-values** (`K1..K5`). Defaults are **K1=K3=1, K2=K4=K5=0**, which means only
**bandwidth** and **delay** count (and both ends must have matching K-values or they
won't peer). The classic 32-bit formula reduces to:

```
metric = 256 × ( 10^7 / min_bandwidth_kbps  +  Σ delay_in_tens_of_µsec )
                     └─ slowest link on path ┘   └─ cumulative interface delay ┘
```

Two practical consequences:

- **Bandwidth is the *minimum* along the path** (a chain is as fast as its slowest
  link); **delay is *cumulative*** (it adds up hop by hop). That's why, in README
  Lab 2, bumping **delay** on one interface cleanly demotes a path — it adds to the
  sum without touching the min-bandwidth term.
- **Prefer tuning `delay` over `bandwidth`** for path selection. The `bandwidth`
  command also feeds QoS, SNMP, and other features; `delay` is a "pure" routing
  knob.

**Wide metrics** ([named mode](#classic-vs-named-mode)) widen the metric to **64
bits** and add a throughput/latency scaling so EIGRP can distinguish links **faster
than ~1 Gbps** — classic 32-bit metrics saturate (every link ≥ a few Gbps looks
identical), exactly the way OSPF's default reference bandwidth does. The values are
much larger numbers but the selection logic is unchanged.

[↩ back to README Lab 5.1](README.md#51-what-the-metric-is-made-of)

---

## DUAL and loop-free path selection

DUAL is the heart of EIGRP. It works with three numbers per destination:

| Term | Meaning |
|------|---------|
| **FD** — Feasible Distance | **my** best (lowest) metric to the destination |
| **RD / AD** — Reported (Advertised) Distance | a **neighbor's** metric to the destination, as it told me |
| **Successor** | the neighbor offering the lowest total metric (the FD) — installed in the RIB |
| **Feasible Successor (FS)** | a backup neighbor that passes the feasibility condition |

The one rule that makes it all loop-free — the **Feasibility Condition (FC):**

> A neighbor is a **feasible successor** if its **Reported Distance is *less than* my
> Feasible Distance** (**RD < FD**).

Why that guarantees no loop: if a neighbor is already **closer** to the destination
than my current best metric, it cannot possibly be routing **through me** to get
there (that would make its distance larger, not smaller). So handing traffic to it
can't create a loop — and DUAL can install it **instantly**, with no network-wide
recomputation. This is EIGRP's superpower: the backup is proven safe *in advance*.

A subtle but exam-relevant point: the FC uses the **FD**, which is the *historically
lowest* metric since the route last went [Passive](#convergence-passive-active-and-queries)
— not necessarily the current metric. That conservatism is deliberate; it's why a
path can be a valid alternate yet still fail the FC and *not* qualify as an FS.

[↩ back to README Lab 2.2](README.md#22-make-r1-the-successor-and-r2-the-feasible-successor)

---

## Convergence: passive, active, and queries

Every topology-table entry is either **Passive (P)** — stable, converged — or
**Active (A)** — "I've lost my path and I'm actively looking." **Passive is the
healthy state**; Active is a route mid-recomputation. What happens when a successor
dies depends entirely on whether an [FS](#dual-and-loop-free-path-selection) exists:

- **FS exists → the route never leaves Passive.** DUAL promotes the feasible
  successor locally, in place, with **no query and no delay** — sub-second, and
  invisible except for a routine Update. (README Lab 3A.)
- **No FS → the route goes Active.** The router has no pre-proven backup, so it
  **multicasts a Query** to its neighbors: *"do you have a path to X?"* It can't
  finalize a new successor until it has collected a **Reply** from **every** queried
  neighbor. (README Lab 3B.)

**Query propagation** is the risk. A neighbor that also has no FS goes Active *itself*
and queries *its* neighbors — the query **diffuses** outward (that's the "D" in DUAL)
until it reaches a boundary that can answer: a router with an alternate path, a
[stub, or a summarization boundary](#bounding-the-query-domain-stub-and-summarization),
or the edge of the network. An unbounded query domain is how a single flap turns into
a network-wide event — and how you get [SIA](#stuck-in-active-sia).

[↩ back to README Lab 3](README.md#lab-3--convergence-two-ways-feasible-successor-vs-going-active)

---

## Stuck-in-Active (SIA)

A route is **Stuck-in-Active** when a router has gone Active and **can't get all its
Replies back** in time. IOS runs an **active timer** (default **3 minutes**); as a
safety check it sends an **SIA-Query** partway through and expects an **SIA-Reply**
proving the neighbor is alive and still working on the answer. If a neighbor never
answers, the querying router declares SIA and **resets that neighbor** — tearing down
the adjacency and everything learned through it, which itself causes more churn.

SIA is almost always a **symptom**, not a root cause. The usual culprits:

- an **oversized query domain** (queries diffusing too far — the #1 cause),
- a **slow, congested, or flapping link** delaying Replies,
- a **low-resource or stuck neighbor** that can't respond.

The fix is architectural, and it's the whole point of the next section: **make the
query domain smaller** so Active states resolve locally and never get the chance to
get stuck.

[↩ back to README Lab 3B](README.md#3b--no-feasible-successor--go-active--flood-queries)

---

## Bounding the query domain: stub and summarization

Because [queries diffuse](#convergence-passive-active-and-queries), the single most
important EIGRP scaling/troubleshooting skill is **stopping queries from spreading**.
Two tools:

**EIGRP stub** (`eigrp stub [connected] [summary] [static] [redistributed]
[receive-only]`) — marks a router as an **edge**: it advertises only its *own*
routes (per the keywords), **never transit routes**, and its neighbors **never send
it a Query**. In a hub-and-spoke design, making every spoke a stub means a hub going
Active only ever queries *other hubs*, never the (numerous) spokes — the definitive
SIA fix. The stub flag is advertised right in the **Hello's Parameters TLV**, so a
neighbor knows not to query it.

**Summarization** (`ip summary-address eigrp <as> <net> <mask>` per interface) —
replaces a set of specific prefixes with one summary at a boundary, and installs a
**Null0** discard route behind it (so the summarizing router can't loop packets for a
missing component). The query-scoping magic: a router that summarizes a block
**answers queries for any component of that block itself** instead of propagating them
— so a summary boundary is *automatically* a query boundary. Smaller tables **and** a
bounded query domain.

> The ENARSI rule of thumb: **every SIA fix is "shrink the query domain"** — summarize
> at distribution/core boundaries, and make edge routers stubs.

[↩ back to README Lab 4](README.md#lab-4--bounding-the-query-domain-stub--summarization)

---

## Load balancing: equal and unequal cost

EIGRP installs **equal-cost** paths automatically — up to `maximum-paths` (default
**4**, up to 16/32). When R4's two uplinks tie (README Lab 2), both go in the RIB and
traffic splits.

The feature OSPF can't match is **unequal-cost** load balancing via **`variance N`**:
EIGRP will additionally install any **feasible successor** whose metric is within
**N ×** the successor's FD, and share traffic **in inverse proportion to metric** (the
worse path carries proportionally less). Two guardrails to memorize:

- **`variance` only ever uses feasible successors.** A path that fails the
  [feasibility condition](#dual-and-loop-free-path-selection) is *never* installed, no
  matter how large N is — so unequal-cost balancing **cannot create a loop**.
- It's **ratio-based, not "any path."** `variance 2` means "up to twice the best
  metric"; paths beyond that ratio still don't qualify.

[↩ back to README Lab 5.2](README.md#52-unequal-cost-load-balancing-with-variance)

---

## Classic vs named mode

Two ways to configure the same protocol:

- **Classic** (`router eigrp <AS>`) — the historic style: `network` statements
  activate interfaces, and per-interface knobs (`ip hello-interval eigrp`,
  `ip authentication …`, `ip summary-address …`) are scattered on the interfaces.
- **Named** (`router eigrp <NAME>`) — a single hierarchical config that holds
  **every address family and every interface setting in one place**:
  `address-family ipv4|ipv6 unicast autonomous-system <AS>`, with `af-interface`
  sub-blocks and a `topology base` section. It **defaults to wide metrics** and is
  required for newer features (e.g. SHA authentication).

They are the **same protocol on the wire** — a named-mode router and a classic-mode
router with the **same AS and K-values interoperate as neighbors**. Named mode is
just a cleaner container; nothing about DUAL, metrics, or packets changes.

[↩ back to README Lab 5.3](README.md#53-optional-named-mode--wide-metrics)

---

## Authentication

EIGRP authenticates **per interface** using a **key chain** (so you can rotate keys
on a schedule). Classic mode supports **MD5**; named mode adds **HMAC-SHA-256**.

```
key chain KC
 key 1
  key-string SECRET
interface e0/2
 ip authentication mode eigrp 100 md5
 ip authentication key-chain eigrp 100 KC
```

Points that matter on the wire and in troubleshooting:

- A mismatched or missing key **prevents the adjacency** — you'll see
  `%DUAL-5-NBRCHANGE … Auth failure` and the neighbor never forms. Authentication is
  one of the *required-to-match* items alongside AS and K-values.
- The digest travels in an **Authentication TLV** in each packet (visible in the
  capture); the key itself never does.
- **Key chains carry lifetimes** (`accept-lifetime` / `send-lifetime`). A very common
  real-world outage is a key expiring, or clocks not matching across routers, so the
  active key differs — worth knowing when an adjacency drops "for no reason."

[↩ back to README Lab 6.1](README.md#61-md5-authentication-watch-the-adjacency-drop-then-match)

---

## auto-summary and discontiguous networks

**Auto-summary** makes EIGRP summarize routes to their **classful boundary** when
advertising across a different major network. In this lab that's `10.0.0.0/8` and
`172.16.0.0/16`. It's **off by default** on modern IOS for a reason:

When a classful network is **discontiguous** — its subnets separated by a *different*
major network, exactly our topology, where pieces of `172.16.0.0/16` (R3/R4/R5's edge
nets) sit across the `10.0.0.0/8` core — auto-summary makes **multiple routers all
advertise the same classful `172.16.0.0/16`**. Their peers see equal-looking summaries
from several directions and **black-hole** whichever specific subnets hash to the wrong
one. That's why `no auto-summary` is reflexive, and why manual, *deliberate*
[summarization](#bounding-the-query-domain-stub-and-summarization) at a real boundary is
the right tool instead.

[↩ back to README Lab 6.2](README.md#62-the-auto-summary-gotcha)

---

## External routes and redistribution

Routes **redistributed into** EIGRP from another source (connected, static, OSPF, BGP…)
become **external** EIGRP routes: they show as **`D EX`** with administrative distance
**170** (vs `D`/**90** for internal), so an EIGRP-internal route is always preferred
over an EIGRP-external one for the same prefix.

On the wire they travel in a different TLV — an **External route TLV** (`0x0103`)
instead of the internal `0x0102` — carrying extra fields that internal routes don't
have:

- the **originating router ID**,
- the **source protocol** and its **AS/process**,
- the **external metric** and a **route tag**.

The **originating-router-id** is EIGRP's redistribution **loop-prevention**: when two
routers mutually redistribute between EIGRP and another protocol, a router that sees
its *own* router-id on a returning external route drops it, so the route can't loop
back in. (Route **tags** — set/matched in route-maps — are the manual, scalable way to
do the same thing across many redistribution points; that's the focus of the upcoming
redistribution lab.)

Redistributed routes need a **seed metric** (`redistribute … metric bw delay
reliability load mtu`, or `default-metric`) because a non-EIGRP source has no EIGRP
composite metric — without one, IOS classic mode won't advertise them.

[↩ back to README Lab 6.4](README.md#64-bridge-to-the-next-lab-redistribution-teaser)

---

## EIGRP for IPv6

EIGRP for IPv6 is the **same DUAL protocol** with a new address family — everything
about successors, feasible successors, queries, stubs, and variance is **identical**.
What changes is only the wrapper and the on-the-wire encapsulation:

- Rides **IP protocol 88 over IPv6**, multicasts to **FF02::A** (not 224.0.0.10), and
  sources packets from the interface **link-local** (`fe80::`) address.
- **No `network` statements** — you enable it **per interface** (`ipv6 eigrp <AS>` in
  classic mode; automatic in named mode).
- Still uses a **32-bit router-id**, which you must set **manually** if the router has
  no IPv4 interface to borrow one from.
- **No `auto-summary`** — IPv6 has no classful concept.

Two config styles, same result:

- **Classic:** `ipv6 router eigrp <AS>` — and it starts **administratively shut down**,
  so `no shutdown` under the process is *the* classic gotcha.
- **Named:** `address-family ipv6 unicast autonomous-system <AS>` inside `router eigrp
  <NAME>` — the modern, unified form (and it interoperates with classic-mode peers on
  the same AS).

[↩ back to README Lab 7](README.md#lab-7--eigrp-for-ipv6-address-families-named-mode--classic)
