# Компьютерын сүлжээний төхөөрөмж

## ECEN322

## Лабораторийн ажил № 4

## Multi-Area OSPFv3 (IPv6) зохион байгуулалт

**Тэнхим:** МТЭС, ЭХИТ, Компьютерын сүлжээ  
**Оюутан:** Д.Билгүүнтөгс, 24B1NUM0440

---

## Онолын судалгаа

Динамик чиглүүлэлтийн протоколын нэг болох OSPF (Open Shortest Path First) протоколын дунд
түвшний тохиргоо хийж сурах. Энэхүү ажлын хүрээнд Лондонгийн хөрөнгийн биржийн
хувьцааны арилжаанд орох шаардлагатай болсон А байгууллагын сүлжээний OSPF
чиглүүлэлтийн аюулгүй байдлыг сайжруулах, тасралтгүй ажиллагааг (FRR) нэмэгдүүлэх,
процессийн ачааллыг бууруулах, алслагдсан бүсийг Virtual-Link ашиглан холбох
тохиргоонуудыг бодит болон виртуал төхөөрөмж (GNS3) хослуулан хийж гүйцэтгэв.

---

## 1. Даалгавар

А байгууллагын 6 рутер бүхий сүлжээнд дараах сайжруулалтыг хийх:

- Зөвхөн IPv6 ашиглан хаяглалт хийх
- OSPF-ийн холболтод зөвшөөрөгдөөгүй route injection үйлдэл хийгдэхээс сэргийлэх.
- Зам сонгож чиглүүлэлтийн хүснэгтэд оруулахад зарцуулах хугацааг хамгийн бага байлгах.
- Хэвийн ажиллагааны үед доголдол үүсэхэд нөхөн сэргээх хугацааг (failover) хамгийн бага байлгах.
- Чиглүүлэлт хийх үеийн процессийн delay-г бууруулах зорилгоор summarization ашиглах.
- Multi-area OSPF-ийг өргөтгөх боломжтой байлгах.
- Virtual link ашиглан Area 0-д холбогдоогүй сүлжээг холбох.
- Тохиргоог TFTP серверт хадгалах.


### 1.1 Сүлжээний топологи

```
         R1 ──────── R3 ────[Area 0]──── R4 ──────── R5 ──── Cloud1 (Vmnet8)
        /              \                  / \          / \
       /                \                /   \        /   \
      R2                 (backbone)     /     \      /     \
       \                              R6       \   R6    (external)
        \                              (Area2)  (Area3)
```

**Зураг 1.** Байгууллагын сүлжээний ерөнхий топологи (GNS3)

**Сүлжээний бүтэц:**

Сүлжээний топологи нь олон бүст (Multi-Area) бүтэцтэй бөгөөд Area 0 (Backbone), Area 1, Area 2,
Area 3 гэсэн хэсгүүдээс бүрдэнэ. R3 болон R4 рутерүүд нь ABR (Area Border Router), R5 нь
ABR болон ASBR (AS Boundary Router) үүрэг гүйцэтгэнэ.

| Area | Гишүүн router/link | ABR/ASBR |
|------|-------------------|----------|
| Area 0 (Backbone) | R3↔R4, R3 Lo0, R4 Lo0 | — |
| Area 1 | R1↔R2, R1↔R3, R2↔R3 | R3 (ABR) |
| Area 2 | R4↔R5, R4↔R6 | R4 (ABR) |
| Area 3 | R5↔R6 | R5 (ABR), Virtual-link-ээр Area 0-той холбогдсон |
| External | R5↔Cloud1 (Vmnet8) | R5 (ASBR) |

**IPv6 хаяглалтын төлөвлөгөө:**

| Холболт | IPv6 Subnet | Area |
|---------|-------------|------|
| R1↔R2 | 2001:DB8:1:12::/64 | 1 |
| R1↔R3 | 2001:DB8:1:13::/64 | 1 |
| R2↔R3 | 2001:DB8:1:23::/64 | 1 |
| R3↔R4 | 2001:DB8:0:34::/64 | 0 |
| R4↔R5 | 2001:DB8:2:45::/64 | 2 |
| R4↔R6 | 2001:DB8:2:46::/64 | 2 |
| R5↔R6 | 2001:DB8:3:56::/64 | 3 |
| R5↔Cloud1 | 2001:DB8:CC::/64 | External |

