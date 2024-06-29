# Домашнее задание №7. VxLAN. Multihoming
[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)
## Задачи
- Настроить маршрутизацию в рамках Overlay между клиентами
- Подключить клиентов 2-я линками к различным Leaf
- Настроить агрегированный канал со стороны клиента
- Настроить multihoming для работы в Overlay сети (vPC, ESI LAG либо MC-LAG)
- Выбран ESI LAG (EVPN Multihoming)

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
 _третий октет - spine, четвертый октет сети по /31_
  - transport spine-**X** - 10.1.**X**.<сеть>/31
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
![Изображение](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/scheme/lab7_scheme.PNG "Схема стенда")

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
| | | | |
|dc1-leaf-101	|Po7	|10.2.10.254/24	|Клиентская сеть, VLAN 10|
|dc1-leaf-101	|Po8	|10.2.20.254/24	|Клиентская сеть, VLAN 20|
|dc1-leaf-102	|Po7	|10.2.10.254/24	|Клиентская сеть, VLAN 10|
|dc1-leaf-102	|Po8	|10.2.20.254/24	|Клиентская сеть, VLAN 20|
|dc1-leaf-103	|Eth7	|10.2.20.254/24	|Клиентская сеть, VLAN 20|
|dc1-leaf-103	|Eth8	|10.2.30.254/24	|Клиентская сеть, VLAN 30|
| | | | |
|dc1-client-101	|Po8	|10.2.10.101/24	|Клиентская сеть, VLAN 10|
|dc1-client-102	|Eth0	|10.2.20.102/24	|Клиентская сеть, VLAN 20|
|dc1-client-103	|Eth0	|10.2.30.103/24	|Клиентская сеть, VLAN 30|
|dc1-server-201	|Po7	|10.2.10.201/24	|Клиентская сеть, VLAN 10|
|dc1-server-201	|Po7	|10.2.20.201/24	|Клиентская сеть, VLAN 20|

### Таблица параметров для VRF tenant-1
|Тип VNI	|Номер VNI	|Номер VLAN	|Значение RT| Значение RD|
|:-|:-|:-|:-|:-|
|L3VNI	|4001	|4001	|4001:4001 |RID:4001|
|L2VNI	|10010	|10	|10010:10 |RID:10|
|L2VNI	|10020	|20	|10020:20 |RID:20|
|L2VNI	|10030	|30	|10030:30 |RID:30|

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
service routing protocols model multi-agent
!
vlan 10
   name NET-10.2.10.0/24
!
vlan 20
   name NET-10.2.20.0/24
!
vrf instance tenant-1
!
interface Port-Channel7
   description ### server-201 ###
   switchport trunk allowed vlan 10,20
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0000:0101:0007:0000
      route-target import 00:00:01:01:00:07
   lacp system-id 0000.0101.0007
!
interface Port-Channel8
   description ### client-101 ###
   switchport trunk allowed vlan 10
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0000:0101:0008:0000
      route-target import 00:00:01:01:00:08
   lacp system-id 0000.0101.0008
!
interface Ethernet1
   description ### sp1-le101 ###
   no switchport
   ip address 10.1.1.1/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet2
   description ### sp2-le101 ###
   no switchport
   ip address 10.1.2.1/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet7
   description ### server-201 ###
   channel-group 7 mode active
!
interface Ethernet8
   description ### client-101 ###
   channel-group 8 mode active
!
interface Loopback0
   ip address 10.0.101.0/32
!
interface Vlan10
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.2.10.254/24
!
interface Vlan20
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.2.20.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf tenant-1 vni 4001
!
ip virtual-router mac-address 00:00:00:00:ca:fe
!
ip routing
ip routing vrf tenant-1
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
      route-target both 10010:10
      redistribute learned
   !
   vlan 20
      rd auto
      route-target both 10020:20
      redistribute learned
   !
   address-family evpn
      neighbor DC1-SPINE activate
   !
   address-family ipv4
      neighbor DC1-SPINE activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
   !
   vrf tenant-1
      rd 10.0.101.0:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
```

- leaf-102
```
service routing protocols model multi-agent
!
vlan 10
   name NET-10.2.10.0/24
!
vlan 20
   name NET-10.2.20.0/24
!
vrf instance tenant-1
!
interface Port-Channel7
   description ### server-201 ###
   switchport trunk allowed vlan 10,20
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0000:0101:0007:0000
      route-target import 00:00:01:01:00:07
   lacp system-id 0000.0101.0007
!
interface Port-Channel8
   description ### client-101 ###
   switchport trunk allowed vlan 10
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0000:0101:0008:0000
      route-target import 00:00:01:01:00:08
   lacp system-id 0000.0101.0008
!
interface Ethernet1
   description ### sp1-le102 ###
   no switchport
   ip address 10.1.1.3/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet2
   description ### sp2-le102 ###
   no switchport
   ip address 10.1.2.3/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet7
   description ### server-201 ###
   channel-group 7 mode active
!
interface Ethernet8
   description ### client-101 ###
   channel-group 8 mode active
!
interface Loopback0
   ip address 10.0.102.0/32
!
interface Vlan10
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.2.10.254/24
!
interface Vlan20
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.2.20.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf tenant-1 vni 4001
!
ip virtual-router mac-address 00:00:00:00:ca:fe
!
ip routing
ip routing vrf tenant-1
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
      route-target both 10010:10
      redistribute learned
   !
   vlan 20
      rd auto
      route-target both 10020:20
      redistribute learned
   !
   address-family evpn
      neighbor DC1-SPINE activate
   !
   address-family ipv4
      neighbor DC1-SPINE activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
   !
   vrf tenant-1
      rd 10.0.101.0:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
```

- leaf-103
```
service routing protocols model multi-agent
!
vlan 20
   name NET-10.2.20.0/24
!
vlan 30
   name NET-10.2.30.0/24
!
vrf instance tenant-1
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
   description ### client-102 ###
   switchport access vlan 20
!
interface Ethernet8
   description ### client-103 ###
   switchport access vlan 30
!
interface Loopback0
   ip address 10.0.103.0/32
!
interface Vlan20
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.2.20.254/24
!
interface Vlan30
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.2.30.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vrf tenant-1 vni 4001
!
ip virtual-router mac-address 00:00:00:00:ca:fe
!
ip routing
ip routing vrf tenant-1
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
   vlan 20
      rd auto
      route-target both 10020:20
      redistribute learned
   !
   vlan 30
      rd auto
      route-target both 10030:30
      redistribute learned
   !
   address-family evpn
      neighbor DC1-SPINE activate
   !
   address-family ipv4
      neighbor DC1-SPINE activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
   !
   vrf tenant-1
      rd 10.0.103.0:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
```

- server-201
```
service routing protocols model ribd
!
vlan 10
   name NET-10.2.10.0/24
!
vlan 20
   name NET-10.2.20.0/24
!
vrf instance vlan-10
!
vrf instance vlan-20
!
interface Port-Channel7
   description ### uplink ###
   switchport trunk allowed vlan 10,20
   switchport mode trunk
!
interface Ethernet1
   channel-group 7 mode active
!
interface Ethernet2
   channel-group 7 mode active
!
interface Vlan10
   description ### server-201 ###
   vrf vlan-10
   ip address 10.2.10.201/24
!
interface Vlan20
   description ### server-201 ###
   vrf vlan-20
   ip address 10.2.20.201/24
