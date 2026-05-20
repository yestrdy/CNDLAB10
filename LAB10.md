# LAB 10 — IPv6 OSPFv3 БҮРЭН ТОХИРГОО (ЗАСВАРЛАСАН)

Lab 6-н топологи дээр хийсэн. R4 ↔ R2 virtual-link холбогдох баталгаатай.

```
============================================================
ТОПОЛОГИ
============================================================
  Area 0 (Backbone): R1, R3, R4
  Area 1 (Transit):  R2, R4   (ABR: R4)
  Area 2:            R2, R5, R6   (ABR: R2 — VL-ээр Area 0-д холбогдоно)
  Virtual-Link:      R4 (4.4.4.4)  <-->  R2 (2.2.2.2)
                     Transit area = Area 1

============================================================
IPv6 ХАЯГЛАЛТ
============================================================
  R1 Lo0:   2001:DB8:0:1::1/128       (Area 0)
  R3 Lo0:   2001:DB8:0:3::3/128       (Area 0)
  R4 Lo0:   2001:DB8:0:4::4/128       (Area 0)
  R2 Lo0:   2001:DB8:0:2::2/128       (Area 1)
  R5 Lo0:   2001:DB8:0:5::5/128       (Area 2)
  R6 Lo0:   2001:DB8:0:6::6/128       (Area 2)

  R1-R3 link (Area 0): 2001:DB8:13::/64
  R1-R4 link (Area 0): 2001:DB8:14::/64
  R3-R4 link (Area 0): 2001:DB8:34::/64
  R4-R2 link (Area 1): 2001:DB8:42::/64   ← VL transit
  R2-R5 link (Area 2): 2001:DB8:25::/64
  R2-R6 link (Area 2): 2001:DB8:26::/64
  R5-R6 link (Area 2): 2001:DB8:56::/64
  R6 Summary (Area 2): 2001:DB8:6::/48     (Lo1-Lo4)

============================================================
IPSEC AUTHENTICATION SPI ХУВААРЬ
============================================================
  Area 0       : SPI 256
  Area 1       : SPI 257
  Virtual-Link : SPI 258
  Area 2       : SPI 259
  Key (SHA1, 40 hex):
       1234567890ABCDEF1234567890ABCDEF12345678
```

---

## 🔧 R1 — Area 0 internal

```cisco
hostname R1
!
ipv6 unicast-routing
ipv6 cef
!
no ip domain lookup
banner motd ^C Unauthorized access is prohibited! ^C
enable secret cisco123
!
interface Loopback0
 ipv6 address 2001:DB8:0:1::1/128
 ipv6 ospf 1 area 0
 no shutdown
!
interface FastEthernet0/0
 description Link-to-R3
 ipv6 address 2001:DB8:13::1/64
 ipv6 ospf 1 area 0
 ipv6 ospf network point-to-point
 no shutdown
!
interface FastEthernet0/1
 description Link-to-R4
 ipv6 address 2001:DB8:14::1/64
 ipv6 ospf 1 area 0
 ipv6 ospf network point-to-point
 no shutdown
!
ipv6 router ospf 1
 router-id 1.1.1.1
 log-adjacency-changes
 passive-interface Loopback0
 timers throttle spf 50 100 5000
 area 0 authentication ipsec spi 256 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 fast-reroute per-prefix enable prefix-priority low
!
line con 0
 exec-timeout 0 0
 privilege level 15
 password cisco
 logging synchronous
 login
line vty 0 4
 password cisco
 login
!
end
```

---

## 🔧 R3 — Area 0 internal