| Router | Loopback IPv6 | Router-ID |
|--------|---------------|-----------|
| R1 | 2001:DB8:FF::1/128 | 1.1.1.1 |
| R2 | 2001:DB8:FF::2/128 | 2.2.2.2 |
| R3 | 2001:DB8:FF::3/128 | 3.3.3.3 |
| R4 | 2001:DB8:FF::4/128 | 4.4.4.4 |
| R5 | 2001:DB8:FF::5/128 | 5.5.5.5 |
| R6 | 2001:DB8:FF::6/128 | 6.6.6.6 |


### 1.2 Үндсэн болон аюулгүй байдлын тохиргоо

Төхөөрөмжүүдийн аюулгүй байдлыг хангах үүднээс сүлжээнд зөвшөөрөлгүй рутер холбогдож
route injection оруулахаас сэргийлж бүх Area дээр IPsec SHA-1 authentication-ийг тохируулав.

| Area | SPI | Хамгаалалт |
|------|-----|-----------|
| Area 0 | 1000 | SHA-1 (40 hex key) |
| Area 1 | 1001 | SHA-1 (40 hex key) |
| Area 2 | 1002 | SHA-1 (40 hex key) |
| Area 3 | 1003 | SHA-1 (40 hex key) |

Fast-Reroute тайлбар: Хэвийн ажиллагааны үед доголдол үүсэхэд нөхөн сэргээх хугацааг багасгах
зорилготой OSPF Fast-Reroute командыг (`fast-reroute per-prefix enable prefix-priority low`) тохируулах
шаардлагатай боловч GNS3 орчин дахь хуучин IOS хувилбар нь уг технологийг бүрэн дэмжихгүй
байсан тул онолын түвшинд судалж, бодит орчинд хэрэгжүүлэх боломжтойг баталгаажуулав.

---

#### R1 рутер дээр хийсэн тохиргоонууд:

**Зураг 2.** R1 дээрх "show running-config" — Basic тохиргоо

```
hostname R1
!
no aaa new-model
memory-size iomem 5
no ip icmp rate-limit unreachable
ip cef
!
no ip domain lookup
ip auth-proxy max-nodata-conns 3
ip admission max-nodata-conns 3
!
ipv6 unicast-routing
ipv6 cef
```

**Зураг 3.** R1 дээрх "show running-config" — Interface тохиргоо

```
interface Loopback0
 no ip address
 ipv6 address 2001:DB8:FF::1/128
 ipv6 ospf 1 area 1
!
interface FastEthernet0/0
 no ip address
 duplex auto
 speed auto
 ipv6 address 2001:DB8:1:12::1/64
 ipv6 ospf network point-to-point
 ipv6 ospf 1 area 1
!
interface FastEthernet0/1
 description To-R3
 no ip address
 duplex auto
 speed auto
 ipv6 address 2001:DB8:1:13::1/64
 ipv6 ospf network point-to-point
 ipv6 ospf 1 area 1
```


**Зураг 4.** R1 дээрх "show running-config" — OSPF тохиргоо

```
ipv6 router ospf 1
 router-id 1.1.1.1
 log-adjacency-changes
 area 1 authentication ipsec spi 1001 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 passive-interface Loopback0
```

---

#### R2 рутер дээр хийсэн тохиргоонууд:

**Зураг 5.** R2 дээрх "show running-config" — Basic тохиргоо

```
hostname R2
!
no aaa new-model
memory-size iomem 5
no ip icmp rate-limit unreachable
ip cef
!
no ip domain lookup
ip auth-proxy max-nodata-conns 3
ip admission max-nodata-conns 3
!
ipv6 unicast-routing
ipv6 cef
```

**Зураг 6.** R2 дээрх "show running-config" — Interface тохиргоо

```
interface Loopback0
 no ip address
 ipv6 address 2001:DB8:FF::2/128
 ipv6 ospf 1 area 1
!
interface FastEthernet0/0
 no ip address
 duplex auto
 speed auto
 ipv6 address 2001:DB8:1:12::2/64
 ipv6 ospf network point-to-point
 ipv6 ospf 1 area 1
!
interface FastEthernet0/1
 no ip address
 duplex auto
 speed auto
 ipv6 address 2001:DB8:1:23::2/64
 ipv6 ospf network point-to-point
 ipv6 ospf 1 area 1
```

**Зураг 7.** R2 дээрх "show running-config" — OSPF тохиргоо

```
ipv6 router ospf 1
 router-id 2.2.2.2
 log-adjacency-changes
 area 1 authentication ipsec spi 1001 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 passive-interface Loopback0
```


---

#### R3 рутер дээр хийсэн тохиргоонууд:

