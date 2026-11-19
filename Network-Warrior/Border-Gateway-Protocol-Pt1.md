# Exploring BGP

I’ve completed my CCNA, but most of what I studied covered interior routing protocols like OSPF. When I started researching Junior / L1 Network Engineer roles, I kept seeing BGP, eBGP, and iBGP in the job requirements. **This lab is my way of closing that gap** by showcasing my ability to configure and verify BGP sessions, reading the BGP table, and troubleshooting when sessions or routes go missing.

## The Scenario

Company A (AS 65001) has a headquarters in Jakarta and two branch offices (Surabaya and Bandung). The HQ connects to the internet through two ISPs for redundancy. Inside the company, OSPF handles internal routing. At the edge, eBGP talks to the ISPs, and iBGP (with route reflectors) shares what the edge routers learn.

## Initial Topology

<img width="657" height="593" alt="image" src="https://github.com/user-attachments/assets/19e5ed95-d3ce-4164-ad91-1543f16dee70" />

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

| Router | Interface | IP Address | OSPF Area |
|---|---|---|---|
| ISP-Alpha | Gi0/0 | 172.16.10.1/30 | None |
| ISP-BETA | Gi0/0 | 172.16.20.1/30 | None |

#### HQ Edge Routers

| Router | Interface | IP Address | OSPF Area |
|---|---|---|---|
| EDGE-1 | Gi0/0 | 172.16.10.2/30 | None (eBGP link) |
| EDGE-1 | Gi0/1 | 10.1.1.1/30 | 0 |
| EDGE-1 | Gi0/2 | 10.1.2.1/30 | 0 |
| EDGE-2 | Gi0/0 | 172.16.20.2/30 | None (eBGP link) |
| EDGE-2 | Gi0/1 | 10.1.3.1/30 | 0 |
| EDGE-2 | Gi0/2 | 10.1.4.1/30 | 0 |

#### HQ Route Reflectors

| Router | Interface | IP Address | OSPF Area |
|---|---|---|---|
| RR1 | Gi0/0 | 10.1.1.2/30 | 0 |
| RR1 | Gi0/1 | 10.1.3.2/30 | 0 |
| RR1 | Gi0/2 | 10.1.11.1/30 | 10 |
| RR1 | Gi0/3 | 10.1.5.1/30 | 0 |
| RR2 | Gi0/0 | 10.1.2.2/30 | 0 |
| RR2 | Gi0/1 | 10.1.4.2/30 | 0 |
| RR2 | Gi0/2 | 10.1.12.1/30 | 10 |
| RR2 | Gi0/3 | 10.1.5.2/30 | 0 |

#### Branch Routers

| Router | Interface | IP Address | OSPF Area |
|---|---|---|---|
| BR-SBY | Gi0/0 | 10.1.11.2/30 | 10 |
| BR-BDG | Gi0/0 | 10.1.12.2/30 | 10 |

That's all for the IP Addressing, next we're moving to the real thing, that is BGP. 

## BGP Configuration

**Quick note before we dive in:** all the necessary groundwork is already done. I already completes the IP addressing, loopbacks, OSPF, and the area design so we can move straight to the BGP configuration. 

**Starting point (already working):**
- All interfaces and loopbacks are configured and reachable
- OSPF is up inside Company A (area 0 for HQ, area 10 for the branches)
- Every Company A router can ping every other Company A router's Lo0
- ISP-Alpha and ISP-BETA have their IPs configured, and nothing else yet

<img width="928" height="176" alt="image" src="https://github.com/user-attachments/assets/11e3be8b-efcf-4f32-b914-f0281a1969a9" />

**As an example**, as you can see Surabaya Branch (BR-SBY - OSPF Area 10) were able to ping Edge 2 Gi0/2 of Jakarta's HQ (HQ-EDGE2 -  OSPF Area 0) without any problem

**What I'll configure Next:**
1. eBGP between the edges and the ISPs
2. iBGP with route reflectors inside Company A
3. Prefix advertisement
4. The `next-hop-self` fix (spoiler: I expect to get this wrong the first time)

### Step 1: eBGP Between the Edges and the ISPs

This is the first real BGP step, so I started small. I connected each HQ edge router to its ISP over the direct link between them.

For **EDGE-1**, I configured ISP-Alpha (`172.16.10.1`, AS 65010) as its neighbor. Then I did the mirror image on **ISP-Alpha**, pointing it back at EDGE-1 (`172.16.10.2`, AS 65001). I did the same for **EDGE-2 and ISP-BETA** on the `172.16.20.0/30` link. BGP neighbors aren't discovered automatically like in OSPF, so I had to configure **both sides by hand**. Because the ASNs on each side are different (65001 vs 65010 / 65020), these are **eBGP** sessions.

| Session | Router | Neighbor IP | Remote ASN |
|---|---|---|---|
| EDGE-1 ↔ ISP-Alpha | EDGE-1 | 172.16.10.1 | 65010 |
| | ISP-Alpha | 172.16.10.2 | 65001 |
| EDGE-2 ↔ ISP-BETA | EDGE-2 | 172.16.20.1 | 65020 |
| | ISP-BETA | 172.16.20.2 | 65001 |

**ISP-Alpha (AS 65010)**
```
router bgp 65010
 neighbor 172.16.10.2 remote-as 65001
```

