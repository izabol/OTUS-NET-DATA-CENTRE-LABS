# Лабораторная работа iBGP курса "Сетевой инженер ЦОД"
Цель: исследовать работу протокола BGP на коммутаторах под управлением OS Cumulus
Так как настройки и работа на назначенных адресах и на самоназначенных ipv6 адресах одинаковые, то, для простоты, будем работать с самоназначенными ipv6
## Первоначальная настройка
Перед работай, на leaf пропишем vlan и зададим им адреса. в данных коммутаторах, это делается изменением настроек в файле ```/etc/network/interfaces``` Для leaf-3 это будет выглядеть следующим образом:
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*.intf

# The loopback network interface
auto lo
iface lo inet loopback
    address 172.16.1.3/32

# The primary network interface
auto eth0
iface eth0 inet dhcp
    vrf mgmt

auto mgmt
iface mgmt
    address 127.0.0.1/8
    address ::1/128
    vrf-table auto

auto swp1
iface swp1

auto swp2
iface swp2

auto swp3
iface swp3
   bridge-pvid 10

auto swp4
iface swp4
   bridge-pvid 11

auto vlan10
iface vlan10
    address 172.20.10.254/24
    vlan-id 10
    vlan-raw-device bridge

auto vlan11
iface vlan11
    address 172.20.11.254/24
    vlan-id 11
    vlan-raw-device bridge

auto bridge
iface bridge
    bridge-ports swp3 swp4
    bridge-vids 10 11
    bridge-vlan-aware yes
```
Как видим, мы просто обозначаем маршрутизируемые адреса, без настройки (их настраивать мы будем в  утилите vtysh). Интерфейсы swp3 и swp4 помещаем в бридж, обозначая, тем самым, что данные порты - коммутируемые, также на каждый порт вешаем vlan-ы в режиме access. Vlan-ы, также, прописываем в бридж. Опция "bridge-vlan-aware yes" в настройках бриджа, означает, что мы выбираем модель работы коммутатора с одним бриджем и мультивлан в нём.
Применяем настройки командой ```sudo ifreload -a```
Проверяем связность с хостами. Например srv-4.
```
eve@ubuntu22-server:~$ ping 172.20.11.254
PING 172.20.11.254 (172.20.11.254) 56(84) bytes of data.
64 bytes from 172.20.11.254: icmp_seq=1 ttl=64 time=0.765 ms
64 bytes from 172.20.11.254: icmp_seq=2 ttl=64 time=0.448 ms
^C
--- 172.20.11.254 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1006ms
rtt min/avg/max/mdev = 0.448/0.606/0.765/0.158 ms
```
Связность присутствует
## Настройка p2p линков
Запустим утилиту vtysh и в ней отключим на p2p интерфейсах фильтрацию автообнаружения соседей```no ipv6 nd suppress-ra```. В результате получим:
```
leaf-3# sho run
Building configuration...

Current configuration:
!
frr version 8.4.3
frr defaults datacenter
hostname leaf-3
log syslog informational
service integrated-vtysh-config
!
interface swp1
 no ipv6 nd suppress-ra
exit
!
interface swp2
 no ipv6 nd suppress-ra
exit
...
```
Посмотрим что на интерфейсах:
```
spine-1# sho int swp1
Interface swp1 is up, line protocol is up
  Link ups:       0    last: (never)
  Link downs:     0    last: (never)
  PTM status: disabled
  vrf: default
  index 3 metric 0 mtu 9216 speed 1000
  flags: <UP,BROADCAST,RUNNING,MULTICAST>
  Ignore all v6 routes with linkdown
  Type: Ethernet
  HWaddr: 50:01:00:01:00:01
  inet6 fe80::5201:ff:fe01:1/64
  Interface Type Other
  Interface Slave Type None
  protodown: off
  ND advertised reachable time is 0 milliseconds
  ND advertised retransmit interval is 0 milliseconds
  ND advertised hop-count limit is 64 hops
  ND router advertisements sent: 5 rcvd: 5
  ND router advertisements are sent every 600 seconds
  ND router advertisements lifetime tracks ra-interval
  ND router advertisement default router preference is medium
  Hosts use stateless autoconfig for addresses.
  Neighbor address(s):
  inet6 fe80::5201:ff:fe03:1/128
