# Домашнее задание №1. Проектирование адресного пространства
[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)
## Задачи
- Собрать схему CLOS
- Распределить адресное пространство

## Решение
### Распределение адресного пространства
- spine-X, leaf-10Y
- блок 10.**1**.0.0/16 - loopback \
  _третий октет - коммутатор, четвертый - номер loopback_
  - loopback**0** spine - 10.1.X.**0**/32
  - loopback**0** leaf - 10.1.10Y.**0**/32
- блок 10.**10**.0.0/16 - transport \
 _третий октет - первая цифра это spine, последняя - leaf,  четвертый октет ниже_ \
_с учетом того, что в 1 POD будет только два spine должно хватить_ \
  - transport spine - 10.10.X0Y.**0**/31
  - transport leaf - 10.10.X0Y.**1**/31
### Cхема сети
![Изображение](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab1/scheme/lab1.PNG "Схема стенда")

### Таблица IP-адресации
|Оборудование	|Интерфейс	|IP-адрес	|Назначение|
|:-|:-|:-|:-|
|dc1-spine-1	|Loopback0	|10.1.1.0/32	|-|
|dc1-spine-1	|Eth1	|10.10.101.0/31	|sp1-le101|
|dc1-spine-1	|Eth2	|10.10.102.0/31	|sp1-le102|
|dc1-spine-1	|Eth3	|10.10.103.0/31	|sp1-le103|
|dc1-spine-2	|Loopback0	|10.1.2.0/32 |-|	
|dc1-spine-2	|Eth1	|10.10.201.0/31	|sp2-le101|
|dc1-spine-2	|Eth2	|10.10.202.0/31	|sp2-le102|
|dc1-spine-2	|Eth3	|10.10.203.0/31	|sp2-le103|
|dc1-leaf-101	|Loopback0	|10.1.101.0/32 |-|
|dc1-leaf-101	|Eth1	|10.10.101.1/31	|sp1-le101|
|dc1-leaf-101	|Eth2	|10.10.201.1/31	|sp2-le101|
|dc1-leaf-102	|Loopback0	|10.1.102.0/32 |-|	
|dc1-leaf-102	|Eth1	|10.10.102.1/31	|sp1-le102|
|dc1-leaf-102	|Eth2	|10.10.202.1/31	|sp2-le102|	
|dc1-leaf-103	|Loopback0	|10.1.103.0/32 |-|	
|dc1-leaf-103	|Eth1	|10.10.103.1/31	|sp1-le103|
|dc1-leaf-103	|Eth2	|10.10.203.1/31	|sp2-le103|

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
   ip address 10.10.101.0/31
!
interface Ethernet2
   description ### sp1-le102 ###
   no switchport
   ip address 10.10.102.0/31
!
interface Ethernet3
   description ### sp1-le103 ###
   no switchport
   ip address 10.10.103.0/31
!
interface Loopback0
   ip address 10.1.1.0/32
!
ip routing
!
end
```
- Spine-2
```
hostname dc1-spine-2
!
interface Ethernet1
   description ### sp2-le101 ###
   no switchport
   ip address 10.10.201.0/31
!
interface Ethernet2
   description ### sp2-le102 ###
   no switchport
   ip address 10.10.202.0/31
!
interface Ethernet3
   description ### sp2-le103 ###
   no switchport
   ip address 10.10.203.0/31
!
interface Loopback0
   ip address 10.1.2.0/32
!
ip routing
!
end
```
- Leaf-1
```
hostname dc1-leaf-101
!
interface Ethernet1
   description ### sp1-le101 ###
   no switchport
   ip address 10.10.101.1/31
!
interface Ethernet2
   description ### sp2-le101 ###
   no switchport
   ip address 10.10.201.1/31
!
interface Loopback0
   ip address 10.1.101.0/32
