# Домашнее задание №6. VxLAN. L3 VNI
[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)
## Задачи
- Настроить маршрутизацию в рамках Overlay между клиентами

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
- блок 10.**2**.0.0/16 - сервисы
 _третий октет - номер VLAN, сети по /24_
 - 10.2.**VLAN**.0/24
- блок 10.**3**.0.0/16 - резерв

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
|dc1-client-102	|Eth0	|10.2.20.102/24	|Клиентская сеть, VLAN 20|
|dc1-client-103	|Eth0	|10.2.30.103/24	|Клиентская сеть, VLAN 30|
|dc1-client-104	|Eth0	|10.2.40.104/24	|Клиентская сеть, VLAN 40|

### Таблица параметров для VRF tenant-1
|Тип VNI	|Номер VNI	|Номер VLAN	|Значение RT| Значение RD|
|:-|:-|:-|:-|:-|
|L3VNI	|4001	|4001	|4001:4001 |RID:4001|
|L2VNI	|10010	|10	|10010:10 |RID:10|
|L2VNI	|10020	|20	|10020:20 |RID:20|
|L2VNI	|10030	|30	|10030:30 |RID:30|
|L2VNI	|10040	|40	|10040:40 |RID:40|

### Важное примечание

<details>
  <summary>Важное примечание</summary>

Без type-5 маршрутов на удаленных leaf (например leaf-103, сидят .103 и .104) нет записи type-5 маршрутов для сети client-101 \
И отсутствует связь между client-101 (.101, MAC 00:50:79:66:68:07) и client-103 (.103) и client-104 (.104) \
Таким образом client-101 ведет себя как silent host.

```
dc1-leaf-103#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.103.0, local AS number 65103
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.103.0:30 mac-ip 0050.7966.6809
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:30 mac-ip 0050.7966.6809 10.2.30.103
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:40 mac-ip 0050.7966.680a
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:40 mac-ip 0050.7966.680a 10.2.40.104
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.103.0:30 imet 10.0.103.0
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:40 imet 10.0.103.0
                                -                     -       -       0       i
```

Если отправить ARP-запрос с client-101 тогда на удаленном leaf-103, появляются записи type-2 маршрутов для client-101 
и трафик начинает ходить. \

```
client-101> ping 10.2.10.44 
host (10.2.10.44) not reachable
client-101> 
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
 * >Ec    RD: 10.0.101.0:10 mac-ip 0050.7966.6807 10.2.10.101
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 mac-ip 0050.7966.6807 10.2.10.101
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.103.0:30 mac-ip 0050.7966.6809
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:30 mac-ip 0050.7966.6809 10.2.30.103
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:40 mac-ip 0050.7966.680a
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:40 mac-ip 0050.7966.680a 10.2.40.104
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.103.0:30 imet 10.0.103.0
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:40 imet 10.0.103.0   
```

Либо можно добавить команду redistribute connected в vrf tenant-1 и тогда появляются type-5 маршруты и такой проблемы нет. \
С учетом того, что данное задание требуется к исполнению до лекции по type-5 маршрутам, решил команду не добавлять.

Есть вот такое пояснение у juniper, у Arista похожего не нашел, но судя по тесту ведет себя также.
```
In addition to the host route, an EVPN Type-5 route (for the IRB subnet) is also advertised in accordance with 
the RFC (RFC 9135) for silent hosts (to trigger the gleaning process for such hosts). This is done via the 
‘redistribute direct’ command under the VRF defined in BGP 
```

Т.к. client-ы ведут себя как silent host, то через 5 минут удаляются записи из MAC-таблицы, что в свою очередь, 
приводит к удалению EVPN-маршрутов type-2 из BGP-таблицы.
 
Для решения проблемы с silent host выставляется ARP-timeout (250 сек.) меньше время жизни в MAC-таблице (300 сек.). \
Если все же каким-то образом устареет MAC-запись, что придется вручную иницировать ARP-запрос от client (пока не будет добавлены type-5 маршруты).

</details>

### Настройка оборудования
_Команды hostname не показаны для облегчения восприятия настроек_ \
_Порты 7,8 настроены на всех leaf для едиообразия, по факту используется только на leaf-103_

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
vrf instance tenant-1
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
   switchport access vlan 10
