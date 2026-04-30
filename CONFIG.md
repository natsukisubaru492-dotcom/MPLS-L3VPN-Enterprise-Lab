# MPLS L3VPN Configuration & Troubleshooting Report


## I. Troubleshooting Report

| No. | Symptom | Root Cause | Solution |
| :--- | :--- | :--- | :--- |
| **1** | `% BGP not active` or configuration loss after reboot.[cite: 1] | Failure to save running configuration (`wr`) and export the node configuration to disk in PnetLab.[cite: 1] | Execute the `write memory` command on each device, then right-click the node and select **"Export CFG"** in the PnetLab interface.[cite: 1] |
| **2** | Ping failure (Request timeout) despite BGP peering being Up.[cite: 1] | **BGP Loop Prevention Mechanism**: CE2 discards the update because the AS-Path contains its own AS (`65101`).[cite: 1] | Configure the `neighbor ... as-override` command on both PE routers to replace the customer's AS with the ISP's AS (`65001`).[cite: 1] |

---

## II. Detailed Device Configurations

### 1. Router P (Provider Core)
```text
hostname P
!
interface Loopback0
 description Router-ID
 ip address 1.1.1.1 255.255.255.255
!
interface Ethernet0/1
 description Link-to-PE1
 ip address 10.0.0.2 255.255.255.252
 mpls ip
!
interface Ethernet0/3
 description Link-to-PE2
 ip address 10.0.0.5 255.255.255.252
 mpls ip
!
router ospf 1
 router-id 1.1.1.1
 network 1.1.1.1 0.0.0.0 area 0
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.0.4 0.0.0.3 area 0
```

### 2. Router PE1 (Provider Edge - Hanoi)
```text
hostname PE1
!
! 1. Define VRF for Customer A
ip vrf VPN-A
 rd 65001:1
 route-target export 65001:1
 route-target import 65001:1
!
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
!
! 2. Core-facing Interface (Internal)
interface Ethernet0/1
 description Link-to-P
 ip address 10.0.0.1 255.255.255.252
 mpls ip
!
! 3. Customer-facing Interface (Edge)
interface Ethernet0/0
 description Link-to-CE1
 ip vrf forwarding VPN-A
 ip address 192.168.1.2 255.255.255.0
!
! 4. OSPF Routing (Core Only)
router ospf 1
 router-id 2.2.2.2
 network 2.2.2.2 0.0.0.0 area 0
 network 10.0.0.0 0.0.0.3 area 0
!
! 5. BGP Configuration
router bgp 65001
 bgp router-id 2.2.2.2
 no bgp default ipv4-unicast
 !
 ! Establish iBGP peering with PE2
 neighbor 3.3.3.3 remote-as 65001
 neighbor 3.3.3.3 update-source Loopback0
 !
 ! Activate VPNv4 Address-Family (MP-BGP)
 address-family vpnv4
  neighbor 3.3.3.3 activate
  neighbor 3.3.3.3 send-community extended
 exit-address-family
 !
 ! Establish eBGP peering with Customer (Within VRF)
 address-family ipv4 vrf VPN-A
  neighbor 192.168.1.1 remote-as 65101
  neighbor 192.168.1.1 activate
  neighbor 192.168.1.1 as-override
 exit-address-family
```

### 3. Router PE2 (Provider Edge - Saigon)
```text
hostname PE2
!
ip vrf VPN-A
 rd 65001:1
 route-target export 65001:1
 route-target import 65001:1
!
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
!
interface Ethernet0/3
 description Link-to-P
 ip address 10.0.0.6 255.255.255.252
 mpls ip
!
interface Ethernet0/0
 description Link-to-CE2
 ip vrf forwarding VPN-A
 ip address 192.168.2.2 255.255.255.0
!
router ospf 1
 router-id 3.3.3.3
 network 3.3.3.3 0.0.0.0 area 0
 network 10.0.0.4 0.0.0.3 area 0
!
router bgp 65001
 bgp router-id 3.3.3.3
 no bgp default ipv4-unicast
 !
 neighbor 2.2.2.2 remote-as 65001
 neighbor 2.2.2.2 update-source Loopback0
 !
 address-family vpnv4
  neighbor 2.2.2.2 activate
  neighbor 2.2.2.2 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf VPN-A
  neighbor 192.168.2.1 remote-as 65101
  neighbor 192.168.2.1 activate
  neighbor 192.168.2.1 as-override
 exit-address-family
```

### 4. Router CE1 (Customer Edge - Hanoi)
```text
hostname CE1
!
! LAN Interface connected to Switch/PC
interface Ethernet0/1
 ip address 172.16.1.1 255.255.255.0
!
! WAN Interface connected to ISP
interface Ethernet0/0
 ip address 192.168.1.1 255.255.255.0
!
! eBGP Routing with ISP
router bgp 65101
 neighbor 192.168.1.2 remote-as 65001
 ! Advertise LAN network into BGP
 network 172.16.1.0 mask 255.255.255.0
```

### 5. Router CE2 (Customer Edge - Saigon)
```text
hostname CE2
!
interface Ethernet0/1
 ip address 172.16.2.1 255.255.255.0
!
interface Ethernet0/0
 ip address 192.168.2.1 255.255.255.0
!
router bgp 65101
 neighbor 192.168.2.2 remote-as 65001
 network 172.16.2.0 mask 255.255.255.0
```

### 6. PC Configurations (VPCS)
```text
ip 172.16.1.10/24 172.16.1.1
save

ip 172.16.2.10/24 172.16.2.1
save
```



