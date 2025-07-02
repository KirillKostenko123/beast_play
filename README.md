# beast_play
>1. Произведите базовую настройку устройств 
● Настройте имена устройств согласно топологии. Используйте 
полное доменное имя 
● На всех устройствах необходимо сконфигурировать IPv4 
● IP-адрес должен быть из приватного диапазона, в случае, если сеть 
локальная, согласно RFC1918 
● Локальная сеть в сторону HQ-SRV(VLAN100) должна вмещать не 
более 64 адресов 
mkdir /etc/net/ifaces/enp6s19.100
vim /etc/net/if	aces/enp6s19.100/options
TYPE=vlan
HOST=enp6s19
VID=100
BOOTPROTO=static
echo ‘192.168.100.2/26’ > /etc/net/ifaces/enp6s19.100/ipv4address
echo ‘default via 192.168.100.1’ > /etc/net/ifaces/enp6s19.100/ipv4route
echo ‘nameserver 77.88.8.8’ > /etc/net/ifaces/enp6s19.100/resolv.conf
systemctl restart network


vim /etc/net/sysctl.conf
forward =1 СДЕЛАТЬ НА РОУТЕРАХ И ISP
iptables -t nat -A POSTROUTING -o enp6s19 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables

● Локальная сеть в сторону HQ-CLI(VLAN200) должна вмещать не 
более 16 адресов 
● Локальная сеть в сторону BR-SRV должна вмещать не более 32 
адресов 
● Локальная сеть для управления(VLAN999) должна вмещать не 
более 8 адресов 
● Сведения об адресах занесите в отчёт, в качестве примера 
используйте Таблицу 3 
Имя устройства 	ipv4	шлюз
BR-RTR	172.16.5.14/28	172.16.5.1
	192.168.200.1/27	

ISP	DHCP
	172.16.4.1/28	
	172.16.5.1/28
	
HQ-RTR	172.16.4.14/28	172.16.4.1
	192.168.100.1/26	
	192.168.100.65/28	
	192.168.100.81/29	

HQ-SRV	192.168.100.2/26	192.168.100.1
BR-SRV	192.168.200.2/27	192.168.200.1
HQ-CLI	DHCP	                192.168.100.65 (DHCP)
Таблица адресации
2. Настройка ISP 
● Настройте адресацию на интерфейсах: 
o Интерфейс, подключенный к магистральному провайдеру, 
получает адрес по DHCP 
o Настройте маршруты по умолчанию там, где это необходимо 
o Интерфейс, к которому подключен HQ-RTR, подключен к сети 
172.16.4.0/28 
o Интерфейс, к которому подключен BR-RTR, подключен к сети 
172.16.5.0/28 
o На ISP настройте динамическую сетевую трансляцию в сторону 
HQ-RTR и BR-RTR для доступа к сети Интернет 

Cделано

3. Создание локальных учетных записей 
● Создайте пользователя sshuser на серверах HQ-SRV и BR-SRV 
o Пароль пользователя sshuser с паролем P@ssw0rd 
o Идентификатор пользователя 1010 
o Пользователь sshuser должен иметь возможность запускать sudo 
без дополнительной аутентификации. 
● Создайте пользователя net_admin на маршрутизаторах HQ-RTR и 
BR-RTR 
o Пароль пользователя net_admin с паролем P@$$word 
o При настройке на EcoRouter пользователь net_admin должен 
обладать максимальными привилегиями 
o При настройке ОС на базе Linux, запускать sudo без 
дополнительной аутентификации 

HQ-BR SRV
useradd sshuser -u 1010
passwd sshuser
P@ssw0rd
usermod -aG wheel sshuser
echo “sshuser ALL=(ALL:ALL) NOPASSWD; ALL” /etc/sudoers

HQ-BR RTR
useradd net_admin
passwd net_admin
usermod -aG wheel  net_admin
echo “net_admin ALL=(ALL:ALL) NOPASSWD; ALL” /etc/sudoers
4.Настройте на интерфейсе HQ-RTR в сторону офиса HQ виртуальный 
коммутатор: 
● Сервер HQ-SRV должен находиться в ID VLAN 100 
● Клиент HQ-CLI в ID VLAN 200  
● Создайте подсеть управления с ID VLAN 999 
● Основные сведения о настройке коммутатора и выбора реализации 
разделения на VLAN занесите в отчёт 