!
ip routing
ip routing vrf vlan-10
ip routing vrf vlan-20
!
ip route vrf vlan-10 0.0.0.0/0 10.2.10.254
ip route vrf vlan-20 0.0.0.0/0 10.2.20.254
```

- client-101
```
service routing protocols model ribd
!
vlan 10
   name NET-10.2.10.0/24
!
interface Port-Channel8
   description ### uplink ###
   switchport trunk allowed vlan 10
   switchport mode trunk
!
interface Ethernet1
   channel-group 8 mode active
!
interface Ethernet2
   channel-group 8 mode active
!
interface Vlan10
   ip address 10.2.10.101/24
!
ip routing
!
ip route 0.0.0.0/0 10.2.10.254
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

</details>


### Проверка взаимодействия

<details>
  <summary>вывод ip/mac хостов </summary>
  
```
client-101#show interfaces | i address|Vlan
Vlan10 is up, line protocol is up (connected)
  Hardware is Vlan, address is 5000.0045.abdf (bia 5000.0045.abdf)
  Internet address is 10.2.10.101/24
  Broadcast address is 255.255.255.255

client-101#show ip arp
Address         Age (sec)  Hardware Addr   Interface
10.2.10.201       2:53:12  5000.0088.fe27  Vlan10, Port-Channel8
10.2.10.254       0:19:33  0000.0000.cafe  Vlan10, Port-Channel8
```
```
client-102> show ip all
NAME   IP/MASK              GATEWAY           MAC                DNS
client-10.2.20.102/24       10.2.20.254       00:50:79:66:68:08  

client-102> show arp
00:00:00:00:ca:fe  10.2.20.254 expires in 101 seconds 
```
```
client-103> show ip all
NAME   IP/MASK              GATEWAY           MAC                DNS
client-10.2.30.103/24       10.2.30.254       00:50:79:66:68:09  

client-103> show arp
00:00:00:00:ca:fe  10.2.30.254 expires in 114 seconds 
```
```
server-201#show interfaces | i address|Vlan
Vlan10 is up, line protocol is up (connected)
  Hardware is Vlan, address is 5000.0088.fe27 (bia 5000.0088.fe27)
  Internet address is 10.2.10.201/24
  Broadcast address is 255.255.255.255

Vlan20 is up, line protocol is up (connected)
  Hardware is Vlan, address is 5000.0088.fe27 (bia 5000.0088.fe27)
  Internet address is 10.2.20.201/24
  Broadcast address is 255.255.255.255

server-201#show ip arp vrf vlan-10
Address         Age (sec)  Hardware Addr   Interface
10.2.10.101       1:11:14  5000.0045.abdf  Vlan10, Port-Channel7
10.2.10.254       2:52:05  0000.0000.cafe  Vlan10, Port-Channel7

server-201#show ip arp vrf vlan-20
Address         Age (sec)  Hardware Addr   Interface
10.2.20.102       2:18:30  0050.7966.6808  Vlan20, Port-Channel7
10.2.20.254       3:25:43  0000.0000.cafe  Vlan20, Port-Channel7
```

</details>

<details>
  <summary>проверки spine-1</summary>
  
```
dc1-spine-1#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.1.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.1.1.1 4 65101            230       221    0    0 00:08:20 Estab   1      1
  10.1.1.3 4 65102            228       217    0    0 00:08:20 Estab   1      1
  10.1.1.5 4 65103            234       223    0    0 00:08:20 Estab   1      1
```
```
dc1-spine-1#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.1.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.1.1.1 4 65101            233       224    0    0 00:08:28 Estab   15     15
  10.1.1.3 4 65102            231       220    0    0 00:08:28 Estab   14     14
  10.1.1.5 4 65103            237       226    0    0 00:08:28 Estab   6      6

```
</details>


<details>
  <summary>проверки spine-2</summary>
  
```
dc1-spine-2#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.2.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.1.2.1 4 65101            572       563    0    0 00:22:42 Estab   1      1
  10.1.2.3 4 65102            577       555    0    0 00:22:42 Estab   1      1
  10.1.2.5 4 65103            579       560    0    0 00:22:42 Estab   1      1
```
```
dc1-spine-2#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.2.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.1.2.1 4 65101            575       566    0    0 00:22:50 Estab   15     15
  10.1.2.3 4 65102            580       558    0    0 00:22:50 Estab   14     14
  10.1.2.5 4 65103            582       563    0    0 00:22:50 Estab   6      6
```
</details>

<details>
  <summary>проверки leaf-101</summary>
  
