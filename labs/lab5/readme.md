# Домашнее задание №5. VxLAN. L2 VNI
[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)
## Задачи
- Настроить Overlay на основе VxLAN EVPN для L2 связанности между клиентами

## Решение
### Распределение адресного пространства

Блок IP-адресов 10.0.0.0/14 для DC1
- spine-X, leaf-1YY
- блок 10.**0**.0.0/16 - loopback \
  _третий октет - коммутатор, четвертый - номер loopback_
  - loopback**0** spine - 10.1.X.**0**/32
  - loopback**0** leaf - 10.1.1YY.**0**/32
- блок 10.**1**.0.0/16 - транспорт \
 _третий октет - spine, четвертый октет сети по /31_
  - transport spine-**X** - 10.1.**X**.<сеть>/31
- блок 10.**2**.0.0/16 - сервисы \
 _третий октет - номер VLAN, сети по /24_
  - подсеть для vlan - 10.**2**.vlan.0/24
- блок 10.**3**.0.0/16 - резерв

### Описание решения VXLAN
В решении используется следующие параметры:
- в overlay используется интерфейс Loopback0 на spine и leaf
- настроено соседство между spine и leaf для BGP AFI/SFI l2vpn evpn
- команда neighbor XXX next-hop-unchanged используется для сохранения next-hop-адреса leaf-коммутатора
- команда redistribute learned используется для анонса MAC-адреса локальных хостов как EVPN type-2 маршрутов
- команда neighbor XXX send-community extended используется для работы EVPN (импорта, экспорта маршрутов)
- номер VNI выбирается так - 1ХХХХ, где ХХХХ номер VLAN
- параметр RD настраивается через auto. Коммутатор сам выставляет в RID:VLAN
- параметр RT настраивается вручную, чтобы он совпадал на всех VTEP, находящихся в разных AS
- параметр RT задается так - 65500:VNI. За счет номера VNI достигается уникальность. AS 65500 в решении не используется.

### Описание решения Underlay.eBGP (из лабы №4)
_Если необходимо вспомнить или ознакомиться_
<details>
  <summary>Описание Underlay.eBGP </summary>

  В решении используется протокол маршрутизации eBGP со следующими параметрами:
- все spine размещены в одной AS 65100
- каждый leaf размещен в свой AS: leaf-1YY в AS 651YY
- на spine используются динамические peer-group с фильтром по номеру AS и транзитному блоку /24
- на leaf используются статические peer-group
- настроены keepalive-интервал 3 сек, hold time 9 сек.
- настроен maximum-paths равным 8 (4 вероятно хватит, но указал с запасом)
- настроен BGP routing updates интервал равным 0  (neighbor out-delay, установлен в 0 по умолчанию)
- настроена administrative distance равна 20 (по рекомендации Arista из предоставленной ссылке, возможно из-за iBGP между leaf в паре)
- отключена автоматическая активация BGP AFI/SFI ipv4 unicast (в данной лабе это было не обязательно)
- включен режим multi-agent model (поддежка redistribute в BGP AFI/SFI ipv4 unicast)
- включена аутентификация BGP-соседа
- настроено взаимодействие с протоколом bfd для улучшения сходимости сети
- таймеры bfd выбраны такие, чтобы сессии в EVE-NG флапали реже
</details>

### Cхема сети
![Изображение](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab5/scheme/lab5_scheme.PNG "Схема стенда")

