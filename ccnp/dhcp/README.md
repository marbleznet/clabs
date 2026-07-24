# CCNP DHCP Lab — IPv4 & IPv6 (Cisco IOL + Edgeshark)

A lab for **DHCP end-to-end** — the **IOS DHCP server**, **DHCP relay** across a
router, **DHCP options**, the **IOS DHCP client**, and the whole **DHCPv6** story
(stateless, stateful, relay). Routers boot with **only interfaces and IPs** (they're
the base gateways) — no DHCP config. You build every pool and relay by hand and use
**Edgeshark** to watch **DORA** (IPv4) and the **DHCPv6** exchange on the wire.

This lab covers **ENARSI 4.4** (IPv4 and IPv6 DHCP: client, IOS server, relay,
options) and picks up directly from `../ipv6-fundamentals` (the RA **M/O flags**).

By the end you will have configured and observed:

- The **IOS DHCP server** — pools, excluded addresses, options, lease, bindings
- **DHCP relay** (`ip helper-address`) and the **giaddr** that makes it work
- The **IOS DHCP client** (`ip address dhcp`) and **address reservations**
- **DHCPv6** — **stateless** (SLAAC + options) vs **stateful** (server-assigned
  address), and **DHCPv6 relay**

> 📖 **This lab has a companion:** [`CONCEPTS.md`](CONCEPTS.md) — DORA, relay/giaddr,
> the DHCPv4 vs DHCPv6 differences, the M/O flags, and a troubleshooting section.
> Each lab opens with a **Deep dive** callout.

---

## Topology

```
  LAN-A 10.0.1.0/24 (direct)                LAN-B 10.0.2.0/24 (relayed)
  2001:db8:1::/64                           2001:db8:2::/64
                                               [pc2]    [r3 · IOS client]
   [pc1] ── R1 ═══ 10.0.12.0/30 ═══ R2 ──────┴── SW1 ──┘
    client  (DHCP server v4+v6)     (DHCP relay + LAN-B gw)
```

| Node | Address                    | Role                                          |
|------|----------------------------|-----------------------------------------------|
| R1   | LAN-A .1 / transit .1      | **IOS DHCP server** (v4 + v6)                 |
| R2   | transit .2 / LAN-B .1      | **DHCP relay** + LAN-B gateway                |
| R3   | LAN-B (leased)             | **IOS DHCP client** (`ip address dhcp`)       |
| pc1  | LAN-A (leased)             | Direct DHCP client                            |
| pc2  | LAN-B (leased)             | Relayed DHCP client                           |
| SW1  | —                          | LAN-B switch                                   |

Interface mapping: containerlab `ethX` → IOL `Ethernet0/X`; `E0/0` is management.

---

## Deploy

```bash
cd /home/lab/clabs/ccnp/dhcp
sudo containerlab deploy -t dhcp.clab.yml
```
Destroy with `sudo containerlab destroy -t dhcp.clab.yml --cleanup`.

> ⚠️ **The Linux hosts are DHCP clients — they get NO address at boot** (there's no
> server until you configure R1). Trigger a lease **on demand** after building the
> server:
> ```bash
> docker exec -it clab-dhcp-lab-pc1 udhcpc -i eth1 -f -q -n     # busybox DHCPv4 client
> ```
> IPv6 DHCP-client support in the busybox image is limited, so the **IOS client
> (R3)** is the reliable **DHCPv6-client** demo. The router-side config and the wire
> captures are the payoff regardless of host tooling.

---

## Using Edgeshark for captures

| Filter                   | Shows                                                      |
|--------------------------|------------------------------------------------------------|
| `bootp` / `dhcp`         | **DHCPv4** — Discover, Offer, Request, Ack (**DORA**)      |
| `dhcpv6`                 | **DHCPv6** — Solicit/Advertise/Request/Reply, Info-Request |
| `udp.port == 67`         | DHCPv4 server/relay side                                    |
| `udp.port == 547`        | DHCPv6 server/relay side                                    |

