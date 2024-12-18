# Лабораторная работа L2VNI с маршрутизацией iBGP курса "Сетевой инженер ЦОД"
Цель: исследовать работу L2VNI на коммутаторах под управлением OS Cumulus используя маршрутизацию iBGP

За основу возьмём предыдущую лабу по iBGP. Изменим на leaf привязку route-map на интерфейсы loopback. А на spine добавим секцию  ```address-family l2vpn evpn``` которую активируем на интерфейсах, а также объявим анонс всех VNI

## Редактируем файл /etc/network/interfaces на leaf
1. Создаём интерфейсы vni для каждого vlan
 - добавляем в него vlan
 - указываем идентификатор
 - настраиваем stp
2. Прописываем в настройках  интерфейса loopback 0 адрес от которого будут рассылаться vxLAN
3. Добавляем интерфейсы VNI в бридж
Получим для leaf-2
```
...
auto lo
iface lo inet loopback
    address 172.16.1.2/32
    vxlan-local-tunnelip 172.16.1.2

auto vni100010
iface vni100010
    bridge-access 10
    vxlan-id 100010
    mstpctl-bpduguard yes
    mstpctl-portbpdufilter yes

auto bridge
iface bridge
    bridge-ports swp3 vni100010
    bridge-vids 10
    bridge-vlan-aware yes
```
## Редактируем FRR
Традиционно службой FRR управляют утилитой vtysh, но перед этим не забываем включить протокол в настройках ```/etc/frr/daemons```. 
За начальные условия возьмём предыдущую лабу. 
Допишем настройки секции "address-family l2vpn evpn"
Для spine-1 получим:
```
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
 !
 address-family l2vpn evpn
  neighbor underlay activate
 exit-address-family
exit
```
Для leaf-2 получим:
```
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
 !
 address-family l2vpn evpn
  neighbor underlay activate
  advertise-all-vni
 exit-address-family
exit
!
route-map ADVERTS permit 10
 match interface lo
exit
```
Проверим получение маршрутов до lo интерфейсов
```
leaf-2# sho ip bgp
BGP table version is 3, local router ID is 172.16.1.2, vrf id 0
Default local pref 100, local AS 65000
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*>i172.16.1.1/32    swp1                     0    100      0 ?
*=i                 swp2                     0    100      0 ?
*> 172.16.1.2/32    0.0.0.0                  0         32768 ?
*>i172.16.1.3/32    swp1                     0    100      0 ?
*=i                 swp2                     0    100      0 ?

Displayed  3 routes and 5 total paths
```
Маршруты присутствуют. Проверим VNI
```
leaf-2# sho evpn vni 100010
VNI: 100010
 Type: L2
 Vlan: 10
 Bridge: bridge
 Tenant VRF: default
 VxLAN interface: vni100010
 VxLAN ifIndex: 9
 Local VTEP IP: 172.16.1.2
 Mcast group: 0.0.0.0
 No remote VTEPs known for this VNI
 Number of MACs (local and remote) known for this VNI: 1
 Number of ARPs (IPv4 and IPv6, local and remote) known for this VNI: 2
 Advertise-gw-macip: No
 Advertise-svi-macip: No
```
VNI - не поднялись. (Курил мануалы, играл настройками, всё без результата. Увы)

## eBGP
Переделаем iBGP на eBGP. Для этого на spine сменим явное указание на external, а на leaf заменим номера AS на уникальные и также сменим указание. Остальные настройки оставим те же.(кроме рефлекторов, конечно, но они сами исчезли из настроек)
Для spine-2 получим:
```
router bgp 65000
 bgp router-id 172.16.0.2
 neighbor underlay peer-group
 neighbor underlay remote-as external
 neighbor swp1 interface peer-group underlay
 neighbor swp2 interface peer-group underlay
 neighbor swp3 interface peer-group underlay
 !
 address-family l2vpn evpn
  neighbor underlay activate
 exit-address-family
exit
```
На leaf-3 получим:
```
router bgp 65003
 bgp router-id 172.16.1.3
 neighbor underlay peer-group
 neighbor underlay remote-as external
 neighbor swp1 interface peer-group underlay
 neighbor swp2 interface peer-group underlay
 !
 address-family ipv4 unicast
  redistribute connected route-map ADVERTS
  neighbor underlay allowas-in origin
 exit-address-family
 !
 address-family l2vpn evpn
  neighbor underlay activate
  advertise-all-vni
 exit-address-family
exit
```
Проверим VNI:
```
leaf-3# sho evpn vni 100010
VNI: 100010
 Type: L2
 Vlan: 10
 Bridge: bridge
 Tenant VRF: default
 VxLAN interface: vni100010
 VxLAN ifIndex: 8
 SVI interface: vlan10
 SVI ifIndex: 11
 Local VTEP IP: 172.16.1.3
 Mcast group: 0.0.0.0
 Remote VTEPs for this VNI:
  172.16.1.2 flood: HER
 Number of MACs (local and remote) known for this VNI: 2
 Number of ARPs (IPv4 and IPv6, local and remote) known for this VNI: 4
 Advertise-gw-macip: No
 Advertise-svi-macip: No
```
Таой же VNI есть на leaf-2, что подтверждается в разделе "Remote VTEPs for this VNI". 
Проверим прохождение трафика между серверами srv-2 и srv-3:
```
eve@srv-2:~$ ip a sho ens4 | grep inet
    inet 172.20.10.1/24 brd 172.20.10.255 scope global ens4
    inet6 fe80::250:1ff:fe00:801/64 scope link
eve@srv-2:~$ ping 172.20.10.2 -c 3
PING 172.20.10.2 (172.20.10.2) 56(84) bytes of data.
64 bytes from 172.20.10.2: icmp_seq=1 ttl=64 time=2.00 ms
64 bytes from 172.20.10.2: icmp_seq=2 ttl=64 time=1.84 ms
64 bytes from 172.20.10.2: icmp_seq=3 ttl=64 time=1.47 ms

--- 172.20.10.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 1.474/1.773/2.001/0.220 ms
```
Трафик весело бегает между серверами. Успех! 

## OSPF-unnumbered


К работе приложена лаба из EVE-NG образ можно скачать по ссылке: https://cloud.mail.ru/public/3grP/RGfmtbnrf