**Зураг 8.** R3 дээрх "show running-config" — Basic тохиргоо

```
hostname R3
!
no aaa new-model
memory-size iomem 5
no ip icmp rate-limit unreachable
ip cef
!
no ip domain lookup
ip auth-proxy max-nodata-conns 3
ip admission max-nodata-conns 3
!
ipv6 unicast-routing
ipv6 cef
```

**Зураг 9.** R3 дээрх "show running-config" — Interface тохиргоо

```
interface Loopback0
 no ip address
 ipv6 address 2001:DB8:FF::3/128
 ipv6 ospf 1 area 0
!
interface FastEthernet0/0
 no ip address
 duplex auto
 speed auto
 ipv6 address 2001:DB8:1:13::3/64
 ipv6 ospf network point-to-point
 ipv6 ospf 1 area 1
!
interface FastEthernet0/1
 no ip address
 duplex auto
 speed auto
 ipv6 address 2001:DB8:1:23::3/64
 ipv6 ospf network point-to-point
 ipv6 ospf 1 area 1
!
interface FastEthernet1/0
 no ip address
 duplex auto
 speed auto
 ipv6 address 2001:DB8:0:34::3/64
 ipv6 ospf network point-to-point
 ipv6 ospf 1 area 0
```

**Зураг 10.** R3 дээрх "show running-config" — OSPF тохиргоо

```
ipv6 router ospf 1
 router-id 3.3.3.3
 log-adjacency-changes
 area 0 authentication ipsec spi 1000 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 area 1 range 2001:DB8:1::/48
 area 1 authentication ipsec spi 1001 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 passive-interface Loopback0
```


---

#### R4 рутер дээр хийсэн тохиргоонууд:

**Зураг 11.** R4 дээрх "show running-config" — Basic тохиргоо

```
hostname R4
!
no aaa new-model
memory-size iomem 5
no ip icmp rate-limit unreachable
ip cef
!
no ip domain lookup
ip auth-proxy max-nodata-conns 3
ip admission max-nodata-conns 3
!
ipv6 unicast-routing
ipv6 cef
```

**Зураг 12.** R4 дээрх "show running-config" — Interface тохиргоо

```
interface Loopback0
 no ip address
 ipv6 address 2001:DB8:FF::4/128
 ipv6 ospf 1 area 0
!
interface FastEthernet0/0
 description To-R3-AREA0
 no ip address
 duplex auto
 speed auto
 ipv6 address 2001:DB8:0:34::4/64
 ipv6 ospf network point-to-point
 ipv6 ospf 1 area 0
!
interface FastEthernet0/1
 description To-R6-AREA2
 no ip address
 duplex auto
 speed auto
 ipv6 address 2001:DB8:2:46::4/64
 ipv6 ospf network point-to-point
 ipv6 ospf 1 area 2
!
interface FastEthernet1/0
 description To-R5-AREA2-VLINK
 no ip address
 duplex auto
 speed auto
 ipv6 address 2001:DB8:2:45::4/64
 ipv6 ospf network point-to-point
 ipv6 ospf 1 area 2
```

**Зураг 13.** R4 дээрх "show running-config" — OSPF тохиргоо

```
ipv6 router ospf 1
 router-id 4.4.4.4
 log-adjacency-changes
 area 0 authentication ipsec spi 1000 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 area 2 range 2001:DB8:2::/48
 area 2 authentication ipsec spi 1002 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 area 2 virtual-link 5.5.5.5
 passive-interface Loopback0
```


---

#### R5 рутер дээр хийсэн тохиргоонууд:

**Зураг 14.** R5 дээрх "show running-config" — Basic тохиргоо

```
hostname R5
!
no aaa new-model
memory-size iomem 5
no ip icmp rate-limit unreachable
ip cef
!
no ip domain lookup
ip auth-proxy max-nodata-conns 3
ip admission max-nodata-conns 3
!
ipv6 unicast-routing
ipv6 cef
```

**Зураг 15.** R5 дээрх "show running-config" — Interface тохиргоо

```
interface Loopback0
 no ip address
 ipv6 address 2001:DB8:FF::5/128
 ipv6 ospf 1 area 2
!
interface FastEthernet0/0
 description To-R6-AREA3
 no ip address
 duplex auto
 speed auto
 ipv6 address 2001:DB8:3:56::5/64
 ipv6 ospf network point-to-point
 ipv6 ospf 1 area 3
!
interface FastEthernet0/1
 description To-R4-AREA2
 no ip address
 duplex auto
 speed auto
 ipv6 address 2001:DB8:2:45::5/64
 ipv6 ospf network point-to-point
 ipv6 ospf 1 area 2
!
interface FastEthernet1/0
 description To-Cloud1-Vmnet8-EXTERNAL
 no ip address
 duplex auto
 speed auto
 ipv6 address 2001:DB8:CC::5/64
```

