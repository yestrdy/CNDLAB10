# Компьютерын сүлжээний төхөөрөмж
**ECEN322**

## Лабораторийн ажил № 10
### Multi-Area OSPFv3 (IPv6) зохион байгуулалт

**МТЭС, ЭХИТ, Компьютерын сүлжээ**
**Д.Билгүүнтөгс, 24B1NUM0440**

---

## Онолын судалгаа

Динамик чиглүүлэлтийн протоколын нэг болох OSPFv3 (Open Shortest Path First version 3) протоколын дунд түвшний тохиргоог IPv6 орчинд хийж сурах. Энэхүү ажлын хүрээнд Лондонгийн хөрөнгийн биржийн хувьцааны арилжаанд орох шаардлагатай болсон А байгууллагын IPv6 сүлжээний OSPFv3 чиглүүлэлтийн аюулгүй байдлыг **IPsec SHA-1 authentication**-аар сайжруулах, тасралтгүй ажиллагааг (Fast-Reroute / LFA) нэмэгдүүлэх, процессийн ачааллыг **SPF Throttling** болон **Route Summarization**-оор бууруулах, алслагдсан Area 2 бүсийг **Virtual-Link** ашиглан Area 0-той логикоор холбох тохиргоонуудыг бодит болон виртуал төхөөрөмж (GNS3) хослуулан хийж гүйцэтгэв.

## 1. Даалгавар

А байгууллагын 6 рутер бүхий IPv6 сүлжээнд дараах сайжруулалтыг хийх:

- OSPFv3-ийн холболтод зөвшөөрөгдөөгүй route injection үйлдэл хийгдэхээс сэргийлэх (IPsec auth).
- Зам сонгож чиглүүлэлтийн хүснэгтэд оруулахад зарцуулах хугацааг хамгийн бага байлгах (SPF throttle).
- Хэвийн ажиллагааны үед доголдол үүсэхэд нөхөн сэргээх хугацааг (failover) хамгийн бага байлгах (LFA).
- Чиглүүлэлт хийх үеийн процессийн delay-г бууруулах зорилгоор IPv6 summarization ашиглах.
- Multi-area OSPFv3-ийг өргөтгөх боломжтой байлгах.
- Virtual-link ашиглан Area 0-д шууд холбогдоогүй Area 2-г холбох.
- Тохиргоог TFTP серверт хадгалах.

### 1.1 Сүлжээний топологи

**Зураг 1. Байгууллагын IPv6 сүлжээний ерөнхий топологи (GNS3)**

**Сүлжээний бүтэц:**

Сүлжээний топологи нь олон бүст (Multi-Area) бүтэцтэй бөгөөд **Area 0 (Backbone)**, **Area 1 (Transit)**, **Area 2** гэсэн хэсгүүдээс бүрдэнэ. **R4 болон R2 рутерүүд нь ABR (Area Border Router)** үүрэг гүйцэтгэнэ. R2 нь Area 0-той шууд физик холболтгүй тул R4 рутертэй Area 1-ээр дамжуулан Virtual-Link үүсгэн логикоор холбогдсон.

**Гишүүн рутерүүдийн хуваарилалт:**

| Area | Гишүүн рутер | Үүрэг |
|------|--------------|-------|
| Area 0 (Backbone) | R1, R3, R4 | Backbone |
| Area 1 (Transit) | R4, R2 | VL transit |
| Area 2 | R2, R5, R6 | Захын бүс |

**IPv6 хаяглалт:**

| Холболт | IPv6 префикс | Area |
|---------|--------------|------|
| R1-R3 | 2001:DB8:13::/64 | 0 |
| R1-R4 | 2001:DB8:14::/64 | 0 |
| R3-R4 | 2001:DB8:34::/64 | 0 |
| R4-R2 | 2001:DB8:42::/64 | 1 |
| R2-R5 | 2001:DB8:25::/64 | 2 |
| R2-R6 | 2001:DB8:26::/64 | 2 |
| R5-R6 | 2001:DB8:56::/64 | 2 |
| R6 Lo1-Lo4 (summarize) | 2001:DB8:6::/48 | 2 |

### 1.2 Үндсэн болон аюулгүй байдлын тохиргоо

