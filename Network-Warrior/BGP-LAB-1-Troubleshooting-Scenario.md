# BGP Lab: Path Selection, Failover, and Troubleshooting

*This is a companion page to my dual-ISP BGP lab. Quick recap: Company A (AS 65001) connects to two ISPs (ISP-Alpha AS 65010, ISP-BETA AS 65020) through two edge routers, EDGE-1 and EDGE-2. Inside, OSPF carries internal routes and iBGP with two route reflectors (RR1 and RR2) shares what the edges learn. A simulated internet router (AS 65100) announces `8.8.8.8/32` and `9.9.9.9/32`, and my public block `203.0.113.0/24` is announced by both edges. A DMZ server (`203.0.113.10`) hangs off RR1.*

[See this page to see how I build this BGP Lab](Network-Warrior/Border-Gateway-Protocol-LAB-1.md)

<img width="847" height="741" alt="Screenshot 2026-11-20 153352" src="https://github.com/user-attachments/assets/8f5e4dde-98a0-41b5-bcdf-e346a2cdbe6e" />


## Step 5: Choosing Between the ISPs and Testing Failover

Company A now has two paths to the same destinations, one through each ISP. This is the BGP concept that made me want to build the lab.

### What happens by default

With no tuning, both paths to `8.8.8.8` look equal to BGP: the same Local Preference (100) and the same AS_PATH length (`65010 65100` vs `65020 65100`). BGP falls through to a later tie-breaker and picks one. I tested it with a traceroute from my DMZ server:

```
traceroute 8.8.8.8
```

| Hop | Device |
|---|---|
| 1 | RR1 (`203.0.113.1`) |
| 2 | EDGE-2 (`10.1.3.1`) |
| 3 | ISP-BETA (`172.16.20.1`) |
| 4 | INTERNET (`172.16.101.2`) |

The traffic left through **EDGE-2 and ISP-BETA**, even though ISP-Alpha was also available. I had no say in the choice, which is the problem with leaving it to the tie-breakers.

<!-- Paste the first traceroute screenshot here -->

### Choosing ISP-Alpha as the preferred exit with Local Preference

I wanted all of Company A to exit through ISP-Alpha, with ISP-BETA as the backup. For that I used **Local Preference**: a higher number wins, the default is 100, and it's shared with every iBGP router, so the whole AS agrees on the exit. It's also checked **early** in BGP's decision process, before the tie-breakers that picked EDGE-2. I set it to 200 on the routes EDGE-1 learns from ISP-Alpha.

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

The `clear ... soft in` matters. A route-map change does nothing until the routes are re-processed.

### Checking that it worked

On RR1:

```
show ip bgp 8.8.8.8
```

The path via `1.1.1.1` (EDGE-1) should show `LocPrf 200` and the best-path marker, and the path via `2.2.2.2` should still show 100.

<!-- Paste `show ip bgp 8.8.8.8` from RR1 here -->

Then I ran the same traceroute again:

| Hop | Before | After |
|---|---|---|
| 1 | RR1 (`203.0.113.1`) | RR1 (`203.0.113.1`) |
| 2 | EDGE-2 (`10.1.3.1`) | **EDGE-1** (`10.1.1.1`) |
| 3 | ISP-BETA (`172.16.20.1`) | **ISP-Alpha** (`172.16.10.1`) |
| 4 | INTERNET (`172.16.101.2`) | INTERNET (`172.16.100.2`) |

The exit moved from EDGE-2 and ISP-BETA to EDGE-1 and ISP-Alpha, and the only thing I changed was Local Preference. EDGE-2 has its own direct path to ISP-BETA, but it forwards through EDGE-1 anyway, because Local Preference is checked before the "prefer eBGP over iBGP" rule.

<!-- Paste the second traceroute screenshot here -->

### Failover test

To simulate ISP-Alpha going down, I started a continuous ping from the DMZ server and shut the link on EDGE-1:

```
DMZ-SRV#ping 8.8.8.8 repeat 1000
```
```
EDGE-1(config)#interface GigabitEthernet0/0
EDGE-1(config-if)#shutdown
```

What I expected, and then checked:
1. The eBGP session to ISP-Alpha drops and EDGE-1 withdraws its routes.
2. `show ip bgp 8.8.8.8` on RR1 now shows only the path via `2.2.2.2` (ISP-BETA).
3. The ping loses a few packets, then recovers by itself.
4. A new traceroute goes through EDGE-2 and ISP-BETA.

Then `no shutdown` on the interface, and the traffic should move back to ISP-Alpha because of the higher Local Preference. I didn't change any configuration to make this happen, and that automatic failover is the whole reason BGP and multihoming exist.

<!-- Paste the ping output, `show ip bgp 8.8.8.8` during the failure, and the traceroute after failover here -->

### Two things I learned about this

- **Local Preference only controls outbound traffic.** It decides how Company A leaves. How the internet reaches me is decided by INTERNET and the ISPs, so the reply may come back through a different ISP. That's called asymmetric routing, and it's normal. To see the return path, I ran `traceroute 203.0.113.10` on INTERNET. Influencing inbound traffic needs something like AS_PATH prepending, which is on my list to try.
- **How fast the failover happens depends on how the link fails.** Shutting the interface on the router drops the session immediately. If a link just stops carrying traffic, BGP waits for the hold timer (180 seconds by default) before declaring the neighbor dead.

