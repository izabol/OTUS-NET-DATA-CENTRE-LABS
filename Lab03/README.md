#  Лабораторная по ISIS Underlay
Цель: настроить маршрутизацию ISIS используя unicast ipv4 адреса и адреса ipv6 LLA (Link local address)

#### Важное замечание!!! Настройки будем производить одновременно на на двух группах устройств первая, куда вхдодят все Leaf, и вторая, куда входят все Spine

За основу конфигурации возьмём лабораторную №1 где уже настроены p2p линки.
## ISIS IPv4
### 1. Поднимем процесс ISIS и зададим параметры: тип адреса, зоны, id-устройства
Тип адреса традиционно зададим 49, что означает адреса для внутренних сетей
Зону выберем случайным образом, например - 2222
id устройства будем задавать:
в первой цифре номер POD - 0
во второй будем задавать номера Spine
в третьей - номера Leaf
Тогда для spine-1 получим
```
spine-1(config-router-isis)#sho ac
router isis Otus
   net 49.2222.0000.0001.0000.00
   !
   address-family ipv4 unicast
```
"1address-family ipv4 unicast" -  означает, что протокол будет работать с адресами ipv4.

Для того чтобы протокол заработал, его нужно включить на интерфейсах, также по аналогии с OSPF, указываем, что интерфейсы p2p. 
```
spine-2(config)#int eth1-3
spine-2(config-if-Et1-3)#isis enable Otus
spine-2(config-if-Et1-3)#isis network point-to-point
```
После чего получаем следующую картину соседства для spine-2:
```
spine-2(config-router-isis)#sho isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
Otus      default  leaf-1           L1   Ethernet1          50:1:0:ca:7a:8d   UP    8           leaf-1.0b
Otus      default  leaf-1           L2   Ethernet1          50:1:0:ca:7a:8d   UP    6           leaf-1.0b
Otus      default  leaf-2           L1   Ethernet2          50:1:0:be:ab:97   UP    7           leaf-2.0b
Otus      default  leaf-2           L2   Ethernet2          50:1:0:be:ab:97   UP    7           leaf-2.0b
Otus      default  leaf-3           L1   Ethernet3          50:1:0:27:3:91    UP    23          spine-2.0c
Otus      default  leaf-3           L2   Ethernet3          50:1:0:27:3:91    UP    26          spine-2.0c
```
### 2. Анонс loopback
Включим протокол ISIS на интерфейсах, к которым нужен доступ  из всей сети
```
leaf-3(config)#interface loopback 1
leaf-3(config-if-Lo1)#isis enable Otus
leaf-3(config-if-Lo1)#int vlan 10-11
leaf-3(config-if-Vl10-11)#isis enable Otus
```
Посмотрим таблицу маршрутизации на leaf-1
```
leaf-1#sho ip route

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

 C        172.16.1.1/32 is directly connected, Loopback1
 I L1     172.16.1.2/32 [115/30] via 172.21.1.2, Ethernet1
                                 via 172.22.1.2, Ethernet2
 I L1     172.16.1.3/32 [115/30] via 172.21.1.2, Ethernet1
                                 via 172.22.1.2, Ethernet2
 C        172.17.1.1/32 is directly connected, Loopback2
 C        172.20.8.0/24 is directly connected, Vlan8
 I L1     172.20.10.0/24 [115/30] via 172.21.1.2, Ethernet1
                                  via 172.22.1.2, Ethernet2
 I L1     172.20.11.0/24 [115/30] via 172.21.1.2, Ethernet1
                                  via 172.22.1.2, Ethernet2
 C        172.21.1.2/31 is directly connected, Ethernet1
 I L1     172.21.2.2/31 [115/20] via 172.21.1.2, Ethernet1
 I L1     172.21.3.2/31 [115/20] via 172.21.1.2, Ethernet1
 C        172.22.1.2/31 is directly connected, Ethernet2
 I L1     172.22.2.2/31 [115/20] via 172.22.1.2, Ethernet2
 I L1     172.22.3.2/31 [115/20] via 172.22.1.2, Ethernet2
```
### 3. Проверим прохождение трафика между клиентами
```
eve@srv-1:~$ ping 172.20.10.1 -c 3 -w 5
PING 172.20.10.1 (172.20.10.1) 56(84) bytes of data.
From 172.21.2.3 icmp_seq=1 Destination Host Unreachable

--- 172.20.10.1 ping statistics ---
4 packets transmitted, 0 received, +1 errors, 100% packet loss, time 3077ms
pipe 4
eve@srv-1:~$ ping 172.20.10.2 -c 3 -w 5
PING 172.20.10.2 (172.20.10.2) 56(84) bytes of data.
64 bytes from 172.20.10.2: icmp_seq=1 ttl=61 time=18.4 ms
64 bytes from 172.20.10.2: icmp_seq=2 ttl=61 time=13.8 ms
64 bytes from 172.20.10.2: icmp_seq=3 ttl=61 time=18.4 ms

--- 172.20.10.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 13.782/16.854/18.411/2.172 ms
eve@srv-1:~$ ping 172.20.11.1 -c 3 -w 5
PING 172.20.11.1 (172.20.11.1) 56(84) bytes of data.
64 bytes from 172.20.11.1: icmp_seq=1 ttl=61 time=21.4 ms
64 bytes from 172.20.11.1: icmp_seq=2 ttl=61 time=15.6 ms
64 bytes from 172.20.11.1: icmp_seq=3 ttl=61 time=20.8 ms

--- 172.20.11.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 15.648/19.292/21.419/2.588 ms
```
Как видим, трафик ходит. Ошибочно прописанный vlan10 на leaf-2 всё так же недоступен. Сменим вес маршрута на обоих spine , увеличив метрику. Было:
```
spine-1(config-if-Et2)#sho isis interface ethernet 2

IS-IS Instance: Otus VRF: default

  Interface Ethernet2:
    Index: 11 SNPA: P2P
    MTU: 1497 Type: point-to-point
    Supported address families: IPv4
    Area proxy boundary is disabled
    Speed: 1000 mbps
    BFD IPv4 is disabled
    BFD IPv6 is disabled
    Hello padding is enabled
    Level 1:
      Metric: 10, Number of adjacencies: 1
      Link-ID: 0B
      Authentication mode: None
      TI-LFA protection is disabled for IPv4
      TI-LFA protection is disabled for IPv6
    Level 2:
      Metric: 11, Number of adjacencies: 1
      Link-ID: 0B
      Authentication mode: None
      TI-LFA protection is disabled for IPv4
      TI-LFA protection is disabled for IPv6
```
Стало:
```
spine-1(config-if-Et2)#isis metric 11
spine-1(config-if-Et2)#sho isis interface ethernet 2

IS-IS Instance: Otus VRF: default

  Interface Ethernet2:
    Index: 11 SNPA: P2P
    MTU: 1497 Type: point-to-point
    Supported address families: IPv4
    Area proxy boundary is disabled
    Speed: 1000 mbps
    BFD IPv4 is disabled
    BFD IPv6 is disabled
    Hello padding is enabled
    Level 1:
      Metric: 11, Number of adjacencies: 1
      Link-ID: 0B
      Authentication mode: None
      TI-LFA protection is disabled for IPv4
      TI-LFA protection is disabled for IPv6
    Level 2:
      Metric: 11, Number of adjacencies: 1
      Link-ID: 0B
      Authentication mode: None
      TI-LFA protection is disabled for IPv4
      TI-LFA protection is disabled for IPv6
```
Проверяем на клиенте:
```
eve@srv-1:~$ ping 172.20.10.1 -c 3 -w 5
PING 172.20.10.1 (172.20.10.1) 56(84) bytes of data.
64 bytes from 172.20.10.1: icmp_seq=1 ttl=61 time=99.7 ms
64 bytes from 172.20.10.1: icmp_seq=2 ttl=61 time=17.1 ms
64 bytes from 172.20.10.1: icmp_seq=3 ttl=61 time=17.1 ms

--- 172.20.10.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 17.088/44.634/99.688/38.928 ms
eve@srv-1:~$ ping 172.20.10.2 -c 3 -w 5
PING 172.20.10.2 (172.20.10.2) 56(84) bytes of data.
From 172.21.3.3 icmp_seq=1 Destination Host Unreachable

--- 172.20.10.2 ping statistics ---
4 packets transmitted, 0 received, +1 errors, 100% packet loss, time 3059ms
pipe 4
```
Картина поменялась

