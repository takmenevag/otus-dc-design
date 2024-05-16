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
```
```
```
```
```
</details>


<details>
  <summary>проверки spine-2</summary>
  
```
```
```
```
```
```
</details>

<details>
  <summary>проверки leaf-101</summary>
  
```
```
```
```
```
```
</details>

<details>
  <summary>проверки leaf-102</summary>
  
```
```
```
```
```
```
</details>

<details>
  <summary>проверки leaf-103</summary>
  
```
```
```
```
```
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