```cisco
hostname R3
!
ipv6 unicast-routing
ipv6 cef
!
no ip domain lookup
banner motd ^C Unauthorized access is prohibited! ^C
enable secret cisco123
!
interface Loopback0
 ipv6 address 2001:DB8:0:3::3/128
 ipv6 ospf 1 area 0
 no shutdown
!
interface FastEthernet0/0
 description Link-to-R1
 ipv6 address 2001:DB8:13::3/64
 ipv6 ospf 1 area 0
 ipv6 ospf network point-to-point
 no shutdown
!
interface FastEthernet0/1
 description Link-to-R4
 ipv6 address 2001:DB8:34::3/64
 ipv6 ospf 1 area 0
 ipv6 ospf network point-to-point
 no shutdown
!
ipv6 router ospf 1
 router-id 3.3.3.3
 log-adjacency-changes
 passive-interface Loopback0
 timers throttle spf 50 100 5000
 area 0 authentication ipsec spi 256 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 fast-reroute per-prefix enable prefix-priority low
!
line con 0
 exec-timeout 0 0
 privilege level 15
 password cisco
 logging synchronous
 login
line vty 0 4
 password cisco
 login
!
end
```

---

## 🔧 R4 — ABR (Area 0 + Area 1) + VL endpoint

```cisco
hostname R4
!
ipv6 unicast-routing
ipv6 cef
!
no ip domain lookup
banner motd ^C Unauthorized access is prohibited! ^C
enable secret cisco123
!
interface Loopback0
 ipv6 address 2001:DB8:0:4::4/128
 ipv6 ospf 1 area 0
 no shutdown
!
interface FastEthernet0/0
 description Link-to-R1 (Area0)
 ipv6 address 2001:DB8:14::4/64
 ipv6 ospf 1 area 0
 ipv6 ospf network point-to-point
 no shutdown
!
interface FastEthernet0/1
 description Link-to-R3 (Area0)
 ipv6 address 2001:DB8:34::4/64
 ipv6 ospf 1 area 0
 ipv6 ospf network point-to-point
 no shutdown
!
interface FastEthernet1/0
 description Link-to-R2 (Area1 - VL transit)
 ipv6 address 2001:DB8:42::4/64
 ipv6 ospf 1 area 1
 ipv6 ospf network point-to-point
 no shutdown
!
ipv6 router ospf 1
 router-id 4.4.4.4
 log-adjacency-changes
 passive-interface Loopback0
 timers throttle spf 50 100 5000
 !
 ! --- Area auth ---
 area 0 authentication ipsec spi 256 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 area 1 authentication ipsec spi 257 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 !
 ! --- Virtual-Link to R2 (2 МӨР!) ---
 area 1 virtual-link 2.2.2.2
 area 1 virtual-link 2.2.2.2 authentication ipsec spi 258 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 !
 ! --- Fast-Reroute (LFA) ---
 fast-reroute per-prefix enable prefix-priority low
!
line con 0
 exec-timeout 0 0
 privilege level 15
 password cisco
 logging synchronous
 login
line vty 0 4
 password cisco
 login
!
end
```

---

## 🔧 R2 — ABR (Area 1 + Area 2) + VL endpoint

```cisco
hostname R2
!
ipv6 unicast-routing
ipv6 cef
!
no ip domain lookup
banner motd ^C Unauthorized access is prohibited! ^C
enable secret cisco123
!
interface Loopback0
 ipv6 address 2001:DB8:0:2::2/128
 ipv6 ospf 1 area 1
 no shutdown
!
interface FastEthernet0/0
 description Link-to-R4 (Area1 - VL transit)
 ipv6 address 2001:DB8:42::2/64
 ipv6 ospf 1 area 1
 ipv6 ospf network point-to-point
 no shutdown
!
interface FastEthernet0/1
 description Link-to-R5 (Area2)
 ipv6 address 2001:DB8:25::2/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
 no shutdown
!
interface FastEthernet1/0
 description Link-to-R6 (Area2)
 ipv6 address 2001:DB8:26::2/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
 no shutdown
!
ipv6 router ospf 1
 router-id 2.2.2.2
 log-adjacency-changes
 passive-interface Loopback0
 timers throttle spf 50 100 5000
 !
 ! --- Area auth ---
 area 1 authentication ipsec spi 257 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 area 2 authentication ipsec spi 259 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 !
 ! --- Virtual-Link to R4 (2 МӨР!) ---
 area 1 virtual-link 4.4.4.4
 area 1 virtual-link 4.4.4.4 authentication ipsec spi 258 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 !
 ! --- Summarization ---
 area 2 range 2001:DB8:6::/48
 !
 ! --- Fast-Reroute ---
 fast-reroute per-prefix enable prefix-priority low
!
line con 0
 exec-timeout 0 0
 privilege level 15
 password cisco
 logging synchronous
 login
line vty 0 4
 password cisco
 login
!
end
```

