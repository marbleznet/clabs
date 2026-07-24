# CCNP Device Security Lab — AAA, Device Management & CoPP (Cisco IOL + Edgeshark)

A lab for securing and managing a router's **management** and **control** planes —
**AAA** (local, **RADIUS**, **TACACS+**), **secure device access** (SSH/SCP/
HTTP(S)/VTY), and **Control Plane Policing (CoPP)**. The routers boot with **only
interfaces and IP addresses**: no SSH, no AAA, no CoPP, no routing. You build every
control by hand and use **Edgeshark captures** to *see* the difference — Telnet
cleartext vs SSH, RADIUS vs TACACS+ on the wire, and CoPP policing a control-plane
flood.

This is the first lab in the series with **Linux helper containers**: a **RADIUS**
server, a **TACACS+** server, and an **admin workstation** run alongside the
`cisco_iol` routers on one management LAN.

By the end you will have configured and observed:

- **Secure device access** — SSH v2 (and why Telnet is unacceptable, *seen* on the
  wire), SCP, HTTP(S), console/VTY hardening
- **AAA** with a **local** database, then **RADIUS**, then **TACACS+** — method
  lists with fallback, privilege levels, and command authorization
- **RADIUS vs TACACS+** side by side, including their very different packet captures
- **CoPP** protecting the control plane — classifying SSH/SNMP/routing/ICMP and
  policing a flood, with the drops visible in `show policy-map control-plane`

> 📖 **This lab has a companion:** [`CONCEPTS.md`](CONCEPTS.md) is the in-depth
> reference behind every step (the three planes, AAA, RADIUS vs TACACS+, CoPP, and
> more). Each lab opens with a **Deep dive** callout — follow it for the *why*, skip
> it when you just want to type.

---

## Topology

```
   [radius 10.0.0.10]    [tacacs 10.0.0.11]    [admin 10.0.0.100]
      FreeRADIUS            tac_plus              SSH/SCP client +
      UDP 1812/1813         TCP 49                CoPP flood generator
           \                   |                    /
            └─────────────── SW1 (L2) ─────────────┘
                        management LAN
                        10.0.0.0/24
                      /                \
                 R1 (.1) ───── OSPF ───── R2 (.2)
              device under test        routing peer
                                       (control-plane traffic for CoPP)
```

**Why this shape?** R1 is the **device under test**. R2 is a routing peer so CoPP
has real **OSPF/BGP** control-plane traffic to protect (not just management
traffic). The **admin** box is where you SSH from and where you generate the CoPP
flood. The two **servers** back the AAA labs — and, importantly, most of the lab
(local AAA, device management, CoPP) needs **no server at all**.

### Roles, addressing & credentials

| Node   | IP           | Role                                                  |
|--------|--------------|-------------------------------------------------------|
| R1     | 10.0.0.1     | Device under test (all security config goes here)     |
| R2     | 10.0.0.2     | Routing peer (CoPP control-plane traffic) + 2nd target|
| SW1    | —            | L2 switch (management LAN)                             |
| radius | 10.0.0.10    | FreeRADIUS — user `raduser` / `radpass`               |
| tacacs | 10.0.0.11    | tac_plus — `tacadmin`/`tacpass`, `tacoper`/`operpass` |
| admin  | 10.0.0.100   | Admin workstation (ssh, scp, ping flood)              |

**Shared secret** (RADIUS + TACACS+ ↔ router): `CISCO123`. **Local** user you'll
create: `netadmin` / `NADMIN`.

Interface mapping: containerlab `ethX` → IOL `Ethernet0/X`; `E0/0` is management.

---

## Deploy

```bash
cd /home/lab/clabs/ccnp/device-security
sudo containerlab deploy -t device-security.clab.yml
```

The three Linux containers pull from Docker Hub on first deploy
(`freeradius/freeradius-server`, `lfkeitel/tacacs_plus`, `wbitt/network-multitool`).
Run commands from the admin box with:

```bash
docker exec -it clab-device-security-admin bash
# then, inside:  ssh netadmin@10.0.0.1
```

