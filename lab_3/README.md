# Лабораторная работа 3. VXLAN между двумя маршрутизаторами SONiC в GNS3

Настройка VXLAN-туннеля между двумя виртуальными маршрутизаторами SONiC (`sonic-vs`) в GNS3. Хосты из разных сегментов находятся в одной L2-сети (VLAN 10 → VNI 1000) поверх IP-underlay. Удалённые VTEP обнаруживаются через BGP EVPN.

## Топология

```
Host1 ── Switch1 ── R1 ══════════ R2 ── Switch2 ── Host2
                     (underlay 10.0.0.0/24)
```

| Линк | Сторона A | Сторона B | Интерфейс SONiC |
|---|---|---|---|
| 1 | Host1 e0 | Switch1 Ethernet0 | — |
| 2 | Switch1 Ethernet1 | R1 eth1 | Ethernet0 |
| 3 | R1 eth2 | R2 eth2 | Ethernet4 |
| 4 | R2 eth1 | Switch2 Ethernet1 | Ethernet0 |
| 5 | Switch2 Ethernet0 | Host2 e0 | — |

Адаптер eth0 на R1 и R2 — management, не подключается.

## Адресация

| Узел | Интерфейс | Адрес |
|---|---|---|
| R1 | Ethernet4 | 10.0.0.1/24 |
| R2 | Ethernet4 | 10.0.0.2/24 |
| Host1 | e0 | 192.168.10.11/24 |
| Host2 | e0 | 192.168.10.12/24 |

Overlay: VLAN 10 ↔ VNI 1000, UDP-порт VXLAN 4789, BGP AS 65100 (iBGP).

## Требования

- GNS3
- образ `sonic-vs` (QEMU), RAM от 4 ГБ, два узла
- два VPCS и два встроенных Ethernet Switch
- Wireshark для проверки инкапсуляции

Учётные данные SONiC по умолчанию: `admin` / `YourPaSsWoRd`.

## Конфигурация

### 1. Очистка конфигурации по умолчанию (R1 и R2)

Пресет `t1` вешает IP-адреса вида `10.0.0.x/31` на порты и создаёт 32 BGP-соседа. Они конфликтуют с underlay.

```bash
sudo config interface ip remove Ethernet0 10.0.0.0/31
sudo config interface ip remove Ethernet4 10.0.0.2/31
sudo config interface startup Ethernet0
sudo config interface startup Ethernet4
for k in $(sonic-db-cli CONFIG_DB keys "BGP_NEIGHBOR|*"); do sonic-db-cli CONFIG_DB del "$k"; done
sudo config save -y
sudo config reload -y
```

### 2. Underlay

R1:

```bash
sudo config interface ip add Ethernet4 10.0.0.1/24
```

R2:

```bash
sudo config interface ip add Ethernet4 10.0.0.2/24
```

### 3. VLAN (R1 и R2)

```bash
sudo config vlan add 10
sudo config vlan member add -u 10 Ethernet0
```

Флаг `-u` обязателен: VPCS отправляет кадры без тега.

### 4. VXLAN

R1:

```bash
sudo config vxlan add vtep 10.0.0.1
sudo config vxlan evpn_nvo add nvo vtep
sudo config vxlan map add vtep 10 1000
```

R2:

```bash
sudo config vxlan add vtep 10.0.0.2
sudo config vxlan evpn_nvo add nvo vtep
sudo config vxlan map add vtep 10 1000
```

### 5. BGP EVPN

R1 (`vtysh`):

```
conf t
router bgp 65100
 bgp router-id 10.0.0.1
 neighbor 10.0.0.2 remote-as 65100
 address-family l2vpn evpn
  neighbor 10.0.0.2 activate
  advertise-all-vni
 exit-address-family
end
write memory
```

R2: то же самое, но `bgp router-id 10.0.0.2` и `neighbor 10.0.0.1`.

### 6. Хосты (VPCS)

Host1:

```
ip 192.168.10.11/24
save
```

Host2:

```
ip 192.168.10.12/24
save
```

### 7. Сохранение

```bash
sudo config save -y
```

## Проверка

```bash
show vlan brief
show vxlan interface
show vxlan vlanvnimap
vtysh -c "show bgp l2vpn evpn summary"
show vxlan remotevtep
show vxlan remotemac all
```

С Host1:

```
ping 192.168.10.12
```

Инкапсуляция: в GNS3 ПКМ по линку R1–R2 → **Start capture**, в Wireshark фильтр `vxlan`. В пакете: внешний IP 10.0.0.1 → 10.0.0.2, UDP dst 4789, VNI 1000, внутри Ethernet-кадр с ICMP 192.168.10.11 → 192.168.10.12.

Или на R1:

```bash
sudo tcpdump -i Ethernet4 -nn udp port 4789
```
