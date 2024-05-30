# Домашнее задание №4. Underlay.eBGP (IPv6)
[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)
## Задачи
- Настроить eBGP для Underlay сети

## Решение
### Распределение адресного пространства

Нумерация коммутаторов
- spine-X, leaf-1YY

Блок IP-адресов 10.0.0.0/14 для DC1
- блок 10.**0**.0.0/16 - loopback \
  _третий октет - коммутатор, четвертый - номер loopback_
  - loopback**0** spine - 10.1.X.**0**/32
  - loopback**0** leaf - 10.1.1YY.**0**/32
- блок 10.**1**.0.0/16 - резерв 
- блок 10.**2**.0.0/16 - сервисы 
- блок 10.**3**.0.0/16 - резерв 

Блок IPv6-адресов для DC1
- fe80::0:0/64 - транспорт, IPv6 LLA \
  _седьмой хекстет - коммутатор, восьмой хекстет - номер порта_
  - fe80::spine-**X**:<порт>/64
  - fe80::leaf-**1YY**:<порт>/64

### Описание решения
В решении используется протокол маршрутизации eBGP со следующими параметрами:
- все spine размещены в одной AS 65100
- каждый leaf размещен в свой AS: leaf-1YY в AS 651YY
- на spine используются динамические peer-group с фильтром по номеру AS
- на leaf используются статические peer-group
- для установления BGP-соседства используются IPv6 link-local адреса
- IPv6-адреса используются в качестве next-hop для IPv4 сетей
- настроены keepalive-интервал 3 сек, hold time 9 сек.
- настроен maximum-paths равным 8 (4 вероятно хватит, но указал с запасом)
- настроен BGP routing updates интервал равным 0  (neighbor out-delay, установлен в 0 по умолчанию)
- настроена administrative distance равна 20 (по рекомендации Arista из предоставленной ссылке, возможно из-за iBGP между leaf в паре)
- отключена автоматическая активация BGP AFI/SFI ipv4 unicast
- включен режим multi-agent model (поддежка redistribute в BGP AFI/SFI ipv4 unicast)
- включена аутентификация BGP-соседа
- настроено взаимодействие с протоколом bfd для улучшения сходимости сети
- таймеры bfd выбраны такие, чтобы сессии в EVE-NG флапали реже
- размер кэша NDP на интерфейсе равен 100 записям (наверное можно поставить и меньше, раз точка-точка)
- время жизни записи в кэша NDP (тайм-аут) установлено 1200 секунд (20 минут)

### Cхема сети
![Изображение](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab4-ipv6/scheme/lab4_ipv6_scheme.PNG "Схема стенд")

### Таблица IP-адресации
|Оборудование	|Интерфейс	|IP-адрес	|Назначение|
|:-|:-|:-|:-|
|dc1-spine-1	|Loopback0	|10.0.1.0/32	|-|
|dc1-spine-1	|Eth1	|fe80::1:1/64	|sp1-le101|
|dc1-spine-1	|Eth2	|fe80::1:2/64	|sp1-le102|
|dc1-spine-1	|Eth3	|fe80::1:3/64	|sp1-le103|
|dc1-spine-2	|Loopback0	|10.0.2.0/32 |-|	
|dc1-spine-2	|Eth1	|fe80::2:1/64	|sp2-le101|
|dc1-spine-2	|Eth2	|fe80::2:2/64	|sp2-le102|
|dc1-spine-2	|Eth3	|fe80::2:3/64	|sp2-le103|
|dc1-leaf-101	|Loopback0	|10.0.101.0/32 |-|
|dc1-leaf-101	|Eth1	|fe80::101:1/64	|sp1-le101|
|dc1-leaf-101	|Eth2	|fe80::101:2/64	|sp2-le101|
|dc1-leaf-102	|Loopback0	|10.0.102.0/32 |-|	
|dc1-leaf-102	|Eth1	|fe80::102:1/64	|sp1-le102|
|dc1-leaf-102	|Eth2	|fe80::102:2/64	|sp2-le102|	
|dc1-leaf-103	|Loopback0	|10.0.103.0/32 |-|	
|dc1-leaf-103	|Eth1	|fe80::103:1/64	|sp1-le103|
|dc1-leaf-103	|Eth2	|fe80::103:2/64	|sp2-le103|

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
   ipv6 nd cache expire 1200
   ipv6 nd cache dynamic capacity 100
   bfd interval 350 min-rx 350 multiplier 3
   ipv6 enable
   ipv6 address fe80::1:1/64 link-local
