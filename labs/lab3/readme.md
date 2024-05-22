# Домашнее задание №2. Underlay.IS-IS
[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)
## Задачи
- Настроить IS-IS для Underlay сети

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
В решении используется протокол маршрутизации IS-IS со следующими параметрами:
- все коммутаторы размещены в area 49.0001
- поле System ID адреса NET равен номеру маршрутизатора
- используется имя процеcса DC1
- все коммутаторы уровня L1
- настроена метрика по умолчанию 400 (reference bw 400/1G как bw интерфейса Arista в EVE)
- для транзитных интерфейсов настроена метрика 4 (reference bw 400/100G, как гипотетическая bw интерфейса)
- настроены hello-интервал 1 сек, hello-multiplier 4 сек.
- используется тип сети point-to-point
- интерфейсы loopback настроены как passive
- включена md5-аутентификация для LSPs, CSNPs, PSNPs (в процессе IS-IS) и  IS-IS hellos (на интерфейсах)
- настроено взаимодействие с протоколом bfd для улучшения сходимости сети
- таймеры bfd выбраны такие, чтобы сессии не флапала в EVE-NG

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
interface Ethernet1
   description ### sp1-le101 ###
   ip address 10.1.1.0/31
   bfd interval 350 min-rx 350 multiplier 3
   isis enable DC1
   isis bfd
   isis hello-interval 1
   isis hello-multiplier 4
   isis metric 4
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
!
interface Ethernet2
   description ### sp1-le102 ###
   ip address 10.1.1.2/31
   bfd interval 350 min-rx 350 multiplier 3
   isis enable DC1
   isis bfd
   isis hello-interval 1
   isis hello-multiplier 4
   isis metric 4
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
!
interface Ethernet3
   description ### sp1-le103 ###
   ip address 10.1.1.4/31
   bfd interval 350 min-rx 350 multiplier 3
   isis enable DC1
   isis bfd
   isis hello-interval 1
   isis hello-multiplier 4
   isis metric 4
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
!
interface Loopback0
   ip address 10.0.1.0/32
   isis enable DC1
   isis passive
!
ip routing
!
router isis DC1
   net 49.0001.0000.0000.0000.0000.0000.0000.0000.0001.00
   is-type level-1
   log-adjacency-changes
   authentication mode md5
   authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
   !
   address-family ipv4 unicast
      metric 400
```
- spine-2
```
interface Ethernet1
   description ### sp2-le101 ###
   ip address 10.1.2.0/31
   bfd interval 350 min-rx 350 multiplier 3
   isis enable DC1
   isis bfd
   isis hello-interval 1
   isis hello-multiplier 4
   isis metric 4
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
!
interface Ethernet2
   description ### sp2-le102 ###
   ip address 10.1.2.2/31
   bfd interval 350 min-rx 350 multiplier 3
   isis enable DC1
   isis bfd
   isis hello-interval 1
   isis hello-multiplier 4
   isis metric 4
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
!
interface Ethernet3
   description ### sp2-le103 ###
   ip address 10.1.2.4/31
   bfd interval 350 min-rx 350 multiplier 3
   isis enable DC1
   isis bfd
   isis hello-interval 1
   isis hello-multiplier 4
   isis metric 4
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
!
interface Loopback0
   ip address 10.0.2.0/32
   isis enable DC1
   isis passive
!
ip routing
!
router isis DC1
   net 49.0001.0000.0000.0000.0000.0000.0000.0000.0002.00
   is-type level-1
   log-adjacency-changes
   authentication mode md5
   authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
   !
   address-family ipv4 unicast
      metric 400
```
- leaf-101
```
interface Ethernet1
   description ### sp1-le101 ###
   ip address 10.1.1.1/31
   bfd interval 350 min-rx 350 multiplier 3
   isis enable DC1
   isis bfd
   isis hello-interval 1
   isis hello-multiplier 4
   isis metric 4
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
!
interface Ethernet2
   description ### sp2-le101 ###
   ip address 10.1.2.1/31
   bfd interval 350 min-rx 350 multiplier 3
   isis enable DC1
   isis bfd
   isis hello-interval 1
   isis hello-multiplier 4
   isis metric 4
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
!
interface Loopback0
   ip address 10.0.101.0/32
   isis enable DC1
   isis passive
!
ip routing
!
router isis DC1
   net 49.0001.0000.0000.0000.0000.0000.0000.0000.0101.00
   is-type level-1
   log-adjacency-changes
   authentication mode md5
   authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
   !
   address-family ipv4 unicast
      metric 400