### Таблица IP-адресации
|Оборудование	|Интерфейс	|IP-адрес	|Назначение|
|:-|:-|:-|:-|
|dc1-spine-1	|Loopback0	|10.0.1.0/32	|-|
|dc1-spine-1	|Eth1	|10.1.1.0/31	|sp1-le101|
|dc1-spine-1	|Eth2	|10.1.1.2/31	|sp1-le102|
|dc1-spine-1	|Eth3	|10.1.1.4/31	|sp1-le103|
|dc1-spine-2	|Loopback0	|10.0.2.0/32 |-|
|dc1-spine-2	|Eth1	|10.1.2.0/31	|sp2-le101|
|dc1-spine-2	|Eth2	|10.1.2.2/31	|sp2-le102|
|dc1-spine-2	|Eth3	|10.1.2.4/31	|sp2-le103|
|dc1-leaf-101	|Loopback0	|10.0.101.0/32 |-|
|dc1-leaf-101	|Eth1	|10.1.1.1/31	|sp1-le101|
|dc1-leaf-101	|Eth2	|10.1.2.1/31	|sp2-le101|
|dc1-leaf-102	|Loopback0	|10.0.102.0/32 |-|		
|dc1-leaf-102	|Eth1	|10.1.1.3/31	|sp1-le102|
|dc1-leaf-102	|Eth2	|10.1.2.3/31	|sp2-le102|	
|dc1-leaf-103	|Loopback0	|10.0.103.0/32 |-|	
|dc1-leaf-103	|Eth1	|10.1.1.5/31	|sp1-le103|
|dc1-leaf-103	|Eth2	|10.1.2.5/31	|sp2-le103|
|dc1-client-101	|Eth0	|10.2.10.101/24	|Клиентская сеть, VLAN 10|
|dc1-client-102	|Eth0	|10.2.10.102/24	|Клиентская сеть, VLAN 10|
|dc1-client-103	|Eth0	|10.2.10.103/24	|Клиентская сеть, VLAN 10|
|dc1-client-104	|Eth0	|10.2.10.104/24	|Клиентская сеть, VLAN 10|

### Настройка оборудования
_Команды hostname и no switchport не показаны для облегчения восприятия настроек_ \
_Порты 7,8 в vlan 10 настроены на всех leaf для едиообразия, по факту используется только на leaf-103_
<details>
  <summary>Команды для настройки </summary>

