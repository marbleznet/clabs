# CCNP BGP Deep-Dive Lab (Cisco IOL + Edgeshark)

A 5-router + 1-switch `cisco_iol` containerlab topology for learning **BGP** — both
**eBGP and iBGP**, a **route reflector**, **best-path** selection, **policy**, and
**MP-BGP for IPv6** — from the packet up. The routers boot with **only interfaces
and IP addresses**: **no IGP, no BGP, no routing config at all**. You build the IGP
underlay, every BGP session, the reflector, and all the policy by hand, and use
**Edgeshark packet captures** at each step to *see* BGP on the wire.

By the end you will have configured and observed:

- The BGP **session state machine** (Idle → Connect → OpenSent → OpenConfirm →
  Established) over **TCP 179**, and the **OPEN/UPDATE/KEEPALIVE** messages
- **eBGP** between three autonomous systems, and **iBGP** inside one AS
- The **iBGP next-hop problem** and `next-hop-self`
- The **iBGP full-mesh rule** and its fix: a **route reflector** (originator-id,
  cluster-list)
- The **best-path algorithm** and the attributes that drive it — **weight,
  local-preference, AS-path, origin, MED**
- **Policy**: prefix-lists, AS-path filter-lists, route-maps; **path manipulation**
  with local-pref, MED, and **AS-path prepending**; `remove-private-as`
- **MP-BGP** — carrying an **IPv6** address family
- Neighbor options: **MD5 authentication**, timers, **peer groups**,
  **ebgp-multihop**, **4-byte ASNs**, **route-refresh**

> 📖 **This lab has a companion:** [`CONCEPTS.md`](CONCEPTS.md) is the in-depth
> reference behind every step (the state machine, attributes, the best-path
> algorithm, route reflection, MP-BGP, and more). Each lab opens with a **Deep
> dive** callout — follow it for the *why*, skip it when you just want to type. It
> also reads well end-to-end as a BGP primer.

---

## Topology

```
        AS 65002 (R4) ========= eBGP  10.45.0.0/30 ========= AS 65003 (R5)
     Lo1 172.16.4.0/24  \                                   /  Lo1 172.16.5.0/24
                         \                                 /
              eBGP 10.14.0.0/30                    eBGP 10.25.0.0/30
                          \                             /
   ── AS 65001 ────────────\───────────────────────────/────────────────── ──
   │                     R1 (1.1.1.1)            R2 (2.2.2.2)               │
   │  our AS             border/eBGP             border/eBGP               │
   │                        \                       /                      │
   │   core LAN 10.0.0.0/24   \                     /   iBGP over Lo0,      │
   │   via SW1 (L2)            └──── SW1 (L2) ──────┘   OSPF underlay       │
   │                                   │                                   │
   │                             R3 (3.3.3.3)                              │
   │                      Route Reflector + Lo1 172.16.3.0/24              │
   └───────────────────────────────────────────────────────────────────────
```

**Why this shape?** Every external prefix is reachable **two ways** from AS 65001:
`172.16.5.0/24` (AS 65003) comes in **directly via R2** (AS-path `65003`) *and*
**via R1 → R4** (AS-path `65002 65003`). That built-in choice is the engine for the
best-path and policy labs. R1 and R2 are the **border** routers (one eBGP session
each); **R3 is an internal route reflector**, so R1 and R2 never need a direct iBGP
session. The core LAN via SW1 gives AS 65001 a shared segment for the **OSPF
underlay** (which makes the iBGP loopbacks reachable) and the iBGP sessions.

### Roles & autonomous systems

| Router | AS      | Lo0 / RID | Role                                                        |
|--------|---------|-----------|-------------------------------------------------------------|
| R1     | 65001   | 1.1.1.1   | Border — **eBGP** to AS 65002 (R4); iBGP **RR client**      |
| R2     | 65001   | 2.2.2.2   | Border — **eBGP** to AS 65003 (R5); iBGP **RR client**      |
| R3     | 65001   | 3.3.3.3   | Internal — **route reflector**; originates 172.16.3.0/24    |
| R4     | 65002   | 4.4.4.4   | External peer; originates 172.16.4.0/24; eBGP to R1 & R5    |
| R5     | 65003   | 5.5.5.5   | External peer; originates 172.16.5.0/24; eBGP to R2 & R4    |
| SW1    | —       | —         | L2 switch — AS 65001 core LAN (OSPF underlay + iBGP)         |