```
- leaf-102
```
interface Ethernet1
   description ### sp1-le102 ###
   ip address 10.1.1.3/31
   bfd interval 350 min-rx 350 multiplier 3
   isis enable DC1
   isis bfd
   isis hello-interval 1
   isis hello-multiplier 4
   isis metric 4
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
!
interface Ethernet2
   description ### sp2-le102 ###
   ip address 10.1.2.3/31
   bfd interval 350 min-rx 350 multiplier 3
   isis enable DC1
   isis bfd
   isis hello-interval 1
   isis hello-multiplier 4
   isis metric 4
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
!
interface Loopback0
   ip address 10.0.102.0/32
   isis enable DC1
   isis passive
!
ip routing
!
router isis DC1
   net 49.0001.0000.0000.0000.0000.0000.0000.0000.0102.00
   is-type level-1
   log-adjacency-changes
   authentication mode md5
   authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
   !
   address-family ipv4 unicast
      metric 400
```
- leaf-103
```
interface Ethernet1
   description ### sp1-le103 ###
   ip address 10.1.1.5/31
   bfd interval 350 min-rx 350 multiplier 3
   isis enable DC1
   isis bfd
   isis hello-interval 1
   isis hello-multiplier 4
   isis metric 4
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
!
interface Ethernet2
   description ### sp2-le103 ###
   ip address 10.1.2.5/31
   bfd interval 350 min-rx 350 multiplier 3
   isis enable DC1
   isis bfd
   isis hello-interval 1
   isis hello-multiplier 4
   isis metric 4
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
!
interface Loopback0
   ip address 10.0.103.0/32
   isis enable DC1
   isis passive
!
ip routing
!
router isis DC1
   net 49.0001.0000.0000.0000.0000.0000.0000.0000.0103.00
   is-type level-1
   log-adjacency-changes
   authentication mode md5
   authentication key 7 ey9fC1FQ4GLmnrqR+QpX7Q==
   !
   address-family ipv4 unicast
      metric 400
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
10.1.1.1  2577281866  3063163343        Ethernet1(14)  normal   05/22/24 05:35 
10.1.1.3  1109207146  1379319596        Ethernet2(15)  normal   05/22/24 04:55 
10.1.1.5  1571597162  2816666892        Ethernet3(16)  normal   05/22/24 05:00 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up

```
```
dc1-spine-1#show isis interface brief 

IS-IS Instance: DC1 VRF: default

Interface Level IPv4 Metric IPv6 Metric Type           Adjacency
--------- ----- ----------- ----------- -------------- ---------
Loopback0 L1            400         400 loopback       (passive)
Ethernet1 L1              4           4 point-to-point         1
Ethernet2 L1              4           4 point-to-point         1
Ethernet3 L1              4           4 point-to-point         1
```
```
dc1-spine-1#show isis neighbors 
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id          
DC1       default  dc1-leaf-101     L1   Ethernet1          P2P               UP    3           0E                  
DC1       default  dc1-leaf-102     L1   Ethernet2          P2P               UP    3           0E                  
DC1       default  dc1-leaf-103     L1   Ethernet3          P2P               UP    3           0E                  
```
```
dc1-spine-1#show isis database

IS-IS Instance: DC1 VRF: default
  IS-IS Level 1 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS Flags
    dc1-spine-1.00-00           481  44404   433    180 L1 <>
    dc1-spine-2.00-00           425  29456  1195    180 L1 <>
    dc1-leaf-101.00-00          415  20893   903    157 L1 <>
    dc1-leaf-102.00-00          342  44757   958    157 L1 <>
    dc1-leaf-103.00-00          361  48904   458    157 L1 <>
```
```
dc1-spine-1#show isis database detail | i dc|Reachability
    dc1-spine-1.00-00           481  44404   425    180 L1 <>
      Hostname: dc1-spine-1
      IS Neighbor          : dc1-leaf-101.00     Metric: 4
      IS Neighbor          : dc1-leaf-103.00     Metric: 4
      IS Neighbor          : dc1-leaf-102.00     Metric: 4
      Reachability         : 10.1.1.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.1.0/32 Metric: 400 Type: 1 Up
    dc1-spine-2.00-00           425  29456  1187    180 L1 <>
      Hostname: dc1-spine-2
      IS Neighbor          : dc1-leaf-101.00     Metric: 4
      IS Neighbor          : dc1-leaf-103.00     Metric: 4
      IS Neighbor          : dc1-leaf-102.00     Metric: 4
      Reachability         : 10.1.2.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.2.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.2.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.2.0/32 Metric: 400 Type: 1 Up
    dc1-leaf-101.00-00          415  20893   895    157 L1 <>
      Hostname: dc1-leaf-101
      IS Neighbor          : dc1-spine-2.00      Metric: 4
      IS Neighbor          : dc1-spine-1.00      Metric: 4
      Reachability         : 10.1.2.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.101.0/32 Metric: 400 Type: 1 Up
    dc1-leaf-102.00-00          342  44757   950    157 L1 <>
      Hostname: dc1-leaf-102
      IS Neighbor          : dc1-spine-1.00      Metric: 4
      IS Neighbor          : dc1-spine-2.00      Metric: 4
      Reachability         : 10.1.2.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.102.0/32 Metric: 400 Type: 1 Up
    dc1-leaf-103.00-00          361  48904   450    157 L1 <>
      Hostname: dc1-leaf-103
      IS Neighbor          : dc1-spine-1.00      Metric: 4
      IS Neighbor          : dc1-spine-2.00      Metric: 4
      Reachability         : 10.1.2.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.103.0/32 Metric: 400 Type: 1 Up
