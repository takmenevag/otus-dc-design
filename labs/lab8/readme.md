# Домашнее задание №7. VxLAN. Routing
[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)
## Задачи
- Разместить все хосты по двум VRF
- Настроить взаимодействие двух VRF через внешнее устройство

## Решение
### Распределение адресного пространства
_Если необходимо вспомнить или ознакомиться_
<details>
  <summary>Описание Underlay.eBGP </summary>
  
Блок IP-адресов 10.0.0.0/14 для DC1
- spine-X, leaf-1YY
- блок 10.**0**.0.0/16 - loopback \
  _третий октет - коммутатор, четвертый - номер loopback_
  - loopback**0** spine - 10.1.X.**0**/32
  - loopback**0** leaf - 10.1.1YY.**0**/32
- блок 10.**1**.0.0/16 - транспорт \
 _третий октет - spine, leaf, четвертый октет сети по /31_
  - transport spine-**X** - 10.1.**X**.<сеть>/31
  - transport leaf-**X** - 10.1.**X**.<сеть>/31
- блок 10.**2**.0.0/16 - сервисы
 _третий октет - номер VLAN, сети по /24_
 - 10.2.**VLAN**.0/24
- блок 10.**3**.0.0/16 - резерв
</details>

### Описание решения VXLAN
В решении используется следующие параметры:
- общие параметры:
	- в overlay используется интерфейс Loopback0 на spine и leaf
	- настроено соседство между spine и leaf для BGP AFI/SFI l2vpn evpn
	- команда neighbor XXX next-hop-unchanged используется для сохранения next-hop-адреса leaf-коммутатора
	- команда redistribute learned используется для анонса MAC-адреса локальных хостов как EVPN type-2 маршрутов
	- команда neighbor XXX send-community extended используется для работы EVPN (импорта, экспорта маршрутов)
	- для настройки шлюза на VTEP используется технология anycast gateway
	- команда ip address virtual используется для задания единого IP-адреса для anycast gateway на всех VTEP, выполняющих функцию шлюза для VLAN
	- команда ip virtual-router mac-address используется для задания единого MAC-адреса для anycast gateway, на всех VTEP, выполняющих функцию шлюза для VLAN
	- ARP Suppression на коммуаторах Arista включен по умолчанию
	- для маршрутизации трафика в сетевой фабрики используется модель Symmetric IRB
	- используется один VRF (tenant), в котором размещены все клиентские подсети
	- для отказоустойчивого подключения хостов используется технология EVPN Multihoming в режиме Active-Active и протокол LACP
	- для возможности огранизации отказоустойчивого подключения хостов к двух разным leaf на leaf настаивается одинаковый lacp system-id
	- индекм коммутатора для работв EVPN Multihoming назначается от меньшего IP-адреса Loopback0 к большему (в поле IP Address в маршруте type-4)
	- коммутатору leaf-101 присвоем индекс 0, а leaf-102 индекс 1
	- для определения Designated Forwarder (DF) использует функция mod - VLAN mod количество leaf (в лабе модель сервиса VLAN-based)
	- в качестве DF для всеx VLAN выбран dc1-leaf-101, т.к. номера VLAN деляться на 2 без остатка (10,20)
	- в VLAN 10 и 20 размещено по одному сервер (server-201). Серверы реализованы в виде VRF на общей платформе
- параметр VNI:
	- номер L2VNI выбирается так - 1ХХХХ, где ХХХХ номер VLAN до 4000
	- номер L3VNI выбирается так - 04YYY, где 4YYY номер VLAN после 4000. Понятно что всего VLAN - 4096 
	- номер L3VNI соотносится с номером VLAN, т.к. у части вендоров L3VNI должен соответствовать VLAN
	- номер tenant выбирается так - YYY, где YYY берется из номера L3VNI (tenant-1 -> L3VNI 4001)
- параметры RT, RT:
	- параметр RD L2VNI настраивается через auto. Коммутатор сам выставляет в RID:VLAN
	- параметр RD L3VNI настраивается вручную и задается как RID:VLAN
	- параметр RT L2VNI и L3VNI настраивается вручную, чтобы он совпадал на всех VTEP, находящихся в разных AS
	- параметр RT L2VNI и L3VNI задается так - VNI:VLAN. За счет номера VNI достигается уникальность.