- spine-1
```
service routing protocols model multi-agent
!
interface Ethernet1
   description ### sp1-le101 ###
   ip address 10.1.1.0/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet2
   description ### sp1-le102 ###
   ip address 10.1.1.2/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet3
   description ### sp1-le103 ###
   ip address 10.1.1.4/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Loopback0
   ip address 10.0.1.0/32
!
ip routing
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
peer-filter PF-DC1-LEAF
   10 match as-range 65101-65199 result accept
!
router bgp 65100
   router-id 10.0.1.0
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   bgp listen range 10.1.1.0/24 peer-group DC1-LEAF peer-filter PF-DC1-LEAF
   neighbor DC1-LEAF peer group
   neighbor DC1-LEAF bfd
   neighbor DC1-LEAF timers 3 9
   neighbor DC1-LEAF password 7 IS09sfEdsucPgvWfPXx0cQ==
   neighbor DC1-LEAF send-community extended
   !
   address-family evpn
      neighbor DC1-LEAF activate
      neighbor DC1-LEAF next-hop-unchanged
   !
   address-family ipv4
      neighbor DC1-LEAF activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
```
- spine-2
```
service routing protocols model multi-agent
!
interface Ethernet1
   description ### sp2-le101 ###
   ip address 10.1.2.0/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet2
   description ### sp2-le102 ###
   ip address 10.1.2.2/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet3
   description ### sp2-le103 ###
   ip address 10.1.2.4/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Loopback0
   ip address 10.0.2.0/32
!
ip routing
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
peer-filter PF-DC1-LEAF
   10 match as-range 65101-65199 result accept
!
router bgp 65100
   router-id 10.0.2.0
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   bgp listen range 10.1.2.0/24 peer-group DC1-LEAF peer-filter PF-DC1-LEAF
   neighbor DC1-LEAF peer group
   neighbor DC1-LEAF bfd
   neighbor DC1-LEAF timers 3 9
   neighbor DC1-LEAF password 7 IS09sfEdsucPgvWfPXx0cQ==
   neighbor DC1-LEAF send-community extended
   !
   address-family evpn
      neighbor DC1-LEAF activate
      neighbor DC1-LEAF next-hop-unchanged
   !
   address-family ipv4
      neighbor DC1-LEAF activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
```
- leaf-101
```
service routing protocols model multi-agent
!
vlan 10
   name NET-10
!
interface Ethernet1
   description ### sp1-le101 ###
   ip address 10.1.1.1/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet2
   description ### sp2-le101 ###
   ip address 10.1.2.1/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet7
   switchport access vlan 10
!
interface Ethernet8
   switchport access vlan 10
!
interface Loopback0
   ip address 10.0.101.0/32
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
!
ip routing
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
router bgp 65101
   router-id 10.0.101.0
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-SPINE peer group
   neighbor DC1-SPINE remote-as 65100
   neighbor DC1-SPINE bfd
   neighbor DC1-SPINE timers 3 9
   neighbor DC1-SPINE password 7 txq0MZ/aCqwJ+sp2WtntdQ==
   neighbor DC1-SPINE send-community extended
   neighbor 10.1.1.0 peer group DC1-SPINE
   neighbor 10.1.1.0 description ### dc1-spine-1 ###
   neighbor 10.1.2.0 peer group DC1-SPINE
   neighbor 10.1.2.0 description ### dc1-spine-2 ###
   !
   vlan 10
      rd auto
      route-target both 65500:10010
      redistribute learned
   !
   address-family evpn
      neighbor DC1-SPINE activate
   !
   address-family ipv4
      neighbor DC1-SPINE activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
```
- leaf-102
```
service routing protocols model multi-agent
!
vlan 10
   name NET-10
!
interface Ethernet1
   description ### sp1-le102 ###
   ip address 10.1.1.3/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet2
   description ### sp2-le102 ###
   ip address 10.1.2.3/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet7
   switchport access vlan 10
!
interface Ethernet8
   switchport access vlan 10
!
interface Loopback0
   ip address 10.0.102.0/32
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
!
ip routing
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
router bgp 65102
   router-id 10.0.102.0
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-SPINE peer group
   neighbor DC1-SPINE remote-as 65100
   neighbor DC1-SPINE bfd
   neighbor DC1-SPINE timers 3 9
   neighbor DC1-SPINE password 7 txq0MZ/aCqwJ+sp2WtntdQ==
   neighbor DC1-SPINE send-community extended
   neighbor 10.1.1.2 peer group DC1-SPINE
   neighbor 10.1.1.2 description ### dc1-spine-1 ###
   neighbor 10.1.2.2 peer group DC1-SPINE
   neighbor 10.1.2.2 description ### dc1-spine-2 ###
   !
   vlan 10
      rd auto
      route-target both 65500:10010
      redistribute learned
   !
   address-family evpn
      neighbor DC1-SPINE activate
   !
   address-family ipv4
      neighbor DC1-SPINE activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
```
- leaf-103
```
service routing protocols model multi-agent
!
vlan 10
   name NET-10
!
interface Ethernet1
   description ### sp1-le103 ###
   no switchport
   ip address 10.1.1.5/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet2
   description ### sp2-le103 ###
   no switchport
   ip address 10.1.2.5/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet7
   switchport access vlan 10
!
interface Ethernet8
   switchport access vlan 10
!
interface Loopback0
   ip address 10.0.103.0/32
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
!
ip routing
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
router bgp 65103
   router-id 10.0.103.0
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-SPINE peer group
   neighbor DC1-SPINE remote-as 65100
   neighbor DC1-SPINE bfd
   neighbor DC1-SPINE timers 3 9
   neighbor DC1-SPINE password 7 txq0MZ/aCqwJ+sp2WtntdQ==
   neighbor DC1-SPINE send-community extended
   neighbor 10.1.1.4 peer group DC1-SPINE
   neighbor 10.1.1.4 description ### dc1-spine-1 ###
   neighbor 10.1.2.4 peer group DC1-SPINE
   neighbor 10.1.2.4 description ### dc1-spine-2 ###
   !
   vlan 10
      rd auto
      route-target both 65500:10010
      redistribute learned
   !
   address-family evpn
      neighbor DC1-SPINE activate
   !
   address-family ipv4
      neighbor DC1-SPINE activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
```

- client-101
```
set pcname client-101
ip 10.2.10.101/24 10.2.10.254
save
```

- client-102
```
set pcname client-102
ip 10.2.10.102/24 10.2.10.254
save
```

- client-103
```
set pcname client-103
ip 10.2.10.103/24 10.2.10.254
save
```

- client-104
```
set pcname client-104
ip 10.2.10.104/24 10.2.10.254
save
```

</details>

### Проверка взаимодействия

<details>
  <summary>вывод ip/mac хостов </summary>
  
```
client-101> show ip all
NAME   IP/MASK              GATEWAY           MAC                DNS
client-10.2.10.101/24       10.2.10.254       00:50:79:66:68:07  

client-102> show ip all
NAME   IP/MASK              GATEWAY           MAC                DNS
client-10.2.10.102/24       10.2.10.254       00:50:79:66:68:08  

client-103> show ip all
NAME   IP/MASK              GATEWAY           MAC                DNS
client-10.2.10.103/24       10.2.10.254       00:50:79:66:68:09  

client-104> show ip all
NAME   IP/MASK              GATEWAY           MAC                DNS
client-10.2.10.104/24       10.2.10.254       00:50:79:66:68:0a  
```
</details>

