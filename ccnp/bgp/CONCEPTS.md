# BGP — In-Depth Concepts

The companion reference to the [README](README.md) lab guide. The README is the
*hands-on* path (configure this, capture that); this file is the *why* behind each
step. Each 📖 **Deep dive** link in the README jumps to a section here. You can
also read it straight through as a BGP primer.

**Contents**
- [What BGP is and when to use it](#what-bgp-is-and-when-to-use-it)
- [TCP 179 and the BGP session state machine](#tcp-179-and-the-bgp-session-state-machine)
- [The OPEN message and capabilities](#the-open-message-and-capabilities)
- [BGP message types](#bgp-message-types)
- [eBGP vs iBGP](#ebgp-vs-ibgp)
- [The iBGP full-mesh rule](#the-ibgp-full-mesh-rule)
- [Route reflectors](#route-reflectors)
- [Next-hop handling and next-hop-self](#next-hop-handling-and-next-hop-self)
- [BGP path attributes](#bgp-path-attributes)
- [The best-path algorithm](#the-best-path-algorithm)
- [Route filtering tools](#route-filtering-tools)
- [Path manipulation](#path-manipulation)
- [BGP communities](#bgp-communities)
- [MP-BGP and address families](#mp-bgp-and-address-families)
- [Authentication, timers, and session knobs](#authentication-timers-and-session-knobs)

---

## What BGP is and when to use it

BGP is the **routing protocol of the Internet** — it's how ~75,000 autonomous
systems exchange reachability. Unlike OSPF/EIGRP (which find the *shortest* path
inside your network), BGP's job is **policy** *between* networks: *whose* routes you
accept, *which* you advertise, and *which exit* you prefer — decisions driven by
business and trust, not just distance.

Two properties define it:

- **Path-vector.** A BGP route carries the full **list of ASes** it traversed (the
  **AS-path**). That list is both the loop-prevention mechanism (an eBGP speaker
  rejects a route whose AS-path already contains its own AS) and a selection input
  (shorter is preferred). Contrast with link-state (full map) or distance-vector
  (just a metric).
- **Policy over metric.** BGP doesn't compute a cost. It walks a **[best-path
  algorithm](#the-best-path-algorithm)** over **[path attributes](#bgp-path-attributes)**
  you can set — so you can deliberately choose a "longer" path for policy reasons.

You reach for BGP when you have **multiple ASes or multiple providers** (multihoming),
need **granular routing policy**, or must carry a **huge table** (the global table
is ~1M routes) that an IGP could never hold. Inside a single AS you still use an IGP;
BGP rides on top.

[↩ back to README top](README.md#ccnp-bgp-deep-dive-lab-cisco-iol--edgeshark)

---

## TCP 179 and the BGP session state machine

BGP is unusual: it has **no protocol of its own on the wire** — it's an application
that runs over a plain **TCP connection to port 179**. TCP gives BGP reliability,
ordering, and flow-control for free, which is why BGP has no acknowledgements or
sequence numbers of its own. It also means **peers are configured by hand** (no
multicast discovery) and must be **IP-reachable** before BGP can even start.

The session walks a finite state machine; the state tells you exactly what's wrong:

| State           | What's happening                                    | Stuck here means…                          |
|-----------------|-----------------------------------------------------|--------------------------------------------|
| **Idle**        | Nothing yet; looking for a route to the peer        | no route to peer, or admin `shutdown`      |
| **Connect**     | TCP handshake in progress                           | (transient)                                |
| **Active**      | TCP failed; retrying the connection                 | **flapping Active/Connect = TCP won't complete** — ACL on 179, wrong `update-source`, unreachable peer |
| **OpenSent**    | TCP up; OPEN sent, awaiting peer's OPEN             | OPEN parameter problem                     |
| **OpenConfirm** | OPENs exchanged; awaiting first KEEPALIVE           | auth (MD5) mismatch, hold-time issue       |
| **Established** | Session up; UPDATEs may flow                        | (the goal)                                 |

Two diagnostics to memorize: **Idle** → it's a *reachability* problem (fix routing
to the peer). **Active/Connect flapping** → it's a *TCP* problem (something is
blocking or resetting port 179). Both happen *before* any BGP policy is even
consulted.

[↩ back to README Lab 1](README.md#lab-1--your-first-ebgp-session-r1--r4)

---

## The OPEN message and capabilities

Once TCP is up, each side sends an **OPEN** to negotiate the session. Fields that
matter (expand one in the capture):

- **My Autonomous System** — must match the other side's `remote-as` expectation.
- **Hold Time** — the negotiated value is the **lower** of the two proposed;
  KEEPALIVEs are sent at ⅓ of it. Cisco defaults: keepalive **60s**, hold **180s**.
  Hold time must be **0** (never expire) or **≥ 3**.
- **BGP Identifier** — the **router-id** (32-bit, like OSPF). Two peers with the
  same RID is an error.
- **Optional Parameters → Capabilities** — this is how modern features are
  negotiated so both ends agree before use:
  - **MP-BGP** (multiprotocol, RFC 4760) — carries [address families](#mp-bgp-and-address-families) beyond IPv4.
  - **Route-Refresh** (RFC 2918) — enables soft re-application of [policy](#route-filtering-tools).
  - **4-octet AS** (RFC 6793) — 32-bit ASNs; 2-byte-only peers see `AS_TRANS` (23456).
  - Graceful restart, Add-Path, etc.

If a **required** parameter mismatches (AS number, an unsupported mandatory
capability, auth), the OPEN triggers a **NOTIFICATION** and the session drops back —
you'll see it never reach Established.

[↩ back to README Lab 1.3](README.md#13-watch-the-session-come-up)

---

## BGP message types

Only five message types exist; all share a 19-byte header (a 16-byte marker + length
+ type):

| Type | Message         | Reliable? | Role                                                        |
|------|-----------------|-----------|-------------------------------------------------------------|
| **1** | OPEN           | (TCP)     | Session setup: AS, hold-time, RID, capabilities. Once, up front. |
| **2** | UPDATE         | (TCP)     | The workhorse: advertise **NLRI** (prefixes) + **path attributes**, and/or **withdraw** routes. |
| **3** | NOTIFICATION   | (TCP)     | A fatal error occurred → the sender **closes the session**. |
| **4** | KEEPALIVE      | (TCP)     | 19-byte header only; a heartbeat so the hold timer doesn't expire. |
| **5** | ROUTE-REFRESH  | (TCP)     | "Please re-advertise everything" — used to re-apply inbound policy without a reset. |

The important mental model: **BGP is incremental and event-driven**. After the
initial burst of UPDATEs at convergence, a stable session sends almost nothing but
KEEPALIVEs. A single UPDATE can advertise many prefixes that share the same
attributes, and can carry withdrawals in the same message.

[↩ back to README captures](README.md#using-edgeshark-for-captures)

---

## eBGP vs iBGP

Whether a session is **external** or **internal** is decided by one thing — is the
neighbor's `remote-as` **different** from yours (eBGP) or the **same** (iBGP)? That
single fact changes five behaviours:

| Behaviour              | eBGP (different AS)                    | iBGP (same AS)                             |
|------------------------|----------------------------------------|--------------------------------------------|
| **Administrative distance** | **20**                            | **200**                                    |
| **TTL on packets**     | **1** (must be directly connected, unless [`ebgp-multihop`](#authentication-timers-and-session-knobs)) | 255 (any distance) |
| **AS-path**            | **prepends** your AS on advertisement  | **unchanged** (stays inside one AS)        |
| **Next-hop**           | set to the advertising router          | **preserved** → often needs [`next-hop-self`](#next-hop-handling-and-next-hop-self) |
| **Re-advertisement**   | routes go to all peers                 | **iBGP-learned routes are NOT relayed to other iBGP peers** ([full-mesh rule](#the-ibgp-full-mesh-rule)) |

Also: **local-preference** is only shared over iBGP (it's an AS-internal policy
signal), while **MED** is advertised *to* an eBGP neighbor but not propagated deeper
than the neighboring AS. The AS-path-based loop prevention only works *between* ASes,
which is exactly why iBGP needs a different loop-prevention rule.

[↩ back to README Lab 2](README.md#lab-2--ibgp-the-next-hop-problem-and-the-full-mesh-rule)

---

## The iBGP full-mesh rule

Because a route learned over iBGP has **no AS-path change** to detect a loop with,
BGP applies a blunt rule instead: **a route learned from an iBGP peer is never
advertised to another iBGP peer.** That prevents internal loops — but it means every
iBGP speaker must hear every route *directly* from its originator, i.e. **every iBGP
router must peer with every other**: a full mesh of **n(n−1)/2** sessions.

That's fine for 3 routers (3 sessions) but explodes at scale (100 routers = 4,950
sessions). Two mechanisms relax it:

- **[Route reflectors](#route-reflectors)** — one router is allowed to *reflect*
  iBGP routes, so clients peer only with it. (This lab.)
- **Confederations** — split the AS into sub-ASes that eBGP-peer internally. (Out of
  ENARSI scope.)

In this lab R1/R2/R3 start as a full mesh (Lab 2), then you remove the R1↔R2 session
and let **R3 reflect** (Lab 3) — the smallest possible demonstration of the rule and
its fix.

[↩ back to README Lab 3](README.md#lab-3--route-reflector-r3-reflects-between-r1-and-r2)

---

## Route reflectors

A **route reflector (RR)** is an iBGP router permitted to break the full-mesh rule
by *reflecting* routes. You configure it by declaring some neighbors as **clients**
(`neighbor x route-reflector-client`); everything else is a **non-client**. The
reflection rules:

- Route from a **client** → reflect to **all** other peers (clients, non-clients,
  and eBGP).
- Route from a **non-client** → reflect to **clients** and eBGP only (**not** other
  non-clients — those are assumed fully meshed).
- Route from an **eBGP** peer → advertise to all.

Because reflection re-introduces the loop risk the full-mesh rule removed, RRs add
two loop-prevention attributes:

- **Originator-ID** — the RID of the router that first injected the route into the
  AS. If a router receives a route with **its own** RID as Originator-ID, it drops
  it.
- **Cluster-list** — the list of **cluster-IDs** (each RR's ID, defaulting to its
  RID) the route has passed through. An RR that sees its own cluster-ID drops the
  route.

The client is unaware it's a client — no special config on R1/R2 — which makes RR a
drop-in for a full mesh. (ENARSI scope is a **single** RR; multiple-RR redundancy
and confederations are explicitly excluded.)

[↩ back to README Lab 3.3](README.md#33-verify-reflection)

---

## Next-hop handling and next-hop-self

The **NEXT_HOP** attribute says where to send traffic for a prefix — and BGP's
default handling of it is the #1 iBGP gotcha:

- On **eBGP** advertisement, the next-hop is set to the **advertising router's**
  interface address (normal — your eBGP peer is directly connected).
- On **iBGP** advertisement, the next-hop is **left unchanged**. So when R1 learns an
  external prefix (next-hop = the *external* peer's address) and passes it to iBGP
  peers, they inherit that external next-hop — an address that lives on the eBGP link
  and usually **isn't in the IGP**. The route is then **"inaccessible"** and unused.

Two fixes:
- **`neighbor x next-hop-self`** (this lab) — the iBGP-advertising router rewrites
  the next-hop to **its own** address (loopback), which the IGP *does* carry. Simple
  and standard on border routers.
- **Redistribute the eBGP link subnets into the IGP** — works, but pollutes the IGP;
  `next-hop-self` is preferred.

This is *why* an iBGP design always has an **IGP underlay** (Lab 0): the IGP's job is
to make BGP next-hops (and the iBGP loopbacks themselves) reachable. BGP carries the
prefixes; the IGP carries the plumbing.

[↩ back to README Lab 2.2](README.md#22-the-next-hop-problem)

---

## BGP path attributes

Every BGP route is a prefix plus a bundle of **path attributes**. They're grouped by
how well-supported and how transitive they are:

| Category                     | Attributes                                | Meaning                                     |
|------------------------------|-------------------------------------------|---------------------------------------------|
| **Well-known mandatory**     | **Origin**, **AS-path**, **Next-hop**     | every UPDATE must carry these               |
| **Well-known discretionary** | **Local-Preference**, Atomic-Aggregate    | all routers understand; may be absent       |
| **Optional transitive**      | **Community**, Aggregator                 | unknown routers pass them on                |
| **Optional non-transitive**  | **MED**, Originator-ID, Cluster-list       | unknown routers drop them                   |

The ones you'll actually manipulate:

- **AS-path** — list of ASes traversed; loop prevention + shorter-is-preferred. You
  lengthen it with [prepending](#path-manipulation).
- **Local-Preference** — AS-internal "preferred exit," **highest wins**, shared over
  iBGP. The main tool for choosing an outbound path AS-wide.
- **MED** (multi-exit discriminator) — a hint to a **neighboring AS** about which of
  *your* multiple links it should use to enter; **lowest wins**, compared only among
  paths from the **same** neighbor AS (unless `bgp always-compare-med`).
- **Origin** — how the route entered BGP: **i** (`network`/IGP) < **e** (legacy EGP)
  < **?** (redistributed/incomplete). Lower is better in best-path.
- **Weight** — Cisco-only, **not a real attribute** (never transmitted); local to one
  router, **highest wins**, checked first.

[↩ back to README Lab 4](README.md#lab-4--best-path-selection-and-the-attributes-that-drive-it)

---

## The best-path algorithm

When BGP has multiple valid paths to a prefix, it picks **one** best by comparing
attributes **in a fixed order** — first difference wins, otherwise fall through to
the next test. The Cisco order (memorize the top of it — that's where policy lives):

1. **Weight** — highest (local to the router, Cisco-only)
2. **Local-Preference** — highest (AS-wide policy)
3. **Locally originated** — a route this router injected (`network`/aggregate/
   redistribute) beats a learned one
4. **AS-path** — shortest
5. **Origin** — IGP (i) < EGP (e) < Incomplete (?)
6. **MED** — lowest (only among paths from the *same* neighbor AS)
7. **eBGP over iBGP** — prefer an externally-learned path
8. **Lowest IGP metric** to the BGP next-hop
9. Tiebreakers — oldest eBGP route, then lowest RID, then lowest cluster-list
   length, then lowest neighbor address

Two things to internalize: **weight and local-pref sit above AS-path**, so policy you
set locally overrides "shortest path" (that's the whole point — see README Lab 4.3
where local-pref 200 beats a shorter AS-path). And a path must be **valid** (its
[next-hop resolvable](#next-hop-handling-and-next-hop-self)) to even enter the
contest — an inaccessible next-hop disqualifies a path before step 1.

[↩ back to README Lab 4.4](README.md#44-walk-the-algorithm)

---

## Route filtering tools

BGP gives four inbound/outbound filters, applied **per neighbor** with a direction
(`in`/`out`). Pick by *what you're matching on*:

| Tool             | Matches on            | Configure with                             |
|------------------|-----------------------|--------------------------------------------|
| **prefix-list**  | the **prefix** (and length range) | `ip prefix-list` + `neighbor x prefix-list NAME in|out` |
| **distribute-list** | prefix, via an ACL  | `neighbor x distribute-list ACL in|out`    |
| **filter-list**  | the **AS-path** (regex) | `ip as-path access-list` + `neighbor x filter-list N in|out` |
| **route-map**    | **anything** (prefix, AS-path, community…) **and can set** attributes | `neighbor x route-map NAME in|out` |

Guidance: use a **prefix-list** to filter *which networks*, a **filter-list** to
filter *by origin/transit AS*, and a **route-map** when you need to both **match and
set** (e.g. accept a prefix *and* raise its local-pref). Prefix-lists are preferred
over distribute-lists/ACLs for prefix filtering (clearer, faster, support length
ranges with `ge`/`le`).

Applying new inbound policy needs the neighbor's routes re-evaluated — do it *without*
a hard reset via **route-refresh** (`clear ip bgp x soft in`); see [session
knobs](#authentication-timers-and-session-knobs).

[↩ back to README Lab 5](README.md#lab-5--policy-filtering-and-path-manipulation)

---

## Path manipulation

Filtering decides *whether* a route is used; manipulation decides *which* route wins
and *which way traffic flows*. The tool depends on the **direction of traffic** you
want to influence:

| Want to influence…            | Set…                     | Where / direction                    |
|-------------------------------|--------------------------|--------------------------------------|
| **Outbound** (how *you* exit) | **Weight** (one router)  | inbound route-map / per-neighbor     |
| **Outbound**, AS-wide         | **Local-Preference** ↑   | inbound route-map on the border      |
| **Inbound** (how others reach you) | **AS-path prepend** | outbound route-map toward the peer   |
| **Inbound**, same-AS multi-link | **MED** ↓             | outbound toward that neighbor AS     |

The asymmetry to remember: you fully control **outbound** (local-pref/weight are
yours to set), but you can only *influence* **inbound** by making your advertisement
**less attractive to the neighbor** — you can't set their local-pref. **AS-path
prepending** (advertising your AS 2–3 extra times) is the blunt, universal inbound
tool; **MED** is the polite one but only works when a single neighbor AS has multiple
links to you. [Communities](#bgp-communities) are the scalable way to *signal* a
provider to apply policy on your behalf.

[↩ back to README Lab 4.2](README.md#42-prefer-r1s-own-link-with-weight-local-to-one-router)

---

## BGP communities

A **community** is a 32-bit tag attached to a route (optional transitive attribute).
It carries no meaning by itself — it's a **label you and/or your provider agree to
act on**, which makes policy scale: tag routes at the edge, match the tag everywhere
else, instead of maintaining prefix lists.

- Written `ASN:value` (e.g. `65001:100`) by convention.
- **Well-known** communities trigger built-in behaviour:
  - **no-export** — don't advertise beyond the AS (to eBGP peers)
  - **no-advertise** — don't advertise to *any* peer
  - **local-as** (no-export-subconfed) — keep within the sub-AS
  - **internet** — advertise normally
- Set/match them in **route-maps** (`set community`, `match community`), and you must
  `neighbor x send-community` to actually transmit them (off by default).

Communities aren't a separate ENARSI lab item but underpin real-world "path
manipulation" — a provider publishes communities like "prepend once to my upstreams"
that customers set to steer traffic without the provider touching config.

[↩ back to README Lab 5](README.md#lab-5--policy-filtering-and-path-manipulation)

---

## MP-BGP and address families

Base BGP (RFC 4271) only carried IPv4 unicast. **Multiprotocol BGP** (RFC 4760) lets
one BGP session carry **many kinds of reachability**, identified by an
**AFI/SAFI** (Address Family Identifier / Subsequent AFI) — IPv6 unicast, VPNv4/VPNv6
(MPLS L3VPN), IPv4 multicast, L2VPN, and more.

Mechanically:
- Non-IPv4 NLRI and next-hops travel in two optional attributes — **MP_REACH_NLRI**
  (advertise) and **MP_UNREACH_NLRI** (withdraw) — so the base UPDATE format is
  unchanged.
- The capability is negotiated in the **[OPEN](#the-open-message-and-capabilities)**;
  each address family must then be **`activate`d per neighbor** — enabling BGP does
  **not** automatically carry every AF to every peer.
- Config splits into a global `router bgp` part (neighbor definitions, transport) and
  per-`address-family` parts (which peers carry that AF, its networks, its policy).

In this lab (Lab 6) you add **IPv6 unicast**. A key idea: the BGP *session* and the
*address family* are separate — one TCP session can carry IPv4 **and** IPv6 NLRI, or
you can run parallel sessions over v4 and v6 transport (this lab does the latter for
clarity).

[↩ back to README Lab 6](README.md#lab-6--mp-bgp-carrying-an-ipv6-address-family)

---

## Authentication, timers, and session knobs

The grab-bag of neighbor options ENARSI 1.11.b lists — all set per-neighbor (or
per-peer-group):

- **Authentication** — `neighbor x password` enables **TCP MD5** (a signature on
  every TCP segment, negotiated below BGP). A mismatch keeps the session from
  Establishing; on capture the TCP segments carry the **MD5 option**. (Modern
  alternative: TCP-AO.)
- **Timers** — `neighbor x timers <keepalive> <hold>`; the **lower hold-time** of the
  two peers is negotiated in the OPEN. Hold **0** = infinite (no keepalives).
- **Peer groups / templates** — group neighbors that share config (`neighbor GROUP
  peer-group`); also lets the router generate one UPDATE per group instead of
  per-neighbor (efficiency).
- **ebgp-multihop** — eBGP defaults to **TTL 1** (directly connected). Peering eBGP on
  **loopbacks** needs `ebgp-multihop <n>` (raise TTL) + `update-source` + a route to
  the peer's loopback. (`ttl-security hops n` is the hardened variant.)
- **4-byte ASNs** (RFC 6793) — 32-bit AS numbers (asplain like `4200000000`).
  Negotiated as a capability; a 2-byte-only peer sees `AS_TRANS` (**23456**) as a
  placeholder.
- **remove-private-as** — strips private ASNs (64512–65534, and the 4-byte private
  range) from the AS-path on eBGP advertisement toward a real upstream.
- **route-refresh vs soft-reconfiguration** — to re-apply *inbound* policy without
  bouncing the session: **route-refresh** (`clear ip bgp x soft in`) asks the peer to
  re-send (no memory cost, needs the capability); **`soft-reconfiguration inbound`**
  keeps a local unfiltered copy (works with old peers, costs memory). Outbound policy
  just needs `soft out`.
- **synchronization** — a legacy rule ("don't advertise an iBGP route until the IGP
  also has it," to protect non-BGP transit routers). **Disabled by default** on modern
  IOS and effectively obsolete now that cores are fully BGP-aware — know the term and
  that it's off.

[↩ back to README Lab 7](README.md#lab-7--neighbor-options-grab-bag)