!
interface Ethernet2
   description ### sp1-le102 ###
   ipv6 nd cache expire 1200
   ipv6 nd cache dynamic capacity 100
   bfd interval 350 min-rx 350 multiplier 3
   ipv6 enable
   ipv6 address fe80::1:2/64 link-local
!
interface Ethernet3
   description ### sp1-le103 ###
   ipv6 nd cache expire 1200
   ipv6 nd cache dynamic capacity 100
   bfd interval 350 min-rx 350 multiplier 3
   ipv6 enable
   ipv6 address fe80::1:3/64 link-local
!
interface Loopback0
   ip address 10.0.1.0/32
!
ip routing ipv6 interfaces 
!
ipv6 unicast-routing
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
router bgp 65100
   router-id 10.0.1.0
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-LEAF peer group
   neighbor DC1-LEAF bfd
   neighbor DC1-LEAF timers 3 9
   neighbor DC1-LEAF password 7 IS09sfEdsucPgvWfPXx0cQ==
   neighbor interface Et1-3 peer-group DC1-LEAF peer-filter PF-DC1-LEAF
   !
   address-family ipv4
      neighbor DC1-LEAF activate
      neighbor DC1-LEAF next-hop address-family ipv6 originate
      redistribute connected route-map RM-CONNECTED-TO-BGP
```
- spine-2
```
service routing protocols model multi-agent
!
interface Ethernet1
   description ### sp2-le101 ###
   ipv6 nd cache expire 1200
   ipv6 nd cache dynamic capacity 100
   bfd interval 350 min-rx 350 multiplier 3
   ipv6 enable
   ipv6 address fe80::2:1/64 link-local
!
interface Ethernet2
   description ### sp2-le102 ###
   ipv6 nd cache expire 1200
   ipv6 nd cache dynamic capacity 100
   bfd interval 350 min-rx 350 multiplier 3
   ipv6 enable
   ipv6 address fe80::2:2/64 link-local
!
interface Ethernet3
   description ### sp2-le103 ###
   ipv6 nd cache expire 1200
   ipv6 nd cache dynamic capacity 100
   bfd interval 350 min-rx 350 multiplier 3
   ipv6 enable
   ipv6 address fe80::2:3/64 link-local
!
interface Loopback0
   ip address 10.0.2.0/32
!
ip routing ipv6 interfaces 
!
ipv6 unicast-routing
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
router bgp 65100
   router-id 10.0.2.0
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-LEAF peer group
   neighbor DC1-LEAF bfd
   neighbor DC1-LEAF timers 3 9
   neighbor DC1-LEAF password 7 IS09sfEdsucPgvWfPXx0cQ==
   neighbor interface Et1-3 peer-group DC1-LEAF peer-filter PF-DC1-LEAF
   !
   address-family ipv4
      neighbor DC1-LEAF activate
      neighbor DC1-LEAF next-hop address-family ipv6 originate
      redistribute connected route-map RM-CONNECTED-TO-BGP
```
- leaf-101
```
service routing protocols model multi-agent
!
interface Ethernet1
   description ### sp1-le101 ###
   ipv6 nd cache expire 1200
   ipv6 nd cache dynamic capacity 100
   bfd interval 350 min-rx 350 multiplier 3
   ipv6 enable
   ipv6 address fe80::101:1/64 link-local
!
interface Ethernet2
   description ### sp2-le101 ###
   ipv6 nd cache expire 1200
   ipv6 nd cache dynamic capacity 100
   bfd interval 350 min-rx 350 multiplier 3
   ipv6 enable
   ipv6 address fe80::101:2/64 link-local
!
interface Loopback0
   ip address 10.0.101.0/32
!
ip routing ipv6 interfaces 
!
ipv6 unicast-routing
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
   neighbor DC1-SPINE bfd
   neighbor DC1-SPINE timers 3 9
   neighbor DC1-SPINE password 7 txq0MZ/aCqwJ+sp2WtntdQ==
   neighbor interface Et1-2 peer-group DC1-SPINE remote-as 65100
   !
   address-family ipv4
      neighbor DC1-SPINE activate
      neighbor DC1-SPINE next-hop address-family ipv6 originate
      redistribute connected route-map RM-CONNECTED-TO-BGP