> 📖 **Deep dive:** [Troubleshooting DHCP](CONCEPTS.md#troubleshooting-dhcp)

---

# Lab 0 — Prep: reachability between the subnets

**Goal:** the relay (R2) forwards to the server (R1), and R1 must be able to send
offers **back** to LAN-B — so both routers need routes to the far subnet.

```
r1(config)# ip route 10.0.2.0 255.255.255.0 10.0.12.2
r1(config)# ipv6 route 2001:DB8:2::/64 2001:DB8:12::2
r2(config)# ip route 10.0.1.0 255.255.255.0 10.0.12.1
r2(config)# ipv6 route 2001:DB8:1::/64 2001:DB8:12::1
```

---

# Lab 1 — IOS DHCP server (IPv4, direct clients)

**Goal:** build a DHCP pool on R1 and watch **DORA** as pc1 (directly connected on
LAN-A) leases an address.

> 📖 **Deep dive:** [DHCP the protocol: DORA](CONCEPTS.md#dhcp-the-protocol-dora)
> · [The IOS DHCP server: pools, options, bindings](CONCEPTS.md#the-ios-dhcp-server-pools-options-bindings)

```
r1(config)# ip dhcp excluded-address 10.0.1.1 10.0.1.9    ! never hand out .1-.9 (gateways/servers)
r1(config)# ip dhcp pool LAN-A
r1(dhcp-config)#  network 10.0.1.0 255.255.255.0
r1(dhcp-config)#  default-router 10.0.1.1
r1(dhcp-config)#  dns-server 8.8.8.8
r1(dhcp-config)#  domain-name lab.local
r1(dhcp-config)#  lease 0 8 0                              ! 0 days 8 hours
```

> **📸 CAPTURE — `r1` `Ethernet0/1`**, filter `dhcp`. Then lease from pc1:
> `docker exec -it clab-dhcp-lab-pc1 udhcpc -i eth1 -f -q -n`

The capture shows the four **DORA** messages:
1. **Discover** (client → broadcast) — "any servers out there?"
2. **Offer** (server → client) — "here's 10.0.1.10, mask, gateway, DNS, lease"
3. **Request** (client → broadcast) — "I'll take that one" (broadcast so other
   servers know it declined theirs)
4. **Ack** (server → client) — "confirmed, it's yours"

```
r1# show ip dhcp binding          ! pc1's MAC -> IP, lease expiry
r1# show ip dhcp pool             ! addresses leased / available in LAN-A
```

> **The excluded-address gotcha:** the DHCP server will happily hand out the
> gateway's own address unless you exclude it. Always `ip dhcp excluded-address` the
> static range (gateways, servers, printers) *before* the pool leases them.

---

# Lab 2 — DHCP relay (the giaddr)

**Goal:** clients on LAN-B are **not** on the server's segment, and DHCP Discovers
are **broadcasts** that routers don't forward. A **relay** (`ip helper-address`)
bridges the gap.

> 📖 **Deep dive:** [DHCP relay and the giaddr](CONCEPTS.md#dhcp-relay-and-the-giaddr)

```
r2(config)# interface Ethernet0/2
r2(config-if)#  ip helper-address 10.0.12.1        ! forward DHCP (and a few UDP services) to R1
!
r1(config)# ip dhcp excluded-address 10.0.2.1 10.0.2.9
r1(config)# ip dhcp pool LAN-B
r1(dhcp-config)#  network 10.0.2.0 255.255.255.0
r1(dhcp-config)#  default-router 10.0.2.1
r1(dhcp-config)#  dns-server 8.8.8.8
```

> **📸 CAPTURE — `r1` `Ethernet0/2`**, filter `dhcp`. Then lease from pc2:
> `docker exec -it clab-dhcp-lab-pc2 udhcpc -i eth1 -f -q -n`

Two things to see on the capture:

- The relayed packets are **unicast** from R2 (10.0.12.2) to R1 (10.0.12.1) on UDP
  67 — not the broadcast the client sent.
- The **giaddr (gateway IP address)** field is set to **10.0.2.1** (R2's LAN-B
  interface). That's the magic: R1 uses giaddr both to know **which pool** to lease
  from (the one whose `network` contains 10.0.2.1) and **where to send** the reply.

```
r1# show ip dhcp binding           ! now bindings for BOTH 10.0.1.x and 10.0.2.x
```

> **`ip helper-address` forwards more than DHCP** — by default it relays a handful of
> UDP services (DHCP/BOOTP, TFTP, DNS, TIME, NetBIOS…). Tune with `ip forward-protocol
> udp <port>` if you want only some.

---

# Lab 3 — DHCP options, the IOS client, and reservations

**Goal:** hand out more than an address (options), lease to a **router** acting as a
client, and pin a fixed address with a **reservation**.

> 📖 **Deep dive:** [The IOS DHCP server: pools, options, bindings](CONCEPTS.md#the-ios-dhcp-server-pools-options-bindings)
> · [The DHCP client and reservations](CONCEPTS.md#the-dhcp-client-and-reservations)

### 3.1 DHCP options (beyond the basics)

`default-router`/`dns-server`/`domain-name` are just common options with friendly
names; anything else you set by number:

```
r1(config)# ip dhcp pool LAN-A
r1(dhcp-config)#  option 150 ip 10.0.1.5        ! TFTP server (Cisco IP phones)
r1(dhcp-config)#  option 42 ip 10.0.1.5         ! NTP server
```

### 3.2 The IOS DHCP client (R3 leases its own interface)

```
r3(config)# interface Ethernet0/1
r3(config-if)#  ip address dhcp
```
```
r3# show ip interface brief        ! e0/1 now has a 10.0.2.x address, "method DHCP"
r3# show dhcp lease                 ! the lease R3 obtained (via the R2 relay)
```
A router (or firewall/AP) taking its address from DHCP is common on WAN/edge links —
that's the "DHCP client" role in the exam.

### 3.3 Reservations (a fixed lease)

Bind a specific address to a specific client so it always gets the same IP:

```
r1(config)# ip dhcp pool RES-R3
r1(dhcp-config)#  host 10.0.2.50 255.255.255.0
r1(dhcp-config)#  client-identifier 0063.6973.636f.2d... ! R3's DHCP client-id (from 'show ip dhcp binding')
```

> Get the client-identifier from `show ip dhcp binding` (or use `hardware-address
> <mac>`). Also enable **conflict detection** (`ip dhcp conflict logging`, on by
> default) — the server pings an address before offering it and logs any conflict.

---

# Lab 4 — DHCPv6 stateless (SLAAC address + DHCPv6 options)

**Goal:** the common IPv6 model — hosts **SLAAC** their address from the RA prefix,
and get **only extra info** (DNS, domain) from a **stateless** DHCPv6 server. This is
the RA **O-flag** in action (see `../ipv6-fundamentals`).

> 📖 **Deep dive:** [DHCPv6: how it differs](CONCEPTS.md#dhcpv6-how-it-differs)
> · [Stateless vs stateful DHCPv6 (M/O flags)](CONCEPTS.md#stateless-vs-stateful-dhcpv6-mo-flags)

```
r1(config)# ipv6 dhcp pool V6-STATELESS
r1(config-dhcpv6)#  dns-server 2001:4860:4860::8888
r1(config-dhcpv6)#  domain-name lab.local
r1(config)# interface Ethernet0/1
r1(config-if)#  ipv6 nd other-config-flag           ! O-flag: "get OTHER config via DHCPv6"
r1(config-if)#  ipv6 dhcp server V6-STATELESS
```

> **📸 CAPTURE — `r1` `Ethernet0/1`**, filter `dhcpv6`. A host that sees the O-flag
> sends a **INFORMATION-REQUEST** and gets a **REPLY** with the DNS/domain — but
> **no address** (it SLAAC'd that from the prefix). Two-message exchange, address
> untouched.

```
r1# show ipv6 dhcp pool             ! the stateless pool (no address prefix)
```

---

# Lab 5 — DHCPv6 stateful (server-assigned address)

**Goal:** the DHCPv4-like model — the DHCPv6 server hands out the **address** itself.
This is the RA **M-flag**.

> 📖 **Deep dive:** [Stateless vs stateful DHCPv6 (M/O flags)](CONCEPTS.md#stateless-vs-stateful-dhcpv6-mo-flags)

Switch LAN-A from stateless to stateful:

```
r1(config)# ipv6 dhcp pool V6-STATEFUL
r1(config-dhcpv6)#  address prefix 2001:DB8:1:0:AAAA::/80
r1(config-dhcpv6)#  dns-server 2001:4860:4860::8888
r1(config-dhcpv6)#  domain-name lab.local
r1(config)# interface Ethernet0/1
r1(config-if)#  no ipv6 nd other-config-flag
r1(config-if)#  ipv6 nd managed-config-flag          ! M-flag: "get your ADDRESS via DHCPv6"
r1(config-if)#  ipv6 dhcp server V6-STATEFUL
```

Lease it from the IOS client R3 (move it to LAN-A behaviourally — or just watch pc1);
the exchange is the DHCPv6 four-step:

> **📸 CAPTURE — `r1` `Ethernet0/1`**, filter `dhcpv6`. The **SARR** exchange:
> **Solicit → Advertise → Request → Reply** (client UDP 546, server 547), with the
> server assigning an address from `2001:db8:1:0:aaaa::/80`.

```
r1# show ipv6 dhcp binding          ! the client DUID -> assigned IPv6 address
r1# show ipv6 dhcp pool             ! stateful pool usage
```

> **DUID, not MAC.** DHCPv6 identifies clients by a **DUID** (DHCP Unique Identifier),
> not the MAC address DHCPv4 uses — so bindings and reservations key off the DUID.

---

# Lab 6 — DHCPv6 relay

**Goal:** relay DHCPv6 across R2 so LAN-B clients reach the R1 server — the IPv6
`ip helper-address` equivalent.

> 📖 **Deep dive:** [DHCPv6 relay](CONCEPTS.md#dhcpv6-relay)

```
r2(config)# interface Ethernet0/2
r2(config-if)#  ipv6 dhcp relay destination 2001:DB8:12::1
```
(and build a stateful `V6-LAN-B` pool on R1 with `address prefix 2001:db8:2:0:bbbb::/80`
+ set the M-flag on R2's LAN-B interface: `ipv6 nd managed-config-flag`)

Lease from R3 (`ipv6 address dhcp` on its LAN-B interface):

```
r3(config)# interface Ethernet0/1
r3(config-if)#  ipv6 address dhcp
r1# show ipv6 dhcp binding          ! R3's DUID -> 2001:db8:2:0:bbbb::x, learned via relay
```

> **📸 CAPTURE — `r2` `Ethernet0/1`**, filter `dhcpv6`. The relay wraps the client's
> messages in **RELAY-FORWARD** / **RELAY-REPLY** (unlike DHCPv4's giaddr field,
> DHCPv6 nests the original message inside a relay message). Note it's sent to the
> **All-DHCP-Servers** multicast or the configured destination.

---

## Suggested end-state verification

```
r1# show ip dhcp binding            ! v4 leases across LAN-A + LAN-B
r1# show ipv6 dhcp binding          ! v6 leases (by DUID)
r1# show ip dhcp pool               ! utilization
r3# show ip interface brief         ! IOS client leased its address
r1# debug ip dhcp server events     ! (troubleshooting) watch DORA decisions live
```

## DHCPv4 vs DHCPv6 cheat-sheet

| | **DHCPv4 (DORA)** | **DHCPv6 (SARR)** |
|---|---|---|
| Transport | UDP 67 (server) / 68 (client), **broadcast** | UDP 547 (server) / 546 (client), **multicast** `ff02::1:2` |
| Messages | Discover, Offer, Request, Ack | Solicit, Advertise, Request, Reply |
| Client ID | MAC (chaddr) | **DUID** |
| Relay marker | **giaddr** field | **RELAY-FORWARD/REPLY** wrapper |
| Address vs SLAAC | always server-assigned | **stateful (M)** vs **stateless (O, SLAAC + options)** |
| Relay command | `ip helper-address` | `ipv6 dhcp relay destination` |

---

## Reset the whole lab

```bash
sudo containerlab destroy -t dhcp.clab.yml --cleanup
sudo containerlab deploy  -t dhcp.clab.yml
```

---

## Where this fits

Fourth **Infrastructure Services** lab (§4.4), completing the services-monitoring
cluster with `../telemetry`. It's the natural sequel to `../ipv6-fundamentals` (the
M/O flags become real here). Remaining single-tech labs: **PBR + IP SLA + tracking +
BFD**, **VRF-Lite**, and **DMVPN** — then the enterprise capstone.
