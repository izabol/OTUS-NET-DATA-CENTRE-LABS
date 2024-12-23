# Лабораторная работа VxLAN L2 VNI курса "Сетевой инженер ЦОД"
Цель: развернуть VxLAN L2 VNI на сети топологии leaf-spine
Зв основу возьмём лабораторную по eBGP с самоназначенными адресами p2p и допишем в неё недостающие настройки или удалим лишнее. Одинаковые vlan (vlan10) у нас есть только на leaf-2 и leaf-3, с ними и будем работать.
## Первоначальная настройка
Перед началом работы удалим с SVI интерфейсов коммутаторов leaf ip адреса. Нам нужен чистый L2. Также изменим route-map для анонсов, оставив там только интерфейс loopback1.
Коммутаторы spine трогать не нужно.
## Создадим интерфейс VxLAN
Интерфейс VxLAN это, как бы, виртуальное устройство на базе которого будем производить настройки VNI. 
```
 interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 100010
```
Как можно видеть, здесь мы прописали интерфейс от которого будут распространятся VxLAN, порт, который прописывается сам, автоматически, и привязку VNI к vlan, где мы, собственно указываем метку VxLAN.
Здесь, также, можно прописать все адреса узлов VTEP в параметре ```vxlan flood vtep``` перечислив их последовательно. Таким образом, произведя статическую настройку. Например:
```
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 100010
   vxlan flood vtep 172.16.1.1 172.16.1.3
```
В этом можно убедится:
```
leaf-2#sho vxlan vni 100010
VNI to VLAN Mapping for Vxlan1
VNI          VLAN       Source       Interface       802.1Q Tag
------------ ---------- ------------ --------------- ----------
100010       10         static       Ethernet3       untagged
                                     Vxlan1          10

VNI to dynamic VLAN Mapping for Vxlan1
VNI       VLAN       VRF       Source
--------- ---------- --------- ------------
```
Но наша цель, настройка динамических EVPN и поэтому последнюю строку мы не пишем.

## Настройка маршрутизации
Всё самое интересное прописывается в настройках BGP
На всех коммутаторах, активируем расширенный обмен сообщениями, и на интерфейсах p2p активируем evpn. Затем, на коммутаторах leaf пропишем каждый vlan который будет работать в фабрике, там же укажем идентификатор источника и получателя получим для leaf-3:
```
router bgp 65003
   router-id 172.16.1.3
   neighbor underlay peer group
   neighbor underlay send-community extended
   redistribute connected route-map ADVERT_INT
   neighbor interface Et1-2 peer-group underlay remote-as 65000
   !
   vlan 10
      rd auto
      route-target both 65000:10
      redistribute learned
   !
   address-family evpn
      neighbor underlay activate
   !
   address-family ipv4
      neighbor underlay activate
      neighbor underlay next-hop address-family ipv6 originate
!
route-map ADVERT_INT permit 10
   match interface Loopback1
```
rd - здесь должна быть указана собственная ASN либо auto, 
redistribute learned - означает будет распространятся информация об изученных МАС на коммутаторе (route type-4)
route-target both - идентификатор источника и получателя, он будет одинаковым для каждого vlan на всей фабрике.