- параметры EVPN Multihoming:
	- используется номер нечерного leaf в параметрах ниже
	- параметр ESI 0000:0000:№ leaf:№ Port-channel:0000
	- параметр ES-Import RT 0000:№ leaf:№ Port-channel (отбрасываются два байта слева и справа в ESI). Формат записи для облегчения понимания
	- парамет lacp system-id 0000.№ leaf.№ Port-channel 

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
![Изображение](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab6/scheme/lab6_scheme.PNG "Схема стенда")

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
|dc1-leaf-103	|Eth5	|10.1.103.252/31	|le103-fw199|
|dc1-leaf-103	|Eth6	|10.1.103.254/31	|le103-fw199|
|dc1-fw-199		|Eth1	|10.1.103.253/31	|le103-fw199|
|dc1-fw-199		|Eth2	|10.1.103.255/31	|le103-fw199|
| | | | |
|dc1-leaf-101	|Po7	|10.2.10.254/24	|Клиентская сеть, VLAN 10|
|dc1-leaf-101	|Po8	|10.2.20.254/24	|Клиентская сеть, VLAN 20|
|dc1-leaf-102	|Po7	|10.2.10.254/24	|Клиентская сеть, VLAN 10|
|dc1-leaf-102	|Po8	|10.2.20.254/24	|Клиентская сеть, VLAN 20|
|dc1-leaf-103	|Eth7	|10.2.20.254/24	|Клиентская сеть, VLAN 20|
|dc1-leaf-103	|Eth8	|10.2.30.254/24	|Клиентская сеть, VLAN 30|
| | | | |
|dc1-client-102	|Eth0	|10.2.20.102/24	|Клиентская сеть, VLAN 20|
|dc1-client-103	|Eth0	|10.2.30.103/24	|Клиентская сеть, VLAN 30|
|dc1-client-104	|Po8	|10.2.40.104/24	|Клиентская сеть, VLAN 10|
|dc1-server-201	|Po7	|10.2.10.201/24	|Клиентская сеть, VLAN 10|
|dc1-server-202	|Po7	|10.2.20.202/24	|Клиентская сеть, VLAN 20|
|dc1-server-203	|Po7	|10.2.20.203/24	|Клиентская сеть, VLAN 20|

### Таблица параметров для VRF tenant-1
|Тип VNI	|Номер VNI	|Номер VLAN	|Значение RT| Значение RD|
|:-|:-|:-|:-|:-|
|L3VNI	|4001	|4001	|4001:4001	|RID:4001|
|L2VNI	|10010	|10 	|10010:10	|RID:10|
|L2VNI	|10020	|20		|10020:20	|RID:20|

### Таблица параметров для VRF tenant-2
|Тип VNI	|Номер VNI	|Номер VLAN	|Значение RT| Значение RD|
|:-|:-|:-|:-|:-|
|L3VNI	|4002	|4002	|4002:4002	|RID:4002|
|L2VNI	|10030	|30 	|10030:30	|RID:30|
|L2VNI	|10040	|40 	10040:40	|RID:40|

</details>

### Настройка оборудования
_Команды hostname не показаны для облегчения восприятия настроек_

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

```

- leaf-102
```

```

- leaf-103
```

```

- fw-199
```

```

- server
```

```

- client-102
```
set pcname client-102
ip 10.2.20.102/24 10.2.20.254
save
```

- client-103
```
set pcname client-103
ip 10.2.30.103/24 10.2.30.254
save
```

- client-104
```

```

</details>


### Проверка взаимодействия

<details>
  <summary>вывод ip/mac хостов </summary>
  
```

```
```

```
```

```
```

```

</details>

<details>
  <summary>проверки spine-1</summary>
  
```

```
</details>


<details>
  <summary>проверки spine-2</summary>
  
```

```
</details>

<details>
  <summary>проверки leaf-101</summary>
  
```

```
</details>

<details>
  <summary>проверки leaf-102</summary>
  
```
```
</details>

<details>
  <summary>проверки leaf-103</summary>
  
```
```
</details>

<details>
  <summary>проверки fw-199</summary>
  
```
```
</details>

<details>
  <summary>проверки с client-102</summary>
  
```

```

</details>

<details>
  <summary>проверки с client-103</summary>
  
```

```
</details>


<details>
  <summary>проверки с client-104</summary>
  
```

```
</details>


<details>
  <summary>проверки с server</summary>
  
```
```
</details>


### Итоговые конфигурации оборудования
- [dc1-spine-1](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-spine-1.txt)
- [dc1-spine-2](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-spine-2.txt)
- [dc1-leaf-101](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-leaf-101.txt)
- [dc1-leaf-102](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-leaf-102.txt)
- [dc1-leaf-103](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-leaf-103.txt)
- [dc1-fw-199](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-fw-199.txt)
- [dc1-client-102](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-client-102.txt)
- [dc1-client-103](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-client-103.txt)
- [dc1-client-104](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-client-104.txt)
- [dc1-server](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-server.txt)
---

[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)

