# IPv6 Fundamentals — In-Depth Concepts

The companion reference to the [README](README.md) lab guide. The README is the
*hands-on* path (configure this, capture that); this file is the *why* behind each
step. Each 📖 **Deep dive** link in the README jumps to a section here. You can
also read it straight through as a primer.

**Contents**
- [Anatomy of an IPv6 address](#anatomy-of-an-ipv6-address)
- [Address types and scopes](#address-types-and-scopes)
- [Why there is no broadcast](#why-there-is-no-broadcast)
- [Multicast and the solicited-node address](#multicast-and-the-solicited-node-address)
- [Modified EUI-64, bit by bit](#modified-eui-64-bit-by-bit)
- [Link-local addresses and zones](#link-local-addresses-and-zones)
- [NDP: the protocol that replaced ARP](#ndp-the-protocol-that-replaced-arp)
- [The neighbor cache and NUD](#the-neighbor-cache-and-nud)
- [Duplicate Address Detection (DAD)](#duplicate-address-detection-dad)
- [Router Advertisements and SLAAC](#router-advertisements-and-slaac)
- [SLAAC vs DHCPv6: the M and O flags](#slaac-vs-dhcpv6-the-m-and-o-flags)
- [Routing and the link-local next-hop rule](#routing-and-the-link-local-next-hop-rule)
- [The /128 local route](#the-128-local-route)
- [IPv4 to IPv6 quick translation](#ipv4-to-ipv6-quick-translation)

---

## Anatomy of an IPv6 address

An IPv6 address is **128 bits**, written as **eight 16-bit "hextets"** in
hexadecimal, separated by colons:

```
2001:0db8:acad:000a:0000:0000:0000:0001
└─────────────────┘ └─────────────────┘
   64-bit prefix        64-bit interface ID
 (network + subnet)     (which host on the link)
```

The near-universal split is **/64**: the first 64 bits identify the link (subnet),
the last 64 identify the interface on it. That's why almost every LAN prefix you
see is a `/64` — SLAAC and EUI-64 *require* a 64-bit interface ID to work.

**Two compression rules** (apply both, in this order, to shorten an address):

1. **Drop leading zeros** in each hextet: `0db8`→`db8`, `000a`→`a`, `0001`→`1`.
2. **Collapse one run of all-zero hextets** to `::` — but only **once** per
   address (otherwise it'd be ambiguous how many zero-hextets `::` stands for).

```
2001:0db8:acad:000a:0000:0000:0000:0001   full
2001:db8:acad:a:0:0:0:1                    after rule 1
2001:db8:acad:a::1                         after rule 2  ✅
```

A **prefix length** works exactly like an IPv4 mask, counted in bits:
`2001:db8:acad:a::/64` means "the first 64 bits are the network." There is no
dotted-decimal mask in IPv6 — always CIDR.

> **The documentation prefix.** `2001:db8::/32` is reserved by RFC 3849 for
> examples and labs, the IPv6 equivalent of `192.0.2.0/24`. Always lab with it so
> you never collide with a real allocation.

[↩ back to README Lab 1](README.md#lab-1--addressing-the-auto-link-local-and-the-four-address-types)

---

## Address types and scopes

IPv6 has no broadcast and no "one address per interface" rule — an interface
normally holds **several** addresses at once (a link-local *plus* one or more
global/ULA addresses *plus* multicast group memberships). The categories:

| Type              | Range        | Scope        | Notes                                            |
|-------------------|--------------|--------------|--------------------------------------------------|
| **Global unicast**| `2000::/3`   | Internet     | Routable; your `2001:db8:…` addresses            |
| **Link-local**    | `fe80::/10`  | single link  | Auto-created; on-link only; NDP & routing peers  |
| **Unique-local**  | `fc00::/7`   | site         | Private (`fd00::/8` in practice); "RFC 1918 of v6"|
| **Multicast**     | `ff00::/8`   | varies       | One-to-many; **replaces broadcast**              |
| **Loopback**      | `::1/128`    | host         | Same idea as `127.0.0.1`                          |
| **Unspecified**   | `::/128`     | —            | "I have no address yet" — the source during DAD  |
| **Anycast**       | (from unicast)| varies      | Same address on many nodes; nearest one answers  |

A few things that trip up IPv4 people:

- **Anycast addresses look identical to unicast.** They're just a unicast address
  deliberately configured on several nodes; routing delivers to the closest. The
  **subnet-router anycast** address (`prefix::`, all-zero interface ID) is one
  every router on a link automatically answers.
- **Unique-local (ULA)** is what you use for internal-only networks. Real ULAs are
  `fd` + 40 random bits + subnet, so two orgs merging rarely collide.
- **`2000::/3`** is just "the currently-allocated global unicast block." Don't
  memorize it as a hard boundary; memorize *link-local = `fe80`*, *multicast =
  `ff`*, *ULA = `fc`/`fd`*, and "everything starting `2` or `3` is globally
  routable" as your fast-recognition rules.

[↩ back to README Lab 1](README.md#lab-1--addressing-the-auto-link-local-and-the-four-address-types)

---

## Why there is no broadcast

IPv4 broadcast (`255.255.255.255`, or a subnet's `.255`) interrupts **every** host
on the segment, even ones that don't care. IPv6 deletes broadcast entirely and
does everything with **multicast** — traffic aimed at a *specific group*, so only
interested nodes process it.

A multicast address is `ff` + **4 flag bits** + **4 scope bits** + a 112-bit group
ID:

```
ff  0  2  ::1
│   │  │
│   │  └─ scope: 1=interface  2=link  5=site  e=global
│   └──── flags (0 = well-known, permanently-assigned group)
└──────── always ff = multicast
```

The **scope nibble** is why `ff02::` addresses dominate this lab: `2` = *link
scope*, i.e. "this segment only," which is exactly where NDP operates. Well-known
groups you'll meet:

| Group        | Members                         | Used by                          |
|--------------|---------------------------------|----------------------------------|
| `ff02::1`    | all nodes on the link           | Router Advertisements; "ping all"|
| `ff02::2`    | all routers on the link         | Router Solicitations             |
| `ff02::5/6`  | OSPFv3 routers / DRs            | OSPFv3 (next labs)               |
| `ff02::a`    | EIGRP routers                   | EIGRP for IPv6 (next labs)       |
| `ff02::1:2`  | all DHCPv6 relays & servers     | DHCPv6 (follow-on lab)           |
| `ff02::1:ffXX:XXXX` | one specific node's "doorbell" | NDP address resolution + DAD |

That last one is special enough to get its own section. →

[↩ back to README captures section](README.md#using-edgeshark-for-captures)

---

## Multicast and the solicited-node address

Address resolution (finding a neighbor's MAC) *could* just multicast to
`ff02::1` (all-nodes) and interrupt everyone — that would be no better than
broadcast. Instead, NDP uses the **solicited-node multicast address**, a group so
narrow that usually only the *one* target node is listening.

**How it's built:** take the low **24 bits** of the target's unicast address and
append them to the fixed prefix `ff02::1:ff00:0/104`:

```
target:            2001:db8:acad:a::1
low 24 bits:                        00:00:01
solicited-node:    ff02::1:ff00:1        (ff02::1:ff + 00:00:01)
```

**Why it works:** a node *joins* the solicited-node group for **each** of its own
unicast addresses. To resolve `…a::1`, a sender multicasts the Neighbor
Solicitation to `ff02::1:ff00:1`. Only nodes whose address ends in those same 24
bits are subscribed, so almost always just the target's NIC is woken — the switch
even filters it at layer 2, because the group maps to the MAC
**`33:33:ff:00:00:01`** (`33:33` + the low 32 bits of the multicast address). This
is IPv6's trick for getting broadcast's *function* without broadcast's *cost*.

The same solicited-node address is the destination for **DAD** — a node about to
claim `…a::1` solicits *its own* future address to see if anyone already answers.

[↩ back to README Lab 3](README.md#lab-3--ndp-replaces-arp-neighbor-solicitation--advertisement)

---

## Modified EUI-64, bit by bit

EUI-64 is the algorithm that stretches a **48-bit MAC** into a **64-bit interface
ID**, so a host can build its own interface ID with no configuration. Worked
example with MAC `aabb.cc00.0110`:

```
1) Split the MAC in half:            aa bb cc | 00 01 10
2) Insert FF FE in the middle:       aa bb cc FF FE 00 01 10
3) Flip the 7th bit of byte 1:       aa = 1010 1010
                                          └─ this bit (value 0x02, the U/L bit)
                                     flip → 1010 1000 = a8
   Result interface ID:              a8 bb cc ff fe 00 01 10
   → link-local:  fe80::a8bb:ccff:fe00:110
```

Two things to lock in:

- **`ff:fe` in the middle of an interface ID is the EUI-64 fingerprint.** See it
  and you know the address was auto-built from a MAC.
- **The flipped bit is the U/L (Universal/Local) bit.** In a normal MAC, `0` =
  globally unique (burned-in, from the vendor OUI) and `1` = locally administered.
  Modified EUI-64 **inverts** its meaning so that, in the *interface ID*, `1` now
  signals "globally unique." The practical upshot is that hand-configured
  identifiers stay short — a manually assigned `fe80::1` has the bit as `0`
  (locally administered), which is exactly what you want for a human-typed address.

> **Privacy angle (why real hosts often *don't* use EUI-64):** an EUI-64 interface
> ID embeds the MAC, so a device keeps the same host bits on every network it
> visits — trivially trackable. Modern OSes default to **random / temporary
> interface IDs** (RFC 4941 privacy extensions) or **stable-but-opaque** IDs
> (RFC 7217) instead. That's why a real laptop's SLAAC address usually has *no*
> `ff:fe`, while a Cisco router interface does. Both are valid SLAAC.

[↩ back to README Lab 2](README.md#lab-2--modified-eui-64-how-a-mac-becomes-an-interface-id)

---

## Link-local addresses and zones

Every IPv6-enabled interface **always** has a link-local address in `fe80::/64`,
created automatically the instant IPv6 comes up — before, and independent of, any
global address. It exists so a node can talk to neighbors *on the same link* with
zero configuration.

What link-locals are used for:

- All **NDP** traffic (RS/RA/NS/NA) is sourced from link-local.
- **Routing-protocol adjacencies**: OSPFv3 and EIGRP-for-IPv6 neighbors peer over
  each other's link-local, and use it as the next-hop in the routes they install.
  (This is why you'll see `fe80::` next-hops all over the routing labs.)
- A host's **default gateway** learned via SLAAC is the router's *link-local*, not
  a global address.

**The catch — zones.** Link-locals are **not routable** and, crucially, the *same*
`fe80::` address can legitimately exist on every link a router has. So a
link-local address alone is ambiguous: "send to `fe80::2`" — *on which
interface?* IPv6 resolves this with a **zone identifier**, written
`fe80::2%Ethernet0/2`. On the CLI you supply the zone by naming the **exit
interface**, which is the entire reason a static route via a link-local next-hop
*must* include an interface (see [the link-local next-hop
rule](#routing-and-the-link-local-next-hop-rule)).

[↩ back to README Lab 1](README.md#12-the-link-local-address-is-automatic)

---

## NDP: the protocol that replaced ARP

**Neighbor Discovery Protocol** (RFC 4861) rolls several IPv4 jobs — ARP, ICMP
router discovery, ICMP redirect, and parts of DHCP — into one protocol that rides
on **ICMPv6**. Five message types do the work:

| ICMPv6 type | Message                       | IPv4 analogue           | Purpose                                   |
|-------------|-------------------------------|-------------------------|-------------------------------------------|
| **133**     | Router Solicitation (RS)      | (none, really)          | Host boots: "routers, advertise now"      |
| **134**     | Router Advertisement (RA)     | ICMP router discovery   | Router: prefix, flags, gateway, MTU       |
| **135**     | Neighbor Solicitation (NS)    | ARP Request             | "who owns this address / are you there?"  |
| **136**     | Neighbor Advertisement (NA)   | ARP Reply               | "I own it, here's my link-layer address"  |
| **137**     | Redirect                      | ICMP redirect           | "use a better first-hop router for this"  |

**Address resolution** (the ARP replacement) in one line: a node multicasts an
**NS** to the target's [solicited-node
address](#multicast-and-the-solicited-node-address) carrying its own link-layer
address; the target unicasts an **NA** back with *its* link-layer address. Both
sides now have each other in the [neighbor cache](#the-neighbor-cache-and-nud).

NS/NA carry **options** — the *Source/Target Link-Layer Address* option is where
the actual MAC travels, and RAs add *Prefix Information*, *MTU*, and (increasingly)
*RDNSS* (recursive DNS server) options. Expanding those options in Wireshark is
where the real learning is.

[↩ back to README Lab 3](README.md#lab-3--ndp-replaces-arp-neighbor-solicitation--advertisement)

---

## The neighbor cache and NUD

`show ipv6 neighbors` is IPv6's ARP table, but it tracks more than "IP → MAC": it
runs **Neighbor Unreachability Detection (NUD)**, a small state machine that
actively notices when a neighbor goes away. The states:

| State        | Meaning                                                              |
|--------------|---------------------------------------------------------------------|
| `INCMP`      | **Incomplete** — NS sent, still waiting for the first NA            |
| `REACH`      | **Reachable** — confirmed live within the reachable-time window (~30s) |
| `STALE`      | Reachable-time expired; entry still *usable* but unconfirmed        |
| `DELAY`      | A packet was just sent to a STALE neighbor; wait briefly before probing |
| `PROBE`      | Actively sending unicast NS probes to re-confirm                    |

The lifecycle: a fresh resolution lands in `REACH`; after the timer it drifts to
`STALE`; sending traffic to a stale entry bumps it to `DELAY` then `PROBE`, where
targeted NS probes either restore `REACH` or evict the entry. This is why IPv6
detects a dead next-hop faster and more cleanly than IPv4's passive ARP aging —
NUD is *designed* to notice.

[↩ back to README Lab 3.1](README.md#31-resolve-a-neighbor-the-arp-replacement)

---

## Duplicate Address Detection (DAD)

Before a node uses **any** unicast address — link-local or global, statically typed
or SLAAC-derived — it must prove no one else already has it. That check is DAD, and
it runs every single time an address is assigned.

Mechanics, exactly as you'll see it on the wire:

1. The new address enters **tentative** state (unusable).
2. The node sends a **Neighbor Solicitation** with:
   - **source = `::`** (the unspecified address — it doesn't yet own the address
     it's asking about), and
   - **target = the tentative address**, sent to that address's **solicited-node
     multicast**.
3. **Silence** → nobody else has it → the address leaves tentative and becomes
   valid. **An NA comes back** → a duplicate exists → the address goes to
   `DUPLICATE` state and is *not* used (`%IPV6-4-DUPLICATE_ADDR` is logged).

The `::`-sourced NS is the signature to look for — it's how you tell a DAD probe
apart from ordinary address resolution in a capture. On a link with lots of nodes,
DAD is also why bringing an interface up produces a little burst of NS traffic:
one probe per address being claimed.

[↩ back to README Lab 1.3](README.md#13-add-a-global-unicast-address--watch-dad)

---

## Router Advertisements and SLAAC

**Stateless Address Autoconfiguration** is how a host gets a working global address
and default gateway with *no server and no configuration* — the headline IPv6
feature. It's driven entirely by **Router Advertisements**.

**The exchange:** a booting host multicasts a **Router Solicitation** to
`ff02::2` (all-routers) rather than waiting for the next periodic RA. Each router
replies (or periodically emits) a **Router Advertisement** to `ff02::1`
(all-nodes) from its link-local. The host then:

1. reads a **Prefix Information** option whose **A (Autonomous) flag** is set,
2. appends its own [interface ID](#modified-eui-64-bit-by-bit) (EUI-64 or random)
   to that /64 prefix to form a global address,
3. runs [DAD](#duplicate-address-detection-dad) on the result, and
4. installs a **default route** toward the RA's source link-local.

**Fields inside an RA that matter** (expand one in Wireshark):

| RA field                     | What it controls                                             |
|------------------------------|-------------------------------------------------------------|
| **M flag** (Managed)         | Get your **address** from stateful DHCPv6 → see next section |
| **O flag** (Other)           | Get **other config** (DNS…) from DHCPv6                      |
| **Router Lifetime**          | Seconds this router is a valid default gateway (**0 = not a gateway**) |
| **Prefix: A flag**           | Autonomous — *this prefix is usable for SLAAC*               |
| **Prefix: L flag**           | On-link — hosts may treat the prefix as directly reachable   |
| **Valid / Preferred lifetime** | How long the SLAAC address stays valid / preferred          |
| **Cur Hop Limit, MTU, RDNSS**| Default TTL, link MTU, DNS servers pushed to hosts          |

A prefix with the **A flag clear** can't be used for SLAAC — a common
"why didn't my host get an address?" cause. And a router advertising **Router
Lifetime 0** provides prefixes but won't be chosen as a default gateway.

[↩ back to README Lab 4](README.md#lab-4--slaac-a-host-builds-its-own-address-from-an-ra)

---

## SLAAC vs DHCPv6: the M and O flags

This is the single most exam-tested IPv6 decision, and it lives in **two bits** of
the Router Advertisement. The host reads them and decides where to get its config:

| M | O | Host behaviour                                                                 |
|---|---|-------------------------------------------------------------------------------|
| 0 | 0 | **Pure SLAAC.** Address from the prefix; no DHCPv6 at all.                     |
| 0 | 1 | **SLAAC + stateless DHCPv6.** Address via SLAAC; *other* info (DNS, domain) from DHCPv6. |
| 1 | 0 | **Stateful DHCPv6.** Address (and options) handed out by a DHCPv6 server, which tracks leases. |
| 1 | 1 | **Stateful DHCPv6** for both address and options (M implies you're going to the server anyway). |

Mental model: **A-flag** (on the prefix) says *SLAAC is allowed*; **M-flag** says
*use a DHCPv6 server for the address*; **O-flag** says *use DHCPv6 just for the
extras*. They're not mutually exclusive — the classic enterprise setup is often
**M=0, O=1**: hosts SLAAC their addresses but learn DNS from a stateless DHCPv6
server (because plain SLAAC historically couldn't hand out DNS until the RDNSS
option existed).

Why it matters for troubleshooting: a host with no address on an IPv6 network is
usually one of — A-flag clear on the prefix, M-flag set but no DHCPv6 server
reachable, or no router sending RAs at all. Knowing which flag drives which
behaviour turns that into a two-minute diagnosis.

> The actual DHCPv6 message exchange (Solicit / Advertise / Request / Reply, plus
> relay) is the subject of the **IPv6 services** follow-on lab; this lab stops at
> *seeing the flags that decide whether DHCPv6 is even used.*

[↩ back to README Lab 4.2](README.md#42-make-r3-a-slaac-client-and-watch-it-autoconfigure)

---

## Routing and the link-local next-hop rule

IPv6 routing works like IPv4 — longest-prefix match, a default route written as
**`::/0`**, connected/static/dynamic routes — with one twist that catches
everyone.

Next-hops in IPv6 are very often **link-local** (that's what RAs and routing
protocols hand you). But a link-local address is only meaningful *on a specific
link* — the same `fe80::2` can exist on several of a router's interfaces — so it
carries no information about which way to send the packet. Therefore:

> **A static route whose next-hop is a link-local address MUST also name the exit
> interface.** A route via a *global* next-hop can stand alone.

```
! link-local next-hop → interface REQUIRED:
ipv6 route 2001:db8:acad:2::/64 Ethernet0/2 fe80::a8bb:ccff:fe00:2200
! global next-hop → interface optional:
ipv6 route 2001:db8:acad:2::/64 2001:db8:acad:12::2
```

Symptoms of getting it wrong: `% Invalid next hop address (it must be a
link-local...` or a route that simply never installs. Nine times out of ten the
fix is "add the exit interface." See [link-local addresses and
zones](#link-local-addresses-and-zones) for *why* the interface is the zone the
address needs.

[↩ back to README Lab 5.2](README.md#52-global-next-hop-vs-link-local-next-hop)

---

## The /128 local route

Configure `2001:db8:acad:a::1/64` on an interface and `show ipv6 route` shows
**two** entries, not one:

```
C   2001:db8:acad:a::/64   [0/0]  via Ethernet0/1, directly connected
L   2001:db8:acad:a::1/128 [0/0]  via Ethernet0/1, receive
```

- The **`C` (connected) /64** is the subnet — packets to anyone on that link.
- The **`L` (local) /128** is *this router's own address*, marked **receive**: it
  tells the forwarding engine "traffic for exactly this address terminates here,
  hand it to the CPU." IPv4 has the same concept internally; IPv6 just makes it
  visible in the routing table. Seeing an `L …/128` for every address you assign
  is normal and correct — not a duplicate or a bug.

[↩ back to README quick extras](README.md#quick-extras-short-high-value)

---

## IPv4 to IPv6 quick translation

A cheat-sheet for carrying IPv4 instincts across without importing IPv4 habits
that no longer apply:

| IPv4 thing                 | IPv6 equivalent                          | Watch out for                                   |
|----------------------------|-------------------------------------------|-------------------------------------------------|
| ARP                        | NDP **NS/NA** (ICMPv6 135/136)            | Uses multicast, not broadcast                   |
| `show ip arp`              | `show ipv6 neighbors`                     | Has NUD states, not just aging                  |
| Broadcast / `.255`         | **Multicast** (`ff02::1` = all-nodes)     | No broadcast exists at all                      |
| DHCP (typical)             | **SLAAC** (RA-driven), *optionally* DHCPv6| M/O flags decide if a server is even used       |
| Default route `0.0.0.0/0`  | **`::/0`**                                | —                                               |
| `127.0.0.1`                | **`::1`**                                 | —                                               |
| RFC 1918 private           | **ULA** `fd00::/8`                         | Randomize the global ID                         |
| Subnet mask `255.255.255.0`| **prefix length** `/64`                    | Always CIDR; /64 is the LAN default             |
| One IP per interface       | **many** per interface                    | Link-local + global(s) + multicast groups       |
| Router discovery = config  | **RA** tells hosts the gateway            | Kill RAs and hosts have no gateway              |

[↩ back to README top](README.md#ccnp-ipv6-fundamentals-lab-cisco-iol--edgeshark)