```
```
dc1-spine-1#show ip route

Gateway of last resort is not set

 C        10.0.1.0/32 is directly connected, Loopback0
 I L1     10.0.2.0/32 [115/408] via 10.1.1.1, Ethernet1
                                via 10.1.1.3, Ethernet2
                                via 10.1.1.5, Ethernet3
 I L1     10.0.101.0/32 [115/404] via 10.1.1.1, Ethernet1
 I L1     10.0.102.0/32 [115/404] via 10.1.1.3, Ethernet2
 I L1     10.0.103.0/32 [115/404] via 10.1.1.5, Ethernet3
 C        10.1.1.0/31 is directly connected, Ethernet1
 C        10.1.1.2/31 is directly connected, Ethernet2
 C        10.1.1.4/31 is directly connected, Ethernet3
 I L1     10.1.2.0/31 [115/8] via 10.1.1.1, Ethernet1
 I L1     10.1.2.2/31 [115/8] via 10.1.1.3, Ethernet2
 I L1     10.1.2.4/31 [115/8] via 10.1.1.5, Ethernet3
```
_Ping spine-2, leaf-101, leaf-102, leaf-103_
```
dc1-spine-1#ping 10.0.2.0 source loopback 0
PING 10.0.2.0 (10.0.2.0) from 10.0.1.0 : 72(100) bytes of data.
80 bytes from 10.0.2.0: icmp_seq=1 ttl=63 time=49.9 ms
80 bytes from 10.0.2.0: icmp_seq=2 ttl=63 time=40.8 ms
80 bytes from 10.0.2.0: icmp_seq=3 ttl=63 time=38.8 ms
80 bytes from 10.0.2.0: icmp_seq=4 ttl=63 time=31.6 ms
80 bytes from 10.0.2.0: icmp_seq=5 ttl=63 time=35.1 ms

--- 10.0.2.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 48ms
rtt min/avg/max/mdev = 31.686/39.307/49.930/6.176 ms, pipe 5, ipg/ewma 12.203/44.273 ms
dc1-spine-1#
dc1-spine-1#ping 10.0.101.0 source loopback 0
PING 10.0.101.0 (10.0.101.0) from 10.0.1.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=64 time=15.2 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=64 time=9.12 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=64 time=7.98 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=64 time=10.8 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=64 time=18.2 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 58ms
rtt min/avg/max/mdev = 7.986/12.292/18.206/3.859 ms, ipg/ewma 14.721/13.943 ms
dc1-spine-1#
dc1-spine-1#ping 10.0.102.0 source loopback 0 
PING 10.0.102.0 (10.0.102.0) from 10.0.1.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=64 time=26.4 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=64 time=11.6 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=64 time=10.7 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=64 time=13.4 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=64 time=8.84 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 89ms
rtt min/avg/max/mdev = 8.846/14.225/26.413/6.271 ms, pipe 2, ipg/ewma 22.347/20.065 ms
dc1-spine-1#
dc1-spine-1#ping 10.0.103.0 source loopback 0 
PING 10.0.103.0 (10.0.103.0) from 10.0.1.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=64 time=38.3 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=64 time=33.0 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=64 time=37.9 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=64 time=31.4 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=64 time=10.8 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 75ms
rtt min/avg/max/mdev = 10.822/30.336/38.339/10.117 ms, pipe 4, ipg/ewma 18.827/33.672 ms
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
10.1.2.1  1694473746  2782178995        Ethernet1(14)  normal   05/22/24 05:35 
10.1.2.3  2700092432  2219191208        Ethernet2(15)  normal   05/22/24 04:28 
10.1.2.5   256708825  2720265123        Ethernet3(16)  normal   05/22/24 05:00 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```
```
dc1-spine-2#show isis interface brief 