---

## 🔧 R5 — Area 2 internal

```cisco
hostname R5
!
ipv6 unicast-routing
ipv6 cef
!
no ip domain lookup
banner motd ^C Unauthorized access is prohibited! ^C
enable secret cisco123
!
interface Loopback0
 ipv6 address 2001:DB8:0:5::5/128
 ipv6 ospf 1 area 2
 no shutdown
!
interface FastEthernet0/0
 description Link-to-R6 (Area2)
 ipv6 address 2001:DB8:56::5/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
 no shutdown
!
interface FastEthernet0/1
 description Link-to-R2 (Area2)
 ipv6 address 2001:DB8:25::5/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
 no shutdown
!
ipv6 router ospf 1
 router-id 5.5.5.5
 log-adjacency-changes
 passive-interface Loopback0
 timers throttle spf 50 100 5000
 area 2 authentication ipsec spi 259 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 fast-reroute per-prefix enable prefix-priority low
!
line con 0
 exec-timeout 0 0
 privilege level 15
 password cisco
 logging synchronous
 login
line vty 0 4
 password cisco
 login
!
end
```

---

## 🔧 R6 — Area 2 internal (Summarization эх)

```cisco
hostname R6
!
ipv6 unicast-routing
ipv6 cef
!
no ip domain lookup
banner motd ^C Unauthorized access is prohibited! ^C
enable secret cisco123
!
interface Loopback0
 description Router-ID loopback
 ipv6 address 2001:DB8:0:6::6/128
 ipv6 ospf 1 area 2
 no shutdown
!
! Summarization-д зориулсан loopback-ууд (2001:db8:6::/48 доторх)
! ВАЖНО: "ipv6 ospf network point-to-point" — /64 болгож зарлахын тулд
interface Loopback1
 ipv6 address 2001:DB8:6:0::1/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
 no shutdown
!
interface Loopback2
 ipv6 address 2001:DB8:6:1::1/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
 no shutdown
!
interface Loopback3
 ipv6 address 2001:DB8:6:2::1/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
 no shutdown
!
interface Loopback4
 ipv6 address 2001:DB8:6:3::1/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
 no shutdown
!
interface FastEthernet0/0
 description Link-to-R5 (Area2)
 ipv6 address 2001:DB8:56::6/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
 no shutdown
!
interface FastEthernet0/1
 description Link-to-R2 (Area2)
 ipv6 address 2001:DB8:26::6/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
 no shutdown
!
ipv6 router ospf 1
 router-id 6.6.6.6
 log-adjacency-changes
 passive-interface Loopback0
 passive-interface Loopback1
 passive-interface Loopback2
 passive-interface Loopback3
 passive-interface Loopback4
 timers throttle spf 50 100 5000
 area 2 authentication ipsec spi 259 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 fast-reroute per-prefix enable prefix-priority low
!
line con 0
 exec-timeout 0 0
 privilege level 15
 password cisco
 logging synchronous
 login
line vty 0 4
 password cisco
 login
!
end
```

---

## ✅ ШАЛГАХ ДАРААЛАЛ (Virtual-Link зорилго)

### 1. Эхлээд Area 1 хөрш үүссэн эсэхийг R4/R2 дээр шалгана:
```
R4# show ipv6 ospf neighbor

Neighbor ID     Pri   State           Dead Time   Interface ID    Interface
2.2.2.2           1   FULL/  -        00:00:38    4               FastEthernet1/0
```

