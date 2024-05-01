# NTP

### На Mikrotik HQ 

/interface bridge add name=local
/interface bridge port add interface=ether2 bridge=local
/ip address add address=192.168.88.1/24 interface=local

### СОЗДАНИЕ loopback
/interface bridge add name=lo
/ip address add address=10.10.1.100 interface=lo network=10.10.1.100

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
