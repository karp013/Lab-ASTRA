# Лабораторная работа №6 "Резервное копирование"

## 1. Установить программу Bacula

adminstd@kmsserver ~ $ `sudo apt install bacula`

Высветится предложение автоматической настройки, надо ответить нет, настроим вручную

adminstd@kmsserver ~ $ `sn /etc/postgresql/15/main/postgresql.conf` 

В секции «Connection Settings» изменить значение `listen_addresses=”localhost”` на `listen_addresses=”*”`, если оно еще не изменено.

adminstd@kmsserver ~ $ `sn /etc/postgresql/15/main/pg_hba.conf` 

192.168.122.13/24 - адрес на котором будет работать директор (я решил что он будет на сервере)

```bash
# Database administrative login by Unix domain socket
local   all             postgres                                trust

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
host    all             all             192.168.122.13/24       trust
#host    all             all             0.0.0.0/0            scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256
```

Перезапустить СУБД командой:

adminstd@kmsserver ~ $ `sudo pg_ctlcluster 15 main restart`

Присвоить пароль пользователю postrges:

adminstd@kmsserver ~ $ `sudo passwd postgres`

Присвоить пароль пользователю bacula:

adminstd@kmsserver ~ $ `sudo passwd bacula`


## 2. Настроить пользователя bacula, создать таблицу в доступной СУБД и выдать привилегии на работу с ней пользователю

Создать пользователя БД bacula с паролем «bacula» для работы с Bacula и сделать это надо не от имени учетной записи администратора.

Войти в интерфейс управления psql командой:

adminstd@kmsserver ~ $ `psql template1 -U postgres -h 192.168.122.13 -p 5432`

```bash
psql (15.6 (Debian 15.6-astra.se2+b1))
SSL-соединение (протокол: TLSv1.3, шифр: TLS_AES_256_GCM_SHA384, сжатие: выкл.)
Введите "help", чтобы получить справку.

template1=# 
```

Создаем пользователя bacula командой:

template1=# `CREATE ROLE bacula;`

Присваиваем созданному пользователю пароль: "bacula" следующей командой:

template1=# `ALTER USER bacula WITH PASSWORD 'bacula';`

Создаем суперпользователя для нашей базы:

template1=# `ALTER USER bacula LOGIN SUPERUSER CREATEDB CREATEROLE;`

Выходим из интерфейса управления psql командой:

template1-# `\q`

Далее нам необходимо создать БД bacula, назначить ее владельцем созданного ранее пользователя bacula.

Заходим в интерфейс управления psql командой:

adminstd@kmsserver ~ $ `psql postgres -p 5432 -U postgres`

Создаем БД командой:

postgres=# `CREATE DATABASE bacula WITH ENCODING = 'SQL_ASCII' LC_COLLATE = 'C' LC_CTYPE = 'C' TEMPLATE = 'template0';`

Назначаем владельцем данной БД пользователя "bacula" командой:

postgres=# `ALTER DATABASE bacula OWNER TO bacula;`

Выходим из интерфейса управления psql командой:

postgres=# `\q`

Выходим из терминала суперпользователя

Далее нам необходимо запустить скрипты, которые создадут все необходимые таблицы и привилегии

adminstd@kmsserver ~ $ `sn /usr/share/bacula-director/make_postgresql_tables` 

Тут нужно изменить две строки, чтобы было вот так

```bash
db_name=${db_name:-bacula}
psql -U bacula -h 192.168.122.13 -p 5432 -f - -d ${db_name} $* <<END-OF-DATA
```

adminstd@kmsserver ~ $ `sn /usr/share/bacula-director/grant_postgresql_privileges` 

Должно быть так

```bash
db_user=${db_user:-bacula}
bindir=/usr/bin
db_name=${db_name:-bacula}
db_password=${db_password:-bacula}
if [ "$db_password" != "" ]; then
   pass="password '$db_password'"
fi


$bindir/psql -U bacula -h 192.168.122.13 -p 5432 -f - -d ${db_name} $* <<END-OF-DATA
```

Для корректного функционирования отредактированных сценариев необходимо выдать права на чтение информации из БД пользователей и сведений о метках безопасности, а так же присвоить для этого необходимые атрибуты пользователю postgres:

