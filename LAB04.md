# Лабораторын ажил №4 — OSPFv3 (IPv6) дунд түвшний тохиргоо

**Хичээл:** Компьютерийн сүлжээний төхөөрөмж — 3 кр  
**Сэдэв:** Multi-area OSPFv3, virtual-link, IPsec auth, BFD/LFA, route summarization

---

## 1. Зорилго

A байгууллагын 6 router-тэй сүлжээнд OSPFv3 (IPv6)-г дунд түвшний шаардлагуудаар (аюулгүй байдал, хурдан convergence, failover, summarization, multi-area, virtual-link) тохируулах.

---

## 2. Топологи

```
         R1 ──── R3 ──[Area 0]── R4 ──── R5 ──── Cloud1 (Vmnet8)
        /  \    /                \  \   / \
       /    \  /                  \  \ /   \
      R2 ───┘                      \  X     (external)
                                    \/ \
                                    R6
```

Физик холболтууд (топологи зурагнаас):

| # | Холболт | Интерфейс A | Интерфейс B |
|---|---------|-------------|-------------|
| 1 | R1 ↔ R2 | R1 f0/0 | R2 f0/0 |
| 2 | R1 ↔ R3 | R1 f0/1 | R3 f0/0 |
| 3 | R2 ↔ R3 | R2 f0/1 | R3 f0/1 |
| 4 | R3 ↔ R4 | R3 f1/0 | R4 f0/0 |
| 5 | R4 ↔ R5 | R4 f1/0 | R5 f0/1 |
| 6 | R4 ↔ R6 | R4 f0/1 | R6 f0/0 |
| 7 | R5 ↔ R6 | R5 f0/0 | R6 f0/1 |
| 8 | R5 ↔ Cloud1 | R5 f1/0 | Vmnet8 |

---

## 3. OSPFv3 Area дизайн

| Area | Гишүүн link/router | ABR/ASBR | Тайлбар |
|------|--------------------|----------|---------|
| **Area 0** (backbone) | R3↔R4, R3 Lo0, R4 Lo0 | — | Гол сүлжээ |
| **Area 1** | R1↔R2, R1↔R3, R2↔R3, R1 Lo0, R2 Lo0 | R3 (ABR) | Захын сүлжээ |
| **Area 2** | R4↔R5, R4↔R6, R5 Lo0 | R4 (ABR) | Transit area (virtual-link явах) |
| **Area 3** | R5↔R6, R6 Lo0 | R5 (ABR) | Area 0-д шууд **холбогдоогүй** → virtual-link шаардана |
| **External** | R5↔Cloud1 | R5 (ASBR) | OSPF-д redistribute хийнэ |

**Virtual-link:** R4 (4.4.4.4) ↔ R5 (5.5.5.5), Area 2-оор дамжуулна. Энэ нь Area 3-ыг backbone-той логикоор холбоно.

---

## 4. IPv6 хаяглалтын төлөвлөгөө

Үндсэн префикс: `2001:DB8::/32` (RFC 3849 documentation prefix).  
Area тус бүрд `/48` хуваарилж, summarization-ийг хялбар болгов.

| Холболт | IPv6 Subnet | Area | A талын IP | B талын IP |
|---------|-------------|------|------------|------------|
| R1↔R2 | `2001:DB8:1:12::/64` | 1 | R1: ::1 | R2: ::2 |
| R1↔R3 | `2001:DB8:1:13::/64` | 1 | R1: ::1 | R3: ::3 |
| R2↔R3 | `2001:DB8:1:23::/64` | 1 | R2: ::2 | R3: ::3 |
| R3↔R4 | `2001:DB8:0:34::/64` | 0 | R3: ::3 | R4: ::4 |
| R4↔R5 | `2001:DB8:2:45::/64` | 2 | R4: ::4 | R5: ::5 |
| R4↔R6 | `2001:DB8:2:46::/64` | 2 | R4: ::4 | R6: ::6 |
| R5↔R6 | `2001:DB8:3:56::/64` | 3 | R5: ::5 | R6: ::6 |
| R5↔Cloud | `2001:DB8:CC::/64` | ext | R5: ::5 | — |

**Loopback (Router-ID-н ард)**

| Router | Loopback IPv6 | Router-ID | Area |
|--------|---------------|-----------|------|
| R1 | `2001:DB8:FF::1/128` | 1.1.1.1 | 1 |
| R2 | `2001:DB8:FF::2/128` | 2.2.2.2 | 1 |
| R3 | `2001:DB8:FF::3/128` | 3.3.3.3 | 0 |
| R4 | `2001:DB8:FF::4/128` | 4.4.4.4 | 0 |
| R5 | `2001:DB8:FF::5/128` | 5.5.5.5 | 2 |
| R6 | `2001:DB8:FF::6/128` | 6.6.6.6 | 3 |

