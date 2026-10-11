# Per-protocol configuration

[← Back to README](../../README.en.md) · [Português](../pt-br/configuracao.md)

Excerpts taken from the running-configs in [`configs/`](../../configs/). Interface descriptions in the configs are in Portuguese (Matriz = headquarters, Filial = branch).

---

## 1. VLANs and inter-VLAN routing

| VLAN | Name | Sites |
|---|---|---|
| 10 | Users | HQ, Branch 1, Branch 2 |
| 20 | IT | HQ, Branch 1, Branch 2 |
| 30 | Services / Web | HQ |
| 99 | Management (native) | HQ, Branch 1, Branch 2 |

PC ports are set to `switchport mode access` in their VLAN. Links between switches, and between the central switch and the router, are 802.1Q trunks with VLAN 99 as native:

```
interface GigabitEthernet0/1
 description **TRUNK p/ R-MATRIZ (Router-on-a-Stick)**
 switchport trunk native vlan 99
 switchport mode trunk
```

Inter-VLAN routing happens on each site's router (router-on-a-stick), with one sub-interface per VLAN. R-M example:

```
interface GigabitEthernet0/0.10
 description **Gateway VLAN 10 - Usuarios**
 encapsulation dot1Q 10
 ip address 200.18.40.1 255.255.255.192
 ip helper-address 200.18.40.174
!
interface GigabitEthernet0/0.99
 description **Gateway VLAN 99 - Gerencia**
 encapsulation dot1Q 99 native
 ip address 200.18.40.177 255.255.255.248
```

Every switch has a management SVI on VLAN 99 and the site's `ip default-gateway`.

![VLANs on SW-CORE](../img/02-vlans.png)

---

## 2. STP (PVST+)

Priorities were set manually to keep the L2 topology predictable:

| Switch | Priority | Role |
|---|---|---|
| SW-CORE | 0 (VLAN 99), 24576 (VLANs 10, 20, 30) | Root for every HQ VLAN |
| SW1-M | 28672 (VLANs 10, 20, 99) | Backup root |
| SW1-A, SW2-A / SW1-B, SW2-B | 28672 (VLANs 10, 20, 99) | Same priority on both switches at each branch: the root is decided by the lowest MAC |

```
spanning-tree mode pvst
spanning-tree vlan 99 priority 0
spanning-tree vlan 10,20,30 priority 24576
```

On access ports, `portfast` brings the port up faster and `bpduguard` shuts the port down if it receives a BPDU, which stops an unauthorized switch from joining the network.

```
interface FastEthernet0/23
 switchport access vlan 99
 switchport mode access
 spanning-tree portfast
 spanning-tree bpduguard enable
```

![STP on SW-CORE](../img/03-stp.png)

---

## 3. Frame Relay WAN

Headquarters (the hub) uses one point-to-point sub-interface per branch, each with its own DLCI:

```
interface Serial0/0/0
 encapsulation frame-relay
!
interface Serial0/0/0.101 point-to-point
 description **Link p/ R-A (Filial 1) | DLCI 101**
 ip address 200.18.40.217 255.255.255.252
 frame-relay interface-dlci 101
!
interface Serial0/0/0.102 point-to-point
 description **Link p/ R-B (Filial 2) | DLCI 102**
 ip address 200.18.40.221 255.255.255.252
 frame-relay interface-dlci 102
```

Each branch has a single sub-interface pointing back to headquarters (R-A on DLCI 101, R-B on DLCI 102).

![Frame Relay](../img/04-frame-relay.png)

---

## 4. Interior routing: OSPF, RIPv2 and redistribution

| Site | Protocol |
|---|---|
| HQ (R-M) | OSPF area 0 + RIPv2 (redistribution point) |
| Branch 1 (R-A) | OSPF area 0 |
| Branch 2 (R-B) | RIPv2 |

**R-A (Branch 1), OSPF only:**

```
router ospf 1
 router-id 1.1.1.1
 network 200.18.40.64 0.0.0.31 area 0
 network 200.18.40.184 0.0.0.7 area 0
 network 200.18.40.208 0.0.0.7 area 0
 network 200.18.40.216 0.0.0.3 area 0
```

**R-B (Branch 2), RIPv2 only.** `no auto-summary` is required with VLSM: without it, RIP would summarize subnets at the classful boundary and lose the variable-length masks.

```
router rip
 version 2
 network 200.18.40.0
 no auto-summary
```

**R-M (HQ), mutual redistribution.** Branch 2's RIP routes go into OSPF (`subnets` carries the subnets, not just classful networks), and OSPF routes go into RIP with metric 1:

```
router ospf 1
 router-id 2.2.2.2
 redistribute rip subnets
 network 200.18.40.0 0.0.0.63 area 0
 network 200.18.40.128 0.0.0.31 area 0
 network 200.18.40.160 0.0.0.15 area 0
 network 200.18.40.176 0.0.0.7 area 0
 network 200.18.40.216 0.0.0.3 area 0
!
router rip
 version 2
 redistribute ospf 1 metric 1
 network 200.18.40.0
 no auto-summary
```

![R-M routing table](../img/05-tabela-de-rotas.png)

---

## 5. BGP with the ISP

TechEdu runs **AS 40** and peers over eBGP with the ISP (**AS 65000**), advertising the aggregate block `200.18.40.0/24`. The static route to `Null0` makes sure the /24 exists in the routing table, which BGP requires before it advertises the prefix with the `network` command:

```
router bgp 40
 bgp router-id 2.2.2.2
 neighbor 200.18.40.226 remote-as 65000
 neighbor 200.18.40.226 next-hop-self
 network 200.18.40.0
!
ip route 200.18.40.0 255.255.255.0 Null0
```

On the ISP side:

```
router bgp 65000
 bgp router-id 3.3.3.3
 neighbor 200.18.40.225 remote-as 40
```

![BGP](../img/06-bgp.png)

---

## 6. Services: DHCP, DNS and Web

- **DNS and Web** run on the HQ VLAN 30 server, with static IP `200.18.40.174`. DNS has an A record `web.techedu.local → 200.18.40.174`, and the `index.html` page was edited so it could be identified during testing.
- **DHCP:** routers relay client requests with `ip helper-address` to a server at each site:

| Router | VLANs | `ip helper-address` |
|---|---|---|
| R-M | 10, 20, 99 | 200.18.40.174 |
| R-A | 10, 20, 99 | 200.18.40.214 |
| R-B | 10, 20, 99 | 200.18.40.206 |

![DNS and connectivity](../img/07-dns-ping.png)

---

## 7. Hardening

- Consistent hostnames (`R-M`, `R-A`, `R-B`, `SW-CORE`, `SW1-M`…) and interface `description`s.
- Restricted-access `banner motd` on every device.
- Console (`line con 0`), VTY (`line vty 0 15`) and `enable secret` passwords set **per site**: one for HQ devices, one for Branch 1 and one for Branch 2. The values were removed from the published configs.
- `bpduguard` on access ports.

**Limitations:** `no service password-encryption` (line passwords stay in cleartext in the config) and remote access over Telnet with a plain password, without SSH.
