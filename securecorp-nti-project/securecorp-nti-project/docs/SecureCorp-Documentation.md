# SecureCorp — Full Network Configuration Documentation

NTI Graduation Project | Cisco Packet Tracer

Covers: Routing (Static + OSPF) · Security · VLAN · Services (DHCP/DNS/HTTP/FTP/Email) · EtherChannel · FHRP (HSRP) · WLAN · NAT · IP Phones

---

## 1. IP Addressing Plan

### 1.1 WAN links (point-to-point /30)

| Link | Network | Device A | IP | Device B | IP |
|---|---|---|---|---|---|
| ISP – Edge2 | 100.64.0.0/30 | ISP Router Se0/1/0 | 100.64.0.1 | Edge Router 2 Se0/1/0 | 100.64.0.2 |
| ISP – Edge1 | 100.64.0.4/30 | ISP Router Se0/1/1 | 100.64.0.5 | Edge Router 1 Se0/1/1 | 100.64.0.6 |
| Edge1 – Branch1 | 172.16.1.0/30 | Edge Router 1 Se0/1/0 | 172.16.1.1 | Branch Router 1 Se0/1/0 | 172.16.1.2 |
| Edge2 – Branch2 | 172.16.2.0/30 | Edge Router 2 Se0/1/1 | 172.16.2.1 | Branch Router 2 Se0/1/1 | 172.16.2.2 |

### 1.2 Edge-to-Core routed links (/30, no switchport)

| Link | Network | Device A | IP | Device B | IP |
|---|---|---|---|---|---|
| Edge1 – Core1 | 10.10.10.0/30 | Edge Router 1 Gi0/0/0 | 10.10.10.1 | Core Switch 1 Gi1/0/1 | 10.10.10.2 |
| Edge2 – Core2 | 10.10.10.4/30 | Edge Router 2 Gi0/0/0 | 10.10.10.5 | Core Switch 2 Gi1/0/1 | 10.10.10.6 |
| Core2 – VoiceGW | 10.10.10.8/30 | Core Switch 2 Gi1/0/7 | 10.10.10.9 | Voice Gateway Fa0/0 (Cisco 2811) | 10.10.10.10 |

### 1.3 Loopbacks (router-id)

| Device | Loopback0 |
|---|---|
| ISP Router | 9.9.9.9/32 |
| Edge Router 1 | 1.1.1.1/32 |
| Edge Router 2 | 2.2.2.2/32 |
| Branch Router 1 | 3.3.3.3/32 |
| Branch Router 2 | 4.4.4.4/32 |
| Voice Gateway | 5.5.5.5/32 |
| Core Switch 1 | 6.6.6.6/32 |
| Core Switch 2 | 7.7.7.7/32 |

### 1.4 HQ VLANs / SVIs (configured identically on Core1 & Core2, HSRP virtual IP = .1)

| VLAN | Name | Subnet | Gateway (VIP) | Core1 IP | Core2 IP | HSRP Active |
|---|---|---|---|---|---|---|
| 10 | HR | 192.168.10.0/24 | .1 | .2 | .3 | Core1 |
| 20 | FINANCE | 192.168.20.0/24 | .1 | .2 | .3 | Core1 |
| 30 | IT | 192.168.30.0/24 | .1 | .2 | .3 | Core2 |
| 40 | SERVERS | 192.168.40.0/24 | .1 | .2 | .3 | Core1 |
| 50 | VOICE | 192.168.50.0/24 | .1 | .2 | .3 | Core2 |
| 60 | WIFI-STAFF | 192.168.60.0/24 | .1 | .2 | .3 | Core1 |
| 70 | WIFI-GUEST | 192.168.70.0/24 | .1 | .2 | .3 | Core2 |
| 100 | MGMT | 192.168.100.0/24 | .1 | .2 | .3 | Core1 |
| 99 | NATIVE (unused) | — | — | — | — | — |

### 1.5 Branch VLANs (router-on-a-stick, no HSRP needed — single router per branch)

| Site | VLAN | Subnet | Gateway |
|---|---|---|---|
| Branch1 | 10 (data) | 192.168.111.0/24 | 192.168.111.1 |
| Branch1 | 50 (voice) | 192.168.115.0/24 | 192.168.115.1 |
| Branch2 | 10 (data) | 192.168.121.0/24 | 192.168.121.1 |
| Branch2 | 50 (voice) | 192.168.125.0/24 | 192.168.125.1 |

### 1.6 Server static IPs (VLAN 40)

| Server | IP | Gateway |
|---|---|---|
| DHCP-DNS | 192.168.40.10/24 | 192.168.40.1 |
| HTTP-FTP | 192.168.40.20/24 | 192.168.40.1 |
| EMAIL-SERVER | 192.168.40.30/24 | 192.168.40.1 |

---

## 2. OSPF Area Plan

- **Area 0 (backbone):** Edge1–Core1 link, Edge2–Core2 link, Core2–VoiceGW link, all HQ VLAN subnets (10/20/30/40/50/60/70/100), all HQ loopbacks
- **Area 1:** Edge1–Branch1 link + Branch1 LAN subnets
- **Area 2:** Edge2–Branch2 link + Branch2 LAN subnets
- **ISP links are NOT in OSPF.** Edge1 and Edge2 each use a static default route to the ISP, redistributed into OSPF with `default-information originate` so the whole org learns internet reachability.

