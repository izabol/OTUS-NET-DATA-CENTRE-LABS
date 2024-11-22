# Underlay OSPF
Цель: на основе предыдущей лабораторной поднять маршрутизацию OSPF на сегменте сети, так чтобы все клиенты сети имели полную связность. Дополнить работу убрав адреса ipv4 с интерфейсов и получить полную связность, используя технологию ip unnumbered
## Ход выполнения работы
1. Поднимаем и настраиваем  процесс OSPF
Для поднятия процесса необходимо назначить номер процесса. А для минимальной настройки переведём все интерфейсы в пассивный режим, активировав только p2p линки. 
Для Spine-1 настройка будет выглядеть так:
```
router ospf 200
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   max-lsa 12000
```
* `max-lsa 12000` - добавляется автоматически
2. На интерфейсах p2p активируем соответствующий режим
  ```
  spine-2(config)#int eth 1-3
spine-2(config-if-Et1-3)#ip ospf network point-to-point
```
3. Активируем OSPFна всех интерфейсах. которые будут участвовать в маршрутизации, добавив их в backbone area (у нас в принципе одна aria, и так как она должна обязательно присутствовать, то будем пользоваться ею) На Leaf-3 получим следующую конфигурацию интерфейсов:
```
leaf-3#sho run sec ospf
interface Ethernet1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
interface Ethernet2
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
interface Loopback1
   ip ospf area 0.0.0.0
interface Vlan10
   ip ospf area 0.0.0.0
interface Vlan11
   ip ospf area 0.0.0.0
router ospf 100
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   max-lsa 12000
```
4. Проверяем прохождения маршрутов и трафика
Проверяем состояние OSPF с соседями у Spine-1:
```
spine-1#sho ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
172.17.1.1      200      default  0   FULL                   00:00:32    172.21.1.3      Ethernet1
172.17.1.2      200      default  0   FULL                   00:00:29    172.21.2.3      Ethernet2
172.17.1.3      200      default  0   FULL                   00:00:36    172.21.3.3      Ethernet3
```
Выводим таблицу маршрутизации OSPF на leaf-2
```
leaf-3#sho ip route ospf

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

 O        172.16.0.1/32 [110/20] via 172.21.3.2, Ethernet1
 O        172.16.0.2/32 [110/20] via 172.22.3.2, Ethernet2
 O        172.16.1.1/32 [110/30] via 172.21.3.2, Ethernet1
                                 via 172.22.3.2, Ethernet2
 O        172.16.1.2/32 [110/30] via 172.21.3.2, Ethernet1
                                 via 172.22.3.2, Ethernet2
 O        172.20.8.0/24 [110/30] via 172.21.3.2, Ethernet1
                                 via 172.22.3.2, Ethernet2
 O        172.21.1.2/31 [110/20] via 172.21.3.2, Ethernet1
 O        172.21.2.2/31 [110/20] via 172.21.3.2, Ethernet1
 O        172.22.1.2/31 [110/20] via 172.22.3.2, Ethernet2
 O        172.22.2.2/31 [110/20] via 172.22.3.2, Ethernet2


```
Как видим доступность интерфейсов loopback 1  обеспечивается двумя путями через линки ведущие на спайны, что несомненно - правильно. 
Vlan10 у нас присутствует на двух коммутаторах, с аналогичными настройками, посмотрим как это "видит" spine-2 
 ```
 spine-2#sho ip route ospf

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

 O        172.16.0.1/32 [110/30] via 172.22.1.3, Ethernet1
                                 via 172.22.2.3, Ethernet2
                                 via 172.22.3.3, Ethernet3
 O        172.16.1.1/32 [110/20] via 172.22.1.3, Ethernet1
 O        172.16.1.2/32 [110/20] via 172.22.2.3, Ethernet2
 O        172.16.1.3/32 [110/20] via 172.22.3.3, Ethernet3
 O        172.20.8.0/24 [110/20] via 172.22.1.3, Ethernet1
 O        172.20.10.0/24 [110/20] via 172.22.2.3, Ethernet2
                                  via 172.22.3.3, Ethernet3
 O        172.20.11.0/24 [110/20] via 172.22.3.3, Ethernet3
 O        172.21.1.2/31 [110/20] via 172.22.1.3, Ethernet1
 O        172.21.2.2/31 [110/20] via 172.22.2.3, Ethernet2
 O        172.21.3.2/31 [110/20] via 172.22.3.3, Ethernet3
```
Видит верно, но ходят ли пакеты?
```
spine-2#ping 172.20.10.1
PING 172.20.10.1 (172.20.10.1) 72(100) bytes of data.
^C
--- 172.20.10.1 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 44ms

spine-2#ping 172.20.10.2
PING 172.20.10.2 (172.20.10.2) 72(100) bytes of data.
80 bytes from 172.20.10.2: icmp_seq=1 ttl=63 time=56.1 ms
80 bytes from 172.20.10.2: icmp_seq=2 ttl=63 time=49.5 ms
80 bytes from 172.20.10.2: icmp_seq=3 ttl=63 time=41.9 ms
80 bytes from 172.20.10.2: icmp_seq=4 ttl=63 time=34.7 ms
80 bytes from 172.20.10.2: icmp_seq=5 ttl=63 time=24.5 ms

--- 172.20.10.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 44ms
rtt min/avg/max/mdev = 24.591/41.396/56.164/11.068 ms, pipe 5, ipg/ewma 11.192/47.958 ms

```
Из двух маршрутов в сеть Spine выбрал маршрут через leaf-2 несмотря на то, что вес маршрута одинаковый

