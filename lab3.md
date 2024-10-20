# Лабораторная работа №3 "Веб-сервер. межсетевой экран и прокси-сервер"

## Задание 1

### Выполните в терминале HTTP-запрос станицы с информацие про сеть TCP/IP, лежащего по адресу lib.ru/unixhelp/network.txt. HTTP-ответ сахраните в домашней директории под именем network.html

adminstd@kmsclient:~$ `sudo apt install netcat`

adminstd@kmsclient:~$ `echo -e "GET http://lib.ru/unixhelp/network.txt HTTP/1.0\n\n" | nc lib.ru 80 | iconv -f koi8-r -t UTF-8 > network.html`

`HTTP/1.0` - указание версии протокола

`nc` приводит к созданию TCP-подключения с указанными реквизитами и 
замыканием стандартного ввода на сетевой вывод и наоборот, стандартного 
вывода на сетевой ввод. Такая функциональность напоминает команду cat

`iconv -f koi8-r -t UTF-8` - для указания кодировки

### Откройте с помощью браузера файл network.html и убедитесь, что текст правильно отформатирован.

adminstd@kmsclient ~ $ `firefox network.html`

## Задание 2

### Создайте в директории /var/www/site/ HTML-документ

adminstd@kmsserver ~ $ `sudo mkdir /var/www/`

adminstd@kmsserver ~ $ `sudo mkdir /var/www/site/`

adminstd@kmsserver ~ $ s`udo touch /var/www/site/mydoc.html`


### Установите веб-сервер Nginx и настройте его работу так, чтобы он при HTTP-запросе выдавал созданный вами HTML-документ

Включить extended репозиторий астры, если не включен

adminstd@kmsclient ~ $ `sudo apt install nginx`

adminstd@kmsclient ~ $ `sudo systemctl enable nginx`

Сделаем backup конфигурационного файла:

adminstd@kmsserver /etc/nginx/sites-enabled $ `sudo mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.orig`

adminstd@kmsserver /etc/nginx/sites-enabled $ `sn /etc/nginx/nginx.conf`

`/etc/nginx/nginx.conf`

```bash
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
        worker_connections 10;
}
http {
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 60;

        access_log /var/www/site/access.log;
        error_log /var/www/site/error.log;
server {
        listen 80 default_server;
        root /var/www/site;
        index site.html;
        server_name site.kms.miet.stu;
}
}
```

adminstd@kmsserver /var/www/site $ `sudo systemctl restart nginx.service` 

adminstd@kmsserver /etc/nginx/sites-enabled $ `sudo nginx -t`

adminstd@kmsserver /etc/nginx/sites-enabled $ `sudo systemctl restart nginx`


### Добавьте созданную страницу в список записей DNS и откройте её с машины клиента, используя доменное имя site."Ваши инициалы".miet.stu