```
- leaf-102
```
service routing protocols model multi-agent
!
interface Ethernet1
   description ### sp1-le102 ###
   ipv6 nd cache expire 1200
   ipv6 nd cache dynamic capacity 100
   bfd interval 350 min-rx 350 multiplier 3
   ipv6 enable
   ipv6 address fe80::102:1/64 link-local
!
interface Ethernet2
   description ### sp2-le102 ###
   ipv6 nd cache expire 1200
   ipv6 nd cache dynamic capacity 100
   bfd interval 350 min-rx 350 multiplier 3
   ipv6 enable
   ipv6 address fe80::102:2/64 link-local
!
interface Loopback0
   ip address 10.0.102.0/32
!
ip routing ipv6 interfaces 
!
ipv6 unicast-routing
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
   neighbor DC1-SPINE bfd
   neighbor DC1-SPINE timers 3 9
   neighbor DC1-SPINE password 7 txq0MZ/aCqwJ+sp2WtntdQ==
   neighbor interface Et1-2 peer-group DC1-SPINE remote-as 65100
   !
   address-family ipv4
      neighbor DC1-SPINE activate
      neighbor DC1-SPINE next-hop address-family ipv6 originate
      redistribute connected route-map RM-CONNECTED-TO-BGP
```
- leaf-103
```
service routing protocols model multi-agent
!
interface Ethernet1
   description ### sp1-le103 ###
   ipv6 nd cache expire 1200
   ipv6 nd cache dynamic capacity 100
   bfd interval 350 min-rx 350 multiplier 3
   ipv6 enable
   ipv6 address fe80::103:1/64 link-local
!
interface Ethernet2
   description ### sp2-le103 ###
   ipv6 nd cache expire 1200
   ipv6 nd cache dynamic capacity 100
   bfd interval 350 min-rx 350 multiplier 3
   ipv6 enable
   ipv6 address fe80::103:2/64 link-local
!
interface Loopback0
   ip address 10.0.103.0/32
!
ip routing ipv6 interfaces 
!
ipv6 unicast-routing
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
   neighbor DC1-SPINE bfd
   neighbor DC1-SPINE timers 3 9
   neighbor DC1-SPINE password 7 txq0MZ/aCqwJ+sp2WtntdQ==
   neighbor interface Et1-2 peer-group DC1-SPINE remote-as 65100
   !
   address-family ipv4
      neighbor DC1-SPINE activate
      neighbor DC1-SPINE next-hop address-family ipv6 originate
      redistribute connected route-map RM-CONNECTED-TO-BGP
```
</details>

### Проверка взаимодействия

<details>
  <summary>проверки spine-1</summary>
  
```
dc1-spine-1#show bfd peer
VRF name: default
-----------------
DstAddr         MyDisc   YourDisc  Interface/Transport    Type          LastUp 
----------- ---------- ----------- -------------------- ------- ---------------
fe80::101:1 2587942250 1132603620        Ethernet1(14)  normal  05/30/24 07:06 
fe80::102:1 2532286078  155394943        Ethernet2(15)  normal  05/30/24 07:21 
fe80::103:1  597797150  264912873        Ethernet3(16)  normal  05/30/24 07:41 

         LastDown            LastDiag    State
-------------------- ------------------- -----
   05/30/24 07:06       No Diagnostic       Up
   05/30/24 07:21       No Diagnostic       Up
   05/30/24 07:41       No Diagnostic       Up
```
```
dc1-spine-1#show ipv6 interface | i line|link-l
Ethernet1 is up, line protocol is up (connected)
  IPv6 is enabled, link-local is fe80::1:1/64
Ethernet2 is up, line protocol is up (connected)
  IPv6 is enabled, link-local is fe80::1:2/64
Ethernet3 is up, line protocol is up (connected)
  IPv6 is enabled, link-local is fe80::1:3/64
```
```
dc1-spine-1#show ipv6 neighbors
IPv6 Address                                  Age Hardware Addr    State Interface
fe80::101:1                               0:00:02 5000.0072.8b31   REACH Et1
fe80::102:1                               0:00:00 5000.00d5.5dc0   REACH Et2
fe80::103:1                               0:00:01 5000.0003.3766   REACH Et3
```
```
dc1-spine-1#show ipv6 nd ra neighbors
VRF: default
   Interface       IPv6 Address    Last RA Received