!
ip routing
!
end
```
- Leaf-2
```
hostname dc1-leaf-102
!
interface Ethernet1
   description ### sp1-le102 ###
   no switchport
   ip address 10.10.102.1/31
!
interface Ethernet2
   description ### sp2-le102 ###
   no switchport
   ip address 10.10.202.1/31
!
interface Loopback0
   ip address 10.1.102.0/32
!
ip routing
!
end
```
- Leaf-3
```
hostname dc1-leaf-103
!
interface Ethernet1
   description ### sp1-le103 ###
   no switchport
   ip address 10.10.103.1/31
!
interface Ethernet2
   description ### sp2-le103 ###
   no switchport
   ip address 10.10.203.1/31
!
interface Loopback0
   ip address 10.1.103.0/32
!
ip routing
!
end
```
</details>

### Проверка взаимодействия
<details>
  <summary>от Leaf-1 к Spine-1 и Spine-2</summary>

```
dc1-leaf-101#ping 10.10.101.0
PING 10.10.101.0 (10.10.101.0) 72(100) bytes of data.
80 bytes from 10.10.101.0: icmp_seq=1 ttl=64 time=36.7 ms
80 bytes from 10.10.101.0: icmp_seq=2 ttl=64 time=30.6 ms
80 bytes from 10.10.101.0: icmp_seq=3 ttl=64 time=26.3 ms
80 bytes from 10.10.101.0: icmp_seq=4 ttl=64 time=8.84 ms
80 bytes from 10.10.101.0: icmp_seq=5 ttl=64 time=9.57 ms

--- 10.10.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 96ms
rtt min/avg/max/mdev = 8.847/22.428/36.730/11.286 ms, pipe 3, ipg/ewma 24.073/28.782 ms
dc1-leaf-101#
dc1-leaf-101#ping 10.10.201.0
PING 10.10.201.0 (10.10.201.0) 72(100) bytes of data.
80 bytes from 10.10.201.0: icmp_seq=1 ttl=64 time=20.8 ms
80 bytes from 10.10.201.0: icmp_seq=2 ttl=64 time=15.1 ms
80 bytes from 10.10.201.0: icmp_seq=3 ttl=64 time=11.6 ms
80 bytes from 10.10.201.0: icmp_seq=4 ttl=64 time=7.78 ms
80 bytes from 10.10.201.0: icmp_seq=5 ttl=64 time=10.8 ms

--- 10.10.201.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 69ms
rtt min/avg/max/mdev = 7.783/13.260/20.817/4.453 ms, pipe 2, ipg/ewma 17.304/16.799 ms
```
```
dc1-leaf-101#sh ip route

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

 C        10.1.101.0/32 is directly connected, Loopback0
 C        10.10.101.0/31 is directly connected, Ethernet1
 C        10.10.201.0/31 is directly connected, Ethernet2
```
```
dc1-leaf-101#show lldp nei
Last table change time   : 0:43:13 ago
Number of table inserts  : 2
Number of table deletes  : 0
Number of table drops    : 0
Number of table age-outs : 0

Port          Neighbor Device ID       Neighbor Port ID    TTL
---------- ------------------------ ---------------------- ---
Et1           dc1-spine-1              Ethernet1           120
Et2           dc1-spine-2              Ethernet1           120
dc1-leaf-101#
```
</details>

<details>
  <summary>от Leaf-2 к Spine-1 и Spine-2</summary>
 
```
dc1-leaf-102#ping 10.10.102.0
PING 10.10.102.0 (10.10.102.0) 72(100) bytes of data.
80 bytes from 10.10.102.0: icmp_seq=1 ttl=64 time=66.5 ms
80 bytes from 10.10.102.0: icmp_seq=2 ttl=64 time=60.0 ms
80 bytes from 10.10.102.0: icmp_seq=3 ttl=64 time=51.6 ms
80 bytes from 10.10.102.0: icmp_seq=4 ttl=64 time=52.4 ms
80 bytes from 10.10.102.0: icmp_seq=5 ttl=64 time=53.5 ms

