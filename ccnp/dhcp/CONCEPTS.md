# DHCP (v4 & v6) — In-Depth Concepts

The companion reference to the [README](README.md) lab guide. The README is the
*hands-on* path (configure this, lease that, capture DORA); this file is the *why*
behind each step. Each 📖 **Deep dive** link in the README jumps to a section here.

**Contents**
- [DHCP the protocol: DORA](#dhcp-the-protocol-dora)
- [The IOS DHCP server: pools, options, bindings](#the-ios-dhcp-server-pools-options-bindings)
- [DHCP relay and the giaddr](#dhcp-relay-and-the-giaddr)
- [The DHCP client and reservations](#the-dhcp-client-and-reservations)
- [DHCPv6: how it differs](#dhcpv6-how-it-differs)
- [Stateless vs stateful DHCPv6 (M/O flags)](#stateless-vs-stateful-dhcpv6-mo-flags)
- [DHCPv6 relay](#dhcpv6-relay)
- [Troubleshooting DHCP](#troubleshooting-dhcp)

---

## DHCP the protocol: DORA

DHCP hands a client a full IP configuration through a four-message exchange, **DORA**
(client uses UDP **68**, server UDP **67**):

1. **Discover** — the client has no address, so it broadcasts (`0.0.0.0` →
   `255.255.255.255`) "any DHCP servers here?"
2. **Offer** — a server replies with a candidate address plus mask, gateway, DNS, and
   lease time.
3. **Request** — the client **broadcasts** its acceptance of one offer. Broadcasting
   (not unicasting) is deliberate: it tells *other* servers their offers were declined
   so they can return those addresses to their pools.
4. **Ack** — the chosen server confirms the lease (or **Nak** if the address is no
   longer valid).

A lease isn't forever — the client tracks two timers: at **T1 (50%)** it **renews**
by unicasting a Request straight to its server; if that fails, at **T2 (87.5%)** it
**rebinds** by broadcasting to any server. Other messages fill out the lifecycle:
**Release** (client gives the address back), **Decline** (client detected the address
is already in use), and **Inform** (client already has an address but wants *options*
only). Knowing DORA cold is the key to reading a DHCP capture and to diagnosing where
a failing client got stuck.

[↩ back to README Lab 1](README.md#lab-1--ios-dhcp-server-ipv4-direct-clients)

---

## The IOS DHCP server: pools, options, bindings

An IOS router is a capable DHCP server. The pieces:

- **`ip dhcp excluded-address`** (global) — the addresses the server must **never**
  lease (gateways, servers, printers). Set this *first*; the server will otherwise
  hand out `10.0.1.1` — the router's own address.
- **`ip dhcp pool NAME`** — one pool per subnet. `network` defines the range; friendly
  sub-commands (`default-router`, `dns-server`, `domain-name`, `lease`) are just the
  common **options** with names.
- **Options by number** — everything else is `option <n>`: **150** (TFTP server IP,
  Cisco phones), **66** (TFTP name), **42** (NTP), **43** (vendor-specific), **82**
  (relay agent info). Options are how DHCP delivers far more than an address.
- **Bindings** (`show ip dhcp binding`) — the server's record of which client has which
  lease, optionally saved to a **database agent** (`ip dhcp database`) so leases
  survive a reboot.

Pool **selection** is automatic: for a directly-connected client the server picks the
pool whose `network` matches the receiving interface; for a **relayed** client it
matches the pool against the **[giaddr](#dhcp-relay-and-the-giaddr)**. One server with
several pools can therefore serve many subnets.

[↩ back to README Lab 1](README.md#lab-1--ios-dhcp-server-ipv4-direct-clients)

---

## DHCP relay and the giaddr

DHCP Discovers are **broadcasts**, and routers don't forward broadcasts — so a client
can only reach a DHCP server on **its own segment**. Since you don't want a server on
every VLAN, the router itself relays: **`ip helper-address <server>`** on the
client-facing interface turns the client's broadcast into a **unicast** to the server.

The field that makes it all work is the **giaddr (gateway IP address)** in the DHCP
header. When the relay forwards a Discover it stamps giaddr with **its own
receiving-interface address**, and that single field does two jobs:

- **Pool selection** — the server matches giaddr against its pools' `network`
  statements to pick the right subnet's pool (that's why a relayed client on
  10.0.2.0/24 gets a 10.0.2.x lease even though the server has no interface there).
- **Return path** — the server unicasts the Offer/Ack back to the giaddr (the relay),
  which delivers it onto the client's segment.

Two things people trip on: the server needs a **route back to the client subnet**
(giaddr), or its replies never arrive — and `ip helper-address` relays a **default set
of UDP services**, not just DHCP (TIME 37, TACACS 49, DNS 53, BOOTP/DHCP 67/68, TFTP
69, NetBIOS 137/138). Trim it with `ip forward-protocol udp <port>` / `no ip
forward-protocol` when you want DHCP only. The relay can also insert **Option 82**
(circuit/remote-ID) so the server knows exactly which port a request came from.

[↩ back to README Lab 2](README.md#lab-2--dhcp-relay-the-giaddr)

---

## The DHCP client and reservations

DHCP isn't only for PCs. A **router, firewall, or AP interface** can take its address
from DHCP with **`ip address dhcp`** (IPv4) / **`ipv6 address dhcp`** (IPv6) — the
"DHCP client" role, extremely common on ISP-facing WAN links where the provider
assigns your edge address dynamically.

A **reservation** pins a fixed address to a specific client so it always leases the
same IP (printers, servers, APs you want at a known address without static config on
the device). On IOS it's a single-host pool:

```
ip dhcp pool RES-PRINTER
 host 10.0.2.50 255.255.255.0
 client-identifier 0100.1122.3344.55     ← or: hardware-address 0011.2233.4455
```

The client is matched by its **client-identifier** (what it sends in the Discover —
read it from `show ip dhcp binding`) or its **hardware-address**. Alongside
reservations, **conflict detection** protects the pool: the server **pings** an
address before offering it, and a client sends a **Decline** (and the server logs a
conflict) if it finds the address already in use — inspect with `show ip dhcp
conflict`.

[↩ back to README Lab 3](README.md#lab-3--dhcp-options-the-ios-client-and-reservations)

---

## DHCPv6: how it differs

DHCPv6 does the same job but is a **different protocol**, and the differences all trace
to how IPv6 works:

| | DHCPv4 | DHCPv6 |
|---|---|---|
| Ports | UDP 67 / 68 | UDP **547** (server/relay) / **546** (client) |
| Delivery | broadcast | **multicast** — clients send to `ff02::1:2` (All-DHCP-Relay-Agents-and-Servers) |
| Client identity | MAC (`chaddr`) | **DUID** (DHCP Unique Identifier) + **IAID** per interface |
| Relationship to autoconfig | *the* way to get an address | **works alongside SLAAC/RA** — the RA flags decide its role |
| Exchange | Discover/Offer/Request/Ack | **Solicit/Advertise/Request/Reply** (SARR) |

The biggest conceptual shift: in IPv4, DHCP is how a host normally gets its address; in
IPv6, a host can get its address from **SLAAC** (RA prefix + self-generated interface
ID) *without any DHCP at all*, and DHCPv6 is an **optional supplement** whose role is
dictated by the **[RA M/O flags](#stateless-vs-stateful-dhcpv6-mo-flags)**. Also note
**DUID**: DHCPv6 identifies a client by a stable DUID (often derived from a MAC + time,
but treated as opaque), *not* the MAC — so bindings and reservations key off the DUID.

[↩ back to README Lab 4](README.md#lab-4--dhcpv6-stateless-slaac-address--dhcpv6-options)

---

## Stateless vs stateful DHCPv6 (M/O flags)

Which (if any) DHCPv6 a host uses is decided by two bits in the **Router
Advertisement** — the same M/O flags from `../ipv6-fundamentals`:

| RA flags | Host behaviour | DHCPv6 server type |
|----------|----------------|--------------------|
| A-flag only (M=0, O=0) | **SLAAC** address, no DHCPv6 | none |
| **O=1** (M=0) | SLAAC address + **stateless DHCPv6** for options (DNS, domain) | **stateless** (no address pool) |
| **M=1** | **stateful DHCPv6** assigns the address (and options) | **stateful** (has `address prefix`) |

- **Stateless DHCPv6** — the server keeps **no per-client state** and hands out only
  *configuration* (DNS, domain, etc.). The host already SLAAC'd its address, so it just
  sends a two-message **INFORMATION-REQUEST → REPLY**. Config: an `ipv6 dhcp pool` with
  **no** `address prefix`, plus **`ipv6 nd other-config-flag`** on the interface.
- **Stateful DHCPv6** — the server **assigns the address** from an `address prefix`
  pool and tracks the lease (a **binding**, by DUID), just like DHCPv4. The host runs
  the full **SARR** exchange. Config: `ipv6 dhcp pool` **with** `address prefix`, plus
  **`ipv6 nd managed-config-flag`** (M) on the interface.

The classic failure is a **flag/server mismatch**: set the M-flag but provide only a
stateless pool (no `address prefix`) and clients ask for an address nobody will give —
"IPv6 host has a link-local but no global address." Make the flag and the pool agree.

[↩ back to README Lab 5](README.md#lab-5--dhcpv6-stateful-server-assigned-address)

---

## DHCPv6 relay

Like IPv4, a DHCPv6 client can't reach a server off-segment, so the router relays with
**`ipv6 dhcp relay destination <server>`** on the client-facing interface. But the
mechanism differs from DHCPv4's giaddr field: DHCPv6 **wraps** the client's message in
a **RELAY-FORWARD** message (server responds with **RELAY-REPLY**), literally nesting
the original packet inside the relay packet — and relays can be **chained**, each
adding a layer.

The relay records the client's **link-address** in the RELAY-FORWARD, which the server
uses the way it uses giaddr in v4 — to pick the right pool for the client's subnet. The
destination can be a specific server address or the **All-DHCP-Servers** multicast
(`ff05::1:3`). Everything else — stateful vs stateless, DUID bindings — works exactly
as it does for a directly-connected client.

[↩ back to README Lab 6](README.md#lab-6--dhcpv6-relay)

---

## Troubleshooting DHCP

A client with **no address** (or a `169.254.x.x` **APIPA** self-assignment, the
"nobody answered" symptom) is the usual ticket. Work the path:

1. **Is the request reaching a server?** On a remote subnet, is **`ip helper-address`**
   configured on the client's gateway? Capture on the server side for the relayed
   unicast.
2. **Does the server have a matching pool?** The pool's `network` must contain the
   client's subnet — i.e. match the **[giaddr](#dhcp-relay-and-the-giaddr)** for
   relayed clients. A missing/typo'd pool = silent no-lease.
3. **Is the pool exhausted or mis-excluded?** `show ip dhcp pool` for utilization;
   check you didn't exclude the whole range (or *fail* to exclude the gateway, so it
   got leased and now conflicts).
4. **Can the server route back?** The server needs a route to the client subnet
   (giaddr) or its Offer never arrives — a frequent one-way failure.
5. **Conflicts?** `show ip dhcp conflict` — duplicate addresses (a static host inside
   the pool range) show here.

Tools: **`debug ip dhcp server events`** (watch pool selection and lease decisions
live), `show ip dhcp binding`, and a capture filtered on `dhcp`. For **DHCPv6**, add:
are the **[M/O flags](#stateless-vs-stateful-dhcpv6-mo-flags)** consistent with the
server type, and is the relay `ipv6 dhcp relay destination` set? `debug ipv6 dhcp` and
`show ipv6 dhcp binding` are the equivalents.

[↩ back to README captures](README.md#using-edgeshark-for-captures)