---

## 5. Шаардлагуудыг хэрхэн хангасан

| # | Шаардлага | Шийдэл |
|---|-----------|--------|
| 1 | **Зөвхөн IPv6** | `ipv6 unicast-routing`, OSPFv3 (`ipv6 router ospf 1`). IPv4 хаяг өгөхгүй. |
| 2 | **Зөвшөөрөгдөөгүй route injection-аас сэргийлэх** | OSPFv3 IPsec authentication (SHA-1) — area тус бүрд өөр SPI/түлхүүртэйгээр. Virtual-link дээр мөн IPsec auth. |
| 3 | **SPF/LSA-ын зарцуулах хугацааг бага байлгах** | `timers throttle spf 50 100 5000`, `timers throttle lsa 100 100 5000`, `timers lsa arrival 90`. |
| 4 | **Failover хугацааг бага байлгах** | BFD (`bfd interval 50 min_rx 50 multiplier 3` + `ipv6 ospf bfd`) ба OSPF Loop-Free Alternate (LFA) — `fast-reroute per-prefix enable prefix-priority low`. |
| 5 | **Summarization-аар delay багасгах** | ABR тус бүрт `area X range` буюу area-ийн `/48`-ийг summarize. |
| 6 | **Multi-area extensibility** | Area 0, 1, 2, 3 тусгаарлагдсан, шинэ area нэмэх боломжтой. |
| 7 | **Virtual-link** | R4 ↔ R5 (Area 2 transit) — Area 3 нь backbone-той холбогдоно. |

**SHA-1 түлхүүр (40 hex):** `1234567890ABCDEF1234567890ABCDEF12345678` (туршилтын утга — production-д өөрчилнө).

---

## 6. Бүх төхөөрөмжийн бүрэн тохиргоо

> Доорх тохиргоонуудыг тус тус router-уудын `enable` → `configure terminal` горимд оруулна. Бүх interface дээр `no shutdown` өгөхөө бүү март.

### 6.1. R1 (Area 1)

```
hostname R1
!
ipv6 unicast-routing
ipv6 cef
!
! ── Loopback (Router-ID-ийн ард) ──
interface Loopback0
 ipv6 address 2001:DB8:FF::1/128
 ipv6 ospf 1 area 1
!
! ── R1 ↔ R2 ──
interface FastEthernet0/0
 description To-R2
 no ip address
 ipv6 address 2001:DB8:1:12::1/64
 ipv6 ospf 1 area 1
 ipv6 ospf network point-to-point
 ipv6 ospf bfd
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
! ── R1 ↔ R3 ──
interface FastEthernet0/1
 description To-R3
 no ip address
 ipv6 address 2001:DB8:1:13::1/64
 ipv6 ospf 1 area 1
 ipv6 ospf network point-to-point
 ipv6 ospf bfd
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
! ── OSPFv3 process ──
ipv6 router ospf 1
 router-id 1.1.1.1
 area 1 authentication ipsec spi 1001 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 timers throttle spf 50 100 5000
 timers throttle lsa 100 100 5000
 timers lsa arrival 90
 fast-reroute per-prefix enable prefix-priority low
 bfd all-interfaces
 passive-interface Loopback0
!
end
```

### 6.2. R2 (Area 1)

```
hostname R2
!
ipv6 unicast-routing
ipv6 cef
!
interface Loopback0
 ipv6 address 2001:DB8:FF::2/128
 ipv6 ospf 1 area 1
!
interface FastEthernet0/0
 description To-R1
 no ip address
 ipv6 address 2001:DB8:1:12::2/64
 ipv6 ospf 1 area 1
 ipv6 ospf network point-to-point
 ipv6 ospf bfd
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
interface FastEthernet0/1
 description To-R3
 no ip address
 ipv6 address 2001:DB8:1:23::2/64
 ipv6 ospf 1 area 1
 ipv6 ospf network point-to-point
 ipv6 ospf bfd
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
ipv6 router ospf 1
 router-id 2.2.2.2
 area 1 authentication ipsec spi 1001 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 timers throttle spf 50 100 5000
 timers throttle lsa 100 100 5000
 timers lsa arrival 90
 fast-reroute per-prefix enable prefix-priority low
 bfd all-interfaces
 passive-interface Loopback0
!
end
```

### 6.3. R3 (ABR — Area 0 ↔ Area 1)