```
dc1-leaf-101#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.101.0, local AS number 65101
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-spine-1 ###      10.1.1.0 4 65100          75515     76322    0    0 00:08:20 Estab   3      3
  ### dc1-spine-2 ###      10.1.2.0 4 65100          75493     76082    0    0 00:22:42 Estab   3      3
```
```
dc1-leaf-101#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.101.0, local AS number 65101
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-spine-1 ###      10.1.1.0 4 65100          75518     76325    0    0 00:08:28 Estab   20     20
  ### dc1-spine-2 ###      10.1.2.0 4 65100          75496     76085    0    0 00:22:50 Estab   20     20
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
dc1-leaf-101#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface           802.1Q Tag
----------- ---------- ------------ ------------------- ----------
10010       10         static       Port-Channel7       10        
                                    Port-Channel8       10        
                                    Vxlan1              10        
10020       20         static       Port-Channel7       20        
                                    Vxlan1              20        

VNI to dynamic VLAN Mapping for Vxlan1
VNI        VLAN       VRF            Source       
---------- ---------- -------------- ------------ 
4001       4094       tenant-1       evpn         
```
```
dc1-leaf-101#show interfaces vxlan 1

Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback0 and is active with 10.0.101.0
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is 
    [10, 10010]       [20, 10020]      
  Dynamic VLAN to VNI mapping for 'evpn' is
    [4094, 4001]     
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is 
   [tenant-1, 4001]
  Headend replication flood vtep list is:
    10 10.0.102.0     
    20 10.0.102.0      10.0.103.0     
  Shared Router MAC is 0000.0000.0000
```
```
dc1-leaf-101#show ip route vrf tenant-1

VRF: tenant-1
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

 C        10.2.10.0/24 is directly connected, Vlan10
 B E      10.2.20.102/32 [20/0] via VTEP 10.0.103.0 VNI 4001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 C        10.2.20.0/24 is directly connected, Vlan20
 B E      10.2.30.103/32 [20/0] via VTEP 10.0.103.0 VNI 4001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
```
```
dc1-leaf-101#show bgp evpn route-type imet
BGP routing table information for VRF default
Router identifier 10.0.101.0, local AS number 65101
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.101.0:10 imet 10.0.101.0
                                 -                     -       -       0       i
 * >      RD: 10.0.101.0:20 imet 10.0.101.0
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:10 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.103.0:20 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:20 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:30 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:30 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
```
```
dc1-leaf-101#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.0.101.0, local AS number 65101
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.103.0:20 mac-ip 0050.7966.6808
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:20 mac-ip 0050.7966.6808
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:20 mac-ip 0050.7966.6808 10.2.20.102
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:20 mac-ip 0050.7966.6808 10.2.20.102
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:30 mac-ip 0050.7966.6809
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:30 mac-ip 0050.7966.6809
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:30 mac-ip 0050.7966.6809 10.2.30.103
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:30 mac-ip 0050.7966.6809 10.2.30.103
                                 10.0.103.0            -       100     0       65100 65103 i
 * >      RD: 10.0.101.0:10 mac-ip 5000.0045.abdf
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:10 mac-ip 5000.0045.abdf
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 mac-ip 5000.0045.abdf
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.101.0:10 mac-ip 5000.0045.abdf 10.2.10.101
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:10 mac-ip 5000.0045.abdf 10.2.10.101
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 mac-ip 5000.0045.abdf 10.2.10.101
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.101.0:10 mac-ip 5000.0088.fe27
                                 -                     -       -       0       i
 * >      RD: 10.0.101.0:20 mac-ip 5000.0088.fe27
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:10 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:20 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.101.0:10 mac-ip 5000.0088.fe27 10.2.10.201
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:10 mac-ip 5000.0088.fe27 10.2.10.201
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 mac-ip 5000.0088.fe27 10.2.10.201
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.101.0:20 mac-ip 5000.0088.fe27 10.2.20.201
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:20 mac-ip 5000.0088.fe27 10.2.20.201
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 mac-ip 5000.0088.fe27 10.2.20.201
                                 10.0.102.0            -       100     0       65100 65102 i
```
```
dc1-leaf-101#show bgp evpn route-type auto-discovery
BGP routing table information for VRF default
Router identifier 10.0.101.0, local AS number 65101
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.101.0:10 auto-discovery 0 0000:0000:0101:0007:0000
                                 -                     -       -       0       i
 * >      RD: 10.0.101.0:20 auto-discovery 0 0000:0000:0101:0007:0000
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:10 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:20 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.101.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.101.0:10 auto-discovery 0 0000:0000:0101:0008:0000
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:10 auto-discovery 0 0000:0000:0101:0008:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 auto-discovery 0 0000:0000:0101:0008:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.101.0:1 auto-discovery 0000:0000:0101:0008:0000
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:1 auto-discovery 0000:0000:0101:0008:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:1 auto-discovery 0000:0000:0101:0008:0000
                                 10.0.102.0            -       100     0       65100 65102 i
```
```
dc1-leaf-101#show bgp evpn route-type ethernet-segment
BGP routing table information for VRF default
Router identifier 10.0.101.0, local AS number 65101
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.101.0:1 ethernet-segment 0000:0000:0101:0007:0000 10.0.101.0
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:1 ethernet-segment 0000:0000:0101:0007:0000 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:1 ethernet-segment 0000:0000:0101:0007:0000 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.101.0:1 ethernet-segment 0000:0000:0101:0008:0000 10.0.101.0
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:1 ethernet-segment 0000:0000:0101:0008:0000 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:1 ethernet-segment 0000:0000:0101:0008:0000 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
```
```
dc1-leaf-101#show bgp evpn route-type ip-prefix ipv4 
BGP routing table information for VRF default
Router identifier 10.0.101.0, local AS number 65101
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
```
```
dc1-leaf-101#show bgp evpn instance
EVPN instance: VLAN 10
  Route distinguisher: 0:0
  Route target import: Route-Target-AS:10010:10
  Route target export: Route-Target-AS:10010:10
  Service interface: VLAN-based
  Local VXLAN IP address: 10.0.101.0
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0000:0000:0101:0008:0000
      Interface: Port-Channel8
      Mode: all-active
      State: up
      ES-Import RT: 00:00:01:01:00:08
      DF election algorithm: modulus
      Designated forwarder: 10.0.101.0
      Non-Designated forwarder: 10.0.102.0
    ESI: 0000:0000:0101:0007:0000
      Interface: Port-Channel7
      Mode: all-active
      State: up
      ES-Import RT: 00:00:01:01:00:07
      DF election algorithm: modulus
      Designated forwarder: 10.0.101.0
      Non-Designated forwarder: 10.0.102.0
EVPN instance: VLAN 20
  Route distinguisher: 0:0
  Route target import: Route-Target-AS:10020:20
  Route target export: Route-Target-AS:10020:20
  Service interface: VLAN-based
  Local VXLAN IP address: 10.0.101.0
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0000:0000:0101:0007:0000
      Interface: Port-Channel7
      Mode: all-active
      State: up
      ES-Import RT: 00:00:01:01:00:07
      DF election algorithm: modulus
      Designated forwarder: 10.0.101.0
      Non-Designated forwarder: 10.0.102.0
```
```
dc1-leaf-101#show port-channel dense 

                 Flags                                                         
------------------------ ---------------------------- -------------------------
  a - LACP Active          p - LACP Passive           * - static fallback      
  F - Fallback enabled     f - Fallback configured    ^ - individual fallback  
  U - In Use               D - Down                                            
  + - In-Sync              - - Out-of-Sync            i - incompatible with agg
  P - bundled in Po        s - suspended              G - Aggregable           
  I - Individual           S - ShortTimeout           w - wait for agg         
  E - Inactive. The number of configured port channels exceeds the config limit
   M - Exceeds maximum weight

Number of channels in use: 2
Number of aggregators: 2

   Port-Channel       Protocol    Ports    
------------------ -------------- ---------
   Po7(U)             LACP(a)     Et7(PG+) 
   Po8(U)             LACP(a)     Et8(PG+) 
```
```
dc1-leaf-101#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  20  0050.7966.6808  EVPN      Vx1  10.0.103.0       1       0:57:54 ago
4094  5000.0003.3766  EVPN      Vx1  10.0.103.0       1       0:57:55 ago
4094  5000.00d5.5dc0  EVPN      Vx1  10.0.102.0       1       0:57:54 ago
Total Remote Mac Addresses for this criterion: 3
```
```
dc1-leaf-101#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
   1    0000.0000.cafe    STATIC      Cpu
  10    0000.0000.cafe    STATIC      Cpu
  10    5000.0045.abdf    DYNAMIC     Po8        2       1 day, 17:49:17 ago
  10    5000.0088.fe27    DYNAMIC     Po7        1       1 day, 17:49:28 ago
  20    0000.0000.cafe    STATIC      Cpu
  20    0050.7966.6808    DYNAMIC     Vx1        1       0:58:01 ago
  20    5000.0088.fe27    DYNAMIC     Po7        1       1 day, 17:49:28 ago
4094    0000.0000.cafe    STATIC      Cpu
4094    5000.0003.3766    DYNAMIC     Vx1        1       0:58:02 ago
4094    5000.00d5.5dc0    DYNAMIC     Vx1        1       0:58:02 ago
Total Mac Addresses for this criterion: 10

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
```
```
dc1-leaf-101#show ip arp vrf tenant-1
Address         Age (sec)  Hardware Addr   Interface
10.2.10.101       0:00:33  5000.0045.abdf  Vlan10, Port-Channel8
10.2.10.201       0:04:02  5000.0088.fe27  Vlan10, Port-Channel7
10.2.20.102             -  0050.7966.6808  Vlan20, Vxlan1
10.2.20.201       0:02:15  5000.0088.fe27  Vlan20, Port-Channel7
```
</details>

<details>
  <summary>проверки leaf-102</summary>
  