**Зураг 16.** R5 дээрх "show running-config" — OSPF тохиргоо

```
ipv6 route ::/0 FastEthernet1/0
ipv6 router ospf 1
 router-id 5.5.5.5
 log-adjacency-changes
 area 2 authentication ipsec spi 1002 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 area 2 virtual-link 4.4.4.4
 area 3 range 2001:DB8:3::/48
 area 3 authentication ipsec spi 1003 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 default-information originate always metric 1 metric-type 1
 passive-interface FastEthernet1/0
 passive-interface Loopback0
```


---

#### R6 рутер дээр хийсэн тохиргоонууд:

**Зураг 17.** R6 дээрх "show running-config" — Basic тохиргоо

```
hostname R6
!
no aaa new-model
memory-size iomem 5
no ip icmp rate-limit unreachable
ip cef
!
no ip domain lookup
ip auth-proxy max-nodata-conns 3
ip admission max-nodata-conns 3
!
ipv6 unicast-routing
ipv6 cef
```

**Зураг 18.** R6 дээрх "show running-config" — Interface тохиргоо

```
interface Loopback0
 no ip address
 ipv6 address 2001:DB8:FF::6/128
 ipv6 ospf 1 area 3
!
interface FastEthernet0/0
 description To-R4-AREA2
 no ip address
 duplex auto
 speed auto
 ipv6 address 2001:DB8:2:46::6/64
 ipv6 ospf network point-to-point
 ipv6 ospf 1 area 2
!
interface FastEthernet0/1
 description To-R5-AREA3
 no ip address
 duplex auto
 speed auto
 ipv6 address 2001:DB8:3:56::6/64
 ipv6 ospf network point-to-point
 ipv6 ospf 1 area 3
```

**Зураг 19.** R6 дээрх "show running-config" — OSPF тохиргоо

```
ipv6 router ospf 1
 router-id 6.6.6.6
 log-adjacency-changes
 area 2 authentication ipsec spi 1002 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 area 3 authentication ipsec spi 1003 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 passive-interface Loopback0
```


---

### 1.3 Route Summarization тохиргоо

Чиглүүлэлтийн процессийн delay-г бууруулахын тулд ABR рутерүүд дээр area range командыг
ашиглан тус тусын area-ийн сүлжээнүүдийг нэгтгэн нэг summary route болгож зарлав:

| ABR | Тохиргоо | Тайлбар |
|-----|----------|---------|
| R3 | `area 1 range 2001:DB8:1::/48` | Area 1-ийн бүх /64 сүлжээг /48-аар summarize |
| R4 | `area 2 range 2001:DB8:2::/48` | Area 2-ийн бүх /64 сүлжээг /48-аар summarize |
| R5 | `area 3 range 2001:DB8:3::/48` | Area 3-ийн бүх /64 сүлжээг /48-аар summarize |

Ингэснээр бусад Area-ийн рутерүүдийн чиглүүлэлтийн хүснэгт багасаж, CPU ачаалал буурна.

**Зураг 20.** R3 дээрх "show ipv6 route summary" командын үр дүн

```
R3#show ipv6 route summary
IPv6 Routing Table Summary - 22 entries
  6 local, 3 connected, 0 static, 0 RIP, 0 BGP, 0 IS-IS, 13 OSPF
  Number of prefixes:
    /0: 1, /8: 1, /10: 1, /48: 3, /64: 6, /128: 10
```

**Тайлбар:** `/48: 3` гэсэн нь 3 area-ийн summary route (`2001:DB8:1::/48`, `2001:DB8:2::/48`,
`2001:DB8:3::/48`) амжилттай үүссэнийг батална. Нийт 13 OSPF route сурагдсан.

---

### 1.4 Virtual-Link тохиргоо

OSPF-ийн дүрмээр бүх Area нь Area 0-тэй шууд холбогдох ёстой байдаг ч Area 3 (R5↔R6) нь
Area 0-той шууд физик холболтгүй, Area 2-оор тусгаарлагдсан байсан тул R4 болон R5
рутерүүдийн хооронд Area 2-ыг transit area болгон логик туннель буюу Virtual-link үүсгэж холбов.

