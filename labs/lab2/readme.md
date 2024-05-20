# Домашнее задание №2. Underlay.OSPF
[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)
## Задачи
- Настроить OSPF для Underlay сети

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
В решении используется протокол маршрутизации OSPF со следующими параметрами:
- все коммутаторы размещены в area 0
- используется номер процеcса 1
- настроены hello-интервал 1 сек., dead-интервал 4 сек.
- в качестве reference bandwidth используется 400 Гбит/c
- используется тип сети point-to-point на транзитных интерфейсах 
- включено установление соседства только на транзитных интерфейсах 
- используется md5-аутентификация при установлении соседства
- настроено взаимодействие с протоколом bfd для улучшения сходимости сети
  
_В EVE на Arista порты по 1G, поэтому OSPF cost получился таким большим, но в жизни по идее 100G минимум, поэтому должно быть нормально_

### Cхема сети
![Изображение](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab2/scheme/lab2_scheme.png "Схема стенда")

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
interface Ethernet1
   description ### sp1-le101 ###
   ip address 10.1.1.0/31
   bfd interval 250 min-rx 250 multiplier 3
   ip ospf neighbor bfd
   ip ospf dead-interval 4
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 LpyMIOE5MPsSRq6pTPawmA==
!
interface Ethernet2
   description ### sp1-le102 ###
   ip address 10.1.1.2/31
   bfd interval 250 min-rx 250 multiplier 3
   ip ospf neighbor bfd
   ip ospf dead-interval 4
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 lIjV8sEai21iNsPCdcUksQ==
!
interface Ethernet3
   description ### sp1-le103 ###
   ip address 10.1.1.4/31
   bfd interval 250 min-rx 250 multiplier 3
   ip ospf neighbor bfd
   ip ospf dead-interval 4
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 lIjV8sEai21iNsPCdcUksQ==
!
interface Loopback0
   ip address 10.0.1.0/32
   ip ospf area 0.0.0.0
!
ip routing
!
router ospf 1
   router-id 10.0.1.0
   auto-cost reference-bandwidth 400000
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   max-lsa 12000
```
- spine-2
```
interface Ethernet1
   description ### sp2-le101 ###
   ip address 10.1.2.0/31
   bfd interval 250 min-rx 250 multiplier 3
   ip ospf neighbor bfd
   ip ospf dead-interval 4
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 LpyMIOE5MPsSRq6pTPawmA==
!
interface Ethernet2
   description ### sp2-le102 ###
   ip address 10.1.2.2/31
   bfd interval 250 min-rx 250 multiplier 3
   ip ospf neighbor bfd
   ip ospf dead-interval 4
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 lIjV8sEai21iNsPCdcUksQ==
!
interface Ethernet3
   description ### sp2-le103 ###
   ip address 10.1.2.4/31
   bfd interval 250 min-rx 250 multiplier 3
   ip ospf neighbor bfd
   ip ospf dead-interval 4
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 lIjV8sEai21iNsPCdcUksQ==
!
interface Loopback0
   ip address 10.0.2.0/32
   ip ospf area 0.0.0.0
!
ip routing
!
router ospf 1
   router-id 10.0.2.0
   auto-cost reference-bandwidth 400000
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   max-lsa 12000
```
- leaf-101
```
interface Ethernet1
   description ### sp1-le101 ###
   ip address 10.1.1.1/31
   bfd interval 250 min-rx 250 multiplier 3
   ip ospf neighbor bfd
   ip ospf dead-interval 4
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 LpyMIOE5MPsSRq6pTPawmA==
!
interface Ethernet2
   description ### sp2-le101 ###
   ip address 10.1.2.1/31
   bfd interval 250 min-rx 250 multiplier 3
   ip ospf neighbor bfd
   ip ospf dead-interval 4
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 lIjV8sEai21iNsPCdcUksQ==
!
interface Loopback0
   ip address 10.0.101.0/32
   ip ospf area 0.0.0.0