```
dc1-leaf-102#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.102.0, local AS number 65102
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-spine-1 ###      10.1.1.2 4 65100          75564     76639    0    0 00:08:20 Estab   3      3
  ### dc1-spine-2 ###      10.1.2.2 4 65100          75460     76332    0    0 00:22:42 Estab   3      3
```
```
dc1-leaf-102#show bgp evpn summary
show vxlan vtep
BGP summary information for VRF default
Router identifier 10.0.102.0, local AS number 65102
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-spine-1 ###      10.1.1.2 4 65100          75571     76645    0    0 00:08:36 Estab   21     21
  ### dc1-spine-2 ###      10.1.2.2 4 65100          75466     76338    0    0 00:22:58 Estab   21     21
```
```
dc1-leaf-102#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP             Tunnel Type(s)
---------------- --------------
10.0.101.0       unicast, flood
10.0.103.0       unicast, flood

Total number of remote VTEPS:  2
```
```
dc1-leaf-102#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface           802.1Q Tag
----------- ---------- ------------ ------------------- ----------
10010       10         static       Port-Channel7       10        
                                    Port-Channel8       10        
                                    Vxlan1              10        
10020       20         static       Port-Channel7       20        
                                    Vxlan1              20        

VNI to dynamic VLAN Mapping for Vxlan1
VNI        VLAN       VRF            Source       
---------- ---------- -------------- ------------ 
4001       4094       tenant-1       evpn         
```
```
dc1-leaf-102#show interfaces vxlan 1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback0 and is active with 10.0.102.0
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is 
    [10, 10010]       [20, 10020]      
  Dynamic VLAN to VNI mapping for 'evpn' is
    [4094, 4001]     
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is 
   [tenant-1, 4001]
  Headend replication flood vtep list is:
    10 10.0.101.0     
    20 10.0.101.0      10.0.103.0     
  Shared Router MAC is 0000.0000.0000
```
```
dc1-leaf-102#show ip route vrf tenant-1

VRF: tenant-1
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

 C        10.2.10.0/24 is directly connected, Vlan10
 B E      10.2.20.102/32 [20/0] via VTEP 10.0.103.0 VNI 4001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 C        10.2.20.0/24 is directly connected, Vlan20
 B E      10.2.30.103/32 [20/0] via VTEP 10.0.103.0 VNI 4001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
```
```
dc1-leaf-102#show bgp evpn route-type imet
BGP routing table information for VRF default
Router identifier 10.0.102.0, local AS number 65102
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.101.0:20 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:20 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:10 imet 10.0.102.0
                                 -                     -       -       0       i
 * >      RD: 10.0.102.0:20 imet 10.0.102.0
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.103.0:20 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:20 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:30 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:30 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
```
```
dc1-leaf-102#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.0.102.0, local AS number 65102
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.103.0:20 mac-ip 0050.7966.6808
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:20 mac-ip 0050.7966.6808
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:20 mac-ip 0050.7966.6808 10.2.20.102
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:20 mac-ip 0050.7966.6808 10.2.20.102
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:30 mac-ip 0050.7966.6809
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:30 mac-ip 0050.7966.6809
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:30 mac-ip 0050.7966.6809 10.2.30.103
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:30 mac-ip 0050.7966.6809 10.2.30.103
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.101.0:10 mac-ip 5000.0045.abdf
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 mac-ip 5000.0045.abdf
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:10 mac-ip 5000.0045.abdf
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:10 mac-ip 5000.0045.abdf 10.2.10.101
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 mac-ip 5000.0045.abdf 10.2.10.101
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:10 mac-ip 5000.0045.abdf 10.2.10.101
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:10 mac-ip 5000.0088.fe27
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 mac-ip 5000.0088.fe27
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.101.0:20 mac-ip 5000.0088.fe27
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:20 mac-ip 5000.0088.fe27
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:10 mac-ip 5000.0088.fe27
                                 -                     -       -       0       i
 * >      RD: 10.0.102.0:20 mac-ip 5000.0088.fe27
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:10 mac-ip 5000.0088.fe27 10.2.10.201
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 mac-ip 5000.0088.fe27 10.2.10.201
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:10 mac-ip 5000.0088.fe27 10.2.10.201
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:20 mac-ip 5000.0088.fe27 10.2.20.201
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:20 mac-ip 5000.0088.fe27 10.2.20.201
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:20 mac-ip 5000.0088.fe27 10.2.20.201
```
```                                 -                     -       -       0       i
dc1-leaf-102#show bgp evpn route-type auto-discovery
BGP routing table information for VRF default
Router identifier 10.0.102.0, local AS number 65102
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.101.0:10 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.101.0:20 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:20 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:10 auto-discovery 0 0000:0000:0101:0007:0000
                                 -                     -       -       0       i
 * >      RD: 10.0.102.0:20 auto-discovery 0 0000:0000:0101:0007:0000
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:10 auto-discovery 0 0000:0000:0101:0008:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 auto-discovery 0 0000:0000:0101:0008:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:10 auto-discovery 0 0000:0000:0101:0008:0000
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:1 auto-discovery 0000:0000:0101:0008:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:1 auto-discovery 0000:0000:0101:0008:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:1 auto-discovery 0000:0000:0101:0008:0000
                                 -                     -       -       0       i
```
```
dc1-leaf-102#show bgp evpn route-type ethernet-segment
BGP routing table information for VRF default
Router identifier 10.0.102.0, local AS number 65102
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.101.0:1 ethernet-segment 0000:0000:0101:0007:0000 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:1 ethernet-segment 0000:0000:0101:0007:0000 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:1 ethernet-segment 0000:0000:0101:0007:0000 10.0.102.0
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:1 ethernet-segment 0000:0000:0101:0008:0000 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:1 ethernet-segment 0000:0000:0101:0008:0000 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:1 ethernet-segment 0000:0000:0101:0008:0000 10.0.102.0
                                 -                     -       -       0       i
```
```
dc1-leaf-102#show bgp evpn route-type ip-prefix ipv4 
BGP routing table information for VRF default
Router identifier 10.0.102.0, local AS number 65102
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
```
```
dc1-leaf-102#show bgp evpn instance
EVPN instance: VLAN 10
  Route distinguisher: 0:0
  Route target import: Route-Target-AS:10010:10
  Route target export: Route-Target-AS:10010:10
  Service interface: VLAN-based
  Local VXLAN IP address: 10.0.102.0
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0000:0000:0101:0008:0000
      Interface: Port-Channel8
      Mode: all-active
      State: up
      ES-Import RT: 00:00:01:01:00:08
      DF election algorithm: modulus
      Designated forwarder: 10.0.101.0
      Non-Designated forwarder: 10.0.102.0
    ESI: 0000:0000:0101:0007:0000
      Interface: Port-Channel7
      Mode: all-active
      State: up
      ES-Import RT: 00:00:01:01:00:07
      DF election algorithm: modulus
      Designated forwarder: 10.0.101.0
      Non-Designated forwarder: 10.0.102.0
EVPN instance: VLAN 20
  Route distinguisher: 0:0
  Route target import: Route-Target-AS:10020:20
  Route target export: Route-Target-AS:10020:20
  Service interface: VLAN-based
  Local VXLAN IP address: 10.0.102.0
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0000:0000:0101:0007:0000
      Interface: Port-Channel7
      Mode: all-active
      State: up
      ES-Import RT: 00:00:01:01:00:07
      DF election algorithm: modulus
      Designated forwarder: 10.0.101.0
      Non-Designated forwarder: 10.0.102.0
```
```
dc1-leaf-102#show port-channel dense 

                 Flags                                                         
------------------------ ---------------------------- -------------------------
  a - LACP Active          p - LACP Passive           * - static fallback      
  F - Fallback enabled     f - Fallback configured    ^ - individual fallback  
  U - In Use               D - Down                                            
  + - In-Sync              - - Out-of-Sync            i - incompatible with agg
  P - bundled in Po        s - suspended              G - Aggregable           
  I - Individual           S - ShortTimeout           w - wait for agg         
  E - Inactive. The number of configured port channels exceeds the config limit
   M - Exceeds maximum weight

Number of channels in use: 2
Number of aggregators: 2

   Port-Channel       Protocol    Ports    
------------------ -------------- ---------
   Po7(U)             LACP(a)     Et7(PG+) 
   Po8(U)             LACP(a)     Et8(PG+) 
```
```
dc1-leaf-102#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  20  0050.7966.6808  EVPN      Vx1  10.0.103.0       1       1:02:12 ago
  20  5000.0088.fe27  EVPN      Vx1  0.0.0.0          1       0:57:55 ago
4094  5000.0003.3766  EVPN      Vx1  10.0.103.0       1       1:02:12 ago
4094  5000.0072.8b31  EVPN      Vx1  10.0.101.0       1       0:57:55 ago
Total Remote Mac Addresses for this criterion: 4
```
```
dc1-leaf-102#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    0000.0000.cafe    STATIC      Cpu
  10    5000.0045.abdf    DYNAMIC     Po8        2       0:03:04 ago
  10    5000.0088.fe27    DYNAMIC     Po7        2       0:03:04 ago
  20    0000.0000.cafe    STATIC      Cpu
  20    0050.7966.6808    DYNAMIC     Vx1        1       1:02:19 ago
  20    5000.0088.fe27    DYNAMIC     Po7        1       0:58:02 ago
4094    0000.0000.cafe    STATIC      Cpu
4094    5000.0003.3766    DYNAMIC     Vx1        1       1:02:19 ago
4094    5000.0072.8b31    DYNAMIC     Vx1        1       0:58:03 ago
Total Mac Addresses for this criterion: 9

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
```
```
dc1-leaf-102#show ip arp vrf tenant-1
Address         Age (sec)  Hardware Addr   Interface
10.2.10.101             -  5000.0045.abdf  Vlan10, Port-Channel8
10.2.10.201             -  5000.0088.fe27  Vlan10, Port-Channel7
10.2.20.102             -  0050.7966.6808  Vlan20, Vxlan1
10.2.20.201             -  5000.0088.fe27  Vlan20, Port-Channel7
```
</details>

