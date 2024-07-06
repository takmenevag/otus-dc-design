# Итоговая работа по курсу
[**Вернуться обратно**](https://github.com/takmenevag/otus-dc-design/tree/main/)
## Задачи
- организация отказоустойчивой сети передач данных центра обработки данных с использованием технологии VXLAN

## Решение

### Распределение адресного пространства
<details>
  <summary>Распределение IP-адресов </summary>

#### Описание:
- для DC1, DC2 и DC**I** испольузется блок IP-адресов 10.Х.0.0/12
- для каджого POD в DC1 и DC2 испольузется блок IP-адресов 10.Х.0.0/14
- каждый POD в DC1 и DC2 испольузется блок IP-адресов 10.Х.0.0/14
- в каждом POD выделяется:
	- блок /15 для адресации "шкафонезависимых" сегментов в DC
	- блок /16 для адресации "шкафозависимых" сегментов в DC
	- блок /16 выделяется в резевр
- в блок /16 для POD выделяется:
	- блок /23 для каждого ТШ (до 100 шт.)
	- блок /25 для каждого spine (до 124 leaf на POD)
	- блок /25 для каждого border leaf (+/25 в резерв)
	- блок /23 для loopback интерфейсов сетевого оборудования
- адресация для сервисов и огранизации взаимодействии между ЦОД назначается из блока DCI

---
#### Вернеуровневые IP-блоки
|Блок IP-адресов |Назначение|
|:-				|:-|
|10.0.0.0/8		|Блок сетей|
|10.0.0.0/12	|DC**I**|
|10.16.0.0/12	|DC1|
|10.32.0.0/12	|DC2|
|10.48.0.0/12	|DC резерв|
|10.64.0.0/12	|DC резерв|
|10.80.0.0/12	|DC резерв|
|10.96.0.0/12	|DC резерв|
|10.112.0.0/12	|DC резерв|
|10.128.0.0/12	|резерв|
|...			|...|
|10.240.0.0/12	|резерв|

---
#### IP-блоки DCI
|Блок IP-адресов |Назначение|
|:-				|:-|
|10.0.0.0/12	|DCI|
|10.0.0.0/16	|транспорт|
|10.1.0.0/16	|резерв транспорт|
|10.2.0.0/15	|резерв|
|10.4.0.0/14	|резерв|
|10.8.0.0/14	|сервисы DC-независимые|
|10.12.0.0/14	|резерв сервисы|
|||
|10.8.0.0/14	|DCI сервисы|
|10.8.0.0/16	|сервисы|
|10.9.0.0/16	|резерв|
|10.10.0.0/15	|резерв|
|||	
|10.8.0.0/16	|DCI сервисы|
|10.8.10.0/24	|vlan10|
|10.8.20.0/24	|vlan20|
|10.8.30.0/24	|vlan30|
|10.8.40.0/24	|vlan40|

---
#### IP-блоки DC1
|Блок IP-адресов |Назначение|
|:-				|:-|
|10.16.0.0/12	|DC1|
|10.16.0.0/14	|POD1|
|10.20.0.0/14	|POD2|
|10.24.0.0/14	|резерв|
|10.28.0.0/14	|резерв|
|||
|10.16.0.0/14	|POD1|
|10.16.0.0/16	|Cеть|
|10.17.0.0/16	|ТШ зависимо+резевр|
|10.18.0.0/15	|ТШ независимо+сервисы|
|||
|10.16.0.0/14	|POD1|
|10.16.0.0/16	|Cеть|
|10.17.0.0/16	|ТШ зависимо+резевр|
|10.18.0.0/15	|ТШ независимо сервисы|
---

</details>

### Распределение номеров AS
<details>
  <summary>Таблицы распределения номеров AS</summary>

#### Примечание:
- наименование АСО взята из распределения для 4 байтных номеров AS (leaf > 85 шт.)
- нумерация AS для лабы взята из распределения для 2 байтных номеров AS (leaf < 85 шт.) для облегчения диагностики \
В связи с чем ниже приведены 2 варианта распределения номеров AS
---  
#### Для случая 2xDC/2xPOD или 4xDC/1xPOD, leaf < 85 шт.
|Тип	|Номер AS	|X,DC/POD 	|Y|
|:-		|:-			|:-			|:-|
|sspine	|65x00		|1-4		|-|	
|spine	|65x0y		|1-4		|1-8|
|leaf	|65xyy		|1-4		|11-84|
|boleaf	|65xyy		|1-4		|86-89|
|fw		|65xyy		|1-4		|90-95|
|br		|65xyy		|1-4		|96-99|
|host	|646yy		|1 DC/POD	|0-99|
|host	|647yy		|2 DC/POD	|0-99|
|host	|648yy		|3 DC/POD	|0-99|
|host	|649yy		|4 DC/POD	|0-99|

---
#### Для остальных вариантов DC/POD или leaf > 85 шт.
|AS	|DC	|POD	|ТШ		|Тип	|Номер|
|:-	|:-	|:-		|:-		|:-		|:-|
|42	|Х	|Х		|ХXX	|ХX		|Х|

#### Соответствие типа оборудования и его номера
|Тип| Оборудование|
|:-	|:-|
|0	|host|
|1	|leaf|
|2	|spine|
|3	|sspine|
|4	|fw|
|5	|-|
|6	|-|
|7	|-|
|8	|-|
|9	|br|
---
</details>

### Наимнование и расположение оборудования
<details>
  <summary>Таблица расположения оборудования по ТШ </summary>

#### Примечание:
- в первой таблице приведено расположение оборудования для первых 20 ТШ
- во второй таблице оставлено оборудования для лабы
В связи с чем ниже приведены 2 варианта распределения номеров AS
--- 
#### Расположение оборудования по ТШ с резервированием места (в лабе меньше устройств)
|ТШ	|АСО	|Номер	|Примечание|
|:-	|:-		|:-		|:-|
|1	|кроссы	|-		|-|
|2	|spine	|1		||
|2	|spine	|2		|резерв|
|2	|boleaf	|1		||
|2	|boleaf	|2		|резерв|
|3	|leaf	|1,2	||
|4	|leaf	|1,2	|резерв|
|5	|leaf	|1,2	|резерв|
|6	|leaf	|1,2	|резерв|
|7	|leaf	|1,2	|резерв|
|8	|leaf	|1,2	|резерв|
|9	|fw		|1		||
|10	|br		|1		||
|11	|кроссы	|-		|-|
|12	|spine	|1		||
|12	|spine	|2		|резерв|
|12	|boleaf	|1		||
|12	|boleaf	|2		|резерв|
|13	|leaf	|1,2	||
|14	|leaf	|1,2	|резерв|
|15	|leaf	|1,2	|резерв|
|16	|leaf	|1,2	|резерв|
|17	|leaf	|1,2	|резерв|
|18	|leaf	|1,2	|резерв|
|19	|fw		|1		||
|20	|br		|1		|| 

---
#### Итоговая таблица наименования и расположения </summary>
В лабе взяты AS 65xxx для облегчения диагностики
|ТШ	|Имя для  номер AS	|Оборудование		|Номер AS	|Cокращение |AS лаба|
|:-	|:-					|:-					|:-			|:-			|:-|
|2	|dc1-p1-r002-02-1	|dc1-p1-r002-sp-1	|4211002021	|spine-1	|65101|
|12	|dc1-p1-r012-02-1	|dc1-p1-r012-sp-1	|4211012021	|spine-2	|65101|
|3	|dc1-p1-r003-01-1	|dc1-p1-r003-lf-1	|4211003011	|leaf-1		|65111|
|3	|dc1-p1-r003-01-2	|dc1-p1-r003-lf-2	|4211003012	|leaf-2		|65112|
|13	|dc1-p1-r013-01-1	|dc1-p1-r013-lf-1	|4211013011	|leaf-3		|65113|
|13	|dc1-p1-r013-01-2	|dc1-p1-r013-lf-2	|4211013012	|leaf-4		|65114|
|2	|dc1-p1-r002-01-1	|dc1-p1-r002-blf-1	|4211002011	|boleaf-1	|65188|
|12	|dc1-p1-r012-01-1	|dc1-p1-r012-blf-1	|4211012011	|boleaf-2	|65189|
|9	|dc1-p1-r009-04-1	|dc1-p1-r009-fw-1	|4211009041	|fw-1		|65190|
|19	|dc1-p1-r019-04-1	|dc1-p1-r019-fw-1	|4211019041	|fw-2		|65190|
|10	|dc1-p1-r010-09-1	|dc1-p1-r010-br-1	|4211010091	|br-1		|65196|
|20	|dc1-p1-r020-09-1	|dc1-p1-r020-br-1	|4211020091	|br-2		|65197|
---
</details>

### IP-адресация оборудования и подсетей
<details>
  <summary>Таблица IP-адресации</summary>
  
#### В лабе подписи интерфейсов в таком виде для облегчения просмотра
|Оборудование		|Интерфейс	|IP-адрес			|Назначение|
|:-					|:-			|:-					|:-|
|dc1-p1-r002-sp-1	|Loopback0	|10.16.254.1/32		|-|
|dc1-p1-r002-sp-1	|Eth1		|10.16.250.0/31		|sp1-lf1|
|dc1-p1-r002-sp-1	|Eth2		|10.16.250.2/31		|sp1-lf2|
|dc1-p1-r002-sp-1	|Eth3		|10.16.250.4/31		|sp1-lf3|
|dc1-p1-r002-sp-1	|Eth4		|10.16.250.6/31		|sp1-lf4|
|dc1-p1-r002-sp-1	|Eth5		|10.16.250.120/31	|sp1-blf1|
|dc1-p1-r002-sp-1	|Eth6		|10.16.250.122/31	|sp1-blf2|
| | | | |
|dc1-p1-r012-sp-1	|Loopback0	|10.16.254.2/32 	|-|
|dc1-p1-r012-sp-1	|Eth1		|10.16.251.0/31		|sp2-lf1|
|dc1-p1-r012-sp-1	|Eth2		|10.16.251.2/31		|sp2-lf2|
|dc1-p1-r012-sp-1	|Eth3		|10.16.251.4/31		|sp2-lf3|
|dc1-p1-r012-sp-1	|Eth4		|10.16.251.6/31		|sp2-lf4|
|dc1-p1-r012-sp-1	|Eth5		|10.16.251.120/31	|sp2-blf1|
|dc1-p1-r012-sp-1	|Eth6		|10.16.251.122/31	|sp2-blf2|
| | | | |
|dc1-p1-r003-lf-1	|Loopback0	|10.16.254.11/32 	|-|
|dc1-p1-r003-lf-1	|Eth1		|10.16.250.1/31		|sp1-lf1|
|dc1-p1-r003-lf-1	|Eth2		|10.16.251.1/31		|sp2-lf1|
|dc1-p1-r003-lf-2	|Loopback0	|10.16.254.12/32 	|-|
|dc1-p1-r003-lf-2	|Eth1		|10.16.250.1/31		|sp1-lf2|
|dc1-p1-r003-lf-2	|Eth2		|10.16.251.1/31		|sp2-lf2|
| | | | |
|dc1-p1-r013-lf-1	|Loopback0	|10.16.254.13/32 	|-|
|dc1-p1-r013-lf-1	|Eth1		|10.16.250.1/31		|sp1-lf3|
|dc1-p1-r013-lf-1	|Eth2		|10.16.251.1/31		|sp2-lf3|
|dc1-p1-r013-lf-2	|Loopback0	|10.16.254.14/32 	|-|
|dc1-p1-r013-lf-2	|Eth1		|10.16.250.1/31		|sp1-lf4|
|dc1-p1-r013-lf-2	|Eth2		|10.16.251.1/31		|sp2-lf4|
| | | | |
|dc1-p1-r002-blf-1	|Loopback0	|10.16.254.188/32 	|-|
|dc1-p1-r002-blf-1	|Eth1		|10.16.250.121/31	|sp1-blf1|
|dc1-p1-r002-blf-1	|Eth2		|10.16.251.121/31	|sp2-blf1|
|dc1-p1-r012-blf-1	|Loopback0	|10.16.254.189/32 	|-|
|dc1-p1-r012-blf-1	|Eth1		|10.16.250.123/31	|sp1-blf2|
|dc1-p1-r012-blf-1	|Eth2		|10.16.251.123/31	|sp2-blf2|
| | | | |
|dc1-p1-r003-lf-1	|Po7		|10.2.10.254/24		|Клиентская сеть, VLAN 10|
|dc1-p1-r003-lf-1	|Po8		|10.2.20.254/24		|Клиентская сеть, VLAN 20|
|dc1-p1-r013-lf-1	|Po7		|10.2.10.254/24		|Клиентская сеть, VLAN 10|
|dc1-p1-r013-lf-1	|Po8		|10.2.20.254/24		|Клиентская сеть, VLAN 20|
|dc1-lfaf-103	|Eth7	|10.2.20.254/24	|Клиентская сеть, VLAN 20|
|dc1-lfaf-103	|Eth8	|10.2.30.254/24	|Клиентская сеть, VLAN 30|
| | | | |
|dc1-client-102	|Eth0	|10.2.20.102/24	|Клиентская сеть, VLAN 20|
|dc1-client-103	|Eth0	|10.2.30.103/24	|Клиентская сеть, VLAN 30|
|dc1-client-104	|Po8	|10.2.40.104/24	|Клиентская сеть, VLAN 10|
|dc1-server-201	|Po7	|10.2.10.201/24	|Клиентская сеть, VLAN 10|
|dc1-server-201	|Po7	|10.2.20.201/24	|Клиентская сеть, VLAN 20|
|dc1-server-201	|Po7	|10.2.30.201/24	|Клиентская сеть, VLAN 30|
</details>


### Описание решения Underlay

<details>
  <summary>Описание решения </summary>

#### Описание
В решении используется протокол маршрутизации eBGP со следующими параметрами:
- все spine одного POD в каждом DC размещены в одной AS 65x0y
- каждый leaf размещен в свой AS: leaf-1YY в AS 651YY
- на spine используются динамические peer-group с фильтром по номеру AS и транзитному блоку /25
- на leaf используются статические peer-group
- настроены keepalive-интервал 3 сек, hold time 9 сек.
- настроен maximum-paths равным 8
- настроен BGP routing updates интервал равным 0  (neighbor out-delay, установлен в 0 по умолчанию)
- настроена administrative distance равна 20 (по рекомендации Arista из предоставленной ссылке, возможно из-за iBGP между leaf в паре)
- отключена автоматическая активация BGP AFI/SFI ipv4 unicast (в данной лабе это было не обязательно)
- включен режим multi-agent model (поддежка redistribute в BGP AFI/SFI ipv4 unicast)
- включена аутентификация BGP-соседа
- настроено взаимодействие с протоколом bfd для улучшения сходимости сети
- таймеры bfd выбраны такие, чтобы сессии в EVE-NG флапали реже
</details>

  
### Описание решения VXLAN
<details>
  <summary>Описание решения VXLAN </summary>

В решении используется следующие параметры:
- общие параметры:
	- в overlay используется интерфейс Loopback0 на spine и leaf
	- настроено соседство между spine и leaf для BGP AFI/SFI l2vpn evpn
	- команда neighbor XXX next-hop-unchanged используется для сохранения next-hop-адреса leaf-коммутатора
	- команда redistribute learned используется для анонса MAC-адреса локальных хостов как EVPN type-2 маршрутов
	- команда neighbor XXX send-community extended используется для работы EVPN (импорта, экспорта маршрутов)
	- для маршрутизации трафика в сетевой фабрики используется модель Symmetric IRB
	- механизм ARP Suppression на коммуаторах Arista включен по умолчанию
- маршрутизация клиентских сетей:
	- для настройки шлюза на VTEP используется технология anycast gateway
	- команда ip address virtual используется для задания единого IP-адреса для anycast gateway на всех VTEP, выполняющих функцию шлюза для VLAN
	- команда ip virtual-router mac-address используется для задания единого MAC-адреса для anycast gateway, на всех VTEP, выполняющих функцию шлюза для VLAN
- параметры VNI:
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
  	- для отказоустойчивого подключения хостов используется технология EVPN Multihoming в режиме Active-Active и протокол LACP
	- для возможности огранизации отказоустойчивого подключения хостов к двух разным leaf на leaf настаивается одинаковый lacp system-id
  	- индекc коммутатора для работв EVPN Multihoming назначается от меньшего IP-адреса Loopback0 к большему (в поле IP Address в маршруте EVPN type 4)
  	- коммутатору leaf-101 присвоем индекс 0, а leaf-102 индекс 1
	- для определения Designated Forwarder (DF) использует функция mod - VLAN mod количество leaf (в лабе модель сервиса VLAN-based)
	- в качестве DF для всеx VLAN выбран dc1-leaf-101, т.к. номера VLAN деляться на 2 без остатка (10,20)
	- используется номер нечерного leaf в параметрах ниже:
		- параметр ESI 0000:0000:№ leaf:№ Port-channel:0000
		- параметр ES-Import RT 0000:№ leaf:№ Port-channel (отбрасываются два байта слева и справа в ESI). Формат записи для облегчения понимания
		- парамет lacp system-id 0000.№ leaf.№ Port-channel 
- взаимодействие между VRF (tenant):
	- используется два VRF (tenant), в которых размещены все клиентские подсети
	- в tenant №1 размещены VLAN 10 и 20, в tenant №2 размещены VLAN 30 и 40
	- в VLAN 10 и 20, 30 размещено по одному серверу (server-20X). Серверы реализованы в виде VRF на общей платформе
	- для анонсирования type-5 маршрутов используется команда redistribute connected в каждом VRF секции BGP
	- взаимодействие подсетей из разных VRF, осуществляется через МЭ dc1-fw-199 с использованием технологии VRF-Lite на leaf-103:
		- между МЭ dc1-fw-199 и коммутатором leaf-103 настроены два транзитных сегмента
		- для огранизации отказоустойчивого подключения dc1-fw-199 к leaf-103 используется два канала связи и технология Etherchannel (LACP)
		- на leaf-103 по одному транзитный сегмент помещены в каждый VRF
		- на dc1-fw-199 настроены оба транзитных сегмента без разделения на VRF
	- взаимодействие межу МЭ dc1-fw-199 (AS 65199) и  leaf-103 (AS 65103) осуществляется с использованием протокола eBGP
	- параметры протокола eBGP заданы аналогичными параметрам eBGP для underlay, кроме поддержки extended community
	- на МЭ dc1-fw-199 используется BGP AFI/SFI ipv4 unicast
	- МЭ dc1-fw-199 анонсирует в сторону leaf-103 маршрут по умолчанию
	- коммутатор leaf-103 анонсирует клиентские подсети в сторону МЭ dc1-fw-199 (являющиеся type-5 маршрутами в EVPN)
	- коммутатор leaf-103 не анонсирует маршруты до хостов (/32) в сторону МЭ dc1-fw-199 (являющиеся type-2 маршрутами в EVPN)
	- запрет анонса маршрутов /32 реализуется с ипользованием префикс-листа - анонсировать клиентские подсети (10.2.X.X) с маской не длиннее /31
</details>



### Cхема сети
![Изображение](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab8/scheme/lab8_scheme.PNG "Схема стенда")



### Параметы сети VXLAN
<details>
  <summary>Таблица VRF</summary>
  
_В решении используется два tenant, возможно масштабирование_
|VRF	|Тип VNI |Номер VNI	|Номер VLAN	|Значение RT| Значение RD|
|:-			|:-		|:-		|:-		|:-			|:-|
|tenant-1	|L3VNI	|4001	|4001	|4001:4001	|RID:4001|
|tenant-1	|L2VNI	|10010	|10 	|10010:10	|RID:10|
|tenant-1	|L2VNI	|10020	|20		|10020:20	|RID:20|
| | | | |
|tenant-2	|L3VNI	|4002	|4002	|4002:4002	|RID:4002|
|tenant-2	|L2VNI	|10030	|30 	|10010:30	|RID:30|
|tenant-2	|L2VNI	|10040	|40		|10020:40	|RID:40|

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
   ip address 10.16.250.0/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet2
   description ### sp1-le102 ###
   ip address 10.16.250.2/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet3
   description ### sp1-le103 ###
   ip address 10.16.250.4/31
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
   bgp listen range 10.16.250.0/24 peer-group DC1-LEAF peer-filter PF-DC1-LEAF
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
   ip address 10.16.251.0/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet2
   description ### sp2-le102 ###
   ip address 10.16.251.2/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet3
   description ### sp2-le103 ###
   ip address 10.16.251.4/31
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
   bgp listen range 10.16.251.0/24 peer-group DC1-LEAF peer-filter PF-DC1-LEAF
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
hostname dc1-leaf-101
!
vlan 10
   name NET-10.2.10.0/24
!
vlan 20
   name NET-10.2.20.0/24
!
vlan 30
   name NET-10.2.30.0/24
!
vlan 40
   name NET-10.2.40.0/24
!
vrf instance tenant-1
!
vrf instance tenant-2
!
interface Port-Channel7
   description ### server-201 ###
   switchport trunk allowed vlan 10,20,30
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0000:0101:0007:0000
      route-target import 00:00:01:01:00:07
   lacp system-id 0000.0101.0007
!
interface Port-Channel8
   description ### client-104 ###
   switchport trunk allowed vlan 40
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
   ip address 10.16.250.1/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet2
   description ### sp2-le101 ###
   no switchport
   ip address 10.16.251.1/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet7
   description ### server-201 ###
   channel-group 7 mode active
!
interface Ethernet8
   description ### client-104 ###
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
interface Vlan30
   description ### client ###
   vrf tenant-2
   arp aging timeout 250
   ip address virtual 10.2.30.254/24
!
interface Vlan40
   description ### client ###
   vrf tenant-2
   arp aging timeout 250
   ip address virtual 10.2.40.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vlan 40 vni 10040
   vxlan vrf tenant-1 vni 4001
   vxlan vrf tenant-2 vni 4002
!
ip virtual-router mac-address 00:00:00:00:ca:fe
!
ip routing
ip routing vrf tenant-1
ip routing vrf tenant-2
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
   neighbor 10.16.250.0 peer group DC1-SPINE
   neighbor 10.16.250.0 description ### dc1-spine-1 ###
   neighbor 10.16.251.0 peer group DC1-SPINE
   neighbor 10.16.251.0 description ### dc1-spine-2 ###
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
      rd 10.0.101.0:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
      redistribute connected
   !
   vrf tenant-2
      rd 10.0.101.0:4002
      route-target import evpn 4002:4002
      route-target export evpn 4002:4002
      redistribute connected
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
vlan 30
   name NET-10.2.30.0/24
!
vlan 40
   name NET-10.2.40.0/24
!
vrf instance tenant-1
!
vrf instance tenant-2
!
interface Port-Channel7
   description ### server-201 ###
   switchport trunk allowed vlan 10,20,30
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0000:0101:0007:0000
      route-target import 00:00:01:01:00:07
   lacp system-id 0000.0101.0007
!
interface Port-Channel8
   description ### client-104 ###
   switchport trunk allowed vlan 40
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
   ip address 10.16.250.3/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet2
   description ### sp2-le102 ###
   no switchport
   ip address 10.16.251.3/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet7
   description ### server-201 ###
   channel-group 7 mode active
!
interface Ethernet8
   description ### client-104 ###
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
interface Vlan30
   description ### client ###
   vrf tenant-2
   arp aging timeout 250
   ip address virtual 10.2.30.254/24
!
interface Vlan40
   description ### client ###
   vrf tenant-2
   arp aging timeout 250
   ip address virtual 10.2.40.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vlan 40 vni 10040
   vxlan vrf tenant-1 vni 4001
   vxlan vrf tenant-2 vni 4002
!
ip virtual-router mac-address 00:00:00:00:ca:fe
!
ip routing
ip routing vrf tenant-1
ip routing vrf tenant-2
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
   neighbor 10.16.250.2 peer group DC1-SPINE
   neighbor 10.16.250.2 description ### dc1-spine-1 ###
   neighbor 10.16.251.2 peer group DC1-SPINE
   neighbor 10.16.251.2 description ### dc1-spine-2 ###
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
      rd 10.0.102.0:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
      redistribute connected
   !
   vrf tenant-2
      rd 10.0.102.0:4002
      route-target import evpn 4002:4002
      route-target export evpn 4002:4002
      redistribute connected
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
vlan 4081
   name tenant-1
!
vlan 4082
   name tenant-2
!
vrf instance tenant-1
!
vrf instance tenant-2
!
interface Port-Channel5
   description ### dc1-fw-199 ###
   switchport mode trunk
!
interface Ethernet1
   description ### sp1-le103 ###
   no switchport
   ip address 10.16.250.5/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet2
   description ### sp2-le103 ###
   no switchport
   ip address 10.16.251.5/31
   bfd interval 800 min-rx 800 multiplier 3
!
interface Ethernet5
   description ### dc1-fw199 ###
   channel-group 5 mode active
!
interface Ethernet6
   description ### dc1-fw199 ###
   channel-group 5 mode active
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
   vrf tenant-2
   arp aging timeout 250
   ip address virtual 10.2.30.254/24
!
interface Vlan4081
   description ### tenant-1 ###
   vrf tenant-1
   ip address 10.16.25003.241/29
!
interface Vlan4082
   description ### tenant-2 ###
   vrf tenant-2
   ip address 10.16.25003.249/29
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vrf tenant-1 vni 4001
   vxlan vrf tenant-2 vni 4002
!
ip virtual-router mac-address 00:00:00:00:ca:fe
!
ip routing
ip routing vrf tenant-1
ip routing vrf tenant-2
!
ip prefix-list PL-VXLAN-FW-OUT seq 100 permit 10.2.0.0/16 le 31
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
route-map RM-VXLAN-FW-OUT permit 100
   match ip address prefix-list PL-VXLAN-FW-OUT
!
router bgp 65103
   router-id 10.0.103.0
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-FW peer group
   neighbor DC1-FW remote-as 65199
   neighbor DC1-FW timers 3 9
   neighbor DC1-FW password 7 n5uk4I9QUorSZi6ToxJeeg==
   neighbor DC1-SPINE peer group
   neighbor DC1-SPINE remote-as 65100
   neighbor DC1-SPINE bfd
   neighbor DC1-SPINE timers 3 9
   neighbor DC1-SPINE password 7 txq0MZ/aCqwJ+sp2WtntdQ==
   neighbor DC1-SPINE send-community extended
   neighbor 10.16.250.4 peer group DC1-SPINE
   neighbor 10.16.250.4 description ### dc1-spine-1 ###
   neighbor 10.16.251.4 peer group DC1-SPINE
   neighbor 10.16.251.4 description ### dc1-spine-2 ###
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
      neighbor DC1-FW activate
      neighbor DC1-SPINE activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
   !
   vrf tenant-1
      rd 10.0.103.0:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
      neighbor 10.16.25003.244 peer group DC1-FW
      neighbor 10.16.25003.244 description ### dc1-fw-199 tenant-1 ###
      neighbor 10.16.25003.244 route-map RM-VXLAN-FW-OUT out
      redistribute connected
   !
   vrf tenant-2
      rd 10.0.103.0:4002
      route-target import evpn 4002:4002
      route-target export evpn 4002:4002
      neighbor 10.16.25003.252 peer group DC1-FW
      neighbor 10.16.25003.252 description ### dc1-fw-199 tenant-2 ###
      neighbor 10.16.25003.252 route-map RM-VXLAN-FW-OUT out
      redistribute connected
```

- fw-199
```
service routing protocols model ribd
!
vlan 4081
   name tenant-1
!
vlan 4082
   name tenant-2
!
interface Port-Channel1
   description ### leaf-103 ###
   switchport mode trunk
!
interface Ethernet1
   description ### leaf-103 ###
   channel-group 1 mode active
!
interface Ethernet2
   description ### leaf-103 ###
   channel-group 1 mode active
!
interface Loopback0
   ip address 10.0.199.0/32
!
interface Vlan4081
   description ### tenant-1 ###
   ip address 10.16.25003.244/29
!
interface Vlan4082
   description ### tenant-2 ###
   ip address 10.16.25003.252/29
!
ip routing
!
router bgp 65199
   router-id 10.0.199.0
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-LEAF-103 peer group
   neighbor DC1-LEAF-103 remote-as 65103
   neighbor DC1-LEAF-103 timers 3 9
   neighbor DC1-LEAF-103 password 7 G6bJhHqcFTKz3Im1mSWygg==
   neighbor DC1-LEAF-103 default-originate always
   neighbor 10.16.25003.241 peer group DC1-LEAF-103
   neighbor 10.16.25003.241 description ### dc1-leaf-103 tenant-1 ###
   neighbor 10.16.25003.249 peer group DC1-LEAF-103
   neighbor 10.16.25003.249 description ### dc1-leaf-103 tenant-2 ###
```

- server
```
service routing protocols model ribd
!
vlan 10
   name NET-10.2.10.0/24
!
vlan 20
   name NET-10.2.20.0/24
!
vlan 30
   name NET-10.2.30.0/24
!
vrf instance vlan-10
!
vrf instance vlan-20
!
vrf instance vlan-30
!
interface Port-Channel7
   description ### uplink ###
   switchport trunk allowed vlan 10,20,30
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
interface Vlan30
   description ### server-201 ###
   vrf vlan-30
   ip address 10.2.30.201/24
!
ip routing
ip routing vrf vlan-10
ip routing vrf vlan-20
ip routing vrf vlan-30
!
ip route vrf vlan-10 0.0.0.0/0 10.2.10.254
ip route vrf vlan-20 0.0.0.0/0 10.2.20.254
ip route vrf vlan-30 0.0.0.0/0 10.2.30.254
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
service routing protocols model ribd
!
hostname client-104
!
vlan 40
   name NET-10.2.40.0/24
!
interface Port-Channel8
   description ### uplink ###
   switchport trunk allowed vlan 40
   switchport mode trunk
!
interface Ethernet1
   channel-group 8 mode active
!
interface Ethernet2
   channel-group 8 mode active
!
interface Vlan40
   ip address 10.2.40.104/24
!
ip routing
!
ip route 0.0.0.0/0 10.2.40.254
```

</details>


### Проверка взаимодействия

<details>
  <summary>вывод ip/mac хостов </summary>
  
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
client-104#show interfaces | i address|Vlan
.....
Vlan40 is up, line protocol is up (connected)
  Hardware is Vlan, address is 5000.0045.abdf (bia 5000.0045.abdf)
  Internet address is 10.2.40.104/24
  Broadcast address is 255.255.255.255
  
client-104#show ip arp
Address         Age (sec)  Hardware Addr   Interface
10.2.40.254       3:06:47  0000.0000.cafe  Vlan40, Port-Channel8
```
```
server-201#show interfaces | i address|Vlan
....
Vlan10 is up, line protocol is up (connected)
  Hardware is Vlan, address is 5000.0088.fe27 (bia 5000.0088.fe27)
  Internet address is 10.2.10.201/24
  Broadcast address is 255.255.255.255
Vlan20 is up, line protocol is up (connected)
  Hardware is Vlan, address is 5000.0088.fe27 (bia 5000.0088.fe27)
  Internet address is 10.2.20.201/24
  Broadcast address is 255.255.255.255
Vlan30 is up, line protocol is up (connected)
  Hardware is Vlan, address is 5000.0088.fe27 (bia 5000.0088.fe27)
  Internet address is 10.2.30.201/24
  Broadcast address is 255.255.255.255

server-201#show ip arp vrf all
VRF: default
Address         Age (sec)  Hardware Addr   Interface

VRF: vlan-10
Address         Age (sec)  Hardware Addr   Interface
10.2.10.254       2:30:04  0000.0000.cafe  Vlan10, Port-Channel7

VRF: vlan-20
Address         Age (sec)  Hardware Addr   Interface
10.2.20.102       0:12:21  0050.7966.6808  Vlan20, not learned
10.2.20.254       1:51:42  0000.0000.cafe  Vlan20, Port-Channel7

VRF: vlan-30
Address         Age (sec)  Hardware Addr   Interface
10.2.30.103       0:11:50  0050.7966.6809  Vlan30, not learned
10.2.30.254       2:25:11  0000.0000.cafe  Vlan30, Port-Channel7
```
```
dc1-fw-199#show interfaces | i address|Vlan
Vlan4081 is up, line protocol is up (connected)
  Hardware is Vlan, address is 5000.00ae.f703 (bia 5000.00ae.f703)
  Internet address is 10.16.25003.244/29
  Broadcast address is 255.255.255.255
Vlan4082 is up, line protocol is up (connected)
  Hardware is Vlan, address is 5000.00ae.f703 (bia 5000.00ae.f703)
  Internet address is 10.16.25003.252/29
  Broadcast address is 255.255.255.255

dc1-fw-199#show ip arp
Address         Age (sec)  Hardware Addr   Interface
10.16.25003.241      0:00:01  5000.0003.3766  Vlan4081, Port-Channel1
10.16.25003.249      0:00:02  5000.0003.3766  Vlan4082, Port-Channel1
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
  10.16.250.1 4 65101           4098      4083    0    0 02:49:22 Estab   1      1
  10.16.250.3 4 65102           2437      2437    0    0 01:42:37 Estab   1      1
  10.16.250.5 4 65103           3674      3728    0    0 02:33:37 Estab   1      1
```
```
dc1-spine-1#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.1.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.16.250.1 4 65101           4101      4086    0    0 02:49:30 Estab   21     21
  10.16.250.3 4 65102           2439      2440    0    0 01:42:45 Estab   23     23
  10.16.250.5 4 65103           3677      3731    0    0 02:33:44 Estab   12     12
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
  10.16.251.1 4 65101           4123      4088    0    0 02:49:22 Estab   1      1
  10.16.251.3 4 65102           2456      2438    0    0 01:42:37 Estab   1      1
  10.16.251.5 4 65103           3710      3723    0    0 02:33:37 Estab   1      1
```
```
dc1-spine-2#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.2.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.16.251.1 4 65101           4126      4091    0    0 02:49:30 Estab   21     21
  10.16.251.3 4 65102           2459      2441    0    0 01:42:45 Estab   23     23
  10.16.251.5 4 65103           3713      3726    0    0 02:33:44 Estab   12     12
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
  ### dc1-spine-1 ###      10.16.250.0 4 65100          74055     75074    0    0 02:49:22 Estab   3      3
  ### dc1-spine-2 ###      10.16.251.0 4 65100          73654     74343    0    0 02:49:22 Estab   3      3
```
```
dc1-leaf-101#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.101.0, local AS number 65101
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-spine-1 ###      10.16.250.0 4 65100          74059     75077    0    0 02:49:30 Estab   35     35
  ### dc1-spine-2 ###      10.16.251.0 4 65100          73657     74346    0    0 02:49:30 Estab   35     35
```
```
dc1-leaf-101#show ip bgp vrf all
BGP routing table information for VRF default
Router identifier 10.0.101.0, local AS number 65101
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.1.0/32            10.16.250.0              0       -          100     0       65100 i
 * >      10.0.2.0/32            10.16.251.0              0       -          100     0       65100 i
 * >      10.0.101.0/32          -                     -       -          -       0       i
 * >Ec    10.0.102.0/32          10.16.250.0              0       -          100     0       65100 65102 i
 *  ec    10.0.102.0/32          10.16.251.0              0       -          100     0       65100 65102 i
 * >Ec    10.0.103.0/32          10.16.251.0              0       -          100     0       65100 65103 i
 *  ec    10.0.103.0/32          10.16.250.0              0       -          100     0       65100 65103 i
BGP routing table information for VRF tenant-1
Router identifier 10.2.20.254, local AS number 65101
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    0.0.0.0/0              10.0.103.0            0       -          100     0       65100 65103 65199 ?
 *  ec    0.0.0.0/0              10.0.103.0            0       -          100     0       65100 65103 65199 ?
 * >Ec    10.16.25003.240/29        10.0.103.0            0       -          100     0       65100 65103 i
 *  ec    10.16.25003.240/29        10.0.103.0            0       -          100     0       65100 65103 i
 * >      10.2.10.0/24           -                     -       -          -       0       i
 *  Ec    10.2.10.0/24           10.0.102.0            0       -          100     0       65100 65102 i
 *  ec    10.2.10.0/24           10.0.102.0            0       -          100     0       65100 65102 i
          10.2.10.201/32         10.0.102.0            0       -          100     0       65100 65102 i
          10.2.10.201/32         10.0.102.0            0       -          100     0       65100 65102 i
 * >      10.2.20.0/24           -                     -       -          -       0       i
 *  Ec    10.2.20.0/24           10.0.103.0            0       -          100     0       65100 65103 i
 *  ec    10.2.20.0/24           10.0.103.0            0       -          100     0       65100 65103 i
 *  ec    10.2.20.0/24           10.0.102.0            0       -          100     0       65100 65102 i
 *  ec    10.2.20.0/24           10.0.102.0            0       -          100     0       65100 65102 i
 * >Ec    10.2.20.102/32         10.0.103.0            0       -          100     0       65100 65103 i
 *  ec    10.2.20.102/32         10.0.103.0            0       -          100     0       65100 65103 i
          10.2.20.201/32         10.0.102.0            0       -          100     0       65100 65102 i
          10.2.20.201/32         10.0.102.0            0       -          100     0       65100 65102 i
BGP routing table information for VRF tenant-2
Router identifier 10.2.40.254, local AS number 65101
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    0.0.0.0/0              10.0.103.0            0       -          100     0       65100 65103 65199 ?
 *  ec    0.0.0.0/0              10.0.103.0            0       -          100     0       65100 65103 65199 ?
 * >Ec    10.16.25003.248/29        10.0.103.0            0       -          100     0       65100 65103 i
 *  ec    10.16.25003.248/29        10.0.103.0            0       -          100     0       65100 65103 i
 * >      10.2.30.0/24           -                     -       -          -       0       i
 *  Ec    10.2.30.0/24           10.0.103.0            0       -          100     0       65100 65103 i
 *  ec    10.2.30.0/24           10.0.103.0            0       -          100     0       65100 65103 i
 *  ec    10.2.30.0/24           10.0.102.0            0       -          100     0       65100 65102 i
 *  ec    10.2.30.0/24           10.0.102.0            0       -          100     0       65100 65102 i
 * >Ec    10.2.30.103/32         10.0.103.0            0       -          100     0       65100 65103 i
 *  ec    10.2.30.103/32         10.0.103.0            0       -          100     0       65100 65103 i
          10.2.30.201/32         10.0.102.0            0       -          100     0       65100 65102 i
          10.2.30.201/32         10.0.102.0            0       -          100     0       65100 65102 i
 * >      10.2.40.0/24           -                     -       -          -       0       i
 *  Ec    10.2.40.0/24           10.0.102.0            0       -          100     0       65100 65102 i
 *  ec    10.2.40.0/24           10.0.102.0            0       -          100     0       65100 65102 i
          10.2.40.104/32         10.0.102.0            0       -          100     0       65100 65102 i
          10.2.40.104/32         10.0.102.0            0       -          100     0       65100 65102 i
```
```
dc1-leaf-101#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP             Tunnel Type(s)
---------------- --------------
10.0.102.0       flood, unicast
10.0.103.0       flood, unicast

Total number of remote VTEPS:  2
```
```
dc1-leaf-101#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface           802.1Q Tag
----------- ---------- ------------ ------------------- ----------
10010       10         static       Port-Channel7       10        
                                    Vxlan1              10        
10020       20         static       Port-Channel7       20        
                                    Vxlan1              20        
10030       30         static       Port-Channel7       30        
                                    Vxlan1              30        
10040       40         static       Port-Channel8       40        
                                    Vxlan1              40        

VNI to dynamic VLAN Mapping for Vxlan1
VNI        VLAN       VRF            Source       
---------- ---------- -------------- ------------ 
4001       4094       tenant-1       evpn         
4002       4093       tenant-2       evpn         
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
    [10, 10010]       [20, 10020]       [30, 10030]       [40, 10040]      
   
  Dynamic VLAN to VNI mapping for 'evpn' is
    [4093, 4002]      [4094, 4001]     
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is 
   [tenant-1, 4001]
   [tenant-2, 4002]
  Headend replication flood vtep list is:
    10 10.0.102.0     
    20 10.0.102.0      10.0.103.0     
    30 10.0.102.0      10.0.103.0     
    40 10.0.102.0     
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

Gateway of last resort:
 B E      0.0.0.0/0 [20/0] via VTEP 10.0.103.0 VNI 4001 router-mac 50:00:00:03:37:66 local-interface Vxlan1

 B E      10.16.25003.240/29 [20/0] via VTEP 10.0.103.0 VNI 4001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 C        10.2.10.0/24 is directly connected, Vlan10
 B E      10.2.20.102/32 [20/0] via VTEP 10.0.103.0 VNI 4001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 C        10.2.20.0/24 is directly connected, Vlan20
```
```
dc1-leaf-101#show ip route vrf tenant-2

VRF: tenant-2
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

Gateway of last resort:
 B E      0.0.0.0/0 [20/0] via VTEP 10.0.103.0 VNI 4002 router-mac 50:00:00:03:37:66 local-interface Vxlan1

 B E      10.16.25003.248/29 [20/0] via VTEP 10.0.103.0 VNI 4002 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 B E      10.2.30.103/32 [20/0] via VTEP 10.0.103.0 VNI 4002 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 C        10.2.30.0/24 is directly connected, Vlan30
 C        10.2.40.0/24 is directly connected, Vlan40
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
 * >      RD: 10.0.101.0:30 imet 10.0.101.0
                                 -                     -       -       0       i
 * >      RD: 10.0.101.0:40 imet 10.0.101.0
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:10 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:30 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:30 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:40 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:40 imet 10.0.102.0
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
 * >      RD: 10.0.101.0:40 mac-ip 5000.0045.abdf
                                 -                     -       -       0       i
 * >      RD: 10.0.101.0:40 mac-ip 5000.0045.abdf 10.2.40.104
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:40 mac-ip 5000.0045.abdf 10.2.40.104
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:40 mac-ip 5000.0045.abdf 10.2.40.104
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:10 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:20 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:30 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:30 mac-ip 5000.0088.fe27
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
 * >      RD: 10.0.101.0:30 mac-ip 5000.0088.fe27 10.2.30.201
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:30 mac-ip 5000.0088.fe27 10.2.30.201
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:30 mac-ip 5000.0088.fe27 10.2.30.201
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
 * >      RD: 10.0.101.0:30 auto-discovery 0 0000:0000:0101:0007:0000
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:10 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:20 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:30 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:30 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.101.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.101.0:40 auto-discovery 0 0000:0000:0101:0008:0000
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:40 auto-discovery 0 0000:0000:0101:0008:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:40 auto-discovery 0 0000:0000:0101:0008:0000
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
 * >Ec    RD: 10.0.103.0:4001 ip-prefix 0.0.0.0/0
                                 10.0.103.0            -       100     0       65100 65103 65199 ?
 *  ec    RD: 10.0.103.0:4001 ip-prefix 0.0.0.0/0
                                 10.0.103.0            -       100     0       65100 65103 65199 ?
 * >Ec    RD: 10.0.103.0:4002 ip-prefix 0.0.0.0/0
                                 10.0.103.0            -       100     0       65100 65103 65199 ?
 *  ec    RD: 10.0.103.0:4002 ip-prefix 0.0.0.0/0
                                 10.0.103.0            -       100     0       65100 65103 65199 ?
 * >Ec    RD: 10.0.103.0:4001 ip-prefix 10.16.25003.240/29
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:4001 ip-prefix 10.16.25003.240/29
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:4002 ip-prefix 10.16.25003.248/29
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:4002 ip-prefix 10.16.25003.248/29
                                 10.0.103.0            -       100     0       65100 65103 i
 * >      RD: 10.0.101.0:4001 ip-prefix 10.2.10.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:4001 ip-prefix 10.2.10.0/24
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:4001 ip-prefix 10.2.10.0/24
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.101.0:4001 ip-prefix 10.2.20.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:4001 ip-prefix 10.2.20.0/24
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:4001 ip-prefix 10.2.20.0/24
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.103.0:4001 ip-prefix 10.2.20.0/24
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:4001 ip-prefix 10.2.20.0/24
                                 10.0.103.0            -       100     0       65100 65103 i
 * >      RD: 10.0.101.0:4002 ip-prefix 10.2.30.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:4002 ip-prefix 10.2.30.0/24
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:4002 ip-prefix 10.2.30.0/24
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.103.0:4002 ip-prefix 10.2.30.0/24
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:4002 ip-prefix 10.2.30.0/24
                                 10.0.103.0            -       100     0       65100 65103 i
 * >      RD: 10.0.101.0:4002 ip-prefix 10.2.40.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:4002 ip-prefix 10.2.40.0/24
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:4002 ip-prefix 10.2.40.0/24
                                 10.0.102.0            -       100     0       65100 65102 i
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
EVPN instance: VLAN 30
  Route distinguisher: 0:0
  Route target import: Route-Target-AS:10030:30
  Route target export: Route-Target-AS:10030:30
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
EVPN instance: VLAN 40
  Route distinguisher: 0:0
  Route target import: Route-Target-AS:10040:40
  Route target export: Route-Target-AS:10040:40
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
  10  5000.0088.fe27  EVPN      Vx1  0.0.0.0          1       1:44:50 ago
  20  0050.7966.6808  EVPN      Vx1  10.0.103.0       1       1:04:05 ago
  20  5000.0088.fe27  EVPN      Vx1  0.0.0.0          1       1:44:50 ago
  30  0050.7966.6809  EVPN      Vx1  10.0.103.0       1       1:04:03 ago
  30  5000.0088.fe27  EVPN      Vx1  0.0.0.0          1       0:02:35 ago
4093  5000.0003.3766  EVPN      Vx1  10.0.103.0       1       2:35:51 ago
4093  5000.00d5.5dc0  EVPN      Vx1  10.0.102.0       1       1:44:51 ago
4094  5000.0003.3766  EVPN      Vx1  10.0.103.0       1       2:35:51 ago
4094  5000.00d5.5dc0  EVPN      Vx1  10.0.102.0       1       1:44:51 ago
Total Remote Mac Addresses for this criterion: 9
```
```
dc1-leaf-101#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
   1    0000.0000.cafe    STATIC      Cpu
  10    0000.0000.cafe    STATIC      Cpu
  10    5000.0088.fe27    DYNAMIC     Po7        1       1:44:59 ago
  20    0000.0000.cafe    STATIC      Cpu
  20    0050.7966.6808    DYNAMIC     Vx1        1       1:04:13 ago
  20    5000.0088.fe27    DYNAMIC     Po7        1       1:44:59 ago
  30    0000.0000.cafe    STATIC      Cpu
  30    0050.7966.6809    DYNAMIC     Vx1        1       1:04:12 ago
  30    5000.0088.fe27    DYNAMIC     Po7        1       0:02:44 ago
  40    0000.0000.cafe    STATIC      Cpu
  40    5000.0045.abdf    DYNAMIC     Po8        1       2 days, 2:40:22 ago
4093    0000.0000.cafe    STATIC      Cpu
4093    5000.0003.3766    DYNAMIC     Vx1        1       2:36:00 ago
4093    5000.00d5.5dc0    DYNAMIC     Vx1        1       1:45:00 ago
4094    0000.0000.cafe    STATIC      Cpu
4094    5000.0003.3766    DYNAMIC     Vx1        1       2:36:00 ago
4094    5000.00d5.5dc0    DYNAMIC     Vx1        1       1:45:00 ago
Total Mac Addresses for this criterion: 17

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
```
```
dc1-leaf-101#show ip arp vrf tenant-1
Address         Age (sec)  Hardware Addr   Interface
10.2.10.201             -  5000.0088.fe27  Vlan10, Port-Channel7
10.2.20.102             -  0050.7966.6808  Vlan20, Vxlan1
10.2.20.201             -  5000.0088.fe27  Vlan20, Port-Channel7
```
```
dc1-leaf-101#show ip arp vrf tenant-2
Address         Age (sec)  Hardware Addr   Interface
10.2.30.103             -  0050.7966.6809  Vlan30, Vxlan1
10.2.30.201             -  5000.0088.fe27  Vlan30, Port-Channel7
10.2.40.104       0:01:40  5000.0045.abdf  Vlan40, Port-Channel8
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
  ### dc1-spine-1 ###      10.16.250.2 4 65100          73957     74674    0    0 01:42:37 Estab   3      3
  ### dc1-spine-2 ###      10.16.251.2 4 65100          73504     73964    0    0 01:42:37 Estab   3      3
```
```
dc1-leaf-102#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.102.0, local AS number 65102
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-spine-1 ###      10.16.250.2 4 65100          73960     74676    0    0 01:42:44 Estab   33     33
  ### dc1-spine-2 ###      10.16.251.2 4 65100          73507     73967    0    0 01:42:44 Estab   33     33
```
```
dc1-leaf-102#show ip bgp vrf all
BGP routing table information for VRF default
Router identifier 10.0.102.0, local AS number 65102
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.1.0/32            10.16.250.2              0       -          100     0       65100 i
 * >      10.0.2.0/32            10.16.251.2              0       -          100     0       65100 i
 * >Ec    10.0.101.0/32          10.16.250.2              0       -          100     0       65100 65101 i
 *  ec    10.0.101.0/32          10.16.251.2              0       -          100     0       65100 65101 i
 * >      10.0.102.0/32          -                     -       -          -       0       i
 * >Ec    10.0.103.0/32          10.16.250.2              0       -          100     0       65100 65103 i
 *  ec    10.0.103.0/32          10.16.251.2              0       -          100     0       65100 65103 i
BGP routing table information for VRF tenant-1
Router identifier 10.2.20.254, local AS number 65102
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    0.0.0.0/0              10.0.103.0            0       -          100     0       65100 65103 65199 ?
 *  ec    0.0.0.0/0              10.0.103.0            0       -          100     0       65100 65103 65199 ?
 * >Ec    10.16.25003.240/29        10.0.103.0            0       -          100     0       65100 65103 i
 *  ec    10.16.25003.240/29        10.0.103.0            0       -          100     0       65100 65103 i
 * >      10.2.10.0/24           -                     -       -          -       0       i
 *  Ec    10.2.10.0/24           10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.10.0/24           10.0.101.0            0       -          100     0       65100 65101 i
          10.2.10.201/32         10.0.101.0            0       -          100     0       65100 65101 i
          10.2.10.201/32         10.0.101.0            0       -          100     0       65100 65101 i
 * >      10.2.20.0/24           -                     -       -          -       0       i
 *  Ec    10.2.20.0/24           10.0.103.0            0       -          100     0       65100 65103 i
 *  ec    10.2.20.0/24           10.0.103.0            0       -          100     0       65100 65103 i
 *  ec    10.2.20.0/24           10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.20.0/24           10.0.101.0            0       -          100     0       65100 65101 i
 * >Ec    10.2.20.102/32         10.0.103.0            0       -          100     0       65100 65103 i
 *  ec    10.2.20.102/32         10.0.103.0            0       -          100     0       65100 65103 i
          10.2.20.201/32         10.0.101.0            0       -          100     0       65100 65101 i
          10.2.20.201/32         10.0.101.0            0       -          100     0       65100 65101 i
BGP routing table information for VRF tenant-2
Router identifier 10.2.40.254, local AS number 65102
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    0.0.0.0/0              10.0.103.0            0       -          100     0       65100 65103 65199 ?
 *  ec    0.0.0.0/0              10.0.103.0            0       -          100     0       65100 65103 65199 ?
 * >Ec    10.16.25003.248/29        10.0.103.0            0       -          100     0       65100 65103 i
 *  ec    10.16.25003.248/29        10.0.103.0            0       -          100     0       65100 65103 i
 * >      10.2.30.0/24           -                     -       -          -       0       i
 *  Ec    10.2.30.0/24           10.0.103.0            0       -          100     0       65100 65103 i
 *  ec    10.2.30.0/24           10.0.103.0            0       -          100     0       65100 65103 i
 *  ec    10.2.30.0/24           10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.30.0/24           10.0.101.0            0       -          100     0       65100 65101 i
 * >Ec    10.2.30.103/32         10.0.103.0            0       -          100     0       65100 65103 i
 *  ec    10.2.30.103/32         10.0.103.0            0       -          100     0       65100 65103 i
          10.2.30.201/32         10.0.101.0            0       -          100     0       65100 65101 i
          10.2.30.201/32         10.0.101.0            0       -          100     0       65100 65101 i
 * >      10.2.40.0/24           -                     -       -          -       0       i
 *  Ec    10.2.40.0/24           10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.40.0/24           10.0.101.0            0       -          100     0       65100 65101 i
          10.2.40.104/32         10.0.101.0            0       -          100     0       65100 65101 i
          10.2.40.104/32         10.0.101.0            0       -          100     0       65100 65101 i
```
```
dc1-leaf-102#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP             Tunnel Type(s)
---------------- --------------
10.0.101.0       flood, unicast
10.0.103.0       flood, unicast

Total number of remote VTEPS:  2
```
```
dc1-leaf-102#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface           802.1Q Tag
----------- ---------- ------------ ------------------- ----------
10010       10         static       Port-Channel7       10        
                                    Vxlan1              10        
10020       20         static       Port-Channel7       20        
                                    Vxlan1              20        
10030       30         static       Port-Channel7       30        
                                    Vxlan1              30        
10040       40         static       Port-Channel8       40        
                                    Vxlan1              40        

VNI to dynamic VLAN Mapping for Vxlan1
VNI        VLAN       VRF            Source       
---------- ---------- -------------- ------------ 
4001       4094       tenant-1       evpn         
4002       4093       tenant-2       evpn         
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
    [10, 10010]       [20, 10020]       [30, 10030]       [40, 10040]      
   
  Dynamic VLAN to VNI mapping for 'evpn' is
    [4093, 4002]      [4094, 4001]     
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is 
   [tenant-1, 4001]
   [tenant-2, 4002]
  Headend replication flood vtep list is:
    10 10.0.101.0     
    20 10.0.101.0      10.0.103.0     
    30 10.0.101.0      10.0.103.0     
    40 10.0.101.0     
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

Gateway of last resort:
 B E      0.0.0.0/0 [20/0] via VTEP 10.0.103.0 VNI 4001 router-mac 50:00:00:03:37:66 local-interface Vxlan1

 B E      10.16.25003.240/29 [20/0] via VTEP 10.0.103.0 VNI 4001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 C        10.2.10.0/24 is directly connected, Vlan10
 B E      10.2.20.102/32 [20/0] via VTEP 10.0.103.0 VNI 4001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 C        10.2.20.0/24 is directly connected, Vlan20
```
```
dc1-leaf-102#show ip route vrf tenant-2

VRF: tenant-2
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

Gateway of last resort:
 B E      0.0.0.0/0 [20/0] via VTEP 10.0.103.0 VNI 4002 router-mac 50:00:00:03:37:66 local-interface Vxlan1

 B E      10.16.25003.248/29 [20/0] via VTEP 10.0.103.0 VNI 4002 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 B E      10.2.30.103/32 [20/0] via VTEP 10.0.103.0 VNI 4002 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 C        10.2.30.0/24 is directly connected, Vlan30
 C        10.2.40.0/24 is directly connected, Vlan40
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
 * >Ec    RD: 10.0.101.0:30 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:30 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.101.0:40 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:40 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:10 imet 10.0.102.0
                                 -                     -       -       0       i
 * >      RD: 10.0.102.0:20 imet 10.0.102.0
                                 -                     -       -       0       i
 * >      RD: 10.0.102.0:30 imet 10.0.102.0
                                 -                     -       -       0       i
 * >      RD: 10.0.102.0:40 imet 10.0.102.0
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
 * >Ec    RD: 10.0.101.0:40 mac-ip 5000.0045.abdf
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:40 mac-ip 5000.0045.abdf
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.101.0:40 mac-ip 5000.0045.abdf 10.2.40.104
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:40 mac-ip 5000.0045.abdf 10.2.40.104
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:40 mac-ip 5000.0045.abdf 10.2.40.104
                                 -                     -       -       0       i
 * >      RD: 10.0.102.0:10 mac-ip 5000.0088.fe27
                                 -                     -       -       0       i
 * >      RD: 10.0.102.0:20 mac-ip 5000.0088.fe27
                                 -                     -       -       0       i
 * >      RD: 10.0.102.0:30 mac-ip 5000.0088.fe27
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
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:30 mac-ip 5000.0088.fe27 10.2.30.201
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:30 mac-ip 5000.0088.fe27 10.2.30.201
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:30 mac-ip 5000.0088.fe27 10.2.30.201
                                 -                     -       -       0       i
```
```
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
 * >Ec    RD: 10.0.101.0:30 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:30 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:10 auto-discovery 0 0000:0000:0101:0007:0000
                                 -                     -       -       0       i
 * >      RD: 10.0.102.0:20 auto-discovery 0 0000:0000:0101:0007:0000
                                 -                     -       -       0       i
 * >      RD: 10.0.102.0:30 auto-discovery 0 0000:0000:0101:0007:0000
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:40 auto-discovery 0 0000:0000:0101:0008:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:40 auto-discovery 0 0000:0000:0101:0008:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:40 auto-discovery 0 0000:0000:0101:0008:0000
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
 * >Ec    RD: 10.0.103.0:4001 ip-prefix 0.0.0.0/0
                                 10.0.103.0            -       100     0       65100 65103 65199 ?
 *  ec    RD: 10.0.103.0:4001 ip-prefix 0.0.0.0/0
                                 10.0.103.0            -       100     0       65100 65103 65199 ?
 * >Ec    RD: 10.0.103.0:4002 ip-prefix 0.0.0.0/0
                                 10.0.103.0            -       100     0       65100 65103 65199 ?
 *  ec    RD: 10.0.103.0:4002 ip-prefix 0.0.0.0/0
                                 10.0.103.0            -       100     0       65100 65103 65199 ?
 * >Ec    RD: 10.0.103.0:4001 ip-prefix 10.16.25003.240/29
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:4001 ip-prefix 10.16.25003.240/29
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.103.0:4002 ip-prefix 10.16.25003.248/29
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:4002 ip-prefix 10.16.25003.248/29
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.101.0:4001 ip-prefix 10.2.10.0/24
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:4001 ip-prefix 10.2.10.0/24
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:4001 ip-prefix 10.2.10.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:4001 ip-prefix 10.2.20.0/24
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:4001 ip-prefix 10.2.20.0/24
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:4001 ip-prefix 10.2.20.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.103.0:4001 ip-prefix 10.2.20.0/24
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:4001 ip-prefix 10.2.20.0/24
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.101.0:4002 ip-prefix 10.2.30.0/24
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:4002 ip-prefix 10.2.30.0/24
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:4002 ip-prefix 10.2.30.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.103.0:4002 ip-prefix 10.2.30.0/24
                                 10.0.103.0            -       100     0       65100 65103 i
 *  ec    RD: 10.0.103.0:4002 ip-prefix 10.2.30.0/24
                                 10.0.103.0            -       100     0       65100 65103 i
 * >Ec    RD: 10.0.101.0:4002 ip-prefix 10.2.40.0/24
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:4002 ip-prefix 10.2.40.0/24
                                 10.0.101.0            -       100     0       65100 65101 i
 * >      RD: 10.0.102.0:4002 ip-prefix 10.2.40.0/24
                                 -                     -       -       0       i
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
EVPN instance: VLAN 30
  Route distinguisher: 0:0
  Route target import: Route-Target-AS:10030:30
  Route target export: Route-Target-AS:10030:30
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
EVPN instance: VLAN 40
  Route distinguisher: 0:0
  Route target import: Route-Target-AS:10040:40
  Route target export: Route-Target-AS:10040:40
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
  20  0050.7966.6808  EVPN      Vx1  10.0.103.0       1       1:04:05 ago
  30  0050.7966.6809  EVPN      Vx1  10.0.103.0       1       1:04:03 ago
  40  5000.0045.abdf  EVPN      Vx1  0.0.0.0          1       1:44:50 ago
4093  5000.0003.3766  EVPN      Vx1  10.0.103.0       1       1:44:50 ago
4093  5000.0072.8b31  EVPN      Vx1  10.0.101.0       1       1:44:50 ago
4094  5000.0003.3766  EVPN      Vx1  10.0.103.0       1       1:44:50 ago
4094  5000.0072.8b31  EVPN      Vx1  10.0.101.0       1       1:44:50 ago
Total Remote Mac Addresses for this criterion: 7
```
```
dc1-leaf-102#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
   1    0000.0000.cafe    STATIC      Cpu
  10    0000.0000.cafe    STATIC      Cpu
  10    5000.0088.fe27    DYNAMIC     Po7        1       6:44:19 ago
  20    0000.0000.cafe    STATIC      Cpu
  20    0050.7966.6808    DYNAMIC     Vx1        1       1:04:14 ago
  20    5000.0088.fe27    DYNAMIC     Po7        1       6:10:46 ago
  30    0000.0000.cafe    STATIC      Cpu
  30    0050.7966.6809    DYNAMIC     Vx1        1       1:04:12 ago
  30    5000.0088.fe27    DYNAMIC     Po7        1       6:39:27 ago
  40    0000.0000.cafe    STATIC      Cpu
  40    5000.0045.abdf    DYNAMIC     Po8        1       1:44:58 ago
4093    0000.0000.cafe    STATIC      Cpu
4093    5000.0003.3766    DYNAMIC     Vx1        1       1:44:59 ago
4093    5000.0072.8b31    DYNAMIC     Vx1        1       1:44:59 ago
4094    0000.0000.cafe    STATIC      Cpu
4094    5000.0003.3766    DYNAMIC     Vx1        1       1:44:59 ago
4094    5000.0072.8b31    DYNAMIC     Vx1        1       1:44:59 ago
Total Mac Addresses for this criterion: 17

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
```
```
dc1-leaf-102#show ip arp vrf tenant-1
Address         Age (sec)  Hardware Addr   Interface
10.2.10.201       0:02:48  5000.0088.fe27  Vlan10, Port-Channel7
10.2.20.102             -  0050.7966.6808  Vlan20, Vxlan1
10.2.20.201       0:01:06  5000.0088.fe27  Vlan20, Port-Channel7
```
```
dc1-leaf-102#show ip arp vrf tenant-2
Address         Age (sec)  Hardware Addr   Interface
10.2.30.103             -  0050.7966.6809  Vlan30, Vxlan1
10.2.30.201       0:01:28  5000.0088.fe27  Vlan30, Port-Channel7
10.2.40.104             -  5000.0045.abdf  Vlan40, Port-Channel8
dc1-leaf-102#
```
</details>

<details>
  <summary>проверки leaf-103</summary>
  
```
dc1-leaf-103#show ip bgp summary vrf all
BGP summary information for VRF default
Router identifier 10.0.103.0, local AS number 65103
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-spine-1 ###      10.16.250.4 4 65100          36295     35904    0    0 02:50:16 Estab   3      3
  ### dc1-spine-2 ###      10.16.251.4 4 65100          36095     35949    0    0 02:50:16 Estab   3      3

BGP summary information for VRF tenant-1
Router identifier 10.2.20.254, local AS number 65103
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor     V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-fw-199 tenant-1  10.16.25003.244 4 65199          29412     34468    0    0 08:34:07 Estab   1      1

BGP summary information for VRF tenant-2
Router identifier 10.2.30.254, local AS number 65103
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor     V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-fw-199 tenant-2  10.16.25003.252 4 65199          29389     34460    0    0 08:34:08 Estab   1      1
```
```
dc1-leaf-103#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.103.0, local AS number 65103
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-spine-1 ###      10.16.250.4 4 65100          35902     35519    0    0 02:33:44 Estab   44     44
  ### dc1-spine-2 ###      10.16.251.4 4 65100          35703     35558    0    0 02:33:44 Estab   44     44
```
```
dc1-leaf-103#show ip bgp vrf all
BGP routing table information for VRF default
Router identifier 10.0.103.0, local AS number 65103
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.1.0/32            10.16.250.4              0       -          100     0       65100 i
 * >      10.0.2.0/32            10.16.251.4              0       -          100     0       65100 i
 * >Ec    10.0.101.0/32          10.16.251.4              0       -          100     0       65100 65101 i
 *  ec    10.0.101.0/32          10.16.250.4              0       -          100     0       65100 65101 i
 * >Ec    10.0.102.0/32          10.16.250.4              0       -          100     0       65100 65102 i
 *  ec    10.0.102.0/32          10.16.251.4              0       -          100     0       65100 65102 i
 * >      10.0.103.0/32          -                     -       -          -       0       i
BGP routing table information for VRF tenant-1
Router identifier 10.2.20.254, local AS number 65103
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      0.0.0.0/0              10.16.25003.244          0       -          100     0       65199 ?
 * >      10.16.25003.240/29        -                     -       -          -       0       i
 * >Ec    10.2.10.0/24           10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.10.0/24           10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.10.0/24           10.0.102.0            0       -          100     0       65100 65102 i
 *  ec    10.2.10.0/24           10.0.102.0            0       -          100     0       65100 65102 i
 * >Ec    10.2.10.201/32         10.0.102.0            0       -          100     0       65100 65102 i
 *  ec    10.2.10.201/32         10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.10.201/32         10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.10.201/32         10.0.102.0            0       -          100     0       65100 65102 i
 * >      10.2.20.0/24           -                     -       -          -       0       i
 *  Ec    10.2.20.0/24           10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.20.0/24           10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.20.0/24           10.0.102.0            0       -          100     0       65100 65102 i
 *  ec    10.2.20.0/24           10.0.102.0            0       -          100     0       65100 65102 i
 * >Ec    10.2.20.201/32         10.0.102.0            0       -          100     0       65100 65102 i
 *  ec    10.2.20.201/32         10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.20.201/32         10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.20.201/32         10.0.102.0            0       -          100     0       65100 65102 i
BGP routing table information for VRF tenant-2
Router identifier 10.2.30.254, local AS number 65103
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      0.0.0.0/0              10.16.25003.252          0       -          100     0       65199 ?
 * >      10.16.25003.248/29        -                     -       -          -       0       i
 * >      10.2.30.0/24           -                     -       -          -       0       i
 *  Ec    10.2.30.0/24           10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.30.0/24           10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.30.0/24           10.0.102.0            0       -          100     0       65100 65102 i
 *  ec    10.2.30.0/24           10.0.102.0            0       -          100     0       65100 65102 i
 * >Ec    10.2.30.201/32         10.0.102.0            0       -          100     0       65100 65102 i
 *  ec    10.2.30.201/32         10.0.102.0            0       -          100     0       65100 65102 i
 *  ec    10.2.30.201/32         10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.30.201/32         10.0.101.0            0       -          100     0       65100 65101 i
 * >Ec    10.2.40.0/24           10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.40.0/24           10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.40.0/24           10.0.102.0            0       -          100     0       65100 65102 i
 *  ec    10.2.40.0/24           10.0.102.0            0       -          100     0       65100 65102 i
 * >Ec    10.2.40.104/32         10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.40.104/32         10.0.101.0            0       -          100     0       65100 65101 i
 *  ec    10.2.40.104/32         10.0.102.0            0       -          100     0       65100 65102 i
 *  ec    10.2.40.104/32         10.0.102.0            0       -          100     0       65100 65102 i
```
```
dc1-leaf-103#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP             Tunnel Type(s)
---------------- --------------
10.0.101.0       unicast, flood
10.0.102.0       unicast, flood

Total number of remote VTEPS:  2
```
```
dc1-leaf-103#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface           802.1Q Tag
----------- ---------- ------------ ------------------- ----------
10020       20         static       Ethernet7           untagged  
                                    Port-Channel5       20        
                                    Vxlan1              20        
10030       30         static       Ethernet8           untagged  
                                    Port-Channel5       30        
                                    Vxlan1              30        

VNI to dynamic VLAN Mapping for Vxlan1
VNI        VLAN       VRF            Source       
---------- ---------- -------------- ------------ 
4001       4094       tenant-1       evpn         
4002       4093       tenant-2       evpn         
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
    [4093, 4002]      [4094, 4001]     
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is 
   [tenant-1, 4001]
   [tenant-2, 4002]
  Headend replication flood vtep list is:
    20 10.0.102.0      10.0.101.0     
    30 10.0.102.0      10.0.101.0     
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

Gateway of last resort:
 B E      0.0.0.0/0 [20/0] via 10.16.25003.244, Vlan4081

 C        10.16.25003.240/29 is directly connected, Vlan4081
 B E      10.2.10.201/32 [20/0] via VTEP 10.0.101.0 VNI 4001 router-mac 50:00:00:72:8b:31 local-interface Vxlan1
                                via VTEP 10.0.102.0 VNI 4001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 B E      10.2.10.0/24 [20/0] via VTEP 10.0.101.0 VNI 4001 router-mac 50:00:00:72:8b:31 local-interface Vxlan1
                              via VTEP 10.0.102.0 VNI 4001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 B E      10.2.20.201/32 [20/0] via VTEP 10.0.101.0 VNI 4001 router-mac 50:00:00:72:8b:31 local-interface Vxlan1
                                via VTEP 10.0.102.0 VNI 4001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 C        10.2.20.0/24 is directly connected, Vlan20
```
```
dc1-leaf-103#show ip route vrf tenant-2

VRF: tenant-2
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

Gateway of last resort:
 B E      0.0.0.0/0 [20/0] via 10.16.25003.252, Vlan4082

 C        10.16.25003.248/29 is directly connected, Vlan4082
 B E      10.2.30.201/32 [20/0] via VTEP 10.0.102.0 VNI 4002 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
                                via VTEP 10.0.101.0 VNI 4002 router-mac 50:00:00:72:8b:31 local-interface Vxlan1
 C        10.2.30.0/24 is directly connected, Vlan30
 B E      10.2.40.104/32 [20/0] via VTEP 10.0.102.0 VNI 4002 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
                                via VTEP 10.0.101.0 VNI 4002 router-mac 50:00:00:72:8b:31 local-interface Vxlan1
 B E      10.2.40.0/24 [20/0] via VTEP 10.0.102.0 VNI 4002 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
                              via VTEP 10.0.101.0 VNI 4002 router-mac 50:00:00:72:8b:31 local-interface Vxlan1
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
 * >Ec    RD: 10.0.101.0:30 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:30 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.101.0:40 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:40 imet 10.0.101.0
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:10 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:30 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:30 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:40 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:40 imet 10.0.102.0
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.103.0:20 imet 10.0.103.0
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:30 imet 10.0.103.0
                                 -                     -       -       0       i
```
```
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
 * >Ec    RD: 10.0.101.0:40 mac-ip 5000.0045.abdf
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:40 mac-ip 5000.0045.abdf
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.101.0:40 mac-ip 5000.0045.abdf 10.2.40.104
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:40 mac-ip 5000.0045.abdf 10.2.40.104
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:40 mac-ip 5000.0045.abdf 10.2.40.104
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:40 mac-ip 5000.0045.abdf 10.2.40.104
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:10 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:20 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:30 mac-ip 5000.0088.fe27
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:30 mac-ip 5000.0088.fe27
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
 * >Ec    RD: 10.0.101.0:30 mac-ip 5000.0088.fe27 10.2.30.201
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:30 mac-ip 5000.0088.fe27 10.2.30.201
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:30 mac-ip 5000.0088.fe27 10.2.30.201
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:30 mac-ip 5000.0088.fe27 10.2.30.201
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
 * >Ec    RD: 10.0.101.0:30 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:30 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:10 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:10 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:20 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:20 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.102.0:30 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:30 auto-discovery 0 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.101.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:1 auto-discovery 0000:0000:0101:0007:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.101.0:40 auto-discovery 0 0000:0000:0101:0008:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:40 auto-discovery 0 0000:0000:0101:0008:0000
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:40 auto-discovery 0 0000:0000:0101:0008:0000
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:40 auto-discovery 0 0000:0000:0101:0008:0000
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
 * >      RD: 10.0.103.0:4001 ip-prefix 0.0.0.0/0
                                 -                     -       100     0       65199 ?
 * >      RD: 10.0.103.0:4002 ip-prefix 0.0.0.0/0
                                 -                     -       100     0       65199 ?
 * >      RD: 10.0.103.0:4001 ip-prefix 10.16.25003.240/29
                                 -                     -       -       0       i
 * >      RD: 10.0.103.0:4002 ip-prefix 10.16.25003.248/29
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:4001 ip-prefix 10.2.10.0/24
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:4001 ip-prefix 10.2.10.0/24
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:4001 ip-prefix 10.2.10.0/24
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:4001 ip-prefix 10.2.10.0/24
                                 10.0.102.0            -       100     0       65100 65102 i
 * >Ec    RD: 10.0.101.0:4001 ip-prefix 10.2.20.0/24
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:4001 ip-prefix 10.2.20.0/24
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:4001 ip-prefix 10.2.20.0/24
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:4001 ip-prefix 10.2.20.0/24
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.103.0:4001 ip-prefix 10.2.20.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:4002 ip-prefix 10.2.30.0/24
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:4002 ip-prefix 10.2.30.0/24
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:4002 ip-prefix 10.2.30.0/24
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:4002 ip-prefix 10.2.30.0/24
                                 10.0.102.0            -       100     0       65100 65102 i
 * >      RD: 10.0.103.0:4002 ip-prefix 10.2.30.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.101.0:4002 ip-prefix 10.2.40.0/24
                                 10.0.101.0            -       100     0       65100 65101 i
 *  ec    RD: 10.0.101.0:4002 ip-prefix 10.2.40.0/24
                                 10.0.101.0            -       100     0       65100 65101 i
 * >Ec    RD: 10.0.102.0:4002 ip-prefix 10.2.40.0/24
                                 10.0.102.0            -       100     0       65100 65102 i
 *  ec    RD: 10.0.102.0:4002 ip-prefix 10.2.40.0/24
                                 10.0.102.0            -       100     0       65100 65102 i
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
  20  5000.0088.fe27  EVPN      Vx1  10.0.101.0       1       1:44:50 ago
                                     10.0.102.0     
  30  5000.0088.fe27  EVPN      Vx1  10.0.101.0       1       1:44:50 ago
                                     10.0.102.0     
4093  5000.0072.8b31  EVPN      Vx1  10.0.101.0       1       1 day, 8:51:30 ago
4093  5000.00d5.5dc0  EVPN      Vx1  10.0.102.0       1       1 day, 9:43:01 ago
4094  5000.0072.8b31  EVPN      Vx1  10.0.101.0       1       2:35:50 ago
4094  5000.00d5.5dc0  EVPN      Vx1  10.0.102.0       1       1:44:51 ago
Total Remote Mac Addresses for this criterion: 6
```
```
dc1-leaf-103#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  20    0000.0000.cafe    STATIC      Cpu
  20    0050.7966.6808    DYNAMIC     Et7        1       1:04:14 ago
  20    5000.0088.fe27    DYNAMIC     Vx1        1       1:44:59 ago
  30    0000.0000.cafe    STATIC      Cpu
  30    0050.7966.6809    DYNAMIC     Et8        1       1:04:12 ago
  30    5000.0088.fe27    DYNAMIC     Vx1        1       1:44:59 ago
4081    0000.0000.cafe    STATIC      Cpu
4081    5000.00ae.f703    DYNAMIC     Po5        1       1 day, 0:30:53 ago
4082    0000.0000.cafe    STATIC      Cpu
4082    5000.00ae.f703    DYNAMIC     Po5        1       1 day, 0:30:51 ago
4093    0000.0000.cafe    STATIC      Cpu
4093    5000.0072.8b31    DYNAMIC     Vx1        1       1 day, 8:51:39 ago
4093    5000.00d5.5dc0    DYNAMIC     Vx1        1       1 day, 9:43:10 ago
4094    0000.0000.cafe    STATIC      Cpu
4094    5000.0072.8b31    DYNAMIC     Vx1        1       2:35:58 ago
4094    5000.00d5.5dc0    DYNAMIC     Vx1        1       1:45:00 ago
Total Mac Addresses for this criterion: 16

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
```
```
dc1-leaf-103#show ip arp vrf tenant-1
Address         Age (sec)  Hardware Addr   Interface
10.2.20.102       0:03:19  0050.7966.6808  Vlan20, Ethernet7
10.2.20.201             -  5000.0088.fe27  Vlan20, Vxlan1
10.16.25003.244      0:00:02  5000.00ae.f703  Vlan4081, Port-Channel5
```
```
dc1-leaf-103#show ip arp vrf tenant-2
Address         Age (sec)  Hardware Addr   Interface
10.2.30.103       0:01:11  0050.7966.6809  Vlan30, Ethernet8
10.2.30.201             -  5000.0088.fe27  Vlan30, Vxlan1
10.16.25003.252      0:00:02  5000.00ae.f703  Vlan4082, Port-Channel5
dc1-leaf-103#
```
</details>

<details>
  <summary>проверки fw-199</summary>
  
```
dc1-fw-199#show port-channel dense 

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
   Po1(U)             LACP(a)     Et1(PG+) Et2(PG+) 
```
```
dc1-fw-199#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.199.0, local AS number 65199
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  ### dc1-leaf-103 tenant- 10.16.25003.241     4  65103          34168     29163    0    0 08:21:32 Estab   2      2
  ### dc1-leaf-103 tenant- 10.16.25003.249     4  65103          34158     29139    0    0 08:21:32 Estab   2      2
```
```
dc1-fw-199#show ip bgp
BGP routing table information for VRF default
Router identifier 10.0.199.0, local AS number 65199
Route status codes: * - valid, > - active, # - not installed, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

         Network                Next Hop            Metric  LocPref Weight  Path
 * >     10.2.10.0/24           10.16.25003.241          0       100     0       65103 65100 65101 i
 * >     10.2.20.0/24           10.16.25003.241          0       100     0       65103 i
 * >     10.2.30.0/24           10.16.25003.249          0       100     0       65103 i
 * >     10.2.40.0/24           10.16.25003.249          0       100     0       65103 65100 65101 i
```
```
dc1-fw-199#show ip route

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

 C        10.0.199.0/32 is directly connected, Loopback0
 C        10.16.25003.240/29 is directly connected, Vlan4081
 C        10.16.25003.248/29 is directly connected, Vlan4082
 B E      10.2.10.0/24 [20/0] via 10.16.25003.241, Vlan4081
 B E      10.2.20.0/24 [20/0] via 10.16.25003.241, Vlan4081
 B E      10.2.30.0/24 [20/0] via 10.16.25003.249, Vlan4082
 B E      10.2.40.0/24 [20/0] via 10.16.25003.249, Vlan4082
```
</details>

<details>
  <summary>проверки с client-102</summary>
  
```
client-102> ping 10.2.10.201

84 bytes from 10.2.10.201 icmp_seq=1 ttl=62 time=62.722 ms
84 bytes from 10.2.10.201 icmp_seq=2 ttl=62 time=145.957 ms
84 bytes from 10.2.10.201 icmp_seq=3 ttl=62 time=45.050 ms
84 bytes from 10.2.10.201 icmp_seq=4 ttl=62 time=132.429 ms
84 bytes from 10.2.10.201 icmp_seq=5 ttl=62 time=62.671 ms

client-102> ping 10.2.20.102

10.2.20.102 icmp_seq=1 ttl=64 time=0.001 ms
10.2.20.102 icmp_seq=2 ttl=64 time=0.001 ms
10.2.20.102 icmp_seq=3 ttl=64 time=0.001 ms
10.2.20.102 icmp_seq=4 ttl=64 time=0.001 ms
10.2.20.102 icmp_seq=5 ttl=64 time=0.001 ms

client-102> ping 10.2.20.201

84 bytes from 10.2.20.201 icmp_seq=1 ttl=64 time=148.749 ms
84 bytes from 10.2.20.201 icmp_seq=2 ttl=64 time=44.236 ms
84 bytes from 10.2.20.201 icmp_seq=3 ttl=64 time=55.723 ms
84 bytes from 10.2.20.201 icmp_seq=4 ttl=64 time=44.719 ms
84 bytes from 10.2.20.201 icmp_seq=5 ttl=64 time=50.565 ms

client-102> ping 10.2.30.103

84 bytes from 10.2.30.103 icmp_seq=1 ttl=61 time=44.663 ms
84 bytes from 10.2.30.103 icmp_seq=2 ttl=61 time=44.917 ms
84 bytes from 10.2.30.103 icmp_seq=3 ttl=61 time=41.675 ms
84 bytes from 10.2.30.103 icmp_seq=4 ttl=61 time=107.622 ms
84 bytes from 10.2.30.103 icmp_seq=5 ttl=61 time=51.528 ms

client-102> ping 10.2.30.201

84 bytes from 10.2.30.201 icmp_seq=1 ttl=60 time=106.428 ms
84 bytes from 10.2.30.201 icmp_seq=2 ttl=60 time=108.634 ms
84 bytes from 10.2.30.201 icmp_seq=3 ttl=60 time=169.784 ms
84 bytes from 10.2.30.201 icmp_seq=4 ttl=60 time=123.006 ms
84 bytes from 10.2.30.201 icmp_seq=5 ttl=60 time=332.334 ms

client-102> ping 10.2.40.104

84 bytes from 10.2.40.104 icmp_seq=1 ttl=60 time=131.422 ms
84 bytes from 10.2.40.104 icmp_seq=2 ttl=60 time=79.072 ms
84 bytes from 10.2.40.104 icmp_seq=3 ttl=60 time=82.737 ms
84 bytes from 10.2.40.104 icmp_seq=4 ttl=60 time=141.154 ms
84 bytes from 10.2.40.104 icmp_seq=5 ttl=60 time=123.060 ms

client-102> ping 10.0.199.0

84 bytes from 10.0.199.0 icmp_seq=1 ttl=63 time=36.670 ms
84 bytes from 10.0.199.0 icmp_seq=2 ttl=63 time=21.815 ms
84 bytes from 10.0.199.0 icmp_seq=3 ttl=63 time=75.401 ms
84 bytes from 10.0.199.0 icmp_seq=4 ttl=63 time=25.196 ms
84 bytes from 10.0.199.0 icmp_seq=5 ttl=63 time=24.349 ms
```
</details>

<details>
  <summary>проверки с client-103</summary>
  
```
client-103> ping 10.2.10.201
84 bytes from 10.2.10.201 icmp_seq=1 ttl=60 time=147.347 ms
84 bytes from 10.2.10.201 icmp_seq=2 ttl=60 time=118.995 ms
84 bytes from 10.2.10.201 icmp_seq=3 ttl=60 time=118.295 ms
84 bytes from 10.2.10.201 icmp_seq=4 ttl=60 time=81.788 ms
84 bytes from 10.2.10.201 icmp_seq=5 ttl=60 time=94.658 ms

client-103> ping 10.2.20.102
84 bytes from 10.2.20.102 icmp_seq=1 ttl=61 time=45.059 ms
84 bytes from 10.2.20.102 icmp_seq=2 ttl=61 time=53.621 ms
84 bytes from 10.2.20.102 icmp_seq=3 ttl=61 time=68.010 ms
84 bytes from 10.2.20.102 icmp_seq=4 ttl=61 time=43.362 ms
84 bytes from 10.2.20.102 icmp_seq=5 ttl=61 time=74.714 ms

client-103> ping 10.2.20.201
84 bytes from 10.2.20.201 icmp_seq=1 ttl=60 time=92.722 ms
84 bytes from 10.2.20.201 icmp_seq=2 ttl=60 time=109.458 ms
84 bytes from 10.2.20.201 icmp_seq=3 ttl=60 time=139.461 ms
84 bytes from 10.2.20.201 icmp_seq=4 ttl=60 time=89.396 ms
84 bytes from 10.2.20.201 icmp_seq=5 ttl=60 time=96.019 ms

client-103> ping 10.2.30.103
10.2.30.103 icmp_seq=1 ttl=64 time=0.001 ms
10.2.30.103 icmp_seq=2 ttl=64 time=0.001 ms
10.2.30.103 icmp_seq=3 ttl=64 time=0.001 ms
10.2.30.103 icmp_seq=4 ttl=64 time=0.001 ms
10.2.30.103 icmp_seq=5 ttl=64 time=0.001 ms

client-103> ping 10.2.30.201
84 bytes from 10.2.30.201 icmp_seq=1 ttl=64 time=233.769 ms
84 bytes from 10.2.30.201 icmp_seq=2 ttl=64 time=58.979 ms
84 bytes from 10.2.30.201 icmp_seq=3 ttl=64 time=55.747 ms
84 bytes from 10.2.30.201 icmp_seq=4 ttl=64 time=57.310 ms
84 bytes from 10.2.30.201 icmp_seq=5 ttl=64 time=54.748 ms

client-103> ping 10.2.40.104
84 bytes from 10.2.40.104 icmp_seq=1 ttl=62 time=76.386 ms
84 bytes from 10.2.40.104 icmp_seq=2 ttl=62 time=281.766 ms
84 bytes from 10.2.40.104 icmp_seq=3 ttl=62 time=73.926 ms
84 bytes from 10.2.40.104 icmp_seq=4 ttl=62 time=58.560 ms
84 bytes from 10.2.40.104 icmp_seq=5 ttl=62 time=69.143 ms

client-103> ping 10.0.199.0
84 bytes from 10.0.199.0 icmp_seq=1 ttl=63 time=33.205 ms
84 bytes from 10.0.199.0 icmp_seq=2 ttl=63 time=53.807 ms
84 bytes from 10.0.199.0 icmp_seq=3 ttl=63 time=74.140 ms
84 bytes from 10.0.199.0 icmp_seq=4 ttl=63 time=25.167 ms
84 bytes from 10.0.199.0 icmp_seq=5 ttl=63 time=24.720 ms
```
</details>


<details>
  <summary>проверки с client-104</summary>
  
```
client-104#show port-channel dense 

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
client-104#ping 10.2.10.201
PING 10.2.10.201 (10.2.10.201) 72(100) bytes of data.
80 bytes from 10.2.10.201: icmp_seq=1 ttl=59 time=245 ms
80 bytes from 10.2.10.201: icmp_seq=2 ttl=59 time=237 ms
80 bytes from 10.2.10.201: icmp_seq=3 ttl=59 time=234 ms
80 bytes from 10.2.10.201: icmp_seq=4 ttl=59 time=231 ms
80 bytes from 10.2.10.201: icmp_seq=5 ttl=59 time=227 ms

--- 10.2.10.201 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 49ms
rtt min/avg/max/mdev = 227.808/235.357/245.919/6.151 ms, pipe 5, ipg/ewma 12.303/240.239 ms

client-104#ping 10.2.20.102
PING 10.2.20.102 (10.2.20.102) 72(100) bytes of data.
80 bytes from 10.2.20.102: icmp_seq=1 ttl=60 time=192 ms
80 bytes from 10.2.20.102: icmp_seq=2 ttl=60 time=184 ms
80 bytes from 10.2.20.102: icmp_seq=3 ttl=60 time=186 ms
80 bytes from 10.2.20.102: icmp_seq=4 ttl=60 time=186 ms
80 bytes from 10.2.20.102: icmp_seq=5 ttl=60 time=183 ms

--- 10.2.20.102 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 46ms
rtt min/avg/max/mdev = 183.380/186.805/192.881/3.253 ms, pipe 5, ipg/ewma 11.618/189.699 ms

client-104#ping 10.2.20.201
PING 10.2.20.201 (10.2.20.201) 72(100) bytes of data.
80 bytes from 10.2.20.201: icmp_seq=1 ttl=59 time=355 ms
80 bytes from 10.2.20.201: icmp_seq=2 ttl=59 time=341 ms
80 bytes from 10.2.20.201: icmp_seq=3 ttl=59 time=338 ms
80 bytes from 10.2.20.201: icmp_seq=4 ttl=59 time=333 ms
80 bytes from 10.2.20.201: icmp_seq=5 ttl=59 time=329 ms

--- 10.2.20.201 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 57ms
rtt min/avg/max/mdev = 329.940/339.683/355.879/9.005 ms, pipe 5, ipg/ewma 14.267/347.233 ms

client-104#ping 10.2.30.103
PING 10.2.30.103 (10.2.30.103) 72(100) bytes of data.
80 bytes from 10.2.30.103: icmp_seq=1 ttl=62 time=155 ms
80 bytes from 10.2.30.103: icmp_seq=2 ttl=62 time=146 ms
80 bytes from 10.2.30.103: icmp_seq=3 ttl=62 time=143 ms
80 bytes from 10.2.30.103: icmp_seq=4 ttl=62 time=141 ms
80 bytes from 10.2.30.103: icmp_seq=5 ttl=62 time=137 ms

--- 10.2.30.103 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 48ms
rtt min/avg/max/mdev = 137.365/144.843/155.180/5.978 ms, pipe 5, ipg/ewma 12.041/149.629 ms

client-104#ping 10.2.30.201
PING 10.2.30.201 (10.2.30.201) 72(100) bytes of data.
80 bytes from 10.2.30.201: icmp_seq=1 ttl=63 time=243 ms
80 bytes from 10.2.30.201: icmp_seq=2 ttl=63 time=240 ms
80 bytes from 10.2.30.201: icmp_seq=3 ttl=63 time=233 ms
80 bytes from 10.2.30.201: icmp_seq=4 ttl=63 time=235 ms
80 bytes from 10.2.30.201: icmp_seq=5 ttl=63 time=236 ms

--- 10.2.30.201 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 53ms
rtt min/avg/max/mdev = 233.796/238.036/243.446/3.545 ms, pipe 5, ipg/ewma 13.319/240.579 ms

client-104#ping 10.2.40.104
PING 10.2.40.104 (10.2.40.104) 72(100) bytes of data.
80 bytes from 10.2.40.104: icmp_seq=1 ttl=64 time=1.21 ms
80 bytes from 10.2.40.104: icmp_seq=2 ttl=64 time=0.128 ms
80 bytes from 10.2.40.104: icmp_seq=3 ttl=64 time=0.128 ms
80 bytes from 10.2.40.104: icmp_seq=4 ttl=64 time=0.327 ms
80 bytes from 10.2.40.104: icmp_seq=5 ttl=64 time=0.137 ms

--- 10.2.40.104 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 10ms
rtt min/avg/max/mdev = 0.128/0.387/1.215/0.420 ms, ipg/ewma 2.531/0.788 ms

client-104#ping 10.0.199.0
PING 10.0.199.0 (10.0.199.0) 72(100) bytes of data.
80 bytes from 10.0.199.0: icmp_seq=1 ttl=62 time=132 ms
80 bytes from 10.0.199.0: icmp_seq=2 ttl=62 time=128 ms
80 bytes from 10.0.199.0: icmp_seq=3 ttl=62 time=127 ms
80 bytes from 10.0.199.0: icmp_seq=4 ttl=62 time=122 ms
80 bytes from 10.0.199.0: icmp_seq=5 ttl=62 time=130 ms

--- 10.0.199.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 46ms
rtt min/avg/max/mdev = 122.869/128.146/132.005/3.075 ms, pipe 5, ipg/ewma 11.531/130.024 ms
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
Остальные ping-и (с клиентов уже пинговали)   
server-201#ping vrf vlan-10 10.2.20.201
PING 10.2.20.201 (10.2.20.201) 72(100) bytes of data.
80 bytes from 10.2.20.201: icmp_seq=1 ttl=63 time=72.1 ms
80 bytes from 10.2.20.201: icmp_seq=2 ttl=63 time=67.5 ms
80 bytes from 10.2.20.201: icmp_seq=3 ttl=63 time=69.9 ms
80 bytes from 10.2.20.201: icmp_seq=4 ttl=63 time=68.8 ms
80 bytes from 10.2.20.201: icmp_seq=5 ttl=63 time=64.8 ms

--- 10.2.20.201 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 51ms
rtt min/avg/max/mdev = 64.827/68.636/72.116/2.438 ms, pipe 5, ipg/ewma 12.910/70.247 ms

server-201#ping vrf vlan-10 10.2.30.201 
PING 10.2.30.201 (10.2.30.201) 72(100) bytes of data.
80 bytes from 10.2.30.201: icmp_seq=1 ttl=59 time=296 ms
80 bytes from 10.2.30.201: icmp_seq=2 ttl=59 time=290 ms
80 bytes from 10.2.30.201: icmp_seq=3 ttl=59 time=303 ms
80 bytes from 10.2.30.201: icmp_seq=4 ttl=59 time=301 ms
80 bytes from 10.2.30.201: icmp_seq=5 ttl=59 time=303 ms

--- 10.2.30.201 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 46ms
rtt min/avg/max/mdev = 290.936/298.862/303.196/4.741 ms, pipe 5, ipg/ewma 11.588/297.728 ms

server-201#ping vrf vlan-10 10.0.199.0
PING 10.0.199.0 (10.0.199.0) 72(100) bytes of data.
80 bytes from 10.0.199.0: icmp_seq=1 ttl=62 time=164 ms
80 bytes from 10.0.199.0: icmp_seq=2 ttl=62 time=158 ms
80 bytes from 10.0.199.0: icmp_seq=3 ttl=62 time=154 ms
80 bytes from 10.0.199.0: icmp_seq=4 ttl=62 time=146 ms
80 bytes from 10.0.199.0: icmp_seq=5 ttl=62 time=144 ms

--- 10.0.199.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 46ms
rtt min/avg/max/mdev = 144.593/153.811/164.458/7.368 ms, pipe 5, ipg/ewma 11.642/158.609 ms

server-201#ping vrf vlan-20 10.2.10.201  
PING 10.2.10.201 (10.2.10.201) 72(100) bytes of data.
80 bytes from 10.2.10.201: icmp_seq=1 ttl=63 time=165 ms
80 bytes from 10.2.10.201: icmp_seq=2 ttl=63 time=151 ms
80 bytes from 10.2.10.201: icmp_seq=3 ttl=63 time=159 ms
80 bytes from 10.2.10.201: icmp_seq=4 ttl=63 time=163 ms
80 bytes from 10.2.10.201: icmp_seq=5 ttl=63 time=164 ms

--- 10.2.10.201 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 48ms
rtt min/avg/max/mdev = 151.868/160.948/165.123/4.890 ms, pipe 5, ipg/ewma 12.100/163.241 ms

server-201#ping vrf vlan-20 10.2.30.201 
PING 10.2.30.201 (10.2.30.201) 72(100) bytes of data.
80 bytes from 10.2.30.201: icmp_seq=1 ttl=59 time=354 ms
80 bytes from 10.2.30.201: icmp_seq=2 ttl=59 time=347 ms
80 bytes from 10.2.30.201: icmp_seq=3 ttl=59 time=348 ms
80 bytes from 10.2.30.201: icmp_seq=4 ttl=59 time=347 ms
80 bytes from 10.2.30.201: icmp_seq=5 ttl=59 time=346 ms

--- 10.2.30.201 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 47ms
rtt min/avg/max/mdev = 346.022/348.811/354.325/2.890 ms, pipe 5, ipg/ewma 11.883/351.426 ms

server-201#ping vrf vlan-20 10.0.199.0 
PING 10.0.199.0 (10.0.199.0) 72(100) bytes of data.
80 bytes from 10.0.199.0: icmp_seq=1 ttl=62 time=154 ms
80 bytes from 10.0.199.0: icmp_seq=2 ttl=62 time=156 ms
80 bytes from 10.0.199.0: icmp_seq=3 ttl=62 time=154 ms
80 bytes from 10.0.199.0: icmp_seq=4 ttl=62 time=150 ms
80 bytes from 10.0.199.0: icmp_seq=5 ttl=62 time=145 ms

--- 10.0.199.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 45ms
rtt min/avg/max/mdev = 145.829/152.499/156.258/3.771 ms, pipe 5, ipg/ewma 11.384/153.366 ms

server-201#ping vrf vlan-30 10.2.10.201  
PING 10.2.10.201 (10.2.10.201) 72(100) bytes of data.
80 bytes from 10.2.10.201: icmp_seq=1 ttl=59 time=293 ms
80 bytes from 10.2.10.201: icmp_seq=2 ttl=59 time=286 ms
80 bytes from 10.2.10.201: icmp_seq=3 ttl=59 time=282 ms
80 bytes from 10.2.10.201: icmp_seq=4 ttl=59 time=278 ms
80 bytes from 10.2.10.201: icmp_seq=5 ttl=59 time=289 ms

--- 10.2.10.201 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 45ms

server-201#ping vrf vlan-30 10.2.20.201 
PING 10.2.20.201 (10.2.20.201) 72(100) bytes of data.
80 bytes from 10.2.20.201: icmp_seq=1 ttl=59 time=300 ms
80 bytes from 10.2.20.201: icmp_seq=2 ttl=59 time=301 ms
80 bytes from 10.2.20.201: icmp_seq=3 ttl=59 time=300 ms
80 bytes from 10.2.20.201: icmp_seq=4 ttl=59 time=298 ms
80 bytes from 10.2.20.201: icmp_seq=5 ttl=59 time=295 ms

--- 10.2.20.201 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 46ms
rtt min/avg/max/mdev = 295.147/299.359/301.948/2.428 ms, pipe 5, ipg/ewma 11.513/299.608 ms

server-201#ping vrf vlan-30 10.0.199.0
PING 10.0.199.0 (10.0.199.0) 72(100) bytes of data.
80 bytes from 10.0.199.0: icmp_seq=1 ttl=62 time=152 ms
80 bytes from 10.0.199.0: icmp_seq=2 ttl=62 time=145 ms
80 bytes from 10.0.199.0: icmp_seq=3 ttl=62 time=143 ms
80 bytes from 10.0.199.0: icmp_seq=4 ttl=62 time=138 ms
80 bytes from 10.0.199.0: icmp_seq=5 ttl=62 time=135 ms

--- 10.0.199.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 48ms
rtt min/avg/max/mdev = 135.350/143.151/152.341/5.823 ms, pipe 5, ipg/ewma 12.042/147.344 ms
```
</details>

<details>
  <summary>трассировка между VRF</summary>

- из VRF tenant-1 в VRF tenant-2 \
_client-102 подключен к leaf-103, поэтому в трассировке только leaf-103 и fw-199_
```
client-102> trace 10.2.30.103 - client-103
trace to 10.2.30.103, 8 hops max, press Ctrl+C to stop
 1   10.2.20.254   7.846 ms  5.423 ms  5.018 ms - leaf-103
 2   10.16.25003.244   21.003 ms  20.361 ms  19.515 ms - fw-199
 3   10.16.25003.249   32.576 ms  39.130 ms  38.124 ms - leaf-103
 4   *10.2.30.103   50.223 ms (ICMP type:3, code:3, Destination port unreachable) - client-103
```

- из VRF tenant-2 в VRF tenant-2 \
_client-104 подключен к leaf-101, поэтому в трассировке еще leaf-103_
```
client-104#traceroute 10.2.20.102  - client-102    
traceroute to 10.2.20.102 (10.2.20.102), 30 hops max, 60 byte packets
 1  _gateway (10.2.40.254)  138.553 ms  161.096 ms  168.617 ms - leaf-101/102
 2  10.2.30.254 (10.2.30.254)  175.692 ms  188.585 ms  194.125 ms - leaf-103 (видимо)
 3  10.16.25003.252 (10.16.25003.252)  230.832 ms  241.484 ms  257.261 ms - fw-199
 4  10.16.25003.241 (10.16.25003.241)  438.936 ms  435.455 ms  454.067 ms - leaf-103
 5  10.2.20.102 (10.2.20.102)  465.384 ms  509.310 ms  546.384 ms - client-102
```
</details>

### Итоговые конфигурации оборудования
- [dc1-spine-1](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab8/config/dc1-spine-1.txt)
- [dc1-spine-2](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab8/config/dc1-spine-2.txt)
- [dc1-leaf-101](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab8/config/dc1-leaf-101.txt)
- [dc1-leaf-102](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab8/config/dc1-leaf-102.txt)
- [dc1-leaf-103](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab8/config/dc1-leaf-103.txt)
- [dc1-fw-199](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab8/config/dc1-fw-199.txt)
- [dc1-client-102](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab8/config/dc1-client-102.txt)
- [dc1-client-103](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab8/config/dc1-client-103.txt)
- [dc1-client-104](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab8/config/dc1-client-104.txt)
- [dc1-server-201](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab8/config/dc1-server-201.txt)
---

[**Вернуться обратно**](https://github.com/takmenevag/otus-dc-design/tree/main/)
