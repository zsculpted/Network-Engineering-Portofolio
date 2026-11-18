# A. FortiGate Flow-Based Inspection Lab

In this session I want to focused on one topic, that is inspection and inspection modes on the FortiGate. I will start off by exploring flow-based inspection after understand it conceptually so now I'll try it in my own lab. This document records what I learned, what I built, what worked, and what didn’t.

## What I wanted to achieve

- After understanding what “inspection” means on a FortiGate and the difference between flow-based and proxy-based mode, I'll take this time to build a small three-zone network (LAN, Internet, DMZ) using PNETLab.
- Create a flow-based firewall policy and attach security profiles (web filter, IPS, antivirus).
- Test the profiles and compare flow-based against proxy-based behavior.

## Lab Environment 
- PNETLab
- FortiGate-VM64-KVM
- Aruba CX switch (SW1), Arista vEOS switch (SW2)
- VPCS (VPC1), a BusyBox Linux node with Firefox, and a Web-SRV node.

## Topology

<img width="542" height="623" alt="image" src="https://github.com/user-attachments/assets/55f2e381-db02-4090-83d8-efbc1d2d7701" />

This one is pretty simple. My lab started with extra routers (vIOS6 and vIOS) between the FortiGate and the LAN and DMZ. I then simplified it so that the FortiGate is the gateway for both zones, with the switches acting only as switches. 

## Initial Configuration That I did

### 1. End devices

VPC1:

```text
ip 192.168.10.50 255.255.255.0 192.168.10.1
save
```

Linux node: 

```bash
ip link set eth0 up
ip addr add 192.168.10.51/24 dev eth0
ip route add default via 192.168.10.1
echo "nameserver 8.8.8.8"> /etc/resolv.conf
```

### 2. Aruba CX (layer 2 only)

AOS-CX ports are shut down and routed by default, which differs from Cisco, which something I haven't notice before:

```text
configure terminal
interface 1/1/1-1/1/3
no shutdown
no routing
vlan access 1
exit
write memory
```

### 3. FortiGate interface

```text
config system interface
    edit port2
        set mode static
        set ip 192.168.10.1 255.255.255.0
        set allowaccess ping https ssh http
    next
end
```

### 4. FortiGate Address Object and Policy 1 (flow-based, LAN to Internet)

```text
config firewall address
    edit "LAN1-Internet"
        set subnet 192.168.10.0 255.255.255.0
    next
end

config firewall policy
    edit 1
        set name "LAN-to-WAN"
        set srcintf "port2"
        set dstintf "port1"
        set srcaddr "LAN1-Internet"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set inspection-mode flow
        set nat enable
    next
end
```

**Something I noticed:** `show firewall policy` did not display the inspection-mode line. That is because flow is the default and `show` only prints non-default settings. I confirmed it two ways:

```text
show full-configuration firewall policy 1
```

and the GUI policy page, where **Inspection Mode shows Flow-based**. When I temporarily set a policy to proxy mode, the line appears in the normal `show` output.


## The problem I encountered along the way 

**The issue below is not part of the flow-based inspection, only a small image issue whic I found interesting to explore. Scroll to HEADING B to jump back on Flow-Based Inspection!!**  

**Symptom:** From the Linux node (192.168.10.51), I can ping public IP addresses, but I cannot resolve any domain name. I already set `nameserver 8.8.8.8` in `/etc/resolv.conf` on the node.

<img width="959" height="405" alt="image" src="https://github.com/user-attachments/assets/b439ccc4-6d12-4629-80ce-8d842b28b32c" />

Firefox on the Linux node also fails: websites show "Server Not Found", and even the lab gateway page at `192.168.1.1` would not open.

<img width="1008" height="570" alt="Screenshot 2026-11-16 160115" src="https://github.com/user-attachments/assets/f5838f05-f72f-4374-be19-93861efcdc13" />


**What makes it strange:** The same names work fine **from the FortiGate itself**.

<img width="738" height="243" alt="Screenshot 2026-11-16 212033" src="https://github.com/user-attachments/assets/a18fb0e3-7b78-4b94-a66e-5b90a72e8eb1" />

So the FortiGate resolves names and reaches the Internet, but a client behind it cannot.