!
interface Ethernet8
   switchport access vlan 10
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
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
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
vlan 20
   name NET-10.2.20.0/24
!
vrf instance tenant-1
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
   switchport access vlan 20
!
interface Ethernet8
   switchport access vlan 20
!
interface Loopback0
   ip address 10.0.102.0/32
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
      rd 10.0.102.0:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
```
- leaf-103
```
service routing protocols model multi-agent
!
vlan 30
   name NET-10.2.30.0/24
!
vlan 40
   name NET-10.2.40.0/24
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
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
   switchport access vlan 30
!
interface Ethernet8
   switchport access vlan 40
!
interface Loopback0
   ip address 10.0.103.0/32
!
interface Management1
!
interface Vlan30
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.2.30.254/24
!
interface Vlan40
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.2.40.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 30 vni 10030
   vxlan vlan 40 vni 10040
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
   vlan 30
      rd auto
      route-target both 10030:30
      redistribute learned
   !
   vlan 40
      rd auto
      route-target both 10040:40
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

- client-101
```
client-101
set pcname client-101
ip 10.2.10.101/24 10.2.10.254
save
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
set pcname client-104
ip 10.2.40.104/24 10.2.40.254
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

client-101> show arp

00:00:00:00:ca:fe  10.2.10.254 expires in 56 seconds 
```

```
client-102> show ip all

NAME   IP/MASK              GATEWAY           MAC                DNS
client-10.2.20.102/24       10.2.20.254       00:50:79:66:68:08  

client-102> show arp

00:00:00:00:ca:fe  10.2.20.254 expires in 56 seconds
```

```
client-103> show ip all

NAME   IP/MASK              GATEWAY           MAC                DNS
client-10.2.30.103/24       10.2.30.254       00:50:79:66:68:09  

client-103> show arp

00:00:00:00:ca:fe  10.2.30.254 expires in 56 seconds 
```