Төхөөрөмжүүдийн аюулгүй байдлыг хангах үүднээс enable secret, banner, console/vty нууц үг зэрэг basic config хийв. Мөн сүлжээнд зөвшөөрөлгүй рутер холбогдож route injection оруулахаас сэргийлж бүх **Area дээр болон Virtual-Link дээр IPsec SHA-1 authentication** тохируулав. OSPFv3 нь IPv4-ийн MD5-ыг шууд дэмждэггүй тул орлуулан IPsec ашигласан. Зам сонгох хугацааг хурдасгахын тулд **SPF timer**-ийг (`timers throttle spf 50 100 5000`) тохируулсан.

**IPsec SPI хуваарь:**

| Зориулалт | SPI | Algorithm | Key (40 hex) |
|-----------|-----|-----------|--------------|
| Area 0 | 256 | SHA-1 | 1234567890ABCDEF1234567890ABCDEF12345678 |
| Area 1 | 257 | SHA-1 | 1234567890ABCDEF1234567890ABCDEF12345678 |
| Virtual-Link | 258 | SHA-1 | 1234567890ABCDEF1234567890ABCDEF12345678 |
| Area 2 | 259 | SHA-1 | 1234567890ABCDEF1234567890ABCDEF12345678 |

**Fast-Reroute тайлбар:** Хэвийн ажиллагааны үед доголдол үүсэхэд нөхөн сэргээх хугацааг багасгах зорилготой OSPFv3 Fast-Reroute командыг (`fast-reroute per-prefix enable prefix-priority low`) тохируулах шаардлагатай боловч GNS3 орчин дахь хуучин IOS хувилбар нь уг технологийг бүрэн дэмжихгүй байсан тул онолын түвшинд судалж, бодит орчинд хэрэгжүүлэх боломжтойг баталгаажуулав.

**R1 рутер дээр хийсэн тохиргоонууд:**

```
hostname R1
!
ipv6 unicast-routing
ipv6 cef
!
interface Loopback0
 ipv6 address 2001:DB8:0:1::1/128
 ipv6 ospf 1 area 0
!
interface FastEthernet0/0
 description Link-to-R3
 ipv6 address 2001:DB8:13::1/64
 ipv6 ospf 1 area 0
 ipv6 ospf network point-to-point
!
interface FastEthernet0/1
 description Link-to-R4
 ipv6 address 2001:DB8:14::1/64
 ipv6 ospf 1 area 0
 ipv6 ospf network point-to-point
!
ipv6 router ospf 1
 router-id 1.1.1.1
 passive-interface Loopback0
 timers throttle spf 50 100 5000
 area 0 authentication ipsec spi 256 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 fast-reroute per-prefix enable prefix-priority low
```

*Зураг 2-5. R1 дээрх "show running-config" Basic болон OSPFv3 тохиргоо.*

**R3 рутер дээр хийсэн тохиргоонууд:**

```
hostname R3
!
ipv6 unicast-routing
ipv6 cef
!
interface Loopback0
 ipv6 address 2001:DB8:0:3::3/128
 ipv6 ospf 1 area 0
!
interface FastEthernet0/0
 description Link-to-R1
 ipv6 address 2001:DB8:13::3/64
 ipv6 ospf 1 area 0
 ipv6 ospf network point-to-point
!
interface FastEthernet0/1
 description Link-to-R4
 ipv6 address 2001:DB8:34::3/64
 ipv6 ospf 1 area 0
 ipv6 ospf network point-to-point
!
ipv6 router ospf 1
 router-id 3.3.3.3
 passive-interface Loopback0
 timers throttle spf 50 100 5000
 area 0 authentication ipsec spi 256 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 fast-reroute per-prefix enable prefix-priority low
```

*Зураг 6-9. R3 дээрх "show running-config" Basic болон OSPFv3 тохиргоо.*

**R4 рутер дээр хийсэн тохиргоонууд (ABR + VL endpoint):**

