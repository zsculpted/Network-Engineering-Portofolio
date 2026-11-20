# Exploring and Building BGP

I’ve completed my CCNA, but most of what I studied covered interior routing protocols like OSPF. When I started researching Junior / L1 Network Engineer roles, I kept seeing BGP, eBGP, and iBGP in the job requirements. **This lab is my way of closing that gap** by showcasing my ability to configure and verify BGP sessions, reading the BGP table, and troubleshooting when sessions or routes go missing.

## The Scenario

**_Company A (AS 65001)_** has a headquarters in Jakarta and two branch offices (Surabaya and Bandung). The HQ connects to the internet through two ISPs for redundancy. Inside the company, OSPF handles internal routing. At the edge, eBGP talks to the ISPs, and iBGP (with route reflectors) shares what the edge routers learn.

## Initial Topology

<img width="1280" height="1222" alt="image" src="https://github.com/user-attachments/assets/f7aeff68-ab5c-438d-af98-050abb860877" />

**This is how i put it on lab environemnt.** 
This topology will evolve as the lab grows and will pretty much changes over the course. 

<img width="628" height="622" alt="Screenshot 2026-11-18 211819" src="https://github.com/user-attachments/assets/f2fbc94b-8452-4017-bff3-dc903c957f57" />

### Autonomous System Numbers

I'm using the **private 16-bit ASN range (64512 to 65534)**.

| AS | ASN | Routers |
|---|---|---|
| Company A | **65001** | HQ EDGE-1, HQ EDGE-2, HQ RR1, HQ RR2, BR-SBY, BR-BDG |
| ISP-Alpha | **65010** | ISP-Alpha |
| ISP-BETA | **65020** | ISP-BETA |
| INTERNET *(planned)* | **65100** | INTERNET |

### IP Addressing

<img width="862" height="623" alt="image" src="https://github.com/user-attachments/assets/268bbd14-74bf-451b-93cc-5cec36587cd3" />


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

This is why I feel like it's only better if I add loopback address, something that I skip at the beginning.

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

<img width="835" height="742" alt="image" src="https://github.com/user-attachments/assets/9f074e6b-cfb5-4696-bfc5-2304b94b6647" />

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


#### 3b. Advertising Company A's Public Block

<img width="847" height="741" alt="image" src="https://github.com/user-attachments/assets/e78fa1fd-f635-4a42-bf55-9097161d81e8" />

At first I was confused about how a router can announce a prefix when the interface isn't "in BGP." The thing I learned is that BGP doesn't run on interfaces. Rather, we have to configure the network prefix on our routing table to our BGP neighbor.

**Where a real company's public block comes from.** I also had the wrong idea about what a company announces. Nobody announces an internal router like loopback adressess to an ISP. A company that needs BGP gets its **own public block** (and its own ASN) from the internet registries. The smallest block the internet accepts is a **/24**, because ISPs filter anything longer, that also something I just discovered.

**For this lab I'm using `203.0.113.0/24` as Company A's block.**

**Where the block actually lives.** This surprised me: the block is *not* configured on the interface facing the ISP. That link keeps the small /30 the ISP gave me, and it only carries the BGP session. 

| What | Real network | My lab |
|---|---|---|
| Who announces the `/24` to the ISPs | The edge routers | EDGE-1 (to ISP-Alpha) and EDGE-2 (to ISP-BETA) |
| Where the block's addresses are used | A firewall's DMZ interface, plus the servers behind it | RR1 Gi0/4 (`203.0.113.1/28`) and DMZ-SRV (`203.0.113.10/28`) |
| Unused addresses in the block | Dropped by a Null0 route on the edges | Dropped by a Null0 route on both edges |
| Firewall between the edges and the DMZ | Yes | Not included, to keep the lab small |

- My internal routers (the RRs and the branches) keep their private `10.x.x.x` addresses. The edges don't hold the block on any interface. They just **announce** the whole `/24`.
- **How I mapped that to my lab.** I don't have a firewall, so I built a small DMZ: one server device (**DMZ-SRV**) connected to RR1 on a new interface, using the first piece of the block.