---

## 3. Device Configurations

### 3.1 ISP Router

```
hostname ISP-Router
!
interface Loopback0
 ip address 9.9.9.9 255.255.255.255
!
interface Loopback1
 description FAKE-INTERNET-TARGET-FOR-DEMO
 ip address 8.8.8.8 255.255.255.255
!
interface Serial0/1/0
 ip address 100.64.0.1 255.255.255.252
 clock rate 64000
 no shutdown
!
interface Serial0/1/1
 ip address 100.64.0.5 255.255.255.252
 clock rate 64000
 no shutdown
!
end
```
ISP has connected routes to both Edge routers' WAN IPs — that's enough to answer NAT'd traffic. No OSPF, no static routes needed here. `Loopback1` (8.8.8.8) exists purely so you have something to `ping 8.8.8.8` from any internal PC as a clean "internet reachability" demo in your defense — there's no real internet in this lab, so without it, pinging any real public address will correctly show "Destination host unreachable."

---

### 3.2 Edge Router 1

```
hostname EDGE-R1
!
ip domain-name securecorp.local
username admin privilege 15 secret Admin@123
enable secret Cisco@123
service password-encryption
crypto key generate rsa modulus 1024
ip ssh version 2
banner motd # Authorized access only - SecureCorp #
line vty 0 4
 login local
 transport input ssh
!
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
!
interface Serial0/1/1
 description LINK-TO-ISP
 ip address 100.64.0.6 255.255.255.252
 ip nat outside
 no shutdown
!
interface Serial0/1/0
 description LINK-TO-BRANCH1
 ip address 172.16.1.1 255.255.255.252
 clock rate 64000
 ip nat inside
 no shutdown
!
interface GigabitEthernet0/0/0
 description LINK-TO-CORE1
 ip address 10.10.10.1 255.255.255.252
 ip nat inside
 no shutdown
!
router ospf 1
 router-id 1.1.1.1
 network 10.10.10.0 0.0.0.3 area 0
 network 172.16.1.0 0.0.0.3 area 1
 network 1.1.1.1 0.0.0.0 area 0
 default-information originate
!
ip route 0.0.0.0 0.0.0.0 Serial0/1/1
!
ip access-list standard NAT-SOURCE
 permit 192.168.0.0 0.0.255.255
!
ip nat inside source list NAT-SOURCE interface Serial0/1/1 overload
ip nat inside source static tcp 192.168.40.20 80 100.64.0.6 80
ip nat inside source static tcp 192.168.40.20 21 100.64.0.6 21
ip nat inside source static tcp 192.168.40.30 25 100.64.0.6 25
ip nat inside source static tcp 192.168.40.30 110 100.64.0.6 110
!
end
```

> **Note:** Packet Tracer's IOS doesn't accept the `extendable` keyword on `ip nat inside source static tcp` — leave it off (already removed above). If you paste configs from real Cisco documentation, strip that keyword before pasting into PT.

---

### 3.3 Edge Router 2

```
hostname EDGE-R2
!
ip domain-name securecorp.local
username admin privilege 15 secret Admin@123
enable secret Cisco@123
service password-encryption
crypto key generate rsa modulus 1024
ip ssh version 2
banner motd # Authorized access only - SecureCorp #
line vty 0 4
 login local
 transport input ssh
!
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
!
interface Serial0/1/0
 description LINK-TO-ISP
 ip address 100.64.0.2 255.255.255.252
 ip nat outside
 no shutdown
!
interface Serial0/1/1
 description LINK-TO-BRANCH2
 ip address 172.16.2.1 255.255.255.252
 clock rate 64000
 ip nat inside
 no shutdown
!
interface GigabitEthernet0/0/0
 description LINK-TO-CORE2
 ip address 10.10.10.5 255.255.255.252
 ip nat inside
 no shutdown
!
router ospf 1
 router-id 2.2.2.2
 network 10.10.10.4 0.0.0.3 area 0
 network 172.16.2.0 0.0.0.3 area 2
 network 2.2.2.2 0.0.0.0 area 0
 default-information originate
!
ip route 0.0.0.0 0.0.0.0 Serial0/1/0
!
ip access-list standard NAT-SOURCE
 permit 192.168.0.0 0.0.255.255
!
ip nat inside source list NAT-SOURCE interface Serial0/1/0 overload
!
end
```

