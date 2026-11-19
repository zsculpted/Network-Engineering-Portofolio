# Exploring BGP

I’ve completed my CCNA, but most of what I studied covered interior routing protocols like OSPF. When I started researching Junior / L1 Network Engineer roles, I kept seeing BGP, eBGP, and iBGP in the job requirements. **This lab is my way of closing that gap** by showcasing my ability to configure and verify BGP sessions, reading the BGP table, and troubleshooting when sessions or routes go missing.

## The scenario

Company A (AS 65001) has a headquarters in Jakarta and two branch offices (Surabaya and Bandung). The HQ connects to the internet through two ISPs for redundancy. Inside the company, OSPF handles internal routing. At the edge, eBGP talks to the ISPs, and iBGP (with route reflectors) shares what the edge routers learn.

## Initial topology

<img width="660" height="590" alt="image" src="https://github.com/user-attachments/assets/42f1906a-a187-4ab6-bcf0-5a2e89db9c24" />

This topology will evolve as the lab grows. 

### Autonomous System Numbers

I'm using the **private 16-bit ASN range (64512 to 65534)**.

| AS | ASN | Routers |
|---|---|---|
| Company A | **65001** | HQ EDGE-1, HQ EDGE-2, HQ RR1, HQ RR2, BR-SBY, BR-BDG |
| ISP-Alpha | **65010** | ISP-Alpha |
| ISP-BETA | **65020** | ISP-BETA |
| INTERNET *(planned)* | **65100** | INTERNET |

### IP Addressing Plan

Three address pools, so I can tell at a glance what an IP belongs to:

| Pool | Used for |
|---|---|
| `10.1.0.0/16` | Company A internal links (/30s) |
| `10.255.0.0/24` | Company A loopbacks (/32s): router IDs and iBGP peering |
| `172.16.0.0/16` | ISP-side links and ISP loopbacks (everything outside Company A) |

Every link is a /30. The lower address (.1) goes to the "upstream" side.

#### Link Addressing

| # | Device A | Int A | IP A | Device B | Int B | IP B | Subnet | Protocol |
|---|---|---|---|---|---|---|---|---|
| 1 | ISP-Alpha | Gi0/0 | 172.16.10.1/30 | EDGE-1 | Gi0/0 | 172.16.10.2/30 | `172.16.10.0/30` | eBGP |
| 2 | ISP-BETA | Gi0/0 | 172.16.20.1/30 | EDGE-2 | Gi0/0 | 172.16.20.2/30 | `172.16.20.0/30` | eBGP |
| 3 | EDGE-1 | Gi0/1 | 10.1.1.1/30 | RR1 | Gi0/0 | 10.1.1.2/30 | `10.1.1.0/30` | OSPF |
| 4 | EDGE-1 | Gi0/2 | 10.1.2.1/30 | RR2 | Gi0/0 | 10.1.2.2/30 | `10.1.2.0/30` | OSPF |
| 5 | EDGE-2 | Gi0/1 | 10.1.3.1/30 | RR1 | Gi0/1 | 10.1.3.2/30 | `10.1.3.0/30` | OSPF |
| 6 | EDGE-2 | Gi0/2 | 10.1.4.1/30 | RR2 | Gi0/1 | 10.1.4.2/30 | `10.1.4.0/30` | OSPF |
| 7 | RR1 | Gi0/3 | 10.1.5.1/30 | RR2 | Gi0/3 | 10.1.5.2/30 | `10.1.5.0/30` | OSPF |
| 8 | RR1 | Gi0/2 | 10.1.11.1/30 | BR-SBY | Gi0/0 | 10.1.11.2/30 | `10.1.11.0/30` | OSPF |
| 9 | RR2 | Gi0/2 | 10.1.12.1/30 | BR-BDG | Gi0/0 | 10.1.12.2/30 | `10.1.12.0/30` | OSPF |

### Routing Summary (ASN, OSPF, BGP role)

For the company A branches (BR-SBY, BR-BDG) I only run OSPF (AREA 10). The other BGP plan (iBGP and eBGP) will look like the table below.

| Router | ASN | Router ID (Lo0) | OSPF area | BGP role | BGP neighbors |
|---|---|---|---|---|---|
| EDGE-1 | 65001 | 10.255.0.1 | 0 | Edge, RR client | eBGP: ISP-Alpha. iBGP: RR1, RR2 |
| EDGE-2 | 65001 | 10.255.0.2 | 0 | Edge, RR client | eBGP: ISP-BETA. iBGP: RR1, RR2 |
| RR1 | 65001 | 10.255.0.11 | 0 and 10 (ABR) | Route reflector | iBGP: EDGE-1, EDGE-2 (clients), RR2 |
| RR2 | 65001 | 10.255.0.12 | 0 and 10 (ABR) | Route reflector | iBGP: EDGE-1, EDGE-2 (clients), RR1 |
| BR-SBY | 65001 | 10.255.0.21 | 10 | None | None (OSPF only) |
| BR-BDG | 65001 | 10.255.0.22 | 10 | None | None (OSPF only) |
| ISP-Alpha | 65010 | 172.16.255.10 | None | External provider | eBGP: EDGE-1 |
| ISP-BETA | 65020 | 172.16.255.20 | None | External provider | eBGP: EDGE-2 |

## BGP Configuration