<details>
  <summary>проверки leaf-103</summary>
  
```
dc1-leaf-103#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.103.0, local AS number 65103
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-spine-1 ###      10.1.1.4 4 65100          63367     63493    0    0 00:08:20 Estab   3      3
  ### dc1-spine-2 ###      10.1.2.4 4 65100          63332     63322    0    0 00:22:42 Estab   3      3
```
```
dc1-leaf-103#show bgp evpn summary
show vxlan vtep
BGP summary information for VRF default
Router identifier 10.0.103.0, local AS number 65103
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-spine-1 ###      10.1.1.4 4 65100          63373     63499    0   19 00:08:36 Estab   29     29
  ### dc1-spine-2 ###      10.1.2.4 4 65100          63339     63328    0    0 00:22:58 Estab   29     29
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
dc1-leaf-103#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10020       20         static       Ethernet7       untagged  
                                    Vxlan1          20        
10030       30         static       Ethernet8       untagged  
                                    Vxlan1          30        

VNI to dynamic VLAN Mapping for Vxlan1
VNI        VLAN       VRF            Source       
---------- ---------- -------------- ------------ 
4001       4094       tenant-1       evpn         
```
```
dc1-leaf-103#show interfaces vxlan 1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback0 and is active with 10.0.103.0
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is 
    [20, 10020]       [30, 10030]      
  Dynamic VLAN to VNI mapping for 'evpn' is
    [4094, 4001]     
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is 
   [tenant-1, 4001]
  Headend replication flood vtep list is:
    20 10.0.102.0      10.0.101.0     
  Shared Router MAC is 0000.0000.0000
```
```
dc1-leaf-103#show ip route vrf tenant-1

VRF: tenant-1
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

 B E      10.2.10.101/32 [20/0] via VTEP 10.0.102.0 VNI 4001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
                                via VTEP 10.0.101.0 VNI 4001 router-mac 50:00:00:72:8b:31 local-interface Vxlan1
 B E      10.2.10.201/32 [20/0] via VTEP 10.0.102.0 VNI 4001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
                                via VTEP 10.0.101.0 VNI 4001 router-mac 50:00:00:72:8b:31 local-interface Vxlan1
 B E      10.2.20.201/32 [20/0] via VTEP 10.0.102.0 VNI 4001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
                                via VTEP 10.0.101.0 VNI 4001 router-mac 50:00:00:72:8b:31 local-interface Vxlan1
 C        10.2.20.0/24 is directly connected, Vlan20
 C        10.2.30.0/24 is directly connected, Vlan30
```
```
dc1-leaf-103#show bgp evpn route-type imet
BGP routing table information for VRF default
Router identifier 10.0.103.0, local AS number 65103
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.101.0:20 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:20 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:10 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.103.0:20 imet 10.0.103.0
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:30 imet 10.0.103.0
```
```                                 -                     -       -       0       i
dc1-leaf-103#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.0.103.0, local AS number 65103
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.103.0:20 mac-ip 0050.7966.6808
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:20 mac-ip 0050.7966.6808 10.2.20.102
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:30 mac-ip 0050.7966.6809
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:30 mac-ip 0050.7966.6809 10.2.30.103
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:10 mac-ip 5000.0045.abdf
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 mac-ip 5000.0045.abdf
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:10 mac-ip 5000.0045.abdf
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 mac-ip 5000.0045.abdf
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.101.0:10 mac-ip 5000.0045.abdf 10.2.10.101
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 mac-ip 5000.0045.abdf 10.2.10.101
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:10 mac-ip 5000.0045.abdf 10.2.10.101
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 mac-ip 5000.0045.abdf 10.2.10.101
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.101.0:10 mac-ip 5000.0088.fe27
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 mac-ip 5000.0088.fe27
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.101.0:20 mac-ip 5000.0088.fe27
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:20 mac-ip 5000.0088.fe27
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:10 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:20 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.101.0:10 mac-ip 5000.0088.fe27 10.2.10.201
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 mac-ip 5000.0088.fe27 10.2.10.201
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:10 mac-ip 5000.0088.fe27 10.2.10.201
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 mac-ip 5000.0088.fe27 10.2.10.201
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.101.0:20 mac-ip 5000.0088.fe27 10.2.20.201
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:20 mac-ip 5000.0088.fe27 10.2.20.201
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:20 mac-ip 5000.0088.fe27 10.2.20.201
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 mac-ip 5000.0088.fe27 10.2.20.201
                                 10.0.102.0            -       100     0       65100 65102 i
```
```
dc1-leaf-103#show bgp evpn route-type auto-discovery
BGP routing table information for VRF default
Router identifier 10.0.103.0, local AS number 65103
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.101.0:10 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.101.0:20 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:20 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:10 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:20 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.101.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.101.0:10 auto-discovery 0 0000:0000:0101:0008:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 auto-discovery 0 0000:0000:0101:0008:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:10 auto-discovery 0 0000:0000:0101:0008:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 auto-discovery 0 0000:0000:0101:0008:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.101.0:1 auto-discovery 0000:0000:0101:0008:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:1 auto-discovery 0000:0000:0101:0008:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:1 auto-discovery 0000:0000:0101:0008:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:1 auto-discovery 0000:0000:0101:0008:0000
                                 10.0.102.0            -       100     0       65100 65102 i
```
```
dc1-leaf-103#show bgp evpn route-type ethernet-segment
BGP routing table information for VRF default
Router identifier 10.0.103.0, local AS number 65103
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.101.0:1 ethernet-segment 0000:0000:0101:0007:0000 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:1 ethernet-segment 0000:0000:0101:0007:0000 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:1 ethernet-segment 0000:0000:0101:0007:0000 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:1 ethernet-segment 0000:0000:0101:0007:0000 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.101.0:1 ethernet-segment 0000:0000:0101:0008:0000 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:1 ethernet-segment 0000:0000:0101:0008:0000 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:1 ethernet-segment 0000:0000:0101:0008:0000 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:1 ethernet-segment 0000:0000:0101:0008:0000 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
```
```
dc1-leaf-103#show bgp evpn route-type ip-prefix ipv4 
BGP routing table information for VRF default
Router identifier 10.0.103.0, local AS number 65103
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
```
```
dc1-leaf-103#show bgp evpn instance
EVPN instance: VLAN 20
  Route distinguisher: 0:0
  Route target import: Route-Target-AS:10020:20
  Route target export: Route-Target-AS:10020:20
  Service interface: VLAN-based
  Local VXLAN IP address: 10.0.103.0
  VXLAN: enabled
  MPLS: disabled
EVPN instance: VLAN 30
  Route distinguisher: 0:0
  Route target import: Route-Target-AS:10030:30
  Route target export: Route-Target-AS:10030:30
  Service interface: VLAN-based
  Local VXLAN IP address: 10.0.103.0
  VXLAN: enabled
  MPLS: disabled
```
```
dc1-leaf-103#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  20  5000.0088.fe27  EVPN      Vx1  10.0.101.0       2       0:57:55 ago
                                     10.0.102.0     
4094  5000.0072.8b31  EVPN      Vx1  10.0.101.0       1       0:57:55 ago
4094  5000.00d5.5dc0  EVPN      Vx1  10.0.102.0       1       11:24:19 ago
Total Remote Mac Addresses for this criterion: 3
```
```
dc1-leaf-103#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
   1    0000.0000.cafe    STATIC      Cpu
  20    0000.0000.cafe    STATIC      Cpu
  20    0050.7966.6808    DYNAMIC     Et7        1       1 day, 18:34:42 ago
  20    5000.0088.fe27    DYNAMIC     Vx1        2       0:58:02 ago
  30    0000.0000.cafe    STATIC      Cpu
  30    0050.7966.6809    DYNAMIC     Et8        1       1 day, 18:34:06 ago
4094    0000.0000.cafe    STATIC      Cpu
4094    5000.0072.8b31    DYNAMIC     Vx1        1       0:58:03 ago
4094    5000.00d5.5dc0    DYNAMIC     Vx1        1       11:24:26 ago
Total Mac Addresses for this criterion: 9

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
```
```
dc1-leaf-103#show ip arp vrf tenant-1
Address         Age (sec)  Hardware Addr   Interface
10.2.20.102       0:03:04  0050.7966.6808  Vlan20, Ethernet7
10.2.20.201             -  5000.0088.fe27  Vlan20, Vxlan1
10.2.30.103       0:03:55  0050.7966.6809  Vlan30, Ethernet8
dc1-leaf-103#
```
</details>

