# Настройка IPv6-адресов на сетевых устройствах.

###  Топология:
![](./topology.png)

###  Таблица адресации:

| Устройство  | Интерфейс | IP-адрес / префикс       | Link local IPv6-адрес | Длина префикса | Шлюз по умолчанию |
|-------------|-----------|--------------------------|-----------------------|----------------|-------------------|
|     R1      |   G0/0/0  | 2001:db8:acad:a::1       |   fe80::1             |   64           |        ---        |
|     R1      |   G0/0/1  | 2001:db8:acad:1::1       |   fe80::1             |   64           |        ---        |
|     S1      |   VLAN 1  | 2001:db8:acad:1::b       |   fe80::b             |   64           |        ---        |
|    PC-A     |   NIC     | 2001:db8:acad:1::3       |   SLACC               |   64           |       fe80::1     |
|    PC-B     |   NIC     | 2001:db8:acad:a::3       |   SLACC               |   64           |       fe80::1     |

###  Задание:

  [Часть 1. Настройка топологии и конфигурация основных параметров маршрутизатора и коммутатора;](#часть-1-настройка-топологии-и-конфигурация-основных-параметров-маршрутизатора-и-коммутатора)

  [Часть 2. Ручная настройка IPv6-адресов;](#часть-2-ручная-настройка-ipv6-адресов)

  [Часть 3. Проверка сквозного соединения;](#часть-3-проверка-сквозного-подключения)

  [Вопросы для повторения.](#вопросы-для-повторения)

###  Решение:
###  Часть 1. Настройка топологии и конфигурация основных параметров маршрутизатора и коммутатора
**Настройка базовых параметры коммутатора и маршрутизатора.**
Команды даны для настройки S1 свича.

Для настройки R1 используется:

hostname R1

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

*Подробнее базовая настройка свичей рассмотрена в LAB01.*




### Часть 2. Ручная настройка IPv6-адресов
**Шаг 1. Назначьте IPv6-адреса интерфейсам Ethernet на R1.**

```
conf t
interface range gigabitEthernet 0/0-1
ipv6 enable
ipv6 address fe80::1 link-local
no shut


interface gigabitEthernet 0/0
ipv6 address 2001:db8:acad:a::1/64


interface gigabitEthernet 0/1
ipv6 address 2001:db8:acad:1::1/64


R1(config-if-range)#do show ipv6 int br
GigabitEthernet0/0         [up/up]
    FE80::1
    2001:DB8:ACAD:A::1
GigabitEthernet0/1         [up/up]
    FE80::1
    2001:DB8:ACAD:1::1
```

1. Какие группы многоадресной рассылки назначены интерфейсу G0/0? 

>    FF02::1 - все IPv6-узлы
>    FF02::2 - все IPv6-маршрутизаторы

```
R1(config)#do show ipv6 interface gigabitEthernet 0/0
GigabitEthernet0/0 is up, line protocol is up
  IPv6 is enabled, link-local address is FE80::1
  No Virtual link-local address(es):
  Global unicast address(es):
    2001:DB8:ACAD:A::1, subnet is 2001:DB8:ACAD:A::/64
  Joined group address(es):
    FF02::1
    FF02::2
    FF02::1:FF00:1
```

**Шаг 2. Активируйте IPv6-маршрутизацию на R1.**

```
R1(config)#IPv6 unicast-routing
```

2. Почему PC-B получил глобальный префикс маршрутизации и идентификатор подсети, которые вы настроили на R1?

>    R1 отправляет объявления маршрутизатора Router Advertisement (RA) через IPv6 multicast, после активации IPv6-маршрутизации на R1.


**Шаг 3. Назначьте IPv6-адреса интерфейсу управления (SVI) на S1.**

```
conf t
sdm prefer dual-ipv4-and-ipv6 default
wr
reload

interface vlan 1
ipv6 address 2001:db8:acad:1::b/64
ipv6 address fe80::b link-local
no shut
```

```
S1(config-if)#do show ipv6 interface vlan 1
Vlan1 is up, line protocol is up
  IPv6 is enabled, link-local address is FE80::B
  No Virtual link-local address(es):
  Global unicast address(es):
    2001:DB8:ACAD:1::B, subnet is 2001:DB8:ACAD:1::/64
  Joined group address(es):
    FF02::1
    FF02::1:FF00:B
```


**Шаг 4. Назначьте компьютерам статические IPv6-адреса.**

**PC-A:**

```
C:\>ipconfig

FastEthernet0 Connection:(default port)

   Connection-specific DNS Suffix..: 
   Link-local IPv6 Address.........: FE80::2E0:A3FF:FE56:4935
   IPv6 Address....................: 2001:DB8:ACAD:1::3
   IPv4 Address....................: 0.0.0.0
   Subnet Mask.....................: 0.0.0.0
   Default Gateway.................: FE80::1
```

**PC-B:**

```
C:\>ipconfig

FastEthernet0 Connection:(default port)

   Connection-specific DNS Suffix..: 
   Link-local IPv6 Address.........: FE80::290:CFF:FE67:199A
   IPv6 Address....................: 2001:DB8:ACAD:A::3
   IPv4 Address....................: 0.0.0.0
   Subnet Mask.....................: 0.0.0.0
   Default Gateway.................: FE80::1
```

### Часть 3. Проверка сквозного подключения

**PC-A:**

```
C:\>ping fe80::1

Pinging fe80::1 with 32 bytes of data:

Reply from FE80::1: bytes=32 time<1ms TTL=255
Reply from FE80::1: bytes=32 time=1ms TTL=255
Reply from FE80::1: bytes=32 time<1ms TTL=255
Reply from FE80::1: bytes=32 time<1ms TTL=255

Ping statistics for FE80::1:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 1ms, Average = 0ms



C:\>ping 2001:db8:acad:1::b

Pinging 2001:db8:acad:1::b with 32 bytes of data:

Reply from 2001:DB8:ACAD:1::B: bytes=32 time<1ms TTL=255
Reply from 2001:DB8:ACAD:1::B: bytes=32 time<1ms TTL=255
Reply from 2001:DB8:ACAD:1::B: bytes=32 time<1ms TTL=255
Reply from 2001:DB8:ACAD:1::B: bytes=32 time<1ms TTL=255

Ping statistics for 2001:DB8:ACAD:1::B:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms



C:\>tracert 2001:db8:acad:a::3

Tracing route to 2001:db8:acad:a::3 over a maximum of 30 hops: 

  1   0 ms      0 ms      0 ms      2001:DB8:ACAD:1::1
  2   11 ms     0 ms      0 ms      2001:DB8:ACAD:A::3

Trace complete.
C:\>
```

**PC-B:**

```
C:\>ping 2001:db8:acad:1::3

Pinging 2001:db8:acad:1::3 with 32 bytes of data:

Reply from 2001:DB8:ACAD:1::3: bytes=32 time<1ms TTL=127
Reply from 2001:DB8:ACAD:1::3: bytes=32 time<1ms TTL=127
Reply from 2001:DB8:ACAD:1::3: bytes=32 time<1ms TTL=127
Reply from 2001:DB8:ACAD:1::3: bytes=32 time=13ms TTL=127

Ping statistics for 2001:DB8:ACAD:1::3:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 13ms, Average = 3ms



C:\>ping fe80::1

Pinging fe80::1 with 32 bytes of data:

Reply from FE80::1: bytes=32 time<1ms TTL=255
Reply from FE80::1: bytes=32 time<1ms TTL=255
Reply from FE80::1: bytes=32 time=2ms TTL=255
Reply from FE80::1: bytes=32 time<1ms TTL=255

Ping statistics for FE80::1:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 2ms, Average = 0ms
```



### Вопросы для повторения
1.	Почему обоим интерфейсам Ethernet на R1 можно назначить один и тот же локальный адрес канала — FE80::1?

Потому что link-local IPv6-адрес действует только внутри конкретного локального сегмента и не маршрутизируется за его пределами. G0/0/0 и G0/0/1 находятся в разных IPv6-сетях, поэтому fe80::1 на обоих интерфейсах не создаёт конфликта.

2.	Какой идентификатор подсети в индивидуальном IPv6-адресе 2001:db8:acad::aaaa:1234/64?

2001:db8:acad::aaaa:1234/64

Полный адрес:

2001:0db8:acad:0000:0000:0000:aaaa:1234

/64 - 2001:0db8:acad:**0000**

Индентификатор подсети - 0