!
ip routing
!
router ospf 1
   router-id 10.0.101.0
   auto-cost reference-bandwidth 400000
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   max-lsa 12000
```
- leaf-102
```
interface Ethernet1
   description ### sp1-le102 ###
   ip address 10.1.1.3/31
   bfd interval 250 min-rx 250 multiplier 3
   ip ospf neighbor bfd
   ip ospf dead-interval 4
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 LpyMIOE5MPsSRq6pTPawmA==
!
interface Ethernet2
   description ### sp2-le102 ###
   ip address 10.1.2.3/31
   bfd interval 250 min-rx 250 multiplier 3
   ip ospf neighbor bfd
   ip ospf dead-interval 4
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 lIjV8sEai21iNsPCdcUksQ==
!
interface Loopback0
   ip address 10.0.102.0/32
   ip ospf area 0.0.0.0
!
ip routing
!
router ospf 1
   router-id 10.0.102.0
   auto-cost reference-bandwidth 400000
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   max-lsa 12000
```
- leaf-103
```
interface Ethernet1
   description ### sp1-le103 ###
   ip address 10.1.1.5/31
   bfd interval 250 min-rx 250 multiplier 3
   ip ospf neighbor bfd
   ip ospf dead-interval 4
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 LpyMIOE5MPsSRq6pTPawmA==
!
interface Ethernet2
   description ### sp2-le103 ###
   ip address 10.1.2.5/31
   bfd interval 250 min-rx 250 multiplier 3
   ip ospf neighbor bfd
   ip ospf dead-interval 4
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 lIjV8sEai21iNsPCdcUksQ==
!
interface Loopback0
   ip address 10.0.103.0/32
   ip ospf area 0.0.0.0
!
ip routing
!
router ospf 1
   router-id 10.0.103.0
   auto-cost reference-bandwidth 400000
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   max-lsa 12000
```
</details>

### Проверка взаимодействия

<details>
  <summary>проверки spine-1</summary>
  
```
dc1-spine-1#show ip ospf neighbor 
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.101.0      1        default  0   FULL                   00:00:02    10.1.1.1        Ethernet1
10.0.102.0      1        default  0   FULL                   00:00:02    10.1.1.3        Ethernet2
10.0.103.0      1        default  0   FULL                   00:00:02    10.1.1.5        Ethernet3

dc1-spine-1#show ip ospf interface brief 
   Interface          Instance VRF        Area            IP Address         Cost  State      Nbrs
   Lo0                1        default    0.0.0.0         10.0.1.0/32        10    DR         0
   Et1                1        default    0.0.0.0         10.1.1.0/31        400   P2P        1
   Et2                1        default    0.0.0.0         10.1.1.2/31        400   P2P        1
   Et3                1        default    0.0.0.0         10.1.1.4/31        400   P2P        1
```
```
```
```
dc1-spine-1#show bfd peer
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.1.1.1  3852238044  4147203893        Ethernet1(14)  normal   05/16/24 11:18 
10.1.1.3  2734906866  2699289538        Ethernet2(15)  normal   05/16/24 11:17 
10.1.1.5  2774733977  2844371976        Ethernet3(16)  normal   05/16/24 11:16 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```
```
dc1-spine-1#show ip route

Gateway of last resort is not set

 C        10.0.1.0/32 is directly connected, Loopback0
 O        10.0.2.0/32 [110/810] via 10.1.1.1, Ethernet1
                                via 10.1.1.3, Ethernet2
                                via 10.1.1.5, Ethernet3
 O        10.0.101.0/32 [110/410] via 10.1.1.1, Ethernet1
 O        10.0.102.0/32 [110/410] via 10.1.1.3, Ethernet2
 O        10.0.103.0/32 [110/410] via 10.1.1.5, Ethernet3
 C        10.1.1.0/31 is directly connected, Ethernet1
 C        10.1.1.2/31 is directly connected, Ethernet2
 C        10.1.1.4/31 is directly connected, Ethernet3
 O        10.1.2.0/31 [110/800] via 10.1.1.1, Ethernet1
 O        10.1.2.2/31 [110/800] via 10.1.1.3, Ethernet2
 O        10.1.2.4/31 [110/800] via 10.1.1.5, Ethernet3