<details>
  <summary>проверки с client-101</summary>
  
```
client-101#show port-channel dense 

                 Flags                                                         
------------------------ ---------------------------- -------------------------
  a - LACP Active          p - LACP Passive           * - static fallback      
  F - Fallback enabled     f - Fallback configured    ^ - individual fallback  
  U - In Use               D - Down                                            
  + - In-Sync              - - Out-of-Sync            i - incompatible with agg
  P - bundled in Po        s - suspended              G - Aggregable           
  I - Individual           S - ShortTimeout           w - wait for agg         
  E - Inactive. The number of configured port channels exceeds the config limit
   M - Exceeds maximum weight

Number of channels in use: 1
Number of aggregators: 1

   Port-Channel       Protocol    Ports             
------------------ -------------- ------------------
   Po8(U)             LACP(a)     Et1(PG+) Et2(PG+) 
```
```
client-101#ping 10.2.10.101
PING 10.2.10.101 (10.2.10.101) 72(100) bytes of data.
80 bytes from 10.2.10.101: icmp_seq=1 ttl=64 time=0.711 ms
80 bytes from 10.2.10.101: icmp_seq=2 ttl=64 time=0.124 ms
80 bytes from 10.2.10.101: icmp_seq=3 ttl=64 time=0.330 ms
80 bytes from 10.2.10.101: icmp_seq=4 ttl=64 time=0.128 ms
80 bytes from 10.2.10.101: icmp_seq=5 ttl=64 time=0.119 ms

--- 10.2.10.101 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 6ms
rtt min/avg/max/mdev = 0.119/0.282/0.711/0.229 ms, ipg/ewma 1.680/0.487 ms

client-101#ping 10.2.10.201
PING 10.2.10.201 (10.2.10.201) 72(100) bytes of data.
80 bytes from 10.2.10.201: icmp_seq=1 ttl=64 time=185 ms
80 bytes from 10.2.10.201: icmp_seq=2 ttl=64 time=182 ms
80 bytes from 10.2.10.201: icmp_seq=3 ttl=64 time=183 ms
80 bytes from 10.2.10.201: icmp_seq=4 ttl=64 time=195 ms
80 bytes from 10.2.10.201: icmp_seq=5 ttl=64 time=197 ms

--- 10.2.10.201 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 46ms
rtt min/avg/max/mdev = 182.593/188.963/197.019/6.180 ms, pipe 5, ipg/ewma 11.587/187.870 ms

client-101#ping 10.2.20.102
PING 10.2.20.102 (10.2.20.102) 72(100) bytes of data.
80 bytes from 10.2.20.102: icmp_seq=1 ttl=62 time=361 ms
80 bytes from 10.2.20.102: icmp_seq=2 ttl=62 time=354 ms
80 bytes from 10.2.20.102: icmp_seq=3 ttl=62 time=349 ms
80 bytes from 10.2.20.102: icmp_seq=4 ttl=62 time=349 ms
80 bytes from 10.2.20.102: icmp_seq=5 ttl=62 time=344 ms

--- 10.2.20.102 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 45ms
rtt min/avg/max/mdev = 344.280/351.886/361.781/5.884 ms, pipe 5, ipg/ewma 11.482/356.453 ms

client-101#ping 10.2.20.201 
PING 10.2.20.201 (10.2.20.201) 72(100) bytes of data.
80 bytes from 10.2.20.201: icmp_seq=1 ttl=63 time=99.2 ms
80 bytes from 10.2.20.201: icmp_seq=2 ttl=63 time=85.8 ms
80 bytes from 10.2.20.201: icmp_seq=3 ttl=63 time=86.7 ms
80 bytes from 10.2.20.201: icmp_seq=4 ttl=63 time=86.3 ms
80 bytes from 10.2.20.201: icmp_seq=5 ttl=63 time=82.1 ms

--- 10.2.20.201 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 49ms
rtt min/avg/max/mdev = 82.133/88.078/99.297/5.845 ms, pipe 5, ipg/ewma 12.298/93.410 ms

client-101#ping 10.2.30.103
PING 10.2.30.103 (10.2.30.103) 72(100) bytes of data.
80 bytes from 10.2.30.103: icmp_seq=1 ttl=62 time=134 ms
80 bytes from 10.2.30.103: icmp_seq=2 ttl=62 time=126 ms
80 bytes from 10.2.30.103: icmp_seq=3 ttl=62 time=121 ms
80 bytes from 10.2.30.103: icmp_seq=4 ttl=62 time=117 ms
80 bytes from 10.2.30.103: icmp_seq=5 ttl=62 time=112 ms

--- 10.2.30.103 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 50ms
rtt min/avg/max/mdev = 112.438/122.558/134.918/7.798 ms, pipe 5, ipg/ewma 12.735/128.199 ms


```

