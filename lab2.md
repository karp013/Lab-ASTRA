# Лабораторная работа №2 Система доменных имен (DNS), DHCP

## Задание 1

### 1. На обе виртуальные машины установите пакет bind9

`sudo apt install bind9`

### 2. На server машине проделайте следующие шаги:

#### Опишите зону DNS "Ваши_инициалы.miet.stu"

#### Опишите обратную зону DNS для подсети 192.168.122


`adminstd@kmsserver:~$ sudo nano /etc/bind/named.conf.local`

```bash
zone "kms.miet.stu" {
        type master;
        file "/etc/bind/zones/db.kms.miet.stu";
};

zone "122.168.192.in-addr.arpa" {
        type master;
        file "etc/bind/zones/192.168.122";
};
```


`adminstd@kmsserver:$ sudo mkdir etc/bind/zones`

`adminstd@kmsserver:$ sudo touch /etc/bind/zones/db.kms.miet.stu`

`adminstd@kmsserver:$ sudo nano /etc/bind/zones/db.kms.miet.stu`

/etc/bind/zones/db.kms.miet.stu

```bash
$TTL 604800
kms.miet.stu.   IN      SOA srv.kms.miet.stu. karpukhin235@gmail.com (
                                2024100501
                                3h
                                1h
                                1w
                                1h
                        )
kms.miet.stu.           IN      NS      srv.kms.miet.stu.
srv.kms.miet.stu.       IN      A       192.168.122.13
cli.kms.miet.stu.       IN      A       192.168.122.14
neighbor                IN      CNAME   cli.kms.miet.stu.
```

/etc/bind/zones/192.168.122

```bash
$TTL 604800
122.168.192.in-addr.arpa.       IN      SOA srv.kms.miet.stu. karpukhin235@gmail.com (
                                        2024100501
                                        3h
                                        1h
                                        1w
                                        1h
                                        ) 
122.168.192.in-addr.arpa.       IN      NS srv.kms.miet.stu.

13                              IN      PTR srv.kms.miet.stu.
14                              IN      PTR cli.kms.miet.stu.
```

#### Проверьте правильность внесенных изменений

`adminstd@kmsserver:/etc/bind/zones$ sudo named-checkconf /etc/bind/named.conf`

`adminstd@kmsserver:/etc/bind/zones$ sudo named-checkzone kms.miet.stu /etc/bind/zones/db.kms.miet.stu`

`adminstd@kmsserver:/etc/bind/zones$ sudo named-checkzone 122.168.192.in-addr.arpa /etc/bind/zones/192.168.122`

`systemctl restart bind9`

#### Создайте каталог /etc/bind/zones. В нем создайне файлы с ресурсными записями для созданных вами зон. Включите в данную зону три машины - две созданные вами (server, client) и еще одну с именем client_2 и адресом  192.168.122.(№ в группе + 3)

/etc/bind/zones/db.kms.miet.stu

```bash
$TTL 604800
kms.miet.stu.   IN      SOA srv.kms.miet.stu. karpukhin235@gmail.com (
                                2024100501
                                3h
                                1h
                                1w
                                1h
                        )
kms.miet.stu.           IN      NS      srv.kms.miet.stu.
srv.kms.miet.stu.       IN      A       192.168.122.13
cli1.kms.miet.stu.      IN      A       192.168.122.14
cli2.kms.miet.stu.      IN      A       192.168.122.15

server                  IN      CNAME   srv.kms.miet.stu.
client1                 IN      CNAME   cli1.kms.miet.stu.
client2                 IN      CNAME   cli2.kms.miet.stu.
```

/etc/bind/zones/192.168.122

```bash
$TTL 604800
122.168.192.in-addr.arpa.       IN      SOA srv.kms.miet.stu. karpukhin235@gmail.com (
                                        2024100501
                                        3h
                                        1h
                                        1w
                                        1h
                                        ) 
122.168.192.in-addr.arpa.       IN      NS srv.kms.miet.stu.

13                              IN      PTR srv.kms.miet.stu.
14                              IN      PTR cli1.kms.miet.stu.
15                              IN      PTR cli2.kms.miet.stu.
```

#### Проверьте правльиность внесенных изменений

`adminstd@kmsserver:/etc/bind/zones$ sudo named-checkconf /etc/bind/named.conf`

`adminstd@kmsserver:/etc/bind/zones$ sudo named-checkzone kms.miet.stu /etc/bind/zones/db.kms.miet.stu`

`adminstd@kmsserver:/etc/bind/zones$ sudo named-checkzone 122.168.192.in-addr.arpa /etc/bind/zones/192.168.122`

#### Перезапустите bind9 и поочередно отправьте ping сообщение машинам с именами server, client, clietn_2, client_3. Объясните полученный результат

### 3. Настройте client машину таким образом, чтобы было возможно отправлять ping сообщения по доменным именам.

sudo systemctl restart bind9

на клиенте:

```bash
# Generated by NetworkManager
# nameserver 192.168.122.1
nameserver 192.168.122.13
domain kms.miet.stu
```

ping клиента 1 и сервера работает, ping client2 не работает, потому что у нас нет такой машины

## Задание 2

### 1. Установите DHCP сервер на серверную машину

на сервере:

`sudo apt-get install fly-admin-dhcp`

/etc/default/isc-dhcp-server

```bash
INTERFACESv4="eth0"
INTERFACESv6=""
```

### 2. Выделите диапазон 192.168.122.(№ в группе + 100) - (№ в группе + 80) для выдачи динамических адресов

на сервере:

/etc/dhcp/dhcpd.conf

```bash
subnet 192.168.122.0 netmask 255.255.255.0
{
range 192.168.122.92 192.168.122.112;
}
```

`sudo dhcpd -t` - проверка что синтаксис правильный

### 3. Запустите службу DHCP и убедитесь, что она работает корректно

на сервере:

`sudo systemctl restart isc-dhcp-server`

`sudo systemctl status isc-dhcp-server` - статус должен быть active

на клиенте:

`sudo dhclient` - запрос ip адреса

делаем `ip a` и убеждаемся, что выдался адрес в диапазоне 192.168.122.92 192.168.122.112

`sudo tail -f /var/log/syslog` - тут можно увидеть весь процесс выдачи адреса 

если хотим сбросить выданный адрес, то прописываем `sudo dhclient -r`

### 4. Измените настройки DHCP таким образом, чтобы машине клиента всегда выдавался адрес 192.168.122.(ваш день рождения)

чтобы статически выдать адрес клиенту, нужно знать мак адрес клиента

делаем `ip a` на клиенте

рядом с link/ether будет написан mac адрес

![alt text](.pic/image-1.png)

на сервере:

/etc/dhcp/dhcpd.conf

добавляем в конце

```bash
host static-client {
    hardware ethernet 52:54:00:a0:5a:0c;  # MAC-адрес устройства
    fixed-address 192.168.122.21;         # Статический IP-адрес
}
```