| Test | From the Linux node | From the FortiGate |
|---|---|---|
| `ping 8.8.8.8` | works | works |
| `ping 1.1.1.1` | works | works |
| `ping google.com` | fails (bad address) | works |
| DNS query to 8.8.8.8 | no reply | reply received |
| `traceroute` | stops after 2 hops | completes all 13 hops |

### What I found out with the FortiGate tools

1. **Policy 1 is correct.** It goes from port2 to port1, service ALL, NAT enabled, and it is the only policy.
2. **The queries arrive on the LAN side.** A sniffer on port2 showed the Linux node sending UDP port 53 packets to 8.8.8.8.
3. **They leave port1 correctly NATed.** A sniffer on port1 showed the source address changed to the port1 address (192.168.1.51), as expected:
4. **No reply ever comes back for the Linux node's queries.** But in the same capture, the FortiGate's own query from the **same address to the same server** gets an answer right away:

### What I already tried so far [TROUBLESHOOTING]

- Set `nameserver 8.8.8.8` in `/etc/resolv.conf` (it did not exist at first, and later writes hit "No space left on device").
- `nslookup google.com 8.8.8.8` and `nslookup google.com 1.1.1.1` directly from the node.
- Pointed the node at the FortiGate (192.168.10.1) as its DNS server. The FortiGate had no `config system dns-server` option on my build, so nothing answered.
- Checked `get system dns` (the FortiGate uses 8.8.8.8 and 1.1.1.1 and works).
- Checked the policy with `show firewall policy`.
- Compared the packets with `diagnose sniffer` on both ports.

### What works

After spending some times figuring out the problem exist in the Linux node all along and not on the FortiGate. 

A post on reddit suggested using `ethtool` on the Linux node:

```text
~ # ethtool -K eth0 tx off
Actual changes:
tx-checksum-ip-generic: off
tx-tcp-segmentation: off [not requested]
tx-tcp-ecn-segmentation: off [not requested]
tx-tcp6-segmentation: off [not requested]
```

Right after that, DNS and ping by name worked as shown on the image below.

<img width="1000" height="667" alt="image" src="https://github.com/user-attachments/assets/b053e832-1ffe-4895-b8eb-917f0d3a36c4" />

### What `ethtool` is and why this worked

`ethtool` is a Linux command for viewing and changing a network card's settings, including **offload** features, which are jobs the operating system hands to the network card so the CPU doesn't have to do them.

**TX checksum offload** is one of them. Every TCP and UDP packet carries a checksum so the receiver can detect damage. With offload on, the operating system leaves the checksum unfinished and expects the network card hardware to fill it in just before the packet goes out.

In a virtual lab (PNETLab/QEMU) the "network card" is software, so nothing completes the checksum. The packet reaches the FortiGate with a wrong checksum. When the FortiGate applies NAT, it updates the checksum from the old (wrong) value, so the result is still wrong, and a device further along drops the packet silently.

This matches everything I saw:

- **Ping worked.** An ICMP checksum does not cover the IP addresses, so NAT doesn't break it.
- **DNS (UDP) and web (TCP) failed.** Their checksums include the IP addresses, so NAT made the bad checksum matter.
- **The FortiGate's own DNS worked.** It calculates its checksums correctly.
- **The sniffer showed queries leaving with no replies.** The packets looked normal in the capture but were damaged on the wire.

I haven't proven exactly where the damaged packets were dropped, but the fact that one command fixed the problem fits this explanation well. I consider it a limitation of this small Linux image together with its virtual network driver, not a FortiGate problem.

### Lessons learned

- A working ping from the FortiGate does not prove that forwarded client traffic works. They are different paths.
- Packet sniffers on both interfaces (`diagnose sniffer packet`) show where traffic stops, so I don't have to guess.
- In a virtual lab, check the endpoint's network driver and offload settings, not only the firewall.

---

# B. Back on track: testing flow-based inspection

Most of this session went to troubleshooting the Linux node, but the point of the lab is **flow-based inspection**. Now that DNS works, I can finally test it. This is what I'm testing and how.

## What I want to prove

A firewall policy only checks addresses and ports (layers 3 and 4). A **security profile in flow-based mode** reads the *content* of the allowed traffic as it streams through. To see the difference, I'll let a policy allow a website and then use a web filter profile to block it, while ping to the same destination keeps working.

## Before I start

- The Linux node can resolve names and reach the Internet (after the `ethtool` fix).

_to be continued_
