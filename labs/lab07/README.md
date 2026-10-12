# Развертывание коммутируемой сети с резервными каналами.

###  Топология:
![](./topology.png)

###  Таблица адресации:

| Устройство  | Интерфейс     | IP-адрес       | Маска подсети    | 
|-------------|---------------|----------------|------------------|
|     S1      |   VLAN 1      | 192.168.1.1   |   255.255.255.0   |
|     S2      |   VLAN 1      | 192.168.1.2   |   255.255.255.0   |
|     S3      |   VLAN 1      | 192.168.1.3   |   255.255.255.0   |



###  Задание:

  [Часть 1. Создание сети и настройка основных параметров устройства;](#часть-1-создание-сети-и-настройка-основных-параметров-устройства)

  [Часть 2. Выбор корневого моста;](#часть-2-выбор-корневого-моста)

  [Часть 3. Наблюдение за процессом выбора протоколом STP порта, исходя из стоимости портов;](#часть-3наблюдение-за-процессом-выбора-протоколом-stp-порта-исходя-из-стоимости-портов)

  [Часть 4. Наблюдение за процессом выбора протоколом STP порта, исходя из приоритета портов;](#часть-4наблюдение-за-процессом-выбора-протоколом-stp-порта-исходя-из-приоритета-портов)

  [Вопросы для повторения;](#вопросы-для-повторения)


###  Решение:
###  Часть 1. Создание сети и настройка основных параметров устройства
**Настройка базовых параметров маршрутизатора и коммутатора .**
Команды даны для настройки S1 свича.

Для настройки S2 и S3 используется:

hostname S2

hostname S3

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

Настройка ip add на интерфейсе VLAN1 на маршрутизаторе S1:
Команды даны для настройки S1 свича.

Для настройки S2 и S3 используются ip адреса из таблицы.

```
interface vlan 1
ip address 192.168.1.1 255.255.255.0
no shut
```

Результатом настройки является успешное выполнение эхо-запросов между всеми 3-мя коммутаторами.



### Часть 2. Выбор корневого моста

На всех коммутаторах необходимо выполнить:

```
# Шаг 1:	Отключите все порты на коммутаторах.
S1(config)#interface range Fa0/1-24
S1(config-if-range)#shut
S1(config-if-range)#interface range Gi0/1-2
S1(config-if-range)#shut

# Шаг 2:	Настройте подключенные порты в качестве транковых.
S1(config)#interface range Fa0/1-4
S1(config-if-range)#switchport mode trunk 

# Шаг 3:	Включите порты F0/2 и F0/4 на всех коммутаторах.
S1(config)#interface range Fa0/2, fa0/4
S1(config-if-range)#no shut

```

```
# Шаг 4:	Отобразите данные протокола spanning-tree.

S1#show spanning-tree 
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     0001.420A.1452
             Cost        19
             Port        2(FastEthernet0/2)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     00E0.F94D.AB6B
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/4            Altn BLK 19        128.4    P2p
Fa0/2            Root FWD 19        128.2    P2p

S2#show spanning-tree 
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     0001.420A.1452
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     0001.420A.1452
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/2            Desg FWD 19        128.2    P2p
Fa0/4            Desg FWD 19        128.4    P2p


S3#show spanning-tree 
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     0001.420A.1452
             Cost        19
             Port        2(FastEthernet0/2)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     00D0.FF79.7745
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/2            Root FWD 19        128.2    P2p
Fa0/4            Desg FWD 19        128.4    P2p

```

**Ответы на вопросы:**
1. Какой коммутатор является корневым мостом?
> *S2*

2. Почему этот коммутатор был выбран протоколом spanning-tree в качестве корневого моста?
> *Приоритет идентификатора моста рассчитывается путем сложения значений приоритета и расширенного идентификатора системы. Расширенным идентификатором системы всегда является номер сети VLAN. В примере ниже все три коммутатора имеют равные значения приоритета идентификатора моста (32769 = 32768 + 1, где приоритет по умолчанию = 32768, номер сети VLAN = 1); следовательно, коммутатор с самым низким значением MAC-адреса становится корневым мостом. В нашем случае - это S2.*

3. Какие порты на коммутаторе являются корневыми портами? 
> *Корневой порт - это порт, который имеет минимальную стоимость до любого порта корневого коммутатора.*
> *S1 - Fa0/2, S3 - Fa0/2*

4. Какие порты на коммутаторе являются назначенными портами? 
> *Назначенный порт - это порт, который имеет кратчайшее расстояние от назначенного коммутатора до корневого коммутатора.*
> *S3 - Fa0/4*

5. Какой порт отображается в качестве альтернативного и в настоящее время заблокирован? 
> *Альтернативный порт -  это резервный путь к корневому коммутатору, который находится в заблокированном состоянии*
> *S1 - Fa0/4*

6. Почему протокол spanning-tree выбрал этот порт в качестве невыделенного (заблокированного) порта?
> *На сегменте S1_fa0/4 - S3_fa0/4 свич S3 имеет наименьший приоритет, поэтому порты S3_fa0/4 выбирается как desg, соответственно другой порт переходит в состояние заблокированного порта.*



### Часть 3:	Наблюдение за процессом выбора протоколом STP порта, исходя из стоимости портов

**Шаг 1:	Определите коммутатор с заблокированным портом**

```
S1#show spanning-tree 
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     0001.420A.1452
             Cost        19
             Port        2(FastEthernet0/2)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     00E0.F94D.AB6B
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/4            Altn BLK 19        128.4    P2p
Fa0/2            Root FWD 19        128.2    P2p


S3#show spanning-tree 
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     0001.420A.1452
             Cost        19
             Port        2(FastEthernet0/2)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     00D0.FF79.7745
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/2            Root FWD 19        128.2    P2p
Fa0/4            Desg FWD 19        128.4    P2p

```

**Шаг 2:	Измените стоимость порта.**
Помимо заблокированного порта, единственным активным портом на этом коммутаторе является порт, выделенный в качестве порта корневого моста. Уменьшите стоимость этого порта корневого моста до 18, выполнив команду spanning-tree vlan 1 cost 18 режима конфигурации интерфейса.


```
S1(config)# interface f0/2
S1(config-if)# spanning-tree vlan 1 cost 18
```

**Шаг 3:	Просмотрите изменения протокола spanning-tree.**



```
S1#show spanning-tree 
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     0001.420A.1452
             Cost        18
             Port        2(FastEthernet0/2)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     00E0.F94D.AB6B
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/4            Desg LRN 19        128.4    P2p
Fa0/2            Root FWD 18        128.2    P2p


S3#show spanning-tree 
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     0001.420A.1452
             Cost        19
             Port        2(FastEthernet0/2)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     00D0.FF79.7745
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/2            Root FWD 19        128.2    P2p
Fa0/4            Altn BLK 19        128.4    P2p
```

Почему протокол spanning-tree заменяет ранее заблокированный порт на назначенный порт и блокирует порт, который был назначенным портом на другом коммутаторе?
> Т.к. мы понизили стоимость порта. От S1 до корневого коммутатора равна 18, а у S3 остаётся 19. Поэтому теперь на сегменте S1-S3 на свиче S1 будет выбран desg порт.

**Шаг 4:	Удалите изменения стоимости порта.**

```
S1(config)# interface f0/2
S1(config-if)# no spanning-tree vlan 1 cost 18
```

Порт S1_fa0/4 снова стал ALTN.


### Часть 4:	Наблюдение за процессом выбора протоколом STP порта, исходя из приоритета портов

```
S1-3(config)#interface range Fa0/1, fa0/3
S1-3(config-if-range)#no shut


S1#show spanning-tree 
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     0001.420A.1452
             Cost        19
             Port        1(FastEthernet0/1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     00E0.F94D.AB6B
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/4            Altn BLK 19        128.4    P2p
Fa0/1            Root LRN 19        128.1    P2p
Fa0/2            Altn BLK 19        128.2    P2p
Fa0/3            Altn BLK 19        128.3    P2p


S2#show spanning-tree 
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     0001.420A.1452
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     0001.420A.1452
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/3            Desg FWD 19        128.3    P2p
Fa0/2            Desg FWD 19        128.2    P2p
Fa0/1            Desg FWD 19        128.1    P2p
Fa0/4            Desg FWD 19        128.4    P2p


S3#show spanning-tree 
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     0001.420A.1452
             Cost        19
             Port        1(FastEthernet0/1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     00D0.FF79.7745
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/1            Root FWD 19        128.1    P2p
Fa0/2            Altn BLK 19        128.2    P2p
Fa0/3            Desg FWD 19        128.3    P2p
Fa0/4            Desg FWD 19        128.4    P2p

```

1. Какой порт выбран протоколом STP в качестве порта корневого моста на каждом коммутаторе некорневого моста?
> *Выбран порт fa0/1*

2. Почему протокол STP выбрал эти порты в качестве портов корневого моста на этих коммутаторах?
> *Т.к. данный порт имеет наименьший Port ID (Port Priority + Port Number). В нашем случае приоритет одинаковый, поэтому выбирается наименьший по номеру порт*


### 	Вопросы для повторения:
1.	Какое значение протокол STP использует первым после выбора корневого моста, чтобы определить выбор порта?
> Path Cost - наименьшую суммарную стоимость пути до корневого моста

2.	Если первое значение на двух портах одинаково, какое следующее значение будет использовать протокол STP при выборе порта?
> Коммутатор с наименьшим Bridge ID. 

3.	Если оба значения на двух портах равны, каким будет следующее значение, которое использует протокол STP при выборе порта?
> Порт с наименьшим Port ID