# CCNP IPv6 Fundamentals Lab (Cisco IOL + Edgeshark)

A 3-router + 1-switch `cisco_iol` topology for getting **solid on IPv6 from the
packet up** — *before* you tackle the routing-protocol address-family labs
(OSPFv3, EIGRP-for-IPv6). The nodes boot with their interfaces **up but with no
IPv6 addresses and IPv6 routing switched off**. You assign every address, turn on
routing, and use **Edgeshark packet captures** to *watch* the things that make
IPv6 different from IPv4 actually happen on the wire.

The whole IPv6 control plane is **ICMPv6 multicast** — and that is exactly why a
capture-driven lab beats a textbook here. You will *see*:

- The four **address types** — global unicast, **link-local**, unique-local,
  multicast — and how a link-local address is **auto-generated** the moment you
  enable IPv6 on an interface
- **Modified EUI-64** turning a MAC into a 64-bit interface ID
- **NDP** (Neighbor Discovery) **replacing ARP** — Neighbor Solicitation /
  Advertisement (NS/NA) and the **neighbor cache**
- **DAD** (Duplicate Address Detection) firing on every new address
- The **solicited-node multicast** address and why it exists
- **SLAAC** — a host building its own global address from a **Router
  Advertisement** (RS → RA), and the **M/O flags** that hand off to DHCPv6
- **Static routing over IPv6**, including the **link-local next-hop** rule

> **No broadcast in IPv6.** Everything ARP used broadcast for, NDP does with
> *multicast* — that single idea explains most of what you'll capture below.

> 📖 **This lab has a companion:** [`CONCEPTS.md`](CONCEPTS.md) is the in-depth
> reference behind every step. Each lab below opens with a **Deep dive** callout
> linking to the relevant sections — follow them when you want the *why*, skip
> them when you just want to type. You can also read `CONCEPTS.md` end-to-end as a
> primer before deploying.

---

## Topology

```
                   Shared LAN  2001:db8:acad:a::/64
              (NDP / DAD / solicited-node multicast / SLAAC)

        R1 (...a::1) ───────┐              ┌─────── R2 (...a::2)
         gateway /          │              │         second router
         RA source       ┌──┴──────────────┴──┐      (NS/NA + DAD peer)
                         │      SW1  (L2)      │
                         └──────────┬──────────┘
                                    │
                             R3  (SLAAC host)
                          'ipv6 address autoconfig'

        R1 ───────── 2001:db8:acad:12::/64  (point-to-point) ───────── R2
                 (static routing + link-local next-hop lesson)
```

**Why this shape?** R1, R2 and R3 share one Ethernet segment (SW1, a zero-config
L2 IOL switch) so *all* the IPv6 multicast control traffic (RS/RA/NS/NA) is
visible when you capture on the LAN. **R3 is deliberately left bare** — it becomes
a **SLAAC client** so you can watch a host autoconfigure. The **R1–R2
point-to-point** link is for the static-routing / link-local-next-hop section.

### Roles

| Node | Role                                                                    |
|------|-------------------------------------------------------------------------|
| R1   | LAN **default gateway** and **Router-Advertisement source** (SLAAC)     |
| R2   | Second router on the LAN — peer for **NS/NA** and **DAD** demonstrations |
| R3   | The **"host"** — a **SLAAC client** (`ipv6 address autoconfig`)          |
| SW1  | L2 switch — builds the shared multicast segment (no config)             |

### Suggested addressing (you configure all of it)

| Link / Interface        | IPv6 prefix              | R1        | R2        | R3        |
|-------------------------|--------------------------|-----------|-----------|-----------|
| LAN (e0/1 ↔ SW1)        | `2001:db8:acad:a::/64`   | `…a::1`   | `…a::2`   | *SLAAC*   |
| R1 ↔ R2 p2p (e0/2)      | `2001:db8:acad:12::/64`  | `…12::1`  | `…12::2`  | —         |
| Loopback0               | —                        | `2001:db8:acad:1::1/64` | `2001:db8:acad:2::1/64` | — |

`2001:db8::/32` is the reserved **documentation** prefix (like `192.0.2.0/24` for
IPv4) — always use it for labs. `acad` is a memorable subnet block; `:a:` and
`:12:` are the per-link subnet IDs.

Interface mapping: containerlab `ethX` → IOL `Ethernet0/X`. `E0/0` is management.
So `e0/1` = `Ethernet0/1`, `e0/2` = `Ethernet0/2`.

---

## Deploy

