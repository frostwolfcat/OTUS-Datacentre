## VxLAN. + VPC

#### Цель:
Настроить маршрутизацию в рамках Overlay между клиентами


**Описание/Пошаговая инструкция выполнения домашнего задания:**   
* Подключить клиентов 2-я линками к различным Leaf
* Настроить агрегированный канал со стороны клиента
* Настроить VPC для работы в Overlay сети
* План работы, адресное пространство, схема сети, настройки - зафиксированы в документации


**Общие характеристики сети**  
* **Топология** - Сеть Клоса
* **Уровней коммутации** - 2 (Spine,Leaf)
* **Протокол underlay маршрутизации** - eBGP
* **Spine AS** - одна;
* **Leaf AS** - уникальные;
* **Overlay** - BGP L2VPN;
* **VxLAN-cfg** - *только на Leaf нодах*;
* **Образ** - NxOS 9.3

**Параметры BGP** (bold means not default)
* BGP feature - *Enabled*
* Reconnect-interval *12*
* Keep alive interval - *3 seconds*
* Hold timer - *9 seconds*
* BGP PIC core - enabled
* Auto-summary - Always disabled
* Synchronization - Always disabled
* Dynamic capability - enabled
* BGP additional paths - *Enabled*
* ECMP - maximum path - 64
* bestpath - multipath-relax 

**Параметры VxLAN**
* Модель сервиса - VLAN based (Router EVPN-L3)
* VRF - default
* EVPN-VRF - PROD
* update-source - loopback1
* ebgp-multihop - 3
* Тип VxLAN-L3 туннелирования - симметричный
* RD - Ручная настройка
* RT - auto
* BUM - ingress replication bgp

**Параметры VPC**
* VPC-domain - 1 пара в связи с производительностью тестовой среды
* peer-switch - enabled
* peer-keepalive - vrf-management
* peer-gateway - enabled
* auto-recovery - enabled
* ip arp synchronize - enabled
* layer3 peer-router - enabled


#### План настройки eBGP: 

+ Шаг-1 - Настройка route-map для редистрибьюции;
+ Шаг-2 - Включить BGP feature на устройстве;
+ Шаг-3 - Создать BGP инстанс (AS); 
+ Шаг-4 - Настройка BGP опций 
+ Шаг-5 - Настройка шаблонов конфигурации соседств (leaf-side);
+ Шаг-6 - Объявление соседей;
+ Шаг-7 - Проверить связность сетей.

#### План настройки vXLAN:
+ Шаг-1 - Включить evpn-feature на коммутаторах;
+ Шаг-2 - Настройка VLAN и vn-segment
+ Шаг-3 - Настройка VNI
+ Шаг-4 - Настройка RD и RT-auto
+ Шаг-5 - Настройка NVE на Lo1 и указание участников  
+ Шаг-6 - Настройка BGP на VTEP
+ Шаг-7 - Настройка VRF, добавление L3VNI
+ Шаг-8 - Проверка работы VxLAN EVI

#### План настройки VPC:

+ Шаг-1 - Включить feature lacp+vpc для коммутаторов
+ Шаг-2 - Настройка VPC-Domain и приоритетов для коммутаторов
+ Шаг-3 - Настройка secondary IP для VxLAN loopback1
+ Шаг-4 - Настройка VPC-link в сторону серверов
+ Шаг-5 - Проверка работы.


**Параметры VPC для коммутаторов**
|Dev-Name   |VPC-D |Role-pri  |#Комментарий        |
|:---------:|:----:|:--------:|--------------------|
|dc1-leaf-01|14    |100 | #dst-sw-for-vlan41-srv|
|dc1-leaf-04|14    |200 | #dst-sw-for-vlan41-srv|

Дополнения к конфигурации VPC, для корректной работы:

Добавить в конфигурацию для VPC:

Согласно рекомендациям Cisco, для VPC увеличить таймеры, для нивелирования проблем с ложно-положительной генерацией TCN

	switch(config)# spanning-tree vlan 1-3967 hello-time 4
	switch(config)# spanning-tree vlan 1-3967 forward-time 30
	switch(config)# spanning-tree vlan 1-3967 max-age 40

Распределить tcam для arp-supression

	hardware access-list tcam region racl 512
	hardware access-list tcam region arp-ether 256 double-wide
	

Увеличить MTU до 9216

    If the fabric only contains Cisco Nexus 9000 and 7000 series switches, then the
    MTU should be set to 9216.


Убрать проверку peer-as-check для распространения маршрутов между нодами

	neighbor 10.0.1.2 remote-as 65551
	address-family ipv4 unicast
	disable-peer-as-check
	send-community both
	
	
Настроить PIP и RMAC для правильной генерации L3 маршуртов в EVPN

	switch(config)# router bgp 65536
	address-family 12vpn evpn
	advertise-pip
	interface nve 1
	advertise virtual-rmac



