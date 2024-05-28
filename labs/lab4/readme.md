# Домашнее задание №4. Underlay.eBGP
[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)
## Задачи
- Настроить eBGP для Underlay сети

## Решение
### Распределение адресного пространства
_Если необходимо вспомнить или ознакомиться_
<details>
  <summary>Логика распределения адресного пространства </summary>

Блок IP-адресов 10.0.0.0/14 для DC1
- spine-X, leaf-1YY
- блок 10.**0**.0.0/16 - loopback \
  _третий октет - коммутатор, четвертый - номер loopback_
  - loopback**0** spine - 10.1.X.**0**/32
  - loopback**0** leaf - 10.1.1YY.**0**/32
- блок 10.**1**.0.0/16 - транспорт \
 _третий октет - spine, четвертый октет сети по /31_
  - transport spine-**X** - 10.10.**X**.<сеть>/31
- блок 10.**2**.0.0/16 - сервисы
- блок 10.**3**.0.0/16 - резерв
</details>

### Описание решения
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

### Cхема сети
![Изображение](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab3/scheme/lab3_scheme.PNG "Схема стенда")

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

### Настройка оборудования
_Команды hostname и no switchport не показаны для облегчения восприятия настроек_
<details>
  <summary>Команды для настройки </summary>

- spine-1
```
service routing protocols model multi-agent
!
interface Ethernet1
   description ### sp1-le101 ###
   ip address 10.1.1.0/31
   bfd interval 350 min-rx 350 multiplier 3
!
interface Ethernet2
   description ### sp1-le102 ###
   ip address 10.1.1.2/31
   bfd interval 350 min-rx 350 multiplier 3
!
interface Ethernet3
   description ### sp1-le103 ###
   ip address 10.1.1.4/31
   bfd interval 350 min-rx 350 multiplier 3
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
   bfd interval 350 min-rx 350 multiplier 3
!
interface Ethernet2
   description ### sp2-le102 ###
   ip address 10.1.2.2/31
   bfd interval 350 min-rx 350 multiplier 3
!
interface Ethernet3
   description ### sp2-le103 ###
   ip address 10.1.2.4/31
   bfd interval 350 min-rx 350 multiplier 3
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
   !
   address-family ipv4
      neighbor DC1-LEAF activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
```
- leaf-101
```
service routing protocols model multi-agent
!
interface Ethernet1
   description ### sp1-le101 ###
   ip address 10.1.1.1/31
   bfd interval 350 min-rx 350 multiplier 3
!
interface Ethernet2
   description ### sp2-le101 ###
   ip address 10.1.2.1/31
   bfd interval 350 min-rx 350 multiplier 3
!
interface Loopback0
   ip address 10.0.101.0/32
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
   neighbor 10.1.1.0 peer group DC1-SPINE
   neighbor 10.1.1.0 description ### dc1-spine-1 ###
   neighbor 10.1.2.0 peer group DC1-SPINE
   neighbor 10.1.2.0 description ### dc1-spine-2 ###
   !
   address-family ipv4
      neighbor DC1-SPINE activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
```
- leaf-102
```
service routing protocols model multi-agent
!
interface Ethernet1
   description ### sp1-le102 ###
   ip address 10.1.1.3/31
   bfd interval 350 min-rx 350 multiplier 3
!
interface Ethernet2
   description ### sp2-le102 ###
   ip address 10.1.2.3/31
   bfd interval 350 min-rx 350 multiplier 3
!
interface Loopback0
   ip address 10.0.102.0/32
!
ip routing
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
router bgp 65102
   router-id 10.0.101.0
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-SPINE peer group
   neighbor DC1-SPINE remote-as 65100
   neighbor DC1-SPINE bfd
   neighbor DC1-SPINE timers 3 9
   neighbor DC1-SPINE password 7 txq0MZ/aCqwJ+sp2WtntdQ==
   neighbor 10.1.1.2 peer group DC1-SPINE
   neighbor 10.1.1.2 description ### dc1-spine-1 ###
   neighbor 10.1.2.2 peer group DC1-SPINE
   neighbor 10.1.2.2 description ### dc1-spine-2 ###
   !
   address-family ipv4
      neighbor DC1-SPINE activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
```
- leaf-103
```
service routing protocols model multi-agent
!
interface Ethernet1
   description ### sp1-le103 ###
   ip address 10.1.1.5/31
   bfd interval 350 min-rx 350 multiplier 3
!
interface Ethernet2
   description ### sp2-le103 ###
   ip address 10.1.2.5/31
   bfd interval 350 min-rx 350 multiplier 3
!
interface Loopback0
   ip address 10.0.103.0/32
!
ip routing
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
router bgp 65103
   router-id 10.0.101.0
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-SPINE peer group
   neighbor DC1-SPINE remote-as 65100
   neighbor DC1-SPINE bfd
   neighbor DC1-SPINE timers 3 9
   neighbor DC1-SPINE password 7 txq0MZ/aCqwJ+sp2WtntdQ==
   neighbor 10.1.1.4 peer group DC1-SPINE
   neighbor 10.1.1.4 description ### dc1-spine-1 ###
   neighbor 10.1.2.4 peer group DC1-SPINE
   neighbor 10.1.2.4 description ### dc1-spine-2 ###
   !
   address-family ipv4
      neighbor DC1-SPINE activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
```
</details>

