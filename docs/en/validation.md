# Validation

[← Back to README](../../README.en.md) · [Português](../pt-br/validacao.md)

Tests were run in order, from layer 2 up to layer 7.

## 1. Connectivity between sites

### Ping from Branch 1 to headquarters

PC1 at Branch 1 (OSPF network) pings `200.18.40.3`, a PC on HQ VLAN 10: 4 packets sent, 4 received.

![Ping Branch 1 → HQ](../img/08-ping-filial1-matriz.png)

### Traceroute to the web server

`tracert web.techedu.local` from PC1 at Branch 1. The name resolves to `200.18.40.174`, and the path goes through Branch 1's gateway (`200.18.40.65`), across the Frame Relay link to R-M (`200.18.40.217`) and on to the server:

![Tracert Branch 1 → server](../img/09-tracert-filial1-web.png)

### DNS

Ping by name to `web.techedu.local`, resolved by the HQ DNS server to `200.18.40.174`:

![DNS](../img/07-dns-ping.png)

## 2. Routing

### R-M routing table

R-M learns Branch 1's networks through OSPF (`O`, via `200.18.40.218`) and Branch 2's through RIP (`R`, via `200.18.40.222`). The static route to `Null0` for the /24 aggregate used by BGP also shows up.

![R-M routing table](../img/05-tabela-de-rotas.png)

### `show ip protocols`

**R-A:** OSPF process 1, router-id `1.1.1.1`, with Branch 1's networks and the WAN link in area 0. Routing information sources are `1.1.1.1` and `2.2.2.2` (R-M).

![show ip protocols on R-A](../img/10-ip-protocols-r-a.png)

**R-M:** OSPF, RIP and BGP running at the same time, with redistribution between OSPF and RIP.

![show ip protocols on R-M](../img/11-ip-protocols-r-m.png)

## 3. BGP

`show ip bgp summary` on R-M: router-id `2.2.2.2`, local AS 40, neighbor `200.18.40.226` in AS 65000 with the session established.

![show ip bgp summary](../img/12-bgp-summary-r-m.png)

## 4. Frame Relay

`show frame-relay map` on R-M: both PVCs, DLCI 101 and DLCI 102, show as `active` on the point-to-point sub-interfaces.

![show frame-relay map](../img/13-frame-relay-map.png)