```
client-104> show ip all

NAME   IP/MASK              GATEWAY           MAC                DNS
client-10.2.40.104/24       10.2.40.254       00:50:79:66:68:0a  

client-104> show arp

00:00:00:00:ca:fe  10.2.40.254 expires in 56 seconds 
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
  10.1.1.1 4 65101           1708      1713    0    0 01:12:11 Estab   3      3
  10.1.1.3 4 65102           1131      1124    0    0 00:47:14 Estab   3      3
  10.1.1.5 4 65103           1800      1805    0    0 01:15:40 Estab   6      6
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
 * >      RD: 10.0.101.0:10 mac-ip 0050.7966.6807 10.2.10.101
                                 10.0.101.0            -       100     0       65101 i
 * >      RD: 10.0.102.0:20 mac-ip 0050.7966.6808
                                 10.0.102.0            -       100     0       65102 i
 * >      RD: 10.0.102.0:20 mac-ip 0050.7966.6808 10.2.20.102
                                 10.0.102.0            -       100     0       65102 i
 * >      RD: 10.0.103.0:30 mac-ip 0050.7966.6809
                                 10.0.103.0            -       100     0       65103 i
 * >      RD: 10.0.103.0:30 mac-ip 0050.7966.6809 10.2.30.103
                                 10.0.103.0            -       100     0       65103 i
 * >      RD: 10.0.103.0:40 mac-ip 0050.7966.680a
                                 10.0.103.0            -       100     0       65103 i
 * >      RD: 10.0.103.0:40 mac-ip 0050.7966.680a 10.2.40.104
                                 10.0.103.0            -       100     0       65103 i
 * >      RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65101 i
 * >      RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65102 i
 * >      RD: 10.0.103.0:30 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65103 i
 * >      RD: 10.0.103.0:40 imet 10.0.103.0
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
  10.1.2.1 4 65101            118       111    0    0 00:04:06 Estab   3      3
  10.1.2.3 4 65102            117       113    0    0 00:04:06 Estab   3      3
  10.1.2.5 4 65103            118       110    0    0 00:04:06 Estab   6      6
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
 * >      RD: 10.0.101.0:10 mac-ip 0050.7966.6807 10.2.10.101
                                 10.0.101.0            -       100     0       65101 i
 * >      RD: 10.0.102.0:20 mac-ip 0050.7966.6808
                                 10.0.102.0            -       100     0       65102 i
 * >      RD: 10.0.102.0:20 mac-ip 0050.7966.6808 10.2.20.102
                                 10.0.102.0            -       100     0       65102 i
 * >      RD: 10.0.103.0:30 mac-ip 0050.7966.6809
                                 10.0.103.0            -       100     0       65103 i
 * >      RD: 10.0.103.0:30 mac-ip 0050.7966.6809 10.2.30.103
                                 10.0.103.0            -       100     0       65103 i
 * >      RD: 10.0.103.0:40 mac-ip 0050.7966.680a
                                 10.0.103.0            -       100     0       65103 i
 * >      RD: 10.0.103.0:40 mac-ip 0050.7966.680a 10.2.40.104
                                 10.0.103.0            -       100     0       65103 i
 * >      RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65101 i
 * >      RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65102 i
 * >      RD: 10.0.103.0:30 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65103 i
 * >      RD: 10.0.103.0:40 imet 10.0.103.0
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
  ### dc1-spine-1 ###      10.1.1.0 4 65100          37252     37298    0    0 01:12:11 Estab   9      9
  ### dc1-spine-2 ###      10.1.2.0 4 65100          37266     37363    0    0 00:04:06 Estab   9      9
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
  Dynamic VLAN to VNI mapping for 'evpn' is
    [4094, 4001]     
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is 
   [tenant-1, 4001]
  Shared Router MAC is 0000.0000.0000
```
```
dc1-leaf-101#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP             Tunnel Type(s)
---------------- --------------
10.0.102.0       unicast       
10.0.103.0       unicast       

Total number of remote VTEPS:  2
```
```
dc1-leaf-101#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10010       10         static       Ethernet7       untagged  
                                    Ethernet8       untagged  
                                    Vxlan1          10        

VNI to dynamic VLAN Mapping for Vxlan1
VNI        VLAN       VRF            Source       
---------- ---------- -------------- ------------ 
4001       4094       tenant-1       evpn         
```
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
 * >      RD: 10.0.101.0:10 mac-ip 0050.7966.6807 10.2.10.101
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:20 mac-ip 0050.7966.6808
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 mac-ip 0050.7966.6808
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:20 mac-ip 0050.7966.6808 10.2.20.102
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 mac-ip 0050.7966.6808 10.2.20.102
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.103.0:30 mac-ip 0050.7966.6809
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:30 mac-ip 0050.7966.6809
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:30 mac-ip 0050.7966.6809 10.2.30.103
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:30 mac-ip 0050.7966.6809 10.2.30.103
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:40 mac-ip 0050.7966.680a
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:40 mac-ip 0050.7966.680a
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:40 mac-ip 0050.7966.680a 10.2.40.104
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:40 mac-ip 0050.7966.680a 10.2.40.104
                                 10.0.103.0            -       100     0       65100 65103 i
 * >      RD: 10.0.101.0:10 imet 10.0.101.0
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.103.0:30 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:30 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:40 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:40 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
```
```
dc1-leaf-101#show vrf
Maximum number of VRFs allowed: 1024
   VRF            Protocols       State         Interfaces   
-------------- --------------- ---------------- -------------
   default        IPv4            routing       Et1, Et2, Lo0
   default        IPv6            no routing                 
   tenant-1       IPv4            routing       Vl10, Vl4094 
   tenant-1       IPv6            no routing    Vl4094              
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
 B E      10.2.20.102/32 [20/0] via VTEP 10.0.102.0 VNI 4001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 B E      10.2.30.103/32 [20/0] via VTEP 10.0.103.0 VNI 4001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 B E      10.2.40.104/32 [20/0] via VTEP 10.0.103.0 VNI 4001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
```
```
dc1-leaf-101#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
4094  5000.0003.3766  EVPN      Vx1  10.0.103.0       1       1:11:27 ago
4094  5000.00d5.5dc0  EVPN      Vx1  10.0.102.0       1       0:46:32 ago
Total Remote Mac Addresses for this criterion: 2
```
```
dc1-leaf-101#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
   1    0000.0000.cafe    STATIC      Cpu
  10    0000.0000.cafe    STATIC      Cpu
  10    0050.7966.6807    DYNAMIC     Et7        1       10:40:46 ago