--------------- ------------------ ----------------
   Ethernet1       fe80::101:1       0:00:04 ago   
   Ethernet2       fe80::102:1       0:00:34 ago   
   Ethernet3       fe80::103:1       0:01:45 ago  
```
```
dc1-spine-1#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.1.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Neighbor        V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fe80::101:1%Et1 4 65101           3274      3273    0    0 00:57:44 Estab   1      1
  fe80::102:1%Et2 4 65102           3260      3269    0    0 00:43:13 Estab   1      1
  fe80::103:1%Et3 4 65103           3281      3292    0    0 00:22:57 Estab   1      1
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
 * >      10.0.101.0/32          fe80::101:1%Et1       0       -          100     0       65101 i
 * >      10.0.102.0/32          fe80::102:1%Et2       0       -          100     0       65102 i
 * >      10.0.103.0/32          fe80::103:1%Et3       0       -          100     0       65103 i
```
```
dc1-spine-1#show ip route

Gateway of last resort is not set

 C        10.0.1.0/32 [0/0]
           via Loopback0, directly connected
 B E      10.0.101.0/32 [20/0]
           via fe80::101:1, Ethernet1
 B E      10.0.102.0/32 [20/0]
           via fe80::102:1, Ethernet2
 B E      10.0.103.0/32 [20/0]
           via fe80::103:1, Ethernet3 
```

_Ping spine-2, leaf-101, leaf-102, leaf-103_ \
_spine-2 не доступен со spine-1 т.к. они в одной AS и BGP update (NLRI) не принимается из-за наличия локальной AS в AS_PATH_
```
dc1-spine-1#ping 10.0.2.0 source loopback 0
PING 10.0.2.0 (10.0.2.0) from 10.0.1.0 : 72(100) bytes of data.
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable

--- 10.0.2.0 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 43ms

dc1-spine-1#ping 10.0.101.0 source loopback 0
PING 10.0.101.0 (10.0.101.0) from 10.0.1.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=65 time=101 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=65 time=98.0 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=65 time=97.3 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=65 time=96.7 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=65 time=95.8 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 46ms
rtt min/avg/max/mdev = 95.808/97.974/101.959/2.146 ms, pipe 5, ipg/ewma 11.629/99.848 ms

dc1-spine-1#ping 10.0.102.0 source loopback 0
PING 10.0.102.0 (10.0.102.0) from 10.0.1.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=65 time=61.0 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=65 time=53.2 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=65 time=52.0 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=65 time=47.3 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=65 time=50.0 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 46ms
rtt min/avg/max/mdev = 47.380/52.770/61.069/4.612 ms, pipe 5, ipg/ewma 11.622/56.680 ms

dc1-spine-1#ping 10.0.103.0 source loopback 0
PING 10.0.103.0 (10.0.103.0) from 10.0.1.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=65 time=86.0 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=65 time=79.6 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=65 time=84.2 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=65 time=79.6 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=65 time=75.4 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 51ms
rtt min/avg/max/mdev = 75.497/81.017/86.072/3.755 ms, pipe 5, ipg/ewma 12.765/83.334 ms
```

</details>


<details>
  <summary>проверки spine-2</summary>
  
```
dc1-spine-2#show bfd peer
VRF name: default
-----------------
DstAddr         MyDisc   YourDisc  Interface/Transport    Type          LastUp 
----------- ---------- ----------- -------------------- ------- ---------------
fe80::101:2 2664828502 2109024286        Ethernet1(14)  normal  05/30/24 07:14 
fe80::102:2 3642254970 3941955158        Ethernet2(15)  normal  05/30/24 07:21 
fe80::103:2 2590327866 4061826928        Ethernet3(16)  normal  05/30/24 07:41 

         LastDown            LastDiag    State
-------------------- ------------------- -----
   05/30/24 07:14       No Diagnostic       Up
   05/30/24 07:20       No Diagnostic       Up
   05/30/24 07:41       No Diagnostic       Up