**Перечень ASN для маршрутизаторов**

|Dev-Name   |AS    |   RID    | #Комментарий |
|:---------:|:----:|:--------:|--------------|
|dc1-spine-1| 65000| 10.0.1.0 | #Lo1-Spine1  |
|dc1-spine-2| 65000| 10.0.2.0 | #Lo1-Spine2  |
|dc1-leaf-01| 65001| 10.0.1.1 | #Lo1-Leaf1   |
|dc1-leaf-01| 65001| 10.0.1.4 | #Lo1-Leaf4   |
|dc1-leaf-02| 65002| 10.0.1.2 | #Lo1-Leaf2   |
|dc1-leaf-03| 65003| 10.0.1.3 | #Lo1-Leaf3   |

**Перечень RD\VNI для маршрутизаторов**
|   VNI |Type|Dev-Name   |AS    |RD              |#Комментарий       |
|:-----:|:----:|:---------:|:----:|:--------------:|-------------------|
| 70041 |L2		 |dc1-leaf-01| 65001| 10.0.1.141:32808 | #RD-Leaf1         |
| 70041 |L2		 |dc1-leaf-04| 65001| 10.0.1.141:32808 | #RD-Leaf1         |
| 70041 |L2      |dc1-leaf-02| 65002| 10.0.1.2:32808 | #RD-Leaf2         |
| 70041 |L2      |dc1-leaf-03| 65003| 10.0.1.3:32808 | #RD-Leaf3         |
| 70043 |L2      |dc1-leaf-01| 65001| 10.0.1.1:65003 | #RD-Leaf1         |
| 70043 |L2      |dc1-leaf-02| 65002| 10.0.1.2:65003 | #RD-Leaf2         |
| 70043 |L2      |dc1-leaf-03| 65003| 10.0.1.3:65003 | #RD-Leaf3         |
| 10000 |L3      |dc1-leaf-XX| 6500X| 10.0.1.X:10000 | #RD-Leafall		|

#### Схема

![img_5.png](img_5.png)


**Адресный план:**

#### Адресация для хостов
|Dev-Name   |Pn   |Dn           |Sn    |Xn    |Mask|#Комментарий              |
|:---------:|:---:|:-----------:|:----:|:----:|:--:|--------------------------|
|dc1-lf1-srv-01| 10  |    4        |  3   |   3  | /24| #ip-dc1-lf1-srv-01    |
|dc1-lf2-srv-01| 10  |    4        |  3   |   5  | /24| #ip-dc1-lf2-srv-01    |
|dc1-lf3-srv-01| 10  |    4        |  1   |   4  | /24| #ip-dc1-lf3-srv-01    |

#### Адресация Loopback интерфейсов

|Dev-Name   |Pn   |Dn           |Sn    |Xn    |Mask|#Комментарий              |
|:---------:|:---:|:-----------:|:----:|:----:|:--:|--------------------------|
|dc1-spine-1| 10  |    0        |  1   |   0  | /32| #Loopback1-Spine1        |
|dc1-spine-2| 10  |    0        |  2   |   0  | /32| #Loopback1-Spine2        |
|dc1-spine-1| 10  |    1        |  1   |   0  | /32| #Loopback2-Spine1        |
|dc1-spine-2| 10  |    1        |  2   |   0  | /32| #Loopback2-Spine2        |
|dc1-leaf-01| 10  |    0        |  1   |   1  | /32| #Loopback1-Leaf1         |
|dc1-leaf-02| 10  |    0        |  1   |   2  | /32| #Loopback1-Leaf2         |
|dc1-leaf-03| 10  |    0        |  1   |   3  | /32| #Loopback1-Leaf3         |
|dc1-leaf-01| 10  |    1        |  2   |   1  | /32| #Loopback2-Leaf1         |
|dc1-leaf-02| 10  |    1        |  2   |   2  | /32| #Loopback2-Leaf2         |
|dc1-leaf-03| 10  |    1        |  2   |   3  | /32| #Loopback2-Leaf3         |

#### Адресация интерфейсов PtP соединений