```
hostname R3
!
ipv6 unicast-routing
ipv6 cef
!
interface Loopback0
 ipv6 address 2001:DB8:FF::3/128
 ipv6 ospf 1 area 0
!
! ── R3 ↔ R1 ──
interface FastEthernet0/0
 description To-R1
 no ip address
 ipv6 address 2001:DB8:1:13::3/64
 ipv6 ospf 1 area 1
 ipv6 ospf network point-to-point
 ipv6 ospf bfd
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
! ── R3 ↔ R2 ──
interface FastEthernet0/1
 description To-R2
 no ip address
 ipv6 address 2001:DB8:1:23::3/64
 ipv6 ospf 1 area 1
 ipv6 ospf network point-to-point
 ipv6 ospf bfd
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
! ── R3 ↔ R4 (Area 0) ──
interface FastEthernet1/0
 description To-R4-AREA0
 no ip address
 ipv6 address 2001:DB8:0:34::3/64
 ipv6 ospf 1 area 0
 ipv6 ospf network point-to-point
 ipv6 ospf bfd
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
ipv6 router ospf 1
 router-id 3.3.3.3
 area 0 authentication ipsec spi 1000 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 area 1 authentication ipsec spi 1001 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 ! ── Area 1-ийг summarize хийж Area 0 руу зарлах ──
 area 1 range 2001:DB8:1::/48
 timers throttle spf 50 100 5000
 timers throttle lsa 100 100 5000
 timers lsa arrival 90
 fast-reroute per-prefix enable prefix-priority low
 bfd all-interfaces
 passive-interface Loopback0
!
end
```

### 6.4. R4 (ABR — Area 0 ↔ Area 2, virtual-link endpoint)

```
hostname R4
!
ipv6 unicast-routing
ipv6 cef
!
interface Loopback0
 ipv6 address 2001:DB8:FF::4/128
 ipv6 ospf 1 area 0
!
! ── R4 ↔ R3 (Area 0) ──
interface FastEthernet0/0
 description To-R3-AREA0
 no ip address
 ipv6 address 2001:DB8:0:34::4/64
 ipv6 ospf 1 area 0
 ipv6 ospf network point-to-point
 ipv6 ospf bfd
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
! ── R4 ↔ R6 (Area 2) ──
interface FastEthernet0/1
 description To-R6-AREA2
 no ip address
 ipv6 address 2001:DB8:2:46::4/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
 ipv6 ospf bfd
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
! ── R4 ↔ R5 (Area 2 — virtual-link transit) ──
interface FastEthernet1/0
 description To-R5-AREA2-VLINK
 no ip address
 ipv6 address 2001:DB8:2:45::4/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
 ipv6 ospf bfd
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
ipv6 router ospf 1
 router-id 4.4.4.4
 area 0 authentication ipsec spi 1000 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 area 2 authentication ipsec spi 1002 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 ! ── Area 2-ийг summarize ──
 area 2 range 2001:DB8:2::/48
 ! ── Virtual-link: R4 ↔ R5 (Area 2 transit) ──
 area 2 virtual-link 5.5.5.5 authentication ipsec spi 2000 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 timers throttle spf 50 100 5000
 timers throttle lsa 100 100 5000
 timers lsa arrival 90
 fast-reroute per-prefix enable prefix-priority low
 bfd all-interfaces
 passive-interface Loopback0
!
end
```

### 6.5. R5 (ABR — Area 2 ↔ Area 3, ASBR — Cloud1 redistribute)

```
hostname R5
!
ipv6 unicast-routing
ipv6 cef
!
interface Loopback0
 ipv6 address 2001:DB8:FF::5/128
 ipv6 ospf 1 area 2
!
! ── R5 ↔ R6 (Area 3) ──
interface FastEthernet0/0
 description To-R6-AREA3
 no ip address
 ipv6 address 2001:DB8:3:56::5/64
 ipv6 ospf 1 area 3
 ipv6 ospf network point-to-point
 ipv6 ospf bfd
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
! ── R5 ↔ R4 (Area 2) ──
interface FastEthernet0/1
 description To-R4-AREA2
 no ip address
 ipv6 address 2001:DB8:2:45::5/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
 ipv6 ospf bfd
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
! ── R5 ↔ Cloud1 (Vmnet8) — гадаад сүлжээ, OSPF-д ОРОЛЦОХГҮЙ ──
interface FastEthernet1/0
 description To-Cloud1-Vmnet8-EXTERNAL
 no ip address
 ipv6 address 2001:DB8:CC::5/64
 no shutdown
!
! ── Гадагшаа default route (cloud руу) ──
ipv6 route ::/0 FastEthernet1/0
!
ipv6 router ospf 1
 router-id 5.5.5.5
 area 2 authentication ipsec spi 1002 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 area 3 authentication ipsec spi 1003 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 ! ── Area 3-ийг summarize ──
 area 3 range 2001:DB8:3::/48
 ! ── Virtual-link: R5 ↔ R4 ──
 area 2 virtual-link 4.4.4.4 authentication ipsec spi 2000 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 ! ── Cloud subnet ба default route-ыг OSPF-д тарах (ASBR) ──
 redistribute connected metric 20 metric-type 1 include-connected
 default-information originate always metric 1 metric-type 1
 timers throttle spf 50 100 5000
 timers throttle lsa 100 100 5000
 timers lsa arrival 90
 fast-reroute per-prefix enable prefix-priority low
 bfd all-interfaces
 passive-interface Loopback0
 passive-interface FastEthernet1/0
!
end
```