```
```
dc1-spine-2#show ipv6 interface | i line|link-l
Ethernet1 is up, line protocol is up (connected)
  IPv6 is enabled, link-local is fe80::2:1/64
Ethernet2 is up, line protocol is up (connected)
  IPv6 is enabled, link-local is fe80::2:2/64
Ethernet3 is up, line protocol is up (connected)
  IPv6 is enabled, link-local is fe80::2:3/64
```
```
dc1-spine-2#show ipv6 neighbors
IPv6 Address                                  Age Hardware Addr    State Interface
fe80::101:2                               0:00:00 5000.0072.8b31   REACH Et1
fe80::102:2                               0:00:00 5000.00d5.5dc0   REACH Et2
fe80::103:2                               0:00:00 5000.0003.3766   REACH Et3
```
```
dc1-spine-2#show ipv6 nd ra neighbors
VRF: default
   Interface       IPv6 Address    Last RA Received
--------------- ------------------ ----------------
   Ethernet1       fe80::101:2       0:00:11 ago   
   Ethernet2       fe80::102:2       0:00:17 ago   
   Ethernet3       fe80::103:2       0:01:40 ago  
```
```
dc1-spine-2#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.2.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Neighbor        V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fe80::101:2%Et1 4 65101           8941      8952    0    0 00:50:08 Estab   1      1
  fe80::102:2%Et2 4 65102           8886      8894    0    0 00:43:12 Estab   1      1
  fe80::103:2%Et3 4 65103           8997      8998    0    0 00:22:56 Estab   1      1
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
 * >      10.0.101.0/32          fe80::101:2%Et1       0       -          100     0       65101 i
 * >      10.0.102.0/32          fe80::102:2%Et2       0       -          100     0       65102 i
 * >      10.0.103.0/32          fe80::103:2%Et3       0       -          100     0       65103 i
```
```
dc1-spine-2#show ip route

Gateway of last resort is not set

 C        10.0.2.0/32 [0/0]
           via Loopback0, directly connected
 B E      10.0.101.0/32 [20/0]
           via fe80::101:2, Ethernet1
 B E      10.0.102.0/32 [20/0]
           via fe80::102:2, Ethernet2
 B E      10.0.103.0/32 [20/0]
           via fe80::103:2, Ethernet3
```

_Ping spine-1, leaf-101, leaf-102, leaf-103_ \ 
_spine-1 не доступен со spine-2 т.к. они в одной AS и BGP update (NLRI) не принимается из-за наличия локальной AS в AS_PATH_
```
dc1-spine-2#ping 10.0.1.0 source loopback 0 
PING 10.0.1.0 (10.0.1.0) from 10.0.2.0 : 72(100) bytes of data.
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable

--- 10.0.1.0 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 49ms