### Проверка взаимодействия

<details>
  <summary>проверки spine-1</summary>
  
```
dc1-spine-1#show bfd peers
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.1.1.1  4066614101   176768653        Ethernet1(14)  normal   05/28/24 13:34 
10.1.1.3  2818399692  2301327172        Ethernet2(15)  normal   05/28/24 13:33 
10.1.1.5  1394568715   156588905        Ethernet3(16)  normal   05/28/24 13:34 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```
```
dc1-spine-1#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.1.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.1.1.1 4 65101             51        53    0    0 00:01:57 Estab   1      1
  10.1.1.3 4 65102             75        77    0    0 00:02:53 Estab   1      1
  10.1.1.5 4 65103             45        44    0    0 00:01:42 Estab   1      1
```
```
dc1-spine-1#show ip bgp
BGP routing table information for VRF default
Router identifier 10.0.1.0, local AS number 65100
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.1.0/32            -                     -       -          -       0       i
 * >      10.0.101.0/32          10.1.1.1              0       -          100     0       65101 i
 * >      10.0.102.0/32          10.1.1.3              0       -          100     0       65102 i
 * >      10.0.103.0/32          10.1.1.5              0       -          100     0       65103 i
```
```
dc1-spine-1#show ip route
Gateway of last resort is not set

 C        10.0.1.0/32 is directly connected, Loopback0
 B E      10.0.101.0/32 [20/0] via 10.1.1.1, Ethernet1
 B E      10.0.102.0/32 [20/0] via 10.1.1.3, Ethernet2
 B E      10.0.103.0/32 [20/0] via 10.1.1.5, Ethernet3
 C        10.1.1.0/31 is directly connected, Ethernet1
 C        10.1.1.2/31 is directly connected, Ethernet2
 C        10.1.1.4/31 is directly connected, Ethernet3
```

_Ping spine-2, leaf-101, leaf-102, leaf-103_ \
_spine-2 не доступе со spine-1 т.к. они в одной AS и BGP update (NLRI) не принимается из-за наличия локальной AS в AS_PATH_
```
dc1-spine-1#ping 10.0.2.0 source loopback 0
PING 10.0.2.0 (10.0.2.0) from 10.0.1.0 : 72(100) bytes of data.
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable

--- 10.0.2.0 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 48ms

dc1-spine-1#ping 10.0.101.0 source loopback 0
PING 10.0.101.0 (10.0.101.0) from 10.0.1.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=64 time=31.5 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=64 time=25.1 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=64 time=19.9 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=64 time=19.2 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=64 time=13.2 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 81ms
rtt min/avg/max/mdev = 13.233/21.808/31.511/6.155 ms, pipe 3, ipg/ewma 20.486/26.240 ms

dc1-spine-1#ping 10.0.102.0 source loopback 0 
PING 10.0.102.0 (10.0.102.0) from 10.0.1.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=64 time=42.1 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=64 time=33.8 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=64 time=42.8 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=64 time=34.1 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=64 time=15.7 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 81ms
rtt min/avg/max/mdev = 15.742/33.728/42.808/9.768 ms, pipe 4, ipg/ewma 20.357/37.334 ms

dc1-spine-1#ping 10.0.103.0 source loopback 0
PING 10.0.103.0 (10.0.103.0) from 10.0.1.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=64 time=13.6 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=64 time=9.33 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=64 time=7.66 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=64 time=7.28 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=64 time=13.1 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 54ms
rtt min/avg/max/mdev = 7.288/10.224/13.687/2.705 ms, ipg/ewma 13.501/11.979 ms
dc1-spine-1#
```

</details>


<details>
  <summary>проверки spine-2</summary>
  
