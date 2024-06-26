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
|Тип VNI	|Номер VNI	|Номер VLAN	|Значение RT|
|:-|:-|:-|:-|
|L3VNI	|4001	|4001	|4001:4001|
|L2VNI	|10010	|10	|10010:10|
|L2VNI	|10020	|20	|10020:20|
|L2VNI	|10030	|30	|10030:30|
|L2VNI	|10040	|40	|10040:40|

### Настройка оборудования
_Команды hostname и no switchport не показаны для облегчения восприятия настроек_ \
_Порты 7,8 в vlan 10 настроены на всех leaf для едиообразия, по факту используется только на leaf-103_

<details>
  <summary>Команды для настройки </summary>

- spine-1
```

```
- spine-2
```

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

### Важное примечание

<details>
  <summary>важное примечание</summary>

Без type-5 маршрутов на удаленных leaf (например leaf-103, сидят .103 и .104) нет записи type-2 маршрутов для client-101 \
И отсутствует связь между client-101 (.101), MAC 00:50:79:66:68:07 и client-103 (.103) и client-104 (.104) \
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
Либо можно добавить команду redistribute connected в vrf tenant-1 и тогда появляются type-5 маршруты и такой проблемы нет. \
С учетом того, что данное задание требуется к исполнению до лекции по type-5 маршрутам, решил команду не добавлять.

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

Есть вот такое пояснение у juniper, у Arista похожего не нашел, но судя по тесту ведет себя также.
```
In addition to the host route, an EVPN Type-5 route (for the IRB subnet) is also advertised in accordance with 
the RFC (RFC 9135) for silent hosts (to trigger the gleaning process for such hosts). This is done via the 
‘redistribute direct’ command under the VRF defined in BGP 
```

Т.к. client-ы ведут себя как silent host, то через 5 минут удаляются записи из MAC-таблицы, что в свою очередь 
приводит к удалению EVPN маршрутов type-2 из BGP-таблицы  \
 
Для решения проблемы с silent host выставляется ARP-timeout (250 сек.) меньше время жизни в MAC-таблице (300 сек.). \
Если все же каким-то образом устареет MAC-запись, что придется вручную иницировать ARP-запрос от client (пока не будет добавлены type-5 маршруты).

</details>


### Проверка взаимодействия

<details>
  <summary>вывод ip/mac хостов </summary>
  
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
  <summary>проверки с client-101</summary>
  
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
client-103> ping 10.2.10.101

```
</details>


<details>
  <summary>проверки с client-104</summary>
  
```

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
