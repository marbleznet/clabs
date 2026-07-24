# Device Security — In-Depth Concepts

The companion reference to the [README](README.md) lab guide. The README is the
*hands-on* path (configure this, capture that); this file is the *why* behind each
step. Each 📖 **Deep dive** link in the README jumps to a section here. You can
also read it straight through as a primer on securing the management and control
planes.

**Contents**
- [The three planes: data, control, management](#the-three-planes-data-control-management)
- [AAA: authentication, authorization, accounting](#aaa-authentication-authorization-accounting)
- [Method lists and fallback](#method-lists-and-fallback)
- [TACACS+ vs RADIUS](#tacacs-vs-radius)
- [Privilege levels and role-based access](#privilege-levels-and-role-based-access)
- [Secure device access](#secure-device-access)
- [Control Plane Policing (CoPP)](#control-plane-policing-copp)
- [Hardening the management plane](#hardening-the-management-plane)

---

## The three planes: data, control, management

Every router does three separate jobs, and almost every security control in this lab
maps to protecting one of them:

| Plane | What it does | Examples | Protected by |
|-------|--------------|----------|--------------|
| **Data plane** | Forwards **transit** traffic through the box, in hardware/CEF | packets passing *through* R1 | ACLs, uRPF, QoS |
| **Control plane** | Builds the tables that make forwarding possible; handles anything destined **to** the router's CPU | OSPF/BGP/EIGRP, ARP, ICMP to the router, punted packets | **CoPP**, CPPr |
| **Management plane** | How admins **access and operate** the device | SSH, SNMP, syslog, NetFlow, AAA, NTP | AAA, SSH, VTY ACLs, MPP |

The key idea behind CoPP is the word **punt**: traffic *addressed to the router
itself* (a routing update, an SSH login, a ping to its interface) can't be handled by
the fast data-plane path — it's **punted up to the CPU**. The CPU is a scarce, shared
resource, so a flood of punted traffic is a denial-of-service against *everything the
control plane does*, including keeping your routing adjacencies alive. Separating the
planes in your head tells you which tool applies: filtering *transit* traffic is an
**ACL** on an interface; protecting the *CPU* from traffic aimed at the box is
**[CoPP](#control-plane-policing-copp)** on the control-plane; controlling *who logs
in* is **[AAA](#aaa-authentication-authorization-accounting)**.

[↩ back to README captures](README.md#using-edgeshark-for-captures)

---

## AAA: authentication, authorization, accounting

**AAA** is the framework (`aaa new-model` turns it on) that answers three separate
questions about anyone touching the device:

- **Authentication** — *who are you?* Validate the identity (password, token,
  certificate) against a local database or a server.
- **Authorization** — *what are you allowed to do?* Which EXEC level, which commands,
  which services (exec, network, config).
- **Accounting** — *what did you do?* An audit trail of logins and commands, sent to
  a server (`start-stop` / `stop-only`).

The three are **independent** — you can authenticate locally but authorize against a
server, or account commands without authorizing them. That independence is exactly
what [TACACS+](#tacacs-vs-radius) exploits (separate exchanges per phase) and what
RADIUS collapses (authn + authz in one Access-Accept).

`aaa new-model` is a **consequential switch**: the moment you enter it, the default
authentication method for lines changes, so an unconsidered enable of AAA can lock you
out. The safe pattern is always to define a `local` [fallback](#method-lists-and-fallback)
and test a second session before closing your first.

[↩ back to README Lab 2](README.md#lab-2--aaa-with-the-local-database)

---

## Method lists and fallback

A **method list** is an **ordered list of ways** to satisfy an AAA request; the router
tries them **left to right** and stops at the first that **responds**:

```
aaa authentication login  default  group RAD-GRP  local
                            │           │           └─ 2. fall back to local users
                            │           └───────────── 1. try the RADIUS server group
                            └─ applies to every line unless a NAMED list overrides it
```

Two rules that trip people up:

- **Fallback triggers only on *no response*** (server unreachable/timeout), **not** on
  an explicit **reject**. If RADIUS answers "Access-Reject," the router honours the
  reject — it does **not** then try `local`. So a wrong password fails even if the
  local user would have worked.
- **`default` vs named lists.** The `default` list auto-applies everywhere; a **named**
  list (`aaa authentication login VTY-LIST …`) applies only where you attach it
  (`line vty 0 4` → `login authentication VTY-LIST`). Named lists let you treat console
  and VTY differently — e.g. console = local only, VTY = server-first.

Always end a server-based list with `local` (or `enable`) as the safety net, or a dead
server becomes a lockout.

[↩ back to README Lab 2.1](README.md#21-enable-aaa-and-local-method-lists)

---

## TACACS+ vs RADIUS

Both are AAA **servers**, but they're built for different jobs and look completely
different on the wire:

| | **RADIUS** | **TACACS+** |
|---|---|---|
| Origin | IETF standard (RFC 2865) | Cisco (protocol is documented/open) |
| Transport | **UDP** — 1812 auth, 1813 acct *(legacy 1645/1646)* | **TCP** 49 (reliable, connection-oriented) |
| What's encrypted | **only the User-Password** attribute | **the entire packet body** |
| AAA phases | **authn + authz combined** (attributes ride the Access-Accept) | **all three separated** into distinct exchanges |
| Per-command authorization | ✗ — not designed for it | ✅ — asks the server about **each command** |
| Extensibility | vendor-specific attributes (VSAs), e.g. `cisco-avpair` | native attribute-value pairs |
| Best fit | **network access** — 802.1X, VPN, Wi-Fi | **device administration** — router/switch CLI |

The practical takeaways: use **RADIUS** when you're authenticating *users onto the
network* (it scales, it's an open standard, everyone speaks it). Use **TACACS+** when
you're controlling *administrators on the device*, because only TACACS+ gives you
**per-command authorization** and **full-payload encryption** — you can let an operator
run every `show` command but no `configure`, and an eavesdropper learns nothing from
the capture. On the wire (README Labs 3–4) the difference is stark: RADIUS leaves the
username and attributes in the clear and hides only the password; TACACS+ encrypts the
whole body and runs a *separate* authorization exchange for every command.

[↩ back to README Lab 4](README.md#lab-4--aaa-with-tacacs-per-command-authorization)

---

## Privilege levels and role-based access

IOS has **16 privilege levels** (0–15). Three matter by default:

- **0** — a handful of commands (`disable`, `enable`, `exit`, `logout`).
- **1** — **user EXEC** (`>` prompt): look-but-don't-touch, no `configure`.
- **15** — **privileged EXEC** (`#` prompt): full access.

Levels **2–14** are yours to define. You **move commands between levels** with the
`privilege` command — e.g. `privilege exec level 5 show running-config` lets a level-5
user run that one command without full enable rights. A user lands at a level via their
local `username … privilege N`, an `enable` secret for that level, or an authorization
attribute from a [server](#tacacs-vs-radius) (the `shell:priv-lvl=N` AV-pair you saw
RADIUS return).

Privilege levels are coarse (a command is either at/above your level or not). For finer
control, IOS also has **role-based CLI (parser views)** — named views that expose an
explicit whitelist of commands, independent of the numeric ladder. TACACS+
per-command authorization is the server-driven equivalent, and usually the cleaner
answer for real fleets.

[↩ back to README Lab 2.2](README.md#22-privilege-levels-role-based-cli)

---

## Secure device access

Management access must be **encrypted and authenticated**. The building blocks:

- **SSH vs Telnet.** Telnet carries everything — including your password — in
  **cleartext** (README Lab 1.1 shows it in the capture). SSH encrypts the session.
  SSH is not on by default; enabling it requires four things: a **hostname**, an **`ip
  domain name`**, an **RSA keypair** (`crypto key generate rsa`, ≥768 bits; use 2048),
  and **`ip ssh version 2`**. Then restrict the lines: `transport input ssh` (never
  `telnet`) plus `login local` or an AAA list.
- **SCP** — secure file copy that rides the SSH transport (`ip scp server enable`);
  replaces the insecure **(T)FTP** for moving configs/images when credentials are
  involved. Plain **TFTP** (UDP 69) and **FTP** have no encryption — lab/image-pull
  only.
- **HTTP(S)** — the web management interface. Disable cleartext **`no ip http
  server`**; use **`ip http secure-server`** (HTTPS/TLS) with `ip http authentication`
  tied to AAA.
- **Line hardening** — `exec-timeout` (auto-logout idle sessions), `logging
  synchronous` (readable console), and an `access-class` [VTY ACL](#hardening-the-management-plane)
  so only management hosts can even open a session.

The theme: every management protocol has a cleartext legacy version (Telnet, HTTP,
TFTP, FTP, SNMPv2c) and a secure modern one (SSH, HTTPS, SCP, SNMPv3). Part of
"troubleshooting device management" is spotting which one is running.

[↩ back to README Lab 1](README.md#lab-1--secure-device-access-ssh-vs-telnet-scp-https)

---

## Control Plane Policing (CoPP)

CoPP protects the **CPU** from the traffic that gets [punted to the control
plane](#the-three-planes-data-control-management). Without it, anyone who can reach the
router can flood it with pings, SSH attempts, or bogus routing packets and starve the
CPU — taking down management *and* the routing adjacencies that keep the network
converged.

CoPP is a normal **MQC** (Modular QoS CLI) policy — class-map → policy-map →
service-policy — applied to a special logical interface, the **`control-plane`**, in the
**input** direction:

1. **Classify** with class-maps that match ACLs: a **management** class (SSH/HTTPS/
   SNMP), a **routing** class (OSPF/BGP/EIGRP), a **monitoring** class (ICMP), an
   **undesirable** class (things to hard-drop), and **class-default** (everything else).
2. **Police** each class with a rate and **conform/exceed actions** (`transmit` /
   `drop`).
3. **Apply**: `control-plane` → `service-policy input <name>`.

The design principles that make CoPP safe:

- **Never starve critical control traffic.** The routing class typically uses
  `exceed-action transmit` (rate-measured but never dropped) so a flood *elsewhere* can
  never break your OSPF/BGP adjacencies — exactly what you verify at the end of README
  Lab 5.
- **Profile before you police.** Set rates from observed normal levels; too-tight a
  policy drops legitimate traffic. Start permissive, tighten with data.
- **class-default is your backstop** — police the unclassified remainder so nothing
  slips past unmetered.

A related, finer tool is **CPPr (Control Plane Protection)**, which splits the punted
traffic into **host / transit / cef-exception** sub-interfaces for separate policing;
**MPP (Management Plane Protection)** restricts management protocols to specific
interfaces. CoPP is the one ENARSI drills, and the one to reach for first.

[↩ back to README Lab 5](README.md#lab-5--control-plane-policing-copp)

---

## Hardening the management plane

Beyond encrypting access, a few reflexes shrink the attack surface:

- **Throttle brute force** — `login block-for 60 attempts 3 within 30` locks logins
  after repeated failures; `login delay` spaces attempts; `login on-failure log` /
  `on-success log` feed **syslog** so attempts are visible.
- **Restrict who can even connect** — an `access-class` ACL on the VTY lines
  (`access-class MGMT-HOSTS in`) means only management subnets can open a session; the
  rest never get a login prompt.
- **Time out idle sessions** — `exec-timeout` on con/vty so an abandoned session
  doesn't stay open.
- **Protect stored secrets** — `enable secret` (modern **type 8/9**, scrypt/PBKDF2)
  over the legacy `enable password`; `service password-encryption` only hides passwords
  with reversible **type 7** (obfuscation, *not* encryption — treat type 7 as
  cleartext).
- **Turn off what you don't use** — unused management services (HTTP server, CDP on
  edge ports, etc.) are attack surface.

None of these replace [AAA](#aaa-authentication-authorization-accounting) or
[CoPP](#control-plane-policing-copp) — they're the cheap, always-do layer on top.

[↩ back to README quick extras](README.md#quick-extras-management-plane-hardening)
