## Underlay. OSPF

Цель:
Настроить OSPF для Underlay сети


**Описание/Пошаговая инструкция выполнения домашнего задания:**   
* настроить OSPF в Underlay сети, для IP связанности между всеми устройствами NXOS
* План работы, адресное пространство, схема сети, настройки - зафиксированы в документации
* План работы, 
  * Выполнить настройки следуя рекомендациям; 
  * Выгрузить настройки в документацию.

**Общие характеристики сети**  
* **Топология** - Сеть Клоса
* **Уровней коммутации** - 2 (Spine,Leaf)
* **Протокол** - OSPFv2
* **Тип OSPF сети** - PtP
* **RID для OSPF** - Lo1
* **Area** - 0.0.0.0 (backbone)
* **OSPF в** Global routing table
* **Аутентификация** в MD5
* **Образ** - NxOS 9.3

**Параметры OSPF**
* Administrative distance	- 110
* Hello interval - 10 seconds
* Dead interval - 40 seconds
* Discard routes - Enabled
* Graceful restart grace period - 60 seconds
* OSPFv2 feature - **Enabled**
* Stub router advertisement announce time - 600 seconds
* Reference bandwidth for link cost calculation - 40 Gb/s
* LSA minimal arrival time - 1000 milliseconds
* LSA group pacing - 10 seconds
* SPF calculation initial delay time - 200 milliseconds
* SPF calculation minimum hold time - 1000 milliseconds
* SPF calculation maximum wait time - 5000 milliseconds
* Для развертывания underlay использую минимально-необходимую настройку OSPF, с дефолтными значениями параметров.

**Перечень RID**

|Dev-Name   |Pn   |Dn           |Sn    |Xn    |Mask|#Комментарий        |
|:---------:|:---:|:-----------:|:----:|:----:|:--:|--------------------|
|dc1-spine-1| 10  |    0        |  1   |   0  | /32| #RID-Spine1        |
|dc1-spine-2| 10  |    0        |  2   |   0  | /32| #RID-Spine2        |
|dc1-leaf-01| 10  |    0        |  1   |   1  | /32| #RID-Leaf1         |
|dc1-leaf-02| 10  |    0        |  1   |   2  | /32| #RID-Leaf2         |
|dc1-leaf-03| 10  |    0        |  1   |   3  | /32| #RID-Leaf3         |




**Глобальные настройки**
- feature ospf
- router ospf 100
- router-id 10.0.2.0
- log-adjacency-changes
- area 0.0.0.0 authentication message-digest
- maximum-paths 4
- passive-interface default

**interface Ethernet1/1-3
  - ip ospf network point-to-point
  - ip ospf passive-interface
  - ip router ospf 100 area 0.0.0.0
  - no shutdown
  - ip ospf message-digest-key 100 md5 0 otus