## Проверка работы
Посмотрим состояние интерфейса vxlan 1:
```
leaf-2#sho int vxl 1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback1 and is active with 172.16.1.2
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is
    [10, 100010]
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Headend replication flood vtep list is:
    10 172.16.1.3
  Shared Router MAC is 0000.0000.0000
```
Как видим, интерфейс нашёл пару и поднялся.
Проверим тип туннеля:
```
leaf-3#sho vxlan vtep detail
Remote VTEPS for Vxlan1:

VTEP             Learned Via         MAC Address Learning       Tunnel Type(s)
---------------- ------------------- -------------------------- --------------
172.16.1.2       control plane       control plane              flood
```
Управление динамическое/
Посмотрим наличие MAC:
```
leaf-3#sho bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 172.16.1.3, local AS number 65003
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 172.16.1.2:10 mac-ip 0050.0100.0801
                                 172.16.1.2            -       100     0       65000 65002 i
 *  ec    RD: 172.16.1.2:10 mac-ip 0050.0100.0801
                                 172.16.1.2            -       100     0       65000 65002 i
```
Запустим пинг междду серверами:
```
eve@srv-2:~$ ping 172.20.10.1
PING 172.20.10.1 (172.20.10.1) 56(84) bytes of data.
64 bytes from 172.20.10.1: icmp_seq=1 ttl=64 time=129 ms
64 bytes from 172.20.10.1: icmp_seq=2 ttl=64 time=21.0 ms
64 bytes from 172.20.10.1: icmp_seq=3 ttl=64 time=16.6 ms
64 bytes from 172.20.10.1: icmp_seq=4 ttl=64 time=14.8 ms
64 bytes from 172.20.10.1: icmp_seq=5 ttl=64 time=15.3 ms
64 bytes from 172.20.10.1: icmp_seq=6 ttl=64 time=16.0 ms
64 bytes from 172.20.10.1: icmp_seq=7 ttl=64 time=14.9 ms
64 bytes from 172.20.10.1: icmp_seq=8 ttl=64 time=15.7 ms
64 bytes from 172.20.10.1: icmp_seq=9 ttl=64 time=17.4 ms
64 bytes from 172.20.10.1: icmp_seq=10 ttl=64 time=16.1 ms
64 bytes from 172.20.10.1: icmp_seq=11 ttl=64 time=16.2 ms
64 bytes from 172.20.10.1: icmp_seq=12 ttl=64 time=15.7 ms
```
Пинг ходит, теперь посмотрим как дела в таблице MAC:
```
leaf-3#sho bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 172.16.1.3, local AS number 65003
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 172.16.1.2:10 mac-ip 0050.0100.0801
                                 172.16.1.2            -       100     0       65000 65002 i
 *  ec    RD: 172.16.1.2:10 mac-ip 0050.0100.0801
                                 172.16.1.2            -       100     0       65000 65002 i
 * >      RD: 172.16.1.3:10 mac-ip 0050.0100.0901
                                 -                     -       -       0       i
```
Добавился МАС. Посмотрим, чей:
```
3: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:50:01:00:08:01 brd ff:ff:ff:ff:ff:ff
    altname enp0s4
    inet 172.20.10.2/24 brd 172.20.10.255 scope global ens4
       valid_lft forever preferred_lft forever
    inet6 fe80::250:1ff:fe00:801/64 scope link
       valid_lft forever preferred_lft forever
eve@srv-2:~$ ip neighbor show
172.20.10.1 dev ens4 lladdr 00:50:01:00:09:01 STALE
172.31.255.254 dev ens3 lladdr 06:df:53:33:43:f1 REACHABLE
```
это МАС машины с адресом 172.20.10.1. 

