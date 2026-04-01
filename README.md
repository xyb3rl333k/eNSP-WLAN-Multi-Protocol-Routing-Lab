# eNSP WLAN & Multi-Protocol Routing Lab

A Huawei eNSP simulation lab covering enterprise WLAN deployment (AC+AP), multi-area OSPF, IS-IS, and route redistribution between routing domains. Built and debugged end-to-end with full STA-to-WAN reachability verified.

---

## Topology Overview

![Network Topology](diagrams/topology.png)

---

## Network Design

### Routing Domains

| Domain | Protocol | Devices | Area/Level |
|---|---|---|---|
| Core backbone | OSPF | PE1, PE2, P1, P2 | Area 0 (backbone) |
| Edge zone | OSPF | PE3, P1 | Area 1 |
| WAN edge | IS-IS | PE3, PE4 | Level-1, AS 49.0001 |
| WLAN access | Static | AC1 → PE4 | Default route |

### Redistribution Points

- **PE3** — redistributes OSPF into IS-IS (`import-route ospf 1`) and IS-IS into OSPF (`import-route isis 1`), acting as the OSPF/IS-IS boundary router
- **PE4** — redistributes static routes into IS-IS (`import-route static level-1`), making the WLAN subnet visible to the WAN

---

## IP Addressing

### OSPF Area 0 (Core)

| Link | Interface | IP Address |
|---|---|---|
| PE1 ↔ P1 | PE1 GE0/0/1 / P1 GE0/0/1 | 10.0.0.5/30 — 10.0.0.6/30 |
| PE1 ↔ PE2 | PE1 GE0/0/0 / PE2 GE0/0/0 | 10.0.0.1/30 — 10.0.0.2/30 |
| PE2 ↔ P2 | PE2 GE0/0/1 / P2 GE0/0/1 | 10.0.0.9/30 — 10.0.0.10/30 |
| P1 ↔ P2 | P1 GE0/0/0 / P2 GE0/0/0 | 10.0.0.13/30 — 10.0.0.14/30 |

### OSPF Area 1 (Edge)

| Link | Interface | IP Address |
|---|---|---|
| P1 ↔ PE3 | P1 GE0/0/2 / PE3 GE0/0/1 | 10.0.0.17/30 — 10.0.0.18/30 |

### IS-IS Domain (49.0001)

| Link | Interface | IP Address |
|---|---|---|
| PE3 ↔ PE4 | PE3 GE0/0/0 / PE4 GE0/0/0 | 10.0.0.25/30 — 10.0.0.26/30 |
| PE4 ↔ AC1 | PE4 GE0/0/1 / AC1 Vlanif100 | 10.0.100.1/30 — 10.0.100.2/30 |
| PE4 ↔ WAN | PE4 GE0/0/2 | 172.16.100.10/24 |

### Loopbacks

| Device | Loopback | Protocol |
|---|---|---|
| PE1 | 10.0.1.1/32 | OSPF Area 0 |
| PE2 | 10.0.2.2/32 | OSPF Area 0 |
| PE3 | 10.0.3.3/32 | OSPF Area 1 + IS-IS |
| PE4 | 10.0.4.4/32 | IS-IS |
| P1 | 10.0.5.5/32 | OSPF Area 0 |
| P2 | 10.0.6.6/32 | OSPF Area 0 |

### WLAN / Access

| Segment | Subnet | Gateway | Purpose |
|---|---|---|---|
| AP management | 172.16.10.0/24 | AC1 Vlanif10 (172.16.10.1) | CAPWAP tunnel, AP DHCP |
| STA data | 172.16.20.0/24 | AC1 Vlanif20 (172.16.20.1) | Wireless client traffic |
| AC1 ↔ PE4 transit | 10.0.100.0/30 | — | Routed uplink |

---

## WLAN Configuration

### AC1 (AC6005)

| Parameter | Value |
|---|---|
| CAPWAP source | Vlanif10 (172.16.10.1) |
| AP DHCP pool | 172.16.10.0/24, option 43 → 172.16.10.1 |
| STA DHCP pool | 172.16.20.0/24 |
| SSID | Office_LAN |
| Security | WPA/WPA2-PSK, AES |
| Forward mode | Tunnel |
| Service VLAN | VLAN 20 |
| Country code | NG |