<details>
  <summary>проверки spine-1</summary>
  
```
dc1-spine-1#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.1.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.1.1.1 4 65101            235       234    0    0 00:09:34 Estab   2      2
  10.1.1.3 4 65102           2113      2118    0    0 01:29:36 Estab   2      2
  10.1.1.5 4 65103           2114      2112    0    0 01:29:36 Estab   3      3
```
```
dc1-spine-1#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.1.0, local AS number 65100
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.101.0:10 mac-ip 0050.7966.6807
                                 10.0.101.0            -       100     0       65101 i
 * >      RD: 10.0.102.0:10 mac-ip 0050.7966.6808
                                 10.0.102.0            -       100     0       65102 i
 * >      RD: 10.0.103.0:10 mac-ip 0050.7966.6809
                                 10.0.103.0            -       100     0       65103 i
 * >      RD: 10.0.103.0:10 mac-ip 0050.7966.680a
                                 10.0.103.0            -       100     0       65103 i
 * >      RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65101 i
 * >      RD: 10.0.102.0:10 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65102 i
 * >      RD: 10.0.103.0:10 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65103 i
```
</details>


<details>
  <summary>проверки spine-2</summary>
  
```
dc1-spine-2#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.2.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.1.2.1 4 65101            237       235    0    0 00:09:34 Estab   2      2
  10.1.2.3 4 65102           2863      2843    0    0 02:00:56 Estab   2      2
  10.1.2.5 4 65103           4328      4319    0    0 03:03:22 Estab   3      3
```
```
dc1-spine-2#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.2.0, local AS number 65100
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.101.0:10 mac-ip 0050.7966.6807
                                 10.0.101.0            -       100     0       65101 i
 * >      RD: 10.0.102.0:10 mac-ip 0050.7966.6808
                                 10.0.102.0            -       100     0       65102 i
 * >      RD: 10.0.103.0:10 mac-ip 0050.7966.6809
                                 10.0.103.0            -       100     0       65103 i
 * >      RD: 10.0.103.0:10 mac-ip 0050.7966.680a
                                 10.0.103.0            -       100     0       65103 i
 * >      RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65101 i
 * >      RD: 10.0.102.0:10 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65102 i
 * >      RD: 10.0.103.0:10 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65103 i
```
</details>

<details>
  <summary>проверки leaf-101</summary>
  
```
dc1-leaf-101#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.101.0, local AS number 65101
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-spine-1 ###      10.1.1.0 4 65100           9137      9171    0    0 00:09:34 Estab   5      5
  ### dc1-spine-2 ###      10.1.2.0 4 65100           7135      7155    0    0 00:09:34 Estab   5      5

```
_Вывод команд по отдельности show bgp evpn route-type < mac-ip | imet > не стал выкладывать, там тоже самое_
```
dc1-leaf-101#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.101.0, local AS number 65101
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.101.0:10 mac-ip 0050.7966.6807
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:10 mac-ip 0050.7966.6808
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 mac-ip 0050.7966.6808
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.103.0:10 mac-ip 0050.7966.6809
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:10 mac-ip 0050.7966.6809
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:10 mac-ip 0050.7966.680a
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:10 mac-ip 0050.7966.680a
                                 10.0.103.0            -       100     0       65100 65103 i
 * >      RD: 10.0.101.0:10 imet 10.0.101.0
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:10 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.103.0:10 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:10 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
```
```
dc1-leaf-101#show interface vxlan1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback0 and is active with 10.0.101.0
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is 
    [10, 10010]      
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Headend replication flood vtep list is:
    10 10.0.102.0      10.0.103.0     
  Shared Router MAC is 0000.0000.0000
```
```
dc1-leaf-101#sh vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10010       10         static       Ethernet7       untagged  
                                    Ethernet8       untagged  
                                    Vxlan1          10        

VNI to dynamic VLAN Mapping for Vxlan1
VNI       VLAN       VRF       Source       
--------- ---------- --------- ------------ 
```
```
dc1-leaf-101#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP             Tunnel Type(s)
---------------- --------------
10.0.102.0       unicast, flood
10.0.103.0       unicast, flood

Total number of remote VTEPS:  2
```
```
dc1-leaf-101#show mac address-table vlan 10
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    0050.7966.6807    DYNAMIC     Et7        1       0:03:28 ago
  10    0050.7966.6808    DYNAMIC     Vx1        1       0:03:27 ago
  10    0050.7966.6809    DYNAMIC     Vx1        1       0:03:27 ago
  10    0050.7966.680a    DYNAMIC     Vx1        1       0:03:27 ago
Total Mac Addresses for this criterion: 4

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
```
```
dc1-leaf-101#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  0050.7966.6808  EVPN      Vx1  10.0.102.0       1       0:03:35 ago
  10  0050.7966.6809  EVPN      Vx1  10.0.103.0       1       0:03:35 ago
  10  0050.7966.680a  EVPN      Vx1  10.0.103.0       1       0:03:35 ago
Total Remote Mac Addresses for this criterion: 3
```
_Вывод для всех MAC не стал выкладывать, но он есть_
```
dc1-leaf-101#show bgp evpn route-type mac-ip 0050.7966.6807 detail 
BGP routing table information for VRF default
Router identifier 10.0.101.0, local AS number 65101
BGP routing table entry for mac-ip 0050.7966.6807, Route Distinguisher: 10.0.101.0:10
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: Route-Target-AS:65500:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000

dc1-leaf-101#show bgp evpn route-type mac-ip 0050.7966.6808 detail 
BGP routing table information for VRF default
Router identifier 10.0.101.0, local AS number 65101
BGP routing table entry for mac-ip 0050.7966.6808, Route Distinguisher: 10.0.102.0:10
 Paths: 2 available
  65100 65102
    10.0.102.0 from 10.1.2.0 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:65500:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
  65100 65102
    10.0.102.0 from 10.1.1.0 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:65500:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
```

