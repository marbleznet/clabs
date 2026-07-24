# CCNP Telemetry & Monitoring Lab (Cisco IOL + Edgeshark)

A lab for the "how do I *see* what the network is doing" toolset — **SNMP** (v2c and
v3), **logging / syslog / debugs**, and **NetFlow** (traditional, Flexible NetFlow,
and IPFIX). R1 boots with **only interfaces and IP addresses**: no SNMP, no logging
config, no NetFlow. You build every agent/exporter by hand and use **Edgeshark
captures** to watch the telemetry leave R1 on the wire — the payoff here is seeing
the SNMP polls, syslog messages, and flow-export packets themselves.

This lab covers **ENARSI 4.2** (SNMP), **4.3** (logging), and **4.6** (NetFlow). It
reuses the helper-container pattern: an **nms** collector and a **host1** traffic
source.

By the end you will have configured and observed:

- **SNMP v2c** (community strings) and **SNMP v3** (users/groups/views, authPriv) —
  and *why* v3 exists, seen on the wire
- **Logging** to buffer and **syslog** server, **severity levels**, **timestamps**,
  and **conditional debugs**
- **NetFlow** — traditional v9 export, then **Flexible NetFlow** and **IPFIX**, with
  the flow cache and the export templates/records on the wire

> 📖 **This lab has a companion:** [`CONCEPTS.md`](CONCEPTS.md) — SNMP internals,
> syslog severities, safe debugging, NetFlow vs FNF vs IPFIX, and where model-driven
> telemetry is heading. Each lab opens with a **Deep dive** callout.

---

## Topology

```
   [nms 10.0.0.10]        [host1 10.0.0.20]          LAN 10.0.0.0/24
   SNMP poller +            traffic source
   syslog + NetFlow             \
   collector                     \
        \                         \
         └──────── SW1 (L2) ───────┘
                       │ e0/1
                     R1  (SNMP agent · syslog source · NetFlow exporter)
                       │ e0/2  10.0.12.0/30
                     R2  (OSPF neighbor; Lo1 172.16.2.0/24 = traffic destination)
```

| Node  | Address        | Role                                                   |
|-------|----------------|--------------------------------------------------------|
| R1    | 10.0.0.1 / .12.1 | **Monitored** router — agent + exporters             |
| R2    | 10.0.12.2      | OSPF neighbor; Lo1 172.16.2.0/24 is the traffic sink    |
| nms   | 10.0.0.10      | Collector — SNMP poller, syslog + NetFlow receiver     |
| host1 | 10.0.0.20      | Traffic source (drives NetFlow)                        |
| SW1   | —              | LAN switch                                             |

Run collector/host commands with `docker exec -it clab-telemetry-lab-nms bash`.

---

## Deploy

```bash
cd /home/lab/clabs/ccnp/telemetry
sudo containerlab deploy -t telemetry.clab.yml
```
Destroy with `sudo containerlab destroy -t telemetry.clab.yml --cleanup`.

> ⚠️ **Scope note.** The **router side is fully hands-on**, and every piece of
> telemetry is **visible on the wire** via Edgeshark — that's the reliable payoff and
> doesn't depend on collector software. The **nms** container is a general Linux
> image; `snmpwalk`, a syslog daemon, and `nfcapd`/`nfdump` may need a tool-specific
> image or a quick `apk add`/`apt install`. Where a collector command is shown, it's
> the standard tool — swap in whatever you have.

### Sanity check

```
r1# show ip interface brief
nms$ ping 10.0.0.1                 ! collector reaches R1
```

---

## Using Edgeshark for captures

Capture on `clab-telemetry-lab-r1` `Ethernet0/1` (toward the nms/LAN).

| Filter                    | Shows                                                    |
|---------------------------|----------------------------------------------------------|
| `snmp`                    | SNMP GET/GET-NEXT/Response (161) and traps (162)         |
| `syslog`                  | syslog messages (UDP 514)                                |
| `cflow`                   | NetFlow / IPFIX export (templates + flow records)        |
| `udp.port == 2055`        | traditional NetFlow v9 export (Lab 5)                    |
| `udp.port == 4739`        | IPFIX export (Lab 6)                                     |

