# Лабораторная работа  L3VPN Arista + Cumulus
За основу возьмём предыдущую лабу полностью оставим конфигурацию Spine и удалив всё что относится к L2VPN на Leaf.
## Настройка Distributed Gateway
Настройка Distributed Gateway подразумевает настройку одинакового шлюза на всех коммутаторах сети, с одинаковым MAC. То есть мы подразумеваем, что связность в сети будет осуществляться на коммутаторах Leaf.
Настройка единого шлюза на коммутаторах обоих производителей производится по протоколу VRRP, но логика настройки разная. Все интерфейсы будут добавлены в рабочую VRF, создание которой будет описано позднее. Здесь я привожу настройки конечной конфигурации

### Arista

На интерфейсе Vlan прописывается собственный IP, а затем создаётся VRRP группа, в которой прописывается общий адрес, который будет осуществлять функции Distributed Gateway

Настройка  интерфейса vlan на leaf11

```
interface Vlan10
   vrf corporate
   ip address 172.20.10.252/24
   vrrp 1 peer authentication text P@ssw0rd
   vrrp 1 ipv4 172.20.10.254
```
Настройка  интерфейса vlan на leaf12

```
interface Vlan11
   vrf corporate
   ip address 172.20.11.253/24
   vrrp 1 peer authentication text P@ssw0rd
   vrrp 1 ipv4 172.20.11.254
```

МАС будет единый, из настройки  ``` ip virtual-router mac-address 44:38:39:00:01:00 ```

### Cumulus

Настройка единого шлюза заключается в прописывании одной строкой МАС и виртуального адреса на интерфейсе Vlan. MAC можно устанавливать свой в пределах  одного Vlan, но так как у нас есть интеграция c Arista, то будем настраивать единый МАС на всех интерфейсах. У Vlan интерфейса должен быть, также, свой ip, а виртуальный идёт как дополнение.
Так как настройка Cumulus производится конфигурацией фала, то здесь и ниже, для описания конфигурации, буду просто приводить куски файла. Полный файл можно найти в файлах конфигурации приложенный к данной лабе.

<details>
<summary> Настройка  интерфейса vlan на leaf21 </summary>

```
auto vlan11
iface vlan11
    address 172.20.11.252/24
    address-virtual 44:38:39:00:01:00 172.20.11.254/24
    vlan-id 11
    vlan-raw-device bridge
    vrf corporate
```
</details>

<details>
<summary>Настройка  интерфейса vlan на leaf2</summary>

```
auto vlan14
iface vlan14
    address 172.20.14.253/24
    address-virtual 44:38:39:00:01:00 172.20.14.254/24
    vlan-id 14
    vlan-raw-device bridge
    vrf corporate
```
</details>

## Настройка VRF

Название VRF может быть любым, даже в одной сети на разных коммутаторах, без пробелов в названии. В инструкциях часто употребляется VRF с названием  "corporate". Возьмём её в качестве названия нашей VRF.

### Arista

Сначала VRF объявляется, затем в интерфейсе Vxlan на неё вешается VNI тег, который примем 101000. Затем, прописывается динамическая маршрутизация, по аналогии с Vlan в предыдущей лабе. Важно, не забыть, включить маршрутизацию в VRF, о чём вам деликатно напомнят, когда будем настраивать BGP.  После чего на интерфейсы Vlan, что будут участвовать в нашем тесте, прописывается принадлежность к данной VRF.

Приведу пример для коммутатора Leaf11,для второго всё будет аналогично 

```
...
!
vrf instance corporate
!
...
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vrf corporate vni 101000
!
...
ip routing vrf corporate
...
router bgp 65001
...
   !
   vrf corporate
      rd 172.16.1.1:3
      route-target import evpn 65002:101000
      route-target import evpn 65003:101000
      route-target import evpn 65004:101000
      route-target export evpn 65001:101000
      !
      address-family ipv4
         redistribute connected
!
end
```

По идее, можно ограничить дистрибуцию Vlan при помощи Route-map, но мы это ограничиваем добавлением нужных Vlan в VRF, поэтому никаких ограничений не применяем.

### Cumulus

У данного производителя настройка производится в два этапа, сначала, VRF прописывается в системе, а затем в утилите маршрутизации. Но и это ещё не всё, необходим дополнительный интерфейс, который будет выступать узлом приёма трафика L3VPN

Часть файла конфигурации интерфейсов leaf21