### 2. Virtual-link шалгах (R4 дээр):
```
R4# show ipv6 ospf virtual-links

Virtual Link OSPFv3_VL0 to router 2.2.2.2 is up
  Interface ID 67, IPv6 address FE80::...
  Run as demand circuit
  DoNotAge LSA allowed.
  Transit area 1, via interface FastEthernet1/0
  Transmit Delay is 1 sec, State POINT_TO_POINT,
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
  Adjacency State FULL (Hello suppressed)
```

R2 дээр мөн адил `is up` гарах ёстой.

### 3. R2 нь ABR болсон эсэхийг шалгах:
```
R2# show ipv6 ospf | include area
   It is an area border router
   Number of areas in this router is 3. 3 normal 0 stub 0 nssa
```
**3 area** (Area 0 — VL-ээс, Area 1, Area 2) гарвал VL ажиллаж байна.

### 4. R1/R3-аас R5/R6 рүү reachability:
```
R1# ping ipv6 2001:DB8:0:5::5
R1# ping ipv6 2001:DB8:0:6::6
R1# show ipv6 route 2001:DB8:6::/48
```
Area 2-н summary route нь `OI` (inter-area) маркаар R1 дээр харагдах ёстой.

### 5. Authentication шалгах:
```
R4# show ipv6 ospf interface FastEthernet1/0 | include auth
R4# show crypto ipsec sa
```

---

## 🛠 ЕСЛИ VL ХОЛБОГДОХГҮЙ БОЛ — Troubleshooting

| Шинж тэмдэг | Шалтгаан | Засвар |
|---|---|---|
| `show ipv6 ospf virtual-links` → DOWN | R4-R2 Area 1 adjacency үүсээгүй | Эхлээд `show ipv6 ospf neighbor` дээр FULL эсэх шалга |
| Adjacency үүсэхгүй | Area 1 auth SPI/key зөрүү | `area 1 authentication ipsec spi 257 sha1 ...` хоёр талд адил эсэхийг шалга |
| VL adjacency Init/Exchange-д хоригдсон | VL auth SPI зөрүү | `area 1 virtual-link X authentication ipsec spi 258 sha1 ...` хоёр талд адил эсэхийг шалга |
| `% OSPF: Cannot configure...` алдаа | VL command нэг мөрөнд бичсэн | Эхлээд `area 1 virtual-link 2.2.2.2`, дараа нь `... authentication ipsec ...` |
| LSA-уд Area 0-д тарахгүй | R2 ABR биш | VL хүлээж байна — Area 1 adj болон VL хоёулаа FULL болсон эсэхийг шалга |

---

## 📋 LAB 6 CONFIG → LAB 10 CONFIG ГОЛ ӨӨРЧЛӨЛТҮҮД

| Юу нэмсэн | Команд | Зорилго |
|---|---|---|
| IPv6 routing | `ipv6 unicast-routing`, `ipv6 cef` | IPv6 идэвхжүүлэх |
| OSPFv3 process | `ipv6 router ospf 1` | OSPFv3-н процесс |
| Per-area auth | `area X authentication ipsec spi N sha1 KEY` | Route injection-аас сэргийлэх |
| VL auth | `area 1 virtual-link X authentication ipsec spi 258 ...` | Backbone бүрэн бүтэн байх |
| SPF throttle | `timers throttle spf 50 100 5000` | Convergence хурдан |
| LFA | `fast-reroute per-prefix enable prefix-priority low` | <50ms failover |
| Summarization | `area 2 range 2001:DB8:6::/48` | LSDB багасгах |
| Virtual-Link | `area 1 virtual-link 2.2.2.2/4.4.4.4` | Area 2-г backbone-д холбох |
| P2P network | `ipv6 ospf network point-to-point` | DR/BDR delay арилгах + loopback /64 болгох |