4094    0000.0000.cafe    STATIC      Cpu
4094    5000.0003.3766    DYNAMIC     Vx1        1       1:11:33 ago
4094    5000.00d5.5dc0    DYNAMIC     Vx1        1       0:46:38 ago
Total Mac Addresses for this criterion: 6

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
```
```
dc1-leaf-101#show ip arp vrf tenant-1
Address         Age (sec)  Hardware Addr   Interface
10.2.10.101       0:03:13  0050.7966.6807  Vlan10, Ethernet7
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
  ### dc1-spine-1 ###      10.1.1.2 4 65100          15117     15132    0    0 00:47:14 Estab   9      9
  ### dc1-spine-2 ###      10.1.2.2 4 65100          15079     15156    0    0 00:04:06 Estab   9      9
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
    [20, 10020]      
  Dynamic VLAN to VNI mapping for 'evpn' is
    [4094, 4001]     
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is 
   [tenant-1, 4001]
  Shared Router MAC is 0000.0000.0000
```
```
dc1-leaf-102#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP             Tunnel Type(s)
---------------- --------------
10.0.101.0       unicast       
10.0.103.0       unicast       

Total number of remote VTEPS:  2
```
```
dc1-leaf-102#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10020       20         static       Ethernet7       untagged  
                                    Ethernet8       untagged  
                                    Vxlan1          20        

VNI to dynamic VLAN Mapping for Vxlan1
VNI        VLAN       VRF            Source       
---------- ---------- -------------- ------------ 
4001       4094       tenant-1       evpn         
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
 * >Ec    RD: 10.0.101.0:10 mac-ip 0050.7966.6807 10.2.10.101
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 mac-ip 0050.7966.6807 10.2.10.101
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:20 mac-ip 0050.7966.6808
                                 -                     -       -       0       i
 * >      RD: 10.0.102.0:20 mac-ip 0050.7966.6808 10.2.20.102
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.103.0:30 mac-ip 0050.7966.6809
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:30 mac-ip 0050.7966.6809
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:30 mac-ip 0050.7966.6809 10.2.30.103
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:30 mac-ip 0050.7966.6809 10.2.30.103
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:40 mac-ip 0050.7966.680a
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:40 mac-ip 0050.7966.680a
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:40 mac-ip 0050.7966.680a 10.2.40.104
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:40 mac-ip 0050.7966.680a 10.2.40.104
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:20 imet 10.0.102.0
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.103.0:30 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:30 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:40 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:40 imet 10.0.103.0
                                 10.0.103.0            -       100     0       65100 65103 i
```
```
dc1-leaf-102#show vrf
Maximum number of VRFs allowed: 1024
   VRF            Protocols       State         Interfaces   
-------------- --------------- ---------------- -------------
   default        IPv4            routing       Et1, Et2, Lo0
   default        IPv6            no routing                 
   tenant-1       IPv4            routing       Vl20, Vl4094 
   tenant-1       IPv6            no routing    Vl4094       
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

 B E      10.2.10.101/32 [20/0] via VTEP 10.0.101.0 VNI 4001 router-mac 50:00:00:72:8b:31 local-interface Vxlan1
 C        10.2.20.0/24 is directly connected, Vlan20
 B E      10.2.30.103/32 [20/0] via VTEP 10.0.103.0 VNI 4001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 B E      10.2.40.104/32 [20/0] via VTEP 10.0.103.0 VNI 4001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
```
```
dc1-leaf-102#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
4094  5000.0003.3766  EVPN      Vx1  10.0.103.0       1       0:46:31 ago
4094  5000.0072.8b31  EVPN      Vx1  10.0.101.0       1       0:46:31 ago
Total Remote Mac Addresses for this criterion: 2
```
```
dc1-leaf-102#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
   1    0000.0000.cafe    STATIC      Cpu
  20    0000.0000.cafe    STATIC      Cpu
  20    0050.7966.6808    DYNAMIC     Et7        1       9:50:24 ago