AS 65001, 65002, 65003 are all in the **private 2-byte** range (64512–65534).

### Addressing

| Link / Interface        | IPv4             | IPv6              | Purpose                         |
|-------------------------|------------------|-------------------|---------------------------------|
| R1/R2/R3 ↔ SW1 (e0/1)   | 10.0.0.0/24      | 2001:db8:0::/64   | AS 65001 core LAN               |
| R1 e0/2 ↔ R4 e0/1       | 10.14.0.0/30     | 2001:db8:14::/64  | eBGP AS65001–AS65002            |
| R2 e0/2 ↔ R5 e0/1       | 10.25.0.0/30     | 2001:db8:25::/64  | eBGP AS65001–AS65003            |
| R4 e0/2 ↔ R5 e0/2       | 10.45.0.0/30     | 2001:db8:45::/64  | eBGP AS65002–AS65003            |
| R1/R2/R3 Lo0            | x.x.x.x/32       | —                 | RID + iBGP peering source       |
| R3 Lo1                  | 172.16.3.0/24    | 2001:db8:3::/64   | AS 65001 advertised net         |
| R4 Lo1                  | 172.16.4.0/24    | 2001:db8:4::/64   | AS 65002 advertised net         |
| R5 Lo1                  | 172.16.5.0/24    | 2001:db8:5::/64   | AS 65003 advertised net         |

On the /30 eBGP links the lower-numbered router is `.1`. The `e0/X` interfaces are
**dual-stacked** — IPv4 for Labs 0–5/7 and IPv6 for MP-BGP in Lab 6.

Interface mapping: containerlab `ethX` → IOL `Ethernet0/X`. `E0/0` is management.

---

## Deploy

```bash
cd /home/lab/clabs/ccnp/bgp
sudo containerlab deploy -t bgp-lab.clab.yml
```

Log in to any node with `admin` / `admin`:

```bash
ssh admin@clab-bgp-lab-r1        # or: docker exec -it clab-bgp-lab-r1 telnet localhost
```

Give the nodes ~60–90s after deploy. Destroy with:

```bash
sudo containerlab destroy -t bgp-lab.clab.yml --cleanup
```

### Sanity check before you start (no routing yet)

```
r1# show ip interface brief          ! e0/1, e0/2, Lo0 up/up
r1# show ip route                    ! only connected + local routes
r1# show ip bgp summary              ! "% BGP not active" -- correct, blank slate
```

---

## Using Edgeshark for captures

Open the **Ghostwire** UI, find the container (e.g. `clab-bgp-lab-r1`), expand its
interfaces, and click the **capture** (shark-fin) icon next to `Ethernet0/2`. A
browser Wireshark session opens and streams live packets.

BGP runs over **TCP port 179** (a plain TCP session — no multicast, no dedicated IP
protocol). Handy display filters:

| Filter                    | Shows                                              |
|---------------------------|----------------------------------------------------|
| `tcp.port == 179`         | the BGP transport session (incl. the TCP handshake)|
| `bgp`                     | all BGP messages                                   |
| `bgp.type == 1`           | **OPEN** (capabilities, AS, hold-time, RID)        |
| `bgp.type == 2`           | **UPDATE** (NLRI + path attributes, or withdrawals)|
| `bgp.type == 3`           | **NOTIFICATION** (error → session tears down)      |
| `bgp.type == 4`           | **KEEPALIVE**                                      |
| `bgp.type == 5`           | **ROUTE-REFRESH**                                  |

> Tip: BGP is bursty, not periodic — it floods UPDATEs at convergence, then goes
> quiet except for KEEPALIVEs. Start the capture *before* the config step that
> brings a session up or changes a policy, so you catch the OPEN and the UPDATEs.

---

# Lab 0 — IGP underlay (so iBGP has something to stand on)

**Goal:** iBGP peers over **loopbacks** and needs those loopbacks mutually
reachable — that's the IGP's job. Bring up a small **OSPF** underlay inside AS
65001 (R1/R2/R3 only) *before* any BGP. This also sets up the **next-hop** lesson
in Lab 2.

