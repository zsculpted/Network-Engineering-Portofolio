# Exploring BGP

I’ve completed my CCNA, but most of what I studied covered interior routing protocols like OSPF. When I started researching Junior / L1 Network Engineer roles, I kept seeing BGP, eBGP, and iBGP in the job requirements. **This lab is my way of closing that gap** by showcasing my ability to configure and verify BGP sessions, reading the BGP table, and troubleshooting when sessions or routes go missing.

## The Scenario

Company A (AS 65001) has a headquarters in Jakarta and two branch offices (Surabaya and Bandung). The HQ connects to the internet through two ISPs for redundancy. Inside the company, OSPF handles internal routing. At the edge, eBGP talks to the ISPs, and iBGP (with route reflectors) shares what the edge routers learn.

## Initial Topology

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

### IP Addressing 

#### ISP Routers

| Router | Interface | IP Address |
|---|---|---|
| ISP-Alpha | Gi0/0 | 172.16.10.1/30 |
| ISP-BETA | Gi0/0 | 172.16.20.1/30 |

#### HQ Edge Routers

| Router | Interface | IP Address |
|---|---|---|
| EDGE-1 | Gi0/0 | 172.16.10.2/30 |
| EDGE-1 | Gi0/1 | 10.1.1.1/30 |
| EDGE-1 | Gi0/2 | 10.1.2.1/30 |
| EDGE-2 | Gi0/0 | 172.16.20.2/30 |
| EDGE-2 | Gi0/1 | 10.1.3.1/30 |
| EDGE-2 | Gi0/2 | 10.1.4.1/30 |

#### HQ Route Reflectors

| Router | Interface | IP Address |
|---|---|---|
| RR1 | Gi0/0 | 10.1.1.2/30 |
| RR1 | Gi0/1 | 10.1.3.2/30 |
| RR1 | Gi0/2 | 10.1.11.1/30 |
| RR1 | Gi0/3 | 10.1.5.1/30 |
| RR2 | Gi0/0 | 10.1.2.2/30 |
| RR2 | Gi0/1 | 10.1.4.2/30 |
| RR2 | Gi0/2 | 10.1.12.1/30 |
| RR2 | Gi0/3 | 10.1.5.2/30 |

#### Branch Routers

| Router | Interface | IP Address |
|---|---|---|
| BR-SBY | Gi0/0 | 10.1.11.2/30 |
| BR-BDG | Gi0/0 | 10.1.12.2/30 |

### Routing Summary (ASN, OSPF, BGP role)

For the company A branches (BR-SBY, BR-BDG) I only run OSPF (AREA 10). The other BGP plan (iBGP and eBGP) will look like the table below.

| Router | ASN | OSPF area | BGP role | BGP neighbors |
|---|---|---|---|---|
| EDGE-1 | 65001 | 0 | Edge, RR client | eBGP: ISP-Alpha. iBGP: RR1, RR2 |
| EDGE-2 | 65001 | 0 | Edge, RR client | eBGP: ISP-BETA. iBGP: RR1, RR2 |
| RR1 | 65001 | 0 and 10 (ABR) | Route reflector | iBGP: EDGE-1, EDGE-2 (clients), RR2 |
| RR2 | 65001 | 0 and 10 (ABR) | Route reflector | iBGP: EDGE-1, EDGE-2 (clients), RR1 |
| BR-SBY | 65001 | 10 | None | None (OSPF only) |
| BR-BDG | 65001 | 10 | None | None (OSPF only) |
| ISP-Alpha | 65010 | None | External provider | eBGP: EDGE-1 |
| ISP-BETA | 65020 | None | External provider | eBGP: EDGE-2 |

## BGP Configuration

**Quick note before we dive in:** all the necessary groundwork is already done. I already completes the IP addressing, loopbacks, OSPF, and the area design so we can move straight to the BGP configuration. 

**Starting point (already working):**
- All interfaces and loopbacks are configured and reachable
- OSPF is up inside Company A (area 0 for HQ, area 10 for the branches)
- Every Company A router can ping every other Company A router's Lo0
- ISP-Alpha and ISP-BETA have their IPs configured, and nothing else yet

**What I'll configure here:**
1. eBGP between the edges and the ISPs
2. iBGP with route reflectors inside Company A
3. Prefix advertisement
4. The `next-hop-self` fix (spoiler: I expect to get this wrong the first time)