```
_Ping spine-2, leaf-101, leaf-102, leaf-103_
```
dc1-spine-1#ping 10.0.2.0 source loopback 0
PING 10.0.2.0 (10.0.2.0) from 10.0.1.0 : 72(100) bytes of data.
80 bytes from 10.0.2.0: icmp_seq=1 ttl=63 time=24.0 ms
80 bytes from 10.0.2.0: icmp_seq=2 ttl=63 time=23.6 ms
80 bytes from 10.0.2.0: icmp_seq=3 ttl=63 time=16.2 ms
80 bytes from 10.0.2.0: icmp_seq=4 ttl=63 time=14.8 ms
80 bytes from 10.0.2.0: icmp_seq=5 ttl=63 time=33.6 ms

--- 10.0.2.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 83ms
rtt min/avg/max/mdev = 14.807/22.485/33.638/6.725 ms, pipe 2, ipg/ewma 20.952/23.456 ms

dc1-spine-1#ping 10.0.101.0 source loopback 0 
PING 10.0.101.0 (10.0.101.0) from 10.0.1.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=64 time=7.62 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=64 time=6.31 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=64 time=6.13 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=64 time=6.67 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=64 time=9.88 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 33ms
rtt min/avg/max/mdev = 6.138/7.327/9.889/1.379 ms, ipg/ewma 8.492/7.550 ms

dc1-spine-1#ping 10.0.102.0 source loopback 0 
PING 10.0.102.0 (10.0.102.0) from 10.0.1.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=64 time=13.5 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=64 time=7.15 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=64 time=7.44 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=64 time=7.52 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=64 time=8.45 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 52ms
rtt min/avg/max/mdev = 7.154/8.827/13.560/2.406 ms, ipg/ewma 13.140/11.140 ms

dc1-spine-1#ping 10.0.103.0 source loopback 0 
PING 10.0.103.0 (10.0.103.0) from 10.0.1.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=64 time=15.8 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=64 time=13.5 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=64 time=13.9 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=64 time=11.5 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=64 time=20.7 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 59ms
rtt min/avg/max/mdev = 11.561/15.098/20.714/3.116 ms, pipe 2, ipg/ewma 14.926/15.581 ms
dc1-spine-1#
```
</details>


<details>
  <summary>проверки spine-2</summary>
  
```
dc1-spine-2#show ip ospf neighbor 
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.101.0      1        default  0   FULL                   00:00:02    10.1.2.1        Ethernet1
10.0.103.0      1        default  0   FULL                   00:00:02    10.1.2.5        Ethernet3
10.0.102.0      1        default  0   FULL                   00:00:02    10.1.2.3        Ethernet2

dc1-spine-2#show ip ospf interface brief 
   Interface          Instance VRF        Area            IP Address         Cost  State      Nbrs
   Lo0                1        default    0.0.0.0         10.0.2.0/32        10    DR         0
   Et1                1        default    0.0.0.0         10.1.2.0/31        400   P2P        1
   Et3                1        default    0.0.0.0         10.1.2.4/31        400   P2P        1
   Et2                1        default    0.0.0.0         10.1.2.2/31        400   P2P        1
```
```
```
```
dc1-spine-2#show bfd peer
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.1.2.1  4124771023  1764870607        Ethernet1(14)  normal   05/16/24 11:22 
10.1.2.3   515090752  2077637238        Ethernet2(15)  normal   05/16/24 11:22 
10.1.2.5  1510447651  4080486105        Ethernet3(16)  normal   05/16/24 11:22 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```
```
dc1-spine-2#show ip route
Gateway of last resort is not set

 O        10.0.1.0/32 [110/810] via 10.1.2.1, Ethernet1
                                via 10.1.2.3, Ethernet2
                                via 10.1.2.5, Ethernet3
 C        10.0.2.0/32 is directly connected, Loopback0
 O        10.0.101.0/32 [110/410] via 10.1.2.1, Ethernet1
 O        10.0.102.0/32 [110/410] via 10.1.2.3, Ethernet2
 O        10.0.103.0/32 [110/410] via 10.1.2.5, Ethernet3
 O        10.1.1.0/31 [110/800] via 10.1.2.1, Ethernet1
 O        10.1.1.2/31 [110/800] via 10.1.2.3, Ethernet2
 O        10.1.1.4/31 [110/800] via 10.1.2.5, Ethernet3
 C        10.1.2.0/31 is directly connected, Ethernet1
 C        10.1.2.2/31 is directly connected, Ethernet2
 C        10.1.2.4/31 is directly connected, Ethernet3