```
dc1-spine-2#show bfd peers
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.1.2.1  4039840855  1351574456        Ethernet1(14)  normal   05/28/24 13:34 
10.1.2.3  3213322091   314983775        Ethernet2(15)  normal   05/28/24 12:46 
10.1.2.5  1750786361  2075013061        Ethernet3(16)  normal   05/28/24 13:34 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```
```
dc1-spine-2#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.2.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.1.2.1 4 65101             52        51    0    0 00:01:57 Estab   1      1
  10.1.2.3 4 65102           1179      1180    0    0 00:49:52 Estab   1      1
  10.1.2.5 4 65103             44        46    0    0 00:01:42 Estab   1      1
```
```
dc1-spine-2#show ip bgp
BGP routing table information for VRF default
Router identifier 10.0.2.0, local AS number 65100
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.2.0/32            -                     -       -          -       0       i
 * >      10.0.101.0/32          10.1.2.1              0       -          100     0       65101 i
 * >      10.0.102.0/32          10.1.2.3              0       -          100     0       65102 i
 * >      10.0.103.0/32          10.1.2.5              0       -          100     0       65103 i
```
```
dc1-spine-2#show ip route
Gateway of last resort is not set

 C        10.0.2.0/32 is directly connected, Loopback0
 B E      10.0.101.0/32 [20/0] via 10.1.2.1, Ethernet1
 B E      10.0.102.0/32 [20/0] via 10.1.2.3, Ethernet2
 B E      10.0.103.0/32 [20/0] via 10.1.2.5, Ethernet3
 C        10.1.2.0/31 is directly connected, Ethernet1
 C        10.1.2.2/31 is directly connected, Ethernet2
 C        10.1.2.4/31 is directly connected, Ethernet3
```

_Ping spine-1, leaf-101, leaf-102, leaf-103_
```
dc1-spine-2#ping 10.0.1.0 source loopback 0 
PING 10.0.1.0 (10.0.1.0) from 10.0.2.0 : 72(100) bytes of data.
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable

--- 10.0.1.0 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 44ms

dc1-spine-2#ping 10.0.101.0 source loopback 0
PING 10.0.101.0 (10.0.101.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=64 time=40.4 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=64 time=36.0 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=64 time=31.9 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=64 time=12.4 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=64 time=16.3 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 118ms
rtt min/avg/max/mdev = 12.473/27.447/40.414/11.054 ms, pipe 3, ipg/ewma 29.558/33.173 ms

dc1-spine-2#ping 10.0.102.0 source loopback 0 
PING 10.0.102.0 (10.0.102.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=64 time=13.5 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=64 time=21.3 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=64 time=19.2 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=64 time=14.9 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=64 time=33.1 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 63ms
rtt min/avg/max/mdev = 13.575/20.457/33.139/6.944 ms, pipe 2, ipg/ewma 15.823/17.366 ms

dc1-spine-2#ping 10.0.103.0 source loopback 0
PING 10.0.103.0 (10.0.103.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=64 time=31.1 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=64 time=24.1 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=64 time=17.3 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=64 time=11.0 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=64 time=13.4 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 82ms
rtt min/avg/max/mdev = 11.044/19.416/31.126/7.343 ms, pipe 3, ipg/ewma 20.600/24.813 ms
dc1-spine-2#
```

</details>

<details>
  <summary>проверки leaf-101</summary>
  
```
dc1-leaf-101#show bfd peers
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.1.1.0   176768653  4066614101        Ethernet1(14)  normal   05/28/24 13:34 
10.1.2.0  1351574456  4039840855        Ethernet2(15)  normal   05/28/24 13:34 

         LastDown            LastDiag    State
-------------------- ------------------- -----
   05/28/24 13:34       No Diagnostic       Up
   05/28/24 13:34       No Diagnostic       Up
```
```
dc1-leaf-101#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.101.0, local AS number 65101
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-spine-1 ###      10.1.1.0 4 65100           1167      1179    0    0 00:01:56 Estab   3      3
  ### dc1-spine-2 ###      10.1.2.0 4 65100           1146      1166    0    0 00:01:56 Estab   3      3
```
```
dc1-leaf-101#show ip bgp
BGP routing table information for VRF default
Router identifier 10.0.101.0, local AS number 65101
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.1.0/32            10.1.1.0              0       -          100     0       65100 i
 * >      10.0.2.0/32            10.1.2.0              0       -          100     0       65100 i
 * >      10.0.101.0/32          -                     -       -          -       0       i
 * >Ec    10.0.102.0/32          10.1.1.0              0       -          100     0       65100 65102 i
 *  ec    10.0.102.0/32          10.1.2.0              0       -          100     0       65100 65102 i
 * >Ec    10.0.103.0/32          10.1.2.0              0       -          100     0       65100 65103 i
 *  ec    10.0.103.0/32          10.1.1.0              0       -          100     0       65100 65103 i
```
```
dc1-leaf-101#show ip route
Gateway of last resort is not set

 B E      10.0.1.0/32 [20/0] via 10.1.1.0, Ethernet1
 B E      10.0.2.0/32 [20/0] via 10.1.2.0, Ethernet2
 C        10.0.101.0/32 is directly connected, Loopback0
 B E      10.0.102.0/32 [20/0] via 10.1.1.0, Ethernet1
                               via 10.1.2.0, Ethernet2
 B E      10.0.103.0/32 [20/0] via 10.1.1.0, Ethernet1
                               via 10.1.2.0, Ethernet2
 C        10.1.1.0/31 is directly connected, Ethernet1
 C        10.1.2.0/31 is directly connected, Ethernet2

```

