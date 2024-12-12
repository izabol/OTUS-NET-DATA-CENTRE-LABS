# Лабораторная работа eBGP курса "Сетевой инженер ЦОД"
Цель: исследовать работу протокола BGP на коммутаторах под управлением EOS (Arista)

## eBGP ipv4
За основу возьмём первую лабораторную, где настроена ip связность

Настроим отношения соседства. На spine это выглядит следующим образом:
```
spine-2(config-router-bgp)#sho ac
router bgp 65000
   neighbor 172.22.1.3 remote-as 65001
   neighbor 172.22.2.3 remote-as 65002
   neighbor 172.22.3.3 remote-as 65003
```
проверим соседство:
```
spine-2(config-router-bgp)#sho ip bgp summ
BGP summary information for VRF default
Router identifier 172.17.0.2, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Neighbor   V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  172.22.1.3 4 65001             79        82    0    0 00:20:49 Estab   1      1
  172.22.2.3 4 65002             67        65    0    0 00:44:12 Estab   1      1
  172.22.3.3 4 65003             67        66    0    0 00:44:12 Estab   2      2
```
Для анонса интерфейсом создадим Route-map куда добавим интерфейсы, что нужно анонсировать. Для leaf получится следующая конфигурация:
```
...
route-map ADVERT_INT permit 10
   match interface Vlan11
!
route-map ADVERT_INT permit 20
   match interface Vlan10
!
router bgp 65003
   neighbor 172.21.3.2 remote-as 65000
   neighbor 172.22.3.2 remote-as 65000
   redistribute connected route-map ADVERT_INT
...
```
Смотрим таблицу маршрутизации:
```
leaf-3#sho ip bgp
BGP routing table information for VRF default
Router identifier 172.17.1.3, local AS number 65003
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      172.20.8.0/24          172.21.3.2            0       -          100     0       65000 65001 i
 *        172.20.8.0/24          172.22.3.2            0       -          100     0       65000 65001 i
 * >      172.20.10.0/24         -                     -       -          -       0       i
 *        172.20.10.0/24         172.21.3.2            0       -          100     0       65000 65002 i
 *        172.20.10.0/24         172.22.3.2            0       -          100     0       65000 65002 i
 * >      172.20.11.0/24         -                     -       -          -       0       i
```
Проверим прохождение трафика между srv-1 и srv-4
```
eve@srv-1:~$ ping 172.20.11.1
PING 172.20.11.1 (172.20.11.1) 56(84) bytes of data.
64 bytes from 172.20.11.1: icmp_seq=1 ttl=61 time=21.0 ms
64 bytes from 172.20.11.1: icmp_seq=2 ttl=61 time=13.5 ms
64 bytes from 172.20.11.1: icmp_seq=3 ttl=61 time=12.0 ms
64 bytes from 172.20.11.1: icmp_seq=4 ttl=61 time=14.8 ms
^C
--- 172.20.11.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 11.979/15.332/21.030/3.440 ms
```
## Unnumbered 
По аналогии с OSPF и ISIS прописать unnumbered адрес на интерфейсе ссылкой на интерфейс loopback, и поднять на нём BGP у меня не получилось. Поэтому, пойдём традиционным для BGP путём, используя самоназначенные ipv6 адреса для установления связности.
Активируем ipv6 на интерфейсах. 
Смотрим, что получилось:
```
spine-2#sho ipv6 int
Ethernet1 is up, line protocol is up (connected)
  IPv6 is enabled, link-local is fe80::5201:ff:fe4b:6277/64
  No global unicast address is assigned
  Joined group address(es):
    ff02::1
    ff02::1:ff4b:6277
    ff02::2
  ND DAD is enabled, number of DAD attempts: 1
  ND Reachable time is 2147483000 milliseconds
  ND retransmit interval is 1000 milliseconds
  ND enhanced duplicate address detection enabled
  ND advertised reachable time is 2147483000 milliseconds (using 2147483000)
  ND advertised retransmit interval is 1000 milliseconds
  ND router advertisements are sent every 200 seconds
  ND router advertisements live for 1800 seconds
  ND advertised default router preference is Medium
  ND advertised maximum hop count limit is 64
  Hosts use stateless autoconfig for addresses.
  ND unsolicited NAs are not accepted
...
```
Интерфейс самоназначил себе адрес, видно, что он входит в ipv6 мультикаст групы:
- все устройства, 
- Solicited-node multicast группа,  
- группа маршрутизаторов(он входит в группу, так как в конфигурации включена ipv6 unicast маршрутизация)