</details>

<details>
  <summary>проверки с client-102</summary>
  
```
client-102> ping 10.2.10.101
84 bytes from 10.2.10.101 icmp_seq=1 ttl=62 time=209.416 ms
84 bytes from 10.2.10.101 icmp_seq=2 ttl=62 time=108.171 ms
84 bytes from 10.2.10.101 icmp_seq=3 ttl=62 time=77.240 ms
84 bytes from 10.2.10.101 icmp_seq=4 ttl=62 time=99.356 ms
84 bytes from 10.2.10.101 icmp_seq=5 ttl=62 time=103.661 ms

client-102> ping 10.2.10.201
84 bytes from 10.2.10.201 icmp_seq=1 ttl=62 time=242.277 ms
84 bytes from 10.2.10.201 icmp_seq=2 ttl=62 time=120.746 ms
84 bytes from 10.2.10.201 icmp_seq=3 ttl=62 time=93.247 ms
84 bytes from 10.2.10.201 icmp_seq=4 ttl=62 time=95.475 ms
84 bytes from 10.2.10.201 icmp_seq=5 ttl=62 time=126.226 ms

client-102> ping 10.2.20.102
10.2.20.102 icmp_seq=1 ttl=64 time=0.001 ms
10.2.20.102 icmp_seq=2 ttl=64 time=0.001 ms
10.2.20.102 icmp_seq=3 ttl=64 time=0.001 ms
10.2.20.102 icmp_seq=4 ttl=64 time=0.001 ms
10.2.20.102 icmp_seq=5 ttl=64 time=0.001 ms

client-102> ping 10.2.20.201
84 bytes from 10.2.20.201 icmp_seq=1 ttl=64 time=123.307 ms
84 bytes from 10.2.20.201 icmp_seq=2 ttl=64 time=58.950 ms
84 bytes from 10.2.20.201 icmp_seq=3 ttl=64 time=60.321 ms
84 bytes from 10.2.20.201 icmp_seq=4 ttl=64 time=60.240 ms
84 bytes from 10.2.20.201 icmp_seq=5 ttl=64 time=50.270 ms

client-102> ping 10.2.30.103
84 bytes from 10.2.30.103 icmp_seq=1 ttl=63 time=36.155 ms
84 bytes from 10.2.30.103 icmp_seq=2 ttl=63 time=19.373 ms
84 bytes from 10.2.30.103 icmp_seq=3 ttl=63 time=18.780 ms
84 bytes from 10.2.30.103 icmp_seq=4 ttl=63 time=19.534 ms
84 bytes from 10.2.30.103 icmp_seq=5 ttl=63 time=26.457 ms

```
</details>


<details>
  <summary>проверки с client-103</summary>
  
```
client-103> ping 10.2.10.101
84 bytes from 10.2.10.101 icmp_seq=1 ttl=62 time=199.095 ms
84 bytes from 10.2.10.101 icmp_seq=2 ttl=62 time=52.607 ms
84 bytes from 10.2.10.101 icmp_seq=3 ttl=62 time=45.132 ms
84 bytes from 10.2.10.101 icmp_seq=4 ttl=62 time=64.842 ms
84 bytes from 10.2.10.101 icmp_seq=5 ttl=62 time=58.919 ms

client-103> ping 10.2.10.201
84 bytes from 10.2.10.201 icmp_seq=1 ttl=62 time=246.439 ms
84 bytes from 10.2.10.201 icmp_seq=2 ttl=62 time=119.702 ms
84 bytes from 10.2.10.201 icmp_seq=3 ttl=62 time=91.396 ms
84 bytes from 10.2.10.201 icmp_seq=4 ttl=62 time=94.699 ms
84 bytes from 10.2.10.201 icmp_seq=5 ttl=62 time=128.248 ms

client-103> ping 10.2.20.102
84 bytes from 10.2.20.102 icmp_seq=1 ttl=63 time=33.817 ms
84 bytes from 10.2.20.102 icmp_seq=2 ttl=63 time=14.744 ms
84 bytes from 10.2.20.102 icmp_seq=3 ttl=63 time=19.535 ms
84 bytes from 10.2.20.102 icmp_seq=4 ttl=63 time=15.289 ms
84 bytes from 10.2.20.102 icmp_seq=5 ttl=63 time=46.897 ms

client-103> ping 10.2.20.201
84 bytes from 10.2.20.201 icmp_seq=1 ttl=62 time=120.183 ms
84 bytes from 10.2.20.201 icmp_seq=2 ttl=62 time=67.048 ms
84 bytes from 10.2.20.201 icmp_seq=3 ttl=62 time=55.707 ms
84 bytes from 10.2.20.201 icmp_seq=4 ttl=62 time=71.077 ms
84 bytes from 10.2.20.201 icmp_seq=5 ttl=62 time=118.270 ms

client-103> ping 10.2.30.103
10.2.30.103 icmp_seq=1 ttl=64 time=0.001 ms
10.2.30.103 icmp_seq=2 ttl=64 time=0.001 ms
10.2.30.103 icmp_seq=3 ttl=64 time=0.001 ms
10.2.30.103 icmp_seq=4 ttl=64 time=0.001 ms
10.2.30.103 icmp_seq=5 ttl=64 time=0.001 ms
```
</details>


<details>
  <summary>проверки с server-201</summary>
  