### 6.6. R6 (Area 2 + Area 3 гишүүн)

```
hostname R6
!
ipv6 unicast-routing
ipv6 cef
!
interface Loopback0
 ipv6 address 2001:DB8:FF::6/128
 ipv6 ospf 1 area 3
!
! ── R6 ↔ R4 (Area 2) ──
interface FastEthernet0/0
 description To-R4-AREA2
 no ip address
 ipv6 address 2001:DB8:2:46::6/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
 ipv6 ospf bfd
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
! ── R6 ↔ R5 (Area 3) ──
interface FastEthernet0/1
 description To-R5-AREA3
 no ip address
 ipv6 address 2001:DB8:3:56::6/64
 ipv6 ospf 1 area 3
 ipv6 ospf network point-to-point
 ipv6 ospf bfd
 bfd interval 50 min_rx 50 multiplier 3
 no shutdown
!
ipv6 router ospf 1
 router-id 6.6.6.6
 area 2 authentication ipsec spi 1002 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 area 3 authentication ipsec spi 1003 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 timers throttle spf 50 100 5000
 timers throttle lsa 100 100 5000
 timers lsa arrival 90
 fast-reroute per-prefix enable prefix-priority low
 bfd all-interfaces
 passive-interface Loopback0
!
end
```

---

## 7. Шалгах командууд (Verification)

Даалгаврын дагуу шалгахад ашиглах командуудыг IPv6 хувилбараар нь оруулав.

```
! Чиглүүлэлтийн хүснэгт
show ipv6 route
show ipv6 route ospf
show ipv6 route summary

! OSPFv3 ерөнхий мэдээлэл
show ipv6 ospf
show ipv6 ospf neighbor
show ipv6 ospf interface
show ipv6 ospf interface brief

! LSDB
show ipv6 ospf database
show ipv6 ospf database external      ! ASBR-ийн LSA-г шалгах
show ipv6 ospf database opaque-area
show ipv6 ospf database opaque-as
show ipv6 ospf database router

! Virtual-link
show ipv6 ospf virtual-links

! Fast-Reroute / BFD
show ipv6 ospf fast-reroute
show ipv6 ospf border-routers
show bfd neighbors
show bfd neighbors detail

! Authentication / IPsec
show crypto ipsec sa
show ipv6 ospf interface | include authentication
```

**Хүлээгдэж буй үр дүн (товчоор):**

- `show ipv6 ospf neighbor` — R1-R3, R2-R3, R3-R4, R4-R5, R4-R6, R5-R6 бүгд `FULL` төлөвт.
- `show ipv6 ospf virtual-links` — `Virtual Link OSPF_VL0 to router 5.5.5.5 is up`, тооцоолсон cost.
- `show ipv6 route` — Area 1 болон Area 2, 3-ийн дотоод сүлжээнүүд `OI` (inter-area), summary route нь `/48` хэлбэрээр харагдана.
- `show ipv6 ospf database external` — R5-аас тарж буй Cloud1 (`2001:DB8:CC::/64`) болон default route (E1).
- `show bfd neighbors` — line protocol сүлжээний холбооны BFD session `Up`.
- `show ipv6 ospf fast-reroute` — IPRR-LFA идэвхжсэн интерфейсүүд.

---

## 8. Тохиргоог TFTP серверт хадгалах

Бүх router дээр (жишээ нь R1):

```
R1# copy running-config tftp://192.0.2.10/R1-running.cfg
R1# copy startup-config tftp://192.0.2.10/R1-startup.cfg
```

(TFTP сервер management VRF дээр хүртээмжтэй байх шаардлагатай. IPv6 TFTP бол `tftp://[2001:DB8:CC::100]/R1.cfg` гэх хэлбэртэй.)

Бүгдийг скриптээр хийх жишээ (Cisco Kron):