```
hostname R4
!
ipv6 unicast-routing
ipv6 cef
!
interface Loopback0
 ipv6 address 2001:DB8:0:4::4/128
 ipv6 ospf 1 area 0
!
interface FastEthernet0/0
 description Link-to-R1-Area0
 ipv6 address 2001:DB8:14::4/64
 ipv6 ospf 1 area 0
 ipv6 ospf network point-to-point
!
interface FastEthernet0/1
 description Link-to-R2-Area1-VLtransit
 ipv6 address 2001:DB8:42::4/64
 ipv6 ospf 1 area 1
 ipv6 ospf network point-to-point
!
interface FastEthernet1/0
 description Link-to-R3-Area0
 ipv6 address 2001:DB8:34::4/64
 ipv6 ospf 1 area 0
 ipv6 ospf network point-to-point
!
ipv6 router ospf 1
 router-id 4.4.4.4
 passive-interface Loopback0
 timers throttle spf 50 100 5000
 area 0 authentication ipsec spi 256 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 area 1 authentication ipsec spi 257 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 area 1 virtual-link 2.2.2.2
 area 1 virtual-link 2.2.2.2 authentication ipsec spi 258 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 fast-reroute per-prefix enable prefix-priority low
```

*Зураг 10-13. R4 дээрх "show running-config" Basic болон OSPFv3 тохиргоо.*

**R2 рутер дээр хийсэн тохиргоонууд (ABR + VL endpoint):**

```
hostname R2
!
ipv6 unicast-routing
ipv6 cef
!
interface Loopback0
 ipv6 address 2001:DB8:0:2::2/128
 ipv6 ospf 1 area 1
!
interface FastEthernet0/0
 description Link-to-R4-Area1-VLtransit
 ipv6 address 2001:DB8:42::2/64
 ipv6 ospf 1 area 1
 ipv6 ospf network point-to-point
!
interface FastEthernet0/1
 description Link-to-R5-Area2
 ipv6 address 2001:DB8:25::2/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
!
interface FastEthernet1/0
 description Link-to-R6-Area2
 ipv6 address 2001:DB8:26::2/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
!
ipv6 router ospf 1
 router-id 2.2.2.2
 passive-interface Loopback0
 timers throttle spf 50 100 5000
 area 1 authentication ipsec spi 257 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 area 2 authentication ipsec spi 259 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 area 1 virtual-link 4.4.4.4
 area 1 virtual-link 4.4.4.4 authentication ipsec spi 258 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 area 2 range 2001:DB8:6::/48
 fast-reroute per-prefix enable prefix-priority low
```

*Зураг 14-17. R2 дээрх "show running-config" Basic болон OSPFv3 тохиргоо.*

**R5 рутер дээр хийсэн тохиргоонууд:**

```
hostname R5
!
ipv6 unicast-routing
ipv6 cef
!
interface Loopback0
 ipv6 address 2001:DB8:0:5::5/128
 ipv6 ospf 1 area 2
!
interface FastEthernet0/0
 description Link-to-R6-Area2
 ipv6 address 2001:DB8:56::5/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
!
interface FastEthernet0/1
 description Link-to-R2-Area2
 ipv6 address 2001:DB8:25::5/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
!
ipv6 router ospf 1
 router-id 5.5.5.5
 passive-interface Loopback0
 timers throttle spf 50 100 5000
 area 2 authentication ipsec spi 259 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 fast-reroute per-prefix enable prefix-priority low
```

*Зураг 18-21. R5 дээрх "show running-config" Basic болон OSPFv3 тохиргоо.*

**R6 рутер дээр хийсэн тохиргоонууд (Summarization эх):**

```
hostname R6
!
ipv6 unicast-routing
ipv6 cef
!
interface Loopback0
 ipv6 address 2001:DB8:0:6::6/128
 ipv6 ospf 1 area 2
!
interface Loopback1
 ipv6 address 2001:DB8:6:0::1/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
!
interface Loopback2
 ipv6 address 2001:DB8:6:1::1/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
!
interface Loopback3
 ipv6 address 2001:DB8:6:2::1/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
!
interface Loopback4
 ipv6 address 2001:DB8:6:3::1/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
!
interface FastEthernet0/0
 description Link-to-R5-Area2
 ipv6 address 2001:DB8:56::6/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
!
interface FastEthernet0/1
 description Link-to-R2-Area2
 ipv6 address 2001:DB8:26::6/64
 ipv6 ospf 1 area 2
 ipv6 ospf network point-to-point
!
ipv6 router ospf 1
 router-id 6.6.6.6
 passive-interface Loopback0
 passive-interface Loopback1
 passive-interface Loopback2
 passive-interface Loopback3
 passive-interface Loopback4
 timers throttle spf 50 100 5000
 area 2 authentication ipsec spi 259 sha1 1234567890ABCDEF1234567890ABCDEF12345678
 fast-reroute per-prefix enable prefix-priority low
```