# iBGP 
За основу возьмём лабу по iBGP unnumdered underlay.
Настроим iBGP на спайн, указав, что соседи будут с той же ASN, что он Route Reflector(RR), указав в сторону соседей, что там клиенты RR, а также, что он будет аннонсить маршруты от собственного интерфейса. Итого для spine-1 получим:
```
router bgp 65000
   router-id 172.16.0.1
   neighbor underlay peer group
   neighbor underlay next-hop-self
   neighbor underlay route-reflector-client
   neighbor underlay send-community extended
   neighbor interface Et1-3 peer-group underlay remote-as 65000
   !
   address-family ipv4
      neighbor underlay activate
      neighbor underlay next-hop address-family ipv6 originate
```
На leaf заведём отдельные peer-group для работы с L2EVPN, которые будут работать поверх underlay, передавая маршруты L2EVPN, для чего активируем её в секции address-family evpn. В качестве источника укажем интерфейс loopback 1. Также пропишем RT и RD, в секции каждого vlan, что участвует в EVPN. Для leaf-2 получим:
```
router bgp 65000
   router-id 172.16.1.2
   neighbor spine_evpn peer group
   neighbor spine_evpn remote-as 65000
   neighbor spine_evpn update-source Loopback1
   neighbor spine_evpn send-community extended
   neighbor underlay peer group
   neighbor 172.16.1.1 peer group spine_evpn
   neighbor 172.16.1.3 peer group spine_evpn
   redistribute connected route-map ADVERT_INT
   neighbor interface Et1-2 peer-group underlay remote-as 65000
   !
   vlan 10
      rd auto
      route-target both 65000:10
      redistribute learned
   !
   address-family evpn
      neighbor spine_evpn activate
   !
   address-family ipv4
      neighbor spine_evpn activate
      neighbor underlay activate
      neighbor underlay next-hop address-family ipv6 originate
```
Так же для теста, на leaf-1 vlan8 замапим в тот же VNI, что и остальные. 
Проверим установление соседства:
```
leaf-2#sho bgp summary
BGP summary information for VRF default
Router identifier 172.16.1.2, local AS number 65000
Neighbor                             AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
--------------------------- ----------- ------------- ----------------------- -------------- ---------- ----------
172.16.1.1                        65000 Established   IPv4 Unicast            Negotiated              1          1
172.16.1.1                        65000 Established   L2VPN EVPN              Negotiated              2          2
172.16.1.3                        65000 Established   IPv4 Unicast            Negotiated              1          1
172.16.1.3                        65000 Established   L2VPN EVPN              Negotiated              2          2
fe80::5201:ff:fe4b:6277%Et2       65000 Established   IPv4 Unicast            Negotiated              2          2
fe80::5201:ff:fee5:e36a%Et1       65000 Established   IPv4 Unicast            Negotiated              2          2

```
Маршруты BGP:
```
leaf-2#sho ip bgp
BGP routing table information for VRF default
Router identifier 172.16.1.2, local AS number 65000
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      172.16.1.1/32          fe80::5201:ff:fe4b:6277%Et2 0       -          100     0       i Or-ID: 172.16.1.1 C-LST: 172.16.0.2
 *        172.16.1.1/32          fe80::5201:ff:fee5:e36a%Et1 0       -          100     0       i Or-ID: 172.16.1.1 C-LST: 172.16.0.1
 *        172.16.1.1/32          172.16.1.1            0       -          100     0       i
 * >      172.16.1.2/32          -                     -       -          -       0       i
 * >      172.16.1.3/32          fe80::5201:ff:fe4b:6277%Et2 0       -          100     0       i Or-ID: 172.16.1.3 C-LST: 172.16.0.2
 *        172.16.1.3/32          fe80::5201:ff:fee5:e36a%Et1 0       -          100     0       i Or-ID: 172.16.1.3 C-LST: 172.16.0.1
 *        172.16.1.3/32          172.16.1.3            0       -          100     0       i
```
Маршруты evpn:
```
leaf-2#sho bgp evpn
BGP routing table information for VRF default
Router identifier 172.16.1.2, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 172.16.1.1:10 mac-ip 0050.0100.0701
                                 172.16.1.1            -       100     0       i
 * >      RD: 172.16.1.2:10 mac-ip 0050.0100.0801
                                 -                     -       -       0       i
 * >      RD: 172.16.1.3:10 mac-ip 0050.0100.0901
                                 172.16.1.3            -       100     0       i
 * >      RD: 172.16.1.1:10 imet 172.16.1.1
                                 172.16.1.1            -       100     0       i
 * >      RD: 172.16.1.2:10 imet 172.16.1.2
                                 -                     -       -       0       i
 * >      RD: 172.16.1.3:10 imet 172.16.1.3
                                 172.16.1.3            -       100     0       i
```
Доступность loopback:
```
leaf-2#ping 172.16.1.3
PING 172.16.1.3 (172.16.1.3) 72(100) bytes of data.
80 bytes from 172.16.1.3: icmp_seq=1 ttl=64 time=8.62 ms
80 bytes from 172.16.1.3: icmp_seq=2 ttl=64 time=6.45 ms
80 bytes from 172.16.1.3: icmp_seq=3 ttl=64 time=6.76 ms

--- 172.16.1.3 ping statistics ---
5 packets transmitted, 3 received, 40% packet loss, time 35ms
rtt min/avg/max/mdev = 6.459/7.283/8.623/0.955 ms, ipg/ewma 8.937/8.154 ms
```
Проверим прохождение трафика между серверами:
```
eve@srv-2:~$ ping 172.20.10.1 -c 3
PING 172.20.10.1 (172.20.10.1) 56(84) bytes of data.
64 bytes from 172.20.10.1: icmp_seq=1 ttl=64 time=61.2 ms
64 bytes from 172.20.10.1: icmp_seq=2 ttl=64 time=17.5 ms
64 bytes from 172.20.10.1: icmp_seq=3 ttl=64 time=16.0 ms

--- 172.20.10.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 16.038/31.598/61.233/20.963 ms
eve@srv-2:~$ ping 172.20.10.10 -c 3
PING 172.20.10.10 (172.20.10.10) 56(84) bytes of data.
64 bytes from 172.20.10.10: icmp_seq=1 ttl=64 time=21.2 ms
64 bytes from 172.20.10.10: icmp_seq=2 ttl=64 time=23.3 ms
64 bytes from 172.20.10.10: icmp_seq=3 ttl=64 time=19.0 ms

--- 172.20.10.10 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 19.030/21.163/23.303/1.744 ms
eve@srv-2:~$ ip a
. . .
3: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:50:01:00:08:01 brd ff:ff:ff:ff:ff:ff
    altname enp0s4
    inet 172.20.10.2/24 brd 172.20.10.255 scope global ens4
       valid_lft forever preferred_lft forever
    inet6 fe80::250:1ff:fe00:801/64 scope link
       valid_lft forever preferred_lft forever
```
Адрес 172.20.10.10 прописан дополнительно на первом сервере
```
eve@srv-1:~$ ip a sho ens4
3: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:50:01:00:07:01 brd ff:ff:ff:ff:ff:ff
    altname enp0s4
    inet 172.20.8.1/24 brd 172.20.8.255 scope global ens4
       valid_lft forever preferred_lft forever
    inet 172.20.10.10/24 scope global ens4
       valid_lft forever preferred_lft forever
    inet6 fe80::250:1ff:fe00:701/64 scope link
       valid_lft forever preferred_lft forever
```
Вывод: iBGP не оказывает нагрузку на spine, вся работа по распространению трафика ложится на leaf
 