apt-get update && apt-get install openvswitch  -y
mkdir /etc/net/ifaces/vlan100, 200, 999
vim /etc/net/ifaces/vlan100, 200, 999/options
TYPE=ovsport
BRIDGE=HQ-SW
VID=100 200 999
BOOTPROTO = static

mkdir /etc/net/ifaces/HQ-SW
vim /etc/net/ifaces/HQ-SW/options
TYPE=ovsbr
systemctl enable –now openvswitch
systemctl restart network
modprobe 8021q
echo “8021q” | tee -a /etc/modules

ovs-vsctl add-port HQ-SW enp6s19 trunk=100,200,999
vim /etc/net/ifaces/default/options сделать ovs remove no
Cделано

5. Настройка безопасного удаленного доступа на серверах HQ-SRV и BR
SRV: 
● Для подключения используйте порт 2024 
● Разрешите подключения только пользователю sshuser 
● Ограничьте количество попыток входа до двух 
● Настройте баннер «Authorized access only» 
vim /etc/openssh/sshd_config
PORT=
AllowUsers=
banner /etc/openssh/banner
echo “Authorized access only” > /etc/openssh/banner
systemctl restart sshd
Сделано 

6. Между офисами HQ и BR необходимо сконфигурировать ip туннель 
• Сведения о туннеле занесите в отчёт 
• На выбор технологии GRE или IP in IP 
HQ-RTR
mkdir /etc/net/ifaces/gre1
vim /etc/net/ifaces/gre1/options
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.5.14
TUNREMOTE=172.16.4.14
TUNOPTIONS=’ttl 64’
HOST=enp6s19
echo “10.10.10.1/30” > /etc/net/ifaces/gre1/ipv4address
systemctl restart network

BR-RTR
mkdir /etc/net/ifaces/gre1
vim /etc/net/ifaces/gre1/options
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.4.14
TUNREMOTE=172.16.5.14
TUNOPTIONS=’ttl 64’
HOST=enp6s19
echo “10.10.10.2/30” > /etc/net/ifaces/gre1/ipv4address
systemctl restart network
Сделано
7. Обеспечьте динамическую маршрутизацию: ресурсы одного офиса 
должны быть доступны из другого офиса. Для обеспечения динамической 
маршрутизации используйте link state протокол на ваше усмотрение. 
● Разрешите выбранный протокол только на интерфейсах в ip 
туннеле 
● Маршрутизаторы должны делиться маршрутами только друг с 
другом 
● Обеспечьте защиту выбранного протокола посредством 
парольной защиты 
● Сведения о настройке и защите протокола занесите в отчёт 

8. Настройка динамической трансляции адресов. 
● Настройте динамическую трансляцию адресов для обоих офисов.  
● Все устройства в офисах должны иметь доступ к сети Интернет 
Добавляем прав
9. Настройка протокола динамической конфигурации хостов.  
● Настройте нужную подсеть 
● Для офиса HQ в качестве сервера DHCP выступает маршрутизатор 
HQ-RTR. 
● Клиентом является машина HQ-CLI. 
● Исключите из выдачи адрес маршрутизатора 
● Адрес шлюза по умолчанию – адрес маршрутизатора HQ-RTR. 
● Адрес DNS-сервера для машины HQ-CLI – адрес сервера HQ-SRV. 
● DNS-суффикс для офисов HQ – au-team.irpo 
● Сведения о настройке протокола занесите в отчёт 

10. Настройка DNS для офисов HQ и BR. 
● Основной DNS-сервер реализован на HQ-SRV. 
● Сервер должен обеспечивать разрешение имён в сетевые адреса 
устройств и обратно в соответствии с таблицей 2 
● В качестве DNS сервера пересылки используйте любой 
общедоступный DNS сервер
