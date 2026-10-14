# Лабораторная работа 4. OpenFlow в GNS3 с OpenDaylight и SONiC

Два коммутатора SONiC с Open vSwitch работают под управлением внешнего SDN-контроллера OpenDaylight. Контроллер ставит потоки (flow entries) на коммутаторы по OpenFlow 1.3, хосты обмениваются трафиком, а правила меняются через RESTCONF.


## Стенд

| Компонент | Версия / вариант |
|---|---|
| Хост | Ubuntu (VirtualBox) |
| Эмулятор | GNS3 |
| Контроллер | OpenDaylight Oxygen SR4 (0.8.4), Java 8, установлен на Ubuntu |
| Коммутаторы | SONiC-VS x2 + Open vSwitch 3.5.0 |
| Хосты | VPCS x2 |

## Топология

(скрин 1)

| Откуда | Куда | Назначение |
|---|---|---|
| NAT1 | Switch1 | management |
| SW1 eth0 | Switch1 | management SW1 |
| SW2 eth0 | Switch1 | management SW2 |
| PC1 e0 | SW1 eth1 | data plane |
| PC2 e0 | SW2 eth1 | data plane |
| SW1 eth2 | SW2 eth2 | транзит между коммутаторами |

| Узел | Интерфейс | Адрес |
|---|---|---|
| Ubuntu (ODL) | virbr0 | 192.168.122.1/24 |
| SW1, SW2 | eth0 | 192.168.122.x/24 (DHCP) |
| PC1 | e0 | 192.168.1.11/24 |
| PC2 | e0 | 192.168.1.12/24 |

eth0 у SONiC используется только для управления (связь с ODL и выход в интернет). eth1 и eth2 включены в мост OVS `br0` и несут трафик хостов.

## Развёртывание

### 1. Подготовка Ubuntu

```bash
sudo apt update
sudo apt install -y openjdk-8-jdk-headless curl wget libvirt-daemon-system
sudo systemctl enable --now libvirtd
sudo virsh net-start default
sudo virsh net-autostart default
sudo iptables -I INPUT -i virbr0 -p tcp --dport 6633 -j ACCEPT
sudo iptables -I INPUT -i virbr0 -p tcp --dport 6653 -j ACCEPT
```

Сеть `default` создаёт `virbr0` с адресом `192.168.122.1`, через неё работает узел NAT в GNS3. Если `net-start` падает с ошибкой про порт 53, нужно остановить BIND: `sudo systemctl stop named`.

### 2. OpenDaylight

```bash
wget https://nexus.opendaylight.org/content/repositories/opendaylight.release/org/opendaylight/integration/karaf/0.8.4/karaf-0.8.4.tar.gz
tar xzf karaf-0.8.4.tar.gz
cd karaf-0.8.4
```

В `etc/org.apache.karaf.features.cfg` в конец строки `featuresBoot` дописать:

```
,odl-restconf,odl-l2switch-switch-ui,odl-dluxapps-topology,odl-dluxapps-nodes,odl-dluxapps-yangui
```

Запуск:

```bash
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
rm -rf data journal snapshots
./bin/karaf clean
```

Через 2-3 минуты проверка:

```bash
sudo ss -tlnp | grep -E '6633|6653|8181'
```

Веб-интерфейс: `http://localhost:8181/index.html` (`admin` / `admin`).

(скрин 2)

> Модули ставятся через `featuresBoot`, а не командой `feature:install` из консоли: в Oxygen при установке на лету часть бандлов (openflowplugin, aaa) падает в `Failure`.

### 3. Open vSwitch на SONiC

На каждом SONiC (логин `admin`, пароль `YourPaSsWoRd`) после получения адреса на eth0:

```bash
sudo apt-get update
sudo apt-get install -y openvswitch-switch
sudo systemctl enable --now openvswitch-switch
```

Конфигурация моста SW1:

```bash
sudo ip link set eth1 up
sudo ip link set eth2 up
sudo ovs-vsctl add-br br0
sudo ovs-vsctl set bridge br0 protocols=OpenFlow13
sudo ovs-vsctl set bridge br0 other-config:datapath-id=0000000000000001
sudo ovs-vsctl set-fail-mode br0 secure
sudo ovs-vsctl add-port br0 eth1
sudo ovs-vsctl add-port br0 eth2
sudo ovs-vsctl set-controller br0 tcp:192.168.122.1:6633
```

На SW2 то же самое, но `datapath-id=0000000000000002`.

Проверка:

```bash
sudo ovs-vsctl show
sudo ovs-ofctl -O OpenFlow13 dump-flows br0
```

(скрин 3)

(скрин 4)

(скрин 5)

(скрин 6)

### 4. Хосты

```
PC1> ip 192.168.1.11/24
PC1> save
PC2> ip 192.168.1.12/24
PC2> save
```

## Проверка работы

### Контроллер видит коммутаторы



```bash
curl -s -u admin:admin http://localhost:8181/restconf/operational/opendaylight-inventory:nodes | python3 -m json.tool | grep '"id": "openflow:[0-9]*"'
```

(скрин 7)

```bash
grep -iE "openflow:1|openflow:2" data/log/karaf.log | tail -20
```

(скрин 8)

### Ping между хостами

```
PC1> ping 192.168.1.12
```

(скрин 9)

## Управление потоками через RESTCONF

Правило `drop-icmp` с приоритетом 1000 блокирует весь ICMP на SW1 (`openflow:1`).

`drop-icmp.json`:

```json
{
  "flow-node-inventory:flow": [
    {
      "id": "drop-icmp",
      "table_id": 0,
      "priority": 1000,
      "flow-name": "drop-icmp",
      "match": {
        "ethernet-match": { "ethernet-type": { "type": 2048 } },
        "ip-match": { "ip-protocol": 1 }
      },
      "instructions": {
        "instruction": [
          {
            "order": 0,
            "apply-actions": {
              "action": [ { "order": 0, "drop-action": {} } ]
            }
          }
        ]
      }
    }
  ]
}
```

Добавление:

```bash
curl -u admin:admin -X PUT -H "Content-Type: application/json" -d @drop-icmp.json http://localhost:8181/restconf/config/opendaylight-inventory:nodes/node/openflow:1/flow-node-inventory:table/0/flow/drop-icmp
```

Чтение:

```bash
curl -s -u admin:admin http://localhost:8181/restconf/config/opendaylight-inventory:nodes/node/openflow:1/flow-node-inventory:table/0/flow/drop-icmp | python3 -m json.tool
```

(скрин 10)

(Скрин 11)

(скрин 12)

Удаление:

```bash
curl -u admin:admin -X DELETE http://localhost:8181/restconf/config/opendaylight-inventory:nodes/node/openflow:1/flow-node-inventory:table/0/flow/drop-icmp
```

(скрин 13)

(скрин 14)

## Запуск после перезагрузки

На Ubuntu:

```bash
sudo systemctl stop named
sudo virsh net-start default
sudo iptables -I INPUT -i virbr0 -p tcp --dport 6633 -j ACCEPT
sudo iptables -I INPUT -i virbr0 -p tcp --dport 6653 -j ACCEPT
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
cd ~/karaf-0.8.4 && ./bin/karaf
```

На SW1 и SW2, если адреса или порты слетели:

```bash
sudo ip link set eth0 up
sudo dhclient eth0
sudo ip link set eth1 up
sudo ip link set eth2 up
sudo ovs-vsctl show
```

В `ovs-vsctl show` у контроллера должно быть `is_connected: true`.