**R4 дээрх тохиргоо:**
```
ipv6 router ospf 1
 area 2 virtual-link 5.5.5.5
```

**R5 дээрх тохиргоо:**
```
ipv6 router ospf 1
 area 2 virtual-link 4.4.4.4
```

Virtual-link нь Area 2-ийн IPsec authentication (SPI 1002)-ийг inherit хийж ашигладаг тул тусдаа
authentication тохиргоо шаардлагагүй.


**Зураг 21.** R4 дээрх "show ipv6 ospf virtual-links" командын үр дүн

```
R4#show ipv6 ospf virtual-links
Virtual Link OSPFv3_VL0 to router 5.5.5.5 is up
  Interface ID 12, IPv6 address 2001:DB8:FF::5
  Run as demand circuit
  DoNotAge LSA allowed.
  Transit area 2, via interface FastEthernet1/0, Cost of using 1
  Transmit Delay is 1 sec, State POINT_TO_POINT,
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    Adjacency State FULL (Hello suppressed)
    Index 1/2/4, retransmission queue length 0, number of retransmission 0
    First 0x0(0)/0x0(0)/0x0(0) Next 0x0(0)/0x0(0)/0x0(0)
    Last retransmission scan length is 0, maximum is 0
    Last retransmission scan time is 0 msec, maximum is 0 msec
```

**Зураг 22.** R5 дээрх "show ipv6 ospf virtual-links" командын үр дүн

```
R5#show ipv6 ospf virtual-links
Virtual Link OSPFv3_VL0 to router 4.4.4.4 is up
  Interface ID 12, IPv6 address 2001:DB8:2:45::4
  Run as demand circuit
  DoNotAge LSA allowed.
  Transit area 2, via interface FastEthernet0/1, Cost of using 10
  Transmit Delay is 1 sec, State POINT_TO_POINT,
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    Adjacency State FULL (Hello suppressed)
    Index 1/1/3, retransmission queue length 0, number of retransmission 0
    First 0x0(0)/0x0(0)/0x0(0) Next 0x0(0)/0x0(0)/0x0(0)
    Last retransmission scan length is 0, maximum is 0
    Last retransmission scan time is 0 msec, maximum is 0 msec
```

**Тайлбар:** Хоёр тал дээр `Adjacency State FULL` гэж харагдаж байна нь Virtual-link амжилттай
ажиллаж, Area 3 нь Area 0-тэй логикоор холбогдсоныг батална. `DoNotAge LSA allowed` нь
virtual-link-ээр дамжих LSA-ууд age-ийг нэмэгдүүлэхгүй (DNA flag) байхыг зөвшөөрч байна.


---

### 1.5 OSPF Neighbor баталгаажуулалт

Бүх 6 рутер дээр `show ipv6 ospf neighbor` командыг ажиллуулж, бүх adjacency FULL төлөвт
байгааг баталгаажуулав.

**Зураг 23.** R1 дээрх "show ipv6 ospf neighbor"

```
R1#show ipv6 ospf neighbor

Neighbor ID   Pri  State      Dead Time  Interface ID  Interface
3.3.3.3         1  FULL/ -    00:00:33   4             FastEthernet0/1
2.2.2.2         1  FULL/ -    00:00:34   4             FastEthernet0/0
```

**Зураг 24.** R2 дээрх "show ipv6 ospf neighbor"

```
R2#show ipv6 ospf neighbor

Neighbor ID   Pri  State      Dead Time  Interface ID  Interface
3.3.3.3         1  FULL/ -    00:00:33   5             FastEthernet0/1
1.1.1.1         1  FULL/ -    00:00:38   4             FastEthernet0/0
```

**Зураг 25.** R3 дээрх "show ipv6 ospf neighbor"

```
R3#show ipv6 ospf neighbor

Neighbor ID   Pri  State      Dead Time  Interface ID  Interface
4.4.4.4         1  FULL/ -    00:00:34   4             FastEthernet1/0
2.2.2.2         1  FULL/ -    00:00:36   5             FastEthernet0/1
1.1.1.1         1  FULL/ -    00:00:30   5             FastEthernet0/0
```

**Зураг 26.** R4 дээрх "show ipv6 ospf neighbor"

```
R4#show ipv6 ospf neighbor

Neighbor ID   Pri  State      Dead Time  Interface ID  Interface
5.5.5.5         1  FULL/ -    -          13            OSPFv3_VL0
3.3.3.3         1  FULL/ -    00:00:39   6             FastEthernet0/0
5.5.5.5         1  FULL/ -    00:00:36   5             FastEthernet1/0
6.6.6.6         1  FULL/ -    00:00:36   4             FastEthernet0/1
```