```
_Ping spine-1, leaf-101, leaf-102, leaf-103_
```
dc1-spine-2#ping 10.0.1.0 source loopback 0
PING 10.0.1.0 (10.0.1.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.1.0: icmp_seq=1 ttl=63 time=25.0 ms
80 bytes from 10.0.1.0: icmp_seq=2 ttl=63 time=18.9 ms
80 bytes from 10.0.1.0: icmp_seq=3 ttl=63 time=25.3 ms
80 bytes from 10.0.1.0: icmp_seq=4 ttl=63 time=24.6 ms
80 bytes from 10.0.1.0: icmp_seq=5 ttl=63 time=22.7 ms

--- 10.0.1.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 85ms
rtt min/avg/max/mdev = 18.987/23.356/25.351/2.374 ms, pipe 2, ipg/ewma 21.323/24.234 ms

dc1-spine-2#ping 10.0.101.0 source loopback 0
PING 10.0.101.0 (10.0.101.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=64 time=14.3 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=64 time=46.0 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=64 time=36.0 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=64 time=30.2 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=64 time=27.2 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 58ms
rtt min/avg/max/mdev = 14.378/30.785/46.004/10.404 ms, pipe 4, ipg/ewma 14.529/22.446 ms

dc1-spine-2#ping 10.0.102.0 source loopback 0 
PING 10.0.102.0 (10.0.102.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=64 time=10.2 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 1 received, 80% packet loss, time 69ms
rtt min/avg/max/mdev = 10.209/10.209/10.209/0.000 ms, ipg/ewma 17.289/10.209 ms

dc1-spine-2#ping 10.0.103.0 source loopback 0 
PING 10.0.103.0 (10.0.103.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=64 time=10.2 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=64 time=6.34 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=64 time=8.16 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=64 time=9.16 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=64 time=7.63 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 47ms
rtt min/avg/max/mdev = 6.343/8.306/10.227/1.326 ms, ipg/ewma 11.828/9.264 ms
```
</details>

<details>
  <summary>проверки leaf-101</summary>
  
```
dc1-leaf-101#show ip ospf neighbor 
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.1.0        1        default  0   FULL                   00:00:02    10.1.1.0        Ethernet1
10.0.2.0        1        default  0   FULL                   00:00:02    10.1.2.0        Ethernet2

dc1-leaf-101#show ip ospf interface brief 
   Interface          Instance VRF        Area            IP Address         Cost  State      Nbrs
   Lo0                1        default    0.0.0.0         10.0.101.0/32      10    DR         0
   Et1                1        default    0.0.0.0         10.1.1.1/31        400   P2P        1
   Et2                1        default    0.0.0.0         10.1.2.1/31        400   P2P        1
```
```
```
```
dc1-leaf-101#show bfd peer
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.1.1.0  4147203893  3852238044        Ethernet1(14)  normal   05/16/24 11:18 
10.1.2.0  2562776883  3497327111        Ethernet2(15)  normal   05/16/24 11:18 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```
```
dc1-leaf-101#show ip route

Gateway of last resort is not set

 O        10.0.1.0/32 [110/410] via 10.1.1.0, Ethernet1
 O        10.0.2.0/32 [110/410] via 10.1.2.0, Ethernet2
 C        10.0.101.0/32 is directly connected, Loopback0
 O        10.0.102.0/32 [110/810] via 10.1.1.0, Ethernet1
                                  via 10.1.2.0, Ethernet2
 O        10.0.103.0/32 [110/810] via 10.1.1.0, Ethernet1
                                  via 10.1.2.0, Ethernet2
 C        10.1.1.0/31 is directly connected, Ethernet1
 O        10.1.1.2/31 [110/800] via 10.1.1.0, Ethernet1
 O        10.1.1.4/31 [110/800] via 10.1.1.0, Ethernet1
 C        10.1.2.0/31 is directly connected, Ethernet2
 O        10.1.2.2/31 [110/800] via 10.1.2.0, Ethernet2
 O        10.1.2.4/31 [110/800] via 10.1.2.0, Ethernet2
```
_Ping leaf-102, leaf-103_
```
dc1-leaf-101#ping 10.0.102.0 source loopback 0
PING 10.0.102.0 (10.0.102.0) from 10.0.101.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=63 time=19.9 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=63 time=17.3 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=63 time=15.5 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=63 time=15.6 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=63 time=18.3 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 75ms
rtt min/avg/max/mdev = 15.581/17.400/19.987/1.672 ms, pipe 2, ipg/ewma 18.978/18.674 ms

dc1-leaf-101#ping 10.0.103.0 source loopback 0 
PING 10.0.103.0 (10.0.103.0) from 10.0.101.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=63 time=33.7 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=63 time=37.5 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=63 time=31.6 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=63 time=67.2 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=63 time=64.3 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 93ms
rtt min/avg/max/mdev = 31.621/46.906/67.217/15.576 ms, pipe 3, ipg/ewma 23.475/41.336 ms
dc1-leaf-101#
```
</details>

<details>
  <summary>проверки leaf-102</summary>
  
```
dc1-leaf-102#show ip ospf neighbor 
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.1.0        1        default  0   FULL                   00:00:02    10.1.1.2        Ethernet1
10.0.2.0        1        default  0   FULL                   00:00:02    10.1.2.2        Ethernet2

dc1-leaf-102#show ip ospf interface brief 
   Interface          Instance VRF        Area            IP Address         Cost  State      Nbrs
   Lo0                1        default    0.0.0.0         10.0.102.0/32      10    DR         0
   Et1                1        default    0.0.0.0         10.1.1.3/31        400   P2P        1
   Et2                1        default    0.0.0.0         10.1.2.3/31        400   P2P        1
```
```
```
```
dc1-leaf-102#show bfd peer
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.1.1.2  2699289538  2734906866        Ethernet1(14)  normal   05/16/24 11:17 
10.1.2.2   524993160   205108505        Ethernet2(15)  normal   05/16/24 11:15 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```
```
dc1-leaf-102#show ip route

Gateway of last resort is not set

 O        10.0.1.0/32 [110/410] via 10.1.1.2, Ethernet1
 O        10.0.2.0/32 [110/410] via 10.1.2.2, Ethernet2
 O        10.0.101.0/32 [110/810] via 10.1.1.2, Ethernet1
                                  via 10.1.2.2, Ethernet2
 C        10.0.102.0/32 is directly connected, Loopback0
 O        10.0.103.0/32 [110/810] via 10.1.1.2, Ethernet1
                                  via 10.1.2.2, Ethernet2
 O        10.1.1.0/31 [110/800] via 10.1.1.2, Ethernet1
 C        10.1.1.2/31 is directly connected, Ethernet1
 O        10.1.1.4/31 [110/800] via 10.1.1.2, Ethernet1
 O        10.1.2.0/31 [110/800] via 10.1.2.2, Ethernet2
 C        10.1.2.2/31 is directly connected, Ethernet2
 O        10.1.2.4/31 [110/800] via 10.1.2.2, Ethernet2
```
_Ping leaf-101, leaf-103_
```
dc1-leaf-102#ping 10.0.101.0 source loopback 0 
PING 10.0.101.0 (10.0.101.0) from 10.0.102.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=63 time=21.5 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=63 time=17.4 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=63 time=13.3 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=63 time=14.2 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=63 time=13.3 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 75ms
rtt min/avg/max/mdev = 13.302/15.982/21.559/3.184 ms, pipe 2, ipg/ewma 18.778/18.597 ms

dc1-leaf-102#ping 10.0.103.0 source loopback 0 
PING 10.0.103.0 (10.0.103.0) from 10.0.102.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=63 time=21.6 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=63 time=14.0 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=63 time=20.1 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=63 time=17.0 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=63 time=19.1 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 81ms
rtt min/avg/max/mdev = 14.075/18.426/21.606/2.629 ms, pipe 2, ipg/ewma 20.451/20.042 ms
```
</details>

<details>
  <summary>проверки leaf-103</summary>
  
```
dc1-leaf-103#show ip ospf neighbor 
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.2.0        1        default  0   FULL                   00:00:02    10.1.2.4        Ethernet2
10.0.1.0        1        default  0   FULL                   00:00:02    10.1.1.4        Ethernet1

dc1-leaf-103#show ip ospf interface brief 
   Interface          Instance VRF        Area            IP Address         Cost  State      Nbrs
   Lo0                1        default    0.0.0.0         10.0.103.0/32      10    DR         0
   Et2                1        default    0.0.0.0         10.1.2.5/31        400   P2P        1
   Et1                1        default    0.0.0.0         10.1.1.5/31        400   P2P        1
```
```
```
```
dc1-leaf-103#show bfd peer
VRF name: default
-----------------
DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
--------- ----------- ----------- -------------------- ------- ----------------
10.1.1.4  2844371976  2774733977        Ethernet1(14)  normal   05/16/24 11:16 
10.1.2.4   686881338  2714916868        Ethernet2(15)  normal   05/16/24 11:15 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```
```
dc1-leaf-103#show ip route

Gateway of last resort is not set

 O        10.0.1.0/32 [110/410] via 10.1.1.4, Ethernet1
 O        10.0.2.0/32 [110/410] via 10.1.2.4, Ethernet2
 O        10.0.101.0/32 [110/810] via 10.1.1.4, Ethernet1
                                  via 10.1.2.4, Ethernet2
 O        10.0.102.0/32 [110/810] via 10.1.1.4, Ethernet1
                                  via 10.1.2.4, Ethernet2
 C        10.0.103.0/32 is directly connected, Loopback0
 O        10.1.1.0/31 [110/800] via 10.1.1.4, Ethernet1
 O        10.1.1.2/31 [110/800] via 10.1.1.4, Ethernet1
 C        10.1.1.4/31 is directly connected, Ethernet1
 O        10.1.2.0/31 [110/800] via 10.1.2.4, Ethernet2
 O        10.1.2.2/31 [110/800] via 10.1.2.4, Ethernet2
 C        10.1.2.4/31 is directly connected, Ethernet2
```
_Ping leaf-101, leaf-102_
```
dc1-leaf-103#ping 10.0.101.0 source loopback 0 
PING 10.0.101.0 (10.0.101.0) from 10.0.103.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=63 time=16.9 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=63 time=15.9 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=63 time=14.2 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=63 time=16.3 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=63 time=16.7 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 65ms
rtt min/avg/max/mdev = 14.285/16.063/16.969/0.958 ms, pipe 2, ipg/ewma 16.444/16.533 ms

dc1-leaf-103#ping 10.0.102.0 source loopback 0 
PING 10.0.102.0 (10.0.102.0) from 10.0.103.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=63 time=31.5 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=63 time=37.5 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=63 time=42.5 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=63 time=60.8 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=63 time=38.6 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 99ms
rtt min/avg/max/mdev = 31.586/42.224/60.885/9.966 ms, pipe 3, ipg/ewma 24.816/37.212 ms
```
</details>

### Итоговые конфигурации оборудования
- [dc1-spine-1](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab2/config/dc1-spine-1.txt)
- [dc1-spine-2](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab2/config/dc1-spine-2.txt)
- [dc1-leaf-101](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab2/config/dc1-leaf-101.txt)
- [dc1-leaf-102](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab2/config/dc1-leaf-102.txt)
- [dc1-leaf-103](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab2/config/dc1-leaf-103.txt)
---

[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)
