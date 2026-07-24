# CCNP Redistribution & Route Control Lab (Cisco IOL + Edgeshark)

A 5-router `cisco_iol` lab that wraps up **ENARSI Section 1** — the cross-protocol
"route control" topics that don't belong to any single routing protocol:
**administrative distance (1.1)**, **route-maps/attributes/tagging/filtering (1.2)**,
**loop-prevention mechanisms (1.3)**, **redistribution between protocols and sources
(1.4)**, and **manual/auto summarization (1.5)**.

The routers boot with **only interfaces and IP addresses** — no OSPF, no EIGRP, no
redistribution. You build **two routing domains** (OSPF and EIGRP), join them at
**two boundary routers**, and then wrestle the problems that mutual redistribution
creates: **feedback loops, suboptimal routing, administrative-distance inversions**,
and the tools that fix them — **tags, filters, AD manipulation, and summarization**.

> This lab is **IPv4-only** on purpose. Redistribution/route-control is an
> IPv4-heavy troubleshooting topic and the lab is already large; every technique
> here (tags, AD, route-maps, summarization) applies identically to IPv6
> (OSPFv3 ↔ EIGRPv6) redistribution.

> 📖 **This lab has a companion:** [`CONCEPTS.md`](CONCEPTS.md) is the in-depth
> reference behind every step (why redistribution is dangerous, the AD table, the
> two-way problem, tags, filtering, summarization). Each lab opens with a **Deep
> dive** callout — follow it for the *why*, skip it when you just want to type.

---

## Topology

```
        OSPF Area 0                                        EIGRP AS 100
 ┌──────────────────────────┐                    ┌──────────────────────────┐
 │  R5 ───── R1 ────┬─── e0/1  R2  e0/2 ───┬──── R4                          │
 │ static   192.168 │       (boundary #1)  │   172.16                        │
 │ source     nets  └─── e0/1  R3  e0/2 ───┘   nets                          │
 │                          (boundary #2)                                    │
 └──────────────────────────┘                    └──────────────────────────┘
       redistribute STATIC          R2 & R3 mutually redistribute
       into OSPF (R5)               OSPF <-> EIGRP  (the loop engine)
```

| Router | Domain(s)          | Role                                                        |
|--------|--------------------|-------------------------------------------------------------|
| R1     | OSPF               | OSPF core — 192.168.1.0/24 + a /22 block (192.168.16–19)     |
| R5     | OSPF               | OSPF edge — redistributes **static** routes (a non-dynamic source) |
| **R2** | **OSPF + EIGRP**   | **Boundary #1** — mutual redistribution                     |
| **R3** | **OSPF + EIGRP**   | **Boundary #2** — mutual redistribution (creates the loop risk) |
| R4     | EIGRP              | EIGRP core — 172.16.4.0/24 + a /22 block (172.16.20–23)      |

**Why two boundary routers?** With a **single** redistribution point, translating
between domains is easy and safe. The instant you have **two**, a route can leave
its home domain at one boundary and **leak back in** at the other — the root of
redistribution loops and suboptimal routing. This topology is built to make that
happen so you can see and fix it.

### Addressing

| Link            | Subnet         | | Loopback blocks |                              |
|-----------------|----------------|-|-----------------|------------------------------|
| R1–R2 (OSPF)    | 10.12.0.0/30   | | R1              | 192.168.1.0/24, 192.168.16–19.0/24 |
| R1–R3 (OSPF)    | 10.13.0.0/30   | | R4              | 172.16.4.0/24, 172.16.20–23.0/24   |
| R1–R5 (OSPF)    | 10.15.0.0/30   | | R5              | 192.168.5.0/24 (+ static 172.20.100–101.0/24) |
| R2–R4 (EIGRP)   | 10.24.0.0/30   | | RIDs            | R1 1.1.1.1 … R5 5.5.5.5      |
| R3–R4 (EIGRP)   | 10.34.0.0/30   | |                 |                              |

