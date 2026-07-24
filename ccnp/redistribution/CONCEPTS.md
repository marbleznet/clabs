# Redistribution & Route Control — In-Depth Concepts

The companion reference to the [README](README.md) lab guide. The README is the
*hands-on* path (configure this, break that, fix it); this file is the *why* behind
each step. Each 📖 **Deep dive** link in the README jumps to a section here. You can
also read it straight through as a route-control primer.

**Contents**
- [Why redistribution exists and why it's risky](#why-redistribution-exists-and-why-its-risky)
- [How redistribution works: seed metrics](#how-redistribution-works-seed-metrics)
- [Administrative distance](#administrative-distance)
- [The two-way redistribution problem](#the-two-way-redistribution-problem)
- [Route tags](#route-tags)
- [Filtering: route-maps, prefix-lists, distribute-lists](#filtering-route-maps-prefix-lists-distribute-lists)
- [Loop-prevention mechanisms](#loop-prevention-mechanisms)
- [Summarization](#summarization)
- [External route types across protocols](#external-route-types-across-protocols)

---

## Why redistribution exists and why it's risky

A network often runs **more than one routing information source** — two protocols
after a merger, an IGP plus static routes to a firewall, EIGRP in the campus and OSPF
in the WAN. Each source is its own little world with its own metric and its own
database, and by default **they don't share routes**. **Redistribution** is the act
of taking routes *out* of one source and *injecting* them into another at a
**boundary router** (an ASBR, in OSPF terms).

It's also the single most **dangerous** thing you routinely do to a routing system,
for three reasons that this whole lab exists to expose:

1. **Metrics don't translate.** A protocol has no idea what another's metric means, so
   redistributed routes start with an arbitrary **[seed metric](#how-redistribution-works-seed-metrics)** —
   real path cost information is lost at the boundary.
2. **The boundary hides origin.** Once a route is redistributed it looks "local" to
   the new domain; nothing intrinsically remembers it came from elsewhere, so it can
   be **redistributed right back** ([the two-way problem](#the-two-way-redistribution-problem)).
3. **Two sources compete.** The same prefix now exists in two protocols, and which
   one a router believes is decided by **[administrative distance](#administrative-distance)**,
   not by which is actually correct.

The skill ENARSI tests is doing it **safely**: control the metric, control which
routes cross, and stop them from looping back — with seed metrics, filters, tags, AD,
and summarization.

[↩ back to README Lab 0](README.md#lab-0--build-the-two-domains-no-redistribution-yet)

---

## How redistribution works: seed metrics

When you `redistribute` a source into a protocol, each imported route needs a
**starting ("seed") metric** in the target protocol's units — because the source's
metric is meaningless to it. How that's supplied differs by protocol, and the
defaults are a classic source of "why didn't my routes show up?":

| Redistribute **into** | Seed metric behaviour                                                    |
|-----------------------|--------------------------------------------------------------------------|
| **OSPF**              | default **20** (BGP→OSPF: **1**), type **E2**; **needs `subnets`** or only classful nets import |
| **EIGRP** (classic)   | **no default** — you **must** give `metric bw delay rel load mtu` or `default-metric`, or routes are **silently dropped** |
| **RIP**               | needs a hop metric (`default-metric` or `metric`), else dropped          |
| **BGP**               | carries the IGP metric into MED by default                               |

Two keywords everyone forgets, both in README Lab 1:

- **OSPF `subnets`** — without it, OSPF only redistributes **classful** networks and
  discards every subnet (so a `/24` of `172.16.0.0` vanishes). Always include it.
- **EIGRP seed metric** — classic EIGRP won't advertise a redistributed route with no
  metric. Supply it inline or set one `default-metric` for the process.

You can redistribute a **dynamic protocol** (`redistribute ospf 1`), **static**
(`redistribute static`), or **connected** (`redistribute connected`) — "any routing
source," in the exam's words. `redistribute connected` is often needed so the
**boundary link subnets** themselves are reachable from the far domain.

[↩ back to README Lab 1](README.md#lab-1--redistribution-at-one-boundary-the-easy-case)

---

## Administrative distance

**Administrative distance (AD)** is how a router chooses **between routing sources**
when the same prefix is learned from more than one. It's a "trustworthiness" rank,
**lower = more trusted**, and it's compared **before** any metric — a lower-AD source
wins even if its path is objectively worse.

| Source            | AD  | | Source              | AD  |
|-------------------|-----|-|---------------------|-----|
| Connected         | 0   | | OSPF (intra/inter/ext) | 110 |
| Static            | 1   | | IS-IS               | 115 |
| eBGP              | 20  | | RIP                 | 120 |
| **EIGRP internal**| **90** | | **EIGRP external** | **170** |
|                   |     | | iBGP                | 200 |

The redistribution connection: after you redistribute prefix *X* from protocol A into
protocol B, a boundary router may see *X* **both** natively (from A) and as an
**external** (from B, redistributed by the *other* boundary). It installs whichever
has the **lower AD** — and if that's the *external* copy, it forwards **back toward
the domain the route came from**. That's an AD-induced loop.

The danger rule: **trouble whenever `AD(external copy) < AD(native protocol)`.**
- **OSPF ↔ EIGRP** is mostly safe by luck: EIGRP-external is **170**, above OSPF's
  110, so native OSPF wins. Native EIGRP is **90**, below OSPF-external 110, so native
  EIGRP wins. Both native routes win — *until you change an AD* (README Lab 3).
- **OSPF ↔ RIP** breaks readily: OSPF-external is 110, **below** RIP's 120, so the
  redistributed OSPF copy can beat native RIP.
- **Two OSPF processes** break: external and native are **both 110** — a coin flip.

You control AD with the **`distance`** command (per protocol, and scoped to specific
prefixes via an ACL). But AD only changes *which copy you install* — it doesn't stop
the route from looping. For that you want [tags](#route-tags).

[↩ back to README Lab 3](README.md#lab-3--administrative-distance-make-it-break-then-fix-it)

---

## The two-way redistribution problem

With **one** redistribution point, life is simple: routes flow A→B and B→A through a
single door, and there's no way back. Add a **second** boundary and you've built a
loop: a route can leave domain A into domain B at boundary 1, travel through B, and be
redistributed **back into A** at boundary 2 — now A has a "B-learned" copy of its own
route.

The three symptoms, in increasing severity:

1. **Suboptimal routing** — a router prefers a redistributed copy that points the long
   way around (very common with OSPF **E2** externals, whose metric is flat everywhere
   so distance to the ASBR is ignored).
2. **Route feedback** — a route re-enters its home domain as an external, competing
   with the native original.
3. **Routing loop / black hole** — when [AD](#administrative-distance) lets the fed-back
   copy win, traffic is forwarded toward the wrong domain and loops until TTL expires.

You cannot avoid two boundaries in real designs (you need the redundancy), so you make
two-way redistribution **safe** with, in order of robustness:

- **[Tags](#route-tags)** — stamp routes at each boundary and refuse to re-redistribute
  a route carrying the far domain's tag. *The route physically can't loop.* **Preferred.**
- **[Filtering](#filtering-route-maps-prefix-lists-distribute-lists)** — explicitly permit
  only the prefixes that should cross each way.
- **AD manipulation** — make sure native always out-ranks external; fragile, prefix-by-prefix.

[↩ back to README Lab 2](README.md#lab-2--the-second-boundary-feedback--suboptimal-routing)

---

## Route tags

A **route tag** is a **32-bit number attached to a route** that has *no effect on path
selection* — it's a pure administrative marker you set and match. It rides in the
places you can capture: the OSPF **Type-5 External Route Tag** field, the EIGRP
**external route TLV** tag, and (via redistribution) between protocols.

For two-way redistribution, tags implement airtight loop prevention:

1. At each boundary, **`set tag`** on everything redistributed **into** a domain — one
   tag per direction (e.g. OSPF→EIGRP = 65001, EIGRP→OSPF = 65002).
2. At each boundary, **`match tag` + `deny`** any route already carrying the *other*
   domain's tag, so it is **never redistributed back**.

```
route-map OSPF-TO-EIGRP deny 10
 match tag 65002        ← it came FROM EIGRP; don't send it back
route-map OSPF-TO-EIGRP permit 20
 set tag 65001          ← stamp the genuinely-OSPF routes
```

Why tags are the **robust** fix (better than AD): AD only decides *which copy a router
installs* — a route can still be advertised in a loop and just lose the AD contest,
which is fragile the moment a metric or AD changes. A tag **stops the redistribution
itself** at the boundary, so the route **physically never re-enters** its origin
domain, regardless of any metric or AD. Tags are protocol-independent (OSPF, EIGRP,
RIP, BGP all carry them) and scale to any number of boundaries.

[↩ back to README Lab 4](README.md#lab-4--route-tags-the-robust-loop-prevention-fix)

---

## Filtering: route-maps, prefix-lists, distribute-lists

Redistribution is **all-or-nothing** by default — every route from the source pours
into the target. Filtering restricts *which* prefixes cross. The tools, by power:

| Tool             | Matches on                              | Can set? | Typical use                          |
|------------------|-----------------------------------------|----------|--------------------------------------|
| **prefix-list**  | prefix + length range (`ge`/`le`)       | no       | clean prefix filtering               |
| **distribute-list** | prefix (via ACL or prefix-list), `in`/`out` of a process | no | filter what a protocol accepts/advertises |
| **route-map**    | prefix, **tag**, metric, next-hop, AS-path… | **yes** (tag, metric, AD, metric-type) | redistribution policy, tagging |

Guidance:

- **`redistribute … route-map NAME`** is the main hook — a route-map is the only tool
  that can both **match** (which routes) and **set** (tag, metric, metric-type) as
  routes cross the boundary, so it's where tag-based loop prevention and selective
  leaking live (README Labs 4–5).
- **`distribute-list … in|out`** filters routes entering/leaving a routing process
  independent of redistribution — e.g. stop a router from *installing* a prefix at all.
- Use a **prefix-list** over an ACL for prefix matching: it's faster, clearer, and
  supports length ranges (`192.168.16.0/22 le 32` = "the /22 and anything more
  specific inside it").

A route-map is a top-down `permit`/`deny` sequence: the first clause whose `match`
succeeds decides, and a route matching **no** clause is **denied** (implicit deny) —
so remember a trailing `permit` clause for "everything else."

[↩ back to README Lab 5](README.md#lab-5--filtering-which-routes-redistribute)

---

## Loop-prevention mechanisms

ENARSI 1.3 names **four** loop-prevention mechanisms. Two are the redistribution tools
above; two come from distance-vector protocol design:

- **Filtering** — prefix/distribute-lists and route-maps that stop specific routes from
  being advertised or redistributed ([above](#filtering-route-maps-prefix-lists-distribute-lists)).
- **Tagging** — mark routes at a boundary and refuse the returning tag ([above](#route-tags));
  the robust two-way-redistribution fix.
- **Split horizon** — a distance-vector rule: *never advertise a route back out the
  interface you learned it on* (that neighbor is closer to the destination, so telling
  it about the route can only create a loop). On by default in EIGRP/RIP **per
  interface**. It famously **breaks hub-and-spoke multipoint** (spokes can't learn each
  other through the hub), where you disable it: `no ip split-horizon eigrp <AS>`.
  Link-state protocols (OSPF/IS-IS) don't use it — they flood LSAs and prevent loops
  with the SPF tree and the area rules.
- **Route poisoning** (+ **poison reverse**) — when a route fails, a distance-vector
  protocol doesn't stay silent or wait for it to age out; it **advertises the route
  with an unreachable/infinite metric** so neighbors purge it immediately. **Poison
  reverse** is the twist on a shared segment: send the poisoned route *back* toward the
  source (deliberately overriding split horizon) to guarantee everyone drops it. EIGRP
  poisons with an all-ones (0xFFFFFFFF) delay; RIP uses metric 16.

Redistribution safety leans on the first two; the last two are how the protocols
themselves stay loop-free and are fair game to troubleshoot.

[↩ back to README Lab 7](README.md#lab-7--loop-prevention-mechanisms-all-four-enarsi-13)

---

## Summarization

**Summarization** advertises one aggregate prefix in place of many specifics. It has
three payoffs, all relevant here: **smaller routing tables**, **hidden instability** (a
flapping /24 inside a summary doesn't ripple out), and — for EIGRP —
**[bounded queries](#loop-prevention-mechanisms)**. At a redistribution boundary it also
cuts the number of externals you inject.

Manual summarization, by protocol and role — and the pair people mix up:

| Command                              | Where             | Summarizes                    |
|--------------------------------------|-------------------|-------------------------------|
| `area <a> range <net> <mask>`        | OSPF **ABR**      | **inter-area** (Type-3) routes |
| `summary-address <net> <mask>`       | OSPF **ASBR**     | **external (redistributed)** routes |
| `ip summary-address eigrp <as> …`    | EIGRP **interface** | anything, per interface (+ **Null0** discard) |

So to summarize the routes you *redistribute into* OSPF, you use **`summary-address`**
on the ASBR (README Lab 6.1) — `area range` wouldn't touch externals. EIGRP's
interface summary auto-installs a **`Null0` discard route** so the summarizing router
can't loop packets for a component it lacks.

**Auto-summary** (`auto-summary`) is the automatic version: summarize to the
**classful** boundary when crossing a major-network border. It's **off by default** on
modern IOS because it **black-holes discontiguous networks** — when pieces of one
classful network (e.g. subnets of `172.16.0.0/16`) live in different parts of the
topology, multiple routers advertise the same classful `/16` and traffic hashes to the
wrong one (README Lab 6.3). Reflexively `no auto-summary`, and summarize *deliberately*
where it makes sense.

[↩ back to README Lab 6](README.md#lab-6--summarization-manual--the-auto-summary-trap)

---

## External route types across protocols

Redistribution turns a route into an **external** route in the target protocol, and
each protocol marks and metrics externals differently — knowing the codes is half of
reading a redistribution table:

- **OSPF — `O E2` (default) vs `O E1`.** **E2** carries only the **external seed
  metric**, identical everywhere in the domain (internal cost to the ASBR is ignored) —
  simple, but it can't tell a near ASBR from a far one, so it's a common **suboptimal
  routing** source with two boundaries. **E1** = external metric **+** internal cost to
  the ASBR, so each router picks the *nearest* ASBR. Switch with `redistribute …
  metric-type 1`. (NSSA externals are the parallel **`N1`/`N2`**.)
- **EIGRP — `D EX`, AD 170.** Redistributed routes are *external* with AD **170** (vs
  **90** internal), so a native EIGRP route always beats a redistributed one — the
  property that keeps OSPF↔EIGRP loops in check. The external TLV also carries the
  **originating router-id**, **source protocol**, and **tag**.
- **BGP.** Redistributed routes get origin code **`?`** (incomplete) and carry the IGP
  metric into **MED**.

Why it matters for this lab: the **metric type** you pick when redistributing decides
whether two ASBRs give you sane closest-exit routing (E1) or flat, potentially
suboptimal routing (E2, the default) — and the **AD** of the external type is what
determines whether a fed-back copy can hijack a native route.

[↩ back to README captures](README.md#using-edgeshark-for-captures)