> **If `show ip nat translations` on Edge Router 2 is empty:** this config is already correct as written (`ip nat inside` on Se0/1/1 and Gi0/0/0, `ip nat outside` on Se0/1/0, ACL + overload rule present) — the most likely cause is one of these:
> 1. **The config was never actually pasted onto the live device** — run `show running-config | section nat` on Edge Router 2 and confirm all four NAT-related lines (2× `ip nat inside`, 1× `ip nat outside`, the `ip nat inside source list...overload` line) are present.
> 2. **No traffic has crossed it yet.** Unlike the static port-forwards on Edge1 (which are permanent), overloaded/PAT entries only appear in the table **while a session is active** and time out afterward. Test it live:
>    - From an IT or Finance PC (Core2-side VLAN), run `ping 100.64.0.1` (a few extended pings so the session stays open, e.g. `ping 100.64.0.1 -n 20` isn't valid PT syntax — just run `ping 100.64.0.1` a couple of times back to back)
>    - **Immediately** run `show ip nat translations` on Edge Router 2 — you should see an `icmp` entry appear with inside global `100.64.0.2` while the ping is running
> 3. **The ACL doesn't match the source subnet** — double check `show access-lists NAT-SOURCE` shows `permit 192.168.0.0 0.0.255.255` and that it has hit counts after you ping (`show access-lists NAT-SOURCE` again — the permit line should show a growing match count).

---

### 3.4 Branch Router 1

```
hostname BRANCH-R1
!
ip domain-name securecorp.local
username admin privilege 15 secret Admin@123
enable secret Cisco@123
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
!
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
!
interface Serial0/1/0
 description LINK-TO-EDGE1
 ip address 172.16.1.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0/0
 description TRUNK-TO-SWITCH-BRANCH1
 no ip address
 no shutdown
!
interface GigabitEthernet0/0/0.10
 encapsulation dot1Q 10
 ip address 192.168.111.1 255.255.255.0
 ip helper-address 192.168.40.10
!
interface GigabitEthernet0/0/0.50
 encapsulation dot1Q 50
 ip address 192.168.115.1 255.255.255.0
 ip helper-address 192.168.40.10
!
router ospf 1
 router-id 3.3.3.3
 network 172.16.1.0 0.0.0.3 area 1
 network 192.168.111.0 0.0.0.255 area 1
 network 192.168.115.0 0.0.0.255 area 1
 network 3.3.3.3 0.0.0.0 area 1
!
end
```

---

### 3.5 Branch Router 2

```
hostname BRANCH-R2
!
ip domain-name securecorp.local
username admin privilege 15 secret Admin@123
enable secret Cisco@123
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
!
interface Loopback0
 ip address 4.4.4.4 255.255.255.255
!
interface Serial0/1/1
 description LINK-TO-EDGE2
 ip address 172.16.2.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0/0
 description TRUNK-TO-SWITCH-BRANCH2
 no ip address
 no shutdown
!
interface GigabitEthernet0/0/0.10
 encapsulation dot1Q 10
 ip address 192.168.121.1 255.255.255.0
 ip helper-address 192.168.40.10
!
interface GigabitEthernet0/0/0.50
 encapsulation dot1Q 50
 ip address 192.168.125.1 255.255.255.0
 ip helper-address 192.168.40.10
!
router ospf 1
 router-id 4.4.4.4
 network 172.16.2.0 0.0.0.3 area 2
 network 192.168.121.0 0.0.0.255 area 2
 network 192.168.125.0 0.0.0.255 area 2
 network 4.4.4.4 0.0.0.0 area 2
!
end
```

---

### 3.6 Voice Gateway Router (CME) — Cisco 2811

> **Platform note:** `telephony-service` (Cisco CME) is **not supported on the ISR4331 in Packet Tracer**. This build uses a **Cisco 2811**, which has onboard `FastEthernet0/0` and `FastEthernet0/1` (no Gigabit ports) — that's why the interface name below is `FastEthernet0/0`, not `GigabitEthernet0/0`.

> **Note:** `!` is just a comment marker in IOS — it does **not** exit a submode. Whenever a config block moves from one submode to a different one (e.g. `router ospf` → `telephony-service`), an explicit `exit` is required first.

```
hostname VOICE-GW
!
interface Loopback0
 ip address 5.5.5.5 255.255.255.255
!
interface FastEthernet0/0
 description LINK-TO-CORE2
 ip address 10.10.10.10 255.255.255.252
 duplex auto
 speed auto
 no shutdown
!
router ospf 1
 router-id 5.5.5.5
 network 10.10.10.8 0.0.0.3 area 0
 network 5.5.5.5 0.0.0.0 area 0
exit
!
telephony-service
 max-ephones 10
 max-dn 10
 ip source-address 10.10.10.10 port 2000
 auto assign 1 to 10
 max-conferences 4
 create cnf-files version-stamp Jan 01 2026 00:00:00
!
ephone-dn 1
 number 1001
 name HR-Phone
!
ephone-dn 2
 number 2001
 name Branch1-Phone
!
ephone-dn 3
 number 3001
 name Branch2-Phone
!
ephone 1
 mac-address <IP Phone0 / HR phone's MAC>
 button 1:1
!
ephone 2
 mac-address <IP Phone1 / Branch1 phone's MAC>
 button 1:2
!
ephone 3
 mac-address 0060.4768.64B8
 button 1:3
!
end
```
Get each phone's MAC from the phone's Config tab in Packet Tracer, then paste it into the matching `ephone` block and reset the phone so it registers.

> **Status:** all three phones are now configured. Branch1 (1001) and Branch2 (2001) were already confirmed registered. To finish HR: on VOICE-GW, run:
> ```
> conf t
> ephone 3
>  mac-address 0060.4768.64B8
>  button 1:3
> end
> wr
> ```
> Then power-cycle IP Phone0 on SW-HR (right-click → toggle power off/on, or turn it off and on in Physical view) so it re-DHCPs, picks up the TFTP address from option 150, and registers. Confirm with `show ephone registered` on VOICE-GW — you should see a third entry for ephone-3, IP in the `192.168.50.0/24` range, number 3001.
>
> **About the `dial-peer voice ... voip` entries you added:** these aren't actually needed. All three phones register to the *same* CME (this one router), so calls between them route automatically through the `ephone-dn` numbers — no dial-peer required for local extensions. `dial-peer voice voip` is only needed when routing calls *out* to a different call-control system (e.g. a second CME, or a real SIP trunk). They're harmless to leave in, but you can remove them with `no dial-peer voice 1 voip` / `no dial-peer voice 2 voip` to keep the config clean for your documentation.

---

### 3.7 Core Switch 1 (multilayer, L3)

```
hostname CORE-SW1
!
ip routing
ip domain-name securecorp.local
username admin privilege 15 secret Admin@123
enable secret Cisco@123
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
!
vlan 10
 name HR
vlan 20
 name FINANCE
vlan 30
 name IT
vlan 40
 name SERVERS
vlan 50
 name VOICE
vlan 60
 name WIFI-STAFF
vlan 70
 name WIFI-GUEST
vlan 99
 name NATIVE
vlan 100
 name MGMT
!
interface Loopback0
 ip address 6.6.6.6 255.255.255.255
!
interface GigabitEthernet1/0/1
 description ROUTED-LINK-TO-EDGE1
 no switchport
 ip address 10.10.10.2 255.255.255.252
 no shutdown
!
interface range GigabitEthernet1/0/2-4
 description ETHERCHANNEL-TO-CORE2
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,40,50,60,70,99,100
 channel-group 1 mode active
!
interface Port-channel1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,40,50,60,70,99,100
!
interface GigabitEthernet1/0/5
 description TRUNK-TO-SW-HR
 switchport mode trunk
 switchport trunk allowed vlan 10,50
!
interface GigabitEthernet1/0/6
 description TRUNK-TO-SW-FINANCE
 switchport mode trunk
 switchport trunk allowed vlan 20
!
interface GigabitEthernet1/0/7
 description ACCESS-HTTP-FTP-SERVER
 switchport mode access
 switchport access vlan 40
!
interface GigabitEthernet1/0/8
 description ACCESS-DHCP-DNS-SERVER
 switchport mode access
 switchport access vlan 40
!
interface GigabitEthernet1/0/9
 description ACCESS-EMAIL-SERVER
 switchport mode access
 switchport access vlan 40
!
interface GigabitEthernet1/0/10
 description TRUNK-TO-ACCESS-POINT
 switchport mode trunk
 switchport trunk allowed vlan 60,70
!
interface Vlan10
 ip address 192.168.10.2 255.255.255.0
 ip helper-address 192.168.40.10
 standby 10 ip 192.168.10.1
 standby 10 priority 150
 standby 10 preempt
!
interface Vlan20
 ip address 192.168.20.2 255.255.255.0
 ip helper-address 192.168.40.10
 standby 20 ip 192.168.20.1
 standby 20 priority 150
 standby 20 preempt
!
interface Vlan30
 ip address 192.168.30.2 255.255.255.0
 ip helper-address 192.168.40.10
 standby 30 ip 192.168.30.1
!
interface Vlan40
 ip address 192.168.40.2 255.255.255.0
 standby 40 ip 192.168.40.1
 standby 40 priority 150
 standby 40 preempt
!
interface Vlan50
 ip address 192.168.50.2 255.255.255.0
 ip helper-address 192.168.40.10
 standby 50 ip 192.168.50.1
!
interface Vlan60
 ip address 192.168.60.2 255.255.255.0
 ip helper-address 192.168.40.10
 standby 60 ip 192.168.60.1
 standby 60 priority 150
 standby 60 preempt
!
interface Vlan70
 ip address 192.168.70.2 255.255.255.0
 ip helper-address 192.168.40.10
 standby 70 ip 192.168.70.1
!
interface Vlan100
 ip address 192.168.100.2 255.255.255.0
 standby 100 ip 192.168.100.1
 standby 100 priority 150
 standby 100 preempt
!
router ospf 1
 router-id 6.6.6.6
 network 10.10.10.0 0.0.0.3 area 0
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.30.0 0.0.0.255 area 0
 network 192.168.40.0 0.0.0.255 area 0
 network 192.168.50.0 0.0.0.255 area 0
 network 192.168.60.0 0.0.0.255 area 0
 network 192.168.70.0 0.0.0.255 area 0
 network 192.168.100.0 0.0.0.255 area 0
 network 6.6.6.6 0.0.0.0 area 0
!
end
```

---

### 3.8 Core Switch 2 (multilayer, L3)

```
hostname CORE-SW2
!
ip routing
ip domain-name securecorp.local
username admin privilege 15 secret Admin@123
enable secret Cisco@123
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
!
vlan 10
 name HR
vlan 20
 name FINANCE
vlan 30
 name IT
vlan 40
 name SERVERS
vlan 50
 name VOICE
vlan 60
 name WIFI-STAFF
vlan 70
 name WIFI-GUEST
vlan 99
 name NATIVE
vlan 100
 name MGMT
!
interface Loopback0
 ip address 7.7.7.7 255.255.255.255
!
interface GigabitEthernet1/0/1
 description ROUTED-LINK-TO-EDGE2
 no switchport
 ip address 10.10.10.6 255.255.255.252
 no shutdown
!
interface range GigabitEthernet1/0/2-4
 description ETHERCHANNEL-TO-CORE1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,40,50,60,70,99,100
 channel-group 1 mode active
!
interface Port-channel1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,40,50,60,70,99,100
!
interface GigabitEthernet1/0/5
 description TRUNK-TO-SW-IT
 switchport mode trunk
 switchport trunk allowed vlan 30
!
interface GigabitEthernet1/0/6
 description TRUNK-TO-WLC
 switchport mode trunk
 switchport trunk allowed vlan 60,70
!
interface GigabitEthernet1/0/7
 description ROUTED-LINK-TO-VOICE-GW
 no switchport
 ip address 10.10.10.9 255.255.255.252
 no shutdown
!
interface GigabitEthernet1/0/8
 description TRUNK-TO-ACCESS-POINT
 switchport mode trunk
 switchport trunk allowed vlan 60,70
!
interface Vlan10
 ip address 192.168.10.3 255.255.255.0
 ip helper-address 192.168.40.10
 standby 10 ip 192.168.10.1
!
interface Vlan20
 ip address 192.168.20.3 255.255.255.0
 ip helper-address 192.168.40.10
 standby 20 ip 192.168.20.1
!
interface Vlan30
 ip address 192.168.30.3 255.255.255.0
 ip helper-address 192.168.40.10
 standby 30 ip 192.168.30.1
 standby 30 priority 150
 standby 30 preempt
!
interface Vlan40
 ip address 192.168.40.3 255.255.255.0
 standby 40 ip 192.168.40.1
!
interface Vlan50
 ip address 192.168.50.3 255.255.255.0
 ip helper-address 192.168.40.10
 standby 50 ip 192.168.50.1
 standby 50 priority 150
 standby 50 preempt
!
interface Vlan60
 ip address 192.168.60.3 255.255.255.0
 ip helper-address 192.168.40.10
 standby 60 ip 192.168.60.1
!
interface Vlan70
 ip address 192.168.70.3 255.255.255.0
 ip helper-address 192.168.40.10
 standby 70 ip 192.168.70.1
 standby 70 priority 150
 standby 70 preempt
!
interface Vlan100
 ip address 192.168.100.3 255.255.255.0
 standby 100 ip 192.168.100.1
!
router ospf 1
 router-id 7.7.7.7
 network 10.10.10.4 0.0.0.3 area 0
 network 10.10.10.8 0.0.0.3 area 0
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.30.0 0.0.0.255 area 0
 network 192.168.40.0 0.0.0.255 area 0
 network 192.168.50.0 0.0.0.255 area 0
 network 192.168.60.0 0.0.0.255 area 0
 network 192.168.70.0 0.0.0.255 area 0
 network 192.168.100.0 0.0.0.255 area 0
 network 7.7.7.7 0.0.0.0 area 0
!
ip access-list extended GUEST-RESTRICT
 permit udp any any eq bootpc
 permit udp any any eq bootps
 permit udp any host 192.168.40.10 eq domain
 deny ip any 192.168.10.0 0.0.0.255
 deny ip any 192.168.20.0 0.0.0.255
 deny ip any 192.168.30.0 0.0.0.255
 deny ip any 192.168.40.0 0.0.0.255
 deny ip any 192.168.50.0 0.0.0.255
 deny ip any 192.168.100.0 0.0.0.255
 permit ip any any
!
interface Vlan70
 ip access-group GUEST-RESTRICT in
!
end
```
`ip helper-address 192.168.40.10` is now configured on both Core Switch 1 and Core Switch 2 for every VLAN that needs DHCP (10, 20, 30, 50, 60, 70) — this is what actually fixes DHCP at HQ. Verify with `show ip interface Vlan10` (etc.) → look for the "Helper address" line, and confirm clients get an IP with `ipconfig /renew` on a PC.

---

### 3.9 Access switch — SW-HR

```
hostname SW-HR
!
username admin privilege 15 secret Admin@123
enable secret Cisco@123
!
vlan 10
 name HR
vlan 50
 name VOICE
!
interface FastEthernet0/1
 description TRUNK-TO-CORE1
 switchport mode trunk
 switchport trunk allowed vlan 10,50
!
interface range FastEthernet0/2-7
 description PC-PORTS
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
!
interface FastEthernet0/8
 description IP-PHONE0
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 50
 switchport port-security
 switchport port-security maximum 3
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
!
end
```

---

### 3.10 Access switch — SW-FINANCE

```
hostname SW-FINANCE
!
username admin privilege 15 secret Admin@123
enable secret Cisco@123
!
vlan 20
 name FINANCE
!
interface FastEthernet0/1
 description TRUNK-TO-CORE1
 switchport mode trunk
 switchport trunk allowed vlan 20
!
interface range FastEthernet0/2-7
 description PC-PORTS
 switchport mode access
 switchport access vlan 20
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
!
end
```

---

### 3.11 Access switch — SW-IT

```
hostname SW-IT
!
username admin privilege 15 secret Admin@123
enable secret Cisco@123
!
vlan 30
 name IT
!
interface FastEthernet0/1
 description TRUNK-TO-CORE2
 switchport mode trunk
 switchport trunk allowed vlan 30
!
interface range FastEthernet0/2-7
 description PC-PORTS
 switchport mode access
 switchport access vlan 30
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
!
end
```

---

### 3.12 Switch Branch 1

```
hostname SW-BRANCH1
!
username admin privilege 15 secret Admin@123
enable secret Cisco@123
!
vlan 10
 name DATA
vlan 50
 name VOICE
!
interface FastEthernet0/1
 description TRUNK-TO-BRANCH-R1
 switchport mode trunk
 switchport trunk allowed vlan 10,50
!
interface FastEthernet0/2
 description ACCESS-POINT3
 switchport mode access
 switchport access vlan 10
!
interface FastEthernet0/3
 description IP-PHONE1
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 50
 switchport port-security
 switchport port-security maximum 3
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
!
interface range FastEthernet0/4-5
 description PC20-PC21
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
!
end
```

---

### 3.13 Switch Branch 2

```
hostname SW-BRANCH2
!
username admin privilege 15 secret Admin@123
enable secret Cisco@123
!
vlan 10
 name DATA
vlan 50
 name VOICE
!
interface FastEthernet0/1
 description TRUNK-TO-BRANCH-R2
 switchport mode trunk
 switchport trunk allowed vlan 10,50
!
interface FastEthernet0/2
 description ACCESS-POINT2
 switchport mode access
 switchport access vlan 10
!
interface FastEthernet0/3
 description IP-PHONE2
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 50
 switchport port-security
 switchport port-security maximum 3
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
!
interface range FastEthernet0/4-5
 description PC18-PC19
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
!
end
```

---

## 4. Services (GUI-based devices — no CLI)

### 4.1 DHCP-DNS Server (192.168.40.10)

Desktop → IP Configuration: static IP 192.168.40.10 /24, gateway 192.168.40.1, DNS 192.168.40.10.
Services → DHCP: create one pool per subnet, DHCP service **On**.

| Pool name | Network | Mask | Default gateway | DNS server | Start IP | Max users |
|---|---|---|---|---|---|---|
| POOL-HR | 192.168.10.0 | 255.255.255.0 | 192.168.10.1 | 192.168.40.10 | 192.168.10.100 | 100 |
| POOL-FIN | 192.168.20.0 | 255.255.255.0 | 192.168.20.1 | 192.168.40.10 | 192.168.20.100 | 100 |
| POOL-IT | 192.168.30.0 | 255.255.255.0 | 192.168.30.1 | 192.168.40.10 | 192.168.30.100 | 100 |
| POOL-VOICE | 192.168.50.0 | 255.255.255.0 | 192.168.50.1 | 192.168.40.10 | 192.168.50.100 | 100 (set TFTP server = 10.10.10.10) |
| POOL-WIFI-STAFF | 192.168.60.0 | 255.255.255.0 | 192.168.60.1 | 192.168.40.10 | 192.168.60.100 | 100 |
| POOL-WIFI-GUEST | 192.168.70.0 | 255.255.255.0 | 192.168.70.1 | 192.168.40.10 | 192.168.70.100 | 100 |
| POOL-BR1-DATA | 192.168.111.0 | 255.255.255.0 | 192.168.111.1 | 192.168.40.10 | 192.168.111.100 | 50 |
| POOL-BR1-VOICE | 192.168.115.0 | 255.255.255.0 | 192.168.115.1 | 192.168.40.10 | 192.168.115.100 | 50 (TFTP = 10.10.10.10) |
| POOL-BR2-DATA | 192.168.121.0 | 255.255.255.0 | 192.168.121.1 | 192.168.40.10 | 192.168.121.100 | 50 |
| POOL-BR2-VOICE | 192.168.125.0 | 255.255.255.0 | 192.168.125.1 | 192.168.40.10 | 192.168.125.100 | 50 (TFTP = 10.10.10.10) |

Services → DNS: **On**, add A records:

| Name | Type | Address |
|---|---|---|
| www.securecorp.local | A | 192.168.40.20 |
| ftp.securecorp.local | A | 192.168.40.20 |
| mail.securecorp.local | A | 192.168.40.30 |

### 4.2 HTTP-FTP Server (192.168.40.20)

Static IP 192.168.40.20/24, gateway 192.168.40.1, DNS 192.168.40.10.
Services → HTTP: **On**, edit index.html with your company homepage.
Services → FTP: **On**, create user `admin` / password `Cisco@123` with Read/Write/Delete/Rename/List permissions.

### 4.3 EMAIL-SERVER (192.168.40.30)

Static IP 192.168.40.30/24, gateway 192.168.40.1, DNS 192.168.40.10.
Services → EMAIL: **On**, domain name `securecorp.local`, SMTP + POP3 enabled.
Create user accounts, e.g. `hr1`, `finance1`, `it1`, each with a password.
On each PC: Desktop → E Mail, configure with the matching username, server = `mail.securecorp.local` (or 192.168.40.30), incoming POP3 / outgoing SMTP.

### 4.4 Wireless LAN Controller (WLC) — full setup + test walkthrough

Everything here is done through the WLC's GUI (click the device → GUI tab), not CLI.

**Step 1 — Give the WLC a management IP**
- Click the WLC → **Config** tab → **Interfaces** → Management interface
- IP address: `192.168.60.5`, subnet `255.255.255.0`, gateway `192.168.60.1`, VLAN ID `60`, Port Num `1` (matches the physical Gi1 link to Core Switch 2 Gi1/0/6)
- Apply. The WLC should now be reachable — verify by pinging `192.168.60.5` from a PC in VLAN 60 (once a staff device exists there).

**Step 2 — Create the two WLANs**
Go to the **WLAN** tab → New:

| Field | WLAN 1 (Staff) | WLAN 2 (Guest) |
|---|---|---|
| Profile Name | Staff-WLAN | Guest-WLAN |
| SSID | SecureCorp-Staff | SecureCorp-Guest |
| WLAN ID | 1 | 2 |
| Interface/Interface Group | Create a new dynamic interface `staff-vlan60` mapped to VLAN 60 | Create a new dynamic interface `guest-vlan70` mapped to VLAN 70 |
| Security → Layer 2 | WPA2-PSK, passphrase `Staff@2026` | WPA2-PSK, passphrase `Guest@2026` (or Open, if your instructor wants a true open guest network) |
| Status | Enabled | Enabled |

For each new **dynamic interface** (staff-vlan60 / guest-vlan70), give it an IP in that VLAN's range too, e.g. `192.168.60.6/24` gw `.1` for staff, `192.168.70.6/24` gw `.1` for guest — this is separate from the management interface and is what actually carries client traffic on that VLAN.

**Step 3 — Let the two access points join**
- Access Point0 and Access Point1 must be on ports trunked for VLANs 60/70 (already true per your cabling — Core1 Gi1/0/10 and Core2 Gi1/0/8)
- Power-cycle both APs after the WLC is configured — in lightweight mode they'll auto-discover the WLC on the same L2/L3 reachable network and pull their config (both SSIDs get broadcast from both APs automatically once they join)
- Confirm on the WLC: **Monitor** tab → Access Points → both APs should show "Registered"

**Step 4 — Test each SSID separately**

*Staff SSID test (full internal access expected):*
1. Add a laptop or use Smartphone0, connect to `SecureCorp-Staff` with passphrase `Staff@2026`
2. `ipconfig` — should get an address in `192.168.60.0/24`, gateway `192.168.60.1`
3. `ping 192.168.10.x` (an HR PC) — should **succeed** (staff VLAN isn't restricted)
4. `ping 8.8.8.8` — should **succeed**

*Guest SSID test (internal access should be blocked):*
1. Connect a second client (or switch the same one) to `SecureCorp-Guest` with passphrase `Guest@2026`
2. `ipconfig` — should get an address in `192.168.70.0/24`, gateway `192.168.70.1`
3. `ping 192.168.10.x` (an HR PC) — should **fail** (blocked by `GUEST-RESTRICT` ACL on Core2's Vlan70)
4. `ping 8.8.8.8` — should still **succeed** (guest still gets internet, just not internal access)

If a client can't pull an IP on either SSID, double-check the dynamic interface's VLAN mapping in Step 2 matches the trunk's allowed VLANs, and that `ip helper-address 192.168.40.10` is present on both Core switches' Vlan60/Vlan70 SVIs (already in this doc's section 3.7/3.8).

### 4.5 Branch Access Points (autonomous)

Each branch AP is standalone (no WLC at the branch) — you've already confirmed these work. For reference, they're configured directly on the AP GUI:
- SSID: `SecureCorp-Branch1` (or `-Branch2`)
- VLAN/port: access VLAN 10, same subnet as the branch LAN
- Security: WPA2-PSK, passphrase `Branch@2026`

---

## 5. Verification checklist

| Check | Command |
|---|---|
| OSPF neighbors up | `show ip ospf neighbor` |
| Full routing table, no missing subnets | `show ip route` |
| EtherChannel bundled (both members `P`) | `show etherchannel summary` |
| HSRP active/standby correct per VLAN | `show standby brief` |
| NAT translations building | `show ip nat translations` |
| Trunk VLANs allowed correctly | `show interfaces trunk` |
| Port security learned MACs | `show port-security` |
| Phones registered to CME | `show ephone registered` |
| PCs pulling correct IP/gateway/DNS | `ipconfig /all` on each PC |
| Internet reachable from an internal PC | `ping 100.64.0.1` and `ping 8.8.8.8`-style test to ISP loopback |
| Guest Wi-Fi blocked from HR/Finance | ping from a guest-VLAN host to 192.168.10.x — should fail |

---

## 6. How to test the Guest Wi-Fi restriction

You have no client on VLAN 70 (Guest) yet — Smartphone0 is currently associated with Access Point0 on whichever SSID it was set up with, but that needs to specifically be the **Guest** SSID for this test to mean anything. Two options:

**Option A — use Smartphone0:**
1. Click Smartphone0 → Desktop → PC Wireless (or the phone's Wi-Fi settings)
2. Connect it to SSID `SecureCorp-Guest` (passphrase `Guest@2026`) instead of the staff SSID
3. Confirm it pulls an address in `192.168.70.0/24` via `ipconfig`
4. From the smartphone's terminal/command app, `ping 192.168.10.x` (any HR PC) — **should time out** (blocked by `GUEST-RESTRICT` on Core2's Vlan70)
5. `ping 8.8.8.8` — **should succeed** (guest traffic is still allowed to the internet, just not to internal VLANs)

**Option B — add a dedicated guest laptop (recommended for your defense, since it's visually clearer than a phone):**
1. Drag a **Laptop-PT** into the topology near Access Point0 or Access Point1
2. Desktop → PC Wireless → connect to `SecureCorp-Guest`
3. Same two pings as above to demonstrate the block live

If the guest ping to HR *succeeds* instead of failing, the most likely cause is the `GUEST-RESTRICT` ACL not actually applied — run `show ip interface Vlan70` on **Core Switch 2** (it's the HSRP active router for VLAN 70) and confirm you see `Outgoing access list is not set` / `Inbound access list is GUEST-RESTRICT`. If it's missing, re-apply:
```
interface Vlan70
 ip access-group GUEST-RESTRICT in
```

---

## 7. Using every VLAN you've built — what's tested vs. still needs a device

| VLAN | Purpose | Current test coverage | What to add |
|---|---|---|---|
| 10 — HR | ✅ Tested | PC0–PC5 confirmed pulling DHCP, pinging out | None needed |
| 20 — FINANCE | ✅ Tested | PC6–PC11 confirmed | None needed |
| 30 — IT | ✅ Tested | PC12–PC17 confirmed | None needed |
| 40 — SERVERS | ✅ In use | DHCP-DNS, HTTP-FTP, EMAIL-SERVER all live | Test HTTP/FTP/Email from a PC browser & email client (see below) |
| 50 — VOICE | ✅ Tested | 2 of 3 phones registered to CME | Add the HR phone's `ephone 3` (section 3.6) |
| 60 — WIFI-STAFF | ⚠️ Not yet tested | No confirmed staff wireless client | Add a laptop/phone, connect to `SecureCorp-Staff` SSID, confirm DHCP + full internal access |
| 70 — WIFI-GUEST | ⚠️ Not yet tested | No confirmed guest client | See section 6 above — this is your key demo for the Security requirement |
| 100 — MGMT | ⚠️ Optional | No host lives here by design | This VLAN is for switch/router management traffic, not end users — nothing to add unless your instructor wants a dedicated "NOC PC" for SSH-only access demos |
| Branch1 (111/115) | ✅ Tested | PC20/PC21 + IP Phone1 confirmed | None needed |
| Branch2 (121/125) | ✅ Tested | PC18/PC19 + IP Phone2 confirmed | None needed |

**To test the services (VLAN 40) from any HQ PC:**
- Web: open a browser on the PC, go to `http://www.securecorp.local` or `http://192.168.40.20`
- FTP: `ftp 192.168.40.20`, log in with the `admin` account you created
- Email: Desktop → E Mail, configure with a user you created on EMAIL-SERVER, send a test message between two PCs
- DNS: `nslookup www.securecorp.local` from a PC's command prompt — should resolve to `192.168.40.20`

---

## 8. Full requirement coverage — where each of your 10 items lives

| # | Requirement | Status | Evidence / where to find it |
|---|---|---|---|
| 1 | Routed | ✅ Done | Every VLAN routed at Core1/Core2 SVIs; branches routed via router-on-a-stick |
| 2 | Routing (Static + OSPF) | ✅ Confirmed working | Your `show ip ospf neighbor` / `show ip route` outputs — full adjacencies, correct costs, static default routes on Edge1/Edge2 redistributed as E2 |
| 3 | Security | ⚠️ Mostly done, 1 gap to verify | Port security confirmed on every access switch (your `show port-security` output). SSH/AAA/enable secret configured on all L3 devices (section 3). Guest ACL configured — **verify it's actually applied and test it** (section 6). DHCP snooping was mentioned as a design goal but never added to any config — optional addition below if your rubric expects it |
| 4 | VLAN | ✅ Done | 8 VLANs + native, all confirmed via `show interfaces trunk` |
| 5 | Services | ✅ Live, needs a demo test | DHCP now fixed with helper-address; DNS/HTTP/FTP/Email configured — walk through section 7's test steps once before your defense |
| 6 | EtherChannel | ✅ Confirmed working | `Po1(SU)` with all 3 links `(P)` on both core switches |
| 7 | FHRP (HSRP) | ✅ Confirmed working | `show standby brief` — clean active/standby split, preempt working |
| 8 | WLAN | ⚠️ Needs verification | WLC and branch AP GUI config is written in section 4.4/4.5 — confirm you've actually applied it in Packet Tracer and add a test client per section 6/7 |
| 9 | NAT | ✅ Confirmed on Edge1, verify Edge2 | Edge1's `show ip nat translations` proven. Generate traffic from an IT/Finance PC (Core2 side) and check `show ip nat translations` on Edge Router 2 the same way |
| 10 | IP Phone | ✅ 2 of 3 working | Branch1 (1001) and Branch2 (2001) registered and confirmed. Add HR phone (`ephone 3`) to finish this — section 3.6 |

### Optional: DHCP snooping (if your grading rubric references it)
Add to each access switch (SW-HR, SW-FINANCE, SW-IT, SW-BRANCH1, SW-BRANCH2) and both Core switches:
```
ip dhcp snooping
ip dhcp snooping vlan 10,20,30,50,60,70
!
interface GigabitEthernet1/0/8    ! the port facing the real DHCP server — trusted
 ip dhcp snooping trust
```
(On Core1, that's the server-facing port Gi1/0/8; on access switches, no `trust` command is needed on end-user ports — they're untrusted by default, which is the whole point.)