| Device | Interface | IP Address | Notes |
|---|---|---|---|
| RR1 | Gi0/4 | 203.0.113.1/28 | DMZ gateway |
| DMZ-SRV | Gi0/0 | 203.0.113.10/28 | Default gateway: 203.0.113.1 |

The `203.0.113.0/28` subnet is my DMZ, and the rest of the `/24` is spare.
_
I skipped the firewall to keep the lab small, and RR1 is also carrying DMZ traffic, though a real route reflector wouldn't. Just to keep this simple._

**The Null0 route.** Each edge has a static route for the whole `/24` pointing at **Null0**, a built-in discard interface. The reason:

1. **It lets BGP announce the block.** The edges will learn `203.0.113.0/28` through RR1 vi OSPF, but they don't have the whole `/24`, so the Null0 route is what creates it.
2. **It drops traffic for unused addresses.** In a real network, traffic for an address nobody uses would otherwise follow the default route back to the ISP, and the ISP would send it right back, **looping** until the TTL runs out, and we don't want that.

I first worried that this would also drop traffic for my real server, turns out it doesn't, because routers always pick the **longest matching prefix**, not the first or lowest-AD route.

##### Step 3b Configuration

**RR1 (AS 65001)**
```
interface GigabitEthernet0/4
 description To DMZ-SRV
 ip address 203.0.113.1 255.255.255.240
 ip ospf 1 area 50
 no shutdown
```

**DMZ-SRV**
```
interface GigabitEthernet0/0
 ip address 203.0.113.10 255.255.255.240
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

**EDGE-1 (AS 65001)**
```
ip route 203.0.113.0 255.255.255.0 Null0
!
router bgp 65001
 network 203.0.113.0 mask 255.255.255.0
!
ip route 203.0.113.0 255.255.255.0 Null0
```

**EDGE-2 (AS 65001)**
```
ip route 203.0.113.0 255.255.255.0 Null0
!
router bgp 65001
 network 203.0.113.0 mask 255.255.255.0
!
ip route 203.0.113.0 255.255.255.0 Null0
```

DMZ-SRV doesn't run OSPF. RR1 advertises the connected `/28` into area 0 for it, which is all the edges need. INTERNET already announces `8.8.8.8/32` and `9.9.9.9/32` from step 3a.

| Router | Prefix | What it represents |
|---|---|---|
| EDGE-1 | 203.0.113.0/24 | Company A's public block, announced to ISP-Alpha |
| EDGE-2 | 203.0.113.0/24 | Company A's public block, announced to ISP-BETA |
| INTERNET | 8.8.8.8/32 and 9.9.9.9/32 | The rest of the internet |

Both edges announce the **same** block. That's what **multihoming** means: the internet can reach Company A through either ISP.

#### Checking that it worked

First, the inside of Company A. On both edges:

```
show ip route 203.0.113.10
show ip route 203.0.113.99
```
`203.0.113.99` is a made-up address that no device in my lab uses, and I picked it only as a test to see if the route is correct.

<img width="937" height="383" alt="image" src="https://github.com/user-attachments/assets/61d072f7-c002-4f98-84f2-0f5579225c23" />

<img width="950" height="378" alt="image" src="https://github.com/user-attachments/assets/ef4ec6a5-4514-4710-b74d-6c6427b9f180" />

As it shown above, `203.0.113.10` is match with the `/28` learned through OSPF (`O`), pointing at RR1. That's the **specific** route to my server. Meanwhile `203.0.113.99` is match only with the static route to **Null0**, which proves unused addresses in the block get dropped.

Then BGP:

```
show ip bgp
show ip bgp summary
```
**Both Edges**
<img width="1919" height="495" alt="image" src="https://github.com/user-attachments/assets/06440e6d-3c24-4ab3-b4ff-b878ae2bbd89" />
**Both HQ-RR**
<img width="1919" height="496" alt="image" src="https://github.com/user-attachments/assets/58ab6dbc-2b02-4383-b64e-89c7b07d50a7" />
**Both ISP**
<img width="1895" height="508" alt="image" src="https://github.com/user-attachments/assets/6eced275-5dd7-4ea0-94bf-726b79a26af3" />
**Internet**

<img width="937" height="502" alt="image" src="https://github.com/user-attachments/assets/cd3750e5-41e1-4b51-945d-9772039925d5" />

### Step 4: Troubleshooting 1 - Fixing the next hop

I saw this problem on RR1 before I fixed it. `show ip bgp` listed `8.8.8.8/32` and `9.9.9.9/32` with `* i` in front, but **no `>`**, and the next hop was `172.16.10.1`, ISP-Alpha's address. The AS_PATH (`65010 65100`) was right, but the route wasn't marked best.

<img width="942" height="280" alt="image" src="https://github.com/user-attachments/assets/3724d6b8-1382-4ec1-93c3-8fe3d1afc776" />

This problem happened because I deliberately kept the ISP links (`172.16.x.x`) **out of OSPF**. Here's what went wrong:

1. **EDGE-1 learns the route from ISP-Alpha.** ISP-Alpha tells EDGE-1, "to reach `8.8.8.8`, send traffic to me at `172.16.10.1`." So the route's **next hop** (the next router to send traffic to) is `172.16.10.1`.
2. **EDGE-1 passes the route to RR1 over iBGP.** By default, iBGP does **not** change the next hop, so RR1 receives the route still saying "next hop: `172.16.10.1`."
3. **RR1 checks whether it can reach that next hop.** `172.16.10.0/30` isn't in OSPF, so RR1 has no route to `172.16.10.1`. It can't use the route, so it never marks it best (no `>`).

```
ISP-Alpha ──────── EDGE-1 ──────── RR1
172.16.10.1       (knows ISP)    "How do I reach 172.16.10.1?
                                  I have no idea."  ✗