Destroy with:

```bash
sudo containerlab destroy -t device-security.clab.yml --cleanup
```

> ⚠️ **Helper-container caveat.** The RADIUS/TACACS+ **images and config paths** are
> the one part of this lab that varies by host and that isn't guaranteed on your
> setup. **Labs 1, 2 and 5 (device management, local AAA, CoPP) need no server** and
> always work. If the RADIUS (Lab 3) or TACACS+ (Lab 4) container misbehaves, check
> `docker logs clab-device-security-radius`, and adjust the image or the bind paths
> in `device-security.clab.yml` / `server-configs/` to match your image. The IOS-side
> config in this guide is correct regardless of which server you point it at.

### Sanity check before you start

```
r1# show ip interface brief         ! e0/1 + Lo0 up/up
r1# show run | include aaa|username|ip ssh   ! nothing -- blank slate
admin$ ping 10.0.0.1                ! admin can reach R1
admin$ ping 10.0.0.10               ! ...and the RADIUS server
```

---

## Using Edgeshark for captures

Capture on `clab-device-security-r1` interface `Ethernet0/1` (the management LAN).
Handy display filters for this lab:

| Filter                     | Shows                                                  |
|----------------------------|--------------------------------------------------------|
| `telnet`                   | **Telnet** — you can literally read the password       |
| `ssh`                      | **SSH** — encrypted; only the handshake is visible     |
| `radius`                   | **RADIUS** Access-Request / Accept / Reject (UDP 1812) |
| `tacplus`                  | **TACACS+** (TCP 49) — body is encrypted               |
| `icmp`                     | the CoPP flood traffic                                 |
| `tcp.port == 22`           | the SSH transport session                              |