> 📖 **Deep dive:** [The telemetry big picture](CONCEPTS.md#the-telemetry-big-picture)

---

# Lab 0 — Prep: OSPF + a traffic path

**Goal:** give NetFlow real flows and syslog real events. Bring up OSPF R1↔R2 so LAN
hosts can reach `172.16.2.0/24` through R1.

```
r1(config)# router ospf 1
r1(config-router)#  network 10.0.0.0 0.0.0.255 area 0
r1(config-router)#  network 10.0.12.0 0.0.0.3 area 0
!
r2(config)# router ospf 1
r2(config-router)#  network 10.0.12.0 0.0.0.3 area 0
r2(config-router)#  network 172.16.2.0 0.0.0.255 area 0
```
```
host1$ ping 172.16.2.1            ! LAN -> through R1 -> R2's Lo1 (the flow we'll watch)
```

---

# Lab 1 — SNMP v2c (community strings)

**Goal:** enable the SNMP agent with v2c, poll it from the nms, send a trap — and see
the security problem that makes v3 necessary.

> 📖 **Deep dive:** [SNMP: managers, agents, MIBs, and OIDs](CONCEPTS.md#snmp-managers-agents-mibs-and-oids)
> · [SNMP versions and security (v2c vs v3)](CONCEPTS.md#snmp-versions-and-security-v2c-vs-v3)

```
r1(config)# snmp-server community LAB-RO ro
r1(config)# snmp-server community LAB-RW rw
r1(config)# snmp-server location Rack1-Lab
r1(config)# snmp-server contact netadmin@lab.local
r1(config)# snmp-server host 10.0.0.10 version 2c LAB-RO
r1(config)# snmp-server enable traps
```

> **📸 CAPTURE — `r1` `Ethernet0/1`**, filter `snmp`. From nms:
> `snmpwalk -v2c -c LAB-RO 10.0.0.1 system`

```
r1# show snmp                     ! packet counters
r1# show snmp community           ! the RO/RW strings (and any ACL)
```

- **`ro`** = read-only, **`rw`** = read-write (with `rw` an attacker can *change*
  config over SNMP — lock it to an ACL: `snmp-server community LAB-RW rw 10`).
- **The v2c security hole:** in the capture, the community string **`LAB-RO` is in
  plaintext** in every packet — anyone sniffing the segment has your credential.
  That's the entire reason for v3.

---

# Lab 2 — SNMP v3 (authentication + encryption)

**Goal:** replace community strings with v3's user-based security model — **authPriv**
(authenticated *and* encrypted).

> 📖 **Deep dive:** [SNMP versions and security (v2c vs v3)](CONCEPTS.md#snmp-versions-and-security-v2c-vs-v3)

```
r1(config)# snmp-server view V-SYSTEM system included
r1(config)# snmp-server group G-ADMIN v3 priv read V-SYSTEM
r1(config)# snmp-server user snmpadmin G-ADMIN v3 auth sha AUTHpass123 priv aes 128 PRIVpass123
```

> **📸 CAPTURE — `r1` `Ethernet0/1`**, filter `snmp`. From nms:
> `snmpwalk -v3 -l authPriv -u snmpadmin -a SHA -A AUTHpass123 -x AES -X PRIVpass123 10.0.0.1 system`

```
r1# show snmp user                ! snmpadmin, auth SHA, priv AES
r1# show snmp group               ! G-ADMIN, security model v3 priv, view V-SYSTEM
```

The three v3 **security levels** (set which you require in the group):

| Level | Auth? | Encrypt? | Keyword |
|-------|-------|----------|---------|
| noAuthNoPriv | no | no | `noauth` |
| authNoPriv | yes | no | `auth` |
| **authPriv** | **yes** | **yes** | `priv` |

On the capture, the v3 PDU is **encrypted** — no OIDs or values are readable, unlike
the v2c walk. A **view** additionally limits *which* OIDs the user can even see
(here just the `system` subtree), so credentials can be least-privilege.

---

# Lab 3 — Logging & syslog

**Goal:** send log messages to a **buffer** and a **syslog server**, control the
**severity** threshold, and stamp messages with accurate **timestamps**.

> 📖 **Deep dive:** [Syslog: severities, facilities, destinations](CONCEPTS.md#syslog-severities-facilities-destinations)
> · [Timestamps and time synchronization](CONCEPTS.md#timestamps-and-time-synchronization)

```
r1(config)# service timestamps log datetime msec localtime show-timezone
r1(config)# logging buffered 32768 informational
r1(config)# logging host 10.0.0.10
r1(config)# logging trap informational          ! send severity 0-6 to the syslog server
r1(config)# logging source-interface Ethernet0/1
```

Generate an event and watch it land locally and on the wire:

> **📸 CAPTURE — `r1` `Ethernet0/1`**, filter `syslog`. Then bounce the R2 link:

```
r1(config)# interface Ethernet0/2
r1(config-if)#  shutdown
r1(config-if)#  no shutdown
```
```
r1# show logging                  ! %OSPF-5-ADJCHG (neighbor down/up), %LINK/%LINEPROTO
```

On the capture you'll see the same messages as **syslog UDP 514** to 10.0.0.10, each
tagged with its **facility.severity**. The **`logging trap <level>`** threshold is
inclusive-and-below: `informational` (6) sends 0–6 but not `debugging` (7). Getting
this wrong is why "my syslog server sees nothing" *or* "it's drowning in noise."

---

# Lab 4 — Debugs & conditional debugs

**Goal:** use `debug` safely — scope it with **conditions**, capture it, and never
melt a production CPU.

> 📖 **Deep dive:** [Debugging safely](CONCEPTS.md#debugging-safely)

```
r1# terminal monitor                        ! see logs/debugs on THIS vty session
r1(config)# logging buffered debugging      ! also capture debug output in the buffer
```

Scope a debug so it only fires for the traffic/interface you care about:

```
r1# debug condition interface Ethernet0/2   ! restrict following debugs to e0/2
r1# debug ip ospf adj                        ! OSPF adjacency events only
r1(config)# interface Ethernet0/2
r1(config-if)#  shutdown
r1(config-if)#  no shutdown                  ! watch the scoped adjacency debug
r1# undebug all                              ! ALWAYS turn it off
r1# show debugging                           ! confirm nothing's still running
```

> **The danger:** `debug ip packet` (and other broad debugs) is **process-switched
> and unthrottled** — on a busy router it can spike the CPU to 100% and take the box
> down. Rules: prefer a **conditional** debug (`debug condition …`), send output to
> the **buffer** not the console on a live box, and `undebug all` the instant you're
> done. `show processes cpu` before/after tells you if a debug is hurting.

---

# Lab 5 — NetFlow (traditional v9 export)

**Goal:** account for **flows** (conversations) crossing R1 and export them to the
collector. Start with interface-based ("traditional") NetFlow.

> 📖 **Deep dive:** [NetFlow: flows and the cache](CONCEPTS.md#netflow-flows-and-the-cache)
> · [Traditional NetFlow, Flexible NetFlow, and IPFIX](CONCEPTS.md#traditional-netflow-flexible-netflow-and-ipfix)

```
r1(config)# ip flow-export destination 10.0.0.10 2055
r1(config)# ip flow-export version 9
r1(config)# ip flow-export source Ethernet0/1
r1(config)# interface Ethernet0/2
r1(config-if)#  ip flow ingress
r1(config-if)#  ip flow egress
```

Generate a few flows, then read the cache and watch the export:

> **📸 CAPTURE — `r1` `Ethernet0/1`**, filter `udp.port==2055`. From host1:
> `ping 172.16.2.1` and `curl` a couple of things through R1.

```
r1# show ip cache flow            ! the flow table: src/dst, ports, protocol, packets, bytes
r1# show ip flow interface        ! where NetFlow is enabled
r1# show ip flow export           ! export destination, version, datagram counts
```

On the capture, a **v9 export** sends a **template FlowSet** (defining the fields)
followed by **data FlowSets** (the actual flow records). A flow is the classic
**7-tuple** (src/dst IP, src/dst port, protocol, ToS, input interface) — see the deep
dive.

---

# Lab 6 — Flexible NetFlow & IPFIX

**Goal:** the modern, configurable pipeline — you define **what** to key on and
collect (**flow record**), **where** to send it (**flow exporter**, here **IPFIX**),
tie them together (**flow monitor**), and apply it.

> 📖 **Deep dive:** [Traditional NetFlow, Flexible NetFlow, and IPFIX](CONCEPTS.md#traditional-netflow-flexible-netflow-and-ipfix)

```
r1(config)# flow record FR-V4
r1(config-flow-record)#  match ipv4 source address
r1(config-flow-record)#  match ipv4 destination address
r1(config-flow-record)#  match transport source-port
r1(config-flow-record)#  match transport destination-port
r1(config-flow-record)#  match ipv4 protocol
r1(config-flow-record)#  collect counter bytes
r1(config-flow-record)#  collect counter packets
r1(config-flow-record)#  collect interface input
!
r1(config)# flow exporter FE-NMS
r1(config-flow-exporter)#  destination 10.0.0.10
r1(config-flow-exporter)#  source Ethernet0/1
r1(config-flow-exporter)#  transport udp 4739
r1(config-flow-exporter)#  export-protocol ipfix
!
r1(config)# flow monitor FM-V4
r1(config-flow-monitor)#  record FR-V4
r1(config-flow-monitor)#  exporter FE-NMS
!
r1(config)# interface Ethernet0/2
r1(config-if)#  ip flow monitor FM-V4 input
```

> **📸 CAPTURE — `r1` `Ethernet0/1`**, filter `udp.port==4739`. Generate traffic from
> host1 again.

```
r1# show flow record FR-V4        ! the key (match) and non-key (collect) fields
r1# show flow monitor FM-V4 cache ! the FNF cache with your custom fields
r1# show flow exporter FE-NMS     ! IPFIX, destination, datagrams sent
```

Two ideas that define FNF/IPFIX:

- **`match` = key fields** (what makes a flow unique); **`collect` = non-key fields**
  (data gathered per flow, e.g. counters). You choose them, instead of the fixed
  7-tuple of traditional NetFlow.
- **IPFIX** (RFC 7011) is the **IETF standard** derived from NetFlow **v9** — same
  template-based design, vendor-neutral. `export-protocol netflow-v9` vs `ipfix` is a
  one-line switch; capture both and compare the templates.

---

## Suggested end-state verification

```
r1# show snmp user / show snmp community    ! v3 user + v2c strings
r1# show logging                             ! buffer + syslog host + trap level
r1# show flow monitor FM-V4 cache            ! Flexible NetFlow cache populated
r1# show flow exporter statistics            ! IPFIX datagrams leaving R1
nms$ snmpwalk -v3 -l authPriv -u snmpadmin ... 10.0.0.1 system   ! v3 poll works
```

## Cheat-sheets

**Syslog severities (0 = worst)** — memorize *"Every Awesome Cisco Engineer Will Need
Ice-cream Daily"*:

| 0 Emergency | 1 Alert | 2 Critical | 3 Error | 4 Warning | 5 Notification | 6 Informational | 7 Debugging |
|---|---|---|---|---|---|---|---|

**SNMP versions**

| | v1 | v2c | v3 |
|---|---|---|---|
| Auth | community | community | **user + auth (MD5/SHA)** |
| Encryption | none | none | **priv (DES/AES)** |
| Use | legacy | still common (RO) | **anything sensitive** |

**Telemetry export ports:** SNMP 161 (poll) / 162 (trap) · syslog UDP 514 · NetFlow
v9 commonly 2055 · IPFIX 4739.

---

## Reset the whole lab

```bash
sudo containerlab destroy -t telemetry.clab.yml --cleanup
sudo containerlab deploy  -t telemetry.clab.yml
```

---

## Where this fits

Third **Infrastructure Services** lab, covering §4.2 / §4.3 / §4.6. Sits alongside
`../device-security` (management plane) and `../filtering-fhs` (data-plane security).
Next: **DHCP v4/v6** (§4.4).