**Зураг 27.** R5 дээрх "show ipv6 ospf neighbor"

```
R5#show ipv6 ospf neighbor

Neighbor ID   Pri  State      Dead Time  Interface ID  Interface
4.4.4.4         1  FULL/ -    -          13            OSPFv3_VL0
4.4.4.4         1  FULL/ -    00:00:35   6             FastEthernet0/1
6.6.6.6         1  FULL/ -    00:00:32   5             FastEthernet0/0
```

**Зураг 28.** R6 дээрх "show ipv6 ospf neighbor"

```
R6#show ipv6 ospf neighbor

Neighbor ID   Pri  State      Dead Time  Interface ID  Interface
4.4.4.4         1  FULL/ -    00:00:33   5             FastEthernet0/0
5.5.5.5         1  FULL/ -    00:00:30   4             FastEthernet0/1
```

**Тайлбар:** Бүх 14 adjacency `FULL` төлөвт байна. R4 болон R5 дээр `OSPFv3_VL0` гэсэн
virtual-link interface харагдаж байгаа нь virtual-link амжилттай ажиллаж буйг нэмж батална.


---

### 1.6 LSDB болон чиглүүлэлтийн баталгаажуулалт

**Зураг 29.** R3 дээрх "show ipv6 ospf database" (товчлол)

```
R3#show ipv6 ospf database

            OSPFv3 Router with ID (3.3.3.3) (Process ID 1)

        Router Link States (Area 0)

ADV Router      Age         Seq#        Fragment ID  Link count  Bits
3.3.3.3         1179        0x80000003  0            1           B
4.4.4.4         974         0x80000006  0            2           B
5.5.5.5         2     (DNA) 0x80000002  0            1           EB

        Inter Area Prefix Link States (Area 0)

ADV Router      Age         Seq#        Prefix
3.3.3.3         1450        0x80000001  2001:DB8:FF::1/128
3.3.3.3         1440        0x80000002  2001:DB8:FF::2/128
3.3.3.3         1426        0x80000001  2001:DB8:1::/48
4.4.4.4         1171        0x80000001  2001:DB8:2::/48
5.5.5.5         2     (DNA) 0x80000001  2001:DB8:3::/48

        Type-5 AS External Link States

ADV Router      Age         Seq#        Prefix
5.5.5.5         977         0x80000001  ::/0
```

**Тайлбар:**
- R3 = `B` (ABR), R4 = `B` (ABR), R5 = `EB` (ASBR + ABR virtual-link-ээр)
- `(DNA)` = DoNotAge — virtual-link-ээр ирсэн LSA
- Summary prefix-үүд: `2001:DB8:1::/48`, `2001:DB8:2::/48`, `2001:DB8:3::/48` — summarization ажиллаж байна
- Type-5 External: `::/0` — R5 (ASBR)-ийн default route

**Зураг 30.** R3 дээрх "show ipv6 route ospf"

```
R3#show ipv6 route ospf
IPv6 Routing Table - 22 entries

OE1   ::/0 [110/3], tag 1
        via FE80::C604:CFF:FECA:0, FastEthernet1/0
O     2001:DB8:1::/48 [110/0]
        via ::, Null0
O     2001:DB8:1:12::/64 [110/20]
        via FE80::C601:CFF:FE73:1, FastEthernet0/0
        via FE80::C602:CFF:FE90:1, FastEthernet0/1
OI    2001:DB8:2::/48 [110/11]
        via FE80::C604:CFF:FECA:0, FastEthernet1/0
OI    2001:DB8:3::/48 [110/12]
        via FE80::C604:CFF:FECA:0, FastEthernet1/0
OI    2001:DB8:FF::5/128 [110/2]
        via FE80::C604:CFF:FECA:0, FastEthernet1/0
OI    2001:DB8:FF::6/128 [110/12]
        via FE80::C604:CFF:FECA:0, FastEthernet1/0
```

**Тайлбар:**
- `OE1 ::/0` — External Type 1 default route (R5 ASBR-аас)
- `O 2001:DB8:1::/48 via Null0` — R3 өөрөө ABR тул summary-г Null0-руу чиглүүлнэ (loop prevention)
- `OI 2001:DB8:2::/48`, `OI 2001:DB8:3::/48` — Inter-area summary route-ууд зөв ирсэн

**Зураг 31.** R5 дээрх "show ipv6 ospf database external"