</details>

<details>
  <summary>проверки leaf-102</summary>
  
```
dc1-leaf-102#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.102.0, local AS number 65102
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-spine-1 ###      10.1.1.2 4 65100           2886      2885    0    0 01:29:36 Estab   5      5
  ### dc1-spine-2 ###      10.1.2.2 4 65100           2843      2863    0    0 02:00:56 Estab   5      5
```
```
dc1-leaf-102#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.102.0, local AS number 65102
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.101.0:10 mac-ip 0050.7966.6807
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 mac-ip 0050.7966.6807
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:10 mac-ip 0050.7966.6808
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.103.0:10 mac-ip 0050.7966.6809
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:10 mac-ip 0050.7966.6809
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:10 mac-ip 0050.7966.680a
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:10 mac-ip 0050.7966.680a
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:10 imet 10.0.102.0
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.103.0:10 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:10 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
```
```
dc1-leaf-102#show interface vxlan1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback0 and is active with 10.0.102.0
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is 
    [10, 10010]      
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Headend replication flood vtep list is:
    10 10.0.101.0      10.0.103.0     
  Shared Router MAC is 0000.0000.0000
```
```
dc1-leaf-102#sh vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10010       10         static       Ethernet7       untagged  
                                    Ethernet8       untagged  
                                    Vxlan1          10        

VNI to dynamic VLAN Mapping for Vxlan1
VNI       VLAN       VRF       Source       
--------- ---------- --------- ------------ 
```
```
dc1-leaf-102#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP             Tunnel Type(s)
---------------- --------------
10.0.101.0       flood, unicast
10.0.103.0       flood, unicast

Total number of remote VTEPS:  2
```
```
dc1-leaf-102#show mac address-table vlan 10
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    0050.7966.6807    DYNAMIC     Vx1        1       0:03:27 ago
  10    0050.7966.6808    DYNAMIC     Et7        1       0:03:28 ago
  10    0050.7966.6809    DYNAMIC     Vx1        1       0:03:28 ago
  10    0050.7966.680a    DYNAMIC     Vx1        1       0:03:28 ago
Total Mac Addresses for this criterion: 4

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
```
```
dc1-leaf-102#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  0050.7966.6807  EVPN      Vx1  10.0.101.0       1       0:03:35 ago
  10  0050.7966.6809  EVPN      Vx1  10.0.103.0       1       0:03:35 ago
  10  0050.7966.680a  EVPN      Vx1  10.0.103.0       1       0:03:35 ago
Total Remote Mac Addresses for this criterion: 3
```
```
dc1-leaf-102#show bgp evpn route-type mac-ip 0050.7966.6807 detail 
BGP routing table information for VRF default
Router identifier 10.0.102.0, local AS number 65102
BGP routing table entry for mac-ip 0050.7966.6807, Route Distinguisher: 10.0.101.0:10
 Paths: 2 available
  65100 65101
    10.0.101.0 from 10.1.1.2 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:65500:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
  65100 65101
    10.0.101.0 from 10.1.2.2 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:65500:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000

dc1-leaf-102#show bgp evpn route-type mac-ip 0050.7966.6808 detail 
BGP routing table information for VRF default
Router identifier 10.0.102.0, local AS number 65102
BGP routing table entry for mac-ip 0050.7966.6808, Route Distinguisher: 10.0.102.0:10
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: Route-Target-AS:65500:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
```