adminstd@kmsserver ~ $ `sudo usermod -a -G shadow postgres`

adminstd@kmsserver ~ $ `sudo setfacl -d -m u:postgres:r  /etc/parsec/macdb`

adminstd@kmsserver ~ $ `sudo setfacl -R -m u:postgres:r  /etc/parsec/macdb`

adminstd@kmsserver ~ $ `sudo setfacl    -m u:postgres:rx /etc/parsec/macdb`

adminstd@kmsserver ~ $ `sudo setfacl -d -m u:postgres:r /etc/parsec/capdb`

adminstd@kmsserver ~ $ `sudo setfacl -R -m u:postgres:r /etc/parsec/capdb`

adminstd@kmsserver ~ $ `sudo setfacl    -m u:postgres:rx /etc/parsec/capd`

Пользователю bacula задаем минимальный и максимальный уровень:

adminstd@kmsserver ~ $ `sudo pdpl-user bacula -l 0:0`

```bash
инимальная метка: Уровень_0:Низкий:Нет:0x0
   0:0:0x0:0x0
максимальная метка: Уровень_0:Низкий:Нет:0x0
   0:0:0x0:0x0
```

Далее по очереди выполнить отредактированные сценарии:

adminstd@kmsserver ~ $ `sudo /usr/share/bacula-director/make_postgresql_tables`

adminstd@kmsserver ~ $ `sudo /usr/share/bacula-director/grant_postgresql_privileges`


## Bacula Director: настройка

Файл конфигурации Bacula по умолчанию /etc/bacula/bacula-dir.conf имеет в себе более 300 строк, в которых много служебной информации и примеры. Нам все это не нужно. Можно оставить только самое нужное

Основная настройка директора

adminstd@kmsserver ~ $ `sn /etc/bacula/bacula-dir.conf`

```bash
Director {
  Name = bacula-dir

  # какой порт слушать (используется значение по умолчанию)
  DIRport = 9101

  # путь к сценарию, содержащему sql запросы для работы с Bacula Catalog
  QueryFile = "/etc/bacula/scripts/query.sql"

  # папка в которой лежат статус-файлы Директора
  WorkingDirectory = "/var/lib/bacula"

  # pid-файл демона Директора
  PidDirectory = "/run/bacula"

  # Максимальное количество выполняемых заданий.
  # (не рекомендуется одновременно запускать более одного задания)
  Maximum Concurrent Jobs = 1

  # Пароль Директора
  Password = "dirpass"

  # Конфигурация параметров уведомлений (описано в секции Messages)
  Messages = Daemon
  
  # IP-адрес Директора
  DirAddress = 192.168.122.13
}
```

Подключение к базе данных (Bacula Catalog)

```bash
Catalog {
  Name = BaculaCatalog

  # имя базы данных на сервере PostgreSQL
  dbname = "bacula"  
  
  # адрес сервера БД PostgreSQL
  DB Address = "192.168.122.13"  

  # порт подключения на сервере
  DB PORT = 5432

  # имя пользователя базы данных на сервере PostgreSQL
  dbuser = "bacula"  
  
  dbpassword = "bacula"
} 
```

Шаблонное задание (DefaultJob)

```bash
JobDefs {
  Name = "DefaultJob"

  # Тип задания (backup, restore и т.д.)
  Type = Backup

  # Уровень бэкапа (Full, Incremental, Differential и т.п)
  Level = Incremental

  # Имя клиента на котором выполняется задание
  Client = dir-fd

  # Набор файлов для выполнения задания
  FileSet = "Full Set"


  # Расписание выполнения задания
  Schedule = "WeeklyCycle"

  # Файловое хранилище
  Storage = stor-sd

  # Поведение уведомлений
  Messages = Standard

  # Пул, куда будем писать бэкапы. Если мы хотим сделать отдельный пул для каждого клиента,
  # или использовать префиксы, тогда пул указывается в задании для каждого клиента
  # переопределяя тем самым эту настройку
  Pool = File

  # Буферизация атрибутов файлов
  SpoolAttributes = yes

  # Приоритет. Давая заданиям приоритеты от 1 (max) до 10 (min), можно регулировать послед>
  Priority = 10

  # Файл хранит информацию откуда извлекать данные при восстановлении
  Write Bootstrap = "/var/lib/bacula/%c.bsr"
}
```