```
R5#show ipv6 ospf database external

            OSPFv3 Router with ID (5.5.5.5) (Process ID 1)

        Type-5 AS External Link States

LS age: 1103
LS Type: AS External Link
Link State ID: 0
Advertising Router: 5.5.5.5
LS Seq Number: 80000001
Checksum: 0x32CF
Length: 32
Prefix Address: ::
Prefix Length: 0, Options: None
Metric Type: 1 (Comparable directly to link state metric)
Metric: 1
External Route Tag: 1
```

**Тайлбар:** R5 нь ASBR болж `::/0` (default route)-ыг бүх OSPF domain-руу Type-1 External LSA
хэлбэрээр тарааж байна. Metric = 1, Type 1 нь internal cost + external metric = нийт cost гэсэн утгатай.

**Зураг 32.** R5 дээрх "show ipv6 ospf border-routers"

```
R5#show ipv6 ospf border-routers

OSPFv3 Process 1 internal Routing Table

Codes: i - Intra-area route, I - Inter-area route

i 4.4.4.4 [10] via FE80::C604:CFF:FECA:10, FastEthernet0/1, ABR, Area 0, SPF 3
i 4.4.4.4 [10] via FE80::C604:CFF:FECA:10, FastEthernet0/1, ABR, Area 2, SPF 8
i 3.3.3.3 [20] via FE80::C604:CFF:FECA:10, FastEthernet0/1, ABR, Area 0, SPF 3
```

**Тайлбар:** R5 нь R4 (ABR)-ыг Area 0 болон Area 2-д, R3 (ABR)-ыг Area 0-д харж байна.
Virtual-link-ээр дамжуулан Area 0-ын routing мэдээлэлд хүрч байгааг батална.


---

### 1.7 Тохиргоог нөөцлөн хадгалах (Backup)

Сүлжээний төхөөрөмжүүдийн тохиргоог лабораторийн TFTP серверт хадгалах даалгаврыг
гүйцэтгэхээр GNS3-ийн Cloud ашиглан сургуулийн дотоод сүлжээтэй холбосон. Гэвч сургуулийн
дотоод сүлжээний Wi-Fi хаяг болон TFTP серверийн хаяг өөр subnet-д байрлаж, R5 дээрх Fa1/0
interface-ээр TFTP серверт хүрэх боломжгүй байсан тул `copy running-config tftp:` команд
timeout болсон. Тиймээс өөр аргаар шийдвэрлэн, R1-ээс R6 хүртэлх бүх рутерийн тохиргоог
`show running-config` командаар харж хуулж аван, өөрийн зөөврийн компьютер дээр текст файл
(.txt) хэлбэрээр нөөцлөн хадгалав.

---

## 2. Шалгах асуултууд

### 1. OSPF authentication ашиглах үед үүсэх давуу тал юу вэ?

Хуурамч рутер холбогдохоос сэргийлнэ: Сүлжээнд дурын хүн шинэ рутер залгаж OSPF хөрш
болохыг оролдоход нууц үг (IPsec SHA-1 key) шаардах тул сүлжээний бүтцийг хамгаална.

Route Injection халдлагаас хамгаална: Гэмт этгээд зориудаар хуурамч IP хаягийн замуудыг (fake
routes) сүлжээнд цацаж, урсгалыг өөр рүүгээ чиглүүлэх (Blackholing) эсвэл сүлжээг унагаах
эрсдэлээс бүрэн сэргийлнэ.

### 2. OSPF fast reroute (FRR) сүлжээний найдвартай, тасралтгүй байдлыг хэрхэн хангаж байна вэ? Алгоритмын түвшинд тайлбарлана уу.

OSPF FRR нь LFA (Loop-Free Alternate) алгоритмыг ашигладаг. Хэвийн ажиллагааны үед OSPF нь
үндсэн замаа (Primary path) тооцоолохын зэрэгцээ, сүлжээний гогцоо (Loop) үүсгэхгүй өөр
нөөц замыг (Backup path) урьдчилан тооцоолж чиглүүлэлтийн хүснэгтэд бэлдэж тавьдаг.
Хэрэв үндсэн кабель тасрах эсвэл порт унтрах үед рутер дахин шинээр SPF алгоритм
ажиллуулж хугацаа алдалгүйгээр, 50 миллисекундийн дотор урьдчилан бэлдсэн LFA нөөц зам
руугаа трафикийг шууд шилжүүлдэг тул хэрэглэгчид сүлжээ тасарсныг мэдрэхгүй өнгөрдөг.