```
Видим, что сосед был обнаружен, для надёжности, можем его пингануть
```
spine-1# ping ipv6 ff02::1%swp1
vrf-wrapper.sh: switching to vrf "default"; use '--no-vrf-switch' to disable
PING ff02::1%swp1(ff02::1%swp1) 56 data bytes
64 bytes from fe80::5201:ff:fe01:1%swp1: icmp_seq=1 ttl=64 time=0.038 ms
64 bytes from fe80::5201:ff:fe03:1%swp1: icmp_seq=1 ttl=64 time=0.223 ms (DUP!)
64 bytes from fe80::5201:ff:fe01:1%swp1: icmp_seq=2 ttl=64 time=0.052 ms
64 bytes from fe80::5201:ff:fe03:1%swp1: icmp_seq=2 ttl=64 time=0.475 ms (DUP!)
64 bytes from fe80::5201:ff:fe01:1%swp1: icmp_seq=3 ttl=64 time=0.053 ms
64 bytes from fe80::5201:ff:fe03:1%swp1: icmp_seq=3 ttl=64 time=0.437 ms (DUP!)
^C
--- ff02::1%swp1 ping statistics ---
3 packets transmitted, 3 received, +3 duplicates, 0% packet loss, time 45ms
rtt min/avg/max/mdev = 0.038/0.213/0.475/0.183 ms
```
Наблюдаем дубликат ответов, который и есть ответ соседа(это можно сверить с адресом который указан в выводе состояния интерфейса).
Связность установлена. Сохраняем настройки и выходим из утилиты vtysh

## Настройка маршрутизации
Для запуска протокола маршрутизации его нужно включить в настройках frr. В файле ```/etc/frr/daemons``` меняем значение напротив протокола dgpd примернотак это будет выглядеть:
```
...
# The watchfrr daemon is always started. Per default in monitoring-only but
# that can be changed.
#
bgpd=yes
ospfd=no
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
fabricd=no
vrrpd=no
...
```
Перегружаем сервис frr командой ```sudo systemctl restart frr.service``` и преступаем к настройке. Запускаем утилиту vtysh и настраиваем iBGP. Для того чтобы работал именно iBGP в настройках это явно указываем. Чтобы фабрика обменивалась маршрутами. на Spine в секции ```address-family ipv4 unicast``` добавляем опцию ```neighbor underlay route-reflector-client```. Для анонсов создаём Route-map, где матчим интерфейсы, что будут анонсироваться. По классике, там будут loopback интерфейсы, но нам, для проверки нужны Vlan. По итогу получим для Spine1:
```
Current configuration:
!
frr version 8.4.3
frr defaults datacenter
hostname spine-1
log syslog informational
service integrated-vtysh-config
!
interface swp1
 no ipv6 nd suppress-ra
exit
!
interface swp2
 no ipv6 nd suppress-ra
exit
!
interface swp3
 no ipv6 nd suppress-ra
exit
!
router bgp 65000
 bgp router-id 172.16.0.1
 neighbor underlay peer-group
 neighbor underlay remote-as internal
 neighbor swp1 interface peer-group underlay
 neighbor swp2 interface peer-group underlay
 neighbor swp3 interface peer-group underlay
 !
 address-family ipv4 unicast
  neighbor underlay route-reflector-client
 exit-address-family
exit
!
end
```
для leaf-3:
```
Current configuration:
!
frr version 8.4.3
frr defaults datacenter
hostname leaf-3
log syslog informational
service integrated-vtysh-config
!
interface swp1
 no ipv6 nd suppress-ra
exit
!
interface swp2
 no ipv6 nd suppress-ra
exit
!
router bgp 65000
 bgp router-id 172.16.1.3
 neighbor underlay peer-group
 neighbor underlay remote-as internal
 neighbor swp1 interface peer-group underlay
 neighbor swp2 interface peer-group underlay
 !
 address-family ipv4 unicast
  redistribute connected route-map ADVERTS
 exit-address-family
exit
!
route-map ADVERTS permit 10
 match interface vlan10
exit
!
route-map ADVERTS permit 20
 match interface vlan11
exit
!
end
```
Проверим соседство
```
leaf-3# sho bgp summary

IPv4 Unicast Summary (VRF default):
BGP router identifier 172.16.1.3, local AS number 65000 vrf-id 0
BGP table version 11
RIB entries 5, using 960 bytes of memory
Peers 2, using 40 KiB of memory
Peer groups 1, using 64 bytes of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
spine-1(swp1)   4      65000       196       192        0    0    0 00:09:15            2        2 N/A
spine-2(swp2)   4      65000       186       183        0    0    0 00:08:46            2        2 N/A

Total number of neighbors 2
```
Проверим прохождение трафика между серверами Srv-1 и Srv-4
```
eve@srv1:~$ ping 172.20.11.1 -c 3
PING 172.20.11.1 (172.20.11.1) 56(84) bytes of data.
64 bytes from 172.20.11.1: icmp_seq=1 ttl=61 time=1.65 ms
64 bytes from 172.20.11.1: icmp_seq=2 ttl=61 time=1.74 ms
64 bytes from 172.20.11.1: icmp_seq=3 ttl=61 time=1.66 ms

--- 172.20.11.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 1.647/1.682/1.739/0.040 ms
```
Полный успех!