> 📖 **Deep dive:** [The three planes: data, control, management](CONCEPTS.md#the-three-planes-data-control-management)

---

# Lab 0 — Prep: a routing adjacency for CoPP to protect

**Goal:** give the control plane something to do. Bring up a minimal OSPF adjacency
between R1 and R2 on the management LAN, so Lab 5's CoPP has real routing traffic to
classify.

```
r1(config)# router ospf 1
r1(config-router)#  network 10.0.0.0 0.0.0.255 area 0
```
(same on R2)

```
r1# show ip ospf neighbor            ! R2 is FULL -- OSPF Hellos now hit R1's control plane
```

---

# Lab 1 — Secure device access (SSH vs Telnet, SCP, HTTP(S))

**Goal:** replace cleartext management with SSH, *prove* the difference on the wire,
then enable SCP and the HTTPS management interface.

> 📖 **Deep dive:** [Secure device access](CONCEPTS.md#secure-device-access)

### 1.1 First, see why Telnet is unacceptable

> **📸 CAPTURE — `r1` `Ethernet0/1`**, filter `telnet`. Start it, then from admin:
> `telnet 10.0.0.1` isn't enabled yet — so temporarily allow it to make the point:

```
r1(config)# username netadmin privilege 15 secret NADMIN
r1(config)# line vty 0 4
r1(config-line)#  login local
r1(config-line)#  transport input telnet
```
From admin: `telnet 10.0.0.1`, log in as `netadmin`. **On the capture, follow the
TCP stream** — the username and password are in **plain ASCII**. That's the lesson.

### 1.2 Enable SSH and disable Telnet

```
r1(config)# ip domain name lab.local
r1(config)# crypto key generate rsa modulus 2048       ! RSA keypair -> enables SSH
r1(config)# ip ssh version 2
r1(config)# line vty 0 4
r1(config-line)#  transport input ssh                   ! SSH only -- Telnet off
r1(config-line)#  login local
r1(config-line)#  exec-timeout 5 0
```

> **📸 CAPTURE — `r1` `Ethernet0/1`**, filter `ssh`. From admin: `ssh
> netadmin@10.0.0.1`. Now the capture shows only the **key exchange**; the
> credentials and session are **encrypted**. Same management, night-and-day
> difference on the wire.

```
r1# show ip ssh                     ! version 2.0, keys present
r1# show ssh                        ! active encrypted sessions
```

### 1.3 SCP and HTTPS

```
r1(config)# ip scp server enable                        ! secure copy (rides SSH)
r1(config)# no ip http server                           ! kill cleartext HTTP
r1(config)# ip http secure-server                       ! HTTPS only
r1(config)# ip http authentication local
```
From admin: `scp netadmin@10.0.0.1:running-config .` copies over the encrypted SSH
transport. `(T)FTP` is the classic **insecure** file transfer — fine for a lab image
pull, never for credentials; prefer **SCP**.

---

# Lab 2 — AAA with the local database

**Goal:** turn on the AAA framework and drive login/authorization from the router's
**local** user database — the foundation (and the fallback) for the server-based
labs that follow.

> 📖 **Deep dive:** [AAA: authentication, authorization, accounting](CONCEPTS.md#aaa-authentication-authorization-accounting)
> · [Method lists and fallback](CONCEPTS.md#method-lists-and-fallback)
> · [Privilege levels and role-based access](CONCEPTS.md#privilege-levels-and-role-based-access)

### 2.1 Enable AAA and local method lists

```
r1(config)# aaa new-model                               ! turns on the AAA framework
r1(config)# aaa authentication login default local
r1(config)# aaa authorization exec default local
r1(config)# username netadmin privilege 15 secret NADMIN
r1(config)# username helpdesk privilege 1  secret HELP
```

> **`aaa new-model` is a big switch** — it changes the default login behaviour
> immediately. Keep your SSH session open and test a *new* login in parallel so you
> don't lock yourself out (Lab's `login local` fallback protects you).

### 2.2 Privilege levels (role-based CLI)

```
r1# show privilege                   ! as netadmin: level 15 (full)
```
Log in as `helpdesk` (priv 1) — you get user EXEC only, no `configure`. Optionally
lift a specific command to a low level so helpdesk can run just it:

```
r1(config)# privilege exec level 5 show running-config
```

---

# Lab 3 — AAA with RADIUS

**Goal:** authenticate against the **RADIUS** server, with **local** as the fallback,
and read the RADIUS exchange on the wire.

> 📖 **Deep dive:** [TACACS+ vs RADIUS](CONCEPTS.md#tacacs-vs-radius)
> · [AAA: authentication, authorization, accounting](CONCEPTS.md#aaa-authentication-authorization-accounting)

### 3.1 Point the router at RADIUS

```
r1(config)# radius server RAD
r1(config-radius-server)#  address ipv4 10.0.0.10 auth-port 1812 acct-port 1813
r1(config-radius-server)#  key CISCO123
r1(config-radius-server)# exit
r1(config)# aaa group server radius RAD-GRP
r1(config-sg-radius)#  server name RAD
r1(config-sg-radius)# exit
r1(config)# aaa authentication login default group RAD-GRP local
r1(config)# aaa authorization exec default group RAD-GRP local
```

> The **`local` at the end is the fallback** — if the RADIUS server is unreachable
> the router falls back to its local database, so a dead server doesn't lock you out.
> (Fallback only triggers on *no response*, **not** on an explicit reject.)

### 3.2 Test and capture

> **📸 CAPTURE — `r1` `Ethernet0/1`**, filter `radius`. Then from admin:
> `ssh raduser@10.0.0.1` (password `radpass`).

On the capture you'll see:
1. **Access-Request** (UDP → 1812) — carries the username and the password
   (**hidden**, XORed with the shared secret), plus NAS attributes.
2. **Access-Accept** — carries the **Cisco-AVPair `shell:priv-lvl=15`**, dropping
   `raduser` straight to full privilege. (A wrong password → **Access-Reject**.)

```
r1# show aaa servers                 ! RADIUS server state + request/accept counters
r1# test aaa group RAD-GRP raduser radpass legacy   ! synthetic test without logging in
```

> **Notice what RADIUS encrypts:** only the **password** is hidden — the username and
> attributes are in cleartext. Contrast this with TACACS+ next.

---

# Lab 4 — AAA with TACACS+ (per-command authorization)

**Goal:** authenticate against **TACACS+**, and use its headline feature —
**per-command authorization** — to give an operator `show` access but deny config.

> 📖 **Deep dive:** [TACACS+ vs RADIUS](CONCEPTS.md#tacacs-vs-radius)

### 4.1 Point the router at TACACS+

```
r1(config)# tacacs server TAC
r1(config-server-tacacs)#  address ipv4 10.0.0.11
r1(config-server-tacacs)#  key CISCO123
r1(config-server-tacacs)# exit
r1(config)# aaa group server tacacs+ TAC-GRP
r1(config-sg-tacacs+)#  server name TAC
r1(config-sg-tacacs+)# exit
r1(config)# aaa authentication login default group TAC-GRP local
r1(config)# aaa authorization exec default group TAC-GRP local
r1(config)# aaa authorization commands 15 default group TAC-GRP local
r1(config)# aaa accounting commands 15 default start-stop group TAC-GRP
```

### 4.2 See command authorization in action

> **📸 CAPTURE — `r1` `Ethernet0/1`**, filter `tacplus`.

- `ssh tacadmin@10.0.0.1` (`tacpass`) → full access; `configure terminal` is
  **permitted**.
- `ssh tacoper@10.0.0.1` (`operpass`) → `show ...` works, but `configure terminal`
  is **denied** by the server (`Command authorization failed`). The router asks the
  TACACS+ server about **each command** — that per-command granularity is the reason
  enterprises use TACACS+ for device administration.

On the capture, note that TACACS+ rides **TCP 49** and **encrypts the entire packet
body** (Wireshark shows an encrypted payload) — where RADIUS left everything but the
password in the clear. Also notice **separate Authorization and Accounting**
exchanges per command — TACACS+ splits the three A's; RADIUS bundles authn+authz.

```
r1# show tacacs                      ! server reachability + counters
```

---

# Lab 5 — Control Plane Policing (CoPP)

**Goal:** protect the router's CPU. A flood of management or routing traffic aimed
*at the router itself* is punted to the control plane and can starve the CPU. CoPP
classifies that traffic and **polices** it. You'll flood R1, then add CoPP and watch
the drops.

> 📖 **Deep dive:** [Control Plane Policing (CoPP)](CONCEPTS.md#control-plane-policing-copp)
> · [The three planes](CONCEPTS.md#the-three-planes-data-control-management)

### 5.1 Baseline: flood the control plane (no CoPP yet)

> **📸 CAPTURE — `r1` `Ethernet0/1`**, filter `icmp`. From admin, flood R1:
> `ping -f 10.0.0.1` (flood ping). With no CoPP every packet is punted to the CPU.

### 5.2 Classify control/management/routing traffic

```
r1(config)# ip access-list extended CoPP-MGMT
r1(config-ext-nacl)#  permit tcp any any eq 22
r1(config-ext-nacl)#  permit tcp any any eq 443
r1(config-ext-nacl)#  permit udp any any eq 161
r1(config)# ip access-list extended CoPP-ROUTING
r1(config-ext-nacl)#  permit ospf any any
r1(config-ext-nacl)#  permit tcp any any eq 179
r1(config-ext-nacl)#  permit eigrp any any
r1(config)# ip access-list extended CoPP-ICMP
r1(config-ext-nacl)#  permit icmp any any
!
r1(config)# class-map match-all CM-ROUTING
r1(config-cmap)#  match access-group name CoPP-ROUTING
r1(config)# class-map match-all CM-MGMT
r1(config-cmap)#  match access-group name CoPP-MGMT
r1(config)# class-map match-all CM-ICMP
r1(config-cmap)#  match access-group name CoPP-ICMP
```

### 5.3 Police each class — protect routing, throttle the rest

```
r1(config)# policy-map CoPP
r1(config-pmap)#  class CM-ROUTING
r1(config-pmap-c)#   police 8000 conform-action transmit exceed-action transmit   ! never starve routing
r1(config-pmap)#  class CM-MGMT
r1(config-pmap-c)#   police 32000 conform-action transmit exceed-action drop
r1(config-pmap)#  class CM-ICMP
r1(config-pmap-c)#   police 8000 conform-action transmit exceed-action drop
r1(config-pmap)#  class class-default
r1(config-pmap-c)#   police 64000 conform-action transmit exceed-action drop
!
r1(config)# control-plane
r1(config-cp)#  service-policy input CoPP
```

### 5.4 Re-flood and watch CoPP police

Flood again from admin (`ping -f 10.0.0.1`) and watch the ICMP class drop the excess:

```
r1# show policy-map control-plane            ! per-class conform/exceed counters + drops
r1# show policy-map control-plane | section ICMP   ! the flood shows as "exceeded ... drop"
```

- The **ICMP** and **MGMT** classes drop traffic above their rate — an attacker can't
  saturate the CPU through them.
- The **ROUTING** class uses `exceed-action transmit` (policed but never dropped) so a
  flood elsewhere can **never break your OSPF/BGP adjacencies**. That prioritisation
  is the whole point of CoPP.
- Your OSPF neighbor from Lab 0 **stays up** throughout the flood — verify:

```
r1# show ip ospf neighbor                    ! still FULL under load
```

---

# Quick extras (management-plane hardening)

> 📖 **Deep dive:** [Hardening the management plane](CONCEPTS.md#hardening-the-management-plane)

```
r1(config)# login block-for 60 attempts 3 within 30    ! throttle brute-force logins
r1(config)# login on-failure log                        ! log failed attempts (syslog)
r1(config)# ip access-list standard MGMT-HOSTS
r1(config-std-nacl)#  permit host 10.0.0.100             ! only the admin box
r1(config)# line vty 0 4
r1(config-line)#  access-class MGMT-HOSTS in             ! restrict who can even reach VTY
r1(config-line)#  exec-timeout 5 0
r1(config)# service password-encryption                  ! (weak) hide type-7 passwords
```

---

## Suggested end-state verification

```
r1# show run | section aaa                   ! method lists (default -> group + local fallback)
r1# show aaa servers                          ! RADIUS/TACACS+ reachability + counters
r1# show ip ssh                               ! SSHv2 only
r1# show policy-map control-plane             ! CoPP classes policing; routing protected
r1# show ip ospf neighbor                     ! adjacency survived the CoPP flood test
admin$ ssh raduser@10.0.0.1                   ! RADIUS login works, falls back to local if server down
```

## RADIUS vs TACACS+ cheat-sheet

| | **RADIUS** | **TACACS+** |
|---|---|---|
| Origin / standard | IETF (open) | Cisco (open-ish) |
| Transport | **UDP** 1812/1813 | **TCP** 49 |
| Encryption | **password only** | **entire packet body** |
| AAA separation | **authn + authz combined** | **all three separate** |
| Per-command authorization | ✗ (not granular) | ✅ (its headline feature) |
| Typical use | **network access** (802.1X, VPN, Wi-Fi) | **device administration** |

## AAA method-list mental model

```
aaa authentication login  default  group RAD-GRP  local
       │             │       │          │           └─ fallback if server unreachable
       │             │       │          └─ try the RADIUS group first
       │             │       └─ "default" applies to all lines unless a named list overrides
       │             └─ the service being controlled (login, enable, ...)
       └─ authentication (who are you?) — also: authorization, accounting
```

---

## Reset the whole lab

```bash
sudo containerlab destroy -t device-security.clab.yml --cleanup
sudo containerlab deploy  -t device-security.clab.yml
```

Because the routers hold **no** security config, every redeploy is a clean slate.

---

## Where this fits

This is the first **Infrastructure Security / Services** lab in the series (the ~45%
of ENARSI that the routing/VPN labs don't touch). It covers **§3.1 AAA**, **§4.1
device management**, and **§3.3 CoPP**. Natural siblings to build next: **filtering &
first-hop security** (ACLs / uRPF / IPv6 FHS, §3.2/§3.4) and **telemetry** (SNMP /
syslog / NetFlow, §4.2/§4.3/§4.6) — the latter reuses this lab's helper-container
pattern.
