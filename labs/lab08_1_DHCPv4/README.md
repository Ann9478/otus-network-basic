# Реализация DHCPv4.

###  Топология:
![](./topology.png)

###  Таблица адресации:

| Устройство  | Интерфейс   | IP-адрес      | Маска подсети    | Шлюз по умолчанию  |
|-------------|-------------|---------------|------------------|--------------------|
|     R1      |   G0/0      | 10.0.0.1      | 255.255.255.252  |                    |
|     R1      |   G0/1      |               |                  |                    |
|     R1      |   G0/1.100  | 192.168.1.1   | 255.255.255.192  |                    |
|     R1      |   G0/1.200  | 192.168.1.65  | 255.255.255.224  |                    |
|     R1      |   G0/1.1000 |               |                  |                    |
|     R2      |   G0/0      | 10.0.0.2      | 255.255.255.252  |                    |
|     R2      |   G0/1      | 192.168.1.97  | 255.255.255.240  |                    |
|     S1      |   VLAN 200  | 192.168.1.66  | 255.255.255.224  |  192.168.1.65      |
|     S2      |   VLAN 1    | 192.168.1.98  | 255.255.255.240  |  192.168.1.97      |
|     PC-A    |   NIC       | DHCP          | DHCP             |  DHCP              |
|     PC-B    |   NIC       | DHCP          | DHCP             |  DHCP              |

###  Таблица VLAN:

| VLAN  |  Имя          | Назначенный интерфейс        | 
|-------|---------------|------------------------------|
|  1    |  Нет          | S2: F0/18                    | 
|  100  |  Клиенты      | S1: gi0/0                    |   
|  200  |  Управление   | S1: VLAN 200                 |  
|  999  |  Parking_Lot  | S1: F0/1-4, F0/7-24, G0/1-2  |
|  1000 |  Собственная  |  ---                         |  




