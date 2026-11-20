# Exploring BGP

I’ve completed my CCNA, but most of what I studied covered interior routing protocols like OSPF. When I started researching Junior / L1 Network Engineer roles, I kept seeing BGP, eBGP, and iBGP in the job requirements. **This lab is my way of closing that gap** by showcasing my ability to configure and verify BGP sessions, reading the BGP table, and troubleshooting when sessions or routes go missing.

## The Scenario

**_Company A (AS 65001)_** has a headquarters in Jakarta and two branch offices (Surabaya and Bandung). The HQ connects to the internet through two ISPs for redundancy. Inside the company, OSPF handles internal routing. At the edge, eBGP talks to the ISPs, and iBGP (with route reflectors) shares what the edge routers learn.

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

**Quick note:** all the necessary groundwork is already done. I already completes the IP addressing, loopbacks, OSPF, and the area design so we can move straight to the BGP configuration. 

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

### *Update: Adding Loopbacks

In my first plan I left loopbacks out because I thought they'd take extra time and wouldn't add much. I changed my mind, because iBGP is normally built **between loopbacks** rather than link IPs, and I just remember If I peer on a link IP and that single cable fails, the iBGP session dies with it. 

Good thing I quickly recall this before I start the iBGP prcess so I can just add loopbakcs before continueing my work. Btw I added a loopback only on the four routers that run iBGP (EDGE-1, EDGE-2, RR1, RR2) to save some time (_obviously in real job I'd add loopbakcs on every router to protect the connection of the routing protocols_).

#### Loopback Addressing

| Router | Interface | IP Address | OSPF Area |
|---|---|---|---|
| EDGE-1 | Lo0 | 1.1.1.1/32 | 0 |
| EDGE-2 | Lo0 | 2.2.2.2/32 | 0 |
| RR1 | Lo0 | 3.3.3.3/32 | 0 |
| RR2 | Lo0 | 4.4.4.4/32 | 0 |

 
Before configuring BGP, I confirmed that the loopbacks could reach each other and it sure did. 

```
ping 4.4.4.4 source Loopback0
```

As you can see in the picture above Lo0 from HQ-EDGE1 is successfully able to ping RR2 Lo0.

<img width="931" height="166" alt="image" src="https://github.com/user-attachments/assets/6bf8237c-8dc8-4752-88b2-d826935f56ea" />

### Step 2: Configuring iBGP With the Route Reflectors

The edges now know about their ISPs, but the rest of Company A doesn't know BGP exists. In this step I built the **iBGP** sessions inside AS 65001 so the edges can share what they learn.

What I've learned, the thing with iBGP is that a router never passes a route learned from one iBGP peer to another iBGP peer. Without a workaround, every BGP router would need a session with every other one (a full mesh). Instead, I made **RR1 and RR2 route reflectors**, which are allowed to **re-advertise** iBGP routes. **EDGE-1 and EDGE-2 are their clients**, and the two reflectors peer with each other. The **Surabaya and Bandung** branches don't run BGP, so I left them alone.

**Here's the full list of iBGP sessions in Company A.**

| Router | Neighbor | Neighbor IP | Neighbor's role |
|---|---|---|---|
| EDGE-1 | RR1 | 3.3.3.3 | Route reflector |
| EDGE-1 | RR2 | 4.4.4.4 | Route reflector |
| EDGE-2 | RR1 | 3.3.3.3 | Route reflector |
| EDGE-2 | RR2 | 4.4.4.4 | Route reflector |
| RR1 | EDGE-1 | 1.1.1.1 | Client |
| RR1 | EDGE-2 | 2.2.2.2 | Client |
| RR1 | RR2 | 4.4.4.4 | Regular iBGP peer (non-client) |
| RR2 | EDGE-1 | 1.1.1.1 | Client |
| RR2 | EDGE-2 | 2.2.2.2 | Client |
| RR2 | RR1 | 3.3.3.3 | Regular iBGP peer (non-client) |

#### iBGP Configurations

**EDGE-1 (AS 65001)**
```
router bgp 65001
 neighbor 3.3.3.3 remote-as 65001
 neighbor 3.3.3.3 update-source Loopback0
 neighbor 4.4.4.4 remote-as 65001
 neighbor 4.4.4.4 update-source Loopback0
```

**EDGE-2 (AS 65001)**
```
router bgp 65001
 neighbor 3.3.3.3 remote-as 65001
 neighbor 3.3.3.3 update-source Loopback0
 neighbor 4.4.4.4 remote-as 65001
 neighbor 4.4.4.4 update-source Loopback0
```

On the route reflectors, I added the edges as **clients** with the `route-reflector-client` line. That one line is what makes a router a reflector for that neighbor.

**RR1 (AS 65001)**
```
router bgp 65001
 neighbor 1.1.1.1 remote-as 65001
 neighbor 1.1.1.1 update-source Loopback0
 neighbor 1.1.1.1 route-reflector-client
 neighbor 2.2.2.2 remote-as 65001
 neighbor 2.2.2.2 update-source Loopback0
 neighbor 2.2.2.2 route-reflector-client
 neighbor 4.4.4.4 remote-as 65001
 neighbor 4.4.4.4 update-source Loopback0
```

**RR2 (AS 65001)**
```
router bgp 65001
 neighbor 1.1.1.1 remote-as 65001
 neighbor 1.1.1.1 update-source Loopback0
 neighbor 1.1.1.1 route-reflector-client
 neighbor 2.2.2.2 remote-as 65001
 neighbor 2.2.2.2 update-source Loopback0
 neighbor 2.2.2.2 route-reflector-client
 neighbor 3.3.3.3 remote-as 65001
 neighbor 3.3.3.3 update-source Loopback0
```

The link between RR1 and RR2 has **no** `route-reflector-client` line on purpose. They're equals, not client and reflector.

#### Checking that it worked

```
show ip bgp summary
```

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/fbed780f-dc9e-4d18-a163-05b569d81235" />

As you can see in the BGP neighbor table summary up there, every connection is established.

Also to confirm the reflector setup, I ran this on an RR:

```
show run| sec Route-Reflector
```

<img width="961" height="386" alt="image" src="https://github.com/user-attachments/assets/1622d8b7-81ba-4b74-9675-6d4a3b01730c" />

I did that to confirm that borth edge router is `route-reflector-client`.

### Step 3: Simulating the Internet and Advertising Prefixes

After Step 2, every BGP session was sstablished but all the tables were empty. I needed something to advertise so I add one more router to 'mimic' the internet.

#### 3a. Adding the internet side

I added one router, **INTERNET (AS 65100)**, connected to both ISPs. To make it feel like the real internet, I gave it two loopbacks with famous-looking addresses, `8.8.8.8` and `9.9.9.9`. Initially I was plan to use `1.1.1.1` but I realise it is the EDGE-1's loopback0 address, using it would devinetly become a serous routing problem.

The link between each ISP and INTERNET is a point-to-point /30, and I ran **eBGP** over it, the same way a real ISP connects to an upstream provider. It also gives me realistic AS_PATHs.

<img width="683" height="744" alt="image" src="https://github.com/user-attachments/assets/6f7bcbd6-18a1-4016-9440-e888df0f5b43" />

| Link | Device A | IP A | Device B | IP B | Subnet | Protocol |
|---|---|---|---|---|---|---|
| ISP-Alpha ↔ INTERNET | ISP-Alpha Gi0/1 | 172.16.100.1/30 | INTERNET Gi0/0 | 172.16.100.2/30 | `172.16.100.0/30` | eBGP |
| ISP-BETA ↔ INTERNET | ISP-BETA Gi0/1 | 172.16.101.1/30 | INTERNET Gi0/1 | 172.16.101.2/30 | `172.16.101.0/30` | eBGP |

| Router | Interface | IP Address | What it pretends to be |
|---|---|---|---|
| INTERNET | Lo8 | 8.8.8.8/24 | A well-known service out on the internet |
| INTERNET | Lo9 | 9.9.9.9/24 | Another service out on the internet |

**ISP-Alpha (AS 65010)**
```
interface GigabitEthernet0/1
 ip address 172.16.100.1 255.255.255.252
 no shutdown
!
router bgp 65010
 neighbor 172.16.100.2 remote-as 65100
```

**ISP-BETA (AS 65020)**
```
interface GigabitEthernet0/1
 ip address 172.16.101.1 255.255.255.252
 no shutdown
!
router bgp 65020
 neighbor 172.16.101.2 remote-as 65100
```

**INTERNET (AS 65100)**
```
interface GigabitEthernet0/0
 ip address 172.16.100.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 ip address 172.16.101.2 255.255.255.252
 no shutdown
!
interface Loopback8
 ip address 8.8.8.8 255.255.255.255
!
interface Loopback9
 ip address 9.9.9.9 255.255.255.255
!
router bgp 65100
 neighbor 172.16.100.1 remote-as 65010
 neighbor 172.16.101.1 remote-as 65020
 network 8.8.8.8 mask 255.255.255.255
 network 9.9.9.9 mask 255.255.255.255
```

I checked `show ip bgp summary` on each ISP and on INTERNET, and every session came up established.

<img width="1899" height="1016" alt="image" src="https://github.com/user-attachments/assets/1b75ca65-5491-4e3e-a822-d25c3fc797a3" />


#### 3b. Advertising Company A's public block

At first I was confused about how a router can announce a prefix when the interface isn't "in BGP." The thing I learned is that BGP doesn't run on interfaces. The `network` command just says: *if this prefix is in my routing table, put it in my BGP table and announce it to my neighbors.* That's how INTERNET announces its loopback networks, since each loopback is a connected route.

Company A needs something to announce too, and it should be its **public block**, not an internal loopback. Nobody announces internal router loopbacks to an ISP, and ISPs filter anything longer than a /24 anyway. I used a **static route to Null0** as the anchor. Null0 is a discard interface, so the route exists without any real device behind it, and real companies do this when they announce a block.

_to be contd_
