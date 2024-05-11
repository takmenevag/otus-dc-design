# Домашнее задание №1. Проектирование адресного пространства
[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)
## Задачи
- Собрать схему CLOS
- Распределить адресное пространство

## Решение
### Распределение адресного пространства
Блок IP-адресов 10.0.0.0/14 для DC1
- spine-X, leaf-10Y
- блок 10.**0**.0.0/16 - loopback \
  _третий октет - коммутатор, четвертый - номер loopback_
  - loopback**0** spine - 10.1.X.**0**/32
  - loopback**0** leaf - 10.1.10Y.**0**/32
- блок 10.**1**.0.0/16 - транспорт \
 _третий октет - spine, четвертый октет сети по /31_
  - transport spine-**X** - 10.10.**X**.<сеть>/31
- блок 10.**2**.0.0/16 - сервисы
- блок 10.**3**.0.0/16 - резерв

### Cхема сети
![Изображение](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab1/scheme/lab1-scheme.png "Схема стенда")

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
<details>
  <summary>Команды для настройки </summary>

- Spine-1
```
hostname dc1-spine-1
!
interface Ethernet1
   description ### sp1-le101 ###
   no switchport
   ip address 10.1.1.0/31
!
interface Ethernet2
   description ### sp1-le102 ###
   no switchport
   ip address 10.1.1.2/31
!
interface Ethernet3
   description ### sp1-le103 ###
   no switchport
   ip address 10.1.1.4/31
!
interface Loopback0
   ip address 10.0.1.0/32
!
ip routing
```
- Spine-2
```
hostname dc1-spine-2
!
interface Ethernet1
   description ### sp2-le101 ###
   no switchport
   ip address 10.1.2.0/31
!
interface Ethernet2
   description ### sp2-le102 ###
   no switchport
   ip address 10.1.2.2/31
!
interface Ethernet3
   description ### sp2-le103 ###
   no switchport
   ip address 10.1.2.4/31
!
interface Loopback0
   ip address 10.0.2.0/32
!
ip routing
```
- Leaf-1
```
hostname dc1-leaf-101
!
interface Ethernet1
   description ### sp1-le101 ###
   no switchport
   ip address 10.1.1.1/31
!
interface Ethernet2
   description ### sp2-le101 ###
   no switchport
   ip address 10.1.2.1/31
!
interface Loopback0
   ip address 10.0.101.0/32
!
ip routing
```
- Leaf-2
```
hostname dc1-leaf-102
!
interface Ethernet1
   description ### sp1-le102 ###
   no switchport
   ip address 10.1.1.3/31
!
interface Ethernet2
   description ### sp2-le102 ###
   no switchport
   ip address 10.1.2.3/31
!
interface Loopback0
   ip address 10.0.102.0/32
!
ip routing
```
- Leaf-3
```
hostname dc1-leaf-103
!
interface Ethernet1
   description ### sp1-le103 ###
   no switchport
   ip address 10.1.1.5/31
!
interface Ethernet2
   description ### sp2-le103 ###
   no switchport
   ip address 10.1.2.5/31
!
interface Loopback0
   ip address 10.0.103.0/32
!
ip routing
```
</details>

### Проверка взаимодействия
<details>
  <summary>от Leaf-1 к Spine-1 и Spine-2</summary>

```
dc1-leaf-101#ping 10.1.1.0
PING 10.1.1.0 (10.1.1.0) 72(100) bytes of data.
80 bytes from 10.1.1.0: icmp_seq=1 ttl=64 time=89.4 ms
80 bytes from 10.1.1.0: icmp_seq=2 ttl=64 time=79.9 ms
80 bytes from 10.1.1.0: icmp_seq=3 ttl=64 time=74.9 ms
80 bytes from 10.1.1.0: icmp_seq=4 ttl=64 time=70.6 ms
80 bytes from 10.1.1.0: icmp_seq=5 ttl=64 time=64.3 ms

--- 10.1.1.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 45ms
rtt min/avg/max/mdev = 64.334/75.865/89.421/8.509 ms, pipe 5, ipg/ewma 11.449/82.056 ms
dc1-leaf-101#ping 10.1.2.0
PING 10.1.2.0 (10.1.2.0) 72(100) bytes of data.
80 bytes from 10.1.2.0: icmp_seq=1 ttl=64 time=66.8 ms
80 bytes from 10.1.2.0: icmp_seq=2 ttl=64 time=61.2 ms
80 bytes from 10.1.2.0: icmp_seq=3 ttl=64 time=58.2 ms
80 bytes from 10.1.2.0: icmp_seq=4 ttl=64 time=52.0 ms
80 bytes from 10.1.2.0: icmp_seq=5 ttl=64 time=45.7 ms

--- 10.1.2.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 42ms
rtt min/avg/max/mdev = 45.702/56.812/66.808/7.322 ms, pipe 5, ipg/ewma 10.554/61.271 ms
dc1-leaf-101#
```
```
dc1-leaf-101#show ip route

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

 C        10.0.101.0/32 is directly connected, Loopback0
 C        10.1.1.0/31 is directly connected, Ethernet1
 C        10.1.2.0/31 is directly connected, Ethernet2
```
```
dc1-leaf-101#show lldp neighbors 
Last table change time   : 0:03:35 ago
Number of table inserts  : 2
Number of table deletes  : 0
Number of table drops    : 0
Number of table age-outs : 0

Port          Neighbor Device ID       Neighbor Port ID    TTL
---------- ------------------------ ---------------------- ---
Et1           dc1-spine-1              Ethernet1           120
Et2           dc1-spine-2              Ethernet1           120
```
</details>