> 📖 **Deep dive:** [eBGP vs iBGP](CONCEPTS.md#ebgp-vs-ibgp)
> · [Next-hop handling and next-hop-self](CONCEPTS.md#next-hop-handling-and-next-hop-self)

```
r1(config)# router ospf 1
r1(config-router)#  network 10.0.0.0 0.0.0.255 area 0
r1(config-router)#  network 1.1.1.1 0.0.0.0 area 0
```
Do the same on R2 (`2.2.2.2`) and R3 (`3.3.3.3`). **Leave the Lo1 "customer" nets
(172.16.x) out of OSPF** — those get *originated into BGP*, not the IGP.

```
r1# show ip route ospf          ! 2.2.2.2/32 and 3.3.3.3/32 learned as O
r1# ping 3.3.3.3 source 1.1.1.1  ! loopback-to-loopback reachability -- REQUIRED for iBGP
```

> R4 and R5 are separate ASes reached by **directly-connected** eBGP links, so they
> need **no IGP**. Only our AS runs one.

---

# Lab 1 — Your first eBGP session (R1 ↔ R4)

**Goal:** bring up a single **eBGP** session, watch it walk the **state machine**
over TCP 179, read the **OPEN**, advertise a prefix, and see the **UPDATE** on the
wire.

> 📖 **Deep dive:** [TCP 179 and the BGP session state machine](CONCEPTS.md#tcp-179-and-the-bgp-session-state-machine)
> · [The OPEN message and capabilities](CONCEPTS.md#the-open-message-and-capabilities)
> · [BGP message types](CONCEPTS.md#bgp-message-types)

### 1.1 Start the capture FIRST

> **📸 CAPTURE — `r1` interface `Ethernet0/2`** (the eBGP link to R4). Filter
> `tcp.port == 179`. Start it before configuring so you catch the **TCP 3-way
> handshake** and then the BGP **OPEN**.

### 1.2 Configure both ends

```
r1# conf t
r1(config)# router bgp 65001
r1(config-router)#  bgp router-id 1.1.1.1
r1(config-router)#  neighbor 10.14.0.2 remote-as 65002
```
```
r4# conf t
r4(config)# router bgp 65002
r4(config-router)#  bgp router-id 4.4.4.4
r4(config-router)#  neighbor 10.14.0.1 remote-as 65001
r4(config-router)#  network 172.16.4.0 mask 255.255.255.0    ! originate AS65002's net
```

> `remote-as` **different from our AS ⇒ eBGP**; the same ⇒ iBGP. That one number
> changes the TTL, the next-hop behaviour, and the AS-path handling (see the deep
> dive). `network 172.16.4.0 mask ...` tells BGP to **originate** that exact prefix
> — it must exist in the routing table (here it's connected via Lo1).

### 1.3 Watch the session come up

```
r4# show ip bgp summary          ! State cycles Idle -> ... -> Established; then a prefix count
r1# show ip bgp neighbors 10.14.0.2 | include state
```

On the capture, in order:
1. **TCP SYN/SYN-ACK/ACK** to port 179 — BGP is just an application over TCP.
2. **OPEN (`bgp.type==1`)** from each side — expand it and find **My AS**, **Hold
   Time** (default 180s), **BGP Identifier (RID)**, and the **Capabilities**
   (4-byte-AS, MP-BGP, route-refresh). Mismatched AS or RID collision fails here.
3. **KEEPALIVE (`bgp.type==4`)** — confirms the OPEN; the session reaches
   **Established**.
4. **UPDATE (`bgp.type==2`)** — R4 advertises `172.16.4.0/24` with its path
   attributes (Origin, AS-path `65002`, Next-Hop `10.14.0.2`).

### 1.4 Verify the learned route

```
r1# show ip bgp                  ! 172.16.4.0/24, next-hop 10.14.0.2, AS-path 65002
r1# show ip bgp 172.16.4.0       ! full attribute detail for one prefix
r1# show ip route bgp            ! installed as B, AD 20 (eBGP)
```

### 1.5 Bring up the other two eBGP sessions

Complete the external fabric so every AS is reachable two ways:

```
r2(config)# router bgp 65001
r2(config-router)#  bgp router-id 2.2.2.2
r2(config-router)#  neighbor 10.25.0.2 remote-as 65003
```
```
r5(config)# router bgp 65003
r5(config-router)#  bgp router-id 5.5.5.5
r5(config-router)#  neighbor 10.25.0.1 remote-as 65001
r5(config-router)#  neighbor 10.45.0.1 remote-as 65002
r5(config-router)#  network 172.16.5.0 mask 255.255.255.0
```
```
r4(config)# router bgp 65002
r4(config-router)#  neighbor 10.45.0.2 remote-as 65003
```

```
r4# show ip bgp                  ! R4 now sees 172.16.5.0/24 (AS-path 65003) via R5
r2# show ip bgp                  ! R2 sees 172.16.5.0/24 (AS-path 65003) and 172.16.4.0/24 (65003 65002)
```

At this point R1 and R2 each have external routes but **can't see each other's** —
they're not iBGP peers yet. That's Lab 2.

---

# Lab 2 — iBGP, the next-hop problem, and the full-mesh rule

**Goal:** peer R1/R2/R3 with **iBGP**, hit the classic **next-hop-inaccessible**
problem, fix it with `next-hop-self`, and see **why iBGP needs a full mesh** (which
motivates the route reflector in Lab 3).

> 📖 **Deep dive:** [eBGP vs iBGP](CONCEPTS.md#ebgp-vs-ibgp)
> · [The iBGP full-mesh rule](CONCEPTS.md#the-ibgp-full-mesh-rule)
> · [Next-hop handling and next-hop-self](CONCEPTS.md#next-hop-handling-and-next-hop-self)

### 2.1 Build the iBGP mesh over loopbacks

iBGP peers over **loopbacks** (stable, and reachable via the OSPF underlay), so you
must tell BGP to source the session from Lo0:

```
r1(config)# router bgp 65001
r1(config-router)#  neighbor 2.2.2.2 remote-as 65001
r1(config-router)#  neighbor 2.2.2.2 update-source Loopback0
r1(config-router)#  neighbor 3.3.3.3 remote-as 65001
r1(config-router)#  neighbor 3.3.3.3 update-source Loopback0
```
Mirror it on R2 (peers 1.1.1.1, 3.3.3.3) and R3 (peers 1.1.1.1, 2.2.2.2). Also have
R3 originate the internal net:
```
r3(config-router)#  network 172.16.3.0 mask 255.255.255.0
```

```
r1# show ip bgp summary          ! 2.2.2.2 and 3.3.3.3 reach Established (AS 65001)
```

### 2.2 The next-hop problem

Look at an external prefix R1 learned and *reflected* to R3 over iBGP:

```
r3# show ip bgp 172.16.4.0       ! next-hop 10.14.0.2 -- and it's "(inaccessible)"!
r3# show ip route 10.14.0.2      ! nothing -- the eBGP link subnet isn't in OSPF
```

> **Why:** iBGP does **not** change the next-hop by default. R1 hands R3 the route
> with next-hop `10.14.0.2` (R4's eBGP address), but that subnet lives only on the
> R1–R4 link and isn't in the IGP — so R3 can't resolve it and the route is
> **unusable**.

### 2.3 Fix with next-hop-self

> **📸 CAPTURE — `r1` interface `Ethernet0/1`** (core LAN). Filter `bgp.type==2`.

```
r1(config)# router bgp 65001
r1(config-router)#  neighbor 2.2.2.2 next-hop-self
r1(config-router)#  neighbor 3.3.3.3 next-hop-self
```
Do the same on R2 for its iBGP peers. Now R1 rewrites the next-hop to **its own
loopback** (1.1.1.1) before advertising over iBGP — and *that* is reachable via
OSPF.

```
r3# show ip bgp 172.16.4.0       ! next-hop now 1.1.1.1, route VALID and best (>)
r3# show ip route 172.16.4.0     ! installed as B now
```

### 2.4 See the iBGP split-horizon rule

```
r2# show ip bgp 172.16.4.0       ! MISSING -- R3 did NOT pass R1's route to R2
```

> **The rule:** a route learned from an **iBGP** peer is **not** re-advertised to
> another iBGP peer (there's no AS-path loop protection *inside* an AS, so BGP
> forbids the second hop). With three routers that means every iBGP speaker must
> peer with every other — a **full mesh**. R3 has R1's route and R2's route but
> won't relay between them. **Lab 3 fixes this** without a full mesh.

---

# Lab 3 — Route reflector (R3 reflects between R1 and R2)

**Goal:** make **R3 a route reflector** with R1 and R2 as **clients**, so R3 *does*
relay iBGP routes between them — removing the need for an R1↔R2 session.

> 📖 **Deep dive:** [Route reflectors](CONCEPTS.md#route-reflectors)
> · [The iBGP full-mesh rule](CONCEPTS.md#the-ibgp-full-mesh-rule)

### 3.1 Tear down the R1↔R2 iBGP session

To *prove* the reflector works, remove the direct session between the clients:

```
r1(config-router)# no neighbor 2.2.2.2
r2(config-router)# no neighbor 1.1.1.1
```
Now `172.16.5.0/24` (from R2) and `172.16.4.0/24` (from R1) can only reach the far
client **through R3**.

### 3.2 Make R3 the reflector

> **📸 CAPTURE — `r3` interface `Ethernet0/1`.** Filter `bgp.type==2`. Watch R3
> reflect updates it would previously have dropped.

```
r3(config)# router bgp 65001
r3(config-router)#  neighbor 1.1.1.1 route-reflector-client
r3(config-router)#  neighbor 2.2.2.2 route-reflector-client
```

### 3.3 Verify reflection

```
r1# show ip bgp 172.16.5.0       ! now visible on R1, reflected by R3
r1# show ip bgp 172.16.5.0       ! note Originator-ID (2.2.2.2) and Cluster-list (3.3.3.3)
```

> R3 adds two loop-prevention attributes when it reflects: **Originator-ID** (the
> RID that first injected the route) and **Cluster-list** (the RRs it passed
> through). A client that sees its *own* RID in the Originator-ID drops the route —
> that's what replaces iBGP's full-mesh rule safely.

---

# Lab 4 — Best-path selection and the attributes that drive it

**Goal:** AS 65001 can reach `172.16.5.0/24` two ways — **via R2** (AS-path
`65003`) and **via R1** (AS-path `65002 65003`). See which wins by default and
steer it with **weight**, **local-preference**, and **AS-path**.

> 📖 **Deep dive:** [BGP path attributes](CONCEPTS.md#bgp-path-attributes)
> · [The best-path algorithm](CONCEPTS.md#the-best-path-algorithm)
> · [Path manipulation](CONCEPTS.md#path-manipulation)

### 4.1 The default winner: shortest AS-path

```
r1# show ip bgp 172.16.5.0
```
R1 has its **own** eBGP path (AS-path `65002 65003`, via R4) *and* an **iBGP** path
from R2 (AS-path `65003`). Even though the local eBGP path is "closer," **AS-path
length is checked before eBGP-over-iBGP**, so R1 picks the **shorter** path — it
sends traffic across the core to **R2**. Confirm the whole AS agrees:

```
r3# show ip bgp 172.16.5.0       ! best = via R2
r2# show ip route 172.16.5.0     ! R2 is the exit
```

### 4.2 Prefer R1's own link with weight (local to one router)

**Weight** is Cisco-proprietary, **highest wins**, and never leaves the router —
the very first tiebreak:

```
r1(config)# router bgp 65001
r1(config-router)#  neighbor 10.14.0.2 weight 200
```
```
r1# show ip bgp 172.16.5.0       ! R1 now prefers its OWN eBGP path (weight 200 > 0)
r2# show ip bgp 172.16.5.0       ! unchanged -- weight didn't propagate to R2
```
Remove it (`no neighbor 10.14.0.2 weight 200`) so it doesn't mask the next step.

### 4.3 Prefer an exit AS-wide with local-preference

**Local-preference** is **highest-wins** and **propagates through iBGP**, so it sets
policy for the *whole* AS. Make all of AS 65001 exit via **R1 → R4** for
`172.16.5.0/24` by raising local-pref on R1's inbound eBGP updates:

> **📸 CAPTURE — `r1` interface `Ethernet0/1`.** Filter `bgp.type==2`. Watch the
> LOCAL_PREF attribute ride the iBGP UPDATE across the core.

```
r1(config)# ip prefix-list P5 permit 172.16.5.0/24
r1(config)# route-map SET-LP permit 10
r1(config-route-map)#  match ip address prefix-list P5
r1(config-route-map)#  set local-preference 200
r1(config-route-map)# exit
r1(config)# router bgp 65001
r1(config-router)#  neighbor 10.14.0.2 route-map SET-LP in
r1(config-router)# do clear ip bgp * soft
```
```
r2# show ip bgp 172.16.5.0       ! best is now via R1 (local-pref 200 beats default 100)
```
Local-pref (200) beats the shorter AS-path via R2 because **local-pref outranks
AS-path** in the algorithm. Remove the inbound route-map when done.

### 4.4 Walk the algorithm

```
r1# show ip bgp 172.16.5.0       ! read Weight, LocalPref, AS-path, Origin, MED, and "best" reason
```
See the ordered list in [the best-path deep dive](CONCEPTS.md#the-best-path-algorithm)
and the [cheat-sheet](#best-path-algorithm-cheat-sheet) below.

---

# Lab 5 — Policy: filtering and path manipulation

**Goal:** control **which** prefixes you accept/advertise and **how** you influence
a neighbour's choice — the daily job of a BGP admin.

> 📖 **Deep dive:** [Route filtering tools](CONCEPTS.md#route-filtering-tools)
> · [Path manipulation](CONCEPTS.md#path-manipulation)
> · [Authentication, timers, and session knobs](CONCEPTS.md#authentication-timers-and-session-knobs) (route-refresh)

### 5.1 Filter inbound with a prefix-list

Stop R1 from accepting AS 65002's `172.16.4.0/24` (pretend it's unwanted):

```
r1(config)# ip prefix-list BLOCK-4 deny 172.16.4.0/24
r1(config)# ip prefix-list BLOCK-4 permit 0.0.0.0/0 le 32
r1(config)# router bgp 65001
r1(config-router)#  neighbor 10.14.0.2 prefix-list BLOCK-4 in
r1(config-router)# do clear ip bgp 10.14.0.2 soft in
```
```
r1# show ip bgp                  ! 172.16.4.0/24 no longer accepted from R4
```

> **Route-refresh, not reset.** `clear ip bgp ... soft in` asks the neighbour to
> **re-send** its routes (the ROUTE-REFRESH capability, `bgp.type==5`) so your new
> inbound policy applies **without** tearing the session down. Old IOS needed
> `neighbor ... soft-reconfiguration inbound` (a local stored copy) instead —
> capture both and compare.

### 5.2 Filter by AS-path with a filter-list

Block anything that *originated* in AS 65003, regardless of prefix, using an
AS-path regex:

```
r2(config)# ip as-path access-list 10 deny _65003$
r2(config)# ip as-path access-list 10 permit .*
r2(config)# router bgp 65001
r2(config-router)#  neighbor 10.25.0.2 filter-list 10 in
```
`_65003$` matches an AS-path *ending* in 65003 (i.e. originated there). This is how
you filter transit by AS rather than by prefix.

### 5.3 Influence inbound traffic with AS-path prepending

You can't set another AS's local-pref, but you **can** make your path *look
longer* so they avoid it. Have R5 prepend its own AS toward R2, pushing AS 65001 to
prefer the R1 → R4 entry for `172.16.5.0/24`:

> **📸 CAPTURE — `r2` interface `Ethernet0/2`.** Filter `bgp.type==2`. Read the
> AS-path in R5's UPDATE before/after.

```
r5(config)# route-map PREPEND permit 10
r5(config-route-map)#  set as-path prepend 65003 65003
r5(config-route-map)# exit
r5(config)# router bgp 65003
r5(config-router)#  neighbor 10.25.0.1 route-map PREPEND out
r5(config-router)# do clear ip bgp 10.25.0.1 soft out
```
```
r2# show ip bgp 172.16.5.0       ! AS-path via R2 is now 65003 65003 65003 (len 3)
r3# show ip bgp 172.16.5.0       ! AS 65001 now prefers R1's path (65002 65003, len 2)
```

> **Prepending influences *inbound* (return) traffic** by lengthening the AS-path
> others see. It's the standard eBGP tool because you can't reach into a neighbour
> AS to set its local-pref.

### 5.4 Strip private ASNs on the way out

`remove-private-as` cleans private ASNs (64512–65534) from the AS-path on an eBGP
advertisement — what you'd do facing a real upstream:

```
r2(config-router)# neighbor 10.25.0.2 remove-private-as
```
(Everything in this lab is private, so use this to *observe* the AS-path being
rewritten on the capture rather than for a functional change.)

---

# Lab 6 — MP-BGP: carrying an IPv6 address family

**Goal:** BGP isn't just IPv4. **Multiprotocol BGP** carries many **address
families**; add **IPv6 unicast** and advertise the `2001:db8:*` nets.

> 📖 **Deep dive:** [MP-BGP and address families](CONCEPTS.md#mp-bgp-and-address-families)

### 6.0 Enable IPv6 routing first (the gotcha)

```
r1(config)# ipv6 unicast-routing
```
On **every** router. Without it IPv6 BGP won't forward and next-hops won't resolve.

### 6.1 eBGP for IPv6 over the link addresses

> **📸 CAPTURE — `r1` interface `Ethernet0/2`.** Filter `bgp`. A separate IPv6 BGP
> session forms over the link's IPv6 addresses.

```
r1(config)# router bgp 65001
r1(config-router)#  neighbor 2001:DB8:14::2 remote-as 65002
r1(config-router)#  address-family ipv6 unicast
r1(config-router-af)#   neighbor 2001:DB8:14::2 activate
r1(config-router-af)#  exit-address-family
```
```
r4(config)# router bgp 65002
r4(config-router)#  neighbor 2001:DB8:14::1 remote-as 65001
r4(config-router)#  address-family ipv6 unicast
r4(config-router-af)#   neighbor 2001:DB8:14::1 activate
r4(config-router-af)#   network 2001:DB8:4::/64
```

> **Why a second neighbor?** The IPv4 session (Lab 1) and this IPv6 session are
> independent TCP sessions here. You *can* carry IPv6 NLRI over the IPv4 session,
> but peering over IPv6 addresses is clearer to reason about and to capture.
> Note also: an address family must be **`activate`d** per neighbor — enabling BGP
> doesn't automatically carry every AF to every peer.

### 6.2 iBGP for IPv6 (shortcut: peer on the shared LAN)

Since R1/R2/R3 share the core LAN, peer iBGP-v6 on the **directly-connected**
global addresses — no IPv6 IGP needed:

```
r1(config-router)# neighbor 2001:DB8:0::3 remote-as 65001
r1(config-router)# address-family ipv6 unicast
r1(config-router-af)#  neighbor 2001:DB8:0::3 activate
```
(peer R1↔R3 and R2↔R3, with R3 the v6 reflector too — add `route-reflector-client`
under the IPv6 AF on R3)

> In a routed core you'd run an **IPv6 IGP** and peer on IPv6 loopbacks, exactly as
> Lab 0/2 did for IPv4 — the shared LAN just lets us skip that here.

### 6.3 Verify

```
r1# show bgp ipv6 unicast              ! 2001:db8:4::/64, :5::/64 with v6 next-hops
r1# show bgp ipv6 unicast summary
r1# show ipv6 route bgp
```

---

# Lab 7 — Neighbor options grab-bag

Short, high-value session knobs from ENARSI 1.11.b.

> 📖 **Deep dive:** [Authentication, timers, and session knobs](CONCEPTS.md#authentication-timers-and-session-knobs)

### 7.1 MD5 authentication

> **📸 CAPTURE — `r1` e0/2**, filter `tcp.port==179`. Set it on one side only and
> watch the session drop, then match.

```
r1(config-router)# neighbor 10.14.0.2 password CISCO
r4(config-router)# neighbor 10.14.0.1 password CISCO
```
A mismatch/absent password holds the session **down** (TCP-level MD5). On the
capture, the TCP segments now carry the **MD5 signature** TCP option.

### 7.2 Timers

```
r1(config-router)# neighbor 10.14.0.2 timers 10 30      ! keepalive 10s, hold 30s (per-neighbor)
r1# show ip bgp neighbors 10.14.0.2 | include hold|Keepalive
```
The **lower** of the two hold-times is negotiated in the OPEN; hold **0** disables
keepalives.

### 7.3 Peer groups (configuration reuse)

Collapse repeated iBGP neighbor config into one **peer-group** (also gains update
generation efficiency):

```
r3(config-router)# neighbor IBGP peer-group
r3(config-router)# neighbor IBGP remote-as 65001
r3(config-router)# neighbor IBGP route-reflector-client
r3(config-router)# neighbor 1.1.1.1 peer-group IBGP
r3(config-router)# neighbor 2.2.2.2 peer-group IBGP
```

### 7.4 eBGP over loopbacks (multihop)

eBGP defaults to **TTL 1** (directly-connected). To peer eBGP on loopbacks you must
raise the TTL and source from the loopback (and have a route to the peer loopback):

```
r1(config-router)# neighbor 4.4.4.4 remote-as 65002
r1(config-router)# neighbor 4.4.4.4 ebgp-multihop 2
r1(config-router)# neighbor 4.4.4.4 update-source Loopback0
```
(needs a static/route to 4.4.4.4; a good exercise in *why* `ebgp-multihop` exists.)

### 7.5 4-byte ASN (optional, disruptive)

Reconfiguring an AS resets sessions, so do this last. Recreate R5 in a **4-byte**
AS to see asplain notation and the 4-byte-AS capability negotiated in the OPEN:

```
r5(config)# no router bgp 65003
r5(config)# router bgp 4200000005
...
```
(and update R2/R4's `remote-as` to `4200000005`). Old 2-byte-only speakers would
see AS_TRANS (23456) — modern IOS negotiates 4-byte natively in the OPEN
capabilities.

---

## Suggested end-state verification

```
r3# show ip bgp                 ! all prefixes; best paths marked ">"
r1# show ip bgp summary         ! eBGP (65002) + iBGP (65001) neighbors Established, prefix counts
r1# show ip bgp 172.16.5.0      ! the two-path prefix -- read the best-path reason
r3# show ip bgp neighbors 1.1.1.1 | include Route-Reflector   ! R1/R2 are RR clients
r1# show bgp ipv6 unicast summary   ! (Lab 6) MP-BGP IPv6 AF
any# show ip bgp neighbors <peer> ! states, timers, capabilities, message counters
```

## BGP message / state cheat-sheet (what you captured)

| Type | Message       | Role                                                  |
|------|---------------|-------------------------------------------------------|
| 1    | OPEN          | AS, hold-time, RID, capabilities — sent once at start |
| 2    | UPDATE        | advertise NLRI + attributes, and/or withdraw routes   |
| 3    | NOTIFICATION  | fatal error → session torn down                       |
| 4    | KEEPALIVE     | hold-timer heartbeat                                  |
| 5    | ROUTE-REFRESH | "re-send me your routes" (soft policy re-apply)       |

**Session states:** Idle → Connect → Active → OpenSent → OpenConfirm →
**Established**. Flapping between **Active/Connect** = TCP not completing (ACL,
routing, wrong `update-source`); stuck in **Idle** = no route to the peer / admin
down.

## Best-path algorithm cheat-sheet

Compared **in order**; first difference wins (all else equal, fall through):

1. **Weight** — highest (Cisco-only, local to the router)
2. **Local-Preference** — highest (AS-wide, via iBGP)
3. **Locally originated** (network/aggregate/redistribute) preferred
4. **AS-path** — shortest
5. **Origin** — IGP (i) < EGP (e) < Incomplete (?)
6. **MED** — lowest (only among paths from the *same* neighbour AS)
7. **eBGP over iBGP**
8. **Lowest IGP metric** to the next-hop
9. oldest eBGP / lowest RID / lowest neighbor address (final tiebreaks)

---

## Reset the whole lab

```bash
sudo containerlab destroy -t bgp-lab.clab.yml --cleanup
sudo containerlab deploy  -t bgp-lab.clab.yml
```

Because the startup configs contain **no IGP and no BGP**, every redeploy gives you
a clean slate.

---

## How this compares to the IGP labs

| Concept              | IGPs (`../ospf`, `../eigrp`)          | BGP (here)                              |
|----------------------|----------------------------------------|-----------------------------------------|
| Goal                 | find the **shortest path**             | enforce **policy** between ASes         |
| Transport            | IP (89) / IP (88), multicast           | **TCP 179**, unicast, manual peers      |
| Metric               | cost / composite DUAL metric           | **path attributes** + best-path list    |
| Scope                | within an AS                           | **between** ASes (and iBGP within)      |
| Loop prevention      | SPF / feasibility condition            | **AS-path** (eBGP), split-horizon + RR (iBGP) |
| Neighbor discovery   | automatic (Hellos)                     | **manually configured** peers           |

Next in the ENARSI track: **route redistribution & filtering** (mutual
redistribution between these BGP/OSPF/EIGRP domains, tags, and admin-distance),
then **DMVPN**.