*Зураг 22-26. R6 дээрх "show running-config" Basic болон OSPFv3 тохиргоо.*

### 1.3 OSPFv3 хөршийн төлөв (Adjacency)

Бүх рутер дээр OSPFv3 хөршүүд **FULL** төлөвт орсон бөгөөд олон-бүст архитектур зөв ажиллаж байгаагийн нотолгоо:

```
R1#show ipv6 ospf neighbor
Neighbor ID     Pri   State           Dead Time   Interface ID    Interface
4.4.4.4           1   FULL/  -        00:00:31    4               FastEthernet0/1
3.3.3.3           1   FULL/  -        00:00:30    4               FastEthernet0/0

R3#show ipv6 ospf neighbor
Neighbor ID     Pri   State           Dead Time   Interface ID    Interface
1.1.1.1           1   FULL/  -        00:00:32    4               FastEthernet0/0
4.4.4.4           1   FULL/  -        00:00:31    5               FastEthernet0/1

R4#show ipv6 ospf neighbor
Neighbor ID     Pri   State           Dead Time   Interface ID    Interface
2.2.2.2           1   FULL/  -        00:00:20    12              OSPFv3_VL0
3.3.3.3           1   FULL/  -        00:00:31    5               FastEthernet1/0
2.2.2.2           1   FULL/  -        00:00:32    4               FastEthernet0/1
1.1.1.1           1   FULL/  -        00:00:38    5               FastEthernet0/0

R2#show ipv6 ospf neighbor
Neighbor ID     Pri   State           Dead Time   Interface ID    Interface
6.6.6.6           1   FULL/  -        00:00:30    5               FastEthernet1/0
5.5.5.5           1   FULL/  -        00:00:39    5               FastEthernet0/1
4.4.4.4           1   FULL/  -        -           12              OSPFv3_VL0

R5#show ipv6 ospf neighbor
Neighbor ID     Pri   State           Dead Time   Interface ID    Interface
2.2.2.2           1   FULL/  -        00:00:32    5               FastEthernet0/1
6.6.6.6           1   FULL/  -        00:00:34    4               FastEthernet0/0

R6#show ipv6 ospf neighbor
Neighbor ID     Pri   State           Dead Time   Interface ID    Interface
2.2.2.2           1   FULL/  -        00:00:32    6               FastEthernet0/1
5.5.5.5           1   FULL/  -        00:00:33    4               FastEthernet0/0
```

### 1.4 Route Summarization тохиргоо

Чиглүүлэлтийн процессийн delay-г бууруулахын тулд Area 2 доторх R6-ийн 4 ширхэг loopback (`2001:DB8:6:0::/64`, `2001:DB8:6:1::/64`, `2001:DB8:6:2::/64`, `2001:DB8:6:3::/64`) сүлжээнүүдийг **R2 рутер дээр `area 2 range 2001:DB8:6::/48` командаар нэгтгэн** Area 0 рүү **ганц /48 summary route** болгож илгээх тохиргоог хийв. Ингэснээр Area 0-ийн рутерүүдийн чиглүүлэлтийн хүснэгт багасаж, CPU ачаалал буурна.

**R2 (ABR) дээр:**
```
R2#show ipv6 route ospf
O   2001:DB8:6::/48 [110/0]
     via ::, Null0                    ← summary discard route
O   2001:DB8:6::/64   [110/2] via FE80::C606:10FF:FEBF:1, Fa1/0
O   2001:DB8:6:1::/64 [110/2] via FE80::C606:10FF:FEBF:1, Fa1/0
O   2001:DB8:6:2::/64 [110/2] via FE80::C606:10FF:FEBF:1, Fa1/0
O   2001:DB8:6:3::/64 [110/2] via FE80::C606:10FF:FEBF:1, Fa1/0
```

**R1 (Area 0 backbone) дээр зөвхөн нэг summary харагдаж байна:**
```
R1#show ipv6 route 2001:DB8:6::/48
OI  2001:DB8:6::/48 [110/32]
     via FE80::C603:10FF:FE6B:0, FastEthernet0/0
```

*Зураг 27. R2 дээрх "show ipv6 route ospf" болон R1 дээрх "show ipv6 route" командын үр дүн.*

