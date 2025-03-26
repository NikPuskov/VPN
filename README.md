# VPN

--------------------------------------------------------------------------------------------------------------------------

Описание домашнего задания:

- Настроить VPN между двумя ВМ в tun/tap режимах, замерить скорость в туннелях, сделать вывод об отличающихся показателях

- Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на ВМ

--------------------------------------------------------------------------------------------------------------------------

Инструкция по выполнению ДЗ:

1. Настроить VPN между двумя ВМ в tun/tap режимах

- Запускаем 2 виртуальные машины с помощью Vagrant - host1 и host2:

`vagrant up host1`

`vagrant up host2`

- Устанавливаем необходимые пакеты на двух серверах и отключаем Selinux:

`apt update`

`apt install openvpn iperf3 selinux-utils`

`setenforce 0`

1.1. Настраиваем host1

- Cоздаем файл-ключ:

`openvpn --genkey secret /etc/openvpn/static.key`

`cp /etc/openvpn/static.key /vagrant/`

- Cоздаем конфигурационный файл OpenVPN /etc/openvpn/server.conf со следующим содержимым:

image1

- Создаем service unit для запуска OpenVPN /etc/systemd/system/openvpn@.service со следующим содержимым:

image2

- Запускаем сервис:

`systemctl start openvpn@server`

`systemctl enable openvpn@server`

1.2. Настраиваем host2

- Cоздаем конфигурационный файл OpenVPN /etc/openvpn/server.conf со следующим содержимым:

image3

- Копируем в директорию /etc/openvpn файл-ключ static.key, который был создан на host1:

`cp /vagrant/static.key /etc/openvpn/`

- Создаем service unit для запуска OpenVPN /etc/systemd/system/openvpn@.service со следующим содержимым:

image4

- Запускаем сервис:

`systemctl start openvpn@server`

`systemctl enable openvpn@server`

Замеряем скорость в туннеле. На host1 запускаем:

`iperf3 -s`

- На host2:

image5

- И в обратную сторону:

image6

- Меняем в конфигурационных файлах режим работы с tap на tun. Замеряем скорость соединения:

image7

- И в обратную сторону:

image8

Выводы: Видим что разницы в пропускной способности между tun и tap режимом нет - ~ 300 Mbit/s. Основное отличие между tun и tap, это то что tap работает на L2, а tun на L3. tap может быть полезен, например, когда в туннельной сети есть тонкие клиенты или устройства с загрузкой по сети. Без VPN туннеля пропускная способность гораздо выше:

image9

2. RAS на базе OpenVPN

- Поднимаем две виртуальные машины "server" и "client":

`vagrant up server`

`vagrant up client`

На сервере:

- Устанавливаем необходимые пакеты:

`apt update`

`apt install openvpn easy-rsa selinux-utils`

- Отключаем SELinux.

- Переходим в директорию /etc/openvpn и инициализируем PKI:

`cd /etc/openvpn`

`/usr/share/easy-rsa/easyrsa init-pki`

- Генерируем необходимые ключи и сертификаты для сервера:

`/usr/share/easy-rsa/easyrsa build-ca nopass`

`echo 'rasvpn' | /usr/share/easy-rsa/easyrsa gen-req server nopass`

`echo 'yes' | /usr/share/easy-rsa/easyrsa sign-req server server`

`/usr/share/easy-rsa/easyrsa gen-dh`

`openvpn --genkey secret ca.key`

- Генерируем необходимые ключи и сертификаты для клиента:

`echo 'client' | /usr/share/easy-rsa/easyrsa gen-req client nopass`

`echo 'yes' | /usr/share/easy-rsa/easyrsa sign-req client client`

`cp /etc/openvpn/pki/issued/client.crt /vagrant`

`cp /etc/openvpn/pki/private/client.key /vagrant`

`cp /etc/openvpn/pki/ca.crt /vagrant`

- Создаем конфигурационный файл сервера /etc/openvpn/server.conf:

image10

- Зададим параметр iroute для клиента:

`echo 'iroute 10.10.10.0 255.255.255.0' > /etc/openvpn/client/client`

- Запускаем сервис (при необходимости создать файл юнита как в задании 1):

`systemctl start openvpn@server`

`systemctl enable openvpn@server`

На клиенте:

- Создаем файл /etc/openvpn/client.conf со следующим содержимым:

image11

- Копируем в одну директорию с client.conf файлы с сервера:

`cd /etc/openvpn`

`cp /vagrant/ca.crt .`

`cp /vagrant/client.crt .`

`cp /vagrant/client.key .`

- Далее можно проверить подключение с помощью:

`openvpn --config client.conf`

- При успешном подключении проверяем пинг по внутреннему IP адресу сервера в туннеле:

image12

- Также проверяем командой ip r (netstat -rn) на хостовой машине что сеть туннеля импортирована в таблицу маршрутизации:

image13