</details>

<details>
  <summary>проверки leaf-103</summary>
  
```
dc1-leaf-103#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.103.0, local AS number 65103
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-spine-1 ###      10.1.1.4 4 65100           7235      7268    0    0 01:29:36 Estab   4      4
  ### dc1-spine-2 ###      10.1.2.4 4 65100           7111      7140    0    0 03:03:22 Estab   4      4
```
```
dc1-leaf-103#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.103.0, local AS number 65103
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.101.0:10 mac-ip 0050.7966.6807
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 mac-ip 0050.7966.6807
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:10 mac-ip 0050.7966.6808
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 mac-ip 0050.7966.6808
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.103.0:10 mac-ip 0050.7966.6809
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:10 mac-ip 0050.7966.680a
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:10 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.103.0:10 imet 10.0.103.0
```
```                                 -                     -       -       0       i
dc1-leaf-103#show interface vxlan1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback0 and is active with 10.0.103.0
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is 
    [10, 10010]      
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Headend replication flood vtep list is:
    10 10.0.102.0      10.0.101.0     
  Shared Router MAC is 0000.0000.0000
```
```
dc1-leaf-103#sh vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10010       10         static       Ethernet7       untagged  
                                    Ethernet8       untagged  
                                    Vxlan1          10        

VNI to dynamic VLAN Mapping for Vxlan1
VNI       VLAN       VRF       Source       
--------- ---------- --------- ------------ 
```
```
dc1-leaf-103#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP             Tunnel Type(s)
---------------- --------------
10.0.101.0       flood, unicast
10.0.102.0       flood, unicast

Total number of remote VTEPS:  2
```
``` 
dc1-leaf-103#show mac address-table vlan 10
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    0050.7966.6807    DYNAMIC     Vx1        1       0:03:27 ago
  10    0050.7966.6808    DYNAMIC     Vx1        1       0:03:27 ago
  10    0050.7966.6809    DYNAMIC     Et7        1       0:03:28 ago
  10    0050.7966.680a    DYNAMIC     Et8        1       0:03:28 ago
Total Mac Addresses for this criterion: 4

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
```
``` 
dc1-leaf-103#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  0050.7966.6807  EVPN      Vx1  10.0.101.0       1       0:03:35 ago
  10  0050.7966.6808  EVPN      Vx1  10.0.102.0       1       0:03:35 ago
Total Remote Mac Addresses for this criterion: 2
```
``` 
dc1-leaf-103#show bgp evpn route-type mac-ip 0050.7966.6807 detail 
BGP routing table information for VRF default
Router identifier 10.0.103.0, local AS number 65103
BGP routing table entry for mac-ip 0050.7966.6807, Route Distinguisher: 10.0.101.0:10
 Paths: 2 available
  65100 65101
    10.0.101.0 from 10.1.1.4 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:65500:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
  65100 65101
    10.0.101.0 from 10.1.2.4 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:65500:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000

dc1-leaf-103#show bgp evpn route-type mac-ip 0050.7966.6808 detail 
BGP routing table information for VRF default
Router identifier 10.0.103.0, local AS number 65103
BGP routing table entry for mac-ip 0050.7966.6808, Route Distinguisher: 10.0.102.0:10
 Paths: 2 available
  65100 65102
    10.0.102.0 from 10.1.2.4 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:65500:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
  65100 65102
    10.0.102.0 from 10.1.1.4 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:65500:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010 ESI: 0000:0000:0000:0000:0000
```
</details>

<details>
  <summary>проверки с client-101</summary>
  
