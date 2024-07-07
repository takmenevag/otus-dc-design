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
|10.17.0.0/16	|ТШ+резевр|
|10.18.0.0/15	|резерв|
|||
|10.16.0.0/14	|POD1|
|10.16.0.0/16	|Cеть|
|10.17.0.0/16	|ТШ+резевр|
|10.18.0.0/15	|резерв|
---

</details>

### Распределение номеров AS
<details>
  <summary>Таблицы распределения номеров AS</summary>

#### Примечание:
- наименование АСО взята из распределения для 4 байтных номеров AS (leaf > 85 шт.)
- нумерация AS для лабы взята из распределения для 2 байтных номеров AS (leaf < 85 шт.) для облегчения диагностики

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
|		|	|DC		|POD	|ТШ		|Тип	|Номер|
|:-		|:-	|:-		|:-		|:-		|:-		|:-|
|**AS**	|42	|Х		|Х		|ХXX	|ХX		|Х|

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


#### Наимнование АСО
dcX-pX-rXXX-XX-X

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
#### Расположение оборудования по ТШ с резервированием места
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
- все spine одного POD в каждом DC размещены в одной AS 65x0y, где x - DC/POD, y -  третий октет в loopback первого spine POD (.y.)
- каждый leaf размещен в свой AS: leaf-xYY в AS 65xYY, где x - DC/POD, y - из третьего октета loopback leaf (.1yy.)
- на spine используются динамические peer-group с фильтром по номеру AS и транзитному блоку /25
- на leaf используются статические peer-group
- настроены keepalive-интервал 3 сек, hold time 9 сек.
- настроен maximum-paths равным 8 (по максимальному числу spine)
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

### Описание решения DCI
<details>
  <summary>Описание решения</summary>

описание
</details>

### Cхема сети
![Изображение](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab8/scheme/lab8_scheme.PNG "Схема стенда")

### Параметы сети VXLAN
<details>
  <summary>Таблица VRF</summary>
  
#### В решении используется два tenant, возможно масштабирование путем увеличения числа tenant
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

- fw-199
```
```

- server
```

```

- client-102
```
```

- client-103
```

```

- client-104
```
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

```
</details>

<details>
  <summary>трассировка между VRF</summary>

- из VRF tenant-1 в VRF tenant-2 \
_client-102 подключен к leaf-103, поэтому в трассировке только leaf-103 и fw-199_
```

```

- из VRF tenant-2 в VRF tenant-2 \
_client-104 подключен к leaf-101, поэтому в трассировке еще leaf-103_
```
```
</details>

### Масштабирование решение
<details>
  <summary>Описание мастабирования решения</summary>
  
#### Решение поддерживает следующие возможности мастабирования:
- увеличение числа DC до 4 шт. или POD до 2 шт. в каждом DC (с использованием 2 байтных номеров AS)
- увеличение числа DC до 8 шт., POD до 4 шт. в кажом DC (с использованием 4 байтных номеров AS)
- увеличение числа spine в каждом POD до 6-8 шт. (ограничение по количеству uplink-портов на leaf)
- увеличение числа leaf в каждом POD до 89 или с использованием 2 байтных номеров AS
- увеличение числа leaf в каждом POD до 125 шт. с использованием 2 байтных номеров AS в зависимости от: 
	- портовой емкости spine
	- размера транспортного сегмента (сеть с маской /25)
	- физических ограничений по размещению leaf и leaf
- увеличение числа tenant до ~90 шт.
- увеличение числа сетевых сегментов в каждом tenant до исчерпания блока 10.8.0.0/14  

Отдельно отметим, что увеличение числа DC потребует увеличение количества каналов связи между ними. \
В зависимости от итоговой технической возможности по огранизации данных каналов связи возможна \
смена физической архитектуры сегмента DCI
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
