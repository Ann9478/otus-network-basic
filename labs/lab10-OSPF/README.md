# Настройка протокола OSPFv2 для одной области.

###  Топология:


###  Таблица адресации:

| Устройство  | Интерфейс/VLAN | IP-адрес    | Маска подсети    | 
|-------------|----------------|-------------|------------------|
|     R1      |   G0/0         | 10.53.0.1   |   255.255.255.0  |
|     R1      |   Loopback 1   | 172.16.1.1  |   255.255.255.0  |
|     R2      |   G0/0         | 10.53.0.2   |   255.255.255.0  |
|     R2      |   Loopback 1   | 192.168.1.1 |   255.255.255.0  |




###  Задание:

  [Часть 1. Создание сети и настройка основных параметров устройства;](#часть-1-создание-сети-и-настройка-основных-параметров-устройства)

  [Часть 2. Часть 2. Настройка и проверка базовой работы протокола OSPFv2 для одной области;](#часть-2-настройка-и-проверка-базовой-работы-протокола-ospfv2-для-одной-области)

  [Часть 3. Часть 3. Оптимизация и проверка конфигурации OSPFv2 для одной области;](#часть-3-оптимизация-и-проверка-конфигурации-ospfv2-для-одной-области)



###  Решение:
###  Часть 1. Создание сети и настройка основных параметров устройства

**Шаг 1.	Настройка базовых параметров маршрутизатора и коммутатора**

Команды даны для настройки S1 свича.

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



### Часть 2. Настройка и проверка базовой работы протокола OSPFv2 для одной области

**Шаг 1. Настройте адреса интерфейса и базового OSPFv2 на каждом маршрутизаторе.**

R1:

```
interface gi0/0
ip address 10.53.0.1 255.255.255.0
no shut

interface loopback1
ip address 172.16.1.1 255.255.255.0
no shut
```

R2:

```
interface gi0/0
ip address 10.53.0.2 255.255.255.0
no shut

interface loopback1
ip address 192.168.1.1 255.255.255.0
no shut
```


**Шаг 2. Настройте OSPF на R1 и R2. Только на R2 добавьте конфигурацию, необходимую для объявления сети Loopback 1**

R1:

```
router ospf 65
router-id 1.1.1.1
network 10.53.0.0 0.0.0.255 area 0 (1 способ)
```

R2:

```
router ospf 65
router-id 2.2.2.2
network 10.53.0.2 0.0.0.0 area 0   (2 способ)

interface lo1
ip ospf 65 area 0                  (3 способ)
```

Проверка смежности.

R1 - BDR

R2 - DR

Критерий отбора - router id, у R2 он больше

```
R1#show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           1   FULL/DR         00:00:36    10.53.0.2       GigabitEtherne

R2#show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           1   FULL/BDR        00:00:36    10.53.0.1       GigabitEtherne 
```


**Шаг 3. Проверка доступности сети R2 Loopback1 с R1**

```
R1#show ip route ospf
Gateway of last resort is not set

      192.168.1.0/32 is subnetted, 1 subnets
O        192.168.1.1 [110/2] via 10.53.0.2, 00:07:09, GigabitEthernet0/0
R1#

R1#ping 192.168.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 4/5/6 ms
```


### Часть 3. Оптимизация и проверка конфигурации OSPFv2 для одной области

**Шаг 1. Реализация различных оптимизаций на каждом маршрутизаторе.**

a.	На R1 настройте приоритет OSPF интерфейса G0/0 на 50, чтобы убедиться, что R1 является назначенным маршрутизатором.
b.	Настройте таймеры OSPF на G0/0 каждого маршрутизатора для таймера приветствия, составляющего 30 секунд.

R1:

```
interface gi0/0
ip ospf priority 50
ip ospf hello-interval 30
```

R2:

```
interface gi0/0
ip ospf hello-interval 30
```

c.	На R1 настройте статический маршрут по умолчанию, который использует интерфейс Loopback 1 в качестве интерфейса выхода. Затем распространите маршрут по умолчанию в OSPF. Обратите внимание на сообщение консоли после установки маршрута по умолчанию.

R1:

```
ip route 0.0.0.0 0.0.0.0 loopback1

router ospf 65
default-information originate
```

d.	добавьте конфигурацию, необходимую для OSPF для обработки R2 Loopback 1 как сети точка-точка. Это приводит к тому, что OSPF объявляет Loopback 1 использует маску подсети интерфейса.
e.	Только на R2 добавьте конфигурацию, необходимую для предотвращения отправки объявлений OSPF в сеть Loopback 1.


R2:

```
interface loopback1
ip ospf network point-to-point

router ospf 65
passive-interface loopback1
```

f.	Измените базовую пропускную способность для маршрутизаторов. После этой настройки перезапустите OSPF с помощью команды clear ip ospf process . Обратите внимание на сообщение консоли после установки новой опорной полосы пропускания.

R1/R2:

```
router ospf 65
auto-cost reference-bandwidth 10000
clear ip ospf process 
```

**Шаг 2. Убедитесь, что оптимизация OSPFv2 реализовалась.**
a.	Выполните команду show ip ospf interface g0/0 на R1 и убедитесь, что приоритет интерфейса установлен равным 50, а временные интервалы — Hello 30, Dead 120, а тип сети по умолчанию — Broadcast

```
R1#show ip ospf interface g0/0
GigabitEthernet0/0 is up, line protocol is up
  Internet Address 10.53.0.1/24, Area 0, Attached via Network Statement
  Process ID 65, Router ID 1.1.1.1, Network Type BROADCAST, Cost: 10
  Topology-MTID    Cost    Disabled    Shutdown      Topology Name
        0           10        no          no            Base
  Transmit Delay is 1 sec, State DR, Priority 50
  Designated Router (ID) 1.1.1.1, Interface address 10.53.0.1
  Backup Designated router (ID) 2.2.2.2, Interface address 10.53.0.2
  Timer intervals configured, **Hello 30, Dead 120**, Wait 120, Retransmit 5
    oob-resync timeout 120
    Hello due in 00:00:05
  Supports Link-local Signaling (LLS)
```

b.	На R1 выполните команду show ip route ospf, чтобы убедиться, что сеть R2 Loopback1 присутствует в таблице маршрутизации. Обратите внимание на разницу в метрике между этим выходным и предыдущим выходным. Также обратите внимание, что маска теперь составляет 24 бита, в отличие от 32 битов, ранее объявленных.

```
R1#show ip route ospf
Gateway of last resort is not set

O     192.168.1.0/24 [110/11] via 10.53.0.2, 00:01:58, GigabitEthernet0/0
```

c.	Введите команду show ip route ospf на маршрутизаторе R2. Единственная информация о маршруте OSPF должна быть распространяемый по умолчанию маршрут R1.

```
R2#sh ip route ospf
Gateway of last resort is 10.53.0.1 to network 0.0.0.0

O*E2  0.0.0.0/0 [110/1] via 10.53.0.1, 00:07:04, GigabitEthernet0/0
```

d.	Запустите Ping до адреса интерфейса R1 Loopback 1 из R2. Выполнение команды ping должно быть успешным.

```
R2#ping 172.16.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 5/5/5 ms
```

Почему стоимость OSPF для маршрута по умолчанию отличается от стоимости OSPF в R1 для сети 192.168.1.0/24?

> Стоимости разные, потому что это маршруты разных типов и метрика у них считается по-разному.
> У маршрута по-умолчанию всегда - 1
> У 192.168.1.0 метрика 11, потому что сегмент от R1 до R2 стоит 10 + до lo - 1



