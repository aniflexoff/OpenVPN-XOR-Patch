# OpenVPN-XOR-Patch
Репозиторий содержит инструкции и скрипты для настройки сервера OpenVPN с использованием XOR Patch (scramble).

## 1. Настройка сервера
### 1.1. Выбор порта для OpenVPN
На вашем сервере выберите случайный номер порта между 10 000 и 50 000 для OpenVPN.

```bash
awk -v min=10000 -v max=50000 'BEGIN{srand(); print int(min+rand()*(max-min+1))}'
```
Для примера, мы будем использовать в остальной части этого гайда порт 16273.

### 1.2. Установка и настройка брандмауэра
Существует несколько способов реализации брандмауэра: nftables, iptables, ufw и firewalld. Современный способ - это nftables, и именно это мы будем использовать здесь. Введите каждую из этих команд по очереди, чтобы установить и запустить nftables:

```bash
apt update && apt upgrade -y
apt install nftables -y
systemctl enable nftables
systemctl start nftables
```

Настройте брандмауэр для принятия связанного трафика и внутреннего трафика:

```bash
nft add rule inet filter input ct state related,established counter accept
nft add rule inet filter input iif lo counter accept
```

Откройте порт 22 для SSH. Если вы можете ограничить правило порта 22, чтобы только определенные IP-адреса источника были в белом списке для доступа по SSH, то тем лучше.

```bash
nft add rule inet filter input tcp dport 22 counter accept
```

Добавьте правило для открытия порта OpenVPN, который вы выбрали случайным образом:

```bash
nft add rule inet filter input udp dport 16273 counter accept
```

Добавьте правило для отбрасывания всех входящих пакетов:

```bash
nft add rule inet filter input counter drop
```

Теперь добавьте таблицу для перевода сетевых адресов (NAT) и маскирования исходящего IP-адреса:

```bash
nft add table nat
nft add chain nat prerouting { type nat hook prerouting priority 0 \; }
nft add chain nat postrouting { type nat hook postrouting priority 100 \; }
nft add rule nat postrouting masquerade
```

Сохраните правила:

```bash
nft list ruleset > /etc/nftables.conf
```

### 1.3. Разрешение пересылки
Включите пересылку IPv4 в ядре Linux. Отредактируйте файл настройки системного контроля:

```bash
nano /etc/sysctl.conf
```

Раскомментируйте строку:

```bash
net.ipv4.ip_forward=1
```

Сохраните файл. Примените новые настройки, выполнив команду:

```bash
sysctl -p
```

### 1.4. Получение исходного кода OpenVPN и XOR Patch
Вы должны использовать соответствующие версии OpenVPN и патча XOR.

Откройте браузер и перейдите на страницу выпусков GitHub для OpenVPN. Определите последний номер выпуска. На момент написания это версия v2.5_beta3. Возможно, на момент прочтения этой статьи это будет более поздняя версия. Мы будем использовать v2.5_beta3 в наших примерах, хотя вам может потребоваться заменить его.

Скачайте и извлеките архив OpenVPN для вашей версии:

```bash
wget https://github.com/aniflexoff/OpenVPN-XOR-Patch/releases/download/files/openvpn-2.6.8.tar.gz
tar -xvf openvpn-2.6.8.tar.gz
```

Скачайте и извлеките репозиторий Tunnelblick с патчами XOR:

```bash
wget https://github.com/aniflexoff/OpenVPN-XOR-Patch/releases/download/files/Tunnelblick-master.zip
apt install unzip -y
unzip Tunnelblick-master.zip
```

### 1.5. Применение патчей
Скопируйте файлы патчей в каталог OpenVPN, заменив openvpn-2.6.8 на текущую версию:

```bash
cp Tunnelblick-master/third_party/sources/openvpn/openvpn-2.6.8/patches/*.diff openvpn-2.6.8
```

Примените патчи к исходному коду OpenVPN:

```bash
cd openvpn-2.6.8
apt install patch -y
patch -p1 < 02-tunnelblick-openvpn_xorpatch-a.diff
patch -p1 < 03-tunnelblick-openvpn_xorpatch-b.diff
patch -p1 < 04-tunnelblick-openvpn_xorpatch-c.diff
patch -p1 < 05-tunnelblick-openvpn_xorpatch-d.diff
patch -p1 < 06-tunnelblick-openvpn_xorpatch-e.diff
patch -p1 < 10-route-gateway-dhcp.diff
```

### 1.6. Build OpenVPN with XOR Patch
Установите необходимые компоненты для сборки:

```bash
apt install build-essential libssl-dev iproute2 liblz4-dev liblzo2-dev libpam0g-dev libpkcs11-helper1-dev libsystemd-dev resolvconf pkg-config autoconf automake libtool libcap-ng-dev liblz4-dev libsystemd-dev liblzo2-dev libpam0g libpam0g-dev -y
```

Скомпилируйте и установите OpenVPN с патчем XOR:

```bash
autoreconf -i -v -f
./configure --enable-systemd --enable-async-push --enable-iproute2
make
make install
```

Программа устанавливается в /usr/local/sbin/openvpn.

### 1.7. Создание каталогов конфигурации
Создайте каталоги, в которых будут храниться ключи, сертификаты и файлы конфигурации OpenVPN:

```bash
mkdir -p /etc/openvpn/{server,client}
```

### 1.8. Создание ключей и сертификатов с помощью EasyRSA
Установите пакет EasyRSA:

```bash
apt install easy-rsa -y
```

Сделайте копию скриптов и файлов конфигурации EasyRSA:

```bash

cp -r /usr/share/easy-rsa ~
cd ~/easy-rsa
```

Сделайте копию примера переменных:

```bash
cp vars.example vars
```

Вы можете отредактировать файл vars, если хотите, но мы просто будем использовать значения по умолчанию. Инициализируйте общественную инфраструктуру ключей:

```bash
./easyrsa init-pki
```

Постройте ваш собственный Центр сертификации (CA):

```bash
./easyrsa build-ca nopass
```

Дайте центру сертификации (CA) имя, или просто нажмите Enter, чтобы принять имя Easy-RSA CA по умолчанию.

Сгенерируйте и подпишите ключ и сертификат сервера. Мы используем для примера имя сервера "server":

```bash
./easyrsa gen-req server nopass
./easyrsa sign-req server server
yes
```

Сгенерируйте и подпишите ключ и сертификат клиента. Мы используем для примера имя "client_name". Вы можете изменить его на имя по вашему выбору.

```bash
./easyrsa gen-req client_name nopass
./easyrsa sign-req client client_name
yes
```

Сгенерируйте параметры Диффи-Хеллмана. Это может занять продолжительное время.

```bash
./easyrsa gen-dh
```

Сгенерируйте предварительный ключ для шифрования начального обмена:

```bash
openvpn --genkey secret pki/tls-crypt.key
```

Скопируйте все ключи и сертификаты в каталог OpenVPN:

```bash
cp pki/ca.crt /etc/openvpn
cp pki/private/server.key /etc/openvpn/server
cp pki/issued/server.crt /etc/openvpn/server
cp pki/private/client_name.key /etc/openvpn/client_name.key
cp pki/issued/client_name.crt /etc/openvpn/client_name.crt
cp pki/tls-crypt.key /etc/openvpn
cp pki/dh.pem /etc/openvpn
```

### 1.9. Генерация кода обфускации Scramble
Для обфускации генерируйте код из 192 бит (24 байта), выраженный в 32 символах в кодировке base64:

```bash
openssl rand -base64 24
```

Примерный результат, который мы будем использовать в остальной части этого гайда:

```bash
r7EaFR2DshpQT+QMfQGYO5BXC2BAV8JG
```

### 1.10. Настройка сервера OpenVPN
Отредактируйте файл конфигурации OpenVPN:

```bash
nano /etc/openvpn/server.conf
```

Замените случайный порт "16273" в примере на свой собственный случайный порт.
Замените сгенерированный код обфускации "r7EaFR2DshpQT+QMfQGYO5BXC2BAV8JG" на свой собственный случайный код.