### 3. OSPF-ийн сүлжээнд route summarization ашиглах шаардлага ямар нөхцөлүүдэд бий болох вэ? 4-с багагүй нөхцөл бичиж тайлбарлана уу.

**Чиглүүлэлтийн хүснэгт хэт томрох үед:** Маш олон салбартай сүлжээнд рутерүүдийн санах ой
(RAM) дүүрэхээс сэргийлж, олон жижиг хаягийг нэгтгэж нэг мөр болгоход ашиглана.

**Сүлжээний тогтворгүй байдлыг нуух (Flapping):** Нэг захын сүлжээ унтарч асах бүрд тэр
мэдээлэл нь бүх рутерээр тархаж CPU ачааллуулдаг. Summarization хийснээр тэрхүү жижиг
тасалдал нь нэгтгэсэн том хаяг дотроо нуугдаж, бусад Area-д нөлөөлөхгүй болно.

**SPF тооцоолох хугацааг багасгах:** Сүлжээнд өөрчлөлт гарах үед рутерийн CPU-гээр ажиллах
Dijkstra (SPF) алгоритмын математик тооцооллыг хөнгөвчилж, delay-г багасгах шаардлагатай үед.

**Зурвасын өргөн хэмнэх (Bandwidth):** OSPF-ийн LSA (Link State Advertisement) пакетууд нь
тасралтгүй солилцогддог бөгөөд summarization хийснээр дамжуулах LSA-ийн тоо эрс багасаж,
сүлжээний ачааллыг бууруулдаг.

### 4. Virtual link-ийг ямар нөхцөлд ашиглах зайлшгүй шаардлагатай вэ? 2-с багагүй нөхцөл бичиж тайлбарлана уу.

**Шинэ сүлжээ нэгтгэх үед:** Өөр байгууллагатай нийлэх эсвэл шинэ салбар нээгдэх үед тухайн
шинэ OSPF Area нь газар зүйн болон физик холболтын хувьд шууд Backbone (Area 0) руу
холбогдох боломжгүй, дундуур нь өөр Area байрлаж байх үед.

**Backbone Area тасрах үед (Partitioned Backbone):** Гэнэтийн ослоор Area 0-ийн голын рутер
эсвэл кабель гэмтэж, Area 0 нь хоёр хуваагдан тусгаарлагдсан үед дундуур нь байгаа өөр
Area-аар (жишээ нь Area 2) дамжуулан логикоор эргүүлж нэгтгэх үед зайлшгүй шаардлагатай
болдог.


---

## 3. Дүгнэлт

Энэхүү лабораторийн ажлаар Multi-Area OSPFv3 (IPv6) сүлжээний ажиллагааг гүнзгийрүүлэн
судалж, шаардлагад нийцэхүйц түвшинд дараах тохиргоонуудыг амжилттай гүйцэтгэв:

| # | Шаардлага | Хэрэгжүүлсэн байдал | Баталгаа |
|---|-----------|---------------------|----------|
| 1 | Зөвхөн IPv6 | `ipv6 unicast-routing`, OSPFv3 | Бүх interface IPv6-only |
| 2 | Route injection хамгаалалт | IPsec SHA-1 auth (4 area, тус бүр өөр SPI) | FULL adjacency = key таарсан |
| 3 | SPF хугацаа багасгах | Онолын түвшинд судалсан (GNS3 IOS хязгаарлалт) | — |
| 4 | Failover (FRR) | Онолын түвшинд судалсан (GNS3 IOS хязгаарлалт) | — |
| 5 | Summarization | `area X range 2001:DB8:N::/48` (R3, R4, R5) | `/48: 3` route summary |
| 6 | Multi-area extensible | Area 0/1/2/3 тусгаарлагдсан | LSDB-д бүгд харагдсан |
| 7 | Virtual-link | R4↔R5 (Area 2 transit) | `FULL`, `DoNotAge LSA` |
| 8 | ASBR default route | R5: `default-information originate` | `OE1 ::/0` бүх router-т |
| 9 | Backup | show running-config → .txt файл | Хадгалагдсан |

Мөн OSPFv3-ийн Virtual-Link-ийг практикт хэрэгжүүлж, LSDB-ийн DoNotAge (DNA) механизмыг
бодитоор ажигласнаар онолын мэдлэгээ бататгав. Энэхүү ажлыг амжилттай дуусгав.

---

*Тайлан бэлтгэсэн: Д.Билгүүнтөгс, 24B1NUM0440*  
*Огноо: 2025 оны 5-р сар*
