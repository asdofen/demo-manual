# Мануал для демонстрационного экзамена
Мануал реализвации сети по заданиям демо экзамена

Образы используемые в работе:

ISP/SRV: ubuntu-server-20.04

R: ubuntu-server-16.04.4-webmin

CLI: debian-10

# Модуль 1


## 1. Составить локальную сеть по схеме и задать адресацию устройствам

![addresses-scheme](https://github.com/asdofen/demo-manual/blob/main/pics/topology-scheme.png)

```
Пулл адресов:
HQ - 64 (255.255.255.192) или /26
BR - 16 (255.255.255.240) или /28
```
```
Серверы имеют статические выделенные адреса:
HQ-SRV - 172.16.1.2/26
BR-SRV - 172.16.2.2/28
```

### Настройкка сети

#### На роутерах (ubuntu-server-16.04) и клиенте настройка сети реализована при помощи ifupdown
```bash
$ sudo nano /etc/network/interfaces
```
```bash
auto lo    # loopback интерфейс необходимо ввести вручную
iface lo inet loopback

auto ens3   # Порт
iface ens3 inet static
  address 192.168.0.2       # Адрес утсройства
  network 192.166.0.0       # Адрес сети
  netmask 255.255.255.0     # Маска подсети
  broadcast 192.168.0.255   # Конечный адрес сети
  gateway 192.168.0.1       # Адрес роутера
```
Применить настройки
```bash
$ sudo systemctl restart networking.service
```

#### На серверах (ubuntu-server-20.04) netplan
```bash
$ sudo nano /etc/netplan/01-netcfg.yaml # Название файла может быть любым, главное чтобы он был формата .yaml
```
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:   # Конфигурируемый порт
      dhcp4: no                       # Изначально стоит yes
      macaddress: XX:XX:XX:XX:XX:XX   # Задаётся в ручную, можно узнать прописав комаду ip a
      addresses: [192.168.0.2/24]
      gateway4: 192.168.0.1           # Адрес роутера
      nameservers:
        addresses: [8.8.8.8]          # DNS для доступа к интернету
```
Применить настройки
```bash
$ sudo netplan apply
```


## 2. Реализовать форвардинг (пока-что по средствам iptables службой MASQUERADE)

Включить формардинг для IPv4 сетей
```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

Настоить постоянный форвардинг
```bash
$ sudo nano /ect/sysctl.conf
```

Найти и раскомментировать строку:
```bash
#net.ipv4.ip_forward=1
```

Поссле чего запустить службу MASQUERADE
```bash
$ sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE   # eth0 - линк порт
```
> Внимание! Каждый раз после перезапуска устройства эту комманду нужно заново вводить


## 3. Настроить DHCP на BR-R

Устанавливаем на сервер DHCP Server
```bash
$ sudo apt install isc-dhcp-server
```

Настроить файл конфигурации dhcpd
```bash
$ sudo nano /etc/dhcp/dhcpd.conf
```

Раскомментировать строку:
```bash
#authoritative
```

В конце файла прописать настройки для DHCP диапозона:
```bash
sunbet 172.16.2.0 netmast 255.255.255.240 {
  range 192.168.188.10 192.168.188.20;
  option routers 172.16.2.1;
  option subnet-mask 255.255.255.240;
  option broadcast-address 172.16.2.15;
}
```

И задать статический адрес для сервера:
```bash
host BR-SRV {
hardware ithernet XX:XX:XX:XX:XX:XX; # MAC-адрес сервера
fixed-address 172.168.2.2;
}
```

После настройки перезапустим DHCP-сервер
```bash
$ sudo systemctl restart isc-dhcp-server
```

Проверим статус сервера
```bash
$ sudo systemctl status isc-dhcp-server
```

Если всё в порядке, запускаем
```bash
$ sudo systemctl enable isc-dhcp-server
```


## 4. Настроить учётные локальные записи на устройствах

```
admin:P@ssw0rd          - CLI HQ-SRV HQ-R
branch-admin:P@ssw0rd   - BR-SRV BRR
network-admin:P@ssw0rd  - HQ-R BR-R BR-SRV
```
Для добавления пользователя нужно войти в систему как root
```bash
adduser admin
```
Задать ему пароль и после настройки выдать права админа sudo
```bash
usermod -aG sudo admin
```

## 5. Измерить пропускную способность между BR-R и ISP

На обоих устройствах установить утилиту iferf3
```bash
$ sudo apt install iperf3
```

Еа ISP запустить прослушивание
```bash
$ iperf3 -s
```

На BR-R запустить проверку
```bash
$ iperf3 -c 10.0.100.1
```


## 6. Создать backup скрипты для файлов конфигурации сети устройств HQ-R и BR-R

В корневой папке root создаём директорию backup и в ней создаём скрипт backups.sh
```bash
$ mkdir backup
$ nano backups.sh
```

В скрипте прописываем команды для копирования файлов в папку backup (или на другой сервер по ssh)
```bash
#!bin/bash

rsync -avz /etc/network/interfaces /root/backup/

rsync -avz /etc/dhcp/dhcpd.conf /root/backup/
```

Делаем скрипт исполняемым
```bash
$ sudo chmod +x backups.sh
```

После чего настравиваем автовыполнение скрипта
```bash
$ sudo crontab -e
```

И в конце файла прописываем условия выполнения скрипта
```bash
0 */1 * * * /root/backup/backups.sh
^  ^  ^ ^ ^
m  h  d m w  # Скрипт будет исполняться каждый час в 0 минут
```

## 7. Настроить контроль подключения по SSH к HQ-SRV по порту 2222

# Модуль 2

# Модуль 3