```
server-201#show port-channel dense 

                 Flags                                                         
------------------------ ---------------------------- -------------------------
  a - LACP Active          p - LACP Passive           * - static fallback      
  F - Fallback enabled     f - Fallback configured    ^ - individual fallback  
  U - In Use               D - Down                                            
  + - In-Sync              - - Out-of-Sync            i - incompatible with agg
  P - bundled in Po        s - suspended              G - Aggregable           
  I - Individual           S - ShortTimeout           w - wait for agg         
  E - Inactive. The number of configured port channels exceeds the config limit
   M - Exceeds maximum weight

Number of channels in use: 1
Number of aggregators: 1

   Port-Channel       Protocol    Ports             
------------------ -------------- ------------------
   Po7(U)             LACP(a)     Et1(PG+) Et2(PG+) 
```
```   
server-201#ping vrf vlan-10 10.2.10.101
PING 10.2.10.101 (10.2.10.101) 72(100) bytes of data.
80 bytes from 10.2.10.101: icmp_seq=1 ttl=64 time=89.3 ms
80 bytes from 10.2.10.101: icmp_seq=2 ttl=64 time=82.8 ms
80 bytes from 10.2.10.101: icmp_seq=3 ttl=64 time=78.7 ms
80 bytes from 10.2.10.101: icmp_seq=4 ttl=64 time=74.3 ms
80 bytes from 10.2.10.101: icmp_seq=5 ttl=64 time=70.9 ms

--- 10.2.10.101 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 45ms
rtt min/avg/max/mdev = 70.963/79.245/89.378/6.463 ms, pipe 5, ipg/ewma 11.477/83.862 ms

server-201#ping vrf vlan-10 10.2.10.201
PING 10.2.10.201 (10.2.10.201) 72(100) bytes of data.
80 bytes from 10.2.10.201: icmp_seq=1 ttl=64 time=0.835 ms
80 bytes from 10.2.10.201: icmp_seq=2 ttl=64 time=0.122 ms
80 bytes from 10.2.10.201: icmp_seq=3 ttl=64 time=0.127 ms
80 bytes from 10.2.10.201: icmp_seq=4 ttl=64 time=0.159 ms
80 bytes from 10.2.10.201: icmp_seq=5 ttl=64 time=0.163 ms

--- 10.2.10.201 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 13ms
rtt min/avg/max/mdev = 0.122/0.281/0.835/0.277 ms, ipg/ewma 3.255/0.549 ms

server-201#ping vrf vlan-10 10.2.20.102
PING 10.2.20.102 (10.2.20.102) 72(100) bytes of data.
80 bytes from 10.2.20.102: icmp_seq=1 ttl=62 time=178 ms
80 bytes from 10.2.20.102: icmp_seq=2 ttl=62 time=170 ms
80 bytes from 10.2.20.102: icmp_seq=3 ttl=62 time=173 ms
80 bytes from 10.2.20.102: icmp_seq=4 ttl=62 time=176 ms
80 bytes from 10.2.20.102: icmp_seq=5 ttl=62 time=173 ms

--- 10.2.20.102 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 49ms
rtt min/avg/max/mdev = 170.894/174.712/178.947/2.921 ms, pipe 5, ipg/ewma 12.342/176.817 ms

server-201#ping vrf vlan-10 10.2.20.201
PING 10.2.20.201 (10.2.20.201) 72(100) bytes of data.
80 bytes from 10.2.20.201: icmp_seq=1 ttl=63 time=131 ms
80 bytes from 10.2.20.201: icmp_seq=2 ttl=63 time=132 ms
80 bytes from 10.2.20.201: icmp_seq=3 ttl=63 time=129 ms
80 bytes from 10.2.20.201: icmp_seq=4 ttl=63 time=126 ms
80 bytes from 10.2.20.201: icmp_seq=5 ttl=63 time=121 ms

--- 10.2.20.201 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 49ms
rtt min/avg/max/mdev = 121.505/128.298/132.460/3.939 ms, pipe 5, ipg/ewma 12.490/129.566 ms

server-201#ping vrf vlan-10 10.2.30.103
PING 10.2.30.103 (10.2.30.103) 72(100) bytes of data.
80 bytes from 10.2.30.103: icmp_seq=1 ttl=62 time=203 ms
80 bytes from 10.2.30.103: icmp_seq=2 ttl=62 time=217 ms
80 bytes from 10.2.30.103: icmp_seq=3 ttl=62 time=215 ms
80 bytes from 10.2.30.103: icmp_seq=4 ttl=62 time=211 ms
80 bytes from 10.2.30.103: icmp_seq=5 ttl=62 time=206 ms

--- 10.2.30.103 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 46ms
rtt min/avg/max/mdev = 203.225/210.884/217.200/5.280 ms, pipe 5, ipg/ewma 11.673/206.943 ms
```
```
server-201#ping vrf vlan-20 10.2.10.101
PING 10.2.10.101 (10.2.10.101) 72(100) bytes of data.
80 bytes from 10.2.10.101: icmp_seq=1 ttl=63 time=112 ms
80 bytes from 10.2.10.101: icmp_seq=2 ttl=63 time=104 ms
80 bytes from 10.2.10.101: icmp_seq=3 ttl=63 time=99.6 ms
80 bytes from 10.2.10.101: icmp_seq=4 ttl=63 time=94.1 ms
80 bytes from 10.2.10.101: icmp_seq=5 ttl=63 time=93.4 ms

--- 10.2.10.101 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 48ms
rtt min/avg/max/mdev = 93.490/100.820/112.514/7.075 ms, pipe 5, ipg/ewma 12.225/106.207 ms

server-201#ping vrf vlan-20 10.2.10.201 
PING 10.2.10.201 (10.2.10.201) 72(100) bytes of data.
80 bytes from 10.2.10.201: icmp_seq=1 ttl=63 time=195 ms
80 bytes from 10.2.10.201: icmp_seq=2 ttl=63 time=202 ms
80 bytes from 10.2.10.201: icmp_seq=3 ttl=63 time=209 ms
80 bytes from 10.2.10.201: icmp_seq=4 ttl=63 time=206 ms
80 bytes from 10.2.10.201: icmp_seq=5 ttl=63 time=202 ms

--- 10.2.10.201 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 45ms
rtt min/avg/max/mdev = 195.301/203.242/209.585/4.857 ms, pipe 5, ipg/ewma 11.474/199.375 ms

server-201#ping vrf vlan-20 10.2.20.102
PING 10.2.20.102 (10.2.20.102) 72(100) bytes of data.
80 bytes from 10.2.20.102: icmp_seq=1 ttl=64 time=220 ms
80 bytes from 10.2.20.102: icmp_seq=2 ttl=64 time=216 ms
80 bytes from 10.2.20.102: icmp_seq=3 ttl=64 time=212 ms
80 bytes from 10.2.20.102: icmp_seq=4 ttl=64 time=233 ms
80 bytes from 10.2.20.102: icmp_seq=5 ttl=64 time=232 ms

--- 10.2.20.102 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 43ms
rtt min/avg/max/mdev = 212.480/222.999/233.170/8.416 ms, pipe 5, ipg/ewma 10.793/222.051 ms

server-201#ping vrf vlan-20 10.2.20.201
PING 10.2.20.201 (10.2.20.201) 72(100) bytes of data.
80 bytes from 10.2.20.201: icmp_seq=1 ttl=64 time=0.375 ms
80 bytes from 10.2.20.201: icmp_seq=2 ttl=64 time=0.164 ms
80 bytes from 10.2.20.201: icmp_seq=3 ttl=64 time=0.122 ms
80 bytes from 10.2.20.201: icmp_seq=4 ttl=64 time=0.118 ms
80 bytes from 10.2.20.201: icmp_seq=5 ttl=64 time=0.116 ms

--- 10.2.20.201 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 1ms
rtt min/avg/max/mdev = 0.116/0.179/0.375/0.099 ms, ipg/ewma 0.424/0.272 ms

server-201#ping vrf vlan-20 10.2.30.103
PING 10.2.30.103 (10.2.30.103) 72(100) bytes of data.
80 bytes from 10.2.30.103: icmp_seq=1 ttl=62 time=180 ms
80 bytes from 10.2.30.103: icmp_seq=2 ttl=62 time=170 ms
80 bytes from 10.2.30.103: icmp_seq=3 ttl=62 time=166 ms
80 bytes from 10.2.30.103: icmp_seq=4 ttl=62 time=160 ms
80 bytes from 10.2.30.103: icmp_seq=5 ttl=62 time=155 ms

--- 10.2.30.103 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 49ms
rtt min/avg/max/mdev = 155.856/166.806/180.064/8.322 ms, pipe 5, ipg/ewma 12.357/172.861 ms
```
</details>


### Итоговые конфигурации оборудования
- [dc1-spine-1](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-spine-1.txt)
- [dc1-spine-2](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-spine-2.txt)
- [dc1-leaf-101](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-leaf-101.txt)
- [dc1-leaf-102](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-leaf-102.txt)
- [dc1-leaf-103](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-leaf-103.txt)
- [dc1-client-101](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-client-101.txt)
- [dc1-client-102](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-client-102.txt)
- [dc1-client-103](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-client-103.txt)
- [dc1-server-201](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab7/config/dc1-server-201.txt)
---

[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)