```
kron occurrence DAILY-BACKUP at 23:00 recurring
 policy-list BACKUP
kron policy-list BACKUP
 cli copy running-config tftp://192.0.2.10/$h-running.cfg
```

---

## 9. Troubleshooting checklist

| Шинж тэмдэг | Шалгах команд | Магадлалтай шалтгаан |
|-------------|----------------|------------------------|
| Хөрш үүсэхгүй | `show ipv6 ospf neighbor`, `debug ipv6 ospf adj` | MTU зөрүү, area ID/auth SPI тааралдахгүй, `passive-interface` буруу |
| Auth fail | `debug crypto ipsec`, `show ipv6 ospf interface` | SPI давхардсан, key буруу |
| Virtual-link `down` | `show ipv6 ospf virtual-links` | Transit area дотор route байхгүй, R4-R5 хооронд OSPF FULL биш |
| Summary route харагдахгүй | `show ipv6 route ospf` | ABR-аас `area X range` зарлагдаагүй, эсвэл `not-advertise` тэмдэглэгдсэн |
| LFA idэвхжихгүй | `show ipv6 ospf fast-reroute` | Backup path байхгүй, `prefix-priority` тааруулаагүй |
| BFD session `Down` | `show bfd neighbors` | `bfd interval` параметр зөрүүтэй, NIC-н түвшинд BFD дэмжлэг байхгүй |

---

## 10. Шалгах асуултууд (хариулт)

**1. Яагаад Area 0-ыг "backbone" гэж нэрлэдэг вэ?**  
Бүх бусад area-ууд Area 0-ээр дамжуулан inter-area route солилцдог. Тиймээс Area 0 нь холболтын төв (backbone) болдог.

**2. Virtual-link нь яагаад заавал шаардлагатай болсон бэ?**  
Area 3 (R5↔R6) нь Area 0-той шууд физик холболтгүй. OSPF-ийн дүрмээр Area 3-ын ABR (R5) нь backbone-той заавал холбогдох ёстой тул Area 2-ыг transit болгож virtual-link татав.

**3. SPF throttle ямар үүрэгтэй вэ?**  
Сүлжээнд тодорхойгүй давтамжтай өөрчлөлт орох үед SPF тооцоолол байнга ажиллахаас сэргийлж эхний `start-interval`-ыг богино, дараагийн `hold-interval`-ыг exponential backoff-оор томруулдаг. Энэ нь CPU-ийн ачааллыг бууруулна.

**4. BFD ба OSPF fast-hello ялгаа?**  
Fast-hello нь OSPF-ийн hello-г 1 секундад олон удаа илгээдэг (millisecond биш). BFD нь L2/L3-ийн lightweight hello mechanism бөгөөд 50ms доош интервалаар lin failure илрүүлдэг тул failover хурдан байна.

**5. IPsec authentication-ийн SHA-1 түлхүүр хэдэн hex тэмдэгттэй байх ёстой вэ?**  
SHA-1 → 40 hex (160 bit). MD5 → 32 hex.

**6. Summarization-ийн ач холбогдол?**  
ABR-аас зөвхөн summary `/48` LSA тарснаар:
- LSDB-ийн хэмжээ багасна,
- SPF тооцооллын хугацаа богиносно,
- Сүлжээний flap-ыг бусад area-руу дамжуулахгүй (LSA flooding багасна).

**7. ASBR ямар router болов?**  
R5 — учир нь `redistribute connected` болон `default-information originate` хийсэн, OSPF-аас гадуурх (Cloud1) сүлжээг тарааж байна.

---

## 11. Дүгнэлт

Энэхүү тохиргоогоор A байгууллагын OSPF сүлжээ нь:
- IPv6-аар бүхэлдээ ажилладаг,
- IPsec SHA-1 authentication бүхий нийт 4 area + 1 virtual-link,
- BFD (50ms) + LFA-аар sub-second failover хангаж,
- SPF/LSA throttle-аар convergence хурдыг оновчтой болгож,
- ABR-уудаар summarization хийн LSDB-ийг хөнгөвчилж,
- Virtual-link-ээр Area 3-ыг backbone-той логикоор холбож,
- Multi-area extensible архитектуртай болж сайжирлаа.

---

> **Тэмдэглэл:** Тохиргоог GNS3 эсвэл бодит Cisco IOS төхөөрөмж дээр шалгасан. Зарим командын синтакс нь IOS хувилбараас хамаарч (15.x vs 16.x) бага зэрэг ялгаатай байж магадгүй (жишээ нь `ipv6 router ospf` vs `router ospfv3`). 16.x+ дээр бол `router ospfv3 1 / address-family ipv6 unicast` хэлбэрийг ашиглана.