_Ping leaf-102, leaf-103_
```
dc1-leaf-101#ping 10.0.102.0 source loopback 0 
PING 10.0.102.0 (10.0.102.0) from 10.0.101.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=63 time=32.6 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=63 time=32.4 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=63 time=31.3 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=63 time=26.6 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=63 time=20.1 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 94ms
rtt min/avg/max/mdev = 20.193/28.661/32.631/4.755 ms, pipe 3, ipg/ewma 23.709/30.287 ms

dc1-leaf-101#ping 10.0.103.0 source loopback 0
PING 10.0.103.0 (10.0.103.0) from 10.0.101.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=63 time=54.7 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=63 time=43.6 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=63 time=36.7 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=63 time=28.6 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=63 time=21.4 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 45ms
rtt min/avg/max/mdev = 21.420/37.035/54.784/11.606 ms, pipe 5, ipg/ewma 11.481/45.089 ms
dc1-leaf-101#
```

</details>

<details>
  <summary>проверки leaf-102</summary>
  
```
dc1-leaf-102#show bfd peers
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.1.1.2  2301327172  2818399692        Ethernet1(14)  normal   05/28/24 13:33 
10.1.2.2   314983775  3213322091        Ethernet2(15)  normal   05/28/24 12:46 

         LastDown            LastDiag    State
-------------------- ------------------- -----
   05/28/24 13:33       No Diagnostic       Up
               NA       No Diagnostic       Up
```
```
dc1-leaf-102#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.101.0, local AS number 65102
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-spine-1 ###      10.1.1.2 4 65100           1195      1209    0    0 00:02:53 Estab   3      3
  ### dc1-spine-2 ###      10.1.2.2 4 65100           1180      1179    0    0 00:49:52 Estab   3      3
```
```
dc1-leaf-102#show ip bgp
BGP routing table information for VRF default
Router identifier 10.0.101.0, local AS number 65102
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.1.0/32            10.1.1.2              0       -          100     0       65100 i
 * >      10.0.2.0/32            10.1.2.2              0       -          100     0       65100 i
 * >Ec    10.0.101.0/32          10.1.1.2              0       -          100     0       65100 65101 i
 *  ec    10.0.101.0/32          10.1.2.2              0       -          100     0       65100 65101 i
 * >      10.0.102.0/32          -                     -       -          -       0       i
 * >Ec    10.0.103.0/32          10.1.2.2              0       -          100     0       65100 65103 i
 *  ec    10.0.103.0/32          10.1.1.2              0       -          100     0       65100 65103 i
```
```
dc1-leaf-102#show ip route
Gateway of last resort is not set

 B E      10.0.1.0/32 [20/0] via 10.1.1.2, Ethernet1
 B E      10.0.2.0/32 [20/0] via 10.1.2.2, Ethernet2
 B E      10.0.101.0/32 [20/0] via 10.1.1.2, Ethernet1
                               via 10.1.2.2, Ethernet2
 C        10.0.102.0/32 is directly connected, Loopback0
 B E      10.0.103.0/32 [20/0] via 10.1.1.2, Ethernet1
                               via 10.1.2.2, Ethernet2
 C        10.1.1.2/31 is directly connected, Ethernet1
 C        10.1.2.2/31 is directly connected, Ethernet2
```