--- 10.10.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 43ms
rtt min/avg/max/mdev = 51.666/56.855/66.528/5.672 ms, pipe 5, ipg/ewma 10.989/61.405 ms
dc1-leaf-102#
dc1-leaf-102#ping 10.10.202.0
PING 10.10.202.0 (10.10.202.0) 72(100) bytes of data.
80 bytes from 10.10.202.0: icmp_seq=1 ttl=64 time=10.4 ms
80 bytes from 10.10.202.0: icmp_seq=2 ttl=64 time=22.4 ms
80 bytes from 10.10.202.0: icmp_seq=3 ttl=64 time=19.0 ms
80 bytes from 10.10.202.0: icmp_seq=4 ttl=64 time=9.80 ms
80 bytes from 10.10.202.0: icmp_seq=5 ttl=64 time=6.59 ms

--- 10.10.202.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 63ms
rtt min/avg/max/mdev = 6.591/13.668/22.456/6.028 ms, pipe 2, ipg/ewma 15.893/11.717 ms
```
```
dc1-leaf-102#sh ip route

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

 C        10.1.102.0/32 is directly connected, Loopback0
 C        10.10.102.0/31 is directly connected, Ethernet1
 C        10.10.202.0/31 is directly connected, Ethernet2
```
```
dc1-leaf-102#show lldp nei
Last table change time   : 0:43:53 ago
Number of table inserts  : 2
Number of table deletes  : 0
Number of table drops    : 0
Number of table age-outs : 0

Port          Neighbor Device ID       Neighbor Port ID    TTL
---------- ------------------------ ---------------------- ---
Et1           dc1-spine-1              Ethernet2           120
Et2           dc1-spine-2              Ethernet2           120

dc1-leaf-102#
```
</details>

<details>
  <summary>от Leaf-3 к Spine-1 и Spine-2</summary>
  
```
dc1-leaf-103#ping 10.10.103.0
PING 10.10.103.0 (10.10.103.0) 72(100) bytes of data.
80 bytes from 10.10.103.0: icmp_seq=1 ttl=64 time=66.5 ms
80 bytes from 10.10.103.0: icmp_seq=2 ttl=64 time=57.7 ms
80 bytes from 10.10.103.0: icmp_seq=3 ttl=64 time=53.0 ms
80 bytes from 10.10.103.0: icmp_seq=4 ttl=64 time=48.7 ms
80 bytes from 10.10.103.0: icmp_seq=5 ttl=64 time=44.7 ms

--- 10.10.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 49ms
rtt min/avg/max/mdev = 44.778/54.182/66.580/7.563 ms, pipe 5, ipg/ewma 12.255/59.870 ms
dc1-leaf-103#
dc1-leaf-103#ping 10.10.203.0
PING 10.10.203.0 (10.10.203.0) 72(100) bytes of data.
80 bytes from 10.10.203.0: icmp_seq=1 ttl=64 time=8.88 ms
80 bytes from 10.10.203.0: icmp_seq=2 ttl=64 time=7.79 ms
80 bytes from 10.10.203.0: icmp_seq=3 ttl=64 time=6.11 ms
80 bytes from 10.10.203.0: icmp_seq=4 ttl=64 time=6.04 ms
80 bytes from 10.10.203.0: icmp_seq=5 ttl=64 time=8.13 ms

--- 10.10.203.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 43ms
rtt min/avg/max/mdev = 6.044/7.394/8.883/1.133 ms, ipg/ewma 10.843/8.123 ms
```
```
dc1-leaf-103#sh ip route

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

 C        10.1.103.0/32 is directly connected, Loopback0
 C        10.10.103.0/31 is directly connected, Ethernet1
 C        10.10.203.0/31 is directly connected, Ethernet2
```
```
dc1-leaf-103#sh lldp nei
Last table change time   : 0:44:48 ago
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