```
# VRF

auto corporate
iface corporate
#    address 172.16.1.3/32
    vrf-table auto

# L3VNI

auto vlan1000
iface vlan1000
    address-virtual 44:38:39:00:01:00
    vlan-id 1000
    vlan-raw-device bridge
    vrf corporate

auto vxlan1000
iface vxlan1000
    bridge-access 1000
    vxlan-id 101000

# Bridge

auto bridge
iface bridge
    bridge-ports swp4 swp5 vxlan1000
    bridge-vids 11 13
    bridge-vlan-aware yes
```

Дополнительно, L3VPN интерфейс должен быть добавлен в bridge.

Затем, настроим маршрутизацию. Для этого пропишем VRF и закрепим VNI тег. А также создадим отдельный процесс динамической маршрутизации, куда запишем следующие настройки.

```
...
!
vrf corporate
 vni 101000
exit-vrf
!
...
router bgp 65004 vrf corporate
 bgp router-id 172.16.1.4
 !
 address-family ipv4 unicast
  redistribute connected route-map VRF_ADVERTS
 exit-address-family
 !
 address-family l2vpn evpn
  advertise ipv4 unicast
 exit-address-family
exit
!
route-map VRF_ADVERTS permit 10
 match interface corporate
exit
```


## Проверка работы
При проверке полученных маршрутов, видим что это маршруты пятого типа, что логично. У нас vlan 11 настроен на двух коммутаторах, Leaf11 и leaf21. Но работает только один из них.

<details>
<summary>На Arista, leaf11</summary>

```
leaf-11#sho bgp evpn
BGP routing table information for VRF default
Router identifier 172.16.1.1, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 172.16.1.1:3 ip-prefix 172.20.10.0/24
                                 -                     -       -       0       i
 * >      RD: 172.16.1.2:3 ip-prefix 172.20.11.0/24
                                 172.16.1.2            -       100     0       65000 65002 i
 *        RD: 172.16.1.2:3 ip-prefix 172.20.11.0/24
                                 172.16.1.2            -       100     0       65000 65002 i
 * >      RD: 172.16.1.3:2 ip-prefix 172.20.11.0/24
                                 172.16.1.3            -       100     0       65000 65003 ?
 *        RD: 172.16.1.3:2 ip-prefix 172.20.11.0/24
                                 172.16.1.3            -       100     0       65000 65003 ?
 * >      RD: 172.16.1.4:2 ip-prefix 172.20.14.0/24
                                 172.16.1.4            -       100     0       65000 65004 ?
 *        RD: 172.16.1.4:2 ip-prefix 172.20.14.0/24
                                 172.16.1.4            -       100     0       65000 65004 ?

```
</details>

<details>
<summary>На Cumulus, leaf21</summary>

```
leaf-21# sho bgp l
l2vpn                 large-community       listeners
labelpool             large-community-list
leaf-21# sho bgp l2vpn evpn
BGP table version is 5, local router ID is 172.16.1.3
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-1 prefix: [1]:[EthTag]:[ESI]:[IPlen]:[VTEP-IP]:[Frag-id]
EVPN type-2 prefix: [2]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-4 prefix: [4]:[ESI]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 172.16.1.1:3
*  [5]:[0]:[24]:[172.20.10.0]
                    172.16.1.1                             0 65000 65001 i
                    RT:65001:101000 ET:8 Rmac:50:01:00:ca:7a:8d
*>                  172.16.1.1 (spine-2)
                                                           0 65000 65001 i
                    RT:65001:101000 ET:8 Rmac:50:01:00:ca:7a:8d
Route Distinguisher: 172.16.1.2:3
*  [5]:[0]:[24]:[172.20.11.0]
                    172.16.1.2                             0 65000 65002 i
                    RT:65002:101000 ET:8 Rmac:50:01:00:80:dd:21
*>                  172.16.1.2 (spine-2)
                                                           0 65000 65002 i
                    RT:65002:101000 ET:8 Rmac:50:01:00:80:dd:21
Route Distinguisher: 172.16.1.3:2
*> [5]:[0]:[24]:[172.20.11.0]
                    172.16.1.3 (leaf-21)
                                             0         32768 ?
                    ET:8 RT:65003:101000 Rmac:50:01:00:04:00:04
Route Distinguisher: 172.16.1.4:2
*  [5]:[0]:[24]:[172.20.14.0]
                    172.16.1.4                             0 65000 65004 ?
                    RT:65004:101000 ET:8 Rmac:50:01:00:05:00:04
*>                  172.16.1.4 (spine-2)
                                                           0 65000 65004 ?
                    RT:65004:101000 ET:8 Rmac:50:01:00:05:00:04

Displayed 4 out of 7 total prefixes

```
</details>

