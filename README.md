#### УДАЛЕНИЕ CURL
sudo apt remove --purge *curl*

#### РАСЧЁТ МАСКИ

n = 32 - log2(H), где H - кол-во адресов.

адрес .0 - адрес сети, последний адрес - бродкаст(если маска 28, то последний адрес .15), .1 - шлюз, gateway.

#### ПОРТЫ

HQ:
порт1 - интернет
порт5 - ISP
порт10 - SRV1 
BR:
порт5 - ISP
порт10 - SRV2
ISP:
порт2 - BR
порт3 - HQ

#### В НАЧАЛЕ

sudo apt install putty
sudo apt remove brltty 

если до этого вставили консольный кабель в ноут, то вытащите его и вставьте обратно, а то будет ошибка при подключении в дальнейшем 

sudo putty
 
#### ELTEX

сбрасываем конфиг, зажимаем F 

дефолт логин, пароль 
admin, password

ПОСЛЕ ПЕРВОГО ПОДКЛЮЧЕНИЕ ДЕЛАЕМ СМЕНУ ПАРОЛЯ
esr-...(change-expired-password)# password 123admin
commit 
confirm

## OSPF
config
hostname ISP
router ospf 10
area 10.10.0.0
enable
exit
redistribute rip
redistribute connected
redistribute static
enable
exit

## ПОРТ 1 (к HQ)
int gi1/0/3 #в этой строке указан номер порта Eltex, через который он подключен к Mikrotik1 (HQ)
mode routerport 
des TO_HQ
ip address 10.10.1.1/24
ip ospf instance 10
ip ospf area 10.10.0.0
ip ospf
ip firewall disable
exit

## ПОРТ 2 (к BR)
int gi1/0/2 #в этой строке указан номер порта Eltex,  через который он подключен к Mikrotik2 (BR)
mode routerport 
des TO_BR
ip address 10.10.2.1/24
ip ospf instance 10
ip ospf area 10.10.0.0
ip ospf
ip firewall disable
exit

После всех настроек:
end 
commit 
confirm

#### MIKROTIK (HQ)

ПОСЛЕ ПЕРВОГО ПОДКЛЮЧЕНИЕ ОБЯЗАТЕЛЬНО СБРОС КОНФИГА
system reset-configuration

ждём, когда появиться авторизация, дефолт логин - admin, пароль пустой, жмём один раз enter И БОЛЬШЕ НИЧЕГО НЕ ЖМЁМ!!!

после появления буковок жмём клавишу “r”. если вдруг буквы не появились, всё равно жмём “r”.  когда появиться цветный буквы, можно настраивать

/system identity set name=HQ

## Конфигурация портов:
/ip address add address=10.10.1.2/24 interface=ether5

# Втыкаем 1 порт Mikrotik в интернет (кабель в пол) и прописываем команду:
/ip dhcp-client add disabled=no interface=ether1 

## Включаем NAT для работы интернета:
/ip firewall nat add action=masquerade chain=srcnat out-interface=ether1 src-address=10.10.0.0/16
(одна строка)

## OSPF:
/routing ospf instance set default  distribute-default=always-as-type-1 redistribute-static=as-type-1  redistribute-connected=as-type-1 redistribute-rip=as-type-1 router-id=0.0.0.0
/routing ospf area add name=backbone0 area-id=10.10.0.0
/routing ospf network add area=backbone0 network=10.10.0.0/16

# Проверка, что OSPF работает:
/routing ospf neighbor print

# Настройка DHCP (командой bridge port add мы объединяем 2 физических интерфейса в 1 логический, то есть порты 9 и 10 будут использоваться для подключения компов и выдаче адресов по DHCP):
/interface bridge add name=dhcp_hq 
/interface bridge port add bridge=dhcp_hq interface=ether9
/interface bridge port add bridge=dhcp_hq interface=ether10
/ip address add address= 10.10.4.1/26 interface=dhcp_hq network=10.10.4.0

/ip pool add name=HQ_pool ranges=10.10.4.2-10.10.4.62
/ip dhcp-server add address-pool=HQ_pool disabled=no interface=dhcp_hq name=HQ
/ip dhcp-server network add address=10.10.4.0/26 gateway=10.10.4.1 netmask=26

# Проверка, что DHCP работает:
- Подключите ноутбук кабелем к порту микротика
- На ноутбуке напишите команду `ip a`, или ifconfig (но его надо качать)
- Если есть адрес с началом 10.10, то все ок

# пингуем шлюз(HQ) с сервера(SRV1): ping 10.10.4.1
# пингуем сервер(SRV1) со шлюза(HQ): ping 10.10.4.[] - в конце адрес, который достался по DHCP (ip a)

#### MIKROTIK (BR)

ПОСЛЕ ПЕРВОГО ПОДКЛЮЧЕНИЕ ОБЯЗАТЕЛЬНО СБРОС КОНФИГА
system reset-configuration

ждём, когда появиться авторизация, дефолт логин - admin, пароль пустой, жмём один раз enter И БОЛЬШЕ НИЧЕГО НЕ ЖМЁМ!!!

после появления буковок жмём клавишу “r”. если вдруг буквы не появились, всё равно жмём “r”.  когда появиться цветный буквы, можно настраивать

/system identity set name=BR

# Конфигурация портов:
/ip address add address=10.10.2.2/24 interface=ether5

# Настройка  OSPF:
/routing ospf instance set default redistribute-static=as-type-1  redistribute-connected=as-type-1 redistribute-rip=as-type-1 router-id=0.0.0.0
/routing ospf area add name=backbone0 area-id=10.10.0.0
/routing ospf network add area=backbone0 network=10.10.0.0/16

# Проверка, что OSPF работает:
/routing ospf neighbor print

# Настройка DHCP (командой bridge port add мы объединяем в 1 логический интерфейс 2 физических, 
то есть порты 9 и 10 будут использоваться для подключения компов и выдаче адресов по DHCP)
/interface bridge add name=dhcp_BR 
/interface bridge port add bridge=dhcp_BR interface=ether9
/interface bridge port add bridge=dhcp_BR interface=ether10
/ip address add address 10.10.3.1/28 interface=dhcp_BR network=10.10.3.0

/ip pool add name=BR_pool ranges=10.10.3.2-10.10.3.14
/ip dhcp-server add address-pool=BR_pool disabled=no interface=dhcp_BR name=BR
/ip dhcp-server network add address=10.10.3.0/28 gateway=10.10.3.1 netmask=28

# Чтобы на втором микроте были днс сервера, необходимо выполнить следующую команду (чтобы браузер открылся)
/ip dns servers=77.88.8.88,77.88.8.2

# Проверка, что DHCP работает:
- Подключите ноутбук консольным кабелем к порту микротика
- На ноутбуке напишите команду `ip a` 
- Если адрес с началом 10.10, то все ок

# проверка OSPF 
ping 10.10.4.1
(к HQ)