4094    0000.0000.cafe    STATIC      Cpu
4094    5000.0003.3766    DYNAMIC     Vx1        1       0:46:37 ago
4094    5000.0072.8b31    DYNAMIC     Vx1        1       0:46:37 ago
Total Mac Addresses for this criterion: 6

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
```
```
dc1-leaf-102#show ip arp vrf tenant-1
Address         Age (sec)  Hardware Addr   Interface
10.2.20.102       0:03:01  0050.7966.6808  Vlan20, Ethernet7

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
  ### dc1-spine-1 ###      10.1.1.4 4 65100          37201     37313    0    0 01:15:40 Estab   6      6
  ### dc1-spine-2 ###      10.1.2.4 4 65100          37192     37350    0    0 00:04:06 Estab   6      6
```
```
dc1-leaf-103#show interface vxlan1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback0 and is active with 10.0.103.0
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is 
    [30, 10030]       [40, 10040]      
  Dynamic VLAN to VNI mapping for 'evpn' is
    [4094, 4001]     
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is 
   [tenant-1, 4001]
  Shared Router MAC is 0000.0000.0000
```
```  
dc1-leaf-103#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP             Tunnel Type(s)
---------------- --------------
10.0.101.0       unicast       
10.0.102.0       unicast       

Total number of remote VTEPS:  2
```
```
dc1-leaf-103#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10030       30         static       Ethernet7       untagged  
                                    Vxlan1          30        
10040       40         static       Ethernet8       untagged  
                                    Vxlan1          40        

VNI to dynamic VLAN Mapping for Vxlan1
VNI        VLAN       VRF            Source       
---------- ---------- -------------- ------------ 
4001       4094       tenant-1       evpn         
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
 * >Ec    RD: 10.0.101.0:10 mac-ip 0050.7966.6807 10.2.10.101
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 mac-ip 0050.7966.6807 10.2.10.101
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:20 mac-ip 0050.7966.6808
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 mac-ip 0050.7966.6808
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:20 mac-ip 0050.7966.6808 10.2.20.102
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 mac-ip 0050.7966.6808 10.2.20.102
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.103.0:30 mac-ip 0050.7966.6809
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:30 mac-ip 0050.7966.6809 10.2.30.103
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:40 mac-ip 0050.7966.680a
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:40 mac-ip 0050.7966.680a 10.2.40.104
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:10 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.103.0:30 imet 10.0.103.0
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:40 imet 10.0.103.0
                                 -                     -       -       0       i
```
```
dc1-leaf-103#show vrf
Maximum number of VRFs allowed: 1024
   VRF            Protocols       State         Interfaces        
-------------- --------------- ---------------- ------------------
   default        IPv4            routing       Et1, Et2, Lo0     
   default        IPv6            no routing                      
   tenant-1       IPv4            routing       Vl30, Vl40, Vl4094
   tenant-1       IPv6            no routing    Vl4094 
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

 B E      10.2.10.101/32 [20/0] via VTEP 10.0.101.0 VNI 4001 router-mac 50:00:00:72:8b:31 local-interface Vxlan1
 B E      10.2.20.102/32 [20/0] via VTEP 10.0.102.0 VNI 4001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 C        10.2.30.0/24 is directly connected, Vlan30
 C        10.2.40.0/24 is directly connected, Vlan40
```
```
dc1-leaf-103#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
4094  5000.0072.8b31  EVPN      Vx1  10.0.101.0       1       1:11:28 ago
4094  5000.00d5.5dc0  EVPN      Vx1  10.0.102.0       1       0:46:32 ago
Total Remote Mac Addresses for this criterion: 2
```
```
dc1-leaf-103#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
   1    0000.0000.cafe    STATIC      Cpu
  30    0000.0000.cafe    STATIC      Cpu
  30    0050.7966.6809    DYNAMIC     Et7        1       10:54:16 ago
  40    0000.0000.cafe    STATIC      Cpu
  40    0050.7966.680a    DYNAMIC     Et8        1       10:39:02 ago
4094    0000.0000.cafe    STATIC      Cpu
4094    5000.0072.8b31    DYNAMIC     Vx1        1       1:11:34 ago
4094    5000.00d5.5dc0    DYNAMIC     Vx1        1       0:46:38 ago
Total Mac Addresses for this criterion: 8

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
```
```
dc1-leaf-103#show ip arp vrf tenant-1
Address         Age (sec)  Hardware Addr   Interface
10.2.30.103       0:03:01  0050.7966.6809  Vlan30, Ethernet7
10.2.40.104       0:03:20  0050.7966.680a  Vlan40, Ethernet8
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

client-101> ping 10.2.20.102