Подключение к Хранилищу (Storage)

```bash
Storage {
  Name = stor-sd
  # IP-адрес хранилища.
  Address = 192.168.122.13

  # порт оставляем стандартный
  SDPort = 9103
  Password = "storpass"

  # имя устройства хранения, описаное в файле bacula-sd.conf
  Device = Autochanger1

  # Должно соответствовать директиве Media Type ресурса Device настройки
  Media Type = File1

  # Максимальное количество выполняемых заданий.
  # (не рекомендуется одновременно запускать более одного задания)
  Maximum Concurrent Jobs = 1
}
```

Настройка групп томов Хранилища (Pool's)

```bash
Pool {
  Name = File
  # тип пула
  Pool Type = Backup

  # Bacula может автоматически очищать или перезаписывать тома пула
  Recycle = yes

  # период в течении которого информация о заданиях и файлах хранится в БД, по истечению п>
  Volume Retention = 365 days

  # удалять из БД записи о заданиях и файлах , срок хранения которых истёк в соответствии >
  AutoPrune = yes

  # максимальный объем тома в пуле
  Maximum Volume Bytes = 5G

  # максимальное количество томов в пуле
  Maximum Volumes = 100

  # с каких символов начинаются имена томов пула
  Label Format = "Vol-"
}
```

Поведение уведомлений (Messages)

```bash
Messages {
  Name = Standard

  # команда отправки письма
  mailcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula: %t %e >

  # команда для передачи сообщений оператору
  operatorcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula: In>

  # куда = кому = и какие уведомления отправлять
  mail = root = all, !skipped

  # требование к оператору смонтировать том по необходимости
  operator = root = mount

  # какие сообщения выводить в консоль
  console = all, !skipped, !saved

  # путь к логу = какие сообщения записывать в лог
  append = "/var/log/bacula/bacula.log" = all, !skipped

  # записывать в СУБД в таблицу Log; очищается одновременно с удалением записи о задании
  catalog = all
}

# Данная секция описывает поведение уведомлений непосредственно Для самого демона Bacula
Messages {
  Name = Daemon

  # команда отправки письма
  mailcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula daemon >

  # куда = кому = и какие уведомления отправлять 
  mail = root = all, !skipped

  # какие сообщения выводить в консоль
  console = all, !skipped, !saved

  # путь к логу = какие сообщения записывать в лог
  append = "/var/log/bacula/bacula.log" = all, !skipped
}
```

Фрагментация конфигурации (Other conf)

Далее необходимо настроить подгрузку остальных конфигурационных файлов, дописав в этом же файле ( /etc/bacula/bacula-dir.conf ) следующее:

```bash
# Дальнейшие строчки подгружают все конфигурационные файлы из папок
# «job.d» «client.d» «fileset.d» «schedule.d»
 
@|"sh -c 'for f in /etc/bacula/job.d/*.conf ; do echo @${f} ; done'"

@|"sh -c 'for f in /etc/bacula/client.d/*.conf ; do echo @${f} ; done'"

@|"sh -c 'for f in /etc/bacula/fileset.d/*.conf ; do echo @${f} ; done'"

@|"sh -c 'for f in /etc/bacula/schedule.d/*.conf ; do echo @${f} ; done'"
```

Дополнительные конфигурационные файлы


Чтобы было проще ориентироваться в конфигурационных файлах, вынесем часть конфигурации в отдельные файлы.

adminstd@kmsserver ~ $ `sudo mkdir /etc/bacula/schedule.d/`

adminstd@kmsserver ~ $ `sudo mkdir /etc/bacula/client.d/`

adminstd@kmsserver ~ $ `sudo mkdir /etc/bacula/fileset.d/`

adminstd@kmsserver ~ $ `sudo mkdir /etc/bacula/job.d/`

Конфигурации расписании (Schedule's)

adminstd@kmsserver ~ $ sn /etc/bacula/schedule.d/dir-fd.conf

```bash
Schedule {
  Name = "WeeklyCycle"

  # Тип бекапа, периодичность и время запуска
  Run = Full 1st sun at 23:05
  Run = Differential 2nd-5th sun at 23:05
  Run = Incremental mon-sat at 23:05
}
```

Создадим файл настроек расписания обработки файлов при выполнении заданий для Bacula Catalog

adminstd@kmsserver ~ $ sn /etc/bacula/schedule.d/catalog.conf

```bash
Schedule {
  Name = "WeeklyCycleAfterBackup"

  # Тип бекапа, периодичность и время запуска
  Run = Full sun-sat at 23:10
}
```

Конфигурация Клиента (Client)

```bash
Client {
  Name = dir-fd

  #IP-адрес Клиента
  Address = 192.168.122.14

  # порт, который клиент слушает
  FDPort = 9102

  # имя PostgreSQL базы данных Bacula
  Catalog = BaculaCatalog

  # пароль Клиента
  Password = "clientpass"

  # период, в течении которого информация о ФАЙЛАХ хранится в базе данных
  File Retention = 60 days

  # период, в течении которого информация о ЗАДАНИЯХ хранится в базе данных
  Job Retention = 6 months

  # удалять записи из Bacula Catalog старше вышеуказанных значений
  AutoPrune = yes
}
```

Конфигурации наборов файлов (FileSet's)

adminstd@kmsserver ~ $ sn /etc/bacula/fileset.d/dir-fd.conf

```bash
FileSet {
  Name = "Full Set"

  # Секция содержит пути к резервируемым файлам/каталогам
  Include {
  
    # Секция определяющая параметры резервирования файлов/каталогов
    Options {
      signature = MD5
      Compression = GZIP

      # Параметр указывает на необходимость рекурсивного резервирования,
      recurse = yes
 
      # Параметр указывающий на необходимость сохранения ACL информации
      aclsupport = yes
   
      # указывает на возможность включения поддержки расширенных атрибутов,
      # !!! это обязательный параметр для работы с метками безопасности
      xattrsupport = yes
    }

    File = /home
  }

  # пути к файлам/каталогам, которые необходимо исключить
  Exclude {
    File = /tmp
  }
}
```

adminstd@kmsserver ~ $ sn /etc/bacula/fileset.d/catalog.conf

```bash
FileSet {
  Name = "Catalog"
  Include {
    Options {
      signature = MD5
      Compression = GZIP
      aclsupport = yes
      xattrsupport = yes
    }
    File = /var/lib/bacula/bacula.sql
  }
}
```

Конфигурации файлов заданий (Job's)

adminstd@kmsserver ~ $ sn /etc/bacula/job.d/backup-dir-fd.conf

```bash
Job {
  Name = "BackupClient1"
  # Имя шаблонного задания
  JobDefs = "DefaultJob"
}
```

adminstd@kmsserver ~ $ sn /etc/bacula/job.d/restore-dir-fd.conf

```bash
Job {
  Name = "RestoreFiles"
  Type = Restore

  # Клиент на который нужно восстановить файлы
  Client=dir-fd

  # Набор восстанавливаемых файлов
  FileSet="Full Set"

  # Хранилище где лежит бекап клиента
  Storage = stor-sd

  # Пул томов где лежит бекап клиента
  Pool = File

  # Поведение уведомлений

  Messages = Standard

  # Куда на клиенте восстанавливать файлы
  Where = /home2
}
```

Создадим  файл настроек задания для резервирования файлов Bacula Catalog:

```bash
Job {
  Name = "BackupCatalog"

  # Имя шаблонного задания
  JobDefs = "DefaultJob"

  # Уровень бэкапа
  Level = Full

  # Набор восстанавливаемых файлов
  FileSet="Catalog"

  # Расписание
  Schedule = "WeeklyCycleAfterBackup"

  # скрипт выполняемый до основного задания
  RunBeforeJob = "/etc/bacula/scripts/make_catalog_backup.pl BaculaCatalog"

  # скрипт выполняемый после основного задания
  RunAfterJob = "/etc/bacula/scripts/delete_catalog_backup"

  # Файл хранит информацию откуда извлекать данные при восстановлении
  Write Bootstrap = "/var/lib/bacula/%n.bsr"

  # Приоритет запуска после выполнения основного бекапа
  Priority = 11
}
```

Bacula Console (bconsole.conf)

adminstd@kmsserver ~ $ sn /etc/bacula/bconsole.conf 

```bash
Director {
  Name = dir-dir
  DIRport = 9101
  # IP-адрес Директора
  address = 192.168.122.13
  Password = "dirpass"
}
```

ПРОВЕРКА

adminstd@kmsserver ~ $ sudo /usr/sbin/bacula-dir -t -c /etc/bacula/bacula-dir.conf 

sudo systemctl restart bacula-director.service

ЕЩЕ ПРОВЕРКА

sudo journalctl -xe



Bacula Storage: настройка

Файл конфигурации Bacula Storage





```bash
Storage {
  Name = stor-sd
  SDPort = 9103
  
  # папка в которой лежат статус-файлы Хранилища
  WorkingDirectory = "/var/lib/bacula"
  
  # pid-файл демона Хранилища
  Pid Directory = "/run/bacula"
  Maximum Concurrent Jobs = 1
  
  # IP-адрес Хранилища
  SDAddress = 192.168.122.13
}
```

Подключение Директора к этому Хранилищу

```bash
Director {
  # Имя Директора который может подключаться к этому Хранилищу
  Name = bacula-dir
  # Пароль подключения к этому Хранилищу
  Password = "storpass"
}
```

```bash
Autochanger {
  # Имя, указывается В Директоре (директива Device ресурса Storage)
  Name = Autochanger1

  # имена устройств хранения через запятую (ресурсы Device)
  Device = FileChgr1-Dev1, FileChgr1-Dev2

  # /dev/null для виртуального дискового, имеются подстановки
  # (%o - команда (load, unload), %a - устройство для данных, %c - устройство позиционирования и т.д.)
  Changer Command = ""
  # /dev/null для виртуального дискового
  Changer Device = /dev/null
}
```

Устройства хранения

adminstd@kmsserver ~ $ sudo mkdir -p /backups/files1/
adminstd@kmsserver ~ $ sudo chmod 755 /backups/files1/
adminstd@kmsserver ~ $ sudo chown bacula:bacula /backups/files1/

adminstd@kmsserver ~ $ sudo mkdir /backups/files2/
adminstd@kmsserver ~ $ sudo chmod 755 /backups/files2/
adminstd@kmsserver ~ $ sudo chown bacula:bacula /backups/files2/

```bash
Device {
  Name = FileChgr1-Dev1
  Media Type = File1
  Archive Device = /backups/files1
  LabelMedia = yes
  Random Access = Yes
  AutomaticMount = yes
  RemovableMedia = no;
  AlwaysOpen = no;
  Maximum Concurrent Jobs = 1
}

Device {
  Name = FileChgr1-Dev2
  Media Type = File1
  Archive Device = /backups/files2
  LabelMedia = yes
  Random Access = Yes
  AutomaticMount = yes
  RemovableMedia = no
  AlwaysOpen = no
  Maximum Concurrent Jobs = 1
}
```

Поведение уведомлений Хранилища

```bash
Messages {
  Name = Standard
  director = dir-dir = all
}
```

Присвоение необходимых прав созданному файлу и назначение ему владельца (chmod and chown on file)

adminstd@kmsserver ~ $ sudo chmod 644 /etc/bacula/bacula-sd.conf
adminstd@kmsserver ~ $ sudo chown root:bacula /etc/bacula/bacula-sd.conf

adminstd@kmsserver ~ $ sudo systemctl restart bacula-sd.service

adminstd@kmsserver ~ $ sudo journalctl -xe


Bacula FileDaemon (клиент): настройка

на клиенте

 sudo apt install bacula-fd

```bash
FileDaemon {
  Name = dir-fd

  FDport = 9102
  # папка в которой лежат статус-файлы Клиента

  WorkingDirectory = /var/lib/bacula

  # pid-файл демона Клиента
  Pid Directory = /run/bacula
  Maximum Concurrent Jobs = 1

  # возможность расширений FD,
  # расширения оформляются в виде разделяемых библиотек (имя-fd.so) и помещаются в указанный каталог
  Plugin Directory = /usr/lib/bacula

  # fqdn имя или IP-адрес Клиента
  FDAddress = 192.168.122.14
}
```

Подключение Директора к этому Клиенту

```bash
Director {
  # Имя Директора который может подключаться к этому Клиенту
  Name = bacula-dir
  # Пароль подключения к этому Клиенту
  Password = "clientpass"
}
```

Поведение уведомлений Клиента

```bash
Messages {
  Name = Standard
  director = dir-dir = all, !skipped, !restored
}
```


Присвоение необходимых прав созданному файлу и назначение ему владельца (chmod and chown on file)

adminstd@kmsclient ~ $ sudo chmod 644 /etc/bacula/bacula-fd.conf

adminstd@kmsclient ~ $ sudo chown root:bacula /etc/bacula/bacula-fd.conf

adminstd@kmsclient ~ $ sudo /usr/sbin/bacula-fd -t -c /etc/bacula/bacula-fd.conf
adminstd@kmsclient ~ $ sudo systemctl restart bacula-fd.service
adminstd@kmsclient ~ $ sudo journalctl -xe



ПРОВЕРКА РАБОТОСПОСОБНОСТИ

adminstd@kmsserver ~ $ sudo bconsole

Если в настройках файла bconsole.conf ошибок нет, то подключение пройдет успешно
В появившемся окне набираем "status" и выбираем статус какого компонента мы хотим посмотреть.

## 3. Настроить Director daemon на одном устройсве и file daemon на втором

adminstd@kmsserver ~ $ sn /etc/bacula/bacula-dir.conf 

<details><summary>тык</summary>

```bash
Director {                            # define myself
  Name = bacula-dir
  DIRport = 9101                # where we listen for UA connections
  
  # путь к сценарию, содержащему sql запросы для работы с Bacula Catalog
  QueryFile = "/etc/bacula/scripts/query.sql"

  # папка в которой лежат статус-файлы Директора
  WorkingDirectory = "/var/lib/bacula"
  
  # pid-файл демона Директора
  PidDirectory = "/run/bacula"
  
  # Максимальное количество выполняемых заданий.
  # (не рекомендуется одновременно запускать более одного задания)
  Maximum Concurrent Jobs = 1
  Password = "1"         # Console password
  Messages = Daemon
  DirAddress = 192.168.122.13
}

JobDefs {
  Name = "DefaultJob"
  Type = Backup
  Level = Incremental
  Client = kmsserver-fd
  FileSet = "Full Set"
  Schedule = "WeeklyCycle"
  Storage = File1
  Messages = Standard
  Pool = File
  SpoolAttributes = yes
  Priority = 10
  Write Bootstrap = "/var/lib/bacula/%c.bsr"
}

Job {
  Name = "BackupClient1"
  # тип задания
  Type = Backup
  Client = bacula-fd
  FileSet = "Catalog"
  Schedule = "DalyCycle"
  Messages = Standard
  Pool = Default
  Write Bootstrab="var/lib/bacula/%n.bsr"
  Priority = 1
}

Job {
  Name = "RestoreFiles"
  Type = Restore
  Client = bacula-fd
  FileSet ="Catalog"
  Storage = File
  Pool = Default
  Messages = Standard
  Where = /home2
}


# Backup the catalog database (after the nightly save)
Job {
  Name = "BackupCatalog"
  JobDefs = "DefaultJob"
  Level = Full
  FileSet="Catalog"
  Schedule = "WeeklyCycleAfterBackup"
  # This creates an ASCII copy of the catalog
  # Arguments to make_catalog_backup.pl are:
  #  make_catalog_backup.pl <catalog-name>
  RunBeforeJob = "/etc/bacula/scripts/make_catalog_backup.pl MyCatalog"
  # This deletes the copy of the catalog
  RunAfterJob  = "/etc/bacula/scripts/delete_catalog_backup"
  Write Bootstrap = "/var/lib/bacula/%n.bsr"
  Priority = 11                   # run after main backup
}

# List of files to be backed up
FileSet {
  Name = "Catalog"
  Include {
    Options {
      signature = MD5
      compression = GZIP
      aclsupport = yes
      xattrsupport = yes
    }
  File = /home
  }

  Exclude {
    File = /var/lib/bacula
    File = /nonexistant/path/to/file/archive/dir
    File = /proc
    File = /tmp
    File = /sys
    File = /.journal
    File = /.fsck
  }
}

#
# When to do the backups, full backup on first sunday of the month,
#  differential (i.e. incremental since full) every other sunday,
#  and incremental backups other days
Schedule {
  Name = "WeeklyCycle"
  Run = Full 1st sun at 23:05
  Run = Differential 2nd-5th sun at 23:05
  Run = Incremental mon-sat at 23:05
}

# This schedule does the catalog. It starts after the WeeklyCycle
Schedule {
  Name = "WeeklyCycleAfterBackup"
  Run = Full sun-sat at 23:10
}

# Client (File Services) to backup
Client {
  Name = bacula-fd
  Address = 192.168.122.14
  FDPort = 9102
  Catalog = MyCatalog
  Password = "1"          # password for FileDaemon
  File Retention = 30 days            # 30 days
  Job Retention = 6 months            # six months
  AutoPrune = yes                     # Prune expired Jobs/Files
}

Storage {
  Name = File
  Address = 10.0.0.24
  SDPort = 9103
  Password = "1"
  Device = FileStorage
  Media Type = File
}

# Definition of file Virtual Autochanger device
Autochanger {
  Name = File1
# Do not use "localhost" here
  Address = localhost                # N.B. Use a fully qualified name here
  SDPort = 9103
  Password = "xOAAcsWJ2uLR7-CgIkiWkqr3Lem8CklOS"
  Device = FileChgr1
  Media Type = File1
  Maximum Concurrent Jobs = 10        # run up to 10 jobs a the same time
  Autochanger = File1                 # point to ourself
}

# Definition of a second file Virtual Autochanger device
#   Possibly pointing to a different disk drive
Autochanger {
  Name = File2
# Do not use "localhost" here
  Address = localhost                # N.B. Use a fully qualified name here
  SDPort = 9103
  Password = "xOAAcsWJ2uLR7-CgIkiWkqr3Lem8CklOS"
  Device = FileChgr2
  Media Type = File2
  Autochanger = File2                 # point to ourself
  Maximum Concurrent Jobs = 10        # run up to 10 jobs a the same time
}

# Generic catalog service
Catalog {
  Name = MyCatalog
  dbname = "bacula"; DB Address = "10.0.0.23"; dbuser = "bacula"; dbpassword = "bacula"
}

# Reasonable message delivery -- send most everything to email address
#  and to the console
Messages {
  Name = Standard
  mailcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula: %t %e of %c %l\" %r"
  operatorcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula: Intervention needed for %j\" %r"
  mail = root = all, !skipped
  operator = root = mount
  console = all, !skipped, !saved
  append = "/var/log/bacula/bacula.log" = all, !skipped
  catalog = all
}


# Message delivery for daemon messages (no job).
Messages {
  Name = Daemon
  mailcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula daemon message\" %r"
  mail = root = all, !skipped
  console = all, !skipped, !saved
  append = "/var/log/bacula/bacula.log" = all, !skipped
}

# Default pool definition
Pool {
  Name = Default
  Pool Type = Backup
  Recycle = yes                       # Bacula can automatically recycle Volumes
  AutoPrune = yes                     # Prune expired volumes
  Volume Retention = 365 days         # one year
  Maximum Volume Bytes = 50G          # Limit Volume size to something reasonable
  Maximum Volumes = 100               # Limit number of Volumes in Pool
}

# File Pool definition
Pool {
  Name = File
  Pool Type = Backup
  Recycle = yes                       # Bacula can automatically recycle Volumes
  AutoPrune = yes                     # Prune expired volumes
  Volume Retention = 365 days         # one year
  Maximum Volume Bytes = 50G          # Limit Volume size to something reasonable
  Maximum Volumes = 100               # Limit number of Volumes in Pool
  Label Format = "Vol-"               # Auto label
}


# Scratch pool definition
Pool {
  Name = Scratch
  Pool Type = Backup
}

#
# Restricted console used by tray-monitor to get the status of the director
#
Console {
  Name = bacula-mon
  Password = "1"
  CommandACL = status, .status
}
```

</details>

## 4. Настроить доступ к коснсоли (это производится в bconsole.conf на каждом устройстве)

adminstd@kmsserver ~ $ sn /etc/bacula/bconsole.conf 

```bash
Director {
  Name = bacula-dir
  DIRport = 9101
  address = 192.168.122.13
  Password = "1"
}
```

## 5. Настроить резервное копирование вашей домашней директории с клиента на сервер