dc1-spine-2#ping 10.0.101.0 source loopback 0
PING 10.0.101.0 (10.0.101.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=65 time=70.0 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=65 time=55.0 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=65 time=57.6 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=65 time=50.8 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=65 time=48.9 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 51ms
rtt min/avg/max/mdev = 48.942/56.517/70.038/7.417 ms, pipe 5, ipg/ewma 12.845/62.867 ms
dc1-spine-2#ping 10.0.102.0 source loopback 0
PING 10.0.102.0 (10.0.102.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=65 time=56.3 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=65 time=51.0 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=65 time=46.5 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=65 time=52.3 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=65 time=51.9 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 44ms
rtt min/avg/max/mdev = 46.537/51.641/56.360/3.149 ms, pipe 5, ipg/ewma 11.191/53.982 ms

dc1-spine-2#ping 10.0.103.0 source loopback 0
May 30 08:06:03 dc1-spine-2 Bgp: %BGP-3-NOTIFICATION: sent to neighbor fe80::102:2%Et2 (VRF default AS 65102) 6/10 (Cease/BFD down <Hard Reset>) 0 bytes
PING 10.0.103.0 (10.0.103.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=65 time=117 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=65 time=113 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=65 time=136 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=65 time=131 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=65 time=133 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 58ms
rtt min/avg/max/mdev = 113.969/126.726/136.655/9.109 ms, pipe 5, ipg/ewma 14.590/122.746 ms
```

</details>

<details>
  <summary>проверки leaf-101</summary>
  
```
dc1-leaf-101#show bfd peer
VRF name: default
-----------------
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp 
---------- ----------- ----------- -------------------- ------- ---------------
fe80::1:1  1132603620  2587942250        Ethernet1(14)  normal  05/30/24 07:06 
fe80::2:1  2109024286  2664828502        Ethernet2(15)  normal  05/30/24 07:14 

         LastDown            LastDiag    State
-------------------- ------------------- -----
   05/30/24 07:06       No Diagnostic       Up
   05/30/24 07:14       No Diagnostic       Up
```
```
dc1-leaf-101#show ipv6 interface | i line|link-l
Ethernet1 is up, line protocol is up (connected)
  IPv6 is enabled, link-local is fe80::101:1/64
Ethernet2 is up, line protocol is up (connected)
  IPv6 is enabled, link-local is fe80::101:2/64
```
```
dc1-leaf-101#show ipv6 neighbors
IPv6 Address                                  Age Hardware Addr    State Interface
fe80::1:1                                 0:00:01 5000.00d7.ee0b   REACH Et1
fe80::2:1                                 0:00:00 5000.00cb.38c2   REACH Et2
```
```
dc1-leaf-101#show ipv6 nd ra neighbors
VRF: default
   Interface       IPv6 Address    Last RA Received
--------------- ------------------ ----------------
   Ethernet1        fe80::1:1        0:02:11 ago   
   Ethernet2        fe80::2:1        0:02:50 ago 
```
```
dc1-leaf-101#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.101.0, local AS number 65101
Neighbor Status Codes: m - Under maintenance
  Neighbor      V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fe80::1:1%Et1 4 65100           3264      3284    0    0 00:57:44 Estab   3      3
  fe80::2:1%Et2 4 65100           8926      8963    0    0 00:50:07 Estab   3      3
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
 * >      10.0.1.0/32            fe80::1:1%Et1         0       -          100     0       65100 i
 * >      10.0.2.0/32            fe80::2:1%Et2         0       -          100     0       65100 i
 * >      10.0.101.0/32          -                     -       -          -       0       i
 * >Ec    10.0.102.0/32          fe80::2:1%Et2         0       -          100     0       65100 65102 i
 *  ec    10.0.102.0/32          fe80::1:1%Et1         0       -          100     0       65100 65102 i
 * >Ec    10.0.103.0/32          fe80::2:1%Et2         0       -          100     0       65100 65103 i
 *  ec    10.0.103.0/32          fe80::1:1%Et1         0       -          100     0       65100 65103 i
```
```
dc1-leaf-101#show ip route

Gateway of last resort is not set

 B E      10.0.1.0/32 [20/0]
           via fe80::1:1, Ethernet1
 B E      10.0.2.0/32 [20/0]
           via fe80::2:1, Ethernet2
 C        10.0.101.0/32 [0/0]
           via Loopback0, directly connected
 B E      10.0.102.0/32 [20/0]
           via fe80::1:1, Ethernet1
           via fe80::2:1, Ethernet2
 B E      10.0.103.0/32 [20/0]
           via fe80::1:1, Ethernet1
           via fe80::2:1, Ethernet2
```

_Ping leaf-102, leaf-103_
```
dc1-leaf-101#ping 10.0.102.0 source loopback 0
PING 10.0.102.0 (10.0.102.0) from 10.0.101.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=64 time=112 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=64 time=105 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=64 time=102 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=64 time=102 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=64 time=100 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 51ms
rtt min/avg/max/mdev = 100.591/104.745/112.216/4.074 ms, pipe 5, ipg/ewma 12.960/108.246 ms

dc1-leaf-101#ping 10.0.103.0 source loopback 0
PING 10.0.103.0 (10.0.103.0) from 10.0.101.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=64 time=149 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=64 time=175 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=64 time=177 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=64 time=171 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=64 time=163 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 49ms
rtt min/avg/max/mdev = 149.976/167.695/177.852/10.141 ms, pipe 5, ipg/ewma 12.369/158.841 ms
```

</details>

<details>
  <summary>проверки leaf-102</summary>
  
```
dc1-leaf-102#show bfd peer
VRF name: default
-----------------
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp 
---------- ----------- ----------- -------------------- ------- ---------------
fe80::1:2   155394943  2532286078        Ethernet1(14)  normal  05/30/24 07:21 
fe80::2:2  3941955158  3642254970        Ethernet2(15)  normal  05/30/24 07:21 

         LastDown            LastDiag    State
-------------------- ------------------- -----
   05/30/24 07:21       No Diagnostic       Up
   05/30/24 07:21       No Diagnostic       Up
```
```
dc1-leaf-102#show ipv6 interface | i line|link-l
Ethernet1 is up, line protocol is up (connected)
  IPv6 is enabled, link-local is fe80::102:1/64
Ethernet2 is up, line protocol is up (connected)
  IPv6 is enabled, link-local is fe80::102:2/64
```
```  
dc1-leaf-102#show ipv6 neighbors
IPv6 Address                                  Age Hardware Addr    State Interface
fe80::1:2                                 0:00:01 5000.00d7.ee0b   REACH Et1
fe80::2:2                                 0:00:02 5000.00cb.38c2   REACH Et2
```
```
dc1-leaf-102#show ipv6 nd ra neighbors
VRF: default
   Interface       IPv6 Address    Last RA Received
--------------- ------------------ ----------------
   Ethernet1        fe80::1:2        0:02:28 ago   
   Ethernet2        fe80::2:2        0:01:44 ago   
```
```
dc1-leaf-102#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.102.0, local AS number 65102
Neighbor Status Codes: m - Under maintenance
  Neighbor      V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fe80::1:2%Et1 4 65100           8887      8892    0    0 00:43:12 Estab   3      3
  fe80::2:2%Et2 4 65100           8876      8901    0    0 00:43:12 Estab   3      3
```
```
dc1-leaf-102#show ip bgp
BGP routing table information for VRF default
Router identifier 10.0.102.0, local AS number 65102
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.1.0/32            fe80::1:2%Et1         0       -          100     0       65100 i
 * >      10.0.2.0/32            fe80::2:2%Et2         0       -          100     0       65100 i
 * >Ec    10.0.101.0/32          fe80::2:2%Et2         0       -          100     0       65100 65101 i
 *  ec    10.0.101.0/32          fe80::1:2%Et1         0       -          100     0       65100 65101 i
 * >      10.0.102.0/32          -                     -       -          -       0       i
 * >Ec    10.0.103.0/32          fe80::2:2%Et2         0       -          100     0       65100 65103 i
 *  ec    10.0.103.0/32          fe80::1:2%Et1         0       -          100     0       65100 65103 i
```
```
dc1-leaf-102#show ip route

Gateway of last resort is not set

 B E      10.0.1.0/32 [20/0]
           via fe80::1:2, Ethernet1
 B E      10.0.2.0/32 [20/0]
           via fe80::2:2, Ethernet2
 B E      10.0.101.0/32 [20/0]
           via fe80::1:2, Ethernet1
           via fe80::2:2, Ethernet2
 C        10.0.102.0/32 [0/0]
           via Loopback0, directly connected
 B E      10.0.103.0/32 [20/0]
           via fe80::1:2, Ethernet1
           via fe80::2:2, Ethernet2
```

_Ping leaf-101, leaf-103_
```
dc1-leaf-102#ping 10.0.101.0 source loopback 0
PING 10.0.101.0 (10.0.101.0) from 10.0.102.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=64 time=96.0 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=64 time=86.8 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=64 time=85.4 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=64 time=81.0 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=64 time=86.2 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 49ms
rtt min/avg/max/mdev = 81.094/87.124/96.005/4.884 ms, pipe 5, ipg/ewma 12.327/91.377 ms

dc1-leaf-102ping 10.0.103.0 source loopback 0
PING 10.0.103.0 (10.0.103.0) from 10.0.102.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=64 time=134 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=64 time=131 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=64 time=135 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=64 time=141 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=64 time=162 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 51ms
rtt min/avg/max/mdev = 131.621/141.385/162.966/11.314 ms, pipe 5, ipg/ewma 12.924/138.761 ms
```

</details>

<details>
  <summary>проверки leaf-103</summary>
  
```
dc1-leaf-103#show bfd peer
VRF name: default
-----------------
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp 
---------- ----------- ----------- -------------------- ------- ---------------
fe80::1:3   264912873   597797150        Ethernet1(14)  normal  05/30/24 07:41 
fe80::2:3  4061826928  2590327866        Ethernet2(15)  normal  05/30/24 07:41 

         LastDown            LastDiag    State
-------------------- ------------------- -----
   05/30/24 07:41       No Diagnostic       Up
   05/30/24 07:41       No Diagnostic       Up
```
```
dc1-leaf-103#show ipv6 interface | i line|link-l
Ethernet1 is up, line protocol is up (connected)
  IPv6 is enabled, link-local is fe80::103:1/64
Ethernet2 is up, line protocol is up (connected)
  IPv6 is enabled, link-local is fe80::103:2/64
```
```
dc1-leaf-103#show ipv6 neighbors
IPv6 Address                                  Age Hardware Addr    State Interface
fe80::1:3                                 0:00:01 5000.00d7.ee0b   REACH Et1
fe80::2:3                                 0:00:00 5000.00cb.38c2   REACH Et2
```
```
dc1-leaf-103#show ipv6 nd ra neighbors
VRF: default
   Interface       IPv6 Address    Last RA Received
--------------- ------------------ ----------------
   Ethernet1        fe80::1:3        0:00:13 ago   
   Ethernet2        fe80::2:3        0:01:20 ago
```
```
dc1-leaf-103#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.103.0, local AS number 65103
Neighbor Status Codes: m - Under maintenance
  Neighbor      V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fe80::1:3%Et1 4 65100           8961      9006    0    0 00:22:56 Estab   3      3
  fe80::2:3%Et2 4 65100           8976      9020    0    0 00:22:57 Estab   3      3
```
```
dc1-leaf-103#show ip bgp
BGP routing table information for VRF default
Router identifier 10.0.103.0, local AS number 65103
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.1.0/32            fe80::1:3%Et1         0       -          100     0       65100 i
 * >      10.0.2.0/32            fe80::2:3%Et2         0       -          100     0       65100 i
 * >Ec    10.0.101.0/32          fe80::2:3%Et2         0       -          100     0       65100 65101 i
 *  ec    10.0.101.0/32          fe80::1:3%Et1         0       -          100     0       65100 65101 i
 * >Ec    10.0.102.0/32          fe80::2:3%Et2         0       -          100     0       65100 65102 i
 *  ec    10.0.102.0/32          fe80::1:3%Et1         0       -          100     0       65100 65102 i
 * >      10.0.103.0/32          -                     -       -          -       0       i
```
```
dc1-leaf-103#show ip route

Gateway of last resort is not set

 B E      10.0.1.0/32 [20/0]
           via fe80::1:3, Ethernet1
 B E      10.0.2.0/32 [20/0]
           via fe80::2:3, Ethernet2
 B E      10.0.101.0/32 [20/0]
           via fe80::1:3, Ethernet1
           via fe80::2:3, Ethernet2
 B E      10.0.102.0/32 [20/0]
           via fe80::1:3, Ethernet1
           via fe80::2:3, Ethernet2
 C        10.0.103.0/32 [0/0]
           via Loopback0, directly connected
```

_Ping leaf-101, leaf-102_
```
dc1-leaf-103#ping 10.0.101.0 source loopback 0
PING 10.0.101.0 (10.0.101.0) from 10.0.103.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=64 time=171 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=64 time=165 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=64 time=186 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=64 time=184 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=64 time=180 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 45ms
rtt min/avg/max/mdev = 165.236/177.661/186.957/8.138 ms, pipe 5, ipg/ewma 11.399/174.915 ms

dc1-leaf-103#ping 10.0.102.0 source loopback 0
PING 10.0.102.0 (10.0.102.0) from 10.0.103.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=64 time=121 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=64 time=132 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=64 time=171 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=64 time=169 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=64 time=178 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 45ms
rtt min/avg/max/mdev = 121.651/154.982/178.735/23.107 ms, pipe 5, ipg/ewma 11.347/139.798 ms
```
</details>

### Итоговые конфигурации оборудования
- [dc1-spine-1](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab4-ipv6/config/dc1-spine-1.txt)
- [dc1-spine-2](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab4-ipv6/config/dc1-spine-2.txt)
- [dc1-leaf-101](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab4-ipv6/config/dc1-leaf-101.txt)
- [dc1-leaf-102](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab4-ipv6/config/dc1-leaf-102.txt)
- [dc1-leaf-103](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab4-ipv6/config/dc1-leaf-103.txt)
---

[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)