84 bytes from 10.2.20.102 icmp_seq=1 ttl=62 time=154.839 ms
84 bytes from 10.2.20.102 icmp_seq=2 ttl=62 time=73.668 ms
84 bytes from 10.2.20.102 icmp_seq=3 ttl=62 time=46.529 ms
84 bytes from 10.2.20.102 icmp_seq=4 ttl=62 time=67.131 ms
84 bytes from 10.2.20.102 icmp_seq=5 ttl=62 time=81.344 ms

client-101> ping 10.2.30.103

84 bytes from 10.2.30.103 icmp_seq=1 ttl=62 time=175.445 ms
84 bytes from 10.2.30.103 icmp_seq=2 ttl=62 time=76.953 ms
84 bytes from 10.2.30.103 icmp_seq=3 ttl=62 time=46.632 ms
84 bytes from 10.2.30.103 icmp_seq=4 ttl=62 time=65.857 ms
84 bytes from 10.2.30.103 icmp_seq=5 ttl=62 time=43.523 ms

client-101> ping 10.2.40.104

84 bytes from 10.2.40.104 icmp_seq=1 ttl=62 time=122.526 ms
84 bytes from 10.2.40.104 icmp_seq=2 ttl=62 time=83.402 ms
84 bytes from 10.2.40.104 icmp_seq=3 ttl=62 time=76.704 ms
84 bytes from 10.2.40.104 icmp_seq=4 ttl=62 time=69.411 ms
84 bytes from 10.2.40.104 icmp_seq=5 ttl=62 time=69.585 ms

```
</details>

<details>
  <summary>проверки с client-102</summary>
  
```
client-102> ping 10.2.10.101

84 bytes from 10.2.10.101 icmp_seq=1 ttl=62 time=243.864 ms
84 bytes from 10.2.10.101 icmp_seq=2 ttl=62 time=68.955 ms
84 bytes from 10.2.10.101 icmp_seq=3 ttl=62 time=65.862 ms
84 bytes from 10.2.10.101 icmp_seq=4 ttl=62 time=57.421 ms
84 bytes from 10.2.10.101 icmp_seq=5 ttl=62 time=44.224 ms

client-102> ping 10.2.20.102

10.2.20.102 icmp_seq=1 ttl=64 time=0.001 ms
10.2.20.102 icmp_seq=2 ttl=64 time=0.001 ms
10.2.20.102 icmp_seq=3 ttl=64 time=0.001 ms
10.2.20.102 icmp_seq=4 ttl=64 time=0.001 ms
10.2.20.102 icmp_seq=5 ttl=64 time=0.001 ms

client-102> ping 10.2.30.103

84 bytes from 10.2.30.103 icmp_seq=1 ttl=62 time=131.150 ms
84 bytes from 10.2.30.103 icmp_seq=2 ttl=62 time=45.025 ms
84 bytes from 10.2.30.103 icmp_seq=3 ttl=62 time=40.115 ms
84 bytes from 10.2.30.103 icmp_seq=4 ttl=62 time=52.775 ms
84 bytes from 10.2.30.103 icmp_seq=5 ttl=62 time=74.272 ms

client-102> ping 10.2.40.104

84 bytes from 10.2.40.104 icmp_seq=1 ttl=62 time=119.506 ms
84 bytes from 10.2.40.104 icmp_seq=2 ttl=62 time=85.337 ms
84 bytes from 10.2.40.104 icmp_seq=3 ttl=62 time=78.016 ms
84 bytes from 10.2.40.104 icmp_seq=4 ttl=62 time=70.140 ms
84 bytes from 10.2.40.104 icmp_seq=5 ttl=62 time=62.562 ms

```
</details>


<details>
  <summary>проверки с client-103</summary>
  
```
client-103> ping 10.2.10.101

84 bytes from 10.2.10.101 icmp_seq=1 ttl=62 time=248.965 ms
84 bytes from 10.2.10.101 icmp_seq=2 ttl=62 time=87.468 ms
84 bytes from 10.2.10.101 icmp_seq=3 ttl=62 time=87.935 ms
84 bytes from 10.2.10.101 icmp_seq=4 ttl=62 time=58.100 ms
84 bytes from 10.2.10.101 icmp_seq=5 ttl=62 time=52.957 ms