IS-IS Instance: DC1 VRF: default

Interface Level IPv4 Metric IPv6 Metric Type           Adjacency
--------- ----- ----------- ----------- -------------- ---------
Loopback0 L1            400         400 loopback       (passive)
Ethernet1 L1              4           4 point-to-point         1
Ethernet2 L1              4           4 point-to-point         1
Ethernet3 L1              4           4 point-to-point         1
```
```
dc1-spine-2#show isis neighbors 
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id          
DC1       default  dc1-leaf-101     L1   Ethernet1          P2P               UP    3           0F                  
DC1       default  dc1-leaf-102     L1   Ethernet2          P2P               UP    3           0F                  
DC1       default  dc1-leaf-103     L1   Ethernet3          P2P               UP    3           0F                  
```
```
dc1-spine-2#show isis database

IS-IS Instance: DC1 VRF: default
  IS-IS Level 1 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS Flags
    dc1-spine-1.00-00           481  44404   433    180 L1 <>
    dc1-spine-2.00-00           425  29456  1195    180 L1 <>
    dc1-leaf-101.00-00          415  20893   903    157 L1 <>
    dc1-leaf-102.00-00          342  44757   958    157 L1 <>
    dc1-leaf-103.00-00          361  48904   458    157 L1 <>
```
```
dc1-spine-2#show isis database detail | i dc|Reachability
    dc1-spine-1.00-00           481  44404   425    180 L1 <>
      Hostname: dc1-spine-1
      IS Neighbor          : dc1-leaf-101.00     Metric: 4
      IS Neighbor          : dc1-leaf-103.00     Metric: 4
      IS Neighbor          : dc1-leaf-102.00     Metric: 4
      Reachability         : 10.1.1.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.1.0/32 Metric: 400 Type: 1 Up
    dc1-spine-2.00-00           425  29456  1187    180 L1 <>
      Hostname: dc1-spine-2
      IS Neighbor          : dc1-leaf-101.00     Metric: 4
      IS Neighbor          : dc1-leaf-103.00     Metric: 4
      IS Neighbor          : dc1-leaf-102.00     Metric: 4
      Reachability         : 10.1.2.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.2.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.2.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.2.0/32 Metric: 400 Type: 1 Up
    dc1-leaf-101.00-00          415  20893   895    157 L1 <>
      Hostname: dc1-leaf-101
      IS Neighbor          : dc1-spine-2.00      Metric: 4
      IS Neighbor          : dc1-spine-1.00      Metric: 4
      Reachability         : 10.1.2.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.101.0/32 Metric: 400 Type: 1 Up
    dc1-leaf-102.00-00          342  44757   950    157 L1 <>
      Hostname: dc1-leaf-102
      IS Neighbor          : dc1-spine-1.00      Metric: 4
      IS Neighbor          : dc1-spine-2.00      Metric: 4
      Reachability         : 10.1.2.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.102.0/32 Metric: 400 Type: 1 Up
    dc1-leaf-103.00-00          361  48904   450    157 L1 <>
      Hostname: dc1-leaf-103
      IS Neighbor          : dc1-spine-1.00      Metric: 4
      IS Neighbor          : dc1-spine-2.00      Metric: 4
      Reachability         : 10.1.2.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.103.0/32 Metric: 400 Type: 1 Up
```
```
dc1-spine-2#show ip route

Gateway of last resort is not set

 I L1     10.0.1.0/32 [115/408] via 10.1.2.1, Ethernet1
                                via 10.1.2.3, Ethernet2
                                via 10.1.2.5, Ethernet3
 C        10.0.2.0/32 is directly connected, Loopback0
 I L1     10.0.101.0/32 [115/404] via 10.1.2.1, Ethernet1
 I L1     10.0.102.0/32 [115/404] via 10.1.2.3, Ethernet2
 I L1     10.0.103.0/32 [115/404] via 10.1.2.5, Ethernet3
 I L1     10.1.1.0/31 [115/8] via 10.1.2.1, Ethernet1
 I L1     10.1.1.2/31 [115/8] via 10.1.2.3, Ethernet2
 I L1     10.1.1.4/31 [115/8] via 10.1.2.5, Ethernet3
 C        10.1.2.0/31 is directly connected, Ethernet1
 C        10.1.2.2/31 is directly connected, Ethernet2
 C        10.1.2.4/31 is directly connected, Ethernet3