Можем прогнозировать, что srv-4 может связаться с srv-3, но не сможет с Srv-2
```
eve@srv4:~$ ping 172.20.10.1 -c 2
PING 172.20.10.1 (172.20.10.1) 56(84) bytes of data.
64 bytes from 172.20.10.1: icmp_seq=1 ttl=63 time=34.2 ms
64 bytes from 172.20.10.1: icmp_seq=2 ttl=63 time=6.98 ms

--- 172.20.10.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 6.984/20.571/34.159/13.587 ms
eve@srv4:~$ ping 172.20.10.2 -c 2
PING 172.20.10.2 (172.20.10.2) 56(84) bytes of data.
From 172.20.11.254 icmp_seq=1 Destination Host Unreachable
From 172.20.11.254 icmp_seq=2 Destination Host Unreachable

--- 172.20.10.2 ping statistics ---
2 packets transmitted, 0 received, +2 errors, 100% packet loss, time 1006ms
pipe 2
```
Как видим наши предположения подтвердились.

5. На интерфейсах p2p удалим ipv4 адреса, но согласно пункта 15.2.3.5 User-Manual, для работы протокола OSPF нужен ipv4 адрес, но его можно заимствовать, например с интерфейса loopback 1. Таким образом на Spine, для работы OSPF необходим адрес только на одном интерфейсе.
В итоге насройка на Spine-2 выглядит следующим образом:
```
spine-2#sho run sec ospf
interface Ethernet1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
interface Ethernet2
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
interface Ethernet3
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
interface Loopback1
   ip ospf area 0.0.0.0
router ospf 200
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   max-lsa 12000
```
Соседи spine-1:
```
spine-1#sho ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
172.17.1.3      200      default  0   FULL                   00:00:36    172.16.1.3      Ethernet3
172.17.1.2      200      default  0   FULL                   00:00:30    172.16.1.2      Ethernet2
172.17.1.1      200      default  0   FULL                   00:00:32    172.16.1.1      Ethernet1

```
Прохождение трафика между srv-1 и srv-4:
```
eve@srv-1:~$ ping 172.20.11.1
PING 172.20.11.1 (172.20.11.1) 56(84) bytes of data.
64 bytes from 172.20.11.1: icmp_seq=1 ttl=61 time=44.9 ms
64 bytes from 172.20.11.1: icmp_seq=2 ttl=61 time=15.3 ms
^C
--- 172.20.11.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 15.312/30.127/44.942/14.815 ms
```
## OSPFv3 через LLA (Link Local Addresses)
Удалим настройки OSPFv2 удалив процесс и сбросив настройки на интерфейсах
```
spine-1(config)#default interface eth 1-3
spine-1(config)#no router ospf 200
```
Удаляем принадлежность к backbone с интерфейсов loopback, а на Liaf ещё и на SVI.

А. Создадим и настроим процесс OSPFv3
```
leaf-3(config)#ipv6 router ospf 100
leaf-3(config-router-ospf3)#router-id 172.16.1.3
leaf-3(config-router-ospf3)#log-adjacency-changes
leaf-3(config-router-ospf3)#passive-interface default
leaf-3(config-router-ospf3)#no passive-interface ethernet 1-2
```
B. На p2p интерфейсах включим ipv6 и обозначим принадлежность к area