```
client-101> ping 10.2.10.101
10.2.10.101 icmp_seq=1 ttl=64 time=0.001 ms
10.2.10.101 icmp_seq=2 ttl=64 time=0.001 ms
10.2.10.101 icmp_seq=3 ttl=64 time=0.001 ms
10.2.10.101 icmp_seq=4 ttl=64 time=0.001 ms
10.2.10.101 icmp_seq=5 ttl=64 time=0.001 ms

client-101> ping 10.2.10.102
84 bytes from 10.2.10.102 icmp_seq=1 ttl=64 time=55.942 ms
84 bytes from 10.2.10.102 icmp_seq=2 ttl=64 time=67.560 ms
84 bytes from 10.2.10.102 icmp_seq=3 ttl=64 time=47.139 ms
84 bytes from 10.2.10.102 icmp_seq=4 ttl=64 time=39.613 ms
84 bytes from 10.2.10.102 icmp_seq=5 ttl=64 time=36.036 ms

client-101> ping 10.2.10.103
84 bytes from 10.2.10.103 icmp_seq=1 ttl=64 time=48.365 ms
84 bytes from 10.2.10.103 icmp_seq=2 ttl=64 time=73.217 ms
84 bytes from 10.2.10.103 icmp_seq=3 ttl=64 time=42.776 ms
84 bytes from 10.2.10.103 icmp_seq=4 ttl=64 time=43.088 ms
84 bytes from 10.2.10.103 icmp_seq=5 ttl=64 time=50.260 ms

client-101> ping 10.2.10.104
84 bytes from 10.2.10.104 icmp_seq=1 ttl=64 time=59.791 ms
84 bytes from 10.2.10.104 icmp_seq=2 ttl=64 time=43.731 ms
84 bytes from 10.2.10.104 icmp_seq=3 ttl=64 time=44.790 ms
84 bytes from 10.2.10.104 icmp_seq=4 ttl=64 time=30.457 ms
84 bytes from 10.2.10.104 icmp_seq=5 ttl=64 time=33.390 ms

```
</details>

<details>
  <summary>проверки с client-102</summary>
  
```
client-102> ping 10.2.10.101
84 bytes from 10.2.10.101 icmp_seq=1 ttl=64 time=94.490 ms
84 bytes from 10.2.10.101 icmp_seq=2 ttl=64 time=44.310 ms
84 bytes from 10.2.10.101 icmp_seq=3 ttl=64 time=45.384 ms
84 bytes from 10.2.10.101 icmp_seq=4 ttl=64 time=35.507 ms
84 bytes from 10.2.10.101 icmp_seq=5 ttl=64 time=33.136 ms

client-102> ping 10.2.10.102
10.2.10.102 icmp_seq=1 ttl=64 time=0.001 ms
10.2.10.102 icmp_seq=2 ttl=64 time=0.001 ms
10.2.10.102 icmp_seq=3 ttl=64 time=0.001 ms
10.2.10.102 icmp_seq=4 ttl=64 time=0.001 ms
10.2.10.102 icmp_seq=5 ttl=64 time=0.001 ms

client-102> ping 10.2.10.103
84 bytes from 10.2.10.103 icmp_seq=1 ttl=64 time=51.648 ms
84 bytes from 10.2.10.103 icmp_seq=2 ttl=64 time=61.367 ms
84 bytes from 10.2.10.103 icmp_seq=3 ttl=64 time=44.492 ms
84 bytes from 10.2.10.103 icmp_seq=4 ttl=64 time=49.014 ms
84 bytes from 10.2.10.103 icmp_seq=5 ttl=64 time=51.349 ms

client-102> ping 10.2.10.104
84 bytes from 10.2.10.104 icmp_seq=1 ttl=64 time=59.131 ms
84 bytes from 10.2.10.104 icmp_seq=2 ttl=64 time=39.821 ms
84 bytes from 10.2.10.104 icmp_seq=3 ttl=64 time=33.710 ms
84 bytes from 10.2.10.104 icmp_seq=4 ttl=64 time=29.395 ms
84 bytes from 10.2.10.104 icmp_seq=5 ttl=64 time=47.674 ms

```
</details>


<details>
  <summary>проверки с client-103</summary>
  
