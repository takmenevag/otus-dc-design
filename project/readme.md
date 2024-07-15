# Итоговая работа по курсу
[**Вернуться обратно**](https://github.com/takmenevag/otus-dc-design/tree/main/)
## Название работы
- организация геораспределенной отказоустойчивой сети передачи данных центра обработки данных с использованием технологии VXLAN

## Задачи
- построение отказоустойчивой геораспеределенной СПД ЦОД с использованием современной архитектуры
- обеспечение возможности взаимодействия между собой оконечного оборудования и сервисов площадок ЦОД на канальном и сетевом уровнях

## Решение

### Распределение адресного пространства
<details>
  <summary>Описание распределения IP-адресов </summary>

#### Описание:
- для площадки DC1, DC2 и DC**I** испольузется блок IP-адресов 10.Х.0.0/12
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
|10.8.0.0/15	|резерв|
|10.4.0.0/14	|резерв|
|10.8.0.0/14	|сервисы DC-независимые|
|10.12.0.0/14	|резерв сервисы|
|||
|10.0.0.0/16	|DCI транспорт|
|10.0.0.0/24	|транспорт|
|10.0.1.0/24	|резерв транспорт|
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
|10.16.0.0/16	|Cеть+резевр|
|10.17.0.0/16	|резевр|
|10.18.0.0/15	|резерв|
|||
|10.16.0.0/16	|Cеть+резевр|
|…					|…|
|10.16.241.0/24		|dc1-p1-r002-blf-1, r012-blf-1|
|10.16.242.0/24		|резерв|
|10.16.243.0/24		|резерв|
|10.16.244.0/24		|резерв|
|10.16.245.0/24		|резерв|
|10.16.246.0/24		|резерв|
|10.16.247.0/24		|резерв|
|10.16.248.0/24		|резерв| 
|10.16.249.0/24		|dc1-p1-r009-fw-1, r019-fw-1|
|10.16.250.0/25		|dc1-p1-r002-sp-1|
|10.16.250.128/25	|резерв spine|
|10.16.251.0/25		|dc1-p1-r012-sp-1|
|10.16.251.128/25	|резерв spine|
|10.16.252.0/25		|резерв spine|
|10.16.252.128/25	|резерв spine|
|10.16.253.0/25		|резерв spine|
|10.16.253.128/25	|резерв spine|
|10.16.254.0/23		|loopback|
|||
|10.16.254.0/23		|loopback|
|10.16.254.0		|резерв|
|10.16.254.1		|dc1-p1-r002-sp-1|
|10.16.254.2		|dc1-p1-r012-sp-1|
|10.16.254.3		|резерв spine|
|10.16.254.4		|резерв spine|
|10.16.254.5		|резерв spine|
|10.16.254.6		|резерв spine|
|10.16.254.7		|резерв spine|
|10.16.254.8		|резерв spine|
|10.16.254.9		|резерв ss|
|10.16.254.10		|резерв ss|
|10.16.254.11		|dc1-p1-r003-lf-1|
|10.16.254.12		|dc1-p1-r003-lf-2|
|10.16.254.13		|dc1-p1-r013-lf-1|
|10.16.254.14		|dc1-p1-r013-lf-2|
|…					|…|
|10.16.254.187		|dc1-p1-r002-blf-1|
|10.16.254.188		|dc1-p1-r012-blf-1|
|10.16.254.189		|резерв boleaf|
|10.16.254.190		|резерв boleaf|
|10.16.254.191		|dc1-p1-r009-fw-1|

---
#### IP-блоки DC2

|Блок IP-адресов |Назначение|
|:-				|:-|
|10.32.0.0/12	|DC1|
|10.32.0.0/14	|POD1|
|10.36.0.0/14	|POD2|
|10.40.0.0/14	|резерв|
|10.44.0.0/14	|резерв|
|||
|10.32.0.0/14	|POD1|
|10.32.0.0/16	|Cеть+резевр|
|10.33.0.0/16	|резевр|
|10.34.0.0/15	|резерв|
|||
|10.32.0.0/16	|Cеть+резевр|
|…					|…|
|10.32.241.0/24		|dc2-p1-r002-blf-1, r012-blf-1|
|10.32.242.0/24		|резерв|
|10.32.243.0/24		|резерв|
|10.32.244.0/24		|резерв|
|10.32.245.0/24		|резерв|
|10.32.246.0/24		|резерв|
|10.32.247.0/24		|резерв|
|10.32.248.0/24		|резерв| 
|10.32.249.0/24		|dc2-p1-r009-fw-1, r019-fw-1|
|10.32.250.0/25		|dc2-p1-r002-sp-1|
|10.32.250.128/25	|резерв spine|
|10.32.251.0/25		|dc2-p1-r012-sp-1|
|10.32.251.128/25	|резерв spine|
|10.32.252.0/25		|резерв spine|
|10.32.252.128/25	|резерв spine|
|10.32.253.0/25		|резерв spine|
|10.32.253.128/25	|резерв spine|
|10.32.254.0/23		|loopback|
|||
|10.32.254.0/23		|loopback|
|10.32.254.0		|резерв|
|10.32.254.1		|dc2-p1-r002-sp-1|
|10.32.254.2		|dc2-p1-r012-sp-1|
|10.32.254.3		|резерв spine|
|10.32.254.4		|резерв spine|
|10.32.254.5		|резерв spine|
|10.32.254.6		|резерв spine|
|10.32.254.7		|резерв spine|
|10.32.254.8		|резерв spine|
|10.32.254.9		|резерв ss|
|10.32.254.10		|резерв ss|
|10.32.254.11		|dc2-p1-r003-lf-1|
|10.32.254.12		|dc2-p1-r003-lf-2|
|…					|…|
|10.32.254.187		|dc2-p1-r002-blf-1|
|10.32.254.188		|dc2-p1-r012-blf-1|
|10.32.254.189		|резерв boleaf|
|10.32.254.190		|резерв boleaf|
|10.32.254.191		|dc2-p1-r009-fw-1|
---

</details>

### Распределение номеров AS
<details>
  <summary>Описание распределения номеров AS</summary>

#### Примечание:
- для решения 2xDC, 2xPOD или 4xDC, 1xPOD с leaf < 70 шт. используется 2 байтные номера AS
- для решения с leaf > 70 шт. или других комбинаций DC/POD используется 4 байтные номера AS

В связи с чем ниже приведены 2 варианта распределения номеров AS \
Для лабы взята 2 байтные номера AS для облегчения диагностики

---  
#### Для случая 2xDC, 2xPOD или 4xDC, 1xPOD, leaf < 70 шт.
|Тип	|Номер AS	|X,DC/POD 	|Y|
|:-		|:-			|:-			|:-|
|sspine	|65x00		|1-4		|-|	
|spine	|65x0y		|1-4		|1-8|
|leaf	|65xyy		|1-4		|11-84|
|boleaf	|65xyy		|1-4		|85-90|
|fw		|65xyy		|1-4		|91-95|
|br		|65xyy		|1-4		|96-99|
|host	|646yy		|1 DC/POD	|0-99|
|host	|647yy		|2 DC/POD	|0-99|
|host	|648yy		|3 DC/POD	|0-99|
|host	|649yy		|4 DC/POD	|0-99|

---
#### Для остальных вариантов DC/POD или leaf > 70 шт.
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

--- 
#### Наимнование АСО определяется следующим образом
dcX-pX-rXXX-XX-X

---
</details>

### Наимнование и расположение оборудования
<details>
  <summary>Описание расположения оборудования по ТШ </summary>

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
|10	|br		|1		|резерв|
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
|20	|br		|1		|резерв| 

---
#### Итоговая таблица наименования и расположения в DC1
Для лабы взяты ASN 2 байта
|ТШ	|Имя для ASN		|Оборудование		|Cокращение	|ASN 4 байта	|ASN 2 байта|
|:-	|:-					|:-					|:-			|:-				|:-|
|2	|dc1-p1-r002-02-1	|dc1-p1-r002-sp-1	|spine-1	|4211002021		|65101|
|12	|dc1-p1-r012-02-1	|dc1-p1-r012-sp-1	|spine-2	|4211012021		|65101|
|3	|dc1-p1-r003-01-1	|dc1-p1-r003-lf-1	|leaf-11	|4211003011		|65111|
|3	|dc1-p1-r003-01-2	|dc1-p1-r003-lf-2	|leaf-12	|4211003012		|65112|
|13	|dc1-p1-r013-01-1	|dc1-p1-r013-lf-1	|leaf-13	|4211013011		|65113|
|13	|dc1-p1-r013-01-2	|dc1-p1-r013-lf-2	|leaf-14	|4211013012		|65114|
|2	|dc1-p1-r002-01-1	|dc1-p1-r002-blf-1	|boleaf-187	|4211002011		|65187|
|12	|dc1-p1-r012-01-1	|dc1-p1-r012-blf-1	|boleaf-188	|4211012011		|65188|
|9	|dc1-p1-r009-04-1	|dc1-p1-r009-fw-1	|fw-1		|4211009041		|65191|
|19	|dc1-p1-r019-04-1	|dc1-p1-r019-fw-1	|fw-2		|4211019041		|65191|

---
#### Итоговая таблица наименования и расположения в DC2
Для лабы взяты ASN 2 байта
|ТШ	|Имя для ASN		|Оборудование		|Cокращение	|ASN 4 байта	|ASN 2 байта|
|:-	|:-					|:-					|:-			|:-				|:-|
|2	|dc2-p1-r002-02-1	|dc2-p1-r002-sp-1	|spine-1	|4211002021		|65101|
|12	|dc2-p1-r012-02-1	|dc2-p1-r012-sp-1	|spine-2	|4211012021		|65101|
|3	|dc2-p1-r003-01-1	|dc2-p1-r003-lf-1	|leaf-11	|4211003011		|65111|
|3	|dc2-p1-r003-01-2	|dc2-p1-r003-lf-2	|leaf-12	|4211003012		|65112|
|2	|dc2-p1-r002-01-1	|dc2-p1-r002-blf-1	|boleaf-187	|4211002011		|65187|
|12	|dc2-p1-r012-01-1	|dc2-p1-r012-blf-1	|boleaf-188	|4211012011		|65188|
|9	|dc2-p1-r009-04-1	|dc2-p1-r009-fw-1	|fw-1		|4211009041		|65191|
|19	|dc2-p1-r019-04-1	|dc2-p1-r019-fw-1	|fw-2		|4211019041		|65191|
---
</details>

### IP-адресация оборудования и подсетей

<details>
  <summary>IP-адресации DC1</summary>
  
#### В лабе подписи интерфейсов совпадают с 4 октетом loopback (для облегчения просмотра)
|Оборудование		|Интерфейс	|IP-адрес				|Назначение|
|:-					|:-			|:-					|:-|
|dc1-p1-r002-sp-1	|Loopback0	|10.16.254.1/32		|-|
|dc1-p1-r002-sp-1	|Eth1		|10.16.250.0/31		|sp1-lf.11|
|dc1-p1-r002-sp-1	|Eth2		|10.16.250.2/31		|sp1-lf.12|
|dc1-p1-r002-sp-1	|Eth3		|10.16.250.4/31		|sp1-lf.13|
|dc1-p1-r002-sp-1	|Eth4		|10.16.250.6/31		|sp1-lf.14|
|dc1-p1-r002-sp-1	|Eth5		|10.16.250.124/31	|sp1-blf.187|
|dc1-p1-r002-sp-1	|Eth6		|10.16.250.126/31	|sp1-blf.188|
| | | | |
|dc1-p1-r012-sp-1	|Loopback0	|10.16.254.2/32 	|-|
|dc1-p1-r012-sp-1	|Eth1		|10.16.251.0/31		|sp2-lf.11|
|dc1-p1-r012-sp-1	|Eth2		|10.16.251.2/31		|sp2-lf.12|
|dc1-p1-r012-sp-1	|Eth3		|10.16.251.4/31		|sp2-lf.13|
|dc1-p1-r012-sp-1	|Eth4		|10.16.251.6/31		|sp2-lf.14|
|dc1-p1-r012-sp-1	|Eth5		|10.16.251.124/31	|sp2-blf.187|
|dc1-p1-r012-sp-1	|Eth6		|10.16.251.126/31	|sp2-blf.188|
| | | | |
|dc1-p1-r003-lf-1	|Loopback0	|10.16.254.11/32 	|-|
|dc1-p1-r003-lf-1	|Eth1		|10.16.250.1/31		|sp1-lf.11|
|dc1-p1-r003-lf-1	|Eth2		|10.16.251.1/31		|sp2-lf.11|
| | | | |
|dc1-p1-r003-lf-2	|Loopback0	|10.16.254.12/32 		|-|
|dc1-p1-r003-lf-2	|Eth1		|10.16.250.3/31		|sp1-lf.12|
|dc1-p1-r003-lf-2	|Eth2		|10.16.251.3/31		|sp2-lf.12|
| | | | |
|dc1-p1-r013-lf-1	|Loopback0	|10.16.254.13/32 	|-|
|dc1-p1-r013-lf-1	|Eth1		|10.16.250.5/31		|sp1-lf.13|
|dc1-p1-r013-lf-1	|Eth2		|10.16.251.5/31		|sp2-lf.13|
| | | | |
|dc1-p1-r013-lf-2	|Loopback0	|10.16.254.14/32 	|-|
|dc1-p1-r013-lf-2	|Eth1		|10.16.250.7/31		|sp1-lf.14|
|dc1-p1-r013-lf-2	|Eth2		|10.16.251.7/31		|sp2-lf.14|
| | | | |
|dc1-p1-r002-blf-1	|Loopback0	|10.16.254.187/32 	|-|
|dc1-p1-r002-blf-1	|Eth1		|10.16.250.125/31	|sp1-blf.187|
|dc1-p1-r002-blf-1	|Eth2		|10.16.251.125/31	|sp2-blf.187|
|dc1-p1-r002-blf-1	|Eth3		|нет, (Po1)			|dc1-blf.187-dc1-blf.188|
|dc1-p1-r002-blf-1	|Eth4		|нет, (Po1)			|dc1-blf.187-dc1-blf.188|
|dc1-p1-r002-blf-1	|Vlan4093	|10.16.241.0/31		|Po1, ibgp|
|dc1-p1-r002-blf-1	|Vlan4094	|10.16.241.2/31		|Po1, mlag|
|dc1-p1-r002-blf-1	|Eth5		|10.0.0.0/31		|dc1-blf.187-dc2-blf.187|
|dc1-p1-r002-blf-1	|Eth7		|нет, (Po7, mlag)	|blf.187-fw1|
|dc1-p1-r002-blf-1	|Eth8		|нет, (Po8, mlag)	|blf.187-fw2|
|dc1-p1-r002-blf-1	|Vlan4081	|10.16.241.241/29	|dc1-fw-01-cl-tenant-1|
|dc1-p1-r002-blf-1	|Vlan4082	|10.16.241.249/29	|dc1-fw-01-cl-tenant-2|
| | | | |
|dc1-p1-r012-blf-1	|Loopback0	|10.16.254.188/32 	|-|
|dc1-p1-r012-blf-1	|Eth1		|10.16.250.127/31	|sp1-blf.188|
|dc1-p1-r012-blf-1	|Eth2		|10.16.251.127/31	|sp2-blf.188|
|dc1-p1-r012-blf-1	|Eth3		|нет, (Po1)			|dc1-blf.187-dc1-blf.188|
|dc1-p1-r012-blf-1	|Eth4		|нет, (Po1)			|dc1-blf.187-dc1-blf.188|
|dc1-p1-r012-blf-1	|Vlan4093	|10.16.241.1/31		|Po1, ibgp|
|dc1-p1-r012-blf-1	|Vlan4094	|10.16.241.3/31		|Po1, mlag|
|dc1-p1-r012-blf-1	|Eth5		|10.0.0.2/31		|dc1-blf.188-dc2-blf.188|
|dc1-p1-r012-blf-1	|Eth7		|нет, (Po7, mlag)	|blf.188-fw1|
|dc1-p1-r012-blf-1	|Eth8		|нет, (Po8, mlag)	|blf.188-fw2|
|dc1-p1-r012-blf-1	|Vlan4081	|10.16.241.242/29	|dc1-fw-01-cl-tenant-1|
|dc1-p1-r012-blf-1	|Vlan4082	|10.16.241.250/29	|dc1-fw-01-cl-tenant-2|
| | | | |
|dc1-p1-r009-fw-1	|Eth1		|нет, (Po7)			|blf.187-fw1|
|dc1-p1-r009-fw-1	|Eth2		|нет, (Po7)			|blf.188-fw1|
|dc1-p1-r009-fw-1	|Eth3		|нет, (Po8)			|blf.187-fw2|
|dc1-p1-r009-fw-1	|Eth4		|нет, (Po8)			|blf.188-fw1|
|dc1-p1-r009-fw-1	|Po7.4081	|10.16.241.244/29	|tenant-1|
|dc1-p1-r009-fw-1	|Po8.4082	|10.16.241.252/29	|tenant-2|
| | | | |
|dc1-p1-r003-lf-1,2	|Po7	|10.8.10.254/24	|Клиентская сеть, VLAN 10|
|dc1-p1-r003-lf-1,2	|Po7	|10.8.20.254/24	|Клиентская сеть, VLAN 20|
|dc1-p1-r003-lf-1,2	|Po7	|10.8.30.254/24	|Клиентская сеть, VLAN 30|
|dc1-p1-r003-lf-1,2	|Po7	|10.8.40.254/24	|Клиентская сеть, VLAN 40|
|dc1-p1-r003-lf-1,2	|Po8	|10.8.10.254/24	|Клиентская сеть, VLAN 10|
| | | | |
|dc1-p1-r013-lf-1,2	|Po7	|10.8.10.254/24	|Клиентская сеть, VLAN 10|
|dc1-p1-r013-lf-1,2	|Po7	|10.8.30.254/24	|Клиентская сеть, VLAN 30|
| | | | |
|dc1-vlx-s201	|Po7		|10.8.10.201/24	|Клиентская сеть, VLAN 10|
|dc1-vlx-s201	|Po7		|10.8.20.201/24	|Клиентская сеть, VLAN 20|
|dc1-vlx-s201	|Po7		|10.8.30.201/24	|Клиентская сеть, VLAN 30|
|dc1-vlx-s201	|Po7		|10.8.30.201/24	|Клиентская сеть, VLAN 40|
| | | | |
|dc1-vl10-h151	|Po7		|10.8.10.151/24	|Клиентская сеть, VLAN 10|
| | | | |
|dc1-vlx-с101	|Po7		|10.8.10.101/24	|Клиентская сеть, VLAN 10|
|dc1-vlx-с101	|Po7		|10.8.20.101/24	|Клиентская сеть, VLAN 20|
|dc1-vlx-с101	|Po7		|10.8.30.101/24	|Клиентская сеть, VLAN 30|
|dc1-vlx-с101	|Po7		|10.8.30.101/24	|Клиентская сеть, VLAN 40|


----
</details>


<details>
  <summary>IP-адресации DC2</summary>
  
#### В лабе подписи интерфейсов совпадают с 4 октетом loopback (для облегчения просмотра)
|Оборудование		|Интерфейс	|IP-адрес				|Назначение|
|:-					|:-			|:-					|:-|
|dc2-p1-r002-sp-1	|Loopback0	|10.32.254.1/32		|-|
|dc2-p1-r002-sp-1	|Eth1		|10.32.250.0/31		|sp1-lf.11|
|dc2-p1-r002-sp-1	|Eth2		|10.32.250.2/31		|sp1-lf.12|
|dc2-p1-r002-sp-1	|Eth3		|10.32.250.4/31		|sp1-lf.13|
|dc2-p1-r002-sp-1	|Eth4		|10.32.250.6/31		|sp1-lf.14|
|dc2-p1-r002-sp-1	|Eth5		|10.32.250.124/31	|sp1-blf.187|
|dc2-p1-r002-sp-1	|Eth6		|10.32.250.126/31	|sp1-blf.188|
| | | | |
|dc2-p1-r012-sp-1	|Loopback0	|10.32.254.2/32 	|-|
|dc2-p1-r012-sp-1	|Eth1		|10.32.251.0/31		|sp2-lf.11|
|dc2-p1-r012-sp-1	|Eth2		|10.32.251.2/31		|sp2-lf.12|
|dc2-p1-r012-sp-1	|Eth3		|10.32.251.4/31		|sp2-lf.13|
|dc2-p1-r012-sp-1	|Eth4		|10.32.251.6/31		|sp2-lf.14|
|dc2-p1-r012-sp-1	|Eth5		|10.32.251.124/31	|sp2-blf.187|
|dc2-p1-r012-sp-1	|Eth6		|10.32.251.126/31	|sp2-blf.188|
| | | | |
|dc2-p1-r003-lf-1	|Loopback0	|10.32.254.11/32 	|-|
|dc2-p1-r003-lf-1	|Eth1		|10.32.250.1/31		|sp1-lf.11|
|dc2-p1-r003-lf-1	|Eth2		|10.32.251.1/31		|sp2-lf.11|
| | | | |
|dc2-p1-r003-lf-2	|Loopback0	|10.32.254.12/32 		|-|
|dc2-p1-r003-lf-2	|Eth1		|10.32.250.3/31		|sp1-lf.12|
|dc2-p1-r003-lf-2	|Eth2		|10.32.251.3/31		|sp2-lf.12|
| | | | |
|dc2-p1-r002-blf-1	|Loopback0	|10.32.254.187/32 	|-|
|dc2-p1-r002-blf-1	|Eth1		|10.32.250.125/31	|sp1-blf.187|
|dc2-p1-r002-blf-1	|Eth2		|10.32.251.125/31	|sp2-blf.187|
|dc2-p1-r002-blf-1	|Eth3		|нет, (Po1)			|dc2-blf.187-dc2-blf.188|
|dc2-p1-r002-blf-1	|Eth4		|нет, (Po1)			|dc2-blf.187-dc2-blf.188|
|dc2-p1-r002-blf-1	|Vlan4093	|10.32.241.0/31		|Po1, ibgp|
|dc2-p1-r002-blf-1	|Vlan4094	|10.32.241.2/31		|Po1, mlag|
|dc2-p1-r002-blf-1	|Eth5		|10.0.0.1/31		|dc2-blf.187-dc2-blf.187|
|dc2-p1-r002-blf-1	|Eth7		|нет, (Po7, mlag)	|blf.187-fw1|
|dc2-p1-r002-blf-1	|Eth8		|нет, (Po8, mlag)	|blf.187-fw2|
|dc2-p1-r002-blf-1	|Vlan4081	|10.32.241.241/29	|dc2-fw-01-cl-tenant-1|
|dc2-p1-r002-blf-1	|Vlan4082	|10.32.241.249/29	|dc2-fw-01-cl-tenant-2|
| | | | |
|dc2-p1-r012-blf-1	|Loopback0	|10.32.254.188/32 	|-|
|dc2-p1-r012-blf-1	|Eth1		|10.32.250.127/31	|sp1-blf.188|
|dc2-p1-r012-blf-1	|Eth2		|10.32.251.127/31	|sp2-blf.188|
|dc2-p1-r012-blf-1	|Eth3		|нет, (Po1)			|dc2-blf.187-dc2-blf.188|
|dc2-p1-r012-blf-1	|Eth4		|нет, (Po1)			|dc2-blf.187-dc2-blf.188|
|dc2-p1-r012-blf-1	|Vlan4093	|10.32.241.1/31		|Po1, ibgp|
|dc2-p1-r012-blf-1	|Vlan4094	|10.32.241.3/31		|Po1, mlag|
|dc2-p1-r012-blf-1	|Eth5		|10.0.0.3/31		|dc2-blf.188-dc2-blf.188|
|dc2-p1-r012-blf-1	|Eth7		|нет, (Po7, mlag)	|blf.188-fw1|
|dc2-p1-r012-blf-1	|Eth8		|нет, (Po8, mlag)	|blf.188-fw2|
|dc2-p1-r012-blf-1	|Vlan4081	|10.32.241.242/29	|dc2-fw-01-cl-tenant-1|
|dc2-p1-r012-blf-1	|Vlan4082	|10.32.241.250/29	|dc2-fw-01-cl-tenant-2|
| | | | |
|dc2-p1-r009-fw-1	|Eth1		|нет, (Po7)			|blf.187-fw1|
|dc2-p1-r009-fw-1	|Eth2		|нет, (Po7)			|blf.188-fw1|
|dc2-p1-r009-fw-1	|Eth3		|нет, (Po8)			|blf.187-fw2|
|dc2-p1-r009-fw-1	|Eth4		|нет, (Po8)			|blf.188-fw1|
|dc2-p1-r009-fw-1	|Po7.4081	|10.32.241.244/29	|tenant-1|
|dc2-p1-r009-fw-1	|Po8.4082	|10.32.241.252/29	|tenant-2|
| | | | |
|dc2-p1-r003-lf-1,2	|Po7	|10.8.10.254/24	|Клиентская сеть, VLAN 10|
|dc2-p1-r003-lf-1,2	|Po7	|10.8.20.254/24	|Клиентская сеть, VLAN 20|
|dc2-p1-r003-lf-1,2	|Po7	|10.8.30.254/24	|Клиентская сеть, VLAN 30|
|dc2-p1-r003-lf-1,2	|Po7	|10.8.40.254/24	|Клиентская сеть, VLAN 40|
| | | | |
|dc2-vlx-s202	|Po7		|10.8.10.202/24	|Клиентская сеть, VLAN 10|
|dc2-vlx-s202	|Po7		|10.8.20.202/24	|Клиентская сеть, VLAN 20|
|dc2-vlx-s202	|Po7		|10.8.30.202/24	|Клиентская сеть, VLAN 30|
|dc2-vlx-s202	|Po7		|10.8.30.202/24	|Клиентская сеть, VLAN 40|


----
</details>

### Описание решения для Underlay-сети

<details>
  <summary>Описание решения</summary>

#### Описание

С точки зрения физической коммутации в решении предполагается:
- подключение к spine только leaf и border leaf
- подключение к leaf только хостов
- подключение к border leaf
  - межсетевых экранов площадки
  - межплощадочных (DCI) каналов связи
  
В решении используется протокол IPv4 и протокол маршрутизации eBGP и со следующими параметрами:
- все spine одного POD в каждом DC размещены в одной AS 65x0y, где x - DC/POD, y -  третий октет в loopback первого spine POD (.y.)
- каждый leaf размещен в свой AS: leaf-xYY в AS 65xYY, где x - DC/POD, y - из четвертого октета loopback leaf (.1yy)
- каждая пара border leaf размещена в свой AS: leaf-xYY в AS 65xYY, где x - DC/POD, y - из четвертого октета loopback leaf (.1yy)
- настроено соседство между spine и leaf для BGP AFI/SFI ipv4 unicast (eBGP)
- настроено соседство между spine и border leaf для BGP AFI/SFI ipv4 unicast (eBGP)
- настроено соседство между border leaf, формирующих пару, для BGP AFI/SFI ipv4 unicast (iBGP)
- на spine используются динамические peer-group с фильтром по номеру AS и транзитному блоку /25
- на leaf используются статические peer-group
- настроены keepalive-интервал 3 сек, hold time 9 сек.
- настроен maximum-paths равным 8 (по максимальному числу spine)
- настроен BGP routing updates интервал равным 0  (neighbor out-delay, установлен в 0 по умолчанию)
- настроена administrative distance равна 20 (по рекомендации Arista из предоставленной ссылке, возможно из-за iBGP между leaf в паре)
- отключена автоматическая активация BGP AFI/SFI ipv4 unicast (в данной лабе это было не обязательно)
- включен режим multi-agent model (поддежка redistribute в BGP AFI/SFI ipv4 unicast и vxlan)
- включена аутентификация BGP-соседа
- настроено взаимодействие с протоколом bfd для улучшения сходимости сети
- таймеры bfd выбраны такими большими, чтобы сессии в EVE-NG не флапали
</details>

  
### Описание решения для Overlay-сети
<details>
  <summary>Описание решения</summary>

В решении используется следующие параметры и технлогии:
- общие параметры:
	- в overlay используется интерфейс Loopback0 на spine и leaf
	- настроено соседство между spine и leaf для BGP AFI/SFI l2vpn evpn
	- команда neighbor XXX next-hop-unchanged используется для сохранения next-hop-адреса leaf-коммутатора
	- команда redistribute learned используется для анонса MAC-адреса локальных хостов как EVPN type-2 маршрутов
	- команда neighbor XXX send-community extended используется для работы EVPN (импорта, экспорта маршрутов)
	- для маршрутизации трафика в сетевой фабрики используется модель Symmetric IRB
	- механизм ARP Suppression на коммуаторах Arista, используемых в лабе, включен по умолчанию
- маршрутизация клиентских сетей:
	- для настройки шлюза на VTEP используется технология anycast gateway
	- команда ip address virtual используется для задания единого IP-адреса для anycast gateway на всех VTEP, выполняющих функцию шлюза для VLAN
	- команда ip virtual-router mac-address используется для задания единого MAC-адреса для anycast gateway, на всех VTEP, выполняющих функцию шлюза для VLAN
- параметры VNI:
	- номер L2VNI выбирается так - 1ХХХХ, где ХХХХ номер VLAN до 4000
	- номер L3VNI выбирается так - 04YYY, где 4YYY номер VLAN с 4001 по 4070.
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
  	- коммутатору leaf-1 в паре присвоем индекс 0, а leaf-2 индекс 1
	- для определения Designated Forwarder (DF) использует функция mod - <VLAN> mod <количество leaf> (модель сервиса VLAN-based)
	- в качестве DF для всеx VLAN выбран leaf-1 в паре, т.к. номера VLAN деляться на 2 без остатка (10,20,30,40)
	- числовые параметры EVPN Multihoming задаются так:
		- параметр ESI 0000:0x0y:00zz:00pp:0000
		- параметр ES-Import RT 0x0y:00zz:00pp (отбрасываются два байта слева и справа в ESI). Формат записи приведен для облегчения понимания
		- парамет lacp system-id 0x0y.00zz.00pp 
		- x - DC, y - POD, zz - четверый октет в loopback 0, pp - номер Port Channel
- взаимодействие между VRF (tenant):
	- используется два VRF (tenant), в которых размещены все клиентские подсети
	- в tenant №1 размещены VLAN 10 и 20, в tenant №2 размещены VLAN 30 и 40
	- в VLAN 10,20,30,40 размещено по одному серверу в каждом DC (dcY-vlX-s20Y). Серверы реализованы в виде VRF на общей платформе
	- в VLAN 10,20,30,40 размещено по одному клиенту в DC1 (dc1-vlX-c101). Клиенты реализованы в виде VRF на общей платформе
	- в VLAN 10 размещен еще один клиент в DC1 (dc1-vl10-h151).
	- для анонсирования type-5 маршрутов используется команда redistribute connected в каждом VRF секции BGP
	- взаимодействие подсетей из разных VRF, осуществляется через кластер МЭ (dcY-p1-r0x9-fw-1) с использованием технологии VRF-Lite на border leaf (dcY-p1-r0x2-blf-1):
		- в каждом DC border leaf объеденены попарно с использованием технологии MultiChassis Link Aggregation (MLAG)
		- в каждом DC межсетевые экраны объеденены в кластер (в лабе в роли кластера выступает один маршрутизатор)
		- в каждом DC для огранизации отказоустойчивого подключения каждого МЭ к паре border leaf используется два канала связи и технология EtherСhannel (LACP)
		- в каждом DC между кластером МЭ и парой коммутаторов border leaf настроены два транзитных сегмента
		- на каждом border leaf по одному транзитный сегмент помещены в каждый VRF
		- на кластере МЭ настроены оба транзитных сегмента без разделения на VRF
	- в каждом DC взаимодействие межу кластером МЭ и парой border leaf осуществляется с использованием протокола eBGP
	- параметры протокола eBGP заданы аналогичными параметрам eBGP для underlay, кроме поддержки extended community
	- на кластерах МЭ используется BGP AFI/SFI ipv4 unicast
	- в каждом DC кластер МЭ анонсирует в сторону пары border leaf маршрут 10.8.0.0/16 (все разделяемые сегменты)
	- каждый коммутатор border leaf анонсирует клиентские подсети из блока 10.8.0.0/16 в сторону кластер МЭ (являющиеся type-5 маршрутами в EVPN)
	- каждый коммутатор border leaf не анонсирует маршруты до хостов (/32) в сторону кластера МЭ (являющиеся type-2 маршрутами в EVPN)
	- запрет анонса маршрутов /32 реализуется с ипользованием префикс-листа - анонсировать клиентские подсети (из блока 10.8.0.0/16) с маской не длиннее /31
	- для обеспечения корректной (симметричной) маршрутизации трафика в сторону МЭ: 
		- на border leaf DC-2 используется ухудшение метрики маршрута от локалького кластера МЭ
		- ухудшение метрики маршрута осуществляется с использованием маршрутной карты и механизма AS Prepend (3 шт. ASN кластера МЭ)
		- в штатном режиме весь трафик между VRF (tenant) как для DC1, так и для DC2 обрабатывает только кластер МЭ DC1
</details>

### Описание решения DCI
<details>
  <summary>Описание решения</summary>
  
С точки зрения физической коммутации в решении предполагается:
- подключение первого border leaf DC1 к первому border leaf DC2
- подключение второго border leaf DC1 ко второму border leaf DC2

В решении используется следующие параметры и технологии:
- технология Multipod для организации взаимодействия между DC1 и DC2
- между border leaf, находящимися в разных DC, организуется транзитный сегмент /31
- настроено соседство между border leaf для BGP AFI/SFI ipv4 unicast (eBGP)
- на border leaf используются статические peer-group
- настроены keepalive-интервал 3 сек, hold time 9 сек.
- включена аутентификация BGP-соседа
- настроено взаимодействие с протоколом bfd для улучшения сходимости сети
- таймеры bfd выбраны такими большими, чтобы сессии в EVE-NG не флапали
- настроено соседство между border leaf для BGP AFI/SFI l2vpn evpnt (eBGP)
- команда neighbor XXX next-hop-unchanged используется для сохранения next-hop-адреса исходного leaf-коммутатора

</details>

### Cхема решения
<details>
  <summary>Cхема решения</summary>

![Изображение](https://github.com/takmenevag/otus-dc-design/blob/main/labs/lab8/scheme/lab8_scheme.PNG "Схема стенда")
</details>


### Параметы VXLAN в Overlay-сети 
<details>
  <summary>Описание параметров</summary>
  
#### В решении используется два tenant
|VRF	|Тип VNI |Номер VNI	|Номер VLAN	|Значение RT| Значение RD|
|:-			|:-		|:-		|:-		|:-			|:-|
|tenant-1	|L3VNI	|4001	|4001	|4001:4001	|RID:4001|
|tenant-1	|L2VNI	|10010	|10 	|10010:10	|RID:10|
|tenant-1	|L2VNI	|10020	|20		|10020:20	|RID:20|
| | | | |
|tenant-2	|L3VNI	|4002	|4002	|4002:4002	|RID:4002|
|tenant-2	|L2VNI	|10030	|30 	|10010:30	|RID:30|
|tenant-2	|L2VNI	|10040	|40		|10020:40	|RID:40|

#### В решении используется следующие параметры EVPN Multihoming
|DC	|Оборудование 		|Порт	|ESI 						|ES-Import RT 		|LACP system-id|
|:- |:-					|:-		|:-							|:-					|:-|
|1	|dc1-p1-r003-lf-1 	|Po7	|0000:0101:0011:0007:0000 	|01:01:00:11:00:07 	|0101.0011.0007|
|1	|dc1-p1-r003-lf-1 	|Po8	|0000:0101:0011:0008:0000 	|01:01:00:11:00:08 	|0101.0011.0008|
|1	|dc1-p1-r003-lf-2 	|Po7	|0000:0101:0011:0007:0000 	|01:01:00:11:00:07 	|0101.0011.0007|
|1	|dc1-p1-r003-lf-2 	|Po8	|0000:0101:0011:0008:0000 	|01:01:00:11:00:08 	|0101.0011.0008|
|1	|dc1-p1-r013-lf-1 	|Po7	|0000:0101:0013:0007:0000 	|01:01:00:13:00:07 	|0101.0013.0007|
|1	|dc1-p1-r013-lf-2 	|Po7	|0000:0101:0013:0007:0000 	|01:01:00:13:00:07 	|0101.0013.0007|
|	|	|	|	|	|
|2	|dc2-p1-r003-lf-1 	|Po7	|0000:0201:0011:0007:0000 	|02:01:00:11:00:07 	|0201.0011.0007|
|2	|dc2-p1-r003-lf-2 	|Po7	|0000:0201:0011:0007:0000 	|02:01:00:11:00:07 	|0201.0011.0007|

</details>

### Настройка оборудования

<details>
  <summary>Команды для настройки DC1 </summary>

- dc1-p1-r002-sp-1 (spine-1)
```
service routing protocols model multi-agent
!
hostname dc1-p1-r002-sp-1
!
spanning-tree mode mstp
!
interface Ethernet1
   description ### sp1-lf.11 ###
   no switchport
   ip address 10.16.250.0/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet2
   description ### sp1-lf.12 ###
   no switchport
   ip address 10.16.250.2/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet3
   description ### sp1-lf.13 ###
   no switchport
   ip address 10.16.250.4/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet4
   description ### sp1-lf.14 ###
   no switchport
   ip address 10.16.250.6/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet5
   description ### sp1-blf.187 ###
   no switchport
   ip address 10.16.250.124/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet6
   description ### sp1-blf.188 ###
   no switchport
   ip address 10.16.250.126/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Loopback0
   ip address 10.16.254.1/32
!
ip routing
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
peer-filter PF-DC1-LEAF
   10 match as-range 65111-65190 result accept
!
router bgp 65101
   router-id 10.16.254.1
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   bgp listen range 10.16.250.0/25 peer-group DC1-LEAF peer-filter PF-DC1-LEAF
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
- dc1-p1-r012-sp-1 (spine-2)
```
service routing protocols model multi-agent
!
hostname dc1-p1-r012-sp-1
!
spanning-tree mode mstp
!
interface Ethernet1
   description ### sp2-lf.11 ###
   no switchport
   ip address 10.16.251.0/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet2
   description ### sp2-lf.12 ###
   no switchport
   ip address 10.16.251.2/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet3
   description ### sp2-lf.13 ###
   no switchport
   ip address 10.16.251.4/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet4
   description ### sp2-lf.14 ###
   no switchport
   ip address 10.16.251.6/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet5
   description ### sp2-blf.187 ###
   no switchport
   ip address 10.16.251.124/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet6
   description ### sp2-blf.188 ###
   no switchport
   ip address 10.16.251.126/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Loopback0
   ip address 10.16.254.2/32
!
ip routing
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
peer-filter PF-DC1-LEAF
   10 match as-range 65111-65190 result accept
!
router bgp 65101
   router-id 10.16.254.2
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   bgp listen range 10.16.251.0/25 peer-group DC1-LEAF peer-filter PF-DC1-LEAF
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

- dc1-p1-r003-lf-1 (leaf-11)
```
service routing protocols model multi-agent
!
hostname dc1-p1-r003-lf-1
!
spanning-tree mode rapid-pvst
no spanning-tree vlan-id 10,20,30,40
!
vlan 10
   name NET-10.8.10.0/24
!
vlan 20
   name NET-10.8.20.0/24
!
vlan 30
   name NET-10.8.30.0/24
!
vlan 40
   name NET-10.8.40.0/24
!
vrf instance tenant-1
!
vrf instance tenant-2
!
interface Port-Channel7
   description ### dc1-vlx-s201 ###
   switchport trunk allowed vlan 10,20,30,40
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0101:0011:0007:0000
      route-target import 01:01:00:11:00:07
   lacp system-id 0101.0011.0007
!
interface Port-Channel8
   description ### dc1-vl10-h151 ###
   switchport trunk allowed vlan 10
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0101:0011:0008:0000
      route-target import 01:01:00:11:00:08
   lacp system-id 0101.0011.0008
!
interface Ethernet1
   description ### sp1-lf.11 ###
   no switchport
   ip address 10.16.250.1/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet2
   description ### sp2-lf.11 ###
   no switchport
   ip address 10.16.251.1/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet7
   description ### dc1-vlX-s201 ###
   channel-group 7 mode active
!
interface Ethernet8
   description ### dc1-vl10-h151 ###
   channel-group 8 mode active
!
interface Loopback0
   ip address 10.16.254.11/32
!
interface Vlan10
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.8.10.254/24
!
interface Vlan20
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.8.20.254/24
!
interface Vlan30
   description ### client ###
   vrf tenant-2
   arp aging timeout 250
   ip address virtual 10.8.30.254/24
!
interface Vlan40
   description ### client ###
   vrf tenant-2
   arp aging timeout 250
   ip address virtual 10.8.40.254/24
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
router bgp 65111
   router-id 10.16.254.11
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-SPINE peer group
   neighbor DC1-SPINE remote-as 65101
   neighbor DC1-SPINE bfd
   neighbor DC1-SPINE timers 3 9
   neighbor DC1-SPINE password 7 txq0MZ/aCqwJ+sp2WtntdQ==
   neighbor DC1-SPINE send-community extended
   neighbor 10.16.250.0 peer group DC1-SPINE
   neighbor 10.16.250.0 description ### dc1-p1-r002-sp-1 ###
   neighbor 10.16.251.0 peer group DC1-SPINE
   neighbor 10.16.251.0 description ### dc1-p1-r012-sp-1 ###
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
      rd 10.16.254.11:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
      redistribute connected
   !
   vrf tenant-2
      rd 10.16.254.11:4002
      route-target import evpn 4002:4002
      route-target export evpn 4002:4002
      redistribute connected
```

- dc1-p1-r003-lf-2 (leaf-12)
```
service routing protocols model multi-agent
!
hostname dc1-p1-r003-lf-2
!
spanning-tree mode rapid-pvst
no spanning-tree vlan-id 10,20,30,40
!
vlan 10
   name NET-10.8.10.0/24
!
vlan 20
   name NET-10.8.20.0/24
!
vlan 30
   name NET-10.8.30.0/24
!
vlan 40
   name NET-10.8.40.0/24
!
vrf instance tenant-1
!
vrf instance tenant-2
!
interface Port-Channel7
   description ### dc1-vlx-s201 ###
   switchport trunk allowed vlan 10,20,30,40
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0101:0011:0007:0000
      route-target import 01:01:00:11:00:07
   lacp system-id 0101.0011.0007
!
interface Port-Channel8
   description ### dc1-vl10-h151 ###
   switchport trunk allowed vlan 10
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0101:0011:0008:0000
      route-target import 01:01:00:11:00:08
   lacp system-id 0101.0011.0008
!
interface Ethernet1
   description ### sp1-lf.12 ###
   no switchport
   ip address 10.16.250.3/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet2
   description ### sp2-lf.12 ###
   no switchport
   ip address 10.16.251.3/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet7
   description ### dc1-vlX-s201 ###
   channel-group 7 mode active
!
interface Ethernet8
   description ### dc1-vl10-h151 ###
   channel-group 8 mode active
!
interface Loopback0
   ip address 10.16.254.12/32
!
interface Vlan10
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.8.10.254/24
!
interface Vlan20
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.8.20.254/24
!
interface Vlan30
   description ### client ###
   vrf tenant-2
   arp aging timeout 250
   ip address virtual 10.8.30.254/24
!
interface Vlan40
   description ### client ###
   vrf tenant-2
   arp aging timeout 250
   ip address virtual 10.8.40.254/24
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
router bgp 65112
   router-id 10.16.254.12
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-SPINE peer group
   neighbor DC1-SPINE remote-as 65101
   neighbor DC1-SPINE bfd
   neighbor DC1-SPINE timers 3 9
   neighbor DC1-SPINE password 7 txq0MZ/aCqwJ+sp2WtntdQ==
   neighbor DC1-SPINE send-community extended
   neighbor 10.16.250.2 peer group DC1-SPINE
   neighbor 10.16.250.2 description ### dc1-p1-r002-sp-1 ###
   neighbor 10.16.251.2 peer group DC1-SPINE
   neighbor 10.16.251.2 description ### dc1-p1-r012-sp-1 ###
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
      rd 10.16.254.12:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
      redistribute connected
   !
   vrf tenant-2
      rd 10.16.254.12:4002
      route-target import evpn 4002:4002
      route-target export evpn 4002:4002
      redistribute connected
```

- dc1-p1-r013-lf-1 (leaf-13)
```
service routing protocols model multi-agent
!
hostname dc1-p1-r013-lf-1
!
spanning-tree mode rapid-pvst
no spanning-tree vlan-id 10,30
!
vlan 10
   name NET-10.8.10.0/24
!
vlan 30
   name NET-10.8.30.0/24
!
vrf instance tenant-1
!
vrf instance tenant-2
!
interface Port-Channel7
   description ### dc1-vlx-c101 ###
   switchport trunk allowed vlan 10,30
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0101:0013:0007:0000
      route-target import 01:01:00:13:00:07
   lacp system-id 0101.0013.0007
!
interface Ethernet1
   description ### sp1-lf.13 ###
   no switchport
   ip address 10.16.250.5/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet2
   description ### sp2-lf.13 ###
   no switchport
   ip address 10.16.251.5/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet7
   description ### dc1-vlx-c101 ###
   channel-group 7 mode active
!
interface Loopback0
   ip address 10.16.254.13/32
!
interface Vlan10
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.8.10.254/24
!
interface Vlan30
   description ### client ###
   vrf tenant-2
   arp aging timeout 250
   ip address virtual 10.8.30.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
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
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
router bgp 65113
   router-id 10.16.254.13
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-SPINE peer group
   neighbor DC1-SPINE remote-as 65101
   neighbor DC1-SPINE bfd
   neighbor DC1-SPINE timers 3 9
   neighbor DC1-SPINE password 7 txq0MZ/aCqwJ+sp2WtntdQ==
   neighbor DC1-SPINE send-community extended
   neighbor 10.16.250.4 peer group DC1-SPINE
   neighbor 10.16.250.4 description ### dc1-p1-r002-sp-1 ###
   neighbor 10.16.251.4 peer group DC1-SPINE
   neighbor 10.16.251.4 description ### dc1-p1-r012-sp-1 ###
   !
   vlan 10
      rd auto
      route-target both 10010:10
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
      rd 10.16.254.13:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
      redistribute connected
   !
   vrf tenant-2
      rd 10.16.254.13:4002
      route-target import evpn 4002:4002
      route-target export evpn 4002:4002
      redistribute connected
```

- dc1-p1-r013-lf-1 (leaf-14)
```
service routing protocols model multi-agent
!
hostname dc1-p1-r013-lf-2
!
spanning-tree mode rapid-pvst
no spanning-tree vlan-id 10,30
!
vlan 10
   name NET-10.8.10.0/24
!
vlan 30
   name NET-10.8.30.0/24
!
vrf instance tenant-1
!
vrf instance tenant-2
!
interface Port-Channel7
   description ### dc1-vlx-c101 ###
   switchport trunk allowed vlan 10,30
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0101:0013:0007:0000
      route-target import 01:01:00:13:00:07
   lacp system-id 0101.0013.0007
!
interface Ethernet1
   description ### sp1-lf.14 ###
   no switchport
   ip address 10.16.250.7/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet2
   description ### sp2-lf.14 ###
   no switchport
   ip address 10.16.251.7/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet7
   description ### dc1-vlx-c101 ###
   channel-group 7 mode active
!
interface Loopback0
   ip address 10.16.254.14/32
!
interface Vlan10
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.8.10.254/24
!
interface Vlan30
   description ### client ###
   vrf tenant-2
   arp aging timeout 250
   ip address virtual 10.8.30.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
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
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
router bgp 65114
   router-id 10.16.254.14
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-SPINE peer group
   neighbor DC1-SPINE remote-as 65101
   neighbor DC1-SPINE bfd
   neighbor DC1-SPINE timers 3 9
   neighbor DC1-SPINE password 7 txq0MZ/aCqwJ+sp2WtntdQ==
   neighbor DC1-SPINE send-community extended
   neighbor 10.16.250.6 peer group DC1-SPINE
   neighbor 10.16.250.6 description ### dc1-p1-r002-sp-1 ###
   neighbor 10.16.251.6 peer group DC1-SPINE
   neighbor 10.16.251.6 description ### dc1-p1-r012-sp-1 ###
   !
   vlan 10
      rd auto
      route-target both 10010:10
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
      rd 10.16.254.14:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
      redistribute connected
   !
   vrf tenant-2
      rd 10.16.254.14:4002
      route-target import evpn 4002:4002
      route-target export evpn 4002:4002
      redistribute connected
```

- dc1-p1-r002-lf-1 (boleaf-187)
```
service routing protocols model multi-agent
!
hostname dc1-p1-r002-blf-1
!
spanning-tree mode mstp
no spanning-tree vlan-id 4093-4094
!
vlan 4081
   name dc1-fw-01-cl-tenant-1
!
vlan 4082
   name dc1-fw-01-cl-tenant-2
!
vlan 4093-4094
   trunk group mlag_peer
!
vrf instance tenant-1
!
vrf instance tenant-2
!
interface Port-Channel1
   description ### peer link ###
   switchport mode trunk
   switchport trunk group mlag_peer
!
interface Port-Channel7
   description ### blf.187-fw1 ###
   switchport trunk allowed vlan 4081-4082
   switchport mode trunk
   mlag 7
!
interface Port-Channel8
   description ### blf.187-fw2 ###
   switchport trunk allowed vlan 4081-4082
   switchport mode trunk
   mlag 8
!
interface Ethernet1
   description ### sp1-blf.187 ###
   no switchport
   ip address 10.16.250.125/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet2
   description ### sp2-blf.187 ###
   no switchport
   ip address 10.16.251.125/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet3
   description ### dc1-blf.187-dc1-blf.188 ###
   channel-group 1 mode active
!
interface Ethernet4
   description ### dc1-blf.187-dc1-blf.188 ###
   channel-group 1 mode active
!
interface Ethernet5
   description ### dc1-blf.187-dc2-blf.187 ###
   no switchport
   ip address 10.0.0.0/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet7
   description ### blf.187-fw1 ###
   channel-group 7 mode active
!
interface Ethernet8
   description ### blf.187-fw2 ###
   channel-group 8 mode active
!
interface Loopback0
   ip address 10.16.254.187/32
!
interface Vlan4081
   description ### dc1-fw-01-cl-tenant-1 ###
   vrf tenant-1
   ip address 10.16.241.241/29
   arp aging timeout 250
!
interface Vlan4082
   description ### dc1-fw-01-cl-tenant-2 ###
   vrf tenant-2
   ip address 10.16.241.249/29
   arp aging timeout 250
!
interface Vlan4093
   description ### ibgp ###
   ip address 10.16.241.0/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Vlan4094
   description ### mlag ###
   ip address 10.16.241.2/31
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vrf tenant-1 vni 4001
   vxlan vrf tenant-2 vni 4002
!
ip virtual-router mac-address 00:00:00:00:ca:fe
!
ip routing
ip routing vrf tenant-1
ip routing vrf tenant-2
!
ip prefix-list PL-VXLAN-FW-OUT seq 100 permit 10.8.0.0/16 le 31
!
mlag configuration
   domain-id dc1-p1-r002-blf-1
   heartbeat-interval 3000
   local-interface Vlan4094
   peer-address 10.16.241.3
   peer-link Port-Channel1
   reload-delay 180
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
route-map RM-VXLAN-FW-OUT permit 100
   match ip address prefix-list PL-VXLAN-FW-OUT
!
router bgp 65187
   router-id 10.16.254.187
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-FW peer group
   neighbor DC1-FW remote-as 65191
   neighbor DC1-FW timers 3 9
   neighbor DC1-FW password 7 n5uk4I9QUorSZi6ToxJeeg==
   neighbor DC1-SPINE peer group
   neighbor DC1-SPINE remote-as 65101
   neighbor DC1-SPINE bfd
   neighbor DC1-SPINE timers 3 9
   neighbor DC1-SPINE password 7 txq0MZ/aCqwJ+sp2WtntdQ==
   neighbor DC1-SPINE send-community extended
   neighbor DC2-BORDER-LEAF peer group
   neighbor DC2-BORDER-LEAF remote-as 65287
   neighbor DC2-BORDER-LEAF bfd
   neighbor DC2-BORDER-LEAF description ### dc2-p1-r002-blf-1 ###
   neighbor DC2-BORDER-LEAF timers 3 9
   neighbor DC2-BORDER-LEAF password 7 qxaQTd8lFX8Uz7oFIP+BNg==
   neighbor DC2-BORDER-LEAF send-community extended
   neighbor 10.0.0.1 peer group DC2-BORDER-LEAF
   neighbor 10.0.0.1 description ### dc2-p1-r002-blf-1 ###
   neighbor 10.16.241.1 remote-as 65187
   neighbor 10.16.241.1 next-hop-self
   neighbor 10.16.241.1 bfd
   neighbor 10.16.241.1 description ### dc1-p1-r012-blf-1 ###
   neighbor 10.16.241.1 timers 3 9
   neighbor 10.16.241.1 password 7 TO66SYg2RzxuUfr2c/IMpQ==
   neighbor 10.16.250.124 peer group DC1-SPINE
   neighbor 10.16.250.124 description ### dc1-p1-r002-sp-1 ###
   neighbor 10.16.251.124 peer group DC1-SPINE
   neighbor 10.16.251.124 description ### dc1-p1-r012-sp-1 ###
   !
   address-family evpn
      neighbor DC1-SPINE activate
      neighbor DC2-BORDER-LEAF activate
      neighbor DC2-BORDER-LEAF next-hop-unchanged
   !
   address-family ipv4
      neighbor DC1-FW activate
      neighbor DC1-SPINE activate
      neighbor DC2-BORDER-LEAF activate
      neighbor 10.16.241.1 activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
   !
   vrf tenant-1
      rd 10.16.254.187:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
      neighbor 10.16.241.244 peer group DC1-FW
      neighbor 10.16.241.244 description ### dc1-p1-r009-fw-1 ###
      neighbor 10.16.241.244 route-map RM-VXLAN-FW-OUT out
      redistribute connected
   !
   vrf tenant-2
      rd 10.16.254.187:4002
      route-target import evpn 4002:4002
      route-target export evpn 4002:4002
      neighbor 10.16.241.252 peer group DC1-FW
      neighbor 10.16.241.252 description ### dc1-p1-r009-fw-1 ###
      neighbor 10.16.241.252 route-map RM-VXLAN-FW-OUT out
      redistribute connected
```

- dc1-p1-r012-lf-1 (boleaf-188)
```
service routing protocols model multi-agent
!
hostname dc1-p1-r012-blf-1
!
spanning-tree mode mstp
no spanning-tree vlan-id 4093-4094
!
vlan 4081
   name dc1-fw-01-cl-tenant-1
!
vlan 4082
   name dc1-fw-01-cl-tenant-2
!
vlan 4093-4094
   trunk group mlag_peer
!
vrf instance tenant-1
!
vrf instance tenant-2
!
interface Port-Channel1
   description ### peer link ###
   switchport mode trunk
   switchport trunk group mlag_peer
!
interface Port-Channel7
   description ### blf.188-fw1 ###
   switchport trunk allowed vlan 4081-4082
   switchport mode trunk
   mlag 7
!
interface Port-Channel8
   description ### blf.188-fw2 ###
   switchport trunk allowed vlan 4081-4082
   switchport mode trunk
   mlag 8
!
interface Ethernet1
   description ### sp1-blf.188 ###
   no switchport
   ip address 10.16.250.127/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet2
   description ### sp2-blf.188 ###
   no switchport
   ip address 10.16.251.127/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet3
   description ### dc1-blf.187-dc1-blf.188 ###
   channel-group 1 mode active
!
interface Ethernet4
   description ### dc1-blf.187-dc1-blf.188 ###
   channel-group 1 mode active
!
interface Ethernet5
   description ### dc1-blf.188-dc2-blf.188 ###
   no switchport
   ip address 10.0.0.2/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet7
   description ### blf.188-fw1 ###
   channel-group 7 mode active
!
interface Ethernet8
   description ### blf.188-fw2 ###
   channel-group 8 mode active
!
interface Loopback0
   ip address 10.16.254.188/32
!
interface Vlan4081
   description ### dc1-fw-01-cl-tenant-1 ###
   vrf tenant-1
   ip address 10.16.241.242/29
   arp aging timeout 250
!
interface Vlan4082
   description ### dc1-fw-01-cl-tenant-2 ###
   vrf tenant-2
   ip address 10.16.241.250/29
   arp aging timeout 250
!
interface Vlan4093
   description ### ibgp ###
   ip address 10.16.241.1/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Vlan4094
   description ### mlag ###
   ip address 10.16.241.3/31
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vrf tenant-1 vni 4001
   vxlan vrf tenant-2 vni 4002
!
ip virtual-router mac-address 00:00:00:00:ca:fe
!
ip routing
ip routing vrf tenant-1
ip routing vrf tenant-2
!
ip prefix-list PL-VXLAN-FW-OUT seq 100 permit 10.8.0.0/16 le 31
!
mlag configuration
   domain-id dc1-p1-r002-blf-1
   heartbeat-interval 3000
   local-interface Vlan4094
   peer-address 10.16.241.2
   peer-link Port-Channel1
   reload-delay 180
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
route-map RM-VXLAN-FW-OUT permit 100
   match ip address prefix-list PL-VXLAN-FW-OUT
!
router bgp 65187
   router-id 10.16.254.188
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-FW peer group
   neighbor DC1-FW remote-as 65191
   neighbor DC1-FW timers 3 9
   neighbor DC1-FW password 7 n5uk4I9QUorSZi6ToxJeeg==
   neighbor DC1-SPINE peer group
   neighbor DC1-SPINE remote-as 65101
   neighbor DC1-SPINE bfd
   neighbor DC1-SPINE timers 3 9
   neighbor DC1-SPINE password 7 txq0MZ/aCqwJ+sp2WtntdQ==
   neighbor DC1-SPINE send-community extended
   neighbor DC2-BORDER-LEAF peer group
   neighbor DC2-BORDER-LEAF remote-as 65287
   neighbor DC2-BORDER-LEAF bfd
   neighbor DC2-BORDER-LEAF description ### dc2-p1-r012-blf-1 ###
   neighbor DC2-BORDER-LEAF timers 3 9
   neighbor DC2-BORDER-LEAF password 7 qxaQTd8lFX8Uz7oFIP+BNg==
   neighbor DC2-BORDER-LEAF send-community extended
   neighbor 10.0.0.3 peer group DC2-BORDER-LEAF
   neighbor 10.0.0.3 description ### dc2-p1-r012-blf-1 ###
   neighbor 10.16.241.0 remote-as 65187
   neighbor 10.16.241.0 next-hop-self
   neighbor 10.16.241.0 bfd
   neighbor 10.16.241.0 description ### dc1-p1-r002-blf-1 ###
   neighbor 10.16.241.0 timers 3 9
   neighbor 10.16.241.0 password 7 TO66SYg2RzxuUfr2c/IMpQ==
   neighbor 10.16.250.126 peer group DC1-SPINE
   neighbor 10.16.250.126 description ### dc1-p1-r002-sp-1 ###
   neighbor 10.16.251.126 peer group DC1-SPINE
   neighbor 10.16.251.126 description ### dc1-p1-r012-sp-1 ###
   !
   address-family evpn
      neighbor DC1-SPINE activate
      neighbor DC2-BORDER-LEAF activate
      neighbor DC2-BORDER-LEAF next-hop-unchanged
   !
   address-family ipv4
      neighbor DC1-FW activate
      neighbor DC1-SPINE activate
      neighbor DC2-BORDER-LEAF activate
      neighbor 10.16.241.0 activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
   !
   vrf tenant-1
      rd 10.16.254.188:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
      neighbor 10.16.241.244 peer group DC1-FW
      neighbor 10.16.241.244 description ### dc1-p1-r009-fw-1 ###
      neighbor 10.16.241.244 route-map RM-VXLAN-FW-OUT out
      redistribute connected
   !
   vrf tenant-2
      rd 10.16.254.188:4002
      route-target import evpn 4002:4002
      route-target export evpn 4002:4002
      neighbor 10.16.241.252 peer group DC1-FW
      neighbor 10.16.241.252 description ### dc1-p1-r009-fw-1 ###
      neighbor 10.16.241.252 route-map RM-VXLAN-FW-OUT out
      redistribute connected
```

- dc1-p1-r009-fw-1 (fw-1)
```
hostname dc1-p1-r009-fw-1
!
interface Port-channel7
 no ip address
 no negotiation auto
 no mop enabled
 no mop sysid
!
interface Port-channel7.4081
 description ### tenant-1 ###
 encapsulation dot1Q 4081
 ip address 10.16.241.244 255.255.255.248
!
interface Port-channel7.4082
 description ### tenant-2 ###
 encapsulation dot1Q 4082
 ip address 10.16.241.252 255.255.255.248
!
interface Port-channel8
 description ### fw2 (imitation) ###
 no ip address
 no negotiation auto
 no mop enabled
 no mop sysid
!
interface GigabitEthernet1
 description ### blf.187-fw1 ###
 no ip address
 negotiation auto
 no mop enabled
 no mop sysid
 channel-group 7 mode active
!
interface GigabitEthernet2
 description ### blf.188-fw1 ###
 no ip address
 negotiation auto
 no mop enabled
 no mop sysid
 channel-group 7 mode active
!
interface GigabitEthernet3
 description ### blf.187-fw2 (imitation) ###
 no ip address
 negotiation auto
 no mop enabled
 no mop sysid
 channel-group 8 mode active
!
interface GigabitEthernet4
 description ### blf.188-fw2 (imitation) ###
 no ip address
 negotiation auto
 no mop enabled
 no mop sysid
 channel-group 8 mode active
!
router bgp 65191
 bgp router-id 10.16.254.191
 bgp log-neighbor-changes
 bgp bestpath as-path multipath-relax
 aggregate-address 10.8.0.0 255.255.0.0 summary-only
 neighbor DC1-BORDER-LEAF peer-group
 neighbor DC1-BORDER-LEAF remote-as 65187
 neighbor DC1-BORDER-LEAF password cisco
 neighbor DC1-BORDER-LEAF timers 3 9
 neighbor 10.16.241.241 peer-group DC1-BORDER-LEAF
 neighbor 10.16.241.241 description ### dc1-p1-r002-blf-1 tenant-1 ###
 neighbor 10.16.241.242 peer-group DC1-BORDER-LEAF
 neighbor 10.16.241.242 description ### dc1-p1-r012-blf-1 tenant-1 ###
 neighbor 10.16.241.249 peer-group DC1-BORDER-LEAF
 neighbor 10.16.241.249 description ### dc1-p1-r002-blf-1 tenant-2 ###
 neighbor 10.16.241.250 peer-group DC1-BORDER-LEAF
 neighbor 10.16.241.250 description ### dc1-p1-r012-blf-1 tenant-2 ###
 maximum-paths 8
```

- dc1-p1-r019-fw-1 (fw-2)
```
по факту отсутствует, т.к. кластер эмулируется одним устройством
```

- dc1-vlX-s201
```
hostname dc1-vlx-s201
!
vtp mode transparent
!
ip vrf vlan10
 rd 10:10
 route-target export 10:10
 route-target import 10:10
!
ip vrf vlan20
 rd 20:20
 route-target export 20:20
 route-target import 20:20
!
ip vrf vlan30
 rd 30:30
 route-target export 30:30
 route-target import 30:30
!
ip vrf vlan40
 rd 40:40
 route-target export 40:40
 route-target import 40:40
!
spanning-tree mode pvst
no spanning-tree vlan 10,20,30,40
!
vlan 10
 name NET-10.8.10.0/24
!
vlan 20
 name NET-10.8.20.0/24
!
vlan 30
 name NET-10.8.30.0/24
!
vlan 40
 name NET-10.8.40.0/24
!
interface Port-channel7
 description ### uplink ###
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface Ethernet0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 7 mode active
!
interface Ethernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 7 mode active
!
interface Vlan10
 ip vrf forwarding vlan10
 ip address 10.8.10.201 255.255.255.0
 arp timeout 250
!
interface Vlan20
 ip vrf forwarding vlan20
 ip address 10.8.20.201 255.255.255.0
 arp timeout 250
!
interface Vlan30
 ip vrf forwarding vlan30
 ip address 10.8.30.201 255.255.255.0
 arp timeout 250
!
interface Vlan40
 ip vrf forwarding vlan40
 ip address 10.8.40.201 255.255.255.0
 arp timeout 250
!
ip route vrf vlan10 0.0.0.0 0.0.0.0 10.8.10.254
ip route vrf vlan20 0.0.0.0 0.0.0.0 10.8.20.254
ip route vrf vlan30 0.0.0.0 0.0.0.0 10.8.30.254
ip route vrf vlan40 0.0.0.0 0.0.0.0 10.8.40.254
```

- dc1-vlx-c101
```
hostname dc1-vlx-c101
!
vtp mode transparent
!
ip vrf vlan10
 rd 10:10
 route-target export 10:10
 route-target import 10:10
!
ip vrf vlan30
 rd 30:30
 route-target export 30:30
 route-target import 30:30
!
!
spanning-tree mode pvst
no spanning-tree vlan 10,30
!
vlan 10
 name NET-10.8.10.0/24
!
vlan 30
 name NET-10.8.30.0/24
!
interface Port-channel7
 description ### uplink ####
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface Ethernet0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 7 mode active
!
interface Ethernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 7 mode active
!
interface Vlan10
 ip vrf forwarding vlan10
 ip address 10.8.10.101 255.255.255.0
 arp timeout 250
!
interface Vlan30
 ip vrf forwarding vlan30
 ip address 10.8.30.101 255.255.255.0
 arp timeout 250
!
ip route vrf vlan10 0.0.0.0 0.0.0.0 10.8.10.254
ip route vrf vlan30 0.0.0.0 0.0.0.0 10.8.30.254
```

- dc1-vl10-h151
```
hostname dc1-vl10-h151
!
vtp mode transparent
!
spanning-tree mode pvst
no spanning-tree vlan 10
!
vlan 10
 name NET-10.8.10.0/24
!
interface Port-channel8
 description ### uplink ####
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface Ethernet0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 8 mode active
!
interface Ethernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 8 mode active
!
interface Vlan10
 ip address 10.8.10.151 255.255.255.0
 arp timeout 250
!
ip route 0.0.0.0 0.0.0.0 10.8.10.254

```

</details>

<details>
  <summary>Команды для настройки DC2 </summary>

- dc2-p1-r002-sp-1 (spine-1)
```
service routing protocols model multi-agent
!
hostname dc2-p1-r002-sp-1
!
spanning-tree mode mstp
!
interface Ethernet1
   description ### sp1-lf.11 ###
   no switchport
   ip address 10.32.250.0/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet2
   description ### sp1-lf.12 ###
   no switchport
   ip address 10.32.250.2/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet5
   description ### sp1-blf.187 ###
   no switchport
   ip address 10.32.250.124/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet6
   description ### sp1-blf.188 ###
   no switchport
   ip address 10.32.250.126/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Loopback0
   ip address 10.32.254.1/32
!
ip routing
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
peer-filter PF-DC2-LEAF
   10 match as-range 65211-65290 result accept
!
router bgp 65201
   router-id 10.32.254.1
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   bgp listen range 10.32.250.0/25 peer-group DC2-LEAF peer-filter PF-DC2-LEAF
   neighbor DC2-LEAF peer group
   neighbor DC2-LEAF bfd
   neighbor DC2-LEAF timers 3 9
   neighbor DC2-LEAF password 7 jxDqu7630p5V557ZdwocCg==
   neighbor DC2-LEAF send-community extended
   !
   address-family evpn
      neighbor DC2-LEAF activate
      neighbor DC2-LEAF next-hop-unchanged
   !
   address-family ipv4
      neighbor DC2-LEAF activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
```
- dc2-p1-r012-sp-1 (spine-2)
```
service routing protocols model multi-agent
!
hostname dc2-p1-r012-sp-1
!
spanning-tree mode mstp
!
interface Ethernet1
   description ### sp2-lf.11 ###
   no switchport
   ip address 10.32.251.0/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet2
   description ### sp2-lf.12 ###
   no switchport
   ip address 10.32.251.2/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet5
   description ### sp2-blf.187 ###
   no switchport
   ip address 10.32.251.124/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet6
   description ### sp2-blf.188 ###
   no switchport
   ip address 10.32.251.126/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Loopback0
   ip address 10.32.254.2/32
!
ip routing
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
peer-filter PF-DC2-LEAF
   10 match as-range 65211-65290 result accept
!
router bgp 65201
   router-id 10.32.254.2
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   bgp listen range 10.32.251.0/25 peer-group DC2-LEAF peer-filter PF-DC2-LEAF
   neighbor DC2-LEAF peer group
   neighbor DC2-LEAF bfd
   neighbor DC2-LEAF timers 3 9
   neighbor DC2-LEAF password 7 jxDqu7630p5V557ZdwocCg==
   neighbor DC2-LEAF send-community extended
   !
   address-family evpn
      neighbor DC2-LEAF activate
      neighbor DC2-LEAF next-hop-unchanged
   !
   address-family ipv4
      neighbor DC2-LEAF activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
```

- dc2-p1-r003-lf-1 (leaf-11)
```
service routing protocols model multi-agent
!
hostname dc2-p1-r003-lf-1
!
spanning-tree mode rapid-pvst
no spanning-tree vlan-id 10,20,30,40
!
vlan 10
   name NET-10.8.10.0/24
!
vlan 20
   name NET-10.8.20.0/24
!
vlan 30
   name NET-10.8.30.0/24
!
vlan 40
   name NET-10.8.40.0/24
!
vrf instance tenant-1
!
vrf instance tenant-2
!
interface Port-Channel7
   description ### dc2-vlx-s202 ###
   switchport trunk allowed vlan 10,20,30,40
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0201:0011:0007:0000
      route-target import 02:01:00:11:00:07
   lacp system-id 0201.0011.0007
!
interface Ethernet1
   description ### sp1-lf.11 ###
   no switchport
   ip address 10.32.250.1/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet2
   description ### sp2-lf.11 ###
   no switchport
   ip address 10.32.251.1/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet7
   description ### dc2-vlX-s202 ###
   channel-group 7 mode active
!
interface Loopback0
   ip address 10.32.254.11/32
!
interface Vlan10
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.8.10.254/24
!
interface Vlan20
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.8.20.254/24
!
interface Vlan30
   description ### client ###
   vrf tenant-2
   arp aging timeout 250
   ip address virtual 10.8.30.254/24
!
interface Vlan40
   description ### client ###
   vrf tenant-2
   arp aging timeout 250
   ip address virtual 10.8.40.254/24
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
router bgp 65211
   router-id 10.32.254.11
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC2-SPINE peer group
   neighbor DC2-SPINE remote-as 65201
   neighbor DC2-SPINE bfd
   neighbor DC2-SPINE timers 3 9
   neighbor DC2-SPINE password 7 qGRzumdncjjWUCaBzrxQSg==
   neighbor DC2-SPINE send-community extended
   neighbor 10.32.250.0 peer group DC2-SPINE
   neighbor 10.32.250.0 description ### dc2-p1-r002-sp-1 ###
   neighbor 10.32.251.0 peer group DC2-SPINE
   neighbor 10.32.251.0 description ### dc2-p1-r012-sp-1 ###
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
      neighbor DC2-SPINE activate
   !
   address-family ipv4
      neighbor DC2-SPINE activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
   !
   vrf tenant-1
      rd 10.32.254.11:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
      redistribute connected
   !
   vrf tenant-2
      rd 10.32.254.11:4002
      route-target import evpn 4002:4002
      route-target export evpn 4002:4002
      redistribute connected
```

- dc2-p1-r003-lf-2 (leaf-12)
```
service routing protocols model multi-agent
!
hostname dc2-p1-r003-lf-2
!
spanning-tree mode rapid-pvst
no spanning-tree vlan-id 10,20,30,40
!
vlan 10
   name NET-10.8.10.0/24
!
vlan 20
   name NET-10.8.20.0/24
!
vlan 30
   name NET-10.8.30.0/24
!
vlan 40
   name NET-10.8.40.0/24
!
vrf instance tenant-1
!
vrf instance tenant-2
!
interface Port-Channel7
   description ### dc2-vlx-s202 ###
   switchport trunk allowed vlan 10,20,30,40
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0201:0011:0007:0000
      route-target import 02:01:00:11:00:07
   lacp system-id 0201.0011.0007
!
interface Ethernet1
   description ### sp1-lf.12 ###
   no switchport
   ip address 10.32.250.3/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet2
   description ### sp2-lf.12 ###
   no switchport
   ip address 10.32.251.3/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet7
   description ### dc2-vlX-s202 ###
   channel-group 7 mode active
!
interface Loopback0
   ip address 10.32.254.12/32
!
interface Vlan10
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.8.10.254/24
!
interface Vlan20
   description ### client ###
   vrf tenant-1
   arp aging timeout 250
   ip address virtual 10.8.20.254/24
!
interface Vlan30
   description ### client ###
   vrf tenant-2
   arp aging timeout 250
   ip address virtual 10.8.30.254/24
!
interface Vlan40
   description ### client ###
   vrf tenant-2
   arp aging timeout 250
   ip address virtual 10.8.40.254/24
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
router bgp 65212
   router-id 10.32.254.12
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC2-SPINE peer group
   neighbor DC2-SPINE remote-as 65201
   neighbor DC2-SPINE bfd
   neighbor DC2-SPINE timers 3 9
   neighbor DC2-SPINE password 7 qGRzumdncjjWUCaBzrxQSg==
   neighbor DC2-SPINE send-community extended
   neighbor 10.32.250.2 peer group DC2-SPINE
   neighbor 10.32.250.2 description ### dc2-p1-r002-sp-1 ###
   neighbor 10.32.251.2 peer group DC2-SPINE
   neighbor 10.32.251.2 description ### dc2-p1-r012-sp-1 ###
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
      neighbor DC2-SPINE activate
   !
   address-family ipv4
      neighbor DC2-SPINE activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
   !
   vrf tenant-1
      rd 10.32.254.12:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
      redistribute connected
   !
   vrf tenant-2
      rd 10.32.254.12:4002
      route-target import evpn 4002:4002
      route-target export evpn 4002:4002
      redistribute connected
```

- dc2-p1-r002-lf-1 (boleaf-187)
```
service routing protocols model multi-agent
!
hostname dc2-p1-r002-blf-1
!
spanning-tree mode mstp
no spanning-tree vlan-id 4093-4094
!
vlan 4081
   name dc2-fw-01-cl-tenant-1
!
vlan 4082
   name dc2-fw-01-cl-tenant-2
!
vlan 4093-4094
   trunk group mlag_peer
!
vrf instance tenant-1
!
vrf instance tenant-2
!
interface Port-Channel1
   description ### peer link ###
   switchport mode trunk
   switchport trunk group mlag_peer
!
interface Port-Channel7
   description ### blf.187-fw1 ###
   switchport trunk allowed vlan 4081-4082
   switchport mode trunk
   mlag 7
!
interface Port-Channel8
   description ### blf.187-fw2 ###
   switchport trunk allowed vlan 4081-4082
   switchport mode trunk
   mlag 8
!
interface Ethernet1
   description ### sp1-blf.187 ###
   no switchport
   ip address 10.32.250.125/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet2
   description ### sp2-blf.187 ###
   no switchport
   ip address 10.32.251.125/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet3
   description ### dc2-blf.187-dc2-blf.188 ###
   channel-group 1 mode active
!
interface Ethernet4
   description ### dc2-blf.187-dc2-blf.188 ###
   channel-group 1 mode active
!
interface Ethernet5
   description ### dc1-blf.187-dc2-blf.187 ###
   no switchport
   ip address 10.0.0.1/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet7
   description ### blf.187-fw1 ###
   channel-group 7 mode active
!
interface Ethernet8
   description ### blf.187-fw2 ###
   channel-group 8 mode active
!
interface Loopback0
   ip address 10.32.254.187/32
!
interface Vlan4081
   description ### dc2-fw-01-cl-tenant-1 ###
   vrf tenant-1
   ip address 10.32.241.241/29
   arp aging timeout 250
!
interface Vlan4082
   description ### dc2-fw-01-cl-tenant-2 ###
   vrf tenant-2
   ip address 10.32.241.249/29
   arp aging timeout 250
!
interface Vlan4093
   description ### ibgp ###
   ip address 10.32.241.0/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Vlan4094
   description ### mlag ###
   ip address 10.32.241.2/31
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vrf tenant-1 vni 4001
   vxlan vrf tenant-2 vni 4002
!
ip virtual-router mac-address 00:00:00:00:ca:fe
!
ip routing
ip routing vrf tenant-1
ip routing vrf tenant-2
!
ip prefix-list PL-VXLAN-FW-OUT seq 100 permit 10.8.0.0/16 le 31
!
mlag configuration
   domain-id dc2-p1-r002-blf-1
   heartbeat-interval 3000
   local-interface Vlan4094
   peer-address 10.32.241.3
   peer-link Port-Channel1
   reload-delay 180
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
route-map RM-VXLAN-FW-IN permit 100
   set as-path prepend last-as 3
!
route-map RM-VXLAN-FW-OUT permit 100
   match ip address prefix-list PL-VXLAN-FW-OUT
!
router bgp 65287
   router-id 10.32.254.187
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-BORDER-LEAF peer group
   neighbor DC1-BORDER-LEAF remote-as 65187
   neighbor DC1-BORDER-LEAF bfd
   neighbor DC1-BORDER-LEAF description ### dc1-p1-r002-blf-1 ###
   neighbor DC1-BORDER-LEAF timers 3 9
   neighbor DC1-BORDER-LEAF password 7 Kx5NnNr/1zILCleofGUzVQ==
   neighbor DC1-BORDER-LEAF send-community extended
   neighbor DC2-FW peer group
   neighbor DC2-FW remote-as 65291
   neighbor DC2-FW timers 3 9
   neighbor DC2-FW password 7 nD1/U+tg5FIefxSaPDGDLg==
   neighbor DC2-SPINE peer group
   neighbor DC2-SPINE remote-as 65201
   neighbor DC2-SPINE bfd
   neighbor DC2-SPINE timers 3 9
   neighbor DC2-SPINE password 7 qGRzumdncjjWUCaBzrxQSg==
   neighbor DC2-SPINE send-community extended
   neighbor 10.0.0.0 peer group DC1-BORDER-LEAF
   neighbor 10.0.0.0 description ### dc1-p1-r002-blf-1 ###
   neighbor 10.32.241.1 remote-as 65287
   neighbor 10.32.241.1 next-hop-self
   neighbor 10.32.241.1 bfd
   neighbor 10.32.241.1 description ### dc2-p1-r012-blf-1 ###
   neighbor 10.32.241.1 timers 3 9
   neighbor 10.32.241.1 password 7 0OFfVb94pKTz3qtaFJaYxQ==
   neighbor 10.32.250.124 peer group DC2-SPINE
   neighbor 10.32.250.124 description ### dc2-p1-r002-sp-1 ###
   neighbor 10.32.251.124 peer group DC2-SPINE
   neighbor 10.32.251.124 description ### dc2-p1-r012-sp-1 ###
   !
   address-family evpn
      neighbor DC1-BORDER-LEAF activate
      neighbor DC1-BORDER-LEAF next-hop-unchanged
      neighbor DC2-SPINE activate
   !
   address-family ipv4
      neighbor DC1-BORDER-LEAF activate
      neighbor DC2-FW activate
      neighbor DC2-SPINE activate
      neighbor 10.32.241.1 activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
   !
   vrf tenant-1
      rd 10.32.254.187:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
      neighbor 10.32.241.244 peer group DC2-FW
      neighbor 10.32.241.244 description ### dc2-p1-r009-fw-1 ###
      neighbor 10.32.241.244 route-map RM-VXLAN-FW-IN in
      neighbor 10.32.241.244 route-map RM-VXLAN-FW-OUT out
      redistribute connected
   !
   vrf tenant-2
      rd 10.32.254.187:4002
      route-target import evpn 4002:4002
      route-target export evpn 4002:4002
      neighbor 10.32.241.252 peer group DC2-FW
      neighbor 10.32.241.252 description ### dc2-p1-r009-fw-1 ###
      neighbor 10.32.241.252 route-map RM-VXLAN-FW-IN in
      neighbor 10.32.241.252 route-map RM-VXLAN-FW-OUT out
      redistribute connected
```

- dc2-p1-r012-lf-1 (boleaf-188)
```
service routing protocols model multi-agent
!
hostname dc2-p1-r012-blf-1
!
spanning-tree mode mstp
no spanning-tree vlan-id 4093-4094
!
vlan 4081
   name dc2-fw-01-cl-tenant-1
!
vlan 4082
   name dc2-fw-01-cl-tenant-2
!
vlan 4093-4094
   trunk group mlag_peer
!
vrf instance tenant-1
!
vrf instance tenant-2
!
interface Port-Channel1
   description ### peer link ###
   switchport mode trunk
   switchport trunk group mlag_peer
!
interface Port-Channel7
   description ### blf.188-fw1 ###
   switchport trunk allowed vlan 4081-4082
   switchport mode trunk
   mlag 7
!
interface Port-Channel8
   description ### blf.188-fw2 ###
   switchport trunk allowed vlan 4081-4082
   switchport mode trunk
   mlag 8
!
interface Ethernet1
   description ### sp1-blf.188 ###
   no switchport
   ip address 10.32.250.127/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet2
   description ### sp2-blf.188 ###
   no switchport
   ip address 10.32.251.127/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet3
   description ### dc2-blf.187-dc2-blf.188 ###
   channel-group 1 mode active
!
interface Ethernet4
   description ### dc2-blf.187-dc2-blf.188 ###
   channel-group 1 mode active
!
interface Ethernet5
   description ### dc1-blf.188-dc2-blf.188 ###
   no switchport
   ip address 10.0.0.3/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Ethernet7
   description ### blf.188-fw1 ###
   channel-group 7 mode active
!
interface Ethernet8
   description ### blf.188-fw2 ###
   channel-group 8 mode active
!
interface Loopback0
   ip address 10.32.254.188/32
!
interface Vlan4081
   description ### client ###
   vrf tenant-1
   ip address 10.32.241.242/29
   arp aging timeout 250
!
interface Vlan4082
   description ### client ###
   vrf tenant-2
   ip address 10.32.241.250/29
   arp aging timeout 250
!
interface Vlan4093
   description ### ibgp ###
   ip address 10.32.241.1/31
   bfd interval 3000 min-rx 3000 multiplier 3
!
interface Vlan4094
   description ### mlag ###
   ip address 10.32.241.3/31
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vrf tenant-1 vni 4001
   vxlan vrf tenant-2 vni 4002
!
ip virtual-router mac-address 00:00:00:00:ca:fe
!
ip routing
ip routing vrf tenant-1
ip routing vrf tenant-2
!
ip prefix-list PL-VXLAN-FW-OUT seq 100 permit 10.8.0.0/16 le 31
!
mlag configuration
   domain-id dc2-p1-r002-blf-1
   heartbeat-interval 3000
   local-interface Vlan4094
   peer-address 10.32.241.2
   peer-link Port-Channel1
   reload-delay 180
!
route-map RM-CONNECTED-TO-BGP permit 100
   match interface Loopback0
!
route-map RM-VXLAN-FW-IN permit 100
   set as-path prepend last-as 3
!
route-map RM-VXLAN-FW-OUT permit 100
   match ip address prefix-list PL-VXLAN-FW-OUT
!
router bgp 65287
   router-id 10.32.254.188
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 8
   neighbor DC1-BORDER-LEAF peer group
   neighbor DC1-BORDER-LEAF remote-as 65187
   neighbor DC1-BORDER-LEAF bfd
   neighbor DC1-BORDER-LEAF description ### dc1-p1-r012-blf-1 ###
   neighbor DC1-BORDER-LEAF timers 3 9
   neighbor DC1-BORDER-LEAF password 7 Kx5NnNr/1zILCleofGUzVQ==
   neighbor DC1-BORDER-LEAF send-community extended
   neighbor DC2-FW peer group
   neighbor DC2-FW remote-as 65291
   neighbor DC2-FW timers 3 9
   neighbor DC2-FW password 7 nD1/U+tg5FIefxSaPDGDLg==
   neighbor DC2-SPINE peer group
   neighbor DC2-SPINE remote-as 65201
   neighbor DC2-SPINE bfd
   neighbor DC2-SPINE timers 3 9
   neighbor DC2-SPINE password 7 qGRzumdncjjWUCaBzrxQSg==
   neighbor DC2-SPINE send-community extended
   neighbor 10.0.0.2 peer group DC1-BORDER-LEAF
   neighbor 10.0.0.2 description ### dc1-p1-r012-blf-1 ###
   neighbor 10.32.241.0 remote-as 65287
   neighbor 10.32.241.0 next-hop-self
   neighbor 10.32.241.0 bfd
   neighbor 10.32.241.0 description ### dc2-p1-r002-blf-1 ###
   neighbor 10.32.241.0 timers 3 9
   neighbor 10.32.241.0 password 7 0OFfVb94pKTz3qtaFJaYxQ==
   neighbor 10.32.250.126 peer group DC2-SPINE
   neighbor 10.32.250.126 description ### dc2-p1-r002-sp-1 ###
   neighbor 10.32.251.126 peer group DC2-SPINE
   neighbor 10.32.251.126 description ### dc2-p1-r012-sp-1 ###
   !
   address-family evpn
      neighbor DC1-BORDER-LEAF activate
      neighbor DC1-BORDER-LEAF next-hop-unchanged
      neighbor DC2-SPINE activate
   !
   address-family ipv4
      neighbor DC1-BORDER-LEAF activate
      neighbor DC2-FW activate
      neighbor DC2-SPINE activate
      neighbor 10.32.241.0 activate
      redistribute connected route-map RM-CONNECTED-TO-BGP
   !
   vrf tenant-1
      rd 10.32.254.188:4001
      route-target import evpn 4001:4001
      route-target export evpn 4001:4001
      neighbor 10.32.241.244 peer group DC2-FW
      neighbor 10.32.241.244 description ### dc2-p1-r009-fw-1 ###
      neighbor 10.32.241.244 route-map RM-VXLAN-FW-IN in
      neighbor 10.32.241.244 route-map RM-VXLAN-FW-OUT out
      redistribute connected
   !
   vrf tenant-2
      rd 10.32.254.188:4002
      route-target import evpn 4002:4002
      route-target export evpn 4002:4002
      neighbor 10.32.241.252 peer group DC2-FW
      neighbor 10.32.241.252 description ### dc2-p1-r009-fw-1 ###
      neighbor 10.32.241.252 route-map RM-VXLAN-FW-IN in
      neighbor 10.32.241.252 route-map RM-VXLAN-FW-OUT out
      redistribute connected
```

- dc2-p1-r009-fw-1 (fw-1)
```
hostname dc2-p1-r009-fw-1
!
interface Port-channel7
 no ip address
 no negotiation auto
 no mop enabled
 no mop sysid
!
interface Port-channel7.4081
 description ### tenant-1 ###
 encapsulation dot1Q 4081
 ip address 10.32.241.244 255.255.255.248
!
interface Port-channel7.4082
 description ### tenant-2 ###
 encapsulation dot1Q 4082
 ip address 10.32.241.252 255.255.255.248
!
interface Port-channel8
 description ### fw2 (imitation) ###
 no ip address
 no negotiation auto
 no mop enabled
 no mop sysid
!
interface GigabitEthernet1
 description ### blf.187-fw1 ###
 no ip address
 negotiation auto
 no mop enabled
 no mop sysid
 channel-group 7 mode active
!
interface GigabitEthernet2
 description ### blf.188-fw1 ###
 no ip address
 negotiation auto
 no mop enabled
 no mop sysid
 channel-group 7 mode active
!
interface GigabitEthernet3
 description ### blf.187-fw2 (imitation) ###
 no ip address
 negotiation auto
 no mop enabled
 no mop sysid
 channel-group 8 mode active
!
interface GigabitEthernet4
 description ### blf.188-fw2 (imitation) ###
 no ip address
 negotiation auto
 no mop enabled
 no mop sysid
 channel-group 8 mode active
!
router bgp 65291
 bgp router-id 10.32.254.191
 bgp log-neighbor-changes
 bgp bestpath as-path multipath-relax
 aggregate-address 10.8.0.0 255.255.0.0 summary-only
 neighbor DC2-BORDER-LEAF peer-group
 neighbor DC2-BORDER-LEAF remote-as 65287
 neighbor DC2-BORDER-LEAF password cisco
 neighbor DC2-BORDER-LEAF timers 3 9
 neighbor 10.32.241.241 peer-group DC2-BORDER-LEAF
 neighbor 10.32.241.241 description ### dc2-p1-r002-blf-1 tenant-1 ###
 neighbor 10.32.241.242 peer-group DC2-BORDER-LEAF
 neighbor 10.32.241.242 description ### dc2-p1-r012-blf-1 tenant-1 ###
 neighbor 10.32.241.249 peer-group DC2-BORDER-LEAF
 neighbor 10.32.241.249 description ### dc2-p1-r002-blf-1 tenant-2 ###
 neighbor 10.32.241.250 peer-group DC2-BORDER-LEAF
 neighbor 10.32.241.250 description ### dc2-p1-r012-blf-1 tenant-2 ###
 maximum-paths 8
```

- dc2-p1-r019-fw-1 (fw-2)
```
по факту отсутствует, т.к. кластер эмулируется одним устройством
```

- dc2-vlX-s202
```
hostname dc2-vlx-s202
!
vtp mode transparent
!
ip vrf vlan10
 rd 10:10
 route-target export 10:10
 route-target import 10:10
!
ip vrf vlan20
 rd 20:20
 route-target export 20:20
 route-target import 20:20
!
ip vrf vlan30
 rd 30:30
 route-target export 30:30
 route-target import 30:30
!
ip vrf vlan40
 rd 40:40
 route-target export 40:40
 route-target import 40:40
!
spanning-tree mode pvst
no spanning-tree vlan 10,20,30,40
!
vlan 10
 name NET-10.8.10.0/24
!
vlan 20
 name NET-10.8.20.0/24
!
vlan 30
 name NET-10.8.30.0/24
!
vlan 40
 name NET-10.8.40.0/24
!
interface Port-channel7
 description ### uplink ###
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface Ethernet0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 7 mode active
!
interface Ethernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 7 mode active
!
interface Vlan10
 ip vrf forwarding vlan10
 ip address 10.8.10.202 255.255.255.0
 arp timeout 250
!
interface Vlan20
 ip vrf forwarding vlan20
 ip address 10.8.20.202 255.255.255.0
 arp timeout 250
!
interface Vlan30
 ip vrf forwarding vlan30
 ip address 10.8.30.202 255.255.255.0
 arp timeout 250
!
interface Vlan40
 ip vrf forwarding vlan40
 ip address 10.8.40.202 255.255.255.0
 arp timeout 250
!
ip route vrf vlan10 0.0.0.0 0.0.0.0 10.8.10.254
ip route vrf vlan20 0.0.0.0 0.0.0.0 10.8.20.254
ip route vrf vlan30 0.0.0.0 0.0.0.0 10.8.30.254
ip route vrf vlan40 0.0.0.0 0.0.0.0 10.8.40.254
```

</details>

### Вывод ip/mac хостов

<details>
  <summary>Вывод ip/mac хостов </summary>

- dc1-vlx-s201 
```
dc1-vlx-s201#show interfaces | i address|Vlan
Vlan10 is up, line protocol is up 
  Hardware is Ethernet SVI, address is aabb.cc81.5000 (bia aabb.cc81.5000)
  Internet address is 10.8.10.201/24
Vlan20 is up, line protocol is up 
  Hardware is Ethernet SVI, address is aabb.cc81.5000 (bia aabb.cc81.5000)
  Internet address is 10.8.20.201/24
Vlan30 is up, line protocol is up 
  Hardware is Ethernet SVI, address is aabb.cc81.5000 (bia aabb.cc81.5000)
  Internet address is 10.8.30.201/24
Vlan40 is up, line protocol is up 
  Hardware is Ethernet SVI, address is aabb.cc81.5000 (bia aabb.cc81.5000)
  Internet address is 10.8.40.201/24
```

- dc1-vlx-с101 
```
dc1-vlx-c101#show interfaces | i address|Vlan
Vlan10 is up, line protocol is up 
  Hardware is Ethernet SVI, address is aabb.cc81.7000 (bia aabb.cc81.7000)
  Internet address is 10.8.10.101/24
Vlan30 is up, line protocol is up 
  Hardware is Ethernet SVI, address is aabb.cc81.7000 (bia aabb.cc81.7000)
  Internet address is 10.8.30.101/24
```

- dc1-vl10-h151 
```
dc1-vl10-h151#show interfaces | i address|Vlan
Vlan10 is up, line protocol is up 
  Hardware is Ethernet SVI, address is aabb.cc81.6000 (bia aabb.cc81.6000)
  Internet address is 10.8.10.151/24
```

- dc2-vlx-s202 
```
dc2-vlx-s202#show interfaces | i address|Vlan
Vlan10 is up, line protocol is up 
  Hardware is Ethernet SVI, address is aabb.cc81.f000 (bia aabb.cc81.f000)
  Internet address is 10.8.10.202/24
Vlan20 is up, line protocol is up 
  Hardware is Ethernet SVI, address is aabb.cc81.f000 (bia aabb.cc81.f000)
  Internet address is 10.8.20.202/24
Vlan30 is up, line protocol is up 
  Hardware is Ethernet SVI, address is aabb.cc81.f000 (bia aabb.cc81.f000)
  Internet address is 10.8.30.202/24
Vlan40 is up, line protocol is up 
  Hardware is Ethernet SVI, address is aabb.cc81.f000 (bia aabb.cc81.f000)
  Internet address is 10.8.40.202/24
```

</details>

### Проверка взаимодействия DC1

<details>
  <summary>Проверки dc1-p1-r002-sp-1 (spine-1)</summary>
  
```
dc1-p1-r002-sp-1#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.16.254.1, local AS number 65101
Neighbor Status Codes: m - Under maintenance
  Neighbor      V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.16.250.1   4 65111            699       677    0    0 00:24:29 Estab   1      1
  10.16.250.3   4 65112            712       675    0    0 00:24:30 Estab   1      1
  10.16.250.5   4 65113            697       684    0    0 00:24:29 Estab   1      1
  10.16.250.7   4 65114            699       689    0    0 00:24:29 Estab   1      1
  10.16.250.125 4 65187            702       685    0    0 00:24:29 Estab   8      8
  10.16.250.127 4 65187            684       651    0   19 00:24:29 Estab   8      8
```
```
dc1-p1-r002-sp-1#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.16.254.1, local AS number 65101
Neighbor Status Codes: m - Under maintenance
  Neighbor      V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.16.250.1   4 65111            707       685    0    0 00:24:51 Estab   27     27
  10.16.250.3   4 65112            721       683    0    0 00:24:51 Estab   27     27
  10.16.250.5   4 65113            705       692    0    0 00:24:51 Estab   12     12
  10.16.250.7   4 65114            708       698    0    0 00:24:51 Estab   12     12
  10.16.250.125 4 65187            710       693    0    0 00:24:50 Estab   52     52
  10.16.250.127 4 65187            691       660    0   19 00:24:51 Estab   52     52
```
```
dc1-p1-r002-sp-1#show ip bgp vrf all
BGP routing table information for VRF default
Router identifier 10.16.254.1, local AS number 65101
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.16.254.1/32         -                     -       -          -       0       i
 * >      10.16.254.11/32        10.16.250.1           0       -          100     0       65111 i
 * >      10.16.254.12/32        10.16.250.3           0       -          100     0       65112 i
 * >      10.16.254.13/32        10.16.250.5           0       -          100     0       65113 i
 * >      10.16.254.14/32        10.16.250.7           0       -          100     0       65114 i
 * >Ec    10.16.254.187/32       10.16.250.127         0       -          100     0       65187 i
 *  ec    10.16.254.187/32       10.16.250.125         0       -          100     0       65187 i
 * >Ec    10.16.254.188/32       10.16.250.127         0       -          100     0       65187 i
 *  ec    10.16.254.188/32       10.16.250.125         0       -          100     0       65187 i
 * >Ec    10.32.254.1/32         10.16.250.127         0       -          100     0       65187 65287 65201 i
 *  ec    10.32.254.1/32         10.16.250.125         0       -          100     0       65187 65287 65201 i
 * >Ec    10.32.254.2/32         10.16.250.127         0       -          100     0       65187 65287 65201 i
 *  ec    10.32.254.2/32         10.16.250.125         0       -          100     0       65187 65287 65201 i
 * >Ec    10.32.254.11/32        10.16.250.127         0       -          100     0       65187 65287 65201 65211 i
 *  ec    10.32.254.11/32        10.16.250.125         0       -          100     0       65187 65287 65201 65211 i
 * >Ec    10.32.254.12/32        10.16.250.127         0       -          100     0       65187 65287 65201 65212 i
 *  ec    10.32.254.12/32        10.16.250.125         0       -          100     0       65187 65287 65201 65212 i
 * >Ec    10.32.254.187/32       10.16.250.127         0       -          100     0       65187 65287 i
 *  ec    10.32.254.187/32       10.16.250.125         0       -          100     0       65187 65287 i
 * >Ec    10.32.254.188/32       10.16.250.127         0       -          100     0       65187 65287 i
 *  ec    10.32.254.188/32       10.16.250.125         0       -          100     0       65187 65287 i
```

</details>

<details>
  <summary>Проверки dc1-p1-r012-sp-1 (spine-2)</summary>
  
```
dc1-p1-r012-sp-1#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.16.254.2, local AS number 65101
Neighbor Status Codes: m - Under maintenance
  Neighbor      V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.16.251.1   4 65111            674       656    0    0 00:23:32 Estab   1      1
  10.16.251.3   4 65112            671       651    0    0 00:23:32 Estab   1      1
  10.16.251.5   4 65113            672       667    0    0 00:23:33 Estab   1      1
  10.16.251.7   4 65114            671       658    0    0 00:23:31 Estab   1      1
  10.16.251.125 4 65187            666       642    0    0 00:23:32 Estab   8      8
  10.16.251.127 4 65187            661       644    0   19 00:23:31 Estab   8      8
```
```
dc1-p1-r012-sp-1#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.16.254.2, local AS number 65101
Neighbor Status Codes: m - Under maintenance
  Neighbor      V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.16.251.1   4 65111            682       664    0    0 00:23:54 Estab   27     27
  10.16.251.3   4 65112            679       660    0    0 00:23:54 Estab   27     27
  10.16.251.5   4 65113            680       676    0    0 00:23:55 Estab   12     12
  10.16.251.7   4 65114            679       667    0    0 00:23:53 Estab   12     12
  10.16.251.125 4 65187            674       650    0    0 00:23:54 Estab   52     52
  10.16.251.127 4 65187            670       653    0   19 00:23:53 Estab   52     52
```
```
dc1-p1-r012-sp-1#show ip bgp vrf all
BGP routing table information for VRF default
Router identifier 10.16.254.2, local AS number 65101
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.16.254.2/32         -                     -       -          -       0       i
 * >      10.16.254.11/32        10.16.251.1           0       -          100     0       65111 i
 * >      10.16.254.12/32        10.16.251.3           0       -          100     0       65112 i
 * >      10.16.254.13/32        10.16.251.5           0       -          100     0       65113 i
 * >      10.16.254.14/32        10.16.251.7           0       -          100     0       65114 i
 * >Ec    10.16.254.187/32       10.16.251.125         0       -          100     0       65187 i
 *  ec    10.16.254.187/32       10.16.251.127         0       -          100     0       65187 i
 * >Ec    10.16.254.188/32       10.16.251.125         0       -          100     0       65187 i
 *  ec    10.16.254.188/32       10.16.251.127         0       -          100     0       65187 i
 * >Ec    10.32.254.1/32         10.16.251.125         0       -          100     0       65187 65287 65201 i
 *  ec    10.32.254.1/32         10.16.251.127         0       -          100     0       65187 65287 65201 i
 * >Ec    10.32.254.2/32         10.16.251.125         0       -          100     0       65187 65287 65201 i
 *  ec    10.32.254.2/32         10.16.251.127         0       -          100     0       65187 65287 65201 i
 * >Ec    10.32.254.11/32        10.16.251.125         0       -          100     0       65187 65287 65201 65211 i
 *  ec    10.32.254.11/32        10.16.251.127         0       -          100     0       65187 65287 65201 65211 i
 * >Ec    10.32.254.12/32        10.16.251.125         0       -          100     0       65187 65287 65201 65212 i
 *  ec    10.32.254.12/32        10.16.251.127         0       -          100     0       65187 65287 65201 65212 i
 * >Ec    10.32.254.187/32       10.16.251.125         0       -          100     0       65187 65287 i
 *  ec    10.32.254.187/32       10.16.251.127         0       -          100     0       65187 65287 i
 * >Ec    10.32.254.188/32       10.16.251.125         0       -          100     0       65187 65287 i
 *  ec    10.32.254.188/32       10.16.251.127         0       -          100     0       65187 65287 i
```

</details>

<details>
  <summary>Проверки dc1-p1-r003-lf-1 (leaf-11)</summary>
  
```

```

</details>

<details>
  <summary>Проверки dc1-p1-r003-lf-2 (leaf-12)</summary>
  
```

```

</details>

<details>
  <summary>Проверки dc1-p1-r013-lf-1 (leaf-13)</summary>
  
```

```

</details>

<details>
  <summary>Проверки dc1-p1-r013-lf-1 (leaf-14)</summary>
  
```

```

</details>

<details>
  <summary>Проверки dc1-p1-r002-lf-1 (boleaf-187)</summary>
  
```

```

</details>

<details>
  <summary>Проверки dc1-p1-r012-lf-1 (boleaf-188)</summary>
  
```

```

</details>

<details>
  <summary>Проверки dc1-p1-r009-fw-1 (fw-1)</summary>
  
```

```

</details>

### Проверка взаимодействия DC2


<details>
  <summary>Проверки dc2-p1-r002-sp-1 (spine-1)</summary>
  
```

```

</details>

<details>
  <summary>Проверки dc2-p1-r012-sp-1 (spine-2)</summary>
  
```

```

</details>

<details>
  <summary>Проверки dc2-p1-r003-lf-1 (leaf-11)</summary>
  
```

```

</details>

<details>
  <summary>Проверки dc2-p1-r003-lf-2 (leaf-12)</summary>
  
```

```

</details>

<details>
  <summary>Проверки dc2-p1-r013-lf-1 (leaf-13)</summary>
  
```

```

</details>

<details>
  <summary>Проверки dc2-p1-r013-lf-1 (leaf-14)</summary>
  
```

```

</details>

<details>
  <summary>Проверки dc2-p1-r002-lf-1 (boleaf-187)</summary>
  
```

```

</details>

<details>
  <summary>Проверки dc2-p1-r012-lf-1 (boleaf-188)</summary>
  
```

```

</details>

<details>
  <summary>Проверки dc2-p1-r009-fw-1 (fw-1)</summary>
  
```

```

</details>

<details>
  <summary>ping внутри и между VRF</summary>

- из VRF tenant-1 в VRF tenant-2 \
_client-102 подключен к leaf-103, поэтому в трассировке только leaf-103 и fw-199_
```

```

- из VRF tenant-2 в VRF tenant-2 \
_client-104 подключен к leaf-101, поэтому в трассировке еще leaf-103_
```
```
</details>


<details>
  <summary>Трассировка внутри и между VRF</summary>

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
- увеличение числа leaf в каждом POD до 70 шт. или с использованием 2 байтных номеров AS
- увеличение числа leaf в каждом POD до 127 шт. с использованием 2 байтных номеров AS в зависимости от: 
	- портовой емкости spine (128 портов)
	- размера транспортного сегмента (сеть с маской /25)
	- физических ограничений по размещению leaf (1U) и spine (4U)
- увеличение числа tenant до 90 шт.
- увеличение числа разделяемыз сетевых сегментов до исчерпания блока 10.8.0.0/14  

Отдельно отметим, что увеличение числа DC потребует увеличение количества каналов связи между ними. \
В зависимости от итоговой технической возможности по огранизации данных каналов связи, возможно \
изменение физической архитектуры сегмента DCI
</details>

### Итоговые конфигурации оборудования DC1
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

### Итоговые конфигурации оборудования DC2
---

[**Вернуться обратно**](https://github.com/takmenevag/otus-dc-design/tree/main/)