```
leaf-2(config-if-Et1-2)#no switchport
leaf-2(config-if-Et1-2)#ipv6 enable
leaf-2(config-if-Et1-2)#ipv6 ospf 100 area 0
leaf-2(config-if-Et1-2)#ipv6 ospf network point-to-point
leaf-2(config-if-Et1-2)#sho ac
interface Ethernet1
   no switchport
   ipv6 enable
   ipv6 ospf network point-to-point
   ipv6 ospf 100 area 0.0.0.0
interface Ethernet2
   no switchport
   ipv6 enable
   ipv6 ospf network point-to-point
   ipv6 ospf 100 area 0.0.0.0
```
С. Проверим отношение соседства
```
spine-2#sho ipv6 ospf neighbor
Routing Process "ospf 100":
Neighbor 172.16.1.3 VRF default priority is 0, state is Full
  In area 0.0.0.0 interface Ethernet3
  DR is None BDR is None
  Options is E R V6
  Dead timer is due in 36 seconds
  Graceful-restart-helper mode is Inactive
  Graceful-restart attempts: 0
Neighbor 172.16.1.2 VRF default priority is 0, state is Full
  In area 0.0.0.0 interface Ethernet2
  DR is None BDR is None
  Options is E R V6
  Dead timer is due in 29 seconds
  Graceful-restart-helper mode is Inactive
  Graceful-restart attempts: 0
Neighbor 172.16.1.1 VRF default priority is 0, state is Full
  In area 0.0.0.0 interface Ethernet1
  DR is None BDR is None
  Options is E R V6
  Dead timer is due in 36 seconds
  Graceful-restart-helper mode is Inactive
  Graceful-restart attempts: 0
```
Статус full - обозначает что соседство установлено.
Пропишем ipv6 адреса на интерфейсах loopback 1 коммутаторов leaf, и проверим таблицу маршрутизации.
```
leaf-3#sho ipv6 route

VRF: default
Displaying 3 of 6 IPv6 routing table entries
Codes: C - connected, S - static, K - kernel, O3 - OSPFv3,
       B - Other BGP Routes, A B - BGP Aggregate, R - RIP,
       I L1 - IS-IS level 1, I L2 - IS-IS level 2, DH - DHCP,
       NG - Nexthop Group Static Route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       RC - Route Cache Route

 O3       2006::1/128 [110/30]
           via fe80::5201:ff:fee5:e36a, Ethernet1
           via fe80::5201:ff:fe4b:6277, Ethernet2
 O3       2006::2/128 [110/30]
           via fe80::5201:ff:fee5:e36a, Ethernet1
           via fe80::5201:ff:fe4b:6277, Ethernet2
 C        2006::3/128 [0/0]
           via Loopback1, directly connected
```
Как видим маршруты ходят. 
Проверим прохождение трафика:
```
leaf-2#ping ipv6 2006::1
PING 2006::1(2006::1) 52 data bytes
60 bytes from 2006::1: icmp_seq=1 ttl=63 time=29.2 ms
60 bytes from 2006::1: icmp_seq=2 ttl=63 time=59.4 ms
60 bytes from 2006::1: icmp_seq=3 ttl=63 time=57.5 ms
60 bytes from 2006::1: icmp_seq=4 ttl=63 time=18.8 ms
60 bytes from 2006::1: icmp_seq=5 ttl=63 time=6.38 ms

--- 2006::1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 98ms
rtt min/avg/max/mdev = 6.382/34.286/59.433/21.062 ms, pipe 3, ipg/ewma 24.564/30.464 ms
leaf-2#ping ipv6 2006::2
PING 2006::2(2006::2) 52 data bytes
60 bytes from 2006::2: icmp_seq=1 ttl=64 time=0.822 ms
60 bytes from 2006::2: icmp_seq=2 ttl=64 time=0.485 ms
60 bytes from 2006::2: icmp_seq=3 ttl=64 time=0.290 ms
60 bytes from 2006::2: icmp_seq=4 ttl=64 time=0.292 ms
60 bytes from 2006::2: icmp_seq=5 ttl=64 time=0.087 ms

--- 2006::2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 6ms
rtt min/avg/max/mdev = 0.087/0.395/0.822/0.248 ms, ipg/ewma 1.616/0.593 ms
leaf-2#ping ipv6 2006::3
PING 2006::3(2006::3) 52 data bytes
60 bytes from 2006::3: icmp_seq=1 ttl=63 time=12.4 ms
60 bytes from 2006::3: icmp_seq=2 ttl=63 time=17.7 ms
60 bytes from 2006::3: icmp_seq=3 ttl=63 time=13.6 ms
60 bytes from 2006::3: icmp_seq=4 ttl=63 time=8.69 ms
60 bytes from 2006::3: icmp_seq=5 ttl=63 time=11.3 ms

--- 2006::3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 52ms
rtt min/avg/max/mdev = 8.694/12.792/17.780/2.988 ms, pipe 2, ipg/ewma 13.109/12.443 ms
```
Полный успех.

К сожалению, мне не удалось пробросить маршруты ipv4 через OSPFv3, но если мы будем использовать VPN тоннели, то это, возможно, не потребуется.
