## Overview

<img width="1127" height="646" alt="Screenshot 2026-11-14 161159" src="https://github.com/user-attachments/assets/d09aafe8-45ef-43d9-b78b-dc9b48b35a74" />

As illustrated in the topology above, I have successfully implemented OSPF (Open Shortest Path First) on the **Fortinet2** device by establishing two OSPF areas: Area 0 and Area 10. Area 0 is configured on ports 2 and 3, while Area 10 is configured on port 5.

## The Issue

Initially, I thought OSPF worked the same way as a traditional router, where having the correct routing configuration would be sufficient to enable communication between different networks. But I was wrong. 

<img width="759" height="544" alt="image" src="https://github.com/user-attachments/assets/c8d0aae0-4118-40b9-8520-333431ea86a5" />

As you can see from the picture above, Fortinate2 able to reach both LAN without any problem, however when both LAN try to connect to one another, that's when the isse occur.

<img width="797" height="151" alt="image" src="https://github.com/user-attachments/assets/482850b7-ae14-417e-9c02-8e73eba61349" />

The screenshot shows that **ping tests** from R4 to the 10.0.0.0/24 LAN were **unsuccessful**.

**It turns out that the issue was related to the FortiGate firewall policies**, which were preventing traffic from being forwarded between the LAN interfaces. 

## Troubleshooting

### 1. Verify OSPF Neighbor Adjacency on FortiGate 2

I started by checking the OSPF neighbor relationships on Fprtinet2 to ensure that the interfaces participating in OSPF had successfully established adjacency.

In this topology, Port 2 and Port 3 are assigned to Area 0, while Port 5 is assigned to Area 10. I wanted to verify that the OSPF configuration on FortiGate 2 was functioning as expected before investigating the connectivity issue.

**Commands:**

    get router info ospf neighbor

**Verification:**

<img width="755" height="268" alt="image" src="https://github.com/user-attachments/assets/d244984c-1497-4a8e-b367-1149f873d8cd" />

The ospf neighbor info clearly shows that the expected adjacencies were established.

### 2. Verify the Routing Table on Fortinate2

After checking the OSPF neighbor relationships, I inspected the routing table on Fortinet2 to determine whether the required remote networks had been successfully learned.

**Commands:**

    get router info routing-table ospf

**Verification:**

<img width="762" height="304" alt="image" src="https://github.com/user-attachments/assets/211e13b7-ade7-40e6-ae80-a550821fb598" />

It shows that the `10.0.0.0/24` network **and other** relevant remote networks appeared in the routing table, along with their corresponding next-hop information.

### 3. Investigate the Connectivity Issue

Although the OSPF configuration and routing information appeared to be in place, R4 was still unable to reach the `10.0.0.0/24` LAN and vice versa.

Initially, I thought OSPF worked the same way as a traditional router, where having the correct routing configuration would be sufficient to enable communication between different networks. However, I later realized that FortiGate also uses firewall policies to control traffic passing between interfaces.

To investigate this further, I inspected the firewall policies and interface configurations on Fortinet2.

**Commands:**

    show firewall policy
    show system interface

**Finding:**

<img width="758" height="176" alt="image" src="https://github.com/user-attachments/assets/8e6eeeb5-f547-475b-8864-65a1f60fa71e" />

<img width="735" height="674" alt="image" src="https://github.com/user-attachments/assets/0648f16e-47b3-4af2-9bcd-f95df4838827" />


The issue was indeed related to the absence of the necessary firewall policy to permit traffic between the relevant interfaces on Fortinet2. Although OSPF provided the routing information, Fortinet2 still needed an appropriate firewall policy to allow the traffic to pass through.

### 4. Configure the Firewall Policy on Fortinet2

To resolve the issue, I configured a firewall policy to permit communication between the relevant interfaces.

**Configuration:**

```bash
config firewall address
    edit "LAN-10.0.0.0"
        set type ipmask
        set subnet 10.0.0.0 255.255.255.0
    next
end

config firewall policy
    edit 0
        set name "R4-LAN"
        set srcintf "port5"
        set dstintf "port2" "port3"
        set srcaddr "all"
        set dstaddr "LAN-10.0.0.0"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```

**Verification:**

    show firewall policy

<img width="759" height="411" alt="image" src="https://github.com/user-attachments/assets/e85bcc4a-9ae0-4a50-ad6a-0e4368b2cac0" />

I verified that the new firewall policy was present and that the incoming interface, outgoing interface, source address, destination address, and permitted services were correctly configured.

### 5. Connectivity Testing

After configuring the firewall policy, I repeated the connectivity tests from PC on LAN2 or 10.0.3.0/24 to determine whether the connection to LAN1 was now reachable.

**Results:**

<img width="796" height="338" alt="image" src="https://github.com/user-attachments/assets/8c0ab187-1c9a-4904-9f4b-64af0d0e5871" />

The successful ping test confirmed that communication between the networks was now possible. 

**HOWERVER, ANOTHER PROBLEM OCCUR**. PCs on LAN1 can't reach PCs on LAN2 while the ping were successful on LAN2 to LAN1.