Проверим связность пингом на общую мультикаст группу интерфейса:
```
spine-1#ping ipv6 ff02::1 interface ethernet 1 interval 1
PING ff02::1(ff02::1) from fe80::5201:ff:fee5:e36a%et1 et1: 52 data bytes
60 bytes from fe80::5201:ff:fee5:e36a%et1: icmp_seq=1 ttl=64 time=0.114 ms
60 bytes from fe80::5201:ff:feca:7a8d%et1: icmp_seq=1 ttl=64 time=4.85 ms (DUP!)
60 bytes from fe80::5201:ff:fee5:e36a%et1: icmp_seq=2 ttl=64 time=0.064 ms
60 bytes from fe80::5201:ff:feca:7a8d%et1: icmp_seq=2 ttl=64 time=4.00 ms (DUP!)
60 bytes from fe80::5201:ff:fee5:e36a%et1: icmp_seq=3 ttl=64 time=0.057 ms
60 bytes from fe80::5201:ff:feca:7a8d%et1: icmp_seq=3 ttl=64 time=7.02 ms (DUP!)
60 bytes from fe80::5201:ff:fee5:e36a%et1: icmp_seq=4 ttl=64 time=0.062 ms
60 bytes from fe80::5201:ff:feca:7a8d%et1: icmp_seq=4 ttl=64 time=4.82 ms (DUP!)
60 bytes from fe80::5201:ff:fee5:e36a%et1: icmp_seq=5 ttl=64 time=0.052 ms

--- ff02::1 ping statistics ---
5 packets transmitted, 5 received, +4 duplicates, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 0.052/2.339/7.022/2.645 ms
```
##### Замечание! Адрес интерфейса также состоит в мультикаст группе и отвечает быстрее чем интерфейс соседа, поэтому необходимо установить интервал посыла пакета. иначе не дождавшись ответа соседа, программа ping завершит ожидание и пошлёт следующий пакет. В нашем случае, пакеты ответов помеченные как дубликаты, это пакеты соседа. Дубликатом помечается пакет с таким же "icmp_seq" вне зависимости от адреса ответа.