## Unnumbered
Делаем по аналогии с OSPF. Вместо ipv4 адреса, делаем ссылку на интерфейс loopback1 
И получаем следующее отношение соседства для leaf-2:
```
leaf-2#conf t
leaf-2(config)#default int eth1-2
leaf-2(config)#int eth 1-2
leaf-2(config-if-Et1-2)#no switchport
leaf-2(config-if-Et1-2)#ip add unnumbered loopback 1
leaf-2(config-if-Et1-2)#isis network point-to-point
leaf-2(config-if-Et1-2)#isis enable Otus
leaf-2(config-if-Et1-2)#sho isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
Otus      default  spine-1          L1L2 Ethernet1          P2P               UP    25          0B
Otus      default  spine-2          L1L2 Ethernet2          P2P               UP    29          0B
leaf-2(config-if-Et1-2)#sho isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
Otus      default  spine-1          L1L2 Ethernet1          P2P               UP    28          0B
Otus      default  spine-2          L1L2 Ethernet2          P2P               UP    27          0B
leaf-2(config-if-Et1-2)#sho isis neighbors det

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
Otus      default  spine-1          L1L2 Ethernet1          P2P               UP    24          0B
  Area addresses: 49.2222
  SNPA: P2P
  Router ID: 0.0.0.0
  Advertised Hold Time: 30
  State Changed: 00:04:07 ago at 2024-11-27 11:56:22
  IPv4 Interface Address: 172.16.0.1
  IPv6 Interface Address: none
  Interface name: Ethernet1
  Graceful Restart: Supported
  Supported Address Families: IPv4
  Neighbor Supported Address Families: IPv4
Otus      default  spine-2          L1L2 Ethernet2          P2P               UP    23          0B
  Area addresses: 49.2222
  SNPA: P2P
  Router ID: 0.0.0.0
  Advertised Hold Time: 30
  State Changed: 00:04:12 ago at 2024-11-27 11:56:17
  IPv4 Interface Address: 172.16.0.2
  IPv6 Interface Address: none
  Interface name: Ethernet2
  Graceful Restart: Supported
  Supported Address Families: IPv4
  Neighbor Supported Address Families: IPv4
```
Проверяем прохождение трафика:
```
eve@srv-1:~ping 172.20.11.1 -c 3 -w 5
PING 172.20.11.1 (172.20.11.1) 56(84) bytes of data.
64 bytes from 172.20.11.1: icmp_seq=1 ttl=61 time=37.7 ms
64 bytes from 172.20.11.1: icmp_seq=2 ttl=61 time=19.2 ms
64 bytes from 172.20.11.1: icmp_seq=3 ttl=61 time=15.0 ms

--- 172.20.11.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 15.048/23.973/37.674/9.834 ms

```
## IPv6 
Ситуация у Arista полностью аналогична протоколу OSPF. Ничего интересного.
## Authentication
Доступ к протоколу можно ограничить паролем, согласно инструкции его можно установить глобально, а можно, непосредственно, на интерфейсе.