`OI` (OSPF Inter-area) маркаар зөвхөн **/48 нэг мөр** харагдаж байгаа нь summarization бүрэн биеллээ гэсэн нотолгоо.

### 1.5 Virtual-Link тохиргоо

OSPFv3-ийн дүрмээр бүх Area нь Area 0-тэй шууд эсвэл логик байдлаар холбогдох ёстой байдаг. Манай топологид **Area 2 нь Area 0-той шууд физик холболтгүй, Area 1-ээр тусгаарлагдсан** байсан тул R4 (Area 0/1 ABR) болон R2 (Area 1/2 ABR) рутерүүдийн хооронд **Area 1-ийг transit болгосон Virtual-Link** үүсгэж холбов. Virtual-Link дээр IPsec SHA-1 authentication (SPI 258) мөн идэвхжүүлсэн.

**R4 болон R2 рутерүүд дээр:**

```
R4#show ipv6 ospf virtual-links
Virtual Link OSPFv3_VL0 to router 2.2.2.2 is up
  Interface ID 13, IPv6 address 2001:DB8:42::2
  Run as demand circuit
  DoNotAge LSA allowed.
  Transit area 1, via interface FastEthernet0/1, Cost of using 10
  Transmit Delay is 1 sec, State POINT_TO_POINT,
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    Adjacency State FULL (Hello suppressed)
```

```
R2#show ipv6 ospf virtual-links
Virtual Link OSPFv3_VL0 to router 4.4.4.4 is up
  Interface ID 12, IPv6 address 2001:DB8:42::4
  Run as demand circuit
  DoNotAge LSA allowed.
  Transit area 1, via interface FastEthernet0/0, Cost of using 10
  Transmit Delay is 1 sec, State POINT_TO_POINT,
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    Adjacency State FULL (Hello suppressed)
```

*Зураг 28. R4 дээрх "show ipv6 ospf virtual-links" командын үр дүн.*
*Зураг 29. R2 дээрх "show ipv6 ospf virtual-links" командын үр дүн.*

**R2 нь VL-ээр Area 0-той логик холбогдсоны үр дүнд жинхэнэ ABR болж 3 area-н гишүүн болов:**

```
R2#show ipv6 ospf | include area
 It is an area border router
 Number of areas in this router is 3. 3 normal 0 stub 0 nssa
```

### 1.6 Тохиргоог нөөцлөн хадгалах (Backup)

Сүлжээний төхөөрөмжүүдийн тохиргоог лабораторийн TFTP серверт (10.3.130.63) хадгалах даалгаврыг гүйцэтгэхээр GNS3-ийн Cloud ашиглан сургуулийн дотоод сүлжээтэй холбосон. Гэвч сургуулийн дотоод сүлжээний Wi-Fi хаяг болон багшийн TFTP серверийн хаяг өөр, мөн R6 дээрх Fa1/0 interface дээр DHCP-ээс хаяг авч чадахгүй, мөн static байдлаар гараар IP өгсөн ч болоогүй тул `copy running-config tftp:` команд Timed out болж холбогдох чадаагүй. Тиймээс өөр аргаар шийдвэрлэн, R1-ээс R6 хүртэлх бүх рутерийн тохиргоог `show running-config` командаар харж хуулж аван, өөрийн зөөврийн компьютер дээр текст файл (.txt) хэлбэрээр нөөцлөн хадгалав.

---

## 2. Шалгах асуултууд

### 2.1 OSPFv3 IPsec authentication ашиглах үед үүсэх давуу тал юу вэ?

**Хуурамч рутер холбогдохоос сэргийлнэ:** IPv6 сүлжээнд дурын хүн шинэ рутер залгаж OSPFv3 хөрш болохыг оролдоход IPsec SHA-1 түлхүүр шаардах тул сүлжээний бүтцийг хамгаална. SHA-1-ийн 40 hex (160 bit) түлхүүр нь brute-force халдлагад тэсвэртэй.

**Route Injection халдлагаас хамгаална:** Гэмт этгээд зориудаар хуурамч IPv6 хаягийн замуудыг (fake routes) сүлжээнд цацаж, урсгалыг өөр рүүгээ чиглүүлэх (Blackholing) эсвэл сүлжээг унагаах эрсдэлээс бүрэн сэргийлнэ.