**EDGE-1 (AS 65001)**
```
router bgp 65001
 neighbor 172.16.10.1 remote-as 65010
```

**ISP-BETA (AS 65020)**
```
router bgp 65020
 neighbor 172.16.20.2 remote-as 65001
```

**EDGE-2 (AS 65001)**
```
router bgp 65001
 neighbor 172.16.20.1 remote-as 65020
```

#### Checking that it worked

After configuring both sides, I checked the sessions with:

```
show ip bgp summary
```

**EDGE-1 ↔ ISP-Alpha**

<img width="1896" height="239" alt="image" src="https://github.com/user-attachments/assets/43b30757-25c7-4e9a-83e9-8e07964bbbe8" />

**EDGE-2 ↔ ISP-Beta**

<img width="1895" height="271" alt="image" src="https://github.com/user-attachments/assets/0e74515d-0c4b-4497-ae82-58a1ccf69670" />


I was looking at the **State/PfxRcd** column. If it says Idle or Active, the session isn't up yet. If it shows a number, the session is **Established** and that number is how many prefixes were received. I expected **0 prefixes** at this point, which **shown** on the **picture above** nobody is advertising anything yet, so an empty table here means **success**, not failure.

### Step 2: iBGP With the Route Reflectors

The edges now know about their ISPs, but the rest of Company A doesn't know BGP exists. In this step I built the **iBGP** sessions inside AS 65001 so the edges can share what they learn.

What I've learned, the thing with iBGP is that a router never passes a route learned from one iBGP peer to another iBGP peer. Without a workaround, every BGP router would need a session with every other one (a full mesh). Instead, I made **RR1 and RR2 route reflectors**, which are allowed to re-advertise iBGP routes. **EDGE-1 and EDGE-2 are their clients**, and the two reflectors peer with each other. The Surabaya and Bandung branches don't run BGP, so I left them alone.

I didn't use loopbacks for this lab because I configure everything all at the same time, and making table here or typing addresses on PnetLab would take so much time, so each session runs over the **direct link IPs** between the two routers. That works because both ends are on the same /30. Since the ASN is the same on both sides (65001), these are **iBGP** sessions.

| Session | Router | Neighbor IP | Role of neighbor |
|---|---|---|---|
| EDGE-1 ↔ RR1 | EDGE-1 | 10.1.1.2 | Route reflector |
| | RR1 | 10.1.1.1 | Client |
| EDGE-1 ↔ RR2 | EDGE-1 | 10.1.2.2 | Route reflector |
| | RR2 | 10.1.2.1 | Client |
| EDGE-2 ↔ RR1 | EDGE-2 | 10.1.3.2 | Route reflector |
| | RR1 | 10.1.3.1 | Client |
| EDGE-2 ↔ RR2 | EDGE-2 | 10.1.4.2 | Route reflector |
| | RR2 | 10.1.4.1 | Client |
| RR1 ↔ RR2 | RR1 | 10.1.5.2 | Regular iBGP peer (non-client) |
| | RR2 | 10.1.5.1 | Regular iBGP peer (non-client) |

For the edges, I pointed each one at both RRs. Both are configured as plain neighbors, because the edge doesn't need to know it's a client. Only the RR does.

**EDGE-1 (AS 65001)**
```
router bgp 65001
 neighbor 10.1.1.2 remote-as 65001
 neighbor 10.1.2.2 remote-as 65001
```

**EDGE-2 (AS 65001)**
```
router bgp 65001
 neighbor 10.1.3.2 remote-as 65001
 neighbor 10.1.4.2 remote-as 65001
```

On the route reflectors, I added the edges as **clients** with the `route-reflector-client` line. That one line is what makes a router a reflector for that neighbor.

**RR1 (AS 65001)**
```
router bgp 65001
 neighbor 10.1.1.1 remote-as 65001
 neighbor 10.1.1.1 route-reflector-client
 neighbor 10.1.3.1 remote-as 65001
 neighbor 10.1.3.1 route-reflector-client
 neighbor 10.1.5.2 remote-as 65001
```

**RR2 (AS 65001)**
```
router bgp 65001
 neighbor 10.1.2.1 remote-as 65001
 neighbor 10.1.2.1 route-reflector-client
 neighbor 10.1.4.1 remote-as 65001
 neighbor 10.1.4.1 route-reflector-client
 neighbor 10.1.5.1 remote-as 65001
```

The link between RR1 and RR2 has **no** `route-reflector-client` line on purpose. They're equals, not client and reflector.

#### Checking that it worked

```
show ip bgp summary
```

I expected to see:

| Router | Established neighbors expected |
|---|---|
| EDGE-1 | 3 (ISP-Alpha, RR1, RR2) |
| EDGE-2 | 3 (ISP-BETA, RR1, RR2) |
| RR1 | 3 (EDGE-1, EDGE-2, RR2) |
| RR2 | 3 (EDGE-1, EDGE-2, RR1) |

To confirm the reflector setup, I ran this on an RR:

```
show ip bgp neighbors 10.1.1.1 | include Route-Reflector
```

It should mention "Route-Reflector Client" for the edge neighbors. As in Step 1, **0 prefixes** is expected, because nothing is being advertised yet.

...


#### What's next

Right now every session is up but there is nothing to see. In **Step 3**, I'll advertise something into BGP so the tables stop being empty, and in **Step 4** I'll deal with the `next-hop-self` problem.