_Ping leaf-101, leaf-103_
```
dc1-leaf-102#ping 10.0.101.0 source loopback 0
PING 10.0.101.0 (10.0.101.0) from 10.0.102.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=63 time=46.6 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=63 time=44.4 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=63 time=55.9 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=63 time=52.4 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=63 time=66.0 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 90ms
rtt min/avg/max/mdev = 44.437/53.115/66.078/7.666 ms, pipe 4, ipg/ewma 22.742/50.428 ms

dc1-leaf-102#ping 10.0.103.0 source loopback 0
PING 10.0.103.0 (10.0.103.0) from 10.0.102.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=63 time=49.5 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=63 time=45.3 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=63 time=43.4 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=63 time=43.8 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=63 time=42.6 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 46ms
rtt min/avg/max/mdev = 42.650/44.954/49.527/2.461 ms, pipe 5, ipg/ewma 11.685/47.110 ms
dc1-leaf-102#
```

</details>

<details>
  <summary>проверки leaf-103</summary>
  
```
dc1-leaf-103#show bfd peers
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.1.1.4   156588905  1394568715        Ethernet1(14)  normal   05/28/24 13:34 
10.1.2.4  2075013061  1750786361        Ethernet2(15)  normal   05/28/24 13:34 

         LastDown            LastDiag    State
-------------------- ------------------- -----
   05/28/24 13:34       No Diagnostic       Up
   05/28/24 13:34       No Diagnostic       Up
```
```
dc1-leaf-103#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.101.0, local AS number 65103
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-spine-1 ###      10.1.1.4 4 65100           1166      1180    0    0 00:01:41 Estab   3      3
  ### dc1-spine-2 ###      10.1.2.4 4 65100           1146      1156    0    0 00:01:42 Estab   3      3
```
```
dc1-leaf-103#show ip bgp
BGP routing table information for VRF default
Router identifier 10.0.101.0, local AS number 65103
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.1.0/32            10.1.1.4              0       -          100     0       65100 i
 * >      10.0.2.0/32            10.1.2.4              0       -          100     0       65100 i
 * >Ec    10.0.101.0/32          10.1.2.4              0       -          100     0       65100 65101 i
 *  ec    10.0.101.0/32          10.1.1.4              0       -          100     0       65100 65101 i
 * >Ec    10.0.102.0/32          10.1.1.4              0       -          100     0       65100 65102 i
 *  ec    10.0.102.0/32          10.1.2.4              0       -          100     0       65100 65102 i
 * >      10.0.103.0/32          -                     -       -          -       0       i
```
```
dc1-leaf-103#show ip route
Gateway of last resort is not set

 B E      10.0.1.0/32 [20/0] via 10.1.1.4, Ethernet1
 B E      10.0.2.0/32 [20/0] via 10.1.2.4, Ethernet2
 B E      10.0.101.0/32 [20/0] via 10.1.1.4, Ethernet1
                               via 10.1.2.4, Ethernet2
 B E      10.0.102.0/32 [20/0] via 10.1.1.4, Ethernet1
                               via 10.1.2.4, Ethernet2
 C        10.0.103.0/32 is directly connected, Loopback0
 C        10.1.1.4/31 is directly connected, Ethernet1
 C        10.1.2.4/31 is directly connected, Ethernet2
```

_Ping leaf-101, leaf-102_
```
dc1-leaf-103#ping 10.0.101.0 source loopback 0
PING 10.0.101.0 (10.0.101.0) from 10.0.103.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=63 time=30.3 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=63 time=34.3 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=63 time=40.4 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=63 time=30.0 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=63 time=28.2 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 96ms
rtt min/avg/max/mdev = 28.278/32.686/40.419/4.354 ms, pipe 3, ipg/ewma 24.083/31.332 ms
dc1-leaf-103#ping 10.0.102.0 source loopback 0 
PING 10.0.102.0 (10.0.102.0) from 10.0.103.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=63 time=35.1 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=63 time=41.0 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=63 time=35.1 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=63 time=45.5 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=63 time=28.9 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 71ms
rtt min/avg/max/mdev = 28.952/37.179/45.570/5.684 ms, pipe 4, ipg/ewma 17.934/36.017 ms
dc1-leaf-103#
```
</details>

### Итоговые конфигурации оборудования
- [dc1-spine-1](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab3/config/dc1-spine-1.txt)
- [dc1-spine-2](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab3/config/dc1-spine-2.txt)
- [dc1-leaf-101](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab3/config/dc1-leaf-101.txt)
- [dc1-leaf-102](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab3/config/dc1-leaf-102.txt)
- [dc1-leaf-103](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab3/config/dc1-leaf-103.txt)
---

[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)