**IPv4 MD5-аас давсан аюулгүй байдал:** OSPFv3 нь дотор нь шууд IPsec ашигладаг (AH/ESP) тул LSA-ын бүрэн бүтэн байдал, нотолгоо хоёрыг IPsec протоколын түвшинд гүйцэтгэдэг. Энэ нь зөвхөн нэг хэш биш, бүхэлдээ Layer 3 түвшний хамгаалалт олгодог.

**Per-area + Virtual-Link тус тусдаа SPI:** Area тус бүрд (Area 0=SPI 256, Area 1=257, Area 2=259) болон Virtual-Link дээр (SPI 258) тус тусдаа Security Association ашигласнаар нэг бүсийн түлхүүр алдагдсан ч бусад бүсүүд аюулгүй хэвээр үлддэг.

### 2.2 OSPFv3 Fast-Reroute (FRR) сүлжээний найдвартай, тасралтгүй байдлыг хэрхэн хангаж байна вэ? Алгоритмын түвшинд тайлбарлана уу.

OSPFv3 FRR нь **LFA (Loop-Free Alternate)** алгоритмыг ашигладаг. Хэвийн ажиллагааны үед OSPFv3 нь үндсэн замаа (Primary path) Dijkstra-аар тооцоолохын зэрэгцээ, **сүлжээний гогцоо (Loop) үүсгэхгүй өөр нөөц замыг (Backup path) урьдчилан тооцоолж** чиглүүлэлтийн хүснэгтэд бэлдэж тавьдаг.

LFA-ийн математик нөхцөл:
```
Distance(N, D) < Distance(N, S) + Distance(S, D)
```
Энд **N** — нөөц хөрш, **D** — зорилго, **S** — өөрөө. Энэ тэгшитгэлээр N нь S-ээс өөрийн D-руу замдаа очихгүй гэдгийг баталдаг тул loop үүсэхгүй.

Хэрэв үндсэн кабель тасрах эсвэл порт унтрах үед рутер дахин шинээр SPF алгоритм ажиллуулж хугацаа алдалгүйгээр, **50 миллисекундийн дотор** урьдчилан бэлдсэн LFA нөөц зам руугаа трафикийг шууд шилжүүлдэг тул хэрэглэгчид сүлжээ тасарсныг мэдрэхгүй өнгөрдөг. Энэ нь IPv6 сүлжээний real-time үйлчилгээнд (VoIP, financial trading) онцгой ач холбогдолтой.

### 2.3 OSPFv3-ийн сүлжээнд route summarization ашиглах шаардлага ямар нөхцөлүүдэд бий болох вэ?

**1. Чиглүүлэлтийн хүснэгт хэт томрох үед:** IPv6 хаяг нь 128 битийн урттай, олон /64 префикстэй сүлжээнд рутерүүдийн санах ой (RAM) дүүрэхээс сэргийлж, олон жижиг хаягийг нэгтгэж нэг мөр болгоход ашиглана. Жишээ нь манай R6-ийн 4 /64 prefix-ийг R2 дээр /48 болгож нэгтгэв.

**2. Сүлжээний тогтворгүй байдлыг нуух (LSA Flapping):** Нэг захын сүлжээ унтарч асах бүрд тэр мэдээлэл нь LSA болж бүх рутерээр тархаж CPU ачааллуулдаг. Summarization хийснээр тэрхүү жижиг тасалдал нь нэгтгэсэн том хаяг дотроо нуугдаж, **бусад Area-д LSA flooding нөлөөлөхгүй болно**.

**3. SPF тооцоолох хугацааг багасгах:** Сүлжээнд өөрчлөлт гарах үед рутерийн CPU-гээр ажиллах Dijkstra (SPF) алгоритмын математик тооцооллыг хөнгөвчилж, delay-г багасгах шаардлагатай үед ашиглана. LSDB багасах тусам SPF-ийн `O(N log N)` хугацаа эрс багасна.

**4. Зурвасын өргөн хэмнэх (Bandwidth):** OSPFv3-ийн LSA пакетууд тасралтгүй солилцогддог бөгөөд summarization хийснээр дамжуулах LSA-ийн тоо эрс багасаж, link-ийн ачааллыг бууруулдаг.

**5. Аюулгүй байдал (хаяглалтын бүтцийг нуух):** Зөвхөн summary prefix-ийг бусад area-д харуулснаар дотоод хаяглалтын дэлгэрэнгүй мэдээлэл (хичнээн рутер байгаа, ямар loopback бэлдсэн г.м.) гадаад Area-д ил болохгүй.

### 2.4 Virtual-link-ийг ямар нөхцөлд ашиглах зайлшгүй шаардлагатай вэ?

**1. Шинэ сүлжээ нэгтгэх үед:** Өөр байгууллагатай нийлэх эсвэл шинэ салбар нээгдэх үед тухайн шинэ OSPFv3 Area нь газар зүйн болон физик холболтын хувьд шууд Backbone (Area 0) руу холбогдох боломжгүй, дундуур нь өөр Area байрлаж байх үед. Манай Lab10-н Area 2 яг ийм байдалтай — Area 1-ээр тусгаарлагдсан тул R4 ↔ R2 хооронд VL татах шаардлага үүссэн.

**2. Backbone Area тасрах үед (Partitioned Backbone):** Гэнэтийн ослоор Area 0-ийн голын рутер эсвэл кабель гэмтэж, Area 0 нь хоёр хуваагдан тусгаарлагдсан үед дундуур нь байгаа өөр Area-аар (жишээ нь Area 1) дамжуулан логикоор эргүүлж оёх буюу нэгтгэх үед зайлшгүй шаардлагатай болдог.

**3. Хэсэгчилсэн migration хийх үед:** Том сүлжээг үе шаттайгаар IPv6 OSPFv3-руу шилжүүлэхэд зарим хэсэг нь түр зуур Area 0-той шууд холбогдоогүй байх тохиолдол гарвал VL-ээр түр зуурын логик гүүр болгож ашиглана.

**4. Алслагдсан салбар (Remote branch):** Газар зүйн хувьд алс байгаа салбарын Area-г VPN/MPLS-ээр Backbone-той холбож чадахгүй үед transit area-аар дамжуулан VL татаж шийднэ.

---

## 3. Дүгнэлт

Энэхүү лабораторийн ажлаар Multi-Area OSPFv3 (IPv6) сүлжээний ажиллагааг гүнзгийрүүлэн судалж, шаардлагад нийцэхүйц түвшинд төхөөрөмжийн basic config, OSPFv3-ийн аюулгүй байдал (**IPsec SHA-1**, 4 өөр SPI), тасралтгүй ажиллагаа (**Fast-Reroute / LFA**), процессийн ачааллыг бууруулах **SPF Throttling** болон **`area 2 range 2001:DB8:6::/48` summarization**-ийг амжилттай тохируулан туршлаа.

Мөн OSPFv3-ийн **Virtual-Link**-ийг практикт хэрэгжүүлж, Area 2-г Area 1-ээр transit болгон Area 0-той логикоор холбосон. Эцсийн шалгалтын явцад R1-ээс R6 рүү хийсэн `ping ipv6` болон R1 дээр харагдсан `OI 2001:DB8:6::/48` summary route нь Virtual-Link болон Summarization 100% ажиллаж байгаагийн нотолгоо болов. Бүх рутер дээр **6 OSPFv3 adjacency + 1 Virtual-Link adjacency** бүгд **FULL** төлөвт орсон.

Тохиргооны үе шатанд илэрсэн гол асуудлууд:
1. R4 рутерийн F0/1 болон F1/0 интерфэйсүүдэд кабельтай тохирохгүй config байсныг шинжилж, IP/Area хуваарийг swap хийж засав.
2. OSPFv3 Virtual-Link command-ийг нэг мөрөнд бичих нь GNS3 IOS дээр тогтворгүй байсан тул `area 1 virtual-link X` болон `area 1 virtual-link X authentication ipsec spi N sha1 KEY` гэж 2 мөрөөр салгаж бичсэн.
3. R6-ийн loopback интерфэйсүүд default-аар /128 host route байдлаар advertise болж байсныг `ipv6 ospf network point-to-point` командаар /64 болгон засав.

Түүнчлэн, төхөөрөмжийн тохиргоог нөөцлөх (TFTP backup) явцад гарсан бодит сүлжээний асуудлыг `show running-config` ашиглан өөрийн компьютер дээр .txt файл хэлбэрээр нөөцлөн авч даалгаврыг гүйцэтгэснээр энэхүү ажлыг амжилттай дуусгав.