```
client-103> ping 10.2.10.101
84 bytes from 10.2.10.101 icmp_seq=1 ttl=64 time=137.180 ms
84 bytes from 10.2.10.101 icmp_seq=2 ttl=64 time=46.000 ms
84 bytes from 10.2.10.101 icmp_seq=3 ttl=64 time=58.151 ms
84 bytes from 10.2.10.101 icmp_seq=4 ttl=64 time=48.096 ms
84 bytes from 10.2.10.101 icmp_seq=5 ttl=64 time=42.235 ms

client-103> ping 10.2.10.102
84 bytes from 10.2.10.102 icmp_seq=1 ttl=64 time=44.636 ms
84 bytes from 10.2.10.102 icmp_seq=2 ttl=64 time=42.281 ms
84 bytes from 10.2.10.102 icmp_seq=3 ttl=64 time=47.920 ms
84 bytes from 10.2.10.102 icmp_seq=4 ttl=64 time=50.842 ms
84 bytes from 10.2.10.102 icmp_seq=5 ttl=64 time=51.838 ms

client-103> ping 10.2.10.103
10.2.10.103 icmp_seq=1 ttl=64 time=0.001 ms
10.2.10.103 icmp_seq=2 ttl=64 time=0.001 ms
10.2.10.103 icmp_seq=3 ttl=64 time=0.001 ms
10.2.10.103 icmp_seq=4 ttl=64 time=0.001 ms
10.2.10.103 icmp_seq=5 ttl=64 time=0.001 ms

client-103> ping 10.2.10.104
84 bytes from 10.2.10.104 icmp_seq=1 ttl=64 time=15.211 ms
84 bytes from 10.2.10.104 icmp_seq=2 ttl=64 time=8.362 ms
84 bytes from 10.2.10.104 icmp_seq=3 ttl=64 time=10.529 ms
84 bytes from 10.2.10.104 icmp_seq=4 ttl=64 time=7.569 ms
84 bytes from 10.2.10.104 icmp_seq=5 ttl=64 time=9.621 ms
```
</details>


<details>
  <summary>проверки с client-104</summary>
  
```
client-104> ping 10.2.10.101
84 bytes from 10.2.10.101 icmp_seq=1 ttl=64 time=127.328 ms
84 bytes from 10.2.10.101 icmp_seq=2 ttl=64 time=49.545 ms
84 bytes from 10.2.10.101 icmp_seq=3 ttl=64 time=58.084 ms
84 bytes from 10.2.10.101 icmp_seq=4 ttl=64 time=48.284 ms
84 bytes from 10.2.10.101 icmp_seq=5 ttl=64 time=42.762 ms

client-104> ping 10.2.10.102
84 bytes from 10.2.10.102 icmp_seq=1 ttl=64 time=48.494 ms
84 bytes from 10.2.10.102 icmp_seq=2 ttl=64 time=42.314 ms
84 bytes from 10.2.10.102 icmp_seq=3 ttl=64 time=48.468 ms
84 bytes from 10.2.10.102 icmp_seq=4 ttl=64 time=49.793 ms
84 bytes from 10.2.10.102 icmp_seq=5 ttl=64 time=52.673 ms

client-104> ping 10.2.10.103
84 bytes from 10.2.10.103 icmp_seq=1 ttl=64 time=15.369 ms
84 bytes from 10.2.10.103 icmp_seq=2 ttl=64 time=17.487 ms
84 bytes from 10.2.10.103 icmp_seq=3 ttl=64 time=9.396 ms
84 bytes from 10.2.10.103 icmp_seq=4 ttl=64 time=12.041 ms
84 bytes from 10.2.10.103 icmp_seq=5 ttl=64 time=11.671 ms

client-104> ping 10.2.10.104
10.2.10.104 icmp_seq=1 ttl=64 time=0.001 ms
10.2.10.104 icmp_seq=2 ttl=64 time=0.001 ms
10.2.10.104 icmp_seq=3 ttl=64 time=0.001 ms
10.2.10.104 icmp_seq=4 ttl=64 time=0.001 ms
10.2.10.104 icmp_seq=5 ttl=64 time=0.001 ms
```
</details>


### Итоговые конфигурации оборудования
- [dc1-spine-1](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab5/config/dc1-spine-1.txt)
- [dc1-spine-2](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab5/config/dc1-spine-2.txt)
- [dc1-leaf-101](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab5/config/dc1-leaf-101.txt)
- [dc1-leaf-102](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab5/config/dc1-leaf-102.txt)
- [dc1-leaf-103](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab5/config/dc1-leaf-103.txt)
- [dc1-clients](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab5/config/dc1-clients.txt)
---

[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)