client-103> ping 10.2.20.102

84 bytes from 10.2.20.102 icmp_seq=1 ttl=62 time=132.004 ms
84 bytes from 10.2.20.102 icmp_seq=2 ttl=62 time=99.133 ms
84 bytes from 10.2.20.102 icmp_seq=3 ttl=62 time=78.162 ms
84 bytes from 10.2.20.102 icmp_seq=4 ttl=62 time=73.624 ms
84 bytes from 10.2.20.102 icmp_seq=5 ttl=62 time=82.931 ms

client-103> ping 10.2.30.103

10.2.30.103 icmp_seq=1 ttl=64 time=0.001 ms
10.2.30.103 icmp_seq=2 ttl=64 time=0.001 ms
10.2.30.103 icmp_seq=3 ttl=64 time=0.001 ms
10.2.30.103 icmp_seq=4 ttl=64 time=0.001 ms
10.2.30.103 icmp_seq=5 ttl=64 time=0.001 ms

client-103> ping 10.2.40.104

84 bytes from 10.2.40.104 icmp_seq=1 ttl=63 time=26.394 ms
84 bytes from 10.2.40.104 icmp_seq=2 ttl=63 time=17.353 ms
84 bytes from 10.2.40.104 icmp_seq=3 ttl=63 time=33.377 ms
84 bytes from 10.2.40.104 icmp_seq=4 ttl=63 time=16.688 ms
84 bytes from 10.2.40.104 icmp_seq=5 ttl=63 time=32.895 ms
```
</details>


<details>
  <summary>проверки с client-104</summary>
  
```
client-104> ping 10.2.10.101

84 bytes from 10.2.10.101 icmp_seq=1 ttl=62 time=281.859 ms
84 bytes from 10.2.10.101 icmp_seq=2 ttl=62 time=74.524 ms
84 bytes from 10.2.10.101 icmp_seq=3 ttl=62 time=102.432 ms
84 bytes from 10.2.10.101 icmp_seq=4 ttl=62 time=59.291 ms
84 bytes from 10.2.10.101 icmp_seq=5 ttl=62 time=43.779 ms

client-104> ping 10.2.20.102

84 bytes from 10.2.20.102 icmp_seq=1 ttl=62 time=136.927 ms
84 bytes from 10.2.20.102 icmp_seq=2 ttl=62 time=101.100 ms
84 bytes from 10.2.20.102 icmp_seq=3 ttl=62 time=76.385 ms
84 bytes from 10.2.20.102 icmp_seq=4 ttl=62 time=75.321 ms
84 bytes from 10.2.20.102 icmp_seq=5 ttl=62 time=78.267 ms

client-104> ping 10.2.30.103

84 bytes from 10.2.30.103 icmp_seq=1 ttl=63 time=54.592 ms
84 bytes from 10.2.30.103 icmp_seq=2 ttl=63 time=17.700 ms
84 bytes from 10.2.30.103 icmp_seq=3 ttl=63 time=32.468 ms
84 bytes from 10.2.30.103 icmp_seq=4 ttl=63 time=25.441 ms
84 bytes from 10.2.30.103 icmp_seq=5 ttl=63 time=22.662 ms

client-104> ping 10.2.40.104

10.2.40.104 icmp_seq=1 ttl=64 time=0.001 ms
10.2.40.104 icmp_seq=2 ttl=64 time=0.001 ms
10.2.40.104 icmp_seq=3 ttl=64 time=0.001 ms
10.2.40.104 icmp_seq=4 ttl=64 time=0.001 ms
10.2.40.104 icmp_seq=5 ttl=64 time=0.001 ms
```
</details>


### Итоговые конфигурации оборудования
- [dc1-spine-1](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab6/config/dc1-spine-1.txt)
- [dc1-spine-2](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab6/config/dc1-spine-2.txt)
- [dc1-leaf-101](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab6/config/dc1-leaf-101.txt)
- [dc1-leaf-102](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab6/config/dc1-leaf-102.txt)
- [dc1-leaf-103](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab6/config/dc1-leaf-103.txt)
- [dc1-clients](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab6/config/dc1-clients.txt)
---

[**Вернуться к списку домашних заданий**](https://github.com/takmenevag/otus-dc-design/tree/main/labs/)
