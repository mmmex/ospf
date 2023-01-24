## OSPF

Задачи:

- [X] Поднять три виртуалки
- [X] Объединить их разными vlan
- [X] поднять OSPF между машинами на базе Quagga или FRR или BIRD;
- [X] изобразить асимметричный роутинг
- [X] сделать один из линков "дорогим", но что бы при этом роутинг был симметричным.
- [X] Формат сдачи: Vagrantfile + ansible



### Запуск проекта

1. Клонируем репозиторий: `git clone https://github.com/mmmex/ospf.git`
2. Переходим в каталог: `cd ospf`
3. Переходим на ветку `vlan`: `git switch vlan`
4. Запускаем проект: `vagrant up`

Будут подняты 3 ВМ: `router1`, `router2` и `router3` с установленным FRR версии 8.4.2

**Топология сети:**

![image](https://raw.githubusercontent.com/mmmex/ospf/vlan/OSPF_with_vlan-diagram.png)

**Таблицы хостов:**

**Router1:**
| Интерфейс   | Адрес           | Сосед   |
|:-----------:|:---------------:|:-------:|
| eth1        | **<TRUNK>**     | -       |
| vlan10@eth1 | 10.0.12.1/30    | Router3 |
| vlan11@eth1 | 10.0.10.1/30    | Router2 |
| eth2        | 192.168.10.1/24 | -       |

**Router2:**
| Интерфейс   | Адрес           | Сосед   |
|:-----------:|:---------------:|:-------:|
| eth1        | **<TRUNK>**     | -       |
| vlan11@eth1 | 10.0.10.2/30    | Router1 |
| vlan12@eth2 | 10.0.11.2/30    | Router3 |
| eth2        | 192.168.20.1/24 | -       |

**Router3:**
| Интерфейс   | Адрес           | Сосед   |
|:-----------:|:---------------:|:-------:|
| eth1        | **<TRUNK>**     | -       |
| vlan12@eth1 | 10.0.11.1/30    | Router2 |
| vlan10@eth2 | 10.0.12.2/30    | Router1 |
| eth2        | 192.168.30.1/24 | -       |

При данной конфигурации маршрутизация осуществляется симметрично.

### Асимметричная маршрутизация

*Что такое асимметричная маршрутизация? При асимметричной маршрутизации пакет проходит от источника к получателю по одному пути и выбирает другой путь, когда возвращается к источнику.*

В основе такой маршрутизации лежит разделение маршрутов исходящего и входящего трафика. Например, входящий трафик может приниматься от одного провайдера, а исходящий будет отправляться через другого. Разделение среды передачи данных приведено для примера и чаще используется единая среда. Использование асимметричной маршрутизации может негативно сказаться на передаче данных в сетях с VPN на пути следования данных. Использование такого типа маршрутизации целесообразно только в больших сетях, в которых больше одного шлюза. В ряде случаев асимметричную маршрутизацию используют с целью снижения нагрузки на канал передачи данных, путем перенаправления части трафика через другие каналы. Такой способ снижения нагрузки применим только в случае, если имеется два и более каналов передачи данных. Также, среди причин использования асимметричной маршрутизации можно рассмотреть коммерческую выгоду при использовании burstable-биллинга, когда организации выгодно платить только за часть трафика. [источник](https://ddos-guard.net/ru/terminology/technology/asimmetrichnaya-marshrutizaciya)

Для выполнения данного вида маршрутизации нужно определенным образом настроить ядро ОС, а также интерфейсы для протокола OSPF. В данной инструкции будет выполнена настройка ВМ `router1` и `router2`.

* В целях экономии времени подготовлен [playbook ansible](ansible/asymmetric-routing.yml), который выполнит необходимые действия автоматически.

#### Выполнение асимметричной маршрутизации вручную

**Можно пропустить этот блок инструкции и приступить к выполнению [следующего](#%D0%B2%D1%8B%D0%BF%D0%BE%D0%BB%D0%BD%D0%B5%D0%BD%D0%B8%D0%B5-%D0%B0%D1%81%D0%B8%D0%BC%D0%BC%D0%B5%D1%82%D1%80%D0%B8%D1%87%D0%BD%D0%BE%D0%B9-%D0%BC%D0%B0%D1%80%D1%88%D1%80%D1%83%D1%82%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D0%B8-%D0%B0%D0%B2%D1%82%D0%BE%D0%BC%D0%B0%D1%82%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%B8-%D1%81%D0%BA%D1%80%D0%B8%D0%BF%D1%82%D0%BE%D0%BC-ansible-playbook).**

Для конфигурации вручную необходимо выполнить следующие шаги:

1. Отключить в ядре ОС проверку адреса источника глобально и на интерфейсе `vlan11` ВМ `router1` (параметр `net.ipv4.conf.all.rp_filter` и `net.ipv4.conf.vlan11.rp_filter`):
   - `vagrant ssh router1` выполняем вход на `router1`;
   - `sudo -i` повышаем привелегию пользователя;
   - `sysctl -w net.ipv4.conf.all.rp_filter=0` отключаем глобально проверку;
   - `sysctl -w net.ipv4.conf.vlan11.rp_filter=0` отключаем проверку на конкретном интерфейсе;
   - если потребуется запомнить состояние конфигурации, то необходимо записать данные значения в файл `/etc/sysctl.conf`.
2. Увеличить стоимость (`cost`) соединения на интерфейсе `vlan11` маршрутизатора `router1`, к примеру до 1000:
   - `vtysh` входим в консоль управления `FRR`;
   - `conf t` входим в режим конфигурирования;
   - `int vlan11` переходим в раздел конфигурации интерфейса `vlan11`;
   - `ip ospf cost 1000` устанавливаем новое значение `cost` в 1000;
   - `exit` выход из режима конфигурирования интерфейса;
   - `exit` выход из терминала режима конфигурирования;
   - `exit` выход из vtysh;
   - `ping -I 192.168.10.1 192.168.20.1` запускаем ping с интерфейса eth2 (ключ -I) `192.168.10.1` на узел `192.168.20.1`.

```bash
test@test-virtual-machine:~/Otus/ospf$ vagrant ssh router1
Last login: Tue Jan 24 15:53:23 2023 from 10.0.2.2
[vagrant@router1 ~]$ sudo -i
[root@router1 ~]# sysctl -w net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.all.rp_filter = 0
[root@router1 ~]# sysctl -w net.ipv4.conf.vlan11.rp_filter=0
net.ipv4.conf.vlan11.rp_filter = 0
[root@router1 ~]# vtysh

Hello, this is FRRouting (version 8.4.2).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# conf t
router1(config)# int vlan11
router1(config-if)# ip ospf cost 1000
router1(config-if)# exit
router1(config)# exit
router1# exit
[root@router1 ~]# ping -I 192.168.10.1 192.168.20.1
PING 192.168.20.1 (192.168.20.1) from 192.168.10.1 : 56(84) bytes of data.
```

Следующие действия осуществляем в новом окне терминала, это окно оставляем открытым.

3. Отключить в ядре ОС проверку адреса источника глобально и на интерфейсе `vlan12` ВМ `router2`:
   - `vagrant ssh router2` выполняем вход на `router2`;
   - `sudo -i` повышаем привелегию пользователя;
   - `sysctl -w net.ipv4.conf.all.rp_filter=0` отключаем глобально проверку;
   - `sysctl -w net.ipv4.conf.vlan12.rp_filter=0` отключаем проверку на конкретном интерфейсе. (после ввода этой команды увидим ответы от узла на предыдущем шаге)

```bash
test@test-virtual-machine:~/Otus/ospf$ vagrant ssh router2
Last login: Mon Jan 23 11:54:21 2023 from 10.0.2.2
[vagrant@router2 ~]$ sudo -i
[root@router2 ~]# sysctl -w net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.all.rp_filter = 0
[root@router2 ~]# sysctl -w net.ipv4.conf.vlan12.rp_filter=0
net.ipv4.conf.vlan12.rp_filter = 0
```

На этом настройка асимметричной маршрутизации закончена. Окно терминала с router2 также оставляем открытым, оно пригодится для [проверки результата](#%D0%BF%D1%80%D0%BE%D0%B2%D0%B5%D1%80%D0%BA%D0%B0-%D0%BD%D0%B0%D1%81%D1%82%D1%80%D0%BE%D0%B9%D0%BA%D0%B8-%D0%B0%D1%81%D0%B8%D0%BC%D0%BC%D0%B5%D1%82%D1%80%D0%B8%D1%87%D0%BD%D0%BE%D0%B9-%D0%BC%D0%B0%D1%80%D1%88%D1%80%D1%83%D1%82%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D0%B8).

#### Выполнение асимметричной маршрутизации автоматически скриптом ansible-playbook

Чтобы выполнить действия автоматически из `ansible-playbook` выполним команду: `cd ansible; ansible-playbook asymmetric-routing.yml`
После исполнения команды переходим к [следующему разделу](#%D0%BF%D1%80%D0%BE%D0%B2%D0%B5%D1%80%D0%BA%D0%B0-%D0%BD%D0%B0%D1%81%D1%82%D1%80%D0%BE%D0%B9%D0%BA%D0%B8-%D0%B0%D1%81%D0%B8%D0%BC%D0%BC%D0%B5%D1%82%D1%80%D0%B8%D1%87%D0%BD%D0%BE%D0%B9-%D0%BC%D0%B0%D1%80%D1%88%D1%80%D1%83%D1%82%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D0%B8).

#### Проверка настройки асимметричной маршрутизации

Если ранее окно терминала на шаге 2 и 3 [раздела Выполнение асимметричной маршрутизации вручную](#%D0%B2%D1%8B%D0%BF%D0%BE%D0%BB%D0%BD%D0%B5%D0%BD%D0%B8%D0%B5-%D0%B0%D1%81%D0%B8%D0%BC%D0%BC%D0%B5%D1%82%D1%80%D0%B8%D1%87%D0%BD%D0%BE%D0%B9-%D0%BC%D0%B0%D1%80%D1%88%D1%80%D1%83%D1%82%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D0%B8-%D0%B2%D1%80%D1%83%D1%87%D0%BD%D1%83%D1%8E) небыло закрыто, то следующие 5 пунктов можно пропустить.

1. Выполним вход на `router1`: `vagrant ssh router1`
2. Запускаем команду `ping` с интерфейса (ключ -I) `192.168.10.1` на узел `192.168.20.1`: `ping -I 192.168.10.1 192.168.20.1`
   - если настройка выполнена корректно, то должны получить ответы `ICMP`.
3. В новом терминале выполняем вход в папку с репозиторием: `cd ospf`
4. Выполняем вход на `router2`: `vagrant ssh router2`
5. Повышаем привелегии пользователя: `sudo -i`
6. Утилитой `tcpdump` захватываем трафик `ICMP` на интерфейсе `vlan12`: `tcpdump -nvvv -ivlan12 icmp` (чтобы прервать вывод команды нажимаем комбинацию `Ctrl-C`)

```bash
[root@router2 ~]# tcpdump -nvvv -ivlan12 icmp
tcpdump: listening on vlan12, link-type EN10MB (Ethernet), capture size 262144 bytes
16:40:07.087710 IP (tos 0x0, ttl 63, id 61516, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.10.1 > 192.168.20.1: ICMP echo request, id 27696, seq 870, length 64
16:40:08.089454 IP (tos 0x0, ttl 63, id 62402, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.10.1 > 192.168.20.1: ICMP echo request, id 27696, seq 871, length 64
16:40:09.091610 IP (tos 0x0, ttl 63, id 62753, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.10.1 > 192.168.20.1: ICMP echo request, id 27696, seq 872, length 64
^C
3 packets captured
3 packets received by filter
0 packets dropped by kernel
```

_Вывод: как показано на диаграмме ниже, запросы `ICMP echo request` приходят от источника `192.168.10.1` (router1 - eth2) для `192.168.20.1` (router2 - eth2), но ответа нет, потому-что `ICMP echo reply` уходят с другого интерфейса `vlan12`._

![image](https://raw.githubusercontent.com/mmmex/ospf/vlan/OSPF_with_vlan-asyrouting-diagram.png)

7. Прерываем работу утилиты `tcpdump` комбинацией `Ctrl-C`
8. Выполняем захват трафика `ICMP` с интерфейса `vlan11`: `tcpdump -nvvv -ivlan11 icmp`

```bash
[root@router2 ~]# tcpdump -nvvv -ivlan11 icmp
tcpdump: listening on vlan11, link-type EN10MB (Ethernet), capture size 262144 bytes
16:26:11.519162 IP (tos 0x0, ttl 64, id 51231, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.20.1 > 192.168.10.1: ICMP echo reply, id 27696, seq 36, length 64
16:26:12.520840 IP (tos 0x0, ttl 64, id 51232, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.20.1 > 192.168.10.1: ICMP echo reply, id 27696, seq 37, length 64
16:26:13.523011 IP (tos 0x0, ttl 64, id 51406, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.20.1 > 192.168.10.1: ICMP echo reply, id 27696, seq 38, length 64
16:26:14.524780 IP (tos 0x0, ttl 64, id 52047, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.20.1 > 192.168.10.1: ICMP echo reply, id 27696, seq 39, length 64
^C
4 packets captured
4 packets received by filter
0 packets dropped by kernel
```

Видим что ответы `ICMP echo reply` действительно уходят с интерфейса `vlan11` `router2`.

Можем посмотреть таблицу маршрутизации на `router1`: `vtysh -c 'sh ip route'`

```bash
[root@router1 ~]# vtysh -c 'sh ip route'
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, A - Babel, F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

K>* 0.0.0.0/0 [0/101] via 10.0.2.2, eth0, 12:56:40
C>* 10.0.2.0/24 is directly connected, eth0, 12:56:40
O   10.0.10.0/30 [110/135] via 10.0.12.2, vlan10, weight 1, 00:48:09
C>* 10.0.10.0/30 is directly connected, vlan11, 12:56:40
O>* 10.0.11.0/30 [110/90] via 10.0.12.2, vlan10, weight 1, 00:48:09
O   10.0.12.0/30 [110/45] is directly connected, vlan10, weight 1, 12:56:40
C>* 10.0.12.0/30 is directly connected, vlan10, 12:56:40
O   192.168.10.0/24 [110/45] is directly connected, eth2, weight 1, 12:56:40
C>* 192.168.10.0/24 is directly connected, eth2, 12:56:40
O>* 192.168.20.0/24 [110/135] via 10.0.12.2, vlan10, weight 1, 00:48:09
O>* 192.168.30.0/24 [110/90] via 10.0.12.2, vlan10, weight 1, 12:46:15
```

_Здесь мы видим что `router1` может добраться до `192.168.20.1` через своего соседа `router3` (`10.0.12.2`). Если-бы стоимость на интерфейсе `vlan11` была равной 45, то в таблицу маршрутизации попал маршрут до узла который ближе (`router2`)._

Таблица маршрутизации на `router1` до изменения стоимости маршрута:

```bash
router1# sh ip ro
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, A - Babel, F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

K>* 0.0.0.0/0 [0/101] via 10.0.2.2, eth0, 12:58:55
C>* 10.0.2.0/24 is directly connected, eth0, 12:58:55
O   10.0.10.0/30 [110/45] is directly connected, vlan11, weight 1, 00:00:07
C>* 10.0.10.0/30 is directly connected, vlan11, 12:58:55
O>* 10.0.11.0/30 [110/90] via 10.0.10.2, vlan11, weight 1, 00:00:07
  *                       via 10.0.12.2, vlan10, weight 1, 00:00:07
O   10.0.12.0/30 [110/45] is directly connected, vlan10, weight 1, 12:58:55
C>* 10.0.12.0/30 is directly connected, vlan10, 12:58:55
O   192.168.10.0/24 [110/45] is directly connected, eth2, weight 1, 12:58:55
C>* 192.168.10.0/24 is directly connected, eth2, 12:58:55
O>* 192.168.20.0/24 [110/90] via 10.0.10.2, vlan11, weight 1, 00:00:07
O>* 192.168.30.0/24 [110/90] via 10.0.12.2, vlan10, weight 1, 12:48:30
```

Смотрим таблицу маршрутизации на `router2`: `vtysh -c 'sh ip route'`

```bash
[root@router2 ~]# vtysh -c 'sh ip route'
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, A - Babel, F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

K>* 0.0.0.0/0 [0/100] via 10.0.2.2, eth0, 12:54:49
C>* 10.0.2.0/24 is directly connected, eth0, 12:54:49
O   10.0.10.0/30 [110/45] is directly connected, vlan11, weight 1, 12:54:44
C>* 10.0.10.0/30 is directly connected, vlan11, 12:54:49
O   10.0.11.0/30 [110/45] is directly connected, vlan12, weight 1, 12:54:49
C>* 10.0.11.0/30 is directly connected, vlan12, 12:54:49
O>* 10.0.12.0/30 [110/90] via 10.0.10.1, vlan11, weight 1, 12:49:34
  *                       via 10.0.11.1, vlan12, weight 1, 12:49:34
O>* 192.168.10.0/24 [110/90] via 10.0.10.1, vlan11, weight 1, 12:54:34
O   192.168.20.0/24 [110/45] is directly connected, eth2, weight 1, 12:54:49
C>* 192.168.20.0/24 is directly connected, eth2, 12:54:49
O>* 192.168.30.0/24 [110/90] via 10.0.11.1, vlan12, weight 1, 12:49:34
```

### Симметричная маршрутизация

Симметричная сеть имеет один маршрут для входящего и исходящего сетевого трафика. Нам достаточно изменить стоимость на интерфейсе `vlan11` ВМ `router2` чтобы маршрутизация была симметричной:

Если ранее не выходили из терминала с router2, то следующие 2 шага можно пропустить и преступить к 3.

1. Выполним вход на `router2`: `vagrant ssh router2`
2. Повышаем привелегии: `sudo -i`
3. Заходим в консоль управления FRR: `vtysh`
   - `conf t` входим в режим конфигурирования;
   - `int eth1` переходим в раздел конфигурации интерфейса `eth1`;
   - `ip ospf cost 1000` устанавливаем новое значение `cost` в 1000;
   - `exit` выход из режима конфигурирования интерфейса `eth1`;
   - `exit` выход из режима конфигурирования `frr`;
   - `exit` выход из `vtysh`

```bash
test@test-virtual-machine:~/Otus/ospf$ vagrant ssh router2
Last login: Mon Jan 23 16:56:40 2023 from 10.0.2.2
[vagrant@router2 ~]$ sudo -i
[root@router2 ~]# vtysh 

Hello, this is FRRouting (version 8.4.2).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router2# conf t
router2(config)# int vlan11
router2(config-if)# ip ospf cost 1000
router2(config-if)# exit
router2(config)# exit
router2# exit
```

#### Проверка настройки симметричной маршрутизации

Если ранее окно терминала с router1 было открыто, то 1й шаг можно пропустить.

1. Выполним вход на `router1`: `vagrant ssh router1`
2. Запускаем команду `ping` с интерфейса (ключ -I) `192.168.10.1` на узел `192.168.20.1`: `ping -I 192.168.10.1 192.168.20.1`
   - если настройка выполнена корректно, то должны получить ответы `ICMP`.
3. В новом терминале выполняем вход в папку с репозиторием: `cd ospf`
4. Выполняем вход на `router2`: `vagrant ssh router2`
5. Повышаем привелегии пользователя: `sudo -i`
6. Утилитой `tcpdump` захватываем трафик `ICMP` на интерфейсе `vlan12`: `tcpdump -nvvv -i vlan12 icmp` (чтобы прервать вывод команды нажимаем комбинацию `Ctrl-C`)

```bash
[root@router2 ~]# tcpdump -nvvv -ivlan12 icmp
tcpdump: listening on vlan12, link-type EN10MB (Ethernet), capture size 262144 bytes
17:00:12.168241 IP (tos 0x0, ttl 63, id 19709, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.10.1 > 192.168.20.1: ICMP echo request, id 29338, seq 75, length 64
17:00:12.168272 IP (tos 0x0, ttl 64, id 36147, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.20.1 > 192.168.10.1: ICMP echo reply, id 29338, seq 75, length 64
17:00:13.169735 IP (tos 0x0, ttl 63, id 20024, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.10.1 > 192.168.20.1: ICMP echo request, id 29338, seq 76, length 64
17:00:13.169771 IP (tos 0x0, ttl 64, id 36230, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.20.1 > 192.168.10.1: ICMP echo reply, id 29338, seq 76, length 64
^C
4 packets captured
6 packets received by filter
0 packets dropped by kernel
```

_Вывод: как показано на диаграмме ниже, запросы `ICMP echo request` приходят от источника `192.168.10.1` (router1 - eth2) для `192.168.20.1` (router2 - eth2) и `ICMP echo reply` теперь уходят с этого-же интерфейса._

![image](https://raw.githubusercontent.com/mmmex/ospf/vlan/OSPF_with_vlan-syrouting-diagram.png)

7. Прерываем работу утилиты `tcpdump` комбинацией `Ctrl-C`

* Vagrant при создании ВМ использует на гостевой ОС свой интерфейс `10.0.2.15/24` и прокидывает для подключения по `SSH` на адрес хоста `127.0.0.1` порт `TCP 2222` для первой ВМ, `TCP 2200` для второй и т.д.

* Командой `vagrant ssh-config` можем посмотреть настройки интерфейсов. Можно просто сопоставить вывод команды `vagrant ssh-config` с файлом инвентаризации ansible.

* Прописывать соседей OSPF необязательно, если они подключены непосредственно (direct connect).

Полезные ссылки: [статья](https://habr.com/ru/company/ddosguard/blog/503826/)