Пинги между серверами успешно бегают, за исключением Leaf21, что понятно, так как динамически выбирается только один из двух маршрут.

## L3VPN + L2VPN

Соединим leaf11 и leaf21 L2VPN туннелем, чтобы отрезанный сегмент начал работать. Настройки возьмём из предыдущей лабы
На Cumulus, нужно будет прописать интерфейс VNI, а на Arista, прописать VNI для Vlan 11 на интерфейсе vxlan, а также добавить секцию vlan 11 c настройками RD и RT в протоколе BGP.

<details>
<summary>В результате, стали прилетать маршруты типа 2 и 3</summary>

на Arista
```
leaf-12#sho bgp evpn
BGP routing table information for VRF default
Router identifier 172.16.1.2, local AS number 65002
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 172.16.1.3:3 mac-ip 0050.0100.0901
                                 172.16.1.3            -       100     0       65000 65003 i
 *  ec    RD: 172.16.1.3:3 mac-ip 0050.0100.0901
                                 172.16.1.3            -       100     0       65000 65003 i
 * >Ec    RD: 172.16.1.3:3 mac-ip 0050.0100.0901 172.20.11.3
                                 172.16.1.3            -       100     0       65000 65003 i
 *  ec    RD: 172.16.1.3:3 mac-ip 0050.0100.0901 172.20.11.3
                                 172.16.1.3            -       100     0       65000 65003 i
 * >Ec    RD: 172.16.1.3:3 mac-ip 0050.0100.0901 fe80::250:1ff:fe00:901
                                 172.16.1.3            -       100     0       65000 65003 i
 *  ec    RD: 172.16.1.3:3 mac-ip 0050.0100.0901 fe80::250:1ff:fe00:901
                                 172.16.1.3            -       100     0       65000 65003 i
 * >      RD: 172.16.1.2:3 imet 172.16.1.2
                                 -                     -       -       0       i
 * >Ec    RD: 172.16.1.3:3 imet 172.16.1.3
                                 172.16.1.3            -       100     0       65000 65003 i
 *  ec    RD: 172.16.1.3:3 imet 172.16.1.3
                                 172.16.1.3            -       100     0       65000 65003 i
 * >      RD: 172.16.1.1:3 ip-prefix 172.20.10.0/24
                                 172.16.1.1            -       100     0       65000 65001 i
 *        RD: 172.16.1.1:3 ip-prefix 172.20.10.0/24
                                 172.16.1.1            -       100     0       65000 65001 i
 * >      RD: 172.16.1.2:3 ip-prefix 172.20.11.0/24
                                 -                     -       -       0       i
 * >      RD: 172.16.1.3:2 ip-prefix 172.20.11.0/24
                                 172.16.1.3            -       100     0       65000 65003 ?
 *        RD: 172.16.1.3:2 ip-prefix 172.20.11.0/24
                                 172.16.1.3            -       100     0       65000 65003 ?
 * >      RD: 172.16.1.4:2 ip-prefix 172.20.14.0/24
                                 172.16.1.4            -       100     0       65000 65004 ?
 *        RD: 172.16.1.4:2 ip-prefix 172.20.14.0/24
                                 172.16.1.4            -       100     0       65000 65004 ?

```
На Cumulus
```
leaf-22# sho bgp l2vpn evpn
BGP table version is 5, local router ID is 172.16.1.4
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-1 prefix: [1]:[EthTag]:[ESI]:[IPlen]:[VTEP-IP]:[Frag-id]
EVPN type-2 prefix: [2]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-4 prefix: [4]:[ESI]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 172.16.1.1:3
*  [5]:[0]:[24]:[172.20.10.0]
                    172.16.1.1                             0 65000 65001 i
                    RT:65001:101000 ET:8 Rmac:50:01:00:ca:7a:8d
*>                  172.16.1.1 (spine-2)
                                                           0 65000 65001 i
                    RT:65001:101000 ET:8 Rmac:50:01:00:ca:7a:8d
Route Distinguisher: 172.16.1.2:3
*  [3]:[0]:[32]:[172.16.1.2]
                    172.16.1.2                             0 65000 65002 i
                    RT:65002:10011 ET:8
*>                  172.16.1.2 (spine-2)
                                                           0 65000 65002 i
                    RT:65002:10011 ET:8
*  [5]:[0]:[24]:[172.20.11.0]
                    172.16.1.2                             0 65000 65002 i
                    RT:65002:101000 ET:8 Rmac:50:01:00:80:dd:21
*>                  172.16.1.2 (spine-2)
                                                           0 65000 65002 i
                    RT:65002:101000 ET:8 Rmac:50:01:00:80:dd:21
Route Distinguisher: 172.16.1.3:2
*  [5]:[0]:[24]:[172.20.11.0]
                    172.16.1.3                             0 65000 65003 ?
                    RT:65003:101000 ET:8 Rmac:50:01:00:04:00:04
*>                  172.16.1.3 (spine-2)
                                                           0 65000 65003 ?
                    RT:65003:101000 ET:8 Rmac:50:01:00:04:00:04
Route Distinguisher: 172.16.1.3:3
*  [2]:[0]:[48]:[00:50:01:00:09:01]
                    172.16.1.3                             0 65000 65003 i
                    RT:65003:10011 ET:8
*>                  172.16.1.3 (spine-2)
                                                           0 65000 65003 i
                    RT:65003:10011 ET:8
*  [2]:[0]:[48]:[00:50:01:00:09:01]:[128]:[fe80::250:1ff:fe00:901]
                    172.16.1.3                             0 65000 65003 i
                    RT:65003:10011 ET:8
*>                  172.16.1.3 (spine-2)
                                                           0 65000 65003 i
                    RT:65003:10011 ET:8
*  [3]:[0]:[32]:[172.16.1.3]
                    172.16.1.3                             0 65000 65003 i
                    RT:65003:10011 ET:8
*>                  172.16.1.3 (spine-2)
                                                           0 65000 65003 i
                    RT:65003:10011 ET:8
Route Distinguisher: 172.16.1.4:2
*> [5]:[0]:[24]:[172.20.14.0]
                    172.16.1.4 (leaf-22)
                                             0         32768 ?
                    ET:8 RT:65004:101000 Rmac:50:01:00:05:00:04

Displayed 8 out of 15 total prefixes

```
</details>

