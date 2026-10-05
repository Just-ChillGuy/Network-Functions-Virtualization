# Лабораторная работа №2. Настройка маршрутизатора SONiC в GNS3 и построение простой сети

Репозиторий с материалами для защиты лабораторной работы по дисциплине «Виртуализация сетевых функций».

## Цель работы

Создать в GNS3 тестовую сеть из одного маршрутизатора на базе SONiC, двух коммутаторов и двух хостов, настроить базовую маршрутизацию между двумя подсетями и убедиться в корректном взаимодействии всех узлов.


## Схема сети

```
PC1 (192.168.10.10/24) -- Switch1 -- Router SONiC -- Switch2 -- PC2 (192.168.20.10/24)
                                      192.168.10.1        192.168.20.1
```

Порту GNS3 `Ethernet1` маршрутизатора соответствует внутренний интерфейс SONiC `Ethernet0` (сегмент к Switch1), порту GNS3 `Ethernet2` -- интерфейс `Ethernet4` (сегмент к Switch2). Порт `Ethernet0` (GNS3) / `eth0` зарезервирован под управление и в топологии не используется.

## Инструкция по воспроизведению

### 1. Сборка топологии

1. Создать новый проект в GNS3
2. Добавить узлы: 1 x SONiC (QEMU), 2 x Ethernet switch, 2 x VPCS
3. Соединить: `PC1 -> Switch1 -> Router (Ethernet1) ... Router (Ethernet2) -> Switch2 -> PC2`
4. Запустить все узлы (`Play`)

<img width="847" height="441" alt="image" src="https://github.com/user-attachments/assets/e0ab9e61-8b5f-413a-95ee-0bb2cf712e66" />



### 2. Настройка маршрутизатора SONiC

Подключиться к консоли SONiC (логин `admin`), дождаться запуска служб `swss`/`syncd` (проверка -- `docker ps`, `show interfaces status`) и назначить адреса интерфейсов:

```
sudo config interface ip add Ethernet0 192.168.10.1/24
sudo config interface ip add Ethernet4 192.168.20.1/24
sudo config interface startup Ethernet0
sudo config interface startup Ethernet4
```

Проверить результат:

```
show interfaces status
show ip interfaces
ip route
```

В таблице маршрутизации должны появиться записи `192.168.10.0/24 dev Ethernet0` и `192.168.20.0/24 dev Ethernet4`. Сохранить конфигурацию:

```
sudo config save -y
```

<img width="477" height="141" alt="image" src="https://github.com/user-attachments/assets/2e9ac721-f1c3-4c9c-bf43-6c37279d436d" />


### 3. Настройка хостов

В консоли каждого VPCS:

```
PC1> ip 192.168.10.10 255.255.255.0 192.168.10.1
PC2> ip 192.168.20.10 255.255.255.0 192.168.20.1
```

Проверить назначение командой `show ip` -- поле `GATEWAY` должно быть заполнено.

<img width="476" height="35" alt="image" src="https://github.com/user-attachments/assets/43ea3b0b-d419-42fe-b5f3-c507504f2e28" />


### 4. Проверка связности

```
PC1> ping 192.168.10.1
PC2> ping 192.168.20.1
PC1> ping 192.168.20.10
PC1> trace 192.168.20.10
```

Ожидаемый результат -- успешный ping до собственного шлюза и до удалённого хоста (TTL=63, поскольку пакет проходит через один маршрутизатор), а в trace первым хопом -- адрес `192.168.10.1`.

<img width="478" height="41" alt="image" src="https://github.com/user-attachments/assets/63bc8fbb-afcc-47da-bb7b-7c699e4c6519" />

<img width="472" height="86" alt="image" src="https://github.com/user-attachments/assets/5edbab55-849c-4c90-9d8f-3c95967684d9" />