###  Задание:

  [Часть 1. Создание сети и настройка основных параметров устройства;](#часть-1-создание-сети-и-настройка-основных-параметров-устройства)

  [Часть 2. Настройка и проверка двух серверов DHCPv4 на R1;](#часть-2настройка-и-проверка-двух-серверов-dhcpv4-на-r1)

  [Часть 3. Настройка и проверка DHCP-ретрансляции на R2;](#часть-3настройка-и-проверка-dhcp-ретрансляции-на-r2)


###  Решение:
###  Часть 1. Создание сети и настройка основных параметров устройства

**Шаг 1.	Создание схемы адресации**

Вся информация ниже занесена в таблицу

a.	Одна подсеть «Подсеть A», поддерживающая 58 хостов (клиентская VLAN на R1)

R1 G0/1.100 (1-ый адрес) - 192.168.1.1

b.	Одна подсеть «Подсеть B», поддерживающая 28 хостов (управляющая VLAN на R1)

R1 G0/1.200 (1-ый адрес) - 192.168.1.65

S1 VLAN 200 (2-й адрес) - 192.168.1.66

c.	Одна подсеть «Подсеть C», поддерживающая 12 узлов (клиентская сеть на R2)

R2 G0/1 (1-ый адрес) - 192.168.1.97

**Шаг 2.	Настройка базовых параметров маршрутизатора и коммутатора**

Команды даны для настройки S1 свича.

Для настройки R1,R2 и S2 используется:

hostname R1

hostname R2

hostname S2


```
conf t
no ip domain-lookup
hostname S1
service password-encryption
enable secret class
banner motd #
Unauthorized access is strictly prohibited. #

line console 0
password cisco
login

logging synchronous

line vty 0 4
password cisco
login
```

*Подробнее базовая настройка рассмотрена в LAB01.*


**Шаг 3. Настройка маршрутизации между сетями VLAN на R1**

```
int G0/1.100
description Client
encapsulation dot1q 100
ip add 192.168.1.1 255.255.255.192

int G0/1.200
description Management
encapsulation dot1q 200
ip add 192.168.1.65 255.255.255.224

int G0/1.1000
description Native
encapsulation dot1q 1000 native

int G0/1
no shut
```


**Шаг 4.	Настройте G0/1 на R2, затем G0/0/0 и статическую маршрутизацию для обоих маршрутизаторов**

```
#R1:
interface gigabitEthernet 0/0
ip address 10.0.0.1 255.255.255.252
no shut
!
ip route 0.0.0.0 0.0.0.0 10.0.0.2

#R2:
interface gigabitEthernet 0/0
ip address 10.0.0.2 255.255.255.252
no shut
!
interface gigabitEthernet 0/1
ip address 192.168.1.97 255.255.255.240
no shut
!
ip route 0.0.0.0 0.0.0.0 10.0.0.1


R2(config)#do ping 192.168.1.65
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.65, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/4 ms
R2(config)#
```

**Шаг 5.	Создайте сети VLAN на коммутаторе S1.**
a.	Создайте необходимые VLAN 
b.	Настройте и активируйте интерфейс управления на S1 (VLAN 200)
c.	Настройте и активируйте интерфейс управления на S2 (VLAN 1)
d.	Назначьте все неиспользуемые порты S1 VLAN Parking_Lot


```
S1:
vlan 100
name Clinet

vlan 200
name Management

vlan 999
name Parking_Lot

vlan 1000
name Native

int vlan 200
ip add 192.168.1.66 255.255.255.252
no shut

ip default-gateway 192.168.1.65

# неиспользуемые интерфейсы
int gi0/2
switchport mode access
switchport access vlan 999
shut

# port to PC-A
int gi0/0
switchport mode access
switchport access vlan 100
no shut

```

```
S2:
vlan 999
name Parking_Lot

vlan 1000
name Native


int vlan 1
ip add 192.168.1.98 255.255.255.240
no shut

ip default-gateway 192.168.1.97

# неиспользуемые интерфейсы
int gi0/2
switchport mode access
switchport access vlan 999
shut

# port to PC-B
int gi0/0
switchport mode access
no shut

```

b.	Убедитесь, что VLAN назначены на правильные интерфейсы.

```
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gi0/1
100  Clinet                           active    Gi0/0
200  Management                       active
999  Parking_Lot                      active    Gi0/2

```

Почему интерфейс gi0/1 указан в VLAN 1?
> Gi0/1 не был настроен как trunk порт, поэтому на данный момент он находится в VLAN1.


**Шаг 6.	Вручную настройте интерфейс S1 gi0/1 в качестве транка 802.1Q.**

```
int gi0/1
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allow vlan 100,200,1000
switchport trunk native vlan 1000
switchport nonegotiate
no shut


S1(config)#do show int trunk

Port        Mode             Encapsulation  Status        Native vlan
Gi0/1       on               802.1q         trunking      1000

Port        Vlans allowed on trunk
Gi0/1       100,200,1000

Port        Vlans allowed and active in management domain
Gi0/1       100,200,1000

Port        Vlans in spanning tree forwarding state and not pruned
Gi0/1       100,200,1000

```

Какой IP-адрес был бы у ПК, если бы он был подключен к сети с помощью DHCP?
> пока DHCP сервер не настроен, он получит адрес из сети 169.254.0.0/16



### Часть 2.	Настройка и проверка двух серверов DHCPv4 на R1

Настройка DHCPv4 на R1. 
Сервер DHCPv4 будет обслуживать две подсети, подсеть A (192.168.1.0/26) и подсеть C (192.168.1.96/28).

```
R1:

ip dhcp excluded-address 192.168.1.1 192.168.1.5
ip dhcp pool R1_Client_LAN 
	network 192.168.1.0 255.255.255.192
	default-router 192.168.1.1
	domain-name CCNA-lab.com
	lease 2 12 30

ip dhcp excluded-address 192.168.1.97 192.168.1.101
ip dhcp pool R2_Client_LAN 
	network 192.168.1.96 255.255.255.240
	default-router 192.168.1.97
	domain-name CCNA-lab.com
	lease 2 12 30
```

```
R1(config)#do show ip dhcp pool

Pool R2_Client_LAN :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0
 Total addresses                : 14
 Leased addresses               : 0
 Pending event                  : none
 1 subnet is currently in the pool :
 Current index        IP address range                    Leased addresses
 192.168.1.97         192.168.1.97     - 192.168.1.110     0

Pool R1_Client_LAN :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0
 Total addresses                : 62
 Leased addresses               : 0
 Pending event                  : none
 1 subnet is currently in the pool :
 Current index        IP address range                    Leased addresses
 192.168.1.1          192.168.1.1      - 192.168.1.62      0

```

```
R1(config)#do show ip dhcp binding
Bindings from all pools not associated with VRF:
IP address          Client-ID/              Lease expiration        Type
                    Hardware address/
                    User name
R1(config)#

```


```
R1(config)#do show ip dhcp server statistics
Memory usage         25175
Address pools        2
Database agents      0
Automatic bindings   0
Manual bindings      0
Expired bindings     0
Malformed messages   0
Secure arp entries   0

Message              Received
BOOTREQUEST          0
DHCPDISCOVER         0
DHCPREQUEST          0
DHCPDECLINE          0
DHCPRELEASE          0
DHCPINFORM           0

Message              Sent
BOOTREPLY            0
DHCPOFFER            0
DHCPACK              0
DHCPNAK              0
```


**Настройка DHCP client на PC-A (router)**
PC-A(config)# int gi0/0
PC-A(config-if)# ip address dhcp
PC-A(config-if)# no shut

PC-A#show ip interface brief
Interface                  IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0         192.168.1.7     YES DHCP   up                    up

Проверка R1:

```
R1#show ip dhcp binding
Bindings from all pools not associated with VRF:
IP address          Client-ID/              Lease expiration        Type
                    Hardware address/
                    User name
192.168.1.7         0063.6973.636f.2d35.    Oct 31 2026 04:12 AM    Automatic
                    3030.302e.3030.3038.
                    2e30.3030.302d.4769.
                    302f.30



R1#show ip dhcp server statistics
Memory usage         41846
Address pools        2
Database agents      0
Automatic bindings   1
Manual bindings      0
Expired bindings     1
Malformed messages   0
Secure arp entries   0

Message              Received
BOOTREQUEST          0
DHCPDISCOVER         263
DHCPREQUEST          1
DHCPDECLINE          0
DHCPRELEASE          0
DHCPINFORM           0

Message              Sent
BOOTREPLY            0
DHCPOFFER            198
DHCPACK              1
DHCPNAK              0
```

### Часть 3.	Настройка и проверка DHCP-ретрансляции на R2

R2:
```
R2(config)#interface g0/1
R2(config-if)#  ip helper-address 10.0.0.1
```

PC-B:
```
PC-B(config)# int gi0/0
PC-B(config-if)# ip address dhcp
PC-B(config-if)# no shut

PC-B#show ip interface brief
Interface                  IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0         192.168.1.104   YES DHCP   up                    up
```

R1:
```
R1#show ip dhcp binding
Bindings from all pools not associated with VRF:
IP address          Client-ID/              Lease expiration        Type
                    Hardware address/
                    User name
192.168.1.7         0063.6973.636f.2d35.    Oct 31 2026 04:12 AM    Automatic
                    3030.302e.3030.3038.
                    2e30.3030.302d.4769.
                    302f.30
192.168.1.104       0063.6973.636f.2d35.    Oct 31 2026 05:21 AM    Automatic
                    3030.302e.3030.3063.
                    2e30.3030.302d.4769.
                    302f.30
R1#


R1#show ip dhcp server statistics
Memory usage         66709
Address pools        2
Database agents      0
Automatic bindings   2
Manual bindings      0
Expired bindings     3
Malformed messages   0
Secure arp entries   0

Message              Received
BOOTREQUEST          0
DHCPDISCOVER         606
DHCPREQUEST          3
DHCPDECLINE          0
DHCPRELEASE          0
DHCPINFORM           0

Message              Sent
BOOTREPLY            0
DHCPOFFER            481
DHCPACK              3
DHCPNAK              0
R1#
```