```bash
cd /home/lab/clabs/ccnp/ipv6-fundamentals
sudo containerlab deploy -t ipv6-lab.clab.yml
```

Log in to any node with `admin` / `admin`:

```bash
ssh admin@clab-ipv6-lab-r1        # or: docker exec -it clab-ipv6-lab-r1 telnet localhost
```

Give the nodes ~60–90s after deploy. Destroy with:

```bash
sudo containerlab destroy -t ipv6-lab.clab.yml --cleanup
```

### Sanity check before you start (no IPv6 yet)

```
r1# show ipv6 interface brief        ! interfaces up, but "unassigned" -- blank slate
r1# show ipv6 route                   ! % IPv6 routing not enabled (or empty)
r1# show running-config | include ipv6 unicast-routing   ! nothing -- you enable it
```

---

## Using Edgeshark for captures

Edgeshark (Ghostwire + Packetflix) lets you capture on any container interface
from the browser: open the **Ghostwire** UI, find the container (e.g.
`clab-ipv6-lab-r1`), expand its interfaces, and click the **capture** (shark-fin)
icon next to `Ethernet0/1`. A browser Wireshark session opens and streams live.

IPv6's control plane is **ICMPv6**. Handy display filters:

| Filter                        | Shows                                             |
|-------------------------------|---------------------------------------------------|
| `icmpv6`                      | all ICMPv6 (NDP lives here)                        |
| `icmpv6.type == 133`          | **Router Solicitation (RS)** — host → routers     |
| `icmpv6.type == 134`          | **Router Advertisement (RA)** — router → hosts    |
| `icmpv6.type == 135`          | **Neighbor Solicitation (NS)** — ARP's "who-has"  |
| `icmpv6.type == 136`          | **Neighbor Advertisement (NA)** — ARP's "is-at"   |
| `icmpv6.type == 128`/`129`    | Echo request / reply (ping)                        |
| `ipv6.dst == ff02::1`         | traffic to **all-nodes** multicast                |
| `ipv6.dst == ff02::2`         | traffic to **all-routers** multicast              |

Well-known multicast addresses to recognize (there is **no broadcast**):

- **ff02::1** — *all-nodes* on the link (RAs are sent here)
- **ff02::2** — *all-routers* on the link (RSs are sent here)
- **ff02::1:ffXX:XXXX** — **solicited-node** multicast (NS/DAD target); the
  `XX:XXXX` is the **low 24 bits** of the target's address. Its Ethernet MAC is
  `33:33:ff:XX:XX:XX`.

> 📖 **Deep dive:** [Why there is no broadcast](CONCEPTS.md#why-there-is-no-broadcast)
> · [Multicast and the solicited-node address](CONCEPTS.md#multicast-and-the-solicited-node-address)

> Tip: NDP is chatty. Start the capture, apply the one config line the step calls
> out, then use the `icmpv6.type ==` filters to isolate the exact exchange.

---

# Lab 1 — Addressing: the auto link-local, and the four address types

**Goal:** enable IPv6 on an interface and watch a **link-local** address appear
*for free*, then add a **global** address and see **DAD** fire. Learn to read an
IPv6 address along the way.

> 📖 **Deep dive:** [Anatomy of an IPv6 address](CONCEPTS.md#anatomy-of-an-ipv6-address)
> · [Address types and scopes](CONCEPTS.md#address-types-and-scopes)
> · [Link-local addresses and zones](CONCEPTS.md#link-local-addresses-and-zones)
> · [DAD](CONCEPTS.md#duplicate-address-detection-dad)

### 1.1 Turn on IPv6 routing (do this first, everywhere)

```
r1(config)# ipv6 unicast-routing
```
On a router this does two jobs: it lets the box **route** IPv6, and it lets it
**send Router Advertisements**. Forget it and SLAAC (Lab 4) silently never works.
Do it on **R1 and R2** now (R3 is a host — leave it off for now).

### 1.2 The link-local address is automatic

> **📸 CAPTURE — `r1` interface `Ethernet0/1`.** Filter `icmpv6`. Start it *before*
> the next command so you catch DAD.

```
r1(config)# interface Ethernet0/1
r1(config-if)#  ipv6 enable          ! turn on IPv6 with NO global address yet
r1(config-if)# end
r1# show ipv6 interface Ethernet0/1
```

Even with no global address, the interface now has a **link-local** address in
**`fe80::/64`**, built automatically from the interface MAC (modified EUI-64 —
Lab 2). Key facts to burn in:

- Every IPv6 interface **always** has a link-local address. It's used for all
  on-link control traffic (NDP, and routing-protocol adjacencies — that's why
  OSPFv3/EIGRPv6 peer over `fe80::`).
- Link-local addresses are **not routable** off the link. Two different links can
  reuse the same `fe80::` address — which is why link-local next-hops need an
  exit interface (Lab 5).
- The interface also **joined multicast groups**: `ff02::1` (all-nodes) and its
  **solicited-node** group `ff02::1:ffXX:XXXX`. Confirm in the `show` output.

### 1.3 Add a global unicast address — watch DAD

```
r1(config)# interface Ethernet0/1
r1(config-if)#  ipv6 address 2001:DB8:ACAD:A::1/64
```

On the capture, the instant you add the address you'll see a **Neighbor
Solicitation (`icmpv6.type==135`)** whose:

- **source is `::`** (the unspecified address — the sender doesn't own an address
  yet, this address is still *tentative*),
- **destination is the solicited-node multicast** of `…a::1` →
  `ff02::1:ff00:1`,
- **target is `…a::1`** itself.

That's **Duplicate Address Detection**: R1 is asking *"does anyone already own the
address I want to claim?"* No answer → the address leaves **tentative** and
becomes usable. If R2 owned it, R2 would reply NA and R1's address would go
**DUPLICATE** (a great failure to reproduce in Lab 3).

```
r1# show ipv6 interface Ethernet0/1     ! now lists link-local AND 2001:db8:acad:a::1, state up
```

Do the same on **R2 e0/1** (`…a::2`) and both p2p ends (Lab 5 uses those).

### 1.4 Read the address / the four types

| Type            | Range          | Purpose                                             |
|-----------------|----------------|-----------------------------------------------------|
| Global unicast  | `2000::/3`     | Internet-routable (your `2001:db8:…`)               |
| Link-local      | `fe80::/10`    | On-link only; auto-generated; NDP & routing peers   |
| Unique-local    | `fc00::/7`     | Private, site-scoped (the "RFC 1918 of IPv6")       |
| Multicast       | `ff00::/8`     | One-to-many; **replaces broadcast** (`ff02::1` etc.)|

Compression rules to practise on `2001:0db8:acad:000a:0000:0000:0000:0001`:
leading zeros in a hextet drop (`0db8`→`db8`, `000a`→`a`), and **one** run of
all-zero hextets collapses to `::` → **`2001:db8:acad:a::1`**. You may use `::`
only once per address.

---

# Lab 2 — Modified EUI-64: how a MAC becomes an interface ID

**Goal:** understand the 64-bit interface ID in that auto link-local (and in
EUI-64 global addresses), because SLAAC (Lab 4) uses the same math.

> 📖 **Deep dive:** [Modified EUI-64, bit by bit](CONCEPTS.md#modified-eui-64-bit-by-bit)
> (includes why real hosts use random/privacy IDs instead)

### 2.1 See the MAC, then the derived interface ID

```
r1# show interfaces Ethernet0/1 | include address       ! the 48-bit MAC, e.g. aabb.cc00.0110
r1# show ipv6 interface Ethernet0/1                       ! link-local fe80::a8bb:ccff:fe00:110
```

**Modified EUI-64** turns a 48-bit MAC into a 64-bit interface ID by:

1. splitting the MAC in half and inserting **`FFFE`** in the middle
   (`aabb.cc` + **`fffe`** + `00.0110`), and
2. **flipping the 7th bit** (the Universal/Local bit) of the first byte
   (`aa` = `1010 1010` → `1010 1000` = `a8`).

So MAC `aabb.cc00.0110` → interface ID `a8bb:ccff:fe00:0110` → link-local
`fe80::a8bb:ccff:fe00:110`. The **`ff:fe`** in the middle is the dead giveaway
that an address was built by EUI-64.

### 2.2 Build a global address with EUI-64

```
r2(config)# interface Ethernet0/2
r2(config-if)#  ipv6 address 2001:DB8:ACAD:12::/64 eui-64
r2# show ipv6 interface Ethernet0/2      ! host part is the EUI-64 ID, not ::2
```

> Real hosts often use **random/temporary** interface IDs (privacy extensions)
> instead of EUI-64, precisely because EUI-64 embeds the MAC and is trackable.
> You'll see a non-EUI-64 (random) ID if you enable SLAAC privacy — noted in Lab 4.

---

# Lab 3 — NDP replaces ARP: Neighbor Solicitation / Advertisement

**Goal:** watch address resolution happen with **multicast NS** + **unicast NA**
instead of ARP, inspect the **neighbor cache**, and reproduce a **duplicate
address**.

> 📖 **Deep dive:** [NDP: the protocol that replaced ARP](CONCEPTS.md#ndp-the-protocol-that-replaced-arp)
> · [The neighbor cache and NUD](CONCEPTS.md#the-neighbor-cache-and-nud)
> · [Multicast and the solicited-node address](CONCEPTS.md#multicast-and-the-solicited-node-address)
> · [DAD](CONCEPTS.md#duplicate-address-detection-dad)

### 3.1 Resolve a neighbor (the ARP replacement)

Make sure R1 (`…a::1`) and R2 (`…a::2`) both have their LAN global addresses
(Lab 1). Clear R1's neighbor cache so you capture the resolution cleanly:

> **📸 CAPTURE — `r1` interface `Ethernet0/1`.** Filter `icmpv6.type==135 ||
> icmpv6.type==136`.

```
r1# clear ipv6 neighbors
r1# ping 2001:DB8:ACAD:A::2
```

On the capture, in order:

1. **NS (`type 135`)** from R1 → **solicited-node multicast** `ff02::1:ff00:2`
   (not a broadcast!), *target = `…a::2`*, carrying R1's **source link-layer
   address** option. Only R2 (and any node sharing those low 24 bits) is
   interrupted — that's the whole point of the solicited-node address.
2. **NA (`type 136`)** from R2 → **unicast** back to R1, with R2's link-layer
   address and the **Solicited (S) flag** set.

```
r1# show ipv6 neighbors                 ! R2's link-local/global -> MAC, state REACH/STALE
```

The **neighbor cache** is IPv6's ARP table. States you'll see: `INCMP`
(resolving), `REACH` (fresh), `STALE`, `DELAY`, `PROBE`. `show ipv6 neighbors` is
your `show ip arp`.

### 3.2 A discovery trick: ping all-nodes

```
r1# ping ff02::1                        ! all-nodes multicast on the link
```
Every IPv6 node on the segment answers — an instant "who's on this link?" that
has no clean IPv4 equivalent. Watch the NA/echo replies stream in.

### 3.3 Reproduce a duplicate address (DAD in action)

> **📸 CAPTURE — `r2` interface `Ethernet0/1`.** Filter `icmpv6.type==135`.

Try to give R2 an address R1 already owns:

```
r2(config)# interface Ethernet0/1
r2(config-if)#  ipv6 address 2001:DB8:ACAD:A::1/64     ! R1 already has this!
```

R2 sends a **DAD NS** (source `::`, target `…a::1`); **R1 answers**, so R2 logs
`%IPV6-4-DUPLICATE` and the address is placed in **DUPLICATE** state (unusable):

```
r2# show ipv6 interface Ethernet0/1     ! the address shows [DUP]/DUPLICATE
```
Remove it (`no ipv6 address 2001:DB8:ACAD:A::1/64`). This is the mechanism that
made the `::` -source NS in Lab 1.3 meaningful.

---

# Lab 4 — SLAAC: a host builds its own address from an RA

**Goal:** turn **R3 into a host** and watch **Stateless Address Autoconfiguration**
— RS → RA → address — the single most important IPv6 concept to *see*, not read.

> 📖 **Deep dive:** [Router Advertisements and SLAAC](CONCEPTS.md#router-advertisements-and-slaac)
> · [SLAAC vs DHCPv6: the M and O flags](CONCEPTS.md#slaac-vs-dhcpv6-the-m-and-o-flags)

### 4.1 Confirm R1 is advertising the prefix

A router with `ipv6 unicast-routing` (Lab 1.1) and a global address on an
interface **already sends periodic RAs** out that interface. Check what R1 offers:

```
r1# show ipv6 interface Ethernet0/1     ! note "ND advertised prefix 2001:DB8:ACAD:A::/64",
                                        ! router lifetime, and the flags (default: A set, M/O clear)
```

The prefix's **A-flag (Autonomous)** being set is what *permits* SLAAC. The RA's
**M (Managed)** and **O (Other)** flags — both clear by default — say "no DHCPv6
needed."

### 4.2 Make R3 a SLAAC client and watch it autoconfigure

> **📸 CAPTURE — `r3` interface `Ethernet0/1`.** Filter `icmpv6`. Start it *before*
> the command.

```
r3# conf t
r3(config)# ipv6 unicast-routing        ! (optional on a host, but harmless)
r3(config)# interface Ethernet0/1
r3(config-if)#  ipv6 address autoconfig  ! become a SLAAC client
r3(config-if)# end
```

On the capture, in order:

1. **RS (`type 133`)** from R3 → **all-routers** `ff02::2`: *"any routers here?
   send an RA now"* (a booting host doesn't wait for the periodic RA).
2. **RA (`type 134`)** from R1's **link-local** → `ff02::1` (all-nodes), carrying
   the **prefix `2001:db8:acad:a::/64`**, the A/M/O flags, router lifetime, and
   MTU.
3. R3 **combines the prefix with its own interface ID** (EUI-64 or random) to form
   a global address, runs **DAD** on it (a `::`-source NS), and installs a
   **default route** toward R1's link-local.

```
r3# show ipv6 interface Ethernet0/1     ! global 2001:db8:acad:a:… built via SLAAC
r3# show ipv6 routers                    ! R1 listed as the default router (from its RA)
r3# show ipv6 route ::/0                  ! default route via R1's fe80:: (learned, not typed)
r3# ping 2001:DB8:ACAD:A::1              ! reach the gateway
```

> **The SLAAC vs DHCPv6 hand-off (exam gold):** the RA flags decide how a host
> gets configured —
> - **A-flag** on the prefix → host may **SLAAC** an address from it.
> - **M-flag** set → "use **stateful DHCPv6**" for the **address**.
> - **O-flag** set → "use DHCPv6 for **other** info only" (DNS, etc.); address
>   still via SLAAC.
>
> Flip them on R1 with `ipv6 nd managed-config-flag` / `ipv6 nd other-config-flag`
> and re-capture the RA to see the bits change. (Actually handing out addresses is
> the **DHCPv6** lab — the follow-on to this one.)

### 4.3 Kill the RA and watch SLAAC fail

```
r1(config)# interface Ethernet0/1
r1(config-if)#  ipv6 nd ra suppress        ! stop sending RAs
```
Bounce R3's interface (`shut`/`no shut`) and confirm it can no longer build a
global address or a default route — the classic "host has only a link-local, no
gateway" symptom. Re-enable with `no ipv6 nd ra suppress`.

---

# Lab 5 — Reachability & static routing (the link-local next-hop rule)

**Goal:** route between the two loopbacks over the R1–R2 p2p link, and learn the
one static-route rule that trips everyone up in IPv6.

> 📖 **Deep dive:** [Routing and the link-local next-hop rule](CONCEPTS.md#routing-and-the-link-local-next-hop-rule)
> · [The /128 local route](CONCEPTS.md#the-128-local-route)

### 5.1 Address the p2p link and loopbacks

```
r1(config)# interface Ethernet0/2
r1(config-if)#  ipv6 address 2001:DB8:ACAD:12::1/64
r1(config)# interface Loopback0
r1(config-if)#  ipv6 address 2001:DB8:ACAD:1::1/64
```
(and on R2: `…12::2` on e0/2, `2001:db8:acad:2::1/64` on Lo0)

```
r1# show ipv6 interface brief            ! link-local + global on each interface
r1# ping 2001:DB8:ACAD:12::2             ! p2p reachability first
```

### 5.2 Global next-hop vs link-local next-hop

A static route to R2's loopback via the **global** next-hop is what you'd expect:

```
r1(config)# ipv6 route 2001:DB8:ACAD:2::/64 2001:DB8:ACAD:12::2
```

But IPv6 next-hops are very often **link-local** (that's what RAs and routing
protocols give you). A link-local next-hop is ambiguous on its own — the same
`fe80::` can exist on every link — so IOS **requires you to name the exit
interface** too:

```
r1(config)# no ipv6 route 2001:DB8:ACAD:2::/64 2001:DB8:ACAD:12::2
r1(config)# ipv6 route 2001:DB8:ACAD:2::/64 Ethernet0/2 FE80::A8BB:CCFF:FE00:XXXX
```
(use R2's actual e0/2 link-local from `show ipv6 interface Ethernet0/2` on R2)

```
r1# show ipv6 route static
r1# ping 2001:DB8:ACAD:2::1 source Loopback0
```

> **The rule:** a static route with a **link-local** next-hop **must** also specify
> an **exit interface**; a global next-hop may stand alone. Getting a `%Invalid
> next hop` or a route that won't install almost always traces back to this.

### 5.3 (Optional) default route toward the LAN

```
r2(config)# ipv6 route ::/0 2001:DB8:ACAD:12::1
```
`::/0` is the IPv6 default route (the `0.0.0.0/0` equivalent).

---

# Quick extras (short, high-value)

### Tune the RA (all captured on `icmpv6.type==134`)
```
r1(config-if)# ipv6 nd ra interval 30            ! how often unsolicited RAs go out
r1(config-if)# ipv6 nd prefix 2001:DB8:ACAD:A::/64 300 120   ! valid / preferred lifetimes
r1(config-if)# ipv6 nd prefix 2001:DB8:ACAD:A::/64 no-autoconfig   ! clear the A-flag -> SLAAC off for it
```

### Verify / troubleshoot toolbox
```
any# show ipv6 interface brief          ! the IPv6 'show ip int brief'
any# show ipv6 neighbors                ! the IPv6 ARP table (neighbor cache)
any# show ipv6 routers                  ! RAs this node has heard (default gateways)
any# show ipv6 route                    ! C connected, L local (/128), S static, ND default
any# debug ipv6 nd                      ! NDP events (RS/RA/NS/NA/DAD) live in the log
```

### The /128 "local" route
Every configured IPv6 address installs a matching **`L … /128` local route**
(the router answering for itself) alongside the **`C … /64` connected** route —
a detail that surprises people reading `show ipv6 route`.

---

## IPv6 quick cheat-sheets (what you captured)

**ICMPv6 / NDP message types**

| Type | Message                    | Replaces / does                              | Sent to        |
|------|----------------------------|----------------------------------------------|----------------|
| 133  | Router Solicitation (RS)   | "routers, advertise now" (host boot)         | `ff02::2`      |
| 134  | Router Advertisement (RA)  | prefix + flags + gateway (SLAAC engine)      | `ff02::1`      |
| 135  | Neighbor Solicitation (NS) | ARP "who-has" **and** DAD                     | solicited-node |
| 136  | Neighbor Advertisement (NA)| ARP "is-at"                                   | unicast / `ff02::1` |
| 128/129 | Echo request / reply    | ping                                          | unicast/mcast  |

**Key multicast addresses**

| Address            | Meaning                                                  |
|--------------------|----------------------------------------------------------|
| `ff02::1`          | all-nodes on the link (RAs, "ping everyone")             |
| `ff02::2`          | all-routers on the link (RSs)                            |
| `ff02::1:ffXX:XXXX`| solicited-node (NS/DAD); low 24 bits of the target       |
| `ff02::5` / `::6`  | OSPFv3 AllSPFRouters / AllDRouters (next labs)           |
| `ff02::a`          | EIGRP for IPv6 (next labs)                               |

**Address types**

| Type           | Range       | Note                                    |
|----------------|-------------|-----------------------------------------|
| Global unicast | `2000::/3`  | routable; docs use `2001:db8::/32`      |
| Link-local     | `fe80::/10` | on-link only; auto; NDP + routing peers |
| Unique-local   | `fc00::/7`  | private/site-scoped                     |
| Multicast      | `ff00::/8`  | replaces broadcast                      |
| Loopback / unspecified | `::1` / `::` | host itself / "no address yet" (DAD) |

---

## Reset the whole lab

```bash
sudo containerlab destroy -t ipv6-lab.clab.yml --cleanup
sudo containerlab deploy  -t ipv6-lab.clab.yml
```

Because the startup configs contain **no IPv6 addressing at all**, every redeploy
gives you a clean slate to practice against.

---

## Where this leads next

This lab is the **foundation** for every IPv6 topic in ENARSI. Once RS/RA/NS/NA,
SLAAC and link-local next-hops feel obvious, the rest is just wrappers around what
you saw here:

| You learned here            | It reappears as…                                             |
|-----------------------------|-------------------------------------------------------------|
| Link-local + `ff02::` NDP   | **OSPFv3** (`ff02::5/6`) and **EIGRP-for-IPv6** (`ff02::a`) peer over link-local — see [`../ospf`](../ospf) Lab 11 and [`../eigrp`](../eigrp) Lab 7 |
| RA **M/O flags**            | **DHCPv6** stateless vs stateful (follow-on lab)            |
| RAs are an attack surface   | **IPv6 First-Hop Security** — **RA guard**, DHCPv6 guard, ND inspection (ENARSI 3.4) |
| Address types & prefixes    | **IPv6 traffic filters / ACLs** (ENARSI 3.2.b)             |

Suggested follow-on: an **IPv6 services** lab (DHCPv6 stateless/stateful/relay +
an RA-guard first-hop-security teaser), which picks up exactly where Lab 4's M/O
flags leave off.
