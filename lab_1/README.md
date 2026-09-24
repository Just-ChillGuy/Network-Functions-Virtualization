# Лабораторная работа №1. Установка GNS3 и базовая настройка

Репозиторий с материалами для защиты лабораторной работы по дисциплине «Виртуализация сетевых функций».

## Цель работы

Установить и настроить GNS3, подключить эмуляторы (Dynamips, QEMU, VirtualBox), подготовить образ SONiC и собрать тестовый проект для проверки связности между узлами.


## Требования

- GNS3 2.x или новее ([gns3.com/software/download](https://www.gns3.com/software/download))
- QEMU (устанавливается вместе с GNS3 на Windows)
- Опционально: VirtualBox или VMware, если используется GNS3 VM
- Образ SONiC VS (sonic-vs.img)

## Инструкция по воспроизведению

### 1. Установка GNS3

Скачать установщик с официального сайта и выполнить установку, оставив компоненты по умолчанию (Dynamips, VPCS, QEMU, Wireshark, uBridge).


### 2. Проверка эмуляторов

Открыть Edit -> Preferences и проверить:

- Dynamips: путь к dynamips.exe, кнопка Test settings
- QEMU: наличие qemu-system-x86_64.exe в списке бинарников
- VirtualBox (если используется GNS3 VM): путь к VBoxManage.exe

<img width="1280" height="923" alt="image" src="https://github.com/user-attachments/assets/e5b9d7c7-87dc-4e1c-b576-97e9dfc8432f" />


### 3. Добавление образа SONiC

1. Скачать sonic-vs.img.gz со сборочного сервера [sonic-buildimage](https://github.com/sonic-net/sonic-buildimage) (Azure Pipelines, артефакт target/sonic-vs.img.gz)
2. Распаковать архив в .img
3. В Edit -> Preferences -> QEMU -> Qemu VMs создать новый шаблон:
   - Qemu binary: qemu-system-x86_64.exe
   - RAM: 4096 MB
   - Console type: telnet
   - Disk image (hda): указать распакованный sonic-vs.img

<img width="1280" height="919" alt="image" src="https://github.com/user-attachments/assets/05ec6f39-3ff6-4332-96fa-f337794ce749" />


### 4. Сборка тестового проекта

1. Создать новый проект в GNS3
2. Добавить два узла VPCS: PC1 и PC2
3. Соединить узлы кабелем через Ethernet0
4. Запустить узлы (Play)
5. Назначить адреса:
   ```
   PC1> ip 192.168.1.1/24
   PC2> ip 192.168.1.2/24
   ```

<img width="1280" height="924" alt="image" src="https://github.com/user-attachments/assets/3e0c2c2c-8f0c-4d90-bb48-59d1e4453126" />


### 5. Проверка связности

С узла PC1 выполнить:

ping 192.168.1.2

Ожидаемый результат -- успешные ответы от 192.168.1.2.

<img width="1280" height="405" alt="image" src="https://github.com/user-attachments/assets/1294bbf2-3f69-4237-adc0-1dba9fa1d930" />