Interface mapping: containerlab `ethX` → IOL `Ethernet0/X`; `E0/0` is management.

---

## Deploy

```bash
cd /home/lab/clabs/ccnp/redistribution
sudo containerlab deploy -t redistribution.clab.yml
```

Log in with `admin` / `admin`. Destroy with:

```bash
sudo containerlab destroy -t redistribution.clab.yml --cleanup
```

---

## Using Edgeshark for captures

Redistribution is mostly a **control-plane / routing-table** topic, but two things
show beautifully on the wire — the **external LSAs/TLVs** the boundary routers
originate, and the **route tags** you'll attach to them:

| Filter                        | Shows                                                     |
|-------------------------------|-----------------------------------------------------------|
| `ospf.lsa.type == 5`          | **Type-5 AS-External LSA** — EIGRP/static nets injected into OSPF |
| `ospf.advrouter == 2.2.2.2`   | LSAs originated by boundary R2                             |
| `eigrp.tlv.type == 0x0103`    | **EIGRP External route TLV** — OSPF nets injected into EIGRP |
| `eigrp`                       | all EIGRP (watch for the external TLV's origin + **tag** fields) |

> The Type-5 LSA carries an **External Route Tag** field, and the EIGRP external
> TLV carries an **originating router-id** and a **tag** — the exact fields that
> make [tag-based loop prevention](CONCEPTS.md#route-tags) work. You'll watch a tag
> you set in a route-map appear in these packets in Lab 4.

> 📖 **Deep dive:** [External route types across protocols](CONCEPTS.md#external-route-types-across-protocols)

---

# Lab 0 — Build the two domains (no redistribution yet)

**Goal:** stand up OSPF and EIGRP as **isolated** domains, and confirm they can't
see each other's routes. That isolation is the "before" picture.

> 📖 **Deep dive:** [Why redistribution exists and why it's risky](CONCEPTS.md#why-redistribution-exists-and-why-its-risky)

### 0.1 OSPF domain (R1, R5, and the OSPF side of R2/R3)

```
r1(config)# router ospf 1
r1(config-router)#  router-id 1.1.1.1
r1(config-router)#  network 10.12.0.0 0.0.0.3 area 0
r1(config-router)#  network 10.13.0.0 0.0.0.3 area 0
r1(config-router)#  network 10.15.0.0 0.0.0.3 area 0
r1(config-router)#  network 192.168.1.0 0.0.0.255 area 0
r1(config-router)#  network 192.168.16.0 0.0.3.255 area 0    ! the /22 block (.16-.19)
```
```
r5(config)# router ospf 1
r5(config-router)#  router-id 5.5.5.5
r5(config-router)#  network 10.15.0.0 0.0.0.3 area 0
r5(config-router)#  network 192.168.5.0 0.0.0.255 area 0
```
```
r2(config)# router ospf 1
r2(config-router)#  router-id 2.2.2.2
r2(config-router)#  network 10.12.0.0 0.0.0.3 area 0
!
r3(config)# router ospf 1
r3(config-router)#  router-id 3.3.3.3
r3(config-router)#  network 10.13.0.0 0.0.0.3 area 0
```

### 0.2 EIGRP domain (R4, and the EIGRP side of R2/R3)

```
r4(config)# router eigrp 100
r4(config-router)#  eigrp router-id 4.4.4.4
r4(config-router)#  network 10.24.0.0 0.0.0.3
r4(config-router)#  network 10.34.0.0 0.0.0.3
r4(config-router)#  network 172.16.0.0 0.0.255.255
r4(config-router)#  no auto-summary
!
r2(config)# router eigrp 100
r2(config-router)#  eigrp router-id 2.2.2.2
r2(config-router)#  network 10.24.0.0 0.0.0.3
r2(config-router)#  no auto-summary
!
r3(config)# router eigrp 100
r3(config-router)#  eigrp router-id 3.3.3.3
r3(config-router)#  network 10.34.0.0 0.0.0.3
r3(config-router)#  no auto-summary
```

### 0.3 Confirm the domains are isolated

```
r1# show ip route ospf         ! sees 192.168.x -- but NO 172.16.x (EIGRP is invisible)
r4# show ip route eigrp        ! sees 172.16.x -- but NO 192.168.x (OSPF is invisible)
r2# show ip route              ! R2 sees BOTH (it runs both protocols) -- the bridge point
```

R2 and R3 each hold *both* routing tables but don't yet pass routes between them.
That handoff is redistribution.

---

# Lab 1 — Redistribution at ONE boundary (the easy case)

**Goal:** mutually redistribute OSPF ↔ EIGRP at **R2 only**, plus bring in R5's
**static** routes — and meet the two keywords everyone forgets: OSPF's `subnets`
and EIGRP's **seed metric**.

> 📖 **Deep dive:** [How redistribution works: seed metrics](CONCEPTS.md#how-redistribution-works-seed-metrics)
> · [External route types across protocols](CONCEPTS.md#external-route-types-across-protocols)

### 1.1 Redistribute EIGRP → OSPF on R2

> **📸 CAPTURE — `r2` interface `Ethernet0/1`** (OSPF side). Filter
> `ospf.lsa.type == 5`.

```
r2(config)# router ospf 1
r2(config-router)#  redistribute eigrp 100 subnets
```

> **The `subnets` gotcha:** without `subnets`, OSPF only injects **classful**
> networks and silently drops every subnet. Try it *without* `subnets` first —
> `show ip route ospf` on R1 shows nothing useful — then add it and watch the
> 172.16.x routes appear as **`O E2`**.

```
r1# show ip route ospf         ! 172.16.4.0/24, 172.16.20-23.0/24 now O E2 (external, via R2)
r1# show ip ospf database external    ! Type-5 LSAs, advertising router 2.2.2.2
```
On the capture you'll see the **Type-5 AS-External LSAs** R2 originates, one per
redistributed EIGRP prefix.

### 1.2 Redistribute OSPF → EIGRP on R2 (seed metric required)

> **📸 CAPTURE — `r2` interface `Ethernet0/2`** (EIGRP side). Filter
> `eigrp.tlv.type == 0x0103`.

```
r2(config)# router eigrp 100
r2(config-router)#  redistribute ospf 1 metric 100000 100 255 1 1500
```

> **The seed-metric requirement:** EIGRP has no metric for a route from another
> protocol, so classic mode **won't advertise it without one**. Supply it inline
> (`metric bw delay reliability load mtu`) or set a `default-metric` under EIGRP.
> Try omitting it first and confirm R4 learns nothing.

```
r4# show ip route eigrp        ! 192.168.x now D EX (external), AD 170, via R2
r4# show ip eigrp topology 192.168.1.0/24    ! external; note "AD 170"
```
On the capture, expand an **External route TLV** — it carries the **originating
router-id** and the **external protocol** (OSPF), fields the internal TLV lacks.

### 1.3 Redistribute a static source (R5)

Static routes are "any routing source" (1.4). Give R5 some and inject them:

```
r5(config)# ip route 172.20.100.0 255.255.255.0 Null0
r5(config)# ip route 172.20.101.0 255.255.255.0 Null0
r5(config)# router ospf 1
r5(config-router)#  redistribute static subnets
```
```
r4# show ip route 172.20.100.0    ! static -> OSPF (R5) -> EIGRP (R2): D EX on R4
```
The 172.20.x routes crossed **two** redistribution boundaries (static→OSPF at R5,
OSPF→EIGRP at R2) to reach R4. Everything works — because there's only one path in.
That changes in Lab 2.

---

# Lab 2 — The second boundary: feedback & suboptimal routing

**Goal:** enable the **same** mutual redistribution on **R3**, and watch a
two-point redistribution domain start to misbehave — routes leaking back, and
external copies competing with native ones.

> 📖 **Deep dive:** [The two-way redistribution problem](CONCEPTS.md#the-two-way-redistribution-problem)
> · [Administrative distance](CONCEPTS.md#administrative-distance)

### 2.1 Turn R3 into the second redistribution point

```
r3(config)# router ospf 1
r3(config-router)#  redistribute eigrp 100 subnets
r3(config)# router eigrp 100
r3(config-router)#  redistribute ospf 1 metric 100000 100 255 1 1500
```

### 2.2 See routes leak back into their home domain

Every EIGRP prefix is now injected into OSPF at **both** R2 and R3, and every OSPF
prefix into EIGRP at both. Look at a native EIGRP prefix from a **boundary** router's
point of view:

```
r3# show ip route 172.16.4.0    ! native via EIGRP? or the OSPF-external copy R2 injected?
r3# show ip eigrp topology 172.16.4.0/24
r3# show ip ospf database external 172.16.4.0
```

R3 now hears `172.16.4.0/24` **two** ways: **natively via EIGRP** (AD 90) *and* as an
**OSPF external** (AD 110) — because R2 redistributed it into OSPF and R1 flooded it
back to R3. The router picks by **administrative distance**: EIGRP-internal **90**
beats OSPF **110**, so the native path wins *this time*. That's luck, not design —
Lab 3 shows how easily the AD ordering flips and breaks it.

### 2.3 Spot the suboptimal / redundant externals

```
r4# show ip route 192.168.1.0    ! learned via R2 AND R3 (two D EX paths) -- is the chosen one optimal?
r1# show ip route 172.16.20.0    ! O E2 from two ASBRs; E2 metric ignores internal cost (may pick "wrong" ASBR)
```

> **Two symptoms to name:** **feedback** (a route re-entering the domain it came
> from — happening now, held in check only by AD) and **suboptimal routing**
> (default `O E2` externals carry the same metric everywhere, so a router can pick
> the farther ASBR). Neither is a hard loop *yet* — but you're one AD change away.
> The robust fixes are **[tags](CONCEPTS.md#route-tags)** (Lab 4) and
> **[filters](CONCEPTS.md#filtering-route-maps-prefix-lists-distribute-lists)** (Lab 5),
> applied regardless of whether AD happens to save you.

---

# Lab 3 — Administrative distance (make it break, then fix it)

**Goal:** AD is what decides *between protocols* for the same prefix (ENARSI 1.1).
Deliberately invert it to create a real redistribution loop, watch it break, then
control it with the `distance` command.

> 📖 **Deep dive:** [Administrative distance](CONCEPTS.md#administrative-distance)

### 3.1 Break it: make the EIGRP-external copy beat native OSPF

By default EIGRP-external is **170** (worse than OSPF's 110), which is what keeps
OSPF prefixes from being hijacked by their redistributed copies. Lower it below 110
and watch the boundary routers prefer the **wrong** (redistributed) copy:

```
r2(config)# router eigrp 100
r2(config-router)#  distance eigrp 90 100      ! internal 90, EXTERNAL now 100 (< OSPF 110!)
r3(config)# router eigrp 100
r3(config-router)#  distance eigrp 90 100
```

```
r2# show ip route 192.168.1.0
```

Now R2 prefers the **EIGRP-external** copy of `192.168.1.0` (AD 100) over its own
**native OSPF** route (AD 110) — even though 192.168.1.0 lives *in the OSPF domain
right next to it*. R2 forwards toward R4 (EIGRP) to reach an OSPF-local prefix →
**suboptimal at best, a routing loop at worst** as the two boundaries chase each
other. This is the canonical AD-inversion redistribution failure.

### 3.2 Fix it: restore sane AD ordering

```
r2(config-router)# distance eigrp 90 170       ! external back above OSPF's 110
r3(config-router)# distance eigrp 90 170
r2# show ip route 192.168.1.0                   ! native OSPF (110) wins again
```

### 3.3 Targeted AD (the precise tool)

`distance` can be scoped to specific routes/sources — e.g. raise the AD of a
particular external prefix so a specific native route always wins, without moving
every route:

```
r2(config)# access-list 30 permit 172.16.4.0 0.0.0.255
r2(config)# router ospf 1
r2(config-router)#  distance 175 0.0.0.0 255.255.255.255 30    ! this prefix from OSPF -> AD 175
```

> **The point of 1.1:** when the "wrong" protocol's copy is installed, it's almost
> always an AD story. `show ip route <prefix>` shows the AD in `[AD/metric]`;
> `distance` is how you override it. But AD is a blunt instrument — tags (next) are
> the *robust* loop fix.

---

# Lab 4 — Route tags: the robust loop-prevention fix

**Goal:** the protocol-independent way to stop a route from being redistributed
**back** into the domain it came from — **tag on the way in, deny the tag on the way
back** (ENARSI 1.2 tagging + 1.3 loop prevention).

> 📖 **Deep dive:** [Route tags](CONCEPTS.md#route-tags)
> · [Loop-prevention mechanisms](CONCEPTS.md#loop-prevention-mechanisms)

### 4.1 The idea

Every route redistributed **OSPF → EIGRP** gets stamped with tag **65001**. Every
route redistributed **EIGRP → OSPF** gets tag **65002**. Then each boundary
**refuses to redistribute a route that already carries the *other* domain's tag** —
so a route can never make a round trip.

### 4.2 Tag OSPF→EIGRP and reject the return tag

> **📸 CAPTURE — `r2` interface `Ethernet0/2`.** Filter `eigrp.tlv.type == 0x0103`.
> Expand a redistributed route and find the **tag = 65001** in the external TLV.

```
r2(config)# route-map OSPF-TO-EIGRP deny 10
r2(config-route-map)#  match tag 65002                 ! came FROM EIGRP already -> don't send back
r2(config)# route-map OSPF-TO-EIGRP permit 20
r2(config-route-map)#  set tag 65001                    ! stamp everything else
r2(config)# router eigrp 100
r2(config-router)#  redistribute ospf 1 metric 100000 100 255 1 1500 route-map OSPF-TO-EIGRP
```

### 4.3 Tag EIGRP→OSPF and reject the return tag

```
r2(config)# route-map EIGRP-TO-OSPF deny 10
r2(config-route-map)#  match tag 65001                 ! came FROM OSPF already -> don't send back
r2(config)# route-map EIGRP-TO-OSPF permit 20
r2(config-route-map)#  set tag 65002
r2(config)# router ospf 1
r2(config-router)#  redistribute eigrp 100 subnets route-map EIGRP-TO-OSPF
```

Apply the **same two route-maps on R3**. Now re-run the Lab 3 AD break (`distance
eigrp 90 100`) and confirm the loop **no longer forms** — the tag stops the route
before AD ever matters:

```
r1# show ip ospf database external 172.16.4.0    ! carries Tag 65002
r4# show ip eigrp topology 192.168.1.0/24        ! carries Tag 65001
r2# show ip route 192.168.1.0                     ! native OSPF wins; no tagged re-injection
```

> **Why tags beat AD:** AD only changes *which copy you install*; a bad AD still
> lets the route loop. A tag **stops the redistribution itself**, so the route
> physically never re-enters its home domain — robust no matter what the metrics or
> ADs do. This is the ENARSI-preferred fix for two-way redistribution.

---

# Lab 5 — Filtering which routes redistribute

**Goal:** redistribution is all-or-nothing by default; use **route-maps with
prefix-lists** (and `distribute-list`) to redistribute only what you intend (ENARSI
1.2 filtering).

> 📖 **Deep dive:** [Filtering: route-maps, prefix-lists, distribute-lists](CONCEPTS.md#filtering-route-maps-prefix-lists-distribute-lists)

### 5.1 Only leak specific prefixes with a route-map + prefix-list

Say policy is: **don't** leak R1's 192.168.16.0/22 block into EIGRP, but do leak
everything else. Extend the tag route-map with a prefix match:

```
r2(config)# ip prefix-list NO-LEAK seq 5 permit 192.168.16.0/22 le 32
r2(config)# route-map OSPF-TO-EIGRP deny 5
r2(config-route-map)#  match ip address prefix-list NO-LEAK      ! drop the whole /22 block
```
(the existing `deny 10 match tag` and `permit 20 set tag` clauses still apply after)

```
r4# show ip route 192.168.16.0    ! GONE from EIGRP; 192.168.1.0 still present
```

### 5.2 distribute-list — filter at the protocol edge

`distribute-list` filters routes going into/out of a routing process (in/out),
independent of redistribution:

```
r4(config)# ip prefix-list BLOCK-100 seq 5 deny 172.20.100.0/24
r4(config)# ip prefix-list BLOCK-100 seq 10 permit 0.0.0.0/0 le 32
r4(config)# router eigrp 100
r4(config-router)#  distribute-list prefix BLOCK-100 in
r4# show ip route 172.20.100.0    ! R4 no longer installs it
```

> Route-maps are the swiss-army tool (match on prefix, tag, metric, AS-path; set
> attributes). Use a `distribute-list`/`prefix-list` for simple prefix filtering,
> a `route-map` when you need to **match on more than the prefix** or **set**
> something.

---

# Lab 6 — Summarization (manual + the auto-summary trap)

**Goal:** shrink the redistributed route count at the boundary, and see why
`auto-summary` is dangerous with discontiguous networks (ENARSI 1.5).

> 📖 **Deep dive:** [Summarization](CONCEPTS.md#summarization)

### 6.1 Summarize EIGRP externals as they enter OSPF (ASBR summary)

R2/R3 inject four `172.16.20–23.0/24` externals into OSPF. Collapse them to one
`/22` at the ASBR with `summary-address`:

```
r2(config)# router ospf 1
r2(config-router)#  summary-address 172.16.20.0 255.255.252.0
r3(config)# router ospf 1
r3(config-router)#  summary-address 172.16.20.0 255.255.252.0
```
```
r1# show ip route ospf         ! one O E2 172.16.20.0/22 instead of four /24s
```

> `summary-address` summarizes **external (redistributed)** routes at an **ASBR**;
> `area range` summarizes **inter-area** routes at an **ABR** — don't mix them up.

### 6.2 Summarize OSPF nets toward EIGRP (interface summary)

```
r2(config)# interface Ethernet0/2
r2(config-if)#  ip summary-address eigrp 100 192.168.16.0 255.255.252.0
r3(config)# interface Ethernet0/2
r3(config-if)#  ip summary-address eigrp 100 192.168.16.0 255.255.252.0
```
```
r4# show ip route 192.168.16.0    ! one D 192.168.16.0/22 (with a Null0 discard on R2/R3)
```

### 6.3 The auto-summary trap

`172.16.0.0/16` (EIGRP) and `192.168.x` cross classful boundaries. Turn EIGRP
`auto-summary` on and watch discontiguous 172.16 pieces collide:

```
r4(config)# router eigrp 100
r4(config-router)#  auto-summary
r2# show ip route 172.16.0.0    ! classful /16 summaries now overlap -> black-holes
r4(config-router)# no auto-summary            ! put it back -- the modern default for a reason
```

---

# Lab 7 — Loop-prevention mechanisms, all four (ENARSI 1.3)

**Goal:** name and demonstrate the **four** loop-prevention mechanisms 1.3 lists —
you've now used two; here are the other two and a recap.

> 📖 **Deep dive:** [Loop-prevention mechanisms](CONCEPTS.md#loop-prevention-mechanisms)

| Mechanism         | Where you met it                                              |
|-------------------|--------------------------------------------------------------|
| **Filtering**     | Lab 5 — prefix-lists / distribute-lists / route-maps         |
| **Tagging**       | Lab 4 — tag on redistribution, deny the returning tag        |
| **Split horizon** | ↓ 7.1 — don't advertise a route back out the interface it came in |
| **Route poisoning** | ↓ 7.2 — advertise a dead route with an unreachable metric  |

### 7.1 Split horizon (distance-vector loop prevention)

EIGRP enables **split horizon** per interface by default — it won't re-advertise a
route back out the interface it learned it on:

```
r4# show ip eigrp interfaces detail Ethernet0/1 | include Split
```
On a **hub-and-spoke multipoint** interface this *causes* missing routes (spokes
can't hear each other), and the fix is to disable it: `no ip split-horizon eigrp
100`. (OSPF, being link-state, floods LSAs and doesn't use split horizon at all —
its loop prevention is the SPF tree + the areas rule.)

### 7.2 Route poisoning

When EIGRP loses a route it doesn't go silent — it **poisons** the route by
advertising it with an **unreachable metric** (delay 0xFFFFFFFF), so neighbors
remove it immediately instead of aging it out. Watch it on the wire:

> **📸 CAPTURE — `r4` interface `Ethernet0/1`.** Filter `eigrp`. Then on R4
> `shutdown` a loopback (e.g. `interface Loopback1 / shutdown`) and find the
> **Update** advertising that prefix with the unreachable metric — that's poisoning
> (paired with **poison reverse** on shared segments).

---

## Suggested end-state verification

```
r1# show ip route ospf         ! O E2 externals (172.16.x, 172.20.x), summarized where configured
r4# show ip route eigrp        ! D EX externals (192.168.x), AD 170, tagged
r2# show ip route 192.168.1.0  ! native OSPF wins (AD sane); no tagged re-injection
r1# show ip ospf database external 172.16.20.0    ! Tag 65002 present
r4# show ip eigrp topology 192.168.1.0/24         ! Tag 65001 present
any# show ip protocols         ! redistribution + route-maps + metrics per process
```

## Administrative distance cheat-sheet

| Source                | AD  | | Source            | AD  |
|-----------------------|-----|-|-------------------|-----|
| Connected             | 0   | | OSPF (all types)  | 110 |
| Static                | 1   | | IS-IS             | 115 |
| eBGP                  | 20  | | RIP               | 120 |
| **EIGRP internal**    | **90**  | | **EIGRP external** | **170** |
| OSPF **inter/intra/ext** are all **110** | | iBGP | 200 |

> The redistribution danger zone: any time a **redistributed (external)** route's AD
> is **lower** than the **native** protocol's AD for the same prefix, the boundary
> installs the wrong copy → loop. EIGRP-external 170 > OSPF 110 is what keeps
> OSPF↔EIGRP mostly safe; OSPF↔RIP (120 vs ext 110) and two-OSPF-process (110 vs
> 110) are the classic breakers.

---

## Reset the whole lab

```bash
sudo containerlab destroy -t redistribution.clab.yml --cleanup
sudo containerlab deploy  -t redistribution.clab.yml
```

Because the configs hold no routing, every redeploy is a clean slate.

---

## Where this fits

This lab **completes ENARSI Section 1** — it's the cross-protocol glue on top of the
`../ospf`, `../eigrp`, and `../bgp` labs (all of whose external-route and
summarization sections it builds on). Next in the series: the remaining
Infrastructure Security/Services labs (filtering & FHS, telemetry, DHCP), then the
**combined enterprise capstone**, where this exact OSPF↔EIGRP redistribution sits at
the core with a BGP edge bolted on.