```
_Ping spine-1, leaf-101, leaf-102, leaf-103_
```
dc1-spine-2#ping 10.0.1.0 source loopback 0   
PING 10.0.1.0 (10.0.1.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.1.0: icmp_seq=1 ttl=63 time=52.6 ms
80 bytes from 10.0.1.0: icmp_seq=2 ttl=63 time=52.1 ms
80 bytes from 10.0.1.0: icmp_seq=3 ttl=63 time=45.6 ms
80 bytes from 10.0.1.0: icmp_seq=4 ttl=63 time=38.1 ms
80 bytes from 10.0.1.0: icmp_seq=5 ttl=63 time=38.5 ms

--- 10.0.1.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 46ms
rtt min/avg/max/mdev = 38.101/45.424/52.685/6.294 ms, pipe 5, ipg/ewma 11.564/48.605 ms
dc1-spine-2#
dc1-spine-2#ping 10.0.101.0 source loopback 0
PING 10.0.101.0 (10.0.101.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=64 time=15.1 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=64 time=18.6 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=64 time=15.7 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=64 time=9.07 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=64 time=8.17 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 60ms
rtt min/avg/max/mdev = 8.178/13.349/18.649/4.046 ms, pipe 2, ipg/ewma 15.235/13.952 ms
dc1-spine-2#
dc1-spine-2#ping 10.0.102.0 source loopback 0 
PING 10.0.102.0 (10.0.102.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=64 time=19.8 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=64 time=15.7 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=64 time=22.8 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=64 time=18.8 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=64 time=9.57 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 70ms
rtt min/avg/max/mdev = 9.574/17.372/22.808/4.505 ms, pipe 2, ipg/ewma 17.543/18.409 ms
dc1-spine-2#
dc1-spine-2#ping 10.0.103.0 source loopback 0 
PING 10.0.103.0 (10.0.103.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=64 time=19.9 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=64 time=15.6 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=64 time=10.8 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=64 time=11.4 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=64 time=14.7 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 71ms
rtt min/avg/max/mdev = 10.826/14.538/19.997/3.301 ms, pipe 2, ipg/ewma 17.920/17.167 ms
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
10.1.1.0  3063163343  2577281866        Ethernet1(14)  normal   05/22/24 05:35 
10.1.2.0  2782178995  1694473746        Ethernet2(15)  normal   05/22/24 05:35 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```
```
dc1-leaf-101#show isis interface brief 

IS-IS Instance: DC1 VRF: default

Interface Level IPv4 Metric IPv6 Metric Type           Adjacency
--------- ----- ----------- ----------- -------------- ---------
Loopback0 L1            400         400 loopback       (passive)
Ethernet1 L1              4           4 point-to-point         1
Ethernet2 L1              4           4 point-to-point         1
```
```
dc1-leaf-101#show isis neighbors 
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id          
DC1       default  dc1-spine-1      L1   Ethernet1          P2P               UP    3           0E                  
DC1       default  dc1-spine-2      L1   Ethernet2          P2P               UP    3           0E                  
```
```
dc1-leaf-101#show isis database

IS-IS Instance: DC1 VRF: default
  IS-IS Level 1 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS Flags
    dc1-spine-1.00-00           481  44404   433    180 L1 <>
    dc1-spine-2.00-00           425  29456  1195    180 L1 <>
    dc1-leaf-101.00-00          415  20893   903    157 L1 <>
    dc1-leaf-102.00-00          342  44757   958    157 L1 <>
    dc1-leaf-103.00-00          361  48904   458    157 L1 <>
```
```
dc1-leaf-101#show isis database detail | i dc|Reachability
    dc1-spine-1.00-00           481  44404   425    180 L1 <>
      Hostname: dc1-spine-1
      IS Neighbor          : dc1-leaf-101.00     Metric: 4
      IS Neighbor          : dc1-leaf-103.00     Metric: 4
      IS Neighbor          : dc1-leaf-102.00     Metric: 4
      Reachability         : 10.1.1.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.1.0/32 Metric: 400 Type: 1 Up
    dc1-spine-2.00-00           425  29456  1187    180 L1 <>
      Hostname: dc1-spine-2
      IS Neighbor          : dc1-leaf-101.00     Metric: 4
      IS Neighbor          : dc1-leaf-103.00     Metric: 4
      IS Neighbor          : dc1-leaf-102.00     Metric: 4
      Reachability         : 10.1.2.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.2.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.2.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.2.0/32 Metric: 400 Type: 1 Up
    dc1-leaf-101.00-00          415  20893   895    157 L1 <>
      Hostname: dc1-leaf-101
      IS Neighbor          : dc1-spine-2.00      Metric: 4
      IS Neighbor          : dc1-spine-1.00      Metric: 4
      Reachability         : 10.1.2.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.101.0/32 Metric: 400 Type: 1 Up
    dc1-leaf-102.00-00          342  44757   950    157 L1 <>
      Hostname: dc1-leaf-102
      IS Neighbor          : dc1-spine-1.00      Metric: 4
      IS Neighbor          : dc1-spine-2.00      Metric: 4
      Reachability         : 10.1.2.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.102.0/32 Metric: 400 Type: 1 Up
    dc1-leaf-103.00-00          361  48904   450    157 L1 <>
      Hostname: dc1-leaf-103
      IS Neighbor          : dc1-spine-1.00      Metric: 4
      IS Neighbor          : dc1-spine-2.00      Metric: 4
      Reachability         : 10.1.2.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.103.0/32 Metric: 400 Type: 1 Up
```
```
dc1-leaf-101#show ip route

Gateway of last resort is not set

 I L1     10.0.1.0/32 [115/404] via 10.1.1.0, Ethernet1
 I L1     10.0.2.0/32 [115/404] via 10.1.2.0, Ethernet2
 C        10.0.101.0/32 is directly connected, Loopback0
 I L1     10.0.102.0/32 [115/408] via 10.1.1.0, Ethernet1
                                  via 10.1.2.0, Ethernet2
 I L1     10.0.103.0/32 [115/408] via 10.1.1.0, Ethernet1
                                  via 10.1.2.0, Ethernet2
 C        10.1.1.0/31 is directly connected, Ethernet1
 I L1     10.1.1.2/31 [115/8] via 10.1.1.0, Ethernet1
 I L1     10.1.1.4/31 [115/8] via 10.1.1.0, Ethernet1
 C        10.1.2.0/31 is directly connected, Ethernet2
 I L1     10.1.2.2/31 [115/8] via 10.1.2.0, Ethernet2
 I L1     10.1.2.4/31 [115/8] via 10.1.2.0, Ethernet2
```
_Ping leaf-102, leaf-103_
```
dc1-leaf-101#ping 10.0.102.0 source loopback 0       
PING 10.0.102.0 (10.0.102.0) from 10.0.101.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=63 time=74.2 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=63 time=56.0 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=63 time=52.9 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=63 time=43.8 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=63 time=37.9 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 57ms
rtt min/avg/max/mdev = 37.925/52.996/74.232/12.419 ms, pipe 5, ipg/ewma 14.324/62.809 ms
dc1-leaf-101#
dc1-leaf-101#ping 10.0.103.0 source loopback 0 
PING 10.0.103.0 (10.0.103.0) from 10.0.101.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=63 time=42.4 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=63 time=32.0 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=63 time=34.4 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=63 time=40.7 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=63 time=32.7 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 80ms
rtt min/avg/max/mdev = 32.015/36.468/42.408/4.278 ms, pipe 4, ipg/ewma 20.164/39.385 ms
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
10.1.1.2  1379319596  1109207146        Ethernet1(14)  normal   05/22/24 04:55 
10.1.2.2  2219191208  2700092432        Ethernet2(15)  normal   05/22/24 04:28 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```
```
dc1-leaf-102#show isis interface brief 

IS-IS Instance: DC1 VRF: default

Interface Level IPv4 Metric IPv6 Metric Type           Adjacency
--------- ----- ----------- ----------- -------------- ---------
Loopback0 L1            400         400 loopback       (passive)
Ethernet1 L1              4           4 point-to-point         1
Ethernet2 L1              4           4 point-to-point         1
```
```
dc1-leaf-102#show isis neighbors 
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id          
DC1       default  dc1-spine-1      L1   Ethernet1          P2P               UP    4           0F                  
DC1       default  dc1-spine-2      L1   Ethernet2          P2P               UP    4           0F                  
```
```
dc1-leaf-102#show isis database

IS-IS Instance: DC1 VRF: default
  IS-IS Level 1 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS Flags
    dc1-spine-1.00-00           481  44404   433    180 L1 <>
    dc1-spine-2.00-00           425  29456  1195    180 L1 <>
    dc1-leaf-101.00-00          415  20893   903    157 L1 <>
    dc1-leaf-102.00-00          342  44757   958    157 L1 <>
    dc1-leaf-103.00-00          361  48904   458    157 L1 <>
```
```
dc1-leaf-102#show isis database detail | i dc|Reachability
    dc1-spine-1.00-00           481  44404   425    180 L1 <>
      Hostname: dc1-spine-1
      IS Neighbor          : dc1-leaf-101.00     Metric: 4
      IS Neighbor          : dc1-leaf-103.00     Metric: 4
      IS Neighbor          : dc1-leaf-102.00     Metric: 4
      Reachability         : 10.1.1.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.1.0/32 Metric: 400 Type: 1 Up
    dc1-spine-2.00-00           425  29456  1187    180 L1 <>
      Hostname: dc1-spine-2
      IS Neighbor          : dc1-leaf-101.00     Metric: 4
      IS Neighbor          : dc1-leaf-103.00     Metric: 4
      IS Neighbor          : dc1-leaf-102.00     Metric: 4
      Reachability         : 10.1.2.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.2.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.2.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.2.0/32 Metric: 400 Type: 1 Up
    dc1-leaf-101.00-00          415  20893   896    157 L1 <>
      Hostname: dc1-leaf-101
      IS Neighbor          : dc1-spine-2.00      Metric: 4
      IS Neighbor          : dc1-spine-1.00      Metric: 4
      Reachability         : 10.1.2.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.101.0/32 Metric: 400 Type: 1 Up
    dc1-leaf-102.00-00          342  44757   950    157 L1 <>
      Hostname: dc1-leaf-102
      IS Neighbor          : dc1-spine-1.00      Metric: 4
      IS Neighbor          : dc1-spine-2.00      Metric: 4
      Reachability         : 10.1.2.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.102.0/32 Metric: 400 Type: 1 Up
    dc1-leaf-103.00-00          361  48904   450    157 L1 <>
      Hostname: dc1-leaf-103
      IS Neighbor          : dc1-spine-1.00      Metric: 4
      IS Neighbor          : dc1-spine-2.00      Metric: 4
      Reachability         : 10.1.2.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.103.0/32 Metric: 400 Type: 1 Up
```
```
dc1-leaf-102#show ip route

Gateway of last resort is not set

 I L1     10.0.1.0/32 [115/404] via 10.1.1.2, Ethernet1
 I L1     10.0.2.0/32 [115/404] via 10.1.2.2, Ethernet2
 I L1     10.0.101.0/32 [115/408] via 10.1.1.2, Ethernet1
                                  via 10.1.2.2, Ethernet2
 C        10.0.102.0/32 is directly connected, Loopback0
 I L1     10.0.103.0/32 [115/408] via 10.1.1.2, Ethernet1
                                  via 10.1.2.2, Ethernet2
 I L1     10.1.1.0/31 [115/8] via 10.1.1.2, Ethernet1
 C        10.1.1.2/31 is directly connected, Ethernet1
 I L1     10.1.1.4/31 [115/8] via 10.1.1.2, Ethernet1
 I L1     10.1.2.0/31 [115/8] via 10.1.2.2, Ethernet2
 C        10.1.2.2/31 is directly connected, Ethernet2
 I L1     10.1.2.4/31 [115/8] via 10.1.2.2, Ethernet2
```
_Ping leaf-101, leaf-103_
```
 dc1-leaf-102#ping 10.0.101.0 source loopback 0     
PING 10.0.101.0 (10.0.101.0) from 10.0.102.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=63 time=40.6 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=63 time=30.5 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=63 time=25.7 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=63 time=38.1 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=63 time=21.7 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 75ms
rtt min/avg/max/mdev = 21.780/31.383/40.632/7.162 ms, pipe 4, ipg/ewma 18.936/35.740 ms
dc1-leaf-102#
dc1-leaf-102#ping 10.0.103.0 source loopback 0 
PING 10.0.103.0 (10.0.103.0) from 10.0.102.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=63 time=32.3 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=63 time=29.9 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=63 time=30.5 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=63 time=19.4 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=63 time=25.4 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 85ms
rtt min/avg/max/mdev = 19.415/27.548/32.380/4.664 ms, pipe 3, ipg/ewma 21.449/29.716 ms
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
10.1.1.4  2816666892  1571597162        Ethernet1(14)  normal   05/22/24 05:00 
10.1.2.4  2720265123   256708825        Ethernet2(15)  normal   05/22/24 05:00 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```
```
dc1-leaf-103#show isis interface brief 

IS-IS Instance: DC1 VRF: default

Interface Level IPv4 Metric IPv6 Metric Type           Adjacency
--------- ----- ----------- ----------- -------------- ---------
Loopback0 L1            400         400 loopback       (passive)
Ethernet1 L1              4           4 point-to-point         1
Ethernet2 L1              4           4 point-to-point         1
```
```
dc1-leaf-103#show isis neighbors 
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id          
DC1       default  dc1-spine-1      L1   Ethernet1          P2P               UP    3           10                  
DC1       default  dc1-spine-2      L1   Ethernet2          P2P               UP    3           10                  
```
```
dc1-leaf-103#show isis database

IS-IS Instance: DC1 VRF: default
  IS-IS Level 1 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS Flags
    dc1-spine-1.00-00           481  44404   433    180 L1 <>
    dc1-spine-2.00-00           425  29456  1195    180 L1 <>
    dc1-leaf-101.00-00          415  20893   903    157 L1 <>
    dc1-leaf-102.00-00          342  44757   958    157 L1 <>
    dc1-leaf-103.00-00          361  48904   458    157 L1 <>
```
```
dc1-leaf-103#show isis database detail | i dc|Reachability
    dc1-spine-1.00-00           481  44404   425    180 L1 <>
      Hostname: dc1-spine-1
      IS Neighbor          : dc1-leaf-101.00     Metric: 4
      IS Neighbor          : dc1-leaf-103.00     Metric: 4
      IS Neighbor          : dc1-leaf-102.00     Metric: 4
      Reachability         : 10.1.1.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.1.0/32 Metric: 400 Type: 1 Up
    dc1-spine-2.00-00           425  29456  1187    180 L1 <>
      Hostname: dc1-spine-2
      IS Neighbor          : dc1-leaf-101.00     Metric: 4
      IS Neighbor          : dc1-leaf-103.00     Metric: 4
      IS Neighbor          : dc1-leaf-102.00     Metric: 4
      Reachability         : 10.1.2.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.2.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.2.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.2.0/32 Metric: 400 Type: 1 Up
    dc1-leaf-101.00-00          415  20893   895    157 L1 <>
      Hostname: dc1-leaf-101
      IS Neighbor          : dc1-spine-2.00      Metric: 4
      IS Neighbor          : dc1-spine-1.00      Metric: 4
      Reachability         : 10.1.2.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.0/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.101.0/32 Metric: 400 Type: 1 Up
    dc1-leaf-102.00-00          342  44757   950    157 L1 <>
      Hostname: dc1-leaf-102
      IS Neighbor          : dc1-spine-1.00      Metric: 4
      IS Neighbor          : dc1-spine-2.00      Metric: 4
      Reachability         : 10.1.2.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.2/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.102.0/32 Metric: 400 Type: 1 Up
    dc1-leaf-103.00-00          361  48904   450    157 L1 <>
      Hostname: dc1-leaf-103
      IS Neighbor          : dc1-spine-1.00      Metric: 4
      IS Neighbor          : dc1-spine-2.00      Metric: 4
      Reachability         : 10.1.2.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.1.1.4/31 Metric: 4 Type: 1 Up
      Reachability         : 10.0.103.0/32 Metric: 400 Type: 1 Up
```
```
dc1-leaf-103#show ip route

Gateway of last resort is not set

 I L1     10.0.1.0/32 [115/404] via 10.1.1.4, Ethernet1
 I L1     10.0.2.0/32 [115/404] via 10.1.2.4, Ethernet2
 I L1     10.0.101.0/32 [115/408] via 10.1.1.4, Ethernet1
                                  via 10.1.2.4, Ethernet2
 I L1     10.0.102.0/32 [115/408] via 10.1.1.4, Ethernet1
                                  via 10.1.2.4, Ethernet2
 C        10.0.103.0/32 is directly connected, Loopback0
 I L1     10.1.1.0/31 [115/8] via 10.1.1.4, Ethernet1
 I L1     10.1.1.2/31 [115/8] via 10.1.1.4, Ethernet1
 C        10.1.1.4/31 is directly connected, Ethernet1
 I L1     10.1.2.0/31 [115/8] via 10.1.2.4, Ethernet2
 I L1     10.1.2.2/31 [115/8] via 10.1.2.4, Ethernet2
 C        10.1.2.4/31 is directly connected, Ethernet2
```
_Ping leaf-101, leaf-102_
```
dc1-leaf-103#ping 10.0.101.0 source loopback 0  
PING 10.0.101.0 (10.0.101.0) from 10.0.103.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=63 time=49.2 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=63 time=37.8 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=63 time=39.5 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=63 time=36.7 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=63 time=33.5 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 49ms
rtt min/avg/max/mdev = 33.565/39.402/49.209/5.288 ms, pipe 5, ipg/ewma 12.255/44.024 ms
dc1-leaf-103#
dc1-leaf-103#ping 10.0.102.0 source loopback 0 
PING 10.0.102.0 (10.0.102.0) from 10.0.103.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=63 time=59.5 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=63 time=62.4 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=63 time=66.4 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=63 time=59.6 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=63 time=60.0 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 44ms
rtt min/avg/max/mdev = 59.580/61.653/66.458/2.631 ms, pipe 5, ipg/ewma 11.102/60.553 ms
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
