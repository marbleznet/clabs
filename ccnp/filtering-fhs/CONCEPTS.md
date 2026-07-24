# Filtering & First-Hop Security — In-Depth Concepts

The companion reference to the [README](README.md) lab guide. The README is the
*hands-on* path (configure this, spoof that, watch it drop); this file is the *why*
behind each step. Each 📖 **Deep dive** link in the README jumps to a section here.

**Contents**
- [How ACLs are processed](#how-acls-are-processed)
- [Standard vs extended, and where to place them](#standard-vs-extended-and-where-to-place-them)
- [Time-based and advanced ACLs](#time-based-and-advanced-acls)
- [IPv6 ACLs: what's different](#ipv6-acls-whats-different)
- [uRPF: strict, loose, and anti-spoofing](#urpf-strict-loose-and-anti-spoofing)
- [The IPv6 first-hop threat model](#the-ipv6-first-hop-threat-model)
- [The binding table and device-tracking](#the-binding-table-and-device-tracking)
- [The First-Hop Security feature set](#the-first-hop-security-feature-set)

---

## How ACLs are processed

An ACL is an **ordered list** of permit/deny rules (ACEs). The router walks it
**top-down and stops at the first match** — order is everything, and a more specific
rule placed *after* a broader one never runs. Three rules that cause most ACL bugs:

- **First match wins, then processing stops.** A `permit` or `deny` is final for that
  packet; later lines aren't consulted.
- **Implicit `deny any` at the end.** Every ACL has an invisible final deny, so an ACL
  that's *all permits for specific things* silently drops everything else. If you want
  a default-allow, you must end with an explicit `permit`.
- **An empty/unapplied ACL does nothing** — an ACL only filters once it's attached to
  an interface with a **direction** (`in`/`out`), and you get **one ACL per protocol,
  per direction, per interface**.

**Wildcard masks** (IPv4) are the inverse of a subnet mask: a `0` bit means "must
match," a `1` bit means "don't care." `10.0.0.0 0.0.0.255` = "match 10.0.0.x";
`0.0.0.0 255.255.255.255` = `any`; `host 10.0.0.66` = `10.0.0.66 0.0.0.0`. Reading
wildcards wrong is a classic exam trap — they are **not** subnet masks.

**Named ACLs** (`ip access-list extended NAME`) are preferred over numbered: they take
**sequence numbers** so you can insert/delete individual lines without rebuilding the
whole list.

[↩ back to README Lab 1](README.md#lab-1--standard-ipv4-acls-filter-by-source)

---

## Standard vs extended, and where to place them

| | **Standard** | **Extended** |
|---|---|---|
| Matches | **source IP only** | source, destination, protocol, port(s), flags |
| Numbered range | 1–99, 1300–1999 | 100–199, 2000–2699 |
| Granularity | coarse | precise (the full 5-tuple) |
| **Placement rule** | **close to the destination** | **close to the source** |

The **placement rules follow from what each can see**:

- A **standard** ACL knows only the *source*. If you place it near the source, it
  would block that host from **every** destination — too blunt. So you place it as
  **close to the destination as possible**, filtering only the traffic actually headed
  there.
- An **extended** ACL knows source *and* destination, so it can be surgical. Place it
  **close to the source** to drop unwanted traffic **before** it crosses the network
  and wastes bandwidth and router CPU.

Because ACLs are **stateless**, return traffic is a constant gotcha: permitting
`LAN → server:80` does **not** automatically permit the server's replies back in. The
classic answers are the **`established`** keyword (permit TCP segments with ACK/RST
set — i.e. replies to sessions the inside started), **reflexive ACLs** (auto-generate
temporary reverse entries), or a real stateful firewall/zone-based policy.

[↩ back to README Lab 2](README.md#lab-2--extended-ipv4-acls-source-destination-protocol-port)

---

## Time-based and advanced ACLs

A **time-based** ACL is an ordinary ACE with a **`time-range`** attached, so the rule
is only active during a schedule:

```
time-range BUSINESS-HOURS
 periodic weekdays 9:00 to 17:00      ← recurring window
 ! absolute start 08:00 1 Jan 2026 end 17:00 31 Dec 2026   ← one-off window
...
 permit tcp … eq 22 time-range BUSINESS-HOURS
```

The critical dependency: **the router's clock.** A time-based ACL is only as accurate
as the system time, so an unsynced or wrong clock is *the* reason "my schedule isn't
working." Always drive it from **NTP** (or at least `clock set` + timezone) — and
remember the check uses the **router's** local time, not the client's.

Two other ACL types worth recognizing (ENARSI mentions the family):

- **Reflexive ACLs** (`permit … reflect NAME` + `evaluate NAME`) — approximate
  stateful filtering by dynamically mirroring outbound sessions into a temporary
  inbound permit. The pre-firewall way to allow return traffic safely.
- **Dynamic ("lock-and-key") ACLs** — open a temporary hole *after* a user
  authenticates (Telnet/SSH to the router), then close it.

[↩ back to README Lab 3](README.md#lab-3--time-based-acls)

---

## IPv6 ACLs: what's different

IPv6 ACLs do the same job as extended IPv4 ACLs but with several syntax and behaviour
differences that each have a matching gotcha:

- **Named only** — there are no numbered IPv6 ACLs; it's always `ipv6 access-list
  NAME`.
- **Applied with `ipv6 traffic-filter … in|out`**, *not* `ip access-group`. Using the
  wrong apply command is a common "my ACL isn't taking effect" cause.
- **Prefix length, not wildcard masks** — `2001:db8:a::/64` or `host <addr>`. No
  inverse masks exist in IPv6.
- **Implicit ND permits.** Because IPv6 literally cannot function without Neighbor
  Discovery, every IPv6 ACL has **two implicit permits — `permit icmp any any nd-na`
  and `permit icmp any any nd-ns`** — placed *just before* the final implicit `deny
  ipv6 any any`. This is the big one: **the instant you add your own explicit `deny
  ipv6 any any`** (or otherwise rely on an explicit terminal deny), **those implicit
  ND permits no longer save you**, and you break ND — which breaks *all* IPv6 on the
  segment (no address resolution, no gateway). The fix is to **explicitly permit
  `nd-ns`/`nd-na`** near the top of any restrictive IPv6 ACL, exactly as README Lab 4
  does.

Everything else — top-down first-match, implicit final deny, per-interface/direction —
works like IPv4.

[↩ back to README Lab 4](README.md#lab-4--ipv6-traffic-filter)

---

## uRPF: strict, loose, and anti-spoofing

**Unicast Reverse Path Forwarding** is anti-spoofing without an ACL to maintain. For
every packet, the router does a **reverse lookup on the *source* address** in the FIB
and asks whether that source could legitimately have arrived where it did. If not, the
packet is dropped. It's Cisco's implementation of **BCP 38 / RFC 2827 ingress
filtering** — the thing that, if everyone deployed it, would kill source-spoofed DDoS.

Two modes:

| Mode | Command | Test | Use where |
|------|---------|------|-----------|
| **Strict** | `… reachable-via rx` | source must be reachable **via the interface it arrived on** | single-homed access/edge |
| **Loose** | `… reachable-via any` | source must exist **somewhere** in the FIB (any interface) | multihomed / asymmetric edges |

- **Strict** is the tightest: it drops a packet if the best return path to its source
  doesn't point back out the receiving interface. But that's exactly what happens with
  **asymmetric routing** (traffic in one interface, return out another), so strict
  uRPF will **false-drop** on multihomed links — hence loose mode there.
- **Loose** only drops sources that are **completely unrouteable** (bogons, unassigned
  space) — weaker, but safe with asymmetry.
- **`allow-default`** additionally accepts sources that only match a default route
  (otherwise a source reachable *only* via `0.0.0.0/0` fails the check).

uRPF requires **CEF** (it reuses the FIB), applies to both IPv4 and IPv6, and is far
cheaper to run than an equivalent anti-spoof ACL you'd have to hand-maintain.

[↩ back to README Lab 5](README.md#lab-5--urpf-drop-spoofed-sources)

---

## The IPv6 first-hop threat model

IPv6's autoconfiguration is wonderfully plug-and-play and **completely
unauthenticated**. Everything a host needs to get on the network — its gateway
(Router Advertisements), its address (SLAAC or DHCPv6), and address resolution
(Neighbor Discovery) — arrives as **unsigned multicast** that any device on the LAN can
send. That's the attack surface:

- **Rogue RA** — the headline threat. A host sending Router Advertisements (maliciously
  with `fake_router6`/`radvd`, or **accidentally** from a mis-configured Windows box
  with Internet Connection Sharing) makes every host on the segment install it as a
  default gateway → **man-in-the-middle** or **denial of service**. It's common enough
  that RA Guard exists specifically for it.
- **Rogue DHCPv6 server** — hands out bogus addresses/DNS, redirecting or MITM-ing
  clients.
- **NDP spoofing / cache poisoning** — the IPv6 equivalent of ARP poisoning: forge
  Neighbor Advertisements to claim someone else's address.
- **Address / source spoofing** and **NDP cache-exhaustion DoS** (scanning a huge /64
  to flood the neighbor table).

The protocol-level fix, **SEND (Secure Neighbor Discovery)** with cryptographic CGAs,
exists but is **almost never deployed** (poor support). So in practice you defend the
**first hop — the access switch** — with the FHS feature set, all built on a
**[binding table](#the-binding-table-and-device-tracking)** of who legitimately owns
what.

[↩ back to README Lab 6](README.md#lab-6--ipv6-first-hop-security-on-sw1)

---

## The binding table and device-tracking

Every FHS feature needs one source of truth: **which IPv6 address belongs to which
MAC, on which port/VLAN.** The switch builds that **binding table** by **snooping** the
control traffic that hands out addresses — **Neighbor Discovery** (NS/NA, including the
DAD probes a host sends when it claims an address) and **DHCPv6** lease messages. A
legitimate host that configures `2001:db8:a::10` announces itself via ND; the switch
records `2001:db8:a::10 ↔ its MAC ↔ Eth0/2` as a **verified binding**.

Modern IOS unifies this under the **device-tracking** framework (the IP Device
Tracking / IPDT engine, covering IPv4 and IPv6); older code called the IPv6 half **IPv6
snooping** (`ipv6 snooping` / `ipv6 neighbor binding`). Either way:

```
show device-tracking database      (or: show ipv6 neighbors binding)
```

is the table, and it's what **RA guard, DHCP guard, ND inspection, and source guard**
all consult. Get the binding table working first (README Lab 6.1) — if it's empty or
wrong, every feature layered on it misbehaves. Tune its port roles carefully: an
**uplink** toward routers/servers is trusted; **access** ports toward hosts are where
you enforce.

[↩ back to README Lab 6.1](README.md#61-the-binding-table-the-foundation--this-one-you-can-watch-populate)

---

## The First-Hop Security feature set

FHS is a **layered** defense on the access switch — each feature blocks one class of
[first-hop attack](#the-ipv6-first-hop-threat-model), and together they lock the LAN:

| Feature | Blocks | How |
|---------|--------|-----|
| **RA Guard** | rogue Router Advertisements | drops RAs on ports whose `device-role` is `host`; only `router` ports may send them |
| **DHCPv6 Guard** | rogue DHCPv6 **servers** | drops DHCPv6 server messages (Advertise/Reply) on `client`-role ports |
| **ND Inspection** | ND/NA spoofing | validates NS/NA against the [binding table](#the-binding-table-and-device-tracking); drops mismatches |
| **IPv6 Snooping / device-tracking** | (foundation) | *builds* the binding table by snooping ND + DHCPv6 |
| **IPv6 Source Guard** | source-address spoofing | drops **data** traffic whose source isn't a bound address on that port |

The mental model is **role-based port trust**: you tell the switch what each port
*should* be (a host that receives RAs and requests DHCP, vs. an uplink that provides
them), and FHS enforces that a host port can never *act* like a router or server. RA
Guard + DHCP Guard stop the **control-plane** hijacks; ND Inspection + Source Guard
stop the **spoofing**; the binding table underpins them all.

Know the **IPv4 parallels**, because ENARSI security is really "the same defenses in
both families": **DHCP Snooping** (build a trusted binding table from DHCP) →
**Dynamic ARP Inspection** (validate ARP against it, = ND inspection) → **IP Source
Guard** (drop spoofed source IPs, = IPv6 source guard). Same three-layer pattern,
different address family.

[↩ back to README Lab 6.4](README.md#64-nd-inspection--ipv6-source-guard)