```bash
port 16273
proto udp
dev tun
ca /etc/openvpn/ca.crt
cert /etc/openvpn/server/server.crt
key /etc/openvpn/server/server.key
dh /etc/openvpn/dh.pem
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist /etc/openvpn/ipp.txt
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
keepalive 10 120
cipher AES-128-GCM
tls-crypt /etc/openvpn/tls-crypt.key
persist-key
persist-tun
status openvpn-status.log
verb 3
scramble obfuscate r7EaFR2DshpQT+QMfQGYO5BXC2BAV8JG
```

Сохраните файл.

### 1.11. Настройка systemd
Создайте файл службы systemd для OpenVPN:

```bash
nano /lib/systemd/system/openvpn@.service
```

Вставьте содержимое, аналогичное этому:

```bash
[Unit]
Description=OpenVPN connection to %i
PartOf=openvpn.service
ReloadPropagatedFrom=openvpn.service
Before=systemd-user-sessions.service
After=network-online.target
Wants=network-online.target
Documentation=man:openvpn(8)
Documentation=https://community.openvpn.net/openvpn/wiki/Openvpn24ManPage
Documentation=https://community.openvpn.net/openvpn/wiki/HOWTO

[Service]
Type=notify
PrivateTmp=true
WorkingDirectory=/etc/openvpn
ExecStart=/usr/local/sbin/openvpn --daemon ovpn-%i --status /run/openvpn/%i.status 10 --cd /etc/openvpn --config /etc/openvpn/%i.conf --writepid /run/openvpn/%i.pid
PIDFile=/run/openvpn/%i.pid
KillMode=process
ExecReload=/bin/kill -HUP $MAINPID
CapabilityBoundingSet=CAP_IPC_LOCK CAP_NET_ADMIN CAP_NET_BIND_SERVICE CAP_NET_RAW CAP_SETGID CAP_SETUID CAP_SYS_CHROOT CAP_DAC_OVERRIDE CAP_AUDIT_WRITE
LimitNPROC=100
DeviceAllow=/dev/null rw
DeviceAllow=/dev/net/tun rw
ProtectSystem=true
ProtectHome=true
RestartSec=5s
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Сохраните файл.

Создайте каталог для файла идентификации процесса (pid):

```bash
mkdir /run/openvpn
```

### 1.12. Запуск OpenVPN
Запустите OpenVPN на сервере:

```bash
systemctl enable openvpn@server
systemctl start openvpn@server
```

Проверьте, что он активен и слушает на указанном вами порту:

```bash
systemctl status openvpn@server
ss -tulpn | grep openvpn
```

Настройка на сервере завершена.

## 2. SНастройка клиента
### 2.1. Загрузка ключей и сертификатов
Теперь перейдите к работе на вашем компьютере. Предполагая, что ваш сервер имеет IP-адрес yy.yy.yy.yy, а вы назвали ключ и сертификат клиента debian10.*, скопируйте необходимые файлы с сервера на клиент, например так:

```bash
scp root@yy.yy.yy.yy:/etc/openvpn/client/client_name.key ~/Downloads/client_name.key
scp root@yy.yy.yy.yy:/etc/openvpn/client/client_name.crt ~/Downloads/client_name.crt
scp root@yy.yy.yy.yy:/etc/openvpn/ca.crt ~/Downloads/ca.crt
scp root@yy.yy.yy.yy:/etc/openvpn/tls-crypt.key ~/Downloads/tls-crypt.key
```

### 2.2. Получение исходного кода OpenVPN и патча XOR
Для клиента Debian/Ubuntu этот процесс практически такой же, как на сервере. Номер версии OpenVPN и патча XOR будет таким же, как на сервере. Мы будем использовать openvpn-2.6.8 в наших примерах.

Загрузите архив OpenVPN для вашей версии:

```bash
cd ~/Downloads
wget https://github.com/TheHavlok/Openvpn-XOR-Patch/releases/download/files/openvpn-2.6.8.tar.gz
tar -xvf openvpn-2.6.8.tar.gz
```

Загрузите репозиторий Tunnelblick с патчем XOR:

```bash
wget https://github.com/TheHavlok/Openvpn-XOR-Patch/releases/download/files/Tunnelblick-master.zip
sudo apt install unzip -y
unzip Tunnelblick-master.zip
```

### 2.3. Применение патчей
Скопируйте файлы патчей в каталог OpenVPN, заменив openvpn-2.6.8 на текущую версию:

```bash
cp Tunnelblick-master/third_party/sources/openvpn/openvpn-2.6.8/patches/*.diff openvpn-2.6.8
```

Примените патчи к исходному коду OpenVPN:

```bash
cd openvpn-2.6.8
sudo apt install patch -y
patch -p1 < 02-tunnelblick-openvpn_xorpatch-a.diff
patch -p1 < 03-tunnelblick-openvpn_xorpatch-b.diff
patch -p1 < 04-tunnelblick-openvpn_xorpatch-c.diff
patch -p1 < 05-tunnelblick-openvpn_xorpatch-d.diff
patch -p1 < 06-tunnelblick-openvpn_xorpatch-e.diff
patch -p1 < 10-route-gateway-dhcp.diff
```

### 2.4. борка OpenVPN с патчем XOR
Установите необходимые компоненты для сборки:

```bash
apt install build-essential libssl-dev iproute2 liblz4-dev liblzo2-dev libpam0g-dev libpkcs11-helper1-dev libsystemd-dev resolvconf pkg-config autoconf automake libtool libcap-ng-dev liblz4-dev libsystemd-dev liblzo2-dev libpam0g libpam0g-dev -y
```

Скомпилируйте и установите OpenVPN с патчем XOR:

```bash
autoreconf -i -v -f
./configure --enable-systemd --enable-async-push --enable-iproute2
make
sudo make install
```

Программа устанавливается в /usr/local/sbin/openvpn.

### 2.5. Исправление разрешения DNS
Есть проблема с игнорированием переданных DNS-серверов в OpenVPN в Linux. Решение зависит от того, что управляет серверами имен. Вот решение, которое работало на клиенте Debian 10 с NetworkManager. Есть альтернативное решение для Ubuntu с Systemd.

Отредактируйте конфигурацию NetworkManager:

```bash
sudo nano /etc/NetworkManager/NetworkManager.conf
```

В разделе [main] вставьте строку:

```bash
dns=none
```

Сохраните файл. Затем отредактируйте файл /etc/resolv.conf:

```bash
sudo nano /etc/resolv.conf
```

Измените содержимое файла, чтобы указать серверы DNS, которые будут доступны при включенном VPN, например:

```bash
nameserver 8.8.8.8
nameserver 8.8.4.4
```

Сохраните файл. Перезапустите NetworkManager:

```bash
sudo systemctl restart NetworkManager
```

### 2.6. Создание файла конфигурации клиента OpenVPN
Создайте файл конфигурации клиента для OpenVPN:

```bash
cd ~/Downloads
nano debian10.conf
```

Вставьте следующие детали конфигурации, адаптируя их под вашу ситуацию:

Замените "yy.yy.yy.yy" на публичный IP-адрес вашего сервера
Замените "16273" на ваш случайный порт
Замените "r7EaFR2DshpQT+QMfQGYO5BXC2BAV8JG" на ваш случайный код обфускации

```bash
client
dev tun
proto udp
remote yy.yy.yy.yy 16273
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert debian10.crt
key debian10.key
remote-cert-tls server
cipher AES-128-GCM
tls-crypt tls-crypt.key
verb 3
scramble obfuscate r7EaFR2DshpQT+QMfQGYO5BXC2BAV8JG
```

Сохраните файл.

### 2.7. Запуск клиента OpenVPN
Откройте терминал на вашем клиентском компьютере и запустите OpenVPN:

```bash
cd ~/Downloads
sudo /usr/local/sbin/openvpn --config debian10.conf
```

Оставьте терминал открытым с запущенным OpenVPN.

### 2.8. Проверка конечного соединения
Откройте Firefox.

Посетите IP Chicken.

Вы должны увидеть IP-адрес вашего удаленного сервера, а не вашего локального клиента.
