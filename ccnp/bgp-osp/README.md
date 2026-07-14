# CCNP BGP Study Lab (OSPF IGP + BGP)

A 5-router `cisco_iol` containerlab topology for practicing the core BGP
principles on the CCNP ENCOR/ENARSI blueprint, with **OSPF** as the interior
gateway protocol inside the enterprise AS.

## Topology

```
                 AS 65100  "ENTERPRISE"  (OSPF area 0 + iBGP RR)
                 ┌───────────────────────────────────────────┐
                 │                 R1 (RR)                    │
                 │              1.1.1.1  core                 │
                 │            /              \                │
                 │   10.1.12 /                \ 10.1.13       │
                 │          /                  \              │
                 │        R2 ----- 10.1.23 ----- R3          │
                 │      2.2.2.2                  3.3.3.3      │
                 └───┬───────────────────────────────┬───────┘
          eBGP 65100 │ 100.64.24.0/30    100.64.35.0/30 │ eBGP 65100
                     │                                 │
                    R4 ───────── 100.64.45.0/30 ────── R5
                  4.4.4.4        eBGP 65200-65300     5.5.5.5
                 AS 65200                              AS 65300
                 "ISP-A"                               "ISP-B"
```

| Router | ASN   | Role                                   | Loopback0 |
|--------|-------|----------------------------------------|-----------|
| R1     | 65100 | Enterprise core + **Route Reflector**  | 1.1.1.1   |
| R2     | 65100 | Enterprise edge → ISP-A (iBGP client)  | 2.2.2.2   |
| R3     | 65100 | Enterprise edge → ISP-B (iBGP client)  | 3.3.3.3   |
| R4     | 65200 | ISP-A upstream                         | 4.4.4.4   |
| R5     | 65300 | ISP-B upstream                         | 5.5.5.5   |

### Addressing

| Link            | Subnet          | Protocol |
|-----------------|-----------------|----------|
| R1–R2           | 10.1.12.0/24    | OSPF a0  |
| R1–R3           | 10.1.13.0/24    | OSPF a0  |
| R2–R3           | 10.1.23.0/24    | OSPF a0  |
| R2–R4           | 100.64.24.0/30  | eBGP     |
| R3–R5           | 100.64.35.0/30  | eBGP     |
| R4–R5           | 100.64.45.0/30  | eBGP     |

### Prefixes advertised into BGP

| Prefix           | Origin        | Notes                                   |
|------------------|---------------|-----------------------------------------|
| 172.16.0.0/16    | R1            | Enterprise aggregate (static → Null0)   |
| 172.16.2.0/24    | R2 (Lo1)      | Enterprise LAN                          |
| 172.16.3.0/24    | R3 (Lo1)      | Enterprise LAN                          |
| 198.51.100.0/24  | R4 (ISP-A)    | Reachable only via ISP-A                |
| 192.0.2.0/24     | R5 (ISP-B)    | Reachable only via ISP-B                |
| **203.0.113.0/24** | **R4 & R5** | **Shared** — best-path decision point   |

## BGP concepts demonstrated

- **eBGP vs iBGP** – eBGP to the two ISPs; iBGP inside AS 65100.
- **Loopback peering + `next-hop-self`** – iBGP peers on Lo0; R2/R3 set
  next-hop-self so the RR-reflected eBGP next-hop is reachable via OSPF.
- **Route Reflector** – R1 reflects between clients R2 and R3 (no full mesh).
- **IGP ↔ BGP interaction** – OSPF advertises the loopbacks that iBGP peers on;
  without OSPF the iBGP sessions never come up.
- **Origination** – `network` statements + `aggregate` (172.16.0.0/16 via Null0).
- **LOCAL_PREF** (outbound TE) – R2 sets local-pref 200 on routes from ISP-A, so
  the enterprise egresses toward `203.0.113.0/24` via **R2/ISP-A**.
- **AS-PATH prepend** (inbound TE) – R3 prepends 65100 x3 to ISP-B so return
  traffic to the enterprise prefers **ISP-A**; ISP-B becomes backup.
- **Communities** – R4 tags the shared prefix with `65200:100`; send-community
  is enabled across sessions.

## Deploy

```bash
cd /home/lab/clabs/bgp-ospf-lab
sudo containerlab deploy -t bgp-ospf-lab.clab.yml
```

Log in with `admin` / `admin`. Destroy with:

```bash
sudo containerlab destroy -t bgp-ospf-lab.clab.yml --cleanup
```

## Verification checklist

**IGP first (BGP depends on it):**
```
R1# show ip ospf neighbor          ! expect R2 and R3 FULL
R2# show ip route ospf             ! learn 1.1.1.1, 3.3.3.3, transit nets
```

**BGP sessions:**
```
R1# show ip bgp summary            ! 2 iBGP clients Established
R2# show ip bgp summary            ! iBGP to R1 + eBGP to R4 (65200)
R4# show ip bgp summary            ! eBGP to R2 (65100) + R5 (65300)
```

**Path selection (the payoff):**
```
R3# show ip bgp 203.0.113.0        ! best path points back to R2 (LOCAL_PREF 200)
R1# show ip bgp 203.0.113.0        ! RR chooses the ISP-A path
R2# show ip bgp 203.0.113.0        ! locally preferred, weight/localpref visible
R5# show ip bgp 172.16.2.0         ! enterprise via ISP-B shows prepended AS-path
```

**Community:**
```
R2# show ip bgp 203.0.113.0        ! community 65200:100 present
```

## Suggested exercises

1. Shut `R2 e0/3` (ISP-A) and watch `203.0.113.0/24` fail over to ISP-B.
2. Change R2's `route-map ISP-A-IN` local-pref below 100 and re-check egress.
3. Add MED on R4/R5 toward the enterprise and observe tie-breaking.
4. Convert the iBGP RR back to a full mesh and compare `show ip bgp` next-hops.
5. Filter `192.0.2.0/24` inbound on R3 with a prefix-list and confirm it's gone.
6. Add `aggregate-address 172.16.0.0 255.255.0.0 summary-only` on R2 and see the
   more-specifics get suppressed.
```