## Рефлексия

1. Почему-то не смог прописать свой MAC для VRRP на Arista. Нужная строка присутствует, но не работает. Вероятно, это ограничение виртуалки

На серверах. это выглядит так:
Сервер присоединённый к Arista
```
eve@ubuntu22-server:~$ ip a sho ens5
4: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:50:01:00:08:02 brd ff:ff:ff:ff:ff:ff
    altname enp0s5
    inet 172.20.11.2/24 brd 172.20.11.255 scope global ens5
       valid_lft forever preferred_lft forever
    inet6 fe80::250:1ff:fe00:802/64 scope link
       valid_lft forever preferred_lft forever
eve@ubuntu22-server:~$ ip neighbor show 172.20.11.254
172.20.11.254 dev ens5 lladdr 00:00:5e:00:01:01 STALE
```
На сервере присоединённому к Cumulus
```
eve@vrv3:~$ ip a sho ens4
3: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:50:01:00:09:01 brd ff:ff:ff:ff:ff:ff
    altname enp0s4
    inet 172.20.11.3/24 brd 172.20.11.255 scope global ens4
       valid_lft forever preferred_lft forever
    inet6 fe80::250:1ff:fe00:901/64 scope link
       valid_lft forever preferred_lft forever
eve@vrv3:~$ ip neighbor sho 172.20.11.254
172.20.11.254 dev ens4 lladdr 44:38:39:00:01:00 STALE
```

Интеграция работает откровенно плохо, нестабильно. MAC прилетает, то от Arista, то от Cumulus. Всё же реализация VRRP  у производителей разная.

2. Непонятна настройка RT на Arista. По идее настройка RT должна быть двумя строками, где указан его собственный ASN и VNI тег vlan на импорт и экспорт. Но этого не происходит. Странно, учитывая что Cumulus  выдаёт 
```
leaf-21# sho bgp l2vpn evpn vni
Advertise Gateway Macip: Disabled
Advertise SVI Macip: Disabled
Advertise All VNI flag: Enabled
BUM flooding: Head-end replication
VXLAN flooding: Enabled
Number of L2 VNIs: 1
Number of L3 VNIs: 1
Flags: * - Kernel
  VNI        Type RD                    Import RT                 Export RT                 MAC-VRF Site-of-Origin    Tenant VRF
* 10011      L2   172.16.1.3:3          65003:10011               65003:10011                                         corporate
* 101000     L3   172.16.1.3:2          65003:101000              65003:101000                                        corporate
```
На Arista, должны быть аналогичные настройки. 
Если настроить вышеуказанным образом, то пути прилетают. Никаких отличий, в путях при разных настройках, как теоретических, так и тех что в лабе, я не нашёл. Но трафик между серверами - не ходит.

Такие дела.