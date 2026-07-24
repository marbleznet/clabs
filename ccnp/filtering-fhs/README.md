# CCNP Filtering & First-Hop Security Lab (Cisco IOL + Edgeshark)

A lab for **data-plane filtering and IPv6 first-hop security** — **IPv4 ACLs**
(standard, extended, time-based), the **IPv6 traffic filter**, **uRPF**
anti-spoofing, and the **IPv6 First-Hop Security** feature set (RA guard, DHCP
guard, binding table, ND inspection, source guard). R1 boots with **only interfaces
and IP addresses**: no ACLs, no uRPF, no FHS. You build every filter by hand and use
the Linux hosts + **Edgeshark captures** to see what's permitted, dropped, and
spoofed.

This lab covers **ENARSI 3.2** (router security — ACLs, IPv6 filter, uRPF) and
**3.4** (IPv6 first-hop security). It reuses the helper-container pattern from the
device-security lab: a legit client, an **attacker**, and an outside host.

By the end you will have configured and observed:

- **IPv4 ACLs** — standard, extended, and **time-based**, and *where* each belongs
- The **IPv6 traffic filter** and the ND-permit gotcha that breaks IPv6 if you miss it
- **uRPF** (strict vs loose) dropping a **spoofed source** in real time
- **IPv6 First-Hop Security** — the **binding table**, **RA guard**, **DHCPv6
  guard**, **ND inspection**, and **source guard**

> 📖 **This lab has a companion:** [`CONCEPTS.md`](CONCEPTS.md) is the in-depth
> reference behind every step (ACL processing, placement, uRPF modes, the IPv6
> first-hop threat model, FHS). Each lab opens with a **Deep dive** callout.

---

## Topology

```
   [pc1  10.0.0.10]        [rogue 10.0.0.66]          LAN 10.0.0.0/24
     legit client            attacker                 2001:db8:a::/64
          \                    /
           └──── SW1 (L2 · IPv6 First-Hop Security) ────┐  e0/1
                                                        R1  (gateway)
                                    ACLs (v4/v6) · uRPF · IPv6 traffic filter
                                                        │  e0/2
                                          external 203.0.113.0/24 · 2001:db8:e::/64
                                                        │
                                             [ext  203.0.113.100]  outside host
```

| Node  | Address(es)                          | Role                                             |
|-------|--------------------------------------|--------------------------------------------------|
| R1    | LAN .1 / ext .1 (+ v6 ::1)           | Gateway — ACLs, uRPF, IPv6 filter                |
| SW1   | —                                    | Access switch — IPv6 First-Hop Security          |
| pc1   | 10.0.0.10 / 2001:db8:a::10           | Legit LAN client                                 |
| rogue | 10.0.0.66 / 2001:db8:a::66           | Attacker — source spoofing, rogue RA/DHCP        |
| ext   | 203.0.113.100 / 2001:db8:e::100      | Outside host / server                            |

Interface mapping: containerlab `ethX` → IOL `Ethernet0/X`; `E0/0` is management.
Run host commands with `docker exec -it clab-filter-fhs-lab-pc1 bash`.

---

## Deploy

```bash
cd /home/lab/clabs/ccnp/filtering-fhs
sudo containerlab deploy -t filtering-fhs.clab.yml
```
Destroy with `sudo containerlab destroy -t filtering-fhs.clab.yml --cleanup`.

> ⚠️ **Scope note.** **Labs 1–5 (ACLs, IPv6 filter, uRPF) are pure router IOS and
> fully hands-on.** **Lab 6 (FHS)** runs on SW1's IOL **L2 image** — the
> device-tracking/FHS command support varies by image, so the **binding-table**
> portion (populated by normal host ND) is the reliable demo, while the rogue-RA /
> rogue-DHCP *attacks* are **described** (they need `thc-ipv6`/`radvd` on the rogue).
> The defensive config shown is exam-correct regardless of what the image accepts.

### Sanity check

```
r1# show ip route                 ! connected 10.0.0.0/24 and 203.0.113.0/24
pc1$ ping 203.0.113.100           ! LAN -> ext works (no filters yet)
pc1$ ping6 2001:db8:e::100        ! ...and over IPv6
```

---

## Using Edgeshark for captures

Capture on `clab-filter-fhs-lab-r1` `Ethernet0/1` (LAN) or `Ethernet0/2` (external).

| Filter                    | Shows                                              |
|---------------------------|----------------------------------------------------|
| `icmp` / `icmpv6`         | ping tests through the ACLs                        |
| `icmpv6.type == 134`      | **Router Advertisements** (the RA-guard target)    |
| `dhcpv6`                  | DHCPv6 (the DHCP-guard target)                     |
| `tcp.port == 80`          | the HTTP flows your extended ACL permits/denies    |
| `icmpv6.type == 135/136`  | ND (NS/NA) — what populates the binding table      |

---

# Lab 1 — Standard IPv4 ACLs (filter by source)

**Goal:** the simplest filter — a **standard** ACL matches **source address only**.
Block the rogue host from the outside world, and learn the placement rule.

> 📖 **Deep dive:** [How ACLs are processed](CONCEPTS.md#how-acls-are-processed)
> · [Standard vs extended, and where to place them](CONCEPTS.md#standard-vs-extended-and-where-to-place-them)

```
r1(config)# ip access-list standard BLOCK-ROGUE
r1(config-std-nacl)#  deny   host 10.0.0.66
r1(config-std-nacl)#  permit any
r1(config)# interface Ethernet0/2
r1(config-if)#  ip access-group BLOCK-ROGUE out
```

> **📸 CAPTURE — `r1` `Ethernet0/2`**, filter `icmp`. From rogue: `ping
> 203.0.113.100` → blocked (no packets leave e0/2). From pc1 → permitted.

```
r1# show access-lists BLOCK-ROGUE     ! per-line match counters climb on the deny
r1# show ip interface Ethernet0/2 | include access list
```

> **Placement:** a standard ACL only knows the *source*, so placing it too early
> would block that source from **every** destination. Rule of thumb: **standard
> ACLs go close to the destination.** Here, outbound on e0/2 blocks the rogue from
> the outside while leaving its LAN-local reachability intact.

> **The implicit `deny any`:** every ACL ends with an invisible `deny any`. Your
> `permit any` is what keeps everything else flowing — leave it out and you'd
> black-hole the whole LAN.

---

# Lab 2 — Extended IPv4 ACLs (source, destination, protocol, port)

**Goal:** an **extended** ACL matches the full 5-tuple. Write a real access policy —
LAN may reach the server on HTTP/HTTPS and ping, nothing else — and place it close
to the source.

> 📖 **Deep dive:** [Standard vs extended, and where to place them](CONCEPTS.md#standard-vs-extended-and-where-to-place-them)

```
r1(config)# ip access-list extended LAN-POLICY
r1(config-ext-nacl)#  permit tcp 10.0.0.0 0.0.0.255 host 203.0.113.100 eq 80
r1(config-ext-nacl)#  permit tcp 10.0.0.0 0.0.0.255 host 203.0.113.100 eq 443
r1(config-ext-nacl)#  permit icmp 10.0.0.0 0.0.0.255 any echo-reply
r1(config-ext-nacl)#  permit icmp 10.0.0.0 0.0.0.255 any echo
r1(config-ext-nacl)#  deny   ip any any log
r1(config)# interface Ethernet0/1
r1(config-if)#  ip access-group LAN-POLICY in
```

> **📸 CAPTURE — `r1` `Ethernet0/2`**, filter `tcp.port==80 || tcp.port==8080`.
> From pc1: `curl http://203.0.113.100` (permitted) then `curl
> http://203.0.113.100:8080` (denied — never reaches e0/2). The `deny … log`
> generates a syslog line for each blocked flow.

```
r1# show access-lists LAN-POLICY      ! match counts per line
r1# show logging | include LAN-POLICY  ! the denied 8080 attempt logged
```

> **Placement:** an extended ACL knows source **and** destination, so put it **close
> to the source** (inbound on e0/1) to drop unwanted traffic *before* it crosses the
> router and consumes resources. That's the mirror image of the standard-ACL rule.

> **Return traffic:** this is a stateless filter. `curl` works because we permitted
> the server's `echo-reply`/ports explicitly; in the real world you'd use
> `permit tcp … established` (or a stateful firewall / reflexive ACL) to allow
> replies without opening the ports inbound.

---

# Lab 3 — Time-based ACLs

**Goal:** make a rule active only during a schedule with a **time-range**.

> 📖 **Deep dive:** [Time-based and advanced ACLs](CONCEPTS.md#time-based-and-advanced-acls)

```
r1(config)# time-range BUSINESS-HOURS
r1(config-time-range)#  periodic weekdays 9:00 to 17:00
r1(config)# ip access-list extended TIMED
r1(config-ext-nacl)#  permit tcp 10.0.0.0 0.0.0.255 host 203.0.113.100 eq 22 time-range BUSINESS-HOURS
r1(config-ext-nacl)#  permit ip any any
r1(config)# interface Ethernet0/1
r1(config-if)#  ip access-group TIMED in
```

```
r1# show time-range                   ! active / inactive right now
r1# show access-lists TIMED           ! the timed line shows "(active)" or "(inactive)"
```

> **The clock dependency:** a time-based ACL is only as right as the router's clock —
> set it (`clock set`) or, better, sync **NTP**. A wrong or unsynced clock is the
> classic "my time-based rule isn't working" cause. Use `periodic` for recurring
> windows, `absolute` for one-off start/end dates.

Set the clock outside the window (`clock set 20:00:00 …`) and confirm the SSH line
goes **inactive** while `permit ip any any` still passes everything else.

---

# Lab 4 — IPv6 traffic filter

**Goal:** filter IPv6 — same idea as an extended ACL, but with the **IPv6 gotchas**:
a different apply command, prefix-length instead of wildcards, and **implicit ND
permits** you can accidentally destroy.

> 📖 **Deep dive:** [IPv6 ACLs: what's different](CONCEPTS.md#ipv6-acls-whats-different)

```
r1(config)# ipv6 access-list V6-FILTER
r1(config-ipv6-acl)#  permit icmp any any nd-ns          ! neighbor solicitation
r1(config-ipv6-acl)#  permit icmp any any nd-na          ! neighbor advertisement
r1(config-ipv6-acl)#  deny   ipv6 host 2001:DB8:A::66 any
r1(config-ipv6-acl)#  permit ipv6 any any
r1(config)# interface Ethernet0/1
r1(config-if)#  ipv6 traffic-filter V6-FILTER in
```

> **📸 CAPTURE — `r1` `Ethernet0/1`**, filter `icmpv6`. From rogue: `ping6
> 2001:db8:e::100` → denied. From pc1 → permitted.

The three IPv6-specific things to burn in:

- **Apply with `ipv6 traffic-filter`**, *not* `ip access-group`.
- **No wildcard masks** — IPv6 ACLs use **prefix/length** (`2001:db8:a::/64`) or
  `host`.
- **The implicit ND permits.** An IPv6 ACL has an implicit `permit icmp any any
  nd-na` / `nd-ns` **before** the final implicit `deny` — because IPv6 can't function
  without Neighbor Discovery. **The moment you add your own explicit `deny ipv6 any
  any`, those implicit permits are gone** and you break ND (and thus all IPv6) on the
  segment. That's why we permit `nd-ns`/`nd-na` explicitly above. This is *the* IPv6
  ACL exam trap.

```
r1# show ipv6 access-list V6-FILTER
```

---

# Lab 5 — uRPF (drop spoofed sources)

**Goal:** **Unicast Reverse Path Forwarding** drops packets whose **source address**
couldn't legitimately have arrived on that interface — automatic anti-spoofing
without an ACL to maintain.

> 📖 **Deep dive:** [uRPF: strict, loose, and anti-spoofing](CONCEPTS.md#urpf-strict-loose-and-anti-spoofing)

### 5.1 Baseline: spoof a source (no uRPF yet)

> **📸 CAPTURE — `r1` `Ethernet0/2`**, filter `icmp`. From rogue, forge a source
> that doesn't belong on the LAN:
> `nping --icmp --source-ip 172.16.99.99 203.0.113.100 -c 3`
> With no uRPF, R1 happily forwards the spoofed packet out e0/2.

### 5.2 Enable strict uRPF on the LAN interface

```
r1(config)# interface Ethernet0/1
r1(config-if)#  ip verify unicast source reachable-via rx
r1(config-if)#  ipv6 verify unicast source reachable-via rx
```

Re-run the spoof. R1 checks: *"is `172.16.99.99` reachable back out the interface it
arrived on (e0/1)?"* No — so the packet is **dropped**.

```
r1# show ip interface Ethernet0/1 | include verify|drop|RPF
r1# show ip traffic | include RPF        ! unicast RPF drop counter climbing
r1# show cef drop                         ! RPF drops
```

Legit traffic from pc1 (`10.0.0.10`, which *is* reachable via e0/1) passes untouched.

> **Strict (`rx`) vs loose (`any`):** **strict** requires the source be reachable via
> **the receiving interface** — the tightest anti-spoof, but it breaks **asymmetric
> routing** (multihomed edges where the return path differs). **Loose** (`reachable-via
> any`) only requires the source to exist **somewhere** in the FIB — survives
> asymmetry but only stops sources that are completely unrouteable (e.g. bogons).
> Strict on single-homed access edges; loose on multihomed. Add `allow-default` to
> also accept sources only matched by a default route.

---

# Lab 6 — IPv6 First-Hop Security (on SW1)

**Goal:** IPv6's plug-and-play discovery (SLAAC, DHCPv6, ND) is trivially abused on a
LAN — a **rogue RA** can hijack every host's gateway, a **rogue DHCPv6** server can
hand out bogus config, and a host can **spoof** another's address. FHS on the access
switch stops all of it. It's built on a **binding table**.

> 📖 **Deep dive:** [The IPv6 first-hop threat model](CONCEPTS.md#the-ipv6-first-hop-threat-model)
> · [The binding table and device-tracking](CONCEPTS.md#the-binding-table-and-device-tracking)
> · [The First-Hop Security feature set](CONCEPTS.md#the-first-hop-security-feature-set)

*(All config in this lab is on **SW1**. Log in: `docker exec -it
clab-filter-fhs-lab-sw1 telnet localhost`.)*

### 6.1 The binding table (the foundation) — this one you can watch populate

The switch **snoops ND and DHCPv6** to learn which **IPv6 address ↔ MAC ↔ port**
bindings are legitimate. Enable device-tracking on the host ports:

```
sw1(config)# device-tracking policy DT-HOSTS
sw1(config-device-tracking)#  tracking enable
sw1(config)# interface range Ethernet0/2 - 3
sw1(config-if-range)#  device-tracking attach-policy DT-HOSTS
```

Generate ND from the hosts (`pc1$ ping6 2001:db8:a::1`), then:

```
sw1# show device-tracking database        ! pc1/rogue addresses, MACs, and ports learned
```
*(On images without `device-tracking`, try the legacy `ipv6 snooping` / `show ipv6
neighbors binding`.)* Every other FHS feature validates against **this** table.

### 6.2 RA Guard — block rogue Router Advertisements

Only the port facing **R1** should ever carry RAs. Mark host ports as `host` (RAs
denied) and the uplink as `router` (RAs allowed):

```
sw1(config)# ipv6 nd raguard policy HOST
sw1(config-nd-raguard)#  device-role host
sw1(config)# ipv6 nd raguard policy ROUTER
sw1(config-nd-raguard)#  device-role router
sw1(config)# interface range Ethernet0/2 - 3
sw1(config-if-range)#  ipv6 nd raguard attach-policy HOST
sw1(config)# interface Ethernet0/1
sw1(config-if)#  ipv6 nd raguard attach-policy ROUTER
```

> **The attack it stops:** from rogue, `fake_router6 eth1 2001:db8:bad::/64`
> (thc-ipv6) or a `radvd` advertising a bogus prefix would make every host install
> the rogue as its default gateway (man-in-the-middle). With RA guard, SW1 **drops
> RAs arriving on host ports** — capture `icmpv6.type==134` on pc1's port and confirm
> the rogue's RA never arrives, while R1's does.

### 6.3 DHCPv6 Guard — block rogue DHCPv6 servers

```
sw1(config)# ipv6 dhcp guard policy CLIENT-ONLY
sw1(config-dhcp-guard)#  device-role client
sw1(config)# interface range Ethernet0/2 - 3
sw1(config-if-range)#  ipv6 dhcp guard attach-policy CLIENT-ONLY
```
Host ports are `client` → the switch drops **DHCPv6 server** messages (Advertise/
Reply) arriving on them, so a rogue on the LAN can't answer clients. (The uplink to a
real DHCPv6 server/relay would be `server`.)

### 6.4 ND Inspection & IPv6 Source Guard

```
sw1(config)# ipv6 source-guard policy SG
sw1(config)# interface range Ethernet0/2 - 3
sw1(config-if-range)#  ipv6 source-guard attach-policy SG
```

- **ND inspection** (part of device-tracking) validates NS/NA against the binding
  table, so a host can't answer for an address it doesn't own — killing ND spoofing /
  IPv6 ARP-poisoning.
- **Source Guard** drops **data** traffic whose IPv6 source isn't in the binding
  table for that port — so even if a host forges packets, they're dropped at the
  first switch. It's the IPv6 sibling of IPv4 **IP Source Guard** (+ DHCP snooping +
  Dynamic ARP Inspection).

```
sw1# show ipv6 source-guard policy
sw1# show device-tracking database        ! the table all of the above enforce against
```

---

## Suggested end-state verification

```
r1# show access-lists                     ! v4 ACLs with hit counts
r1# show ipv6 access-list                 ! v6 filter
r1# show ip interface Ethernet0/1 | include verify   ! uRPF active
r1# show ip traffic | include RPF          ! spoofed packets dropped
sw1# show device-tracking database         ! FHS binding table populated
sw1# show ipv6 nd raguard policy           ! RA guard attached to host ports
```

## ACL quick cheat-sheet

| | **Standard** | **Extended** |
|---|---|---|
| Matches | source only | source, dest, protocol, port, flags |
| Number range | 1–99, 1300–1999 | 100–199, 2000–2699 (or named) |
| Place… | close to **destination** | close to **source** |
| IPv6 | (named only) | `ipv6 access-list` + `ipv6 traffic-filter` |

Every ACL: **top-down, first match wins, implicit `deny any` at the end.** IPv6 adds
implicit **ND permits** before that deny — preserve them.

---

## Reset the whole lab

```bash
sudo containerlab destroy -t filtering-fhs.clab.yml --cleanup
sudo containerlab deploy  -t filtering-fhs.clab.yml
```

---

## Where this fits

This is the second **Infrastructure Security** lab (§3.2 + §3.4), after
`../device-security` (§3.1/§4.1/§3.3). Next in the series: **telemetry & monitoring**
(SNMP / syslog / NetFlow, §4.2/§4.3/§4.6) and **DHCP v4/v6** (§4.4) — both reuse this
lab's helper-container pattern.