<details>
  <summary>от Leaf-2 к Spine-1 и Spine-2</summary>
 
```
dc1-leaf-102#ping 10.1.1.2
PING 10.1.1.2 (10.1.1.2) 72(100) bytes of data.
80 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=65.8 ms
80 bytes from 10.1.1.2: icmp_seq=2 ttl=64 time=60.9 ms
80 bytes from 10.1.1.2: icmp_seq=3 ttl=64 time=55.2 ms
80 bytes from 10.1.1.2: icmp_seq=4 ttl=64 time=50.1 ms
80 bytes from 10.1.1.2: icmp_seq=5 ttl=64 time=44.7 ms

--- 10.1.1.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 44ms
rtt min/avg/max/mdev = 44.737/55.376/65.834/7.497 ms, pipe 5, ipg/ewma 11.118/60.055 ms
dc1-leaf-102#
dc1-leaf-102#ping 10.1.2.2
PING 10.1.2.2 (10.1.2.2) 72(100) bytes of data.
80 bytes from 10.1.2.2: icmp_seq=1 ttl=64 time=42.9 ms
80 bytes from 10.1.2.2: icmp_seq=2 ttl=64 time=22.3 ms
80 bytes from 10.1.2.2: icmp_seq=3 ttl=64 time=18.1 ms
80 bytes from 10.1.2.2: icmp_seq=4 ttl=64 time=6.75 ms
80 bytes from 10.1.2.2: icmp_seq=5 ttl=64 time=8.08 ms

--- 10.1.2.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 109ms
rtt min/avg/max/mdev = 6.753/19.652/42.928/13.049 ms, pipe 3, ipg/ewma 27.284/30.521 ms
dc1-leaf-102#
```
```
dc1-leaf-102#show ip route

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

 C        10.0.102.0/32 is directly connected, Loopback0
 C        10.1.1.2/31 is directly connected, Ethernet1
 C        10.1.2.2/31 is directly connected, Ethernet2
```
```
dc1-leaf-102#show lldp neighbors
Last table change time   : 0:04:33 ago
Number of table inserts  : 2
Number of table deletes  : 0
Number of table drops    : 0
Number of table age-outs : 0

Port          Neighbor Device ID       Neighbor Port ID    TTL
---------- ------------------------ ---------------------- ---
Et1           dc1-spine-1              Ethernet2           120
Et2           dc1-spine-2              Ethernet2           120
```
</details>

<details>
  <summary>от Leaf-3 к Spine-1 и Spine-2</summary>
  
```
dc1-leaf-103#ping 10.1.1.4
PING 10.1.1.4 (10.1.1.4) 72(100) bytes of data.
80 bytes from 10.1.1.4: icmp_seq=1 ttl=64 time=90.1 ms
80 bytes from 10.1.1.4: icmp_seq=2 ttl=64 time=84.7 ms
80 bytes from 10.1.1.4: icmp_seq=3 ttl=64 time=71.9 ms
80 bytes from 10.1.1.4: icmp_seq=4 ttl=64 time=63.2 ms
80 bytes from 10.1.1.4: icmp_seq=5 ttl=64 time=58.0 ms

--- 10.1.1.4 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 55ms
rtt min/avg/max/mdev = 58.080/73.621/90.111/12.227 ms, pipe 5, ipg/ewma 13.867/80.979 ms
dc1-leaf-103#
dc1-leaf-103#ping 10.1.2.4
PING 10.1.2.4 (10.1.2.4) 72(100) bytes of data.
80 bytes from 10.1.2.4: icmp_seq=1 ttl=64 time=53.7 ms
80 bytes from 10.1.2.4: icmp_seq=2 ttl=64 time=39.4 ms
80 bytes from 10.1.2.4: icmp_seq=3 ttl=64 time=30.6 ms
80 bytes from 10.1.2.4: icmp_seq=4 ttl=64 time=21.9 ms
80 bytes from 10.1.2.4: icmp_seq=5 ttl=64 time=9.87 ms

--- 10.1.2.4 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 90ms
rtt min/avg/max/mdev = 9.879/31.128/53.714/14.925 ms, pipe 4, ipg/ewma 22.746/41.359 ms
```
```
dc1-leaf-103#show ip route

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

 C        10.0.103.0/32 is directly connected, Loopback0
 C        10.1.1.4/31 is directly connected, Ethernet1
 C        10.1.2.4/31 is directly connected, Ethernet2
```
```
dc1-leaf-103#show lldp neighbors 
Last table change time   : 0:05:46 ago
Number of table inserts  : 2
Number of table deletes  : 0
Number of table drops    : 0
Number of table age-outs : 0

Port          Neighbor Device ID       Neighbor Port ID    TTL
---------- ------------------------ ---------------------- ---
Et1           dc1-spine-1              Ethernet3           120
Et2           dc1-spine-2              Ethernet3           120
```
</details>

### Итоговые конфигурации оборудования
- [dc1-spine-1](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab1/config/dc1-spine-1.txt)
- [dc1-spine-2](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab1/config/dc1-spine-2.txt)
- [dc1-leaf-101](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab1/config/dc1-leaf-101.txt)
- [dc1-leaf-102](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab1/config/dc1-leaf-102.txt)
- [dc1-leaf-103](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab1/config/dc1-leaf-103.txt)
---
[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)