```
#### Troubleshooting process

**The correct one I believe** is for EDGE-1 to say "send it to **me** instead," and I can reach EDGE-1 because its loopback is in OSPF.

The solution is `next-hop-self` on the edges. It tells the edge to put **itself** as the next hop when it advertises to its iBGP neighbors, and the RRs already know how to reach the edges' loopbacks through OSPF.

**EDGE-1 (AS 65001)**
```
router bgp 65001
 neighbor 3.3.3.3 next-hop-self
 neighbor 4.4.4.4 next-hop-self
```

**EDGE-2 (AS 65001)**
```
router bgp 65001
 neighbor 3.3.3.3 next-hop-self
 neighbor 4.4.4.4 next-hop-self
```

**A problem occured on Edge-2**

<img width="799" height="395" alt="image" src="https://github.com/user-attachments/assets/77368142-b512-4872-bc11-59a5b06a8c32" />

After seeing this, I assume I missed to add 3.3.3.3 as BGP neighbor. So i did `sh run | sect router bgp`

<img width="932" height="159" alt="image" src="https://github.com/user-attachments/assets/33275d7c-db5b-434c-95d9-e3f72688df99" />

My assumption was correct. Maybe because I use pnetlab and forgot to save the running config that's why it got removed. 

After adding everything you can see at the very bottom the log for adj appeared. That also explains why RR1 only showed one path per prefix, via 172.16.10.1, because EDGE-2 was never talking to the route reflectors.

<img width="938" height="457" alt="image" src="https://github.com/user-attachments/assets/fcf241f3-0738-4c6c-bba6-e0cf191f2846" />

I also did the `show` command on HQ-RR1 just in case every router config is not saved, luckly it doesnt.

<img width="929" height="262" alt="image" src="https://github.com/user-attachments/assets/63413ec1-b11e-498a-b4ec-c9a1e54c9cb8" />



To push the update without waiting:
```
clear ip bgp * soft
```

#### Checking that it worked

```
show ip bgp
show ip route bgp
```

<img width="934" height="423" alt="image" src="https://github.com/user-attachments/assets/ce8657fb-6393-4a64-9727-8c59dca2e9c5" />

<img width="928" height="423" alt="image" src="https://github.com/user-attachments/assets/c4d2c2a7-6e9f-4930-8217-d8e72722076d" />


Both RR1 and RR2, the internet routes now had `*>i`, and the next hop had changed from the ISP's address to the edge's loopback (`1.1.1.1` or `2.2.2.2`). They also showed up in `show ip route bgp`, which means they're actually usable. `203.0.113.0/24` was in the table too, coming from the edges.


#### The end-to-end test

With the next hop fixed, I ran the test I couldn't run before, from my DMZ server:

```
ping 8.8.8.8
ping 9.9.9.9
```

<img width="921" height="244" alt="image" src="https://github.com/user-attachments/assets/f2259a8f-d447-4bfa-9947-f143d4e72f42" />

## It worked

he packet went DMZ-SRV → RR1 → an edge → an ISP → INTERNET, and the reply came back because INTERNET learned `203.0.113.0/24` through BGP from both ISPs. Inside Company A, the edges know the specific route to my server (`203.0.113.0/28` through OSPF), so the reply finds its way back to DMZ-SRV.

### Step 5: Choosing Between the ISPs and Testing Failover

Now that both ISPs lead to the same destinations, Company A has two paths to `8.8.8.8` and `9.9.9.9`. This is the BGP concept that made me want to build the lab.

#### What happens by default

With no tuning, each edge tends to use the path through **its own ISP**. Both paths have the same Local Preference (100) and the same AS_PATH length (`65010 65100` vs `65020 65100`), so BGP falls to a tie-breaker: a route learned over eBGP beats one learned over iBGP.

#### Making ISP-Alpha the preferred exit with Local Preference

I wanted all of Company A to exit through ISP-Alpha, with ISP-BETA as the backup. At first when I did traceroute 8.8.8.8 on DMZ SRV, it seems like ISP-BETA is the prefered route.

<img width="933" height="202" alt="image" src="https://github.com/user-attachments/assets/4655a99c-8705-467d-b007-9942864e26cc" />

To change that I used **Local Preference**: a higher number wins, the default is 100, and it's shared with every iBGP router, so the whole AS agrees on the exit. I set it to 200 on the routes EDGE-1 learns from ISP-Alpha.

**EDGE-1 (AS 65001)**
```
route-map PREFER-ALPHA permit 10
 set local-preference 200
!
router bgp 65001
 neighbor 172.16.10.1 route-map PREFER-ALPHA in
```
```
clear ip bgp 172.16.10.1 soft in
```

I did traceroute on DMZ-SRV as well as `show ip bgp 8.8.8.8` on EDGE-2, now it showed the best path going through EDGE-1 (next hop `1.1.1.1`) with local preference 200, even though EDGE-2 has its own direct path to ISP-BETA. Local Preference is checked before the "prefer eBGP" tie-breaker, so it wins.

<img width="931" height="188" alt="image" src="https://github.com/user-attachments/assets/542edcdb-9e2c-4d90-83d0-7dec75852d02" />

<img width="931" height="490" alt="image" src="https://github.com/user-attachments/assets/04ea67ec-53f2-4b00-a054-262645c4fdd2" />



#### Failover test and troubleshooting scenario

To simulate ISP-Alpha going down, I shut the link on EDGE-1:

```
interface GigabitEthernet0/0
 shutdown
```

The eBGP session to ISP-Alpha dropped, the routes were withdrawn, and `show ip bgp 8.8.8.8` on both edges switched to the ISP-BETA path. While this was happening, I kept pinging `8.8.8.8` from DMZ-SRV. It lost a few packets during the switch, then recovered on its own. Then I ran `no shutdown` and watched everything move back to ISP-Alpha. I didn't change any configuration to make it happen, and that automatic failover is the whole reason BGP and multihoming exist.

One thing I noticed: Local Preference only controls **outbound** traffic, meaning how Company A leaves. How the internet reaches **me** is decided by INTERNET and the ISPs. Influencing that needs something like AS_PATH prepending, which is on my list below.

<!-- Paste output from before, during, and after the shutdown here -->

---

## Troubleshooting Scenarios

I learned an L1 engineer spends more time noticing and fixing BGP rather than building it. So next time I want to create scenario and broke some connection on purpose to try to the fix. 

_to be continued_
