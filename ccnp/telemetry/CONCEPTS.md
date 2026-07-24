# Telemetry & Monitoring — In-Depth Concepts

The companion reference to the [README](README.md) lab guide. The README is the
*hands-on* path (configure this, poll that, capture the export); this file is the
*why* behind each step. Each 📖 **Deep dive** link in the README jumps to a section
here.

**Contents**
- [SNMP: managers, agents, MIBs, and OIDs](#snmp-managers-agents-mibs-and-oids)
- [SNMP versions and security (v2c vs v3)](#snmp-versions-and-security-v2c-vs-v3)
- [Syslog: severities, facilities, destinations](#syslog-severities-facilities-destinations)
- [Debugging safely](#debugging-safely)
- [Timestamps and time synchronization](#timestamps-and-time-synchronization)
- [NetFlow: flows and the cache](#netflow-flows-and-the-cache)
- [Traditional NetFlow, Flexible NetFlow, and IPFIX](#traditional-netflow-flexible-netflow-and-ipfix)
- [The telemetry big picture](#the-telemetry-big-picture)

---

## SNMP: managers, agents, MIBs, and OIDs

**SNMP** (Simple Network Management Protocol) is the classic pull-based monitoring
model. Two roles:

- **Manager (NMS)** — polls devices and collects data. It sends **GET / GET-NEXT /
  GET-BULK** requests (read) and **SET** (write) to UDP **161** on the agent.
- **Agent** — runs on the device, answers the manager, and can send unsolicited
  **traps / informs** to the manager on UDP **162** when something happens.

The data itself lives in the **MIB** (Management Information Base) — a **tree** of
managed objects. Each object has an **OID** (Object Identifier), a dotted numeric path
down the tree: `1.3.6.1.2.1.1.5.0` is `iso.org.dod.internet.mgmt.mib-2.system.sysName`
(the hostname). `snmpwalk` just walks a subtree with repeated GET-NEXTs.

Two push mechanisms, and the difference matters for reliability:

- **Trap** — fire-and-forget UDP. Cheap, but if it's lost, it's gone.
- **Inform** — an acknowledged trap; the agent retransmits until the manager confirms.
  Use informs when you can't afford to miss the event.

`ro` (read-only) vs `rw` (read-write) access controls whether the manager can just
*read* or also *change* config via SET — an unprotected `rw` community is a full
device compromise, so always bind it to an ACL.

[↩ back to README Lab 1](README.md#lab-1--snmp-v2c-community-strings)

---

## SNMP versions and security (v2c vs v3)

| | **v1** | **v2c** | **v3** |
|---|---|---|---|
| Credential | community string | community string | **user** (USM) |
| Authentication | none | none | **MD5 / SHA** |
| Encryption | none | none | **DES / AES (priv)** |
| Bulk transfers / informs | no | **yes** | yes |
| Verdict | obsolete | **insecure** (still common for RO) | **the secure choice** |

The headline: **v1 and v2c send the community string in cleartext in every packet**
(you saw `LAB-RO` plainly in the README Lab 1 capture). Anyone on the path can read
it, and with a captured `rw` string, rewrite the device. v2c is fine only for
low-value read-only polling on a trusted management VLAN.

**v3** replaces communities with the **User-based Security Model (USM)** and adds two
independent protections, giving three **security levels**:

- **noAuthNoPriv** — identify by username only (no better than v2c).
- **authNoPriv** — messages are **authenticated** (HMAC-MD5/SHA) so they can't be
  forged or altered, but are still readable.
- **authPriv** — authenticated **and encrypted** (AES) — the only level for anything
  sensitive.

On top of USM, the **View-based Access Control Model (VACM)** — the `snmp-server view`
+ `group` you configured — restricts *which OIDs* a user can touch, so a monitoring
account can be least-privilege (read `system` only, never the config). v3 is more
setup, but it's the difference between a credential you can sniff and one you can't.

[↩ back to README Lab 2](README.md#lab-2--snmp-v3-authentication--encryption)

---

## Syslog: severities, facilities, destinations

Every IOS log message has the form `%FACILITY-SEVERITY-MNEMONIC: text` (e.g.
`%OSPF-5-ADJCHG`). Two fields drive how you filter it:

- **Severity 0–7** — lower is worse. **0** Emergency, **1** Alert, **2** Critical,
  **3** Error, **4** Warning, **5** Notification, **6** Informational, **7** Debugging.
  (*"Every Awesome Cisco Engineer Will Need Ice-cream Daily."*) A logging threshold is
  **inclusive and below**: `informational` (6) means "send 0 through 6."
- **Facility** — the subsystem/source (OSPF, LINK, SYS, or the numeric `local0–7` used
  when forwarding to a syslog server).

A message can be sent to several **destinations**, each with its **own** severity
threshold:

| Destination | Command | Notes |
|-------------|---------|-------|
| Console | `logging console <sev>` | on by default; can flood a serial console |
| Monitor (VTY) | `logging monitor` + `terminal monitor` | must enable per session |
| **Buffer** | `logging buffered <size> <sev>` | in RAM, survives no reboot; `show logging` |
| **Syslog server** | `logging host x` + `logging trap <sev>` | UDP 514; the durable archive |
| SNMP | `snmp-server enable traps syslog` | logs as SNMP traps |

The two classic mistakes: setting `logging trap` too **low** (server sees nothing) or
including **debugging (7)** on a busy box (the server drowns, and debug logging can
hurt the CPU — see [debugging safely](#debugging-safely)).

[↩ back to README Lab 3](README.md#lab-3--logging--syslog)

---

## Debugging safely

`debug` is the most powerful and most dangerous troubleshooting tool. Unlike a
`show`, a debug hooks into live packet/event processing and prints continuously —
and some debugs are **process-switched and unthrottled**, so on a production router
they can drive the **CPU to 100%** and take the device offline. The discipline:

- **Scope it.** `debug condition interface <if>` (or `username`, `ip address`, `vlan`)
  limits following debugs to just that context, so `debug ip ospf adj` fires only for
  the interface you care about instead of every neighbor.
- **Aim the output.** On a live box, avoid the console; use `logging buffered
  debugging` and read `show logging`, or `terminal monitor` on a single VTY session
  you can close.
- **Never run broad debugs blind.** `debug ip packet` matches *every* packet — always
  pair it with an ACL (`debug ip packet <acl>`) and watch `show processes cpu`.
- **Always `undebug all`** when done, and `show debugging` to confirm nothing lingers.

Rule of thumb: reach for `show` commands first, a **conditional** debug second, and a
broad debug only on a lab box or during a maintenance window.

[↩ back to README Lab 4](README.md#lab-4--debugs--conditional-debugs)

---

## Timestamps and time synchronization

Logs are only useful if you can trust *when* they happened. `service timestamps log
datetime msec localtime show-timezone` stamps each message with the real date/time to
the millisecond (the alternative, `uptime`, is nearly useless for correlation). But an
accurate timestamp needs an accurate clock — and that's **NTP**.

Why NTP is foundational to this whole lab (and several exam topics):

- **Log correlation** — tracing an incident across routers, switches, and servers is
  impossible if their clocks disagree by minutes. Synced time lets you line up events
  from different devices.
- **[Time-based ACLs](../filtering-fhs/README.md)** fire on the router's local clock —
  wrong clock, wrong policy.
- **Certificates / SNMPv3 / crypto** care about time (validity windows, replay
  windows).

NTP organizes sources into **strata** (stratum 0 = reference clock, each hop +1),
supports **authentication** so a device only trusts legitimate servers, and should be
sourced from a stable interface. "Set the clock manually" (`clock set`) works for a
lab but drifts; production syncs to NTP.

[↩ back to README Lab 3](README.md#lab-3--logging--syslog)

---

## NetFlow: flows and the cache

SNMP tells you *how much* traffic an interface passed; **NetFlow** tells you *what the
traffic was* — who talked to whom, on what ports. It does this by tracking **flows**.

A **flow** is a **unidirectional** stream of packets that share a set of key fields —
classically the **7-tuple**: source IP, destination IP, source port, destination port,
L3 protocol, ToS byte, and input interface. The router keeps a **flow cache**: the
first packet of a conversation creates an entry; subsequent matching packets just
increment its packet/byte counters.

Entries leave the cache and are **exported** to a collector when a flow **expires**:

- **Inactive timeout** (default ~15s) — no packets seen for a while.
- **Active timeout** (default ~30min) — a long-lived flow is exported periodically so
  you don't wait forever.
- **TCP FIN/RST** — the conversation ended.

Two things to keep straight: NetFlow is **directional** (`ip flow ingress` vs
`egress` — a through-flow is seen inbound on one interface and outbound on another, so
enabling both double-counts unless you plan for it), and by default it accounts
**every** packet — **sampled** NetFlow (1-in-N) exists to cut load on high-speed
links at the cost of statistical precision. Uses: traffic analysis, capacity
planning, security/anomaly detection (a host suddenly talking to thousands of
destinations), and billing.

[↩ back to README Lab 5](README.md#lab-5--netflow-traditional-v9-export)

---

## Traditional NetFlow, Flexible NetFlow, and IPFIX

Three generations of the same idea:

- **Traditional NetFlow** — enabled per interface (`ip flow ingress`), with **fixed**
  key fields (the 7-tuple). Exports in **v5** (fixed format) or **v9** (template-based,
  extensible). Simple, but you take the fields it gives you.
- **Flexible NetFlow (FNF)** — a configurable pipeline of three pieces:
  - **flow record** — *what* to track: **`match`** fields are the **keys** (what makes
    a flow unique) and **`collect`** fields are **non-keys** (data gathered per flow,
    e.g. byte/packet counters, TCP flags). *You* choose them.
  - **flow exporter** — *where/how* to send: destination, source, transport, and
    **`export-protocol`** (`netflow-v9` or `ipfix`).
  - **flow monitor** — the glue that binds a record to an exporter and holds the cache;
    applied to an interface (`ip flow monitor NAME input`).

  FNF supports IPv6, multiple simultaneous monitors, and custom fields — it's the
  modern way and what ENARSI expects you to configure.
- **IPFIX** (IP Flow Information eXport, RFC 7011, sometimes "NetFlow v10") — the
  **IETF standard** derived directly from **v9**. Same template-based export, but
  **vendor-neutral** and extensible with standardized/enterprise information elements.
  In FNF it's a one-line choice: `export-protocol ipfix` (commonly UDP 4739) instead
  of `netflow-v9`.

**Template-based export** (v9/IPFIX) is the key evolution: the exporter periodically
sends a **template** describing the record layout, then sends **data records** that
reference it — so new fields don't break the wire format, and a collector that missed
the template can't parse the data (which is why you'll see templates re-sent
periodically in the capture).

[↩ back to README Lab 6](README.md#lab-6--flexible-netflow--ipfix)

---

## The telemetry big picture

The tools in this lab split cleanly into **pull** and **push**, and knowing which is
which explains their strengths:

- **Pull — SNMP polling.** The manager asks, periodically (every 5 min, say). Great for
  steady device-health metrics (CPU, memory, interface counters), but the polling
  interval is a floor on how fresh your data is, and polling thousands of OIDs across
  thousands of devices doesn't scale to high frequency.
- **Push — syslog, SNMP traps/informs, NetFlow.** The device speaks when something
  happens (an event, a flow expiring), so you learn about it immediately without
  polling. Syslog = events, traps = state changes, NetFlow = traffic.

The modern successor ENARSI nods to with the word **"telemetry"** is **model-driven /
streaming telemetry**: the device **streams** structured data (modeled in **YANG**)
over **gRPC/gNMI** or NETCONF to a collector on a **subscription**, at high frequency
and far more efficiently than SNMP polling. It's push-based, structured, and built for
automation pipelines — the direction monitoring is heading, while SNMP (health),
syslog (events), and NetFlow/IPFIX (traffic) remain the workhorses you must still know
cold.

[↩ back to README captures](README.md#using-edgeshark-for-captures)