Установим пароль глобально на leaf-1
```
leaf-1(config-router-isis)#sho ac
router isis Otus
   net 49.2222.0000.0000.0001.00
   authentication mode md5
   authentication key 7 v7g20eURx0inAMyxm5CWHQ==
   !
   address-family ipv4 unicast
```
Проверим установление соседства на Spine-2 
```
spine-2#sho isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
Otus      default  leaf-1           L1L2 Ethernet1          P2P               UP    22          0B
Otus      default  leaf-2           L1L2 Ethernet2          P2P               UP    26          0B
Otus      default  leaf-3           L1L2 Ethernet3          P2P               UP    23          0C
```
Соседство установилось, что странно.
Пропишем то же на интерфейсе и проверим соседство:
```
spine-1#sho isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
Otus      default  leaf-1           L1L2 Ethernet1          P2P               INIT  25          00
Otus      default  leaf-2           L1L2 Ethernet2          P2P               INIT  27          00
Otus      default  leaf-3           L1L2 Ethernet3          P2P               INIT  27          00
```
Установим авторизацию на интерфейсах Spine и проверим соседство:
```
spine-1(config-if-Et1-3)#do sho isis nei

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
Otus      default  leaf-1           L1L2 Ethernet1          P2P               INIT  25          00
Otus      default  leaf-2           L1L2 Ethernet2          P2P               INIT  25          00
Otus      default  leaf-3           L1L2 Ethernet3          P2P               INIT  27          00
spine-1(config-if-Et1-3)#isis authentication key Secret
spine-1(config-if-Et1-3)#isis authentication mode md5
spine-1(config-if-Et1-3)#do sho isis nei

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
Otus      default  leaf-1           L1L2 Ethernet1          P2P               UP    28          0A
Otus      default  leaf-2           L1L2 Ethernet2          P2P               UP    25          0A
Otus      default  leaf-3           L1L2 Ethernet3          P2P               UP    30          0B
```
Соседство установилось. Вывод глобальная конфигурация не работает, пароль нужно ставить на интерфейсах