Маршрутизация ipv6 уже включена, поэтому приступим к настройке BGP
Чтобы проще манипулировать, объединим интерфейсы в одну peer группу под названием "underlay"(наличие peer group, вроде как, обязательное требование)
Для spine-2 это будет:
```
spine-2#sho run sec bgp
router bgp 65000
   neighbor underlay peer group
   neighbor interface Et1 peer-group underlay remote-as 65001
   neighbor interface Et2 peer-group underlay remote-as 65002
   neighbor interface Et3 peer-group underlay remote-as 65003
```
Для активации работы протокола на интерфейсах необходимо активировать их в секции "address-family ipv4" (что выглядит странно, так как адреса на интерфейсах из сети ipv6), а чтобы через сеть ipv6 пробрасывались маршруты сети ipv4 добавим строчку
```
leaf-3(config-router-bgp-af)#neighbor underlay next-hop address-family ipv6 originate
```
Для Leaf-2 будет следующая конфигурация:
```
interface Ethernet1
   no switchport
   ipv6 enable
!
interface Ethernet2
   no switchport
   ipv6 enable
...
...
ip routing
...
...
ipv6 unicast-routing
...
...
route-map ADVERT_INT permit 10
   match interface Vlan10
!
router bgp 65002
   neighbor underlay peer group
   redistribute connected route-map ADVERT_INT
   neighbor interface Et1-2 peer-group underlay remote-as 65000
   !
   address-family ipv4
      neighbor underlay activate
      neighbor underlay next-hop address-family ipv6 originate
!
end
```
В данный момент есть соседство и коммутатор получает маршруты:
```
leaf-1#sho ip bgp summary
BGP summary information for VRF default
Router identifier 172.17.1.1, local AS number 65001
Neighbor Status Codes: m - Under maintenance
  Neighbor                    V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fe80::5201:ff:fe4b:6277%Et2 4 65000             94        97    0    0 00:18:06 Estab   2      2
  fe80::5201:ff:fee5:e36a%Et1 4 65000             91        94    0    0 00:18:06 Estab   2      2
leaf-1#sh ip route

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 C        172.16.1.1/32 [0/0]
           via Loopback1, directly connected
 C        172.17.1.1/32 [0/0]
           via Loopback2, directly connected
 C        172.20.8.0/24 [0/0]
           via Vlan8, directly connected
 B E      172.20.10.0/24 [200/0]
           via fe80::5201:ff:fee5:e36a, Ethernet1
 B E      172.20.11.0/24 [200/0]
           via fe80::5201:ff:fee5:e36a, Ethernet1

```
Но трафик не ходит.
Посмотрим детали BGP:
```
leaf-1#sho ip bgp det
BGP routing table information for VRF default
Router identifier 172.17.1.1, local AS number 65001
BGP routing table entry for 172.20.8.0/24
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, IGP metric 1, weight 0, tag 0
      Received 00:27:47 ago, valid, local, best, redistributed (Connected)
      Rx SAFI: Unicast
BGP routing table entry for 172.20.10.0/24
 Paths: 2 available
  65000 65002
    fe80::5201:ff:fee5:e36a%Et1 from fe80::5201:ff:fee5:e36a%Et1 (172.17.0.1)
      Origin IGP, metric 0, localpref 100, IGP metric 1, weight 0, tag 0
      Received 00:19:57 ago, valid, external, best
      Rx SAFI: Unicast
  65000 65002
    fe80::5201:ff:fe4b:6277%Et2 from fe80::5201:ff:fe4b:6277%Et2 (172.17.0.2)
      Origin IGP, metric 0, localpref 100, IGP metric 1, weight 0, tag 0
      Received 00:19:57 ago, valid, external
      Rx SAFI: Unicast
BGP routing table entry for 172.20.11.0/24
 Paths: 2 available
  65000 65003
    fe80::5201:ff:fee5:e36a%Et1 from fe80::5201:ff:fee5:e36a%Et1 (172.17.0.1)
      Origin IGP, metric 0, localpref 100, IGP metric 1, weight 0, tag 0
      Received 00:19:57 ago, valid, external, best
      Rx SAFI: Unicast
  65000 65003
    fe80::5201:ff:fe4b:6277%Et2 from fe80::5201:ff:fe4b:6277%Et2 (172.17.0.2)
      Origin IGP, metric 0, localpref 100, IGP metric 1, weight 0, tag 0
      Received 00:19:57 ago, valid, external
      Rx SAFI: Unicast
```
Вроде всё есть, даже сам нашёл для себя router-id. 
причина оказалась в том, что маршрутизация ipv6 включается, в данном случае двумя командами. Нужно ещё включить маршрутизацию на интерфейсах:
```
leaf-1(config)#ip routing ipv6 interfaces
```
Проверяем прохождение трафика  
```
eve@srv-1:~$ ping 172.20.11.1
PING 172.20.11.1 (172.20.11.1) 56(84) bytes of data.
64 bytes from 172.20.11.1: icmp_seq=1 ttl=61 time=26.7 ms
64 bytes from 172.20.11.1: icmp_seq=2 ttl=61 time=12.1 ms
64 bytes from 172.20.11.1: icmp_seq=3 ttl=61 time=12.0 ms
64 bytes from 172.20.11.1: icmp_seq=4 ttl=61 time=12.0 ms
64 bytes from 172.20.11.1: icmp_seq=5 ttl=61 time=12.1 ms
64 bytes from 172.20.11.1: icmp_seq=6 ttl=61 time=12.0 ms
64 bytes from 172.20.11.1: icmp_seq=7 ttl=61 time=12.1 ms
64 bytes from 172.20.11.1: icmp_seq=8 ttl=61 time=11.6 ms
64 bytes from 172.20.11.1: icmp_seq=9 ttl=61 time=12.1 ms
64 bytes from 172.20.11.1: icmp_seq=10 ttl=61 time=11.7 ms
64 bytes from 172.20.11.1: icmp_seq=11 ttl=61 time=12.0 ms
64 bytes from 172.20.11.1: icmp_seq=12 ttl=61 time=11.5 ms
^C
--- 172.20.11.1 ping statistics ---
12 packets transmitted, 12 received, 0% packet loss, time 11015ms
rtt min/avg/max/mdev = 11.495/13.153/26.710/4.092 ms
```
