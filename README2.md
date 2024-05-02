# ПРОПУСКНАЯ СПОСОБНОСТЬ IPERF3

К сожалению, стандартный пакет RouterOS от MikroTik не включает iperf. Вы можете использовать Linux машину или Docker контейнер с iperf3, подключённые к каждому из маршрутизаторов, для выполнения тестов.

sudo apt install iperf3

### запуск на сервере iperf3 (HQ)

iperf3 -s

### запуск на клиенте iperf3 (ISP)

iperf3 -c <IP_адрес_сервера>

ЧТО ПОКАЗЫВАЕТ:
- [ 4] - Это номер потока (или ID соединения). 
- Interval - Это интервал времени, за который были измерены данные. В данном случае с 0.00 до 1.00 секунды.
- Transfer - Объем данных, переданных за указанный интервал времени.
- Bitrate - Средняя скорость передачи данных за интервал, является показателем пропускной способности канала.
- Retr - Количество повторных передач (retransmissions). В TCP это механизм, используемый для обеспечения доставки данных, если предыдущие пакеты были потеряны или повреждены. Значение 0 означает, что повторных передач не было, что указывает на отсутствие ошибок в сетевом соединении во время теста.
- Cwnd - Размер окна конгестии (congestion window). Это количество данных, которое может быть отправлено без подтверждения получения. 

# NTP

### На Mikrotik HQ 

/interface bridge add name=local
/interface bridge port add interface=ether2 bridge=local
/ip address add address=192.168.88.1/24 interface=local

### СОЗДАНИЕ loopback
/interface bridge add name=lo
/ip address add address=10.10.1.100 interface=lo network=10.10.1.100

/system ntp client set enabled=yes primary-ntp=194.190.168.1

162.159.200.1, 91.207.136.55, 162.159.200.123

### ИЛИ

Через web открыть mikrotik и обновить до версии 6.49.15
Скачать https://download.mikrotik.com/routeros/6.49.15/all_packages-arm-6.49.15.zip
Разорхивировать
Найти пакет NTP и загрузить его
делаем Reboot Mikrotik

/system ntp server set broadcast=yes broadcast-addresses=10.10.1.100 enabled=yes

### На Mikrotik BR 
/system ntp client set enabled=yes primary-ntp=10.10.1.100

### на ELTEX
ntp enable
ntp server 10.10.1.100
  prefer
exit

# SSH для удалённого конфигурирования устройства HQ-SRV по порту 2222

## на HQ-SRV

sudo apt install ssh

sudo apt install openssh-server

sudo systemctl enable sshd

systemctl status sshd

sudo nano /etc/ssh/sshd_config
Port 2222

sudo systemctl restart sshd

Разрешите SSH доступ только с IP-адреса CLI устройства:
sudo ufw allow from <CLI-IP-ADDRESS> to any port 2222 proto tcp

## HQ-R

/ip firewall nat add chain=dstnat action=dst-nat protocol=tcp dst-port=2222 to-addresses=<HQ-SRV-IP> to-ports=2222


Войдите в ваш Mikrotik Router через WinBox или SSH.
Откройте список правил Firewall:
- Перейдите к IP -> Firewall -> NAT.
Добавьте новое правило NAT для перенаправления портов:
Нажмите на кнопку + для добавления нового правила.
Во вкладке General, установите:
- Chain: dstnat (это указывает, что правило относится к входящему трафику).
- Protocol: tcp (SSH использует TCP).
- Dst. Port: 2222 (порт на котором внешние пользователи будут инициировать SSH-сессии).
Во вкладке Action, установите:
- Action: dst-nat (действие, которое изменит пункт назначения пакетов).
- To Addresses: IP-адрес сервера HQ-SRV.
- To Ports: 2222 (если SSH сервер на HQ-SRV настроен на этом порту).

## на CLI

ssh username@HQ-address -p 2222 


# Настройте систему управления трафиком на роутере BR-R

- a:
/ip firewall filter add chain=forward action=accept comment="Allow DNS" protocol=udp dst-port=53
/ip firewall filter add chain=forward action=accept comment="Allow DNS" protocol=tcp dst-port=53

/ip firewall filter add chain=forward action=accept comment="Allow HTTP" protocol=tcp dst-port=80
/ip firewall filter add chain=forward action=accept comment="Allow HTTPS" protocol=tcp dst-port=443

- b:
/ip firewall filter add chain=forward action=accept protocol=udp dst-port=4500 comment="Allow NAT-T for IPsec"

- c:
/ip firewall filter add chain=forward action=accept comment="Allow ICMP" protocol=icmp

- d:
/ip firewall filter add chain=forward action=accept comment="Allow SSH" protocol=tcp dst-port=22

- e, f: 
/ip firewall filter add chain=forward action=drop comment="Drop all other traffic"

# ОТ ИГОРЯ

#### Например нужно для адреса 10.10.3.30 заблокировать пинг по 8.8.8.8
/ip firewall filter add action=drop chain=forward dst-address=8.8.8.8 log=yes protocol=icmp src-address=10.10.3.30
#### разрешаем обращаться к нам по 21 порту оттовсюду
/ip firewall filter add action=accept chain=input dst-port=21 protocol=tcp

