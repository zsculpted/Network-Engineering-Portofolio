## Current Situation

<img width="1127" height="646" alt="Screenshot 2026-11-14 161159" src="https://github.com/user-attachments/assets/d09aafe8-45ef-43d9-b78b-dc9b48b35a74" />

As illustrated in the topology above, I have successfully implemented OSPF (Open Shortest Path First) on the **Fortinet2** device by establishing two OSPF areas: Area 0 and Area 10. Area 0 is configured on ports 2 and 3, while Area 10 is configured on port 5.

## The Issue

Initially, I thought OSPF worked the same way as a traditional router, where having the correct routing configuration would be sufficient to enable communication between different networks. But I was wrong. 

<img width="759" height="544" alt="image" src="https://github.com/user-attachments/assets/c8d0aae0-4118-40b9-8520-333431ea86a5" />

As you can see from the picture above, Fortinate2 able to reach both LAN without any problem, however when both LAN try to connect to one another, that's when the isse occur.

When I try to **ping tests** from LAN2 to LAN1 or vice versa, the ping were **unsuccessful**.

**It turns out that the issue was related to the FortiGate firewall policies**, which were preventing traffic from being forwarded between interfaces. 

## Troubleshooting

### 1. Verify OSPF Neighbor Adjacency on FortiGate 2

I started by checking the OSPF neighbor relationships on Fprtinet2 to ensure that the interfaces participating in OSPF had successfully established adjacency.

In this topology, port2 and port3 are assigned to Area 0, while port5 is assigned to Area 10. I wanted to verify that the OSPF configuration on Fortinet2 was functioning as expected, before I start investigating the connectivity issue.

**Commands:**

    get router info ospf neighbor

**Verification:**

<img width="755" height="268" alt="image" src="https://github.com/user-attachments/assets/d244984c-1497-4a8e-b367-1149f873d8cd" />

The ospf neighbor info clearly shows that the expected adjacency were established.

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
        set subnet 10.0.0.0 255.255.0.0
    next
end

config firewall policy
    edit 1
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

## Troubleshoot 2

At first, I thought every traffic leaving port5 through port2 or port3 meaning every packet entering port5 by those port will automatically forwarded as well. Turns out this is another FortiGate rules. The configuration I did above only permits new sessions initiated from LAN2 toward LAN1. 

Below is the previous firewall policy (edit 1), I did change the policy name into 'LAN2-LAN1' to give more clarity.

<img width="759" height="408" alt="image" src="https://github.com/user-attachments/assets/e5cd1575-b4c1-4d8c-8101-3ef595564338" />

I then proceed to configure reverse firewall policy.

```bash
config firewall policy
    edit 2
        set name "LAN1-to-LAN2"
        set srcintf "port2" "port3"
        set dstintf "port5"
        set srcaddr ALL
        set dstaddr "LAN-10.0.0.0"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```

**Resutlts:**

<img width="756" height="714" alt="image" src="https://github.com/user-attachments/assets/fa7187f8-c5c8-4bce-9bab-90705e13cc3f" />

<img width="1457" height="283" alt="image" src="https://github.com/user-attachments/assets/b39eb401-919b-4c2b-b293-d183da240b7c" />

I then proceed to go to a PC in LAN1 and try to ping VPC on LAN2 and it was **successful** as shown below!

<img width="935" height="217" alt="image" src="https://github.com/user-attachments/assets/904d7223-b159-4b91-8737-404fec3a5dfd" />