### APs

| AP | Name | MAC | Group |
|---|---|---|---|
| AP1 | Office_AP1 | 00e0-fc6a-6e90 | Office |
| AP2 | Office_AP2 | 00e0-fc87-6430 | Office |

Both APs broadcast `Office_LAN` on:
- Radio 0 — 2.4GHz (802.11bgn), Channel 1
- Radio 1 — 5GHz (802.11ac/n), Channel 36

---

## Key Design Decisions & Lessons Learned

### 1. AC6005 physical interfaces do not support `ip address`
The AC6005 is a wireless AC, not a standard router. IP addressing must be done via SVIs (Vlanif interfaces). The uplink to PE4 uses an access port (VLAN 100) mapped to Vlanif100.

### 2. PE4 GE0/0/1 required `undo portswitch`
PE4 is an S-series device. Its GigabitEthernet ports default to switchport mode. Running `undo portswitch` converts them to routed L3 ports so `ip address` works and routing functions correctly.

### 3. AP-to-AC link must be trunk with PVID = management VLAN
AP-facing ports on AC1 are configured as trunks with `port trunk pvid vlan 10` so untagged CAPWAP traffic from the AP lands in the management VLAN automatically.

### 4. DHCP option 43 sub-option 3 is required for AP discovery
Without option 43 pointing to the AC's IP, APs cannot find the AC via DHCP and will not form a CAPWAP tunnel.

### 5. AP2 was offline due to missing ap-group assignment
`ap-id 1` had no `ap-group` configured, causing AP2 to fall into the `default` group which had no VAP profiles bound. Fix: `ap-group Office` under ap-id 1.

### 6. Return route on PE4 is mandatory
STA traffic reaches PE4 via AC1's default route, but PE4 had no route back to 172.16.20.0/24. A static route `ip route-static 172.16.20.0 255.255.255.0 10.0.100.2` on PE4 completes the bidirectional path.

### 7. Why STAs see 4 SSIDs in eNSP
Each AP has 2 radios (2.4GHz + 5GHz), each with its own BSSID. 2 APs × 2 radios = 4 entries. This is correct 802.11 behaviour — real OS WiFi pickers group entries by SSID name and hide this detail from users.

---

## Traffic Flow: STA → WAN

```
STA (172.16.20.x)
  └─► AP (CAPWAP tunnel)
        └─► AC1 Vlanif20 gateway (172.16.20.1)
              └─► AC1 default route → PE4 (10.0.100.1)
                    └─► PE4 IS-IS → PE3
                          └─► PE3 redistribution → OSPF Area 1
                                └─► WAN
```

---

## Verification Commands

```bash
# WLAN
display ap all                        # check AP state (should be 'normal')
display vap all                       # check VAP is 'up' on both radios
display station all                   # show connected STAs
display capwap all                    # CAPWAP tunnel status

# Routing
display ip routing-table              # full routing table
display isis peer                     # IS-IS neighbour adjacencies
display ospf peer                     # OSPF neighbour adjacencies

# Reachability
ping 10.0.100.1                       # AC1 → PE4 (transit link)
ping 172.16.20.x                      # PE4 → STA (return path)
```

---

## Repository Structure

```
ensp-wlan-lab/
├── README.md
├── configs/
│   ├── PE1.cfg       # OSPF Area 0 router
│   ├── PE2.cfg       # OSPF Area 0 router
│   ├── PE3.cfg       # OSPF/IS-IS redistribution point
│   ├── PE4.cfg       # IS-IS WAN edge, WLAN uplink
│   ├── P1.cfg        # OSPF core P-router
│   ├── P2.cfg        # OSPF core P-router
│   ├── AC1.cfg       # Huawei AC6005 wireless controller
│   └── LSW1.cfg      # Layer 2 access switch
└── diagrams/
    └── topology.png  # eNSP topology screenshot
```

---

## Tools & Versions

- **Simulator**: Huawei eNSP
- **AC**: AC6005 (V200R007C10SPC300)
- **Routers**: AR series (V200R003C00)
- **Switch**: S-series (V200R003C00)
  Link to drive (Download Tools) - https://drive.google.com/drive/folders/14l0xUrktNrjS88_4ZFxzoRBfq3uKksU-?usp=drive_link