|Dev-Name   |Pn   |Dn           |Sn    |Xn    |Mask|#Комментарий              |
|:---------:|:---:|:-----------:|:----:|:----:|:--:|--------------------------|
|dc1-spine-1| 10  |    2        |  1   |   0  | /32| #p2p-link-from-dc1-leaf-01-to-dc1-spine-1|
|dc1-leaf-01| 10  |    2        |  1   |   1  | /32| #p2p-link-from-dc1-leaf-01-to-dc1-spine-1|
|dc1-spine-1| 10  |    2        |  1   |   2  | /32| #p2p-link-from-dc1-leaf-02-to-dc1-spine-1|
|dc1-leaf-02| 10  |    2        |  1   |   3  | /32| #p2p-link-from-dc1-leaf-02-to-dc1-spine-1|
|dc1-spine-1| 10  |    2        |  1   |   4  | /32| #p2p-link-from-dc1-leaf-03-to-dc1-spine-1|
|dc1-leaf-03| 10  |    2        |  1   |   5  | /32| #p2p-link-from-dc1-leaf-03-to-dc1-spine-1|
|dc1-spine-2| 10  |    2        |  2   |   0  | /32| #p2p-link-from-dc1-leaf-01-to-dc1-spine-2|
|dc1-leaf-01| 10  |    2        |  2   |   1  | /32| #p2p-link-from-dc1-leaf-01-to-dc1-spine-2|
|dc1-spine-2| 10  |    2        |  2   |   2  | /32| #p2p-link-from-dc1-leaf-02-to-dc1-spine-2|
|dc1-leaf-02| 10  |    2        |  2   |   3  | /32| #p2p-link-from-dc1-leaf-02-to-dc1-spine-2|
|dc1-spine-2| 10  |    2        |  2   |   4  | /32| #p2p-link-from-dc1-leaf-03-to-dc1-spine-2|
|dc1-leaf-03| 10  |    2        |  2   |   5  | /32| #p2p-link-from-dc1-leaf-03-to-dc1-spine-2|


#### Проверка работы VPC:

*Вывод информации о VPC*

```
dc1-leaf-04# show vpc 
Legend:
                (*) - local vPC is down, forwarding via vPC peer-link

vPC domain id                     : 14  
Peer status                       : peer adjacency formed ok      
vPC keep-alive status             : peer is alive                 
Configuration consistency status  : success 
Per-vlan consistency status       : success                       
Type-2 consistency status         : success 
vPC role                          : secondary, operational primary
Number of vPCs configured         : 2   
Peer Gateway                      : Enabled
Dual-active excluded VLANs        : -
Graceful Consistency Check        : Enabled
Auto-recovery status              : Enabled, timer is off.(timeout = 240s)
Delay-restore status              : Timer is off.(timeout = 240s)
Delay-restore SVI status          : Timer is off.(timeout = 80s)
Operational Layer3 Peer-router    : Enabled
Virtual-peerlink mode             : Disabled

vPC Peer-link status
---------------------------------------------------------------------
id    Port   Status Active vlans    
--    ----   ------ -------------------------------------------------
1     Po1000 up     1,41-43,1000                                                
         

vPC status
----------------------------------------------------------------------------
Id    Port          Status Consistency Reason                Active vlans
--    ------------  ------ ----------- ------                ---------------
6     Po6           up     success     success               41                 
         
                                                                                
         
7     Po7           up     success     success               41                 
         

Please check "show vpc consistency-parameters vpc <vpc-num>" for the 
consistency reason of down vpc and for type-2 consistency reasons for 
any vpc.
```

## Проверка работы L2VNI и L3VNI

Скриншот развернутого qemu Образа линукс Ubuntu с Bond0 агрегированным интерфейсом:

![img_3.png](img_3.png)

Проверка PING с ноды Docker, в сторону сервера Ubuntu с агрегацией интерфейсов в Bond0:

```
root@Docker:/home# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.177.0.2  netmask 255.255.0.0  broadcast 10.177.255.255
        ether 02:42:0a:b1:00:02  txqueuelen 0  (Ethernet)
        RX packets 28  bytes 2152 (2.1 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.4.3.10  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 50:00:00:37:00:01  txqueuelen 1000  (Ethernet)
        RX packets 1276  bytes 86918 (86.9 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 123  bytes 10598 (10.5 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 34  bytes 3248 (3.2 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 34  bytes 3248 (3.2 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

root@Docker:/home# ping 10.4.1.3
PING 10.4.1.3 (10.4.1.3) 56(84) bytes of data.
64 bytes from 10.4.1.3: icmp_seq=1 ttl=62 time=13.9 ms
64 bytes from 10.4.1.3: icmp_seq=2 ttl=62 time=14.5 ms
64 bytes from 10.4.1.3: icmp_seq=3 ttl=62 time=15.3 ms
64 bytes from 10.4.1.3: icmp_seq=4 ttl=62 time=13.2 ms
64 bytes from 10.4.1.3: icmp_seq=5 ttl=62 time=12.6 ms
64 bytes from 10.4.1.3: icmp_seq=6 ttl=62 time=7.54 ms
64 bytes from 10.4.1.3: icmp_seq=7 ttl=62 time=19.3 ms
64 bytes from 10.4.1.3: icmp_seq=8 ttl=62 time=9.90 ms
^C
--- 10.4.1.3 ping statistics ---
8 packets transmitted, 8 received, 0% packet loss, time 7008ms
rtt min/avg/max/mdev = 7.538/13.280/19.252/3.293 ms
```

Проверки к нодам в разных VNI (l2vni and l3vni)

![img_4.png](img_4.png)