---

## Troubleshooting Scenarios

An L1 engineer spends more time fixing BGP than building it. This section has two parts: problems I hit by accident while building, with the real symptoms, and scenarios I'm breaking on purpose to practice.

### Problems I actually hit

| # | What I saw | Command that showed it | Cause | Fix |
|---|---|---|---|---|
| 1 | RR2 couldn't ping the Surabaya branch. On RR2, the route to `10.1.11.0/30` was "not in table" | `show ip protocols` on RR1 showed **two** OSPF processes with two different router IDs | I had put the branch interface in OSPF process 10 and the HQ interfaces in process 1. Separate processes don't share routes, so RR1 never passed the branch route into area 0 | Put every interface on a router in the same process (`ip ospf 1 area X`) and delete the extra one |
| 2 | From DMZ-SRV, I could ping RR1, RR2, and the branches, but **not** EDGE-1 or EDGE-2 | `show ip route 203.0.113.10` on EDGE-1 showed a **static `/28` via Null0**, distance 1, "Advertised by bgp 65001" | The Null0 route and the `network` statement used the DMZ's `/28` mask instead of the whole `/24`. Both routes were for the same prefix, so the static route (distance 1) beat OSPF (110) and sent every reply for the DMZ to Null0 | Replace them with the `/24`: `ip route 203.0.113.0 255.255.255.0 Null0` and `network 203.0.113.0 mask 255.255.255.0`. The longer `/28` from OSPF then wins for the real server, and the `/24` only catches unused addresses |
| 3 | RR1's BGP table listed `8.8.8.8/32` and `9.9.9.9/32` with `* i` but **no `>`**, and the next hop was `172.16.10.1` | `show ip bgp` on RR1 | iBGP doesn't change the next hop, so the route-reflector received an ISP address it had no route to (I deliberately kept the ISP links out of OSPF) | `neighbor <RR> next-hop-self` on both edges, then `clear ip bgp * soft`. Afterward the routes showed `*>i` with next hop `1.1.1.1` or `2.2.2.2` |
| 4 | Typing `neighbor 3.3.3.3 next-hop-self` on EDGE-2 returned `% Specify remote-as or peer-group commands first`. RR1 also only showed one path per prefix | The CLI error itself, then `show run \| section router bgp` | EDGE-2 never had its iBGP neighbor lines from Step 2. A neighbor must exist (have a `remote-as`) before any other option can be set | Add `neighbor 3.3.3.3 remote-as 65001`, `update-source Loopback0`, and `next-hop-self` (and the same for `4.4.4.4`). After that, RR1 showed paths through both edges |
| 5 | `ping 8.8.8.8` failed (0/5) from RR1 and RR2, even though they had a valid BGP route | `show ip route 8.8.8.8` showed a good route, so the problem was the **return** path | The ping used a private `10.1.x.x` source address, and INTERNET has no route back to private ranges | Ping from an address in my public block (`ping 8.8.8.8 source GigabitEthernet0/4` on RR1, or ping from DMZ-SRV). In a real network, NAT would handle this, and I left NAT out of the lab |

The second one taught me the most. A static route has an administrative distance of 1 and OSPF has 110, but distance only matters between routes for the **same prefix**. A `/24` and a `/28` are different prefixes, so the longer match always wins. My mistake was making both `/28`.

<!-- Add output snippets for the best ones -->

### Scenarios I'm breaking on purpose

For each of these, I break one thing, write down the symptom, find it with a command, and fix it. The "Expected" column is what I predict, and I fill in the "What I saw" column from my real lab.

| Scenario | How I break it | Expected symptom | Command to find it | Fix | What I saw |
|---|---|---|---|---|---|
| **Wrong remote ASN** | Set the wrong `remote-as` on one side of an eBGP session | Session never reaches Established, flipping between Idle and Active, with a log message about a bad peer AS | `show ip bgp summary` | Correct `remote-as` | |
| **Link down** | Shut the interface between an edge and its ISP | Neighbor drops, and its routes disappear | `show ip bgp summary`, `show ip interface brief` | `no shutdown` | |
| **Missing `network` statement** | Remove `network 203.0.113.0 mask 255.255.255.0` from an edge | The block disappears from the ISP and INTERNET tables | `show ip bgp` on the ISP | Add the `network` command back | |
| **Missing `update-source`** | Remove `update-source Loopback0` on one side of an iBGP session | Session stays in Active | `show ip bgp summary`, `show ip bgp neighbors` | Add it back on that side | |
| **OSPF area mismatch** | Put RR1 Gi0/2 in area 0 while BR-SBY's side stays in area 10 | No OSPF adjacency, and the branch loses its routes | `show ip ospf neighbor`, `show ip ospf interface brief` | Use the same area on both ends of the link | |
| **Missing `route-reflector-client`** | Remove it for one edge on RR1 | Routes from that edge don't reach the other clients | `show ip bgp` on the other edge | Add the line back | |

<!-- Replace the empty "What I saw" cells with my real results and add output snippets for the best ones -->
