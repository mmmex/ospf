## OSPF

Задачи:

- [X] Поднять три виртуалки
- [ ] Объединить их разными vlan
- [X] поднять OSPF между машинами на базе Quagga или FRR или BIRD;
- [X] изобразить ассиметричный роутинг
- [ ] сделать один из линков "дорогим", но что бы при этом роутинг был симметричным.
- [X] Формат сдачи: Vagrantfile + ansible

* Vagrant при создании ВМ использует на гостевой ОС свой интерфейс `10.0.2.15/24` и прокидывает для подключения по `SSH` на адрес хоста `127.0.0.1` порт `TCP 2222` для первой ВМ, `TCP 2200` для второй и т.д.

* Командой `vagrant ssh-config` можем посмотреть настройки интерфейсов. Можно просто сопоставить вывод команды `vagrant ssh-config` с файлом инвентаризации ansible.

* Прописывать соседей OSPF необязательно, если они подключены непосредственно (direct connect).

### Запуск проекта

1. Клонируем репозиторий: `git clone ...`
2. Переходим в каталог: `cd ospf`
3. Запускаем проект: `vagrant up`

Будут подняты 3 ВМ: `router1`, `router2` и `router3` с установленным FRR версии 8.4.2

**Топология сети:**

![image](https://raw.githubusercontent.com/mmmex/ospf/master/OSPF-diagram.png)

**Таблицы хостов:**

**Router1:**
| Интерфейс | Адрес           | Сосед   |
|:---------:|:---------------:|:-------:|
| eth1      | 10.0.10.1/30    | Router2 |
| eth2      | 10.0.12.1/30    | Router3 |
| eth3      | 192.168.10.1/24 | -       |

**Router2:**
| Интерфейс | Адрес           | Сосед   |
|:---------:|:---------------:|:-------:|
| eth1      | 10.0.10.2/30    | Router1 |
| eth2      | 10.0.11.2/30    | Router3 |
| eth3      | 192.168.20.1/24 | -       |

**Router3:**
| Интерфейс | Адрес           | Сосед   |
|:---------:|:---------------:|:-------:|
| eth1      | 10.0.11.1/30    | Router2 |
| eth2      | 10.0.12.2/30    | Router1 |
| eth3      | 192.168.30.1/24 | -       |

При данной конфигурации маршрутизация осуществляется симметрично.

### Асимметричная маршрутизация

![image](https://raw.githubusercontent.com/mmmex/ospf/master/OSPF-asyrouting-diagram.png)

*Что такое асимметричная маршрутизация? При асимметричной маршрутизации пакет проходит от источника к получателю по одному пути и выбирает другой путь, когда возвращается к источнику.*

В основе такой маршрутизации лежит разделение маршрутов исходящего и входящего трафика. Например, входящий трафик может приниматься от одного провайдера, а исходящий будет отправляться через другого. Разделение среды передачи данных приведено для примера и чаще используется единая среда. Использование асимметричной маршрутизации может негативно сказаться на передаче данных в сетях с VPN на пути следования данных. Использование такого типа маршрутизации целесообразно только в больших сетях, в которых больше одного шлюза. В ряде случаев асимметричную маршрутизацию используют с целью снижения нагрузки на канал передачи данных, путем перенаправления части трафика через другие каналы. Такой способ снижения нагрузки применим только в случае, если имеется два и более каналов передачи данных. Также, среди причин использования асимметричной маршрутизации можно рассмотреть коммерческую выгоду при использовании burstable-биллинга, когда организации выгодно платить только за часть трафика. [источник](https://ddos-guard.net/ru/terminology/technology/asimmetrichnaya-marshrutizaciya)

Для выполнения данного вида маршрутизации нужно определенным образом настроить ядро ОС, а также интерфейсы для протокола OSPF. В данной инструкции будет выполнена настройка ВМ `router1` и `router2`.

* В целях экономии времени подготовлен [playbook ansible](ansible/asymmetric-routing.yml), который выполнит необходимые действия автоматически.

#### Выполнение асимметричной маршрутизации вручную

Для конфигурации вручную необходимо выполнить следующие шаги:

1. Отключить в ядре ОС проверку адреса источника глобально и на интерфейсе `eth1` ВМ `router1` (параметр `net.ipv4.conf.all.rp_filter` и `net.ipv4.conf.eth1.rp_filter`):
   - `vagrant ssh router1` выполняем вход на `router1`;
   - `sudo -i` повышаем привелегию пользователя;
   - `sysctl -w net.ipv4.conf.all.rp_filter=0` отключаем глобально проверку;
   - `sysctl -w net.ipv4.conf.eth1.rp_filter=0` отключаем проверку на конкретном интерфейсе;
   - если потребуется запомнить состояние конфигурации, то необходимо записать данные значения в файл `/etc/sysctl.conf`.
2. Увеличить стоимость (`cost`) соединения на интерфейсе `eth1` маршрутизатора `router1`, к примеру до 1000:
   - `vagrant ssh router1` выполняем вход на router1;
   - `sudo -i` повышаем привелегию пользователя;
   - `vtysh` входим в консоль управления FRR;
   - `conf t` входим в режим конфигурирования;
   - `int eth1` переходим в раздел конфигурации интерфейса `eth1`;
   - `ip ospf cost 1000` устанавливаем новое значение `cost` в 1000;
   - `exit` выход из терминала режима конфигурирования.

```bash
test@test-virtual-machine:~/Otus/ospf$ vagrant ssh router1
Last login: Mon Jan 23 11:54:15 2023 from 10.0.2.2
[vagrant@router1 ~]$ sudo -i
[root@router1 ~]# sysctl -w net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.all.rp_filter = 0
[root@router1 ~]# sysctl -w net.ipv4.conf.eth1.rp_filter=0
net.ipv4.conf.eth1.rp_filter = 0
[root@router1 ~]# vtysh 

Hello, this is FRRouting (version 8.4.2).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# conf t
router1(config)# int eth1
router1(config-if)# ip ospf cost 1000
router1(config-if)# exit
router1(config)# exit
```

3. Отключить в ядре ОС проверку адреса источника глобально и на интерфейсе `eth2` ВМ `router2`:
   - `vagrant ssh router2`
   - `sudo -i`
   - `sysctl -w net.ipv4.conf.all.rp_filter=0`
   - `sysctl -w net.ipv4.conf.eth2.rp_filter=0`

```bash
test@test-virtual-machine:~/Otus/ospf$ vagrant ssh router2
Last login: Mon Jan 23 11:54:21 2023 from 10.0.2.2
[vagrant@router2 ~]$ sudo -i
[root@router2 ~]# sysctl -w net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.all.rp_filter = 0
[root@router2 ~]# sysctl -w net.ipv4.conf.eth2.rp_filter=0
net.ipv4.conf.eth2.rp_filter = 0
```

На этом настройка асимметричной маршрутизации закончена. 

#### Выполнение асимметричной маршрутизации автоматически скриптом ansible-playbook

Чтобы выполнить действия автоматически из `ansible-playbook` выполним команду: `cd ansible; ansible-playbook asymmetric-routing.yml`

#### Проверка настройки асимметричной маршрутизации

1. Выполним вход на `router1`: `vagrant ssh router1`
2. Запускаем команду `ping` с интерфейса (ключ -I) `192.168.10.1` на узел `192.168.20.1`: `ping -I 192.168.10.1 192.168.20.1`
3. В новом терминале выполняем вход в папку с репозиторием: `cd ospf`
4. Выполняем вход на ВМ `router2`: `vagrant ssh router2`
5. Повышаем привелегии пользователя: `sudo -i`
6. Утилитой `tcpdump` захватываем трафик `ICMP` на интерфейсе `eth2`: `tcpdump -nvvv -i eth2 icmp` (чтобы прервать вывод команды нажимаем комбинацию `Ctrl-C`)

```bash
[root@router2 ~]# tcpdump -nvvv -i eth2 icmp
tcpdump: listening on eth2, link-type EN10MB (Ethernet), capture size 262144 bytes
16:22:24.795120 IP (tos 0x0, ttl 63, id 31057, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.10.1 > 192.168.20.1: ICMP echo request, id 25615, seq 21, length 64
16:22:25.796533 IP (tos 0x0, ttl 63, id 31090, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.10.1 > 192.168.20.1: ICMP echo request, id 25615, seq 22, length 64
^C
2 packets captured
2 packets received by filter
0 packets dropped by kernel
```

_Вывод: запросы `ICMP echo request` приходят от источника `192.168.10.1` (router1 - eth3) для `192.168.20.1` (router2 - eth3), но ответа нет, потому-что `ICMP echo reply` уходят с другого интерфейса `eth1`_

7. Прерываем работу утилиты `tcpdump` комбинацией `Ctrl-C`
8. Выполняем захват трафика `ICMP` с интерфейса `eth1`: `tcpdump -nvvv -i eth1 icmp`

```bash
[root@router2 ~]# tcpdump -nvvv -i eth1 icmp
tcpdump: listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
17:06:37.010066 IP (tos 0x0, ttl 64, id 1420, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.20.1 > 192.168.10.1: ICMP echo reply, id 25615, seq 2668, length 64
17:06:38.011599 IP (tos 0x0, ttl 64, id 1991, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.20.1 > 192.168.10.1: ICMP echo reply, id 25615, seq 2669, length 64
^C
2 packets captured
2 packets received by filter
0 packets dropped by kernel
```
