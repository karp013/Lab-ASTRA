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

Создать пользователя БД bacula с паролем «bacula» для работы с Bacula

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

adminstd@kmsserver ~ $ `sudo setfacl    -m u:postgres:rx /etc/parsec/capdb`

Пользователю bacula задаем минимальный и максимальный уровень:

adminstd@kmsserver ~ $ `sudo pdpl-user bacula -l 0:0`

```bash
минимальная метка: Уровень_0:Низкий:Нет:0x0
   0:0:0x0:0x0
максимальная метка: Уровень_0:Низкий:Нет:0x0
   0:0:0x0:0x0
```

Далее по очереди выполнить отредактированные сценарии:

adminstd@kmsserver ~ $ `sudo /usr/share/bacula-director/make_postgresql_tables`

adminstd@kmsserver ~ $ `sudo /usr/share/bacula-director/grant_postgresql_privileges`


## Настройка директора

### Основной файл директора

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
  Messages = Standard
  
  # IP-адрес Директора
  DirAddress = 192.168.122.13
}

  # Подключение к базе данных (Bacula Catalog)
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


  # Подключение к Хранилищу (Storage)
Storage {
  Name = stor-sd
  # IP-адрес хранилища.
  Address = 192.168.122.13

  # порт оставляем стандартный
  SDPort = 9103
  Password = "storpass"

  # имя устройства хранения, описаное в файле bacula-sd.conf
  Device = DevStorage

  # Должно соответствовать директиве Media Type ресурса Device настройки
  Media Type = File1

  # Максимальное количество выполняемых заданий.
  # (не рекомендуется одновременно запускать более одного задания)
  Maximum Concurrent Jobs = 1
}


# Настройка групп томов Хранилища (Pool's)
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


Messages {
  Name = Standard

  # команда отправки письма
  mailcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula: %t %e of %c %l\" %r"

  # команда для передачи сообщений оператору
  operatorcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula: Intervention needed for %j\" %r"

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


# эта строка подгружает все конфигурационные файлы из папки «job.d»
@|"sh -c 'for f in /etc/bacula/job.d/*.conf ; do echo @${f} ; done'"


Schedule {
  Name = "WeeklyCycle"

  # Тип бекапа, периодичность и время запуска
  Run = Full 1st sun at 23:05
  Run = Differential 2nd-5th sun at 23:05
  Run = Incremental mon-sat at 23:05
}


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

Поведение уведомлений (Messages)

```bash
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

### Создание задач (Jobs)

adminstd@kmsserver ~ $ `sudo mkdir /etc/bacula/job.d/`

adminstd@kmsserver ~ $ `sn /etc/bacula/job.d/backup-dir-fd.conf`

Задача для создания бекапа

```bash
Job {
  Name = "BackupClient1"
  
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

Задача для восстановления данных из бекапа

adminstd@kmsserver ~ $ `sn /etc/bacula/job.d/restore-dir-fd.conf`

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

### Проверка работоспособности

adminstd@kmsserver ~ $ `sudo /usr/sbin/bacula-dir -t -c /etc/bacula/bacula-dir.conf` 

adminstd@kmsserver ~ $ `sudo systemctl restart bacula-director.service`

adminstd@kmsserver ~ $ `sudo journalctl -xe`


### 2. Настройка Bacula Storage

adminstd@kmsserver ~ $ `sn /etc/bacula/bacula-sd.conf`

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


Director {
  # Имя Директора который может подключаться к этому Хранилищу
  Name = bacula-dir
  # Пароль подключения к этому Хранилищу
  Password = "storpass"
}


Device {
  Name = DevStorage
  Media Type = File1
  Archive Device = /backups/files1
  LabelMedia = yes
  Random Access = Yes
  AutomaticMount = yes
  RemovableMedia = no;
  AlwaysOpen = no;
  Maximum Concurrent Jobs = 1
}


Messages {
  Name = Standard
  director = dir-dir = all
}
```


### Создание папок в которых будет храниться бекап

adminstd@kmsserver ~ $ `sudo mkdir -p /backups/files1/`

adminstd@kmsserver ~ $ `sudo chmod 755 /backups/files1/`

adminstd@kmsserver ~ $ `sudo chown bacula:bacula /backups/files1/`


### Присвоение необходимых прав созданному файлу и назначение ему владельца (chmod and chown on file)

adminstd@kmsserver ~ $ `sudo chmod 644 /etc/bacula/bacula-sd.conf`

adminstd@kmsserver ~ $ `sudo chown root:bacula /etc/bacula/bacula-sd.conf`

adminstd@kmsserver ~ $ `sudo systemctl restart bacula-sd.service`

adminstd@kmsserver ~ $ `sudo journalctl -xe`


## 3. Настройка Bacula FileDaemon

На клиенте

adminstd@kmsclient ~ $ `sudo apt install bacula-fd`

adminstd@kmsclient ~ $ `sn /etc/bacula/bacula-fd.conf`

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


Director {
  # Имя Директора который может подключаться к этому Клиенту
  Name = bacula-dir
  # Пароль подключения к этому Клиенту
  Password = "clientpass"
}


Messages {
  Name = Standard
  director = dir-dir = all, !skipped, !restored
}
```

Присвоение необходимых прав созданному файлу и назначение ему владельца (chmod and chown on file)

adminstd@kmsclient ~ $ `sudo chmod 644 /etc/bacula/bacula-fd.conf`

adminstd@kmsclient ~ $ `sudo chown root:bacula /etc/bacula/bacula-fd.conf`

adminstd@kmsclient ~ $ `sudo /usr/sbin/bacula-fd -t -c /etc/bacula/bacula-fd.conf`

adminstd@kmsclient ~ $ `sudo systemctl restart bacula-fd.service`

adminstd@kmsclient ~ $ `sudo journalctl -xe`


## 4. Настройка Bacula console

adminstd@kmsserver ~ $ `sn /etc/bacula/bconsole.conf`

```bash
Director {
  Name = dir-dir
  DIRport = 9101
  # IP-адрес Директора
  address = 192.168.122.13
  Password = "dirpass"
}
```

### Проверка работоспососбности

adminstd@kmsserver ~ $ `sudo bconsole`

Если в настройках файла bconsole.conf ошибок нет, то подключение пройдет успешно
В появившемся окне набираем можно набрать `status` и выбираем статус какого компонента мы хотим посмотреть.

Далее для создания бекапа пишем

`run`

Выбираем задачу

Поскольку это первый бекап, нужно изменит `Level` с `Incremental` на `Full`

Далее, если немного подождать и написать `messages`, можно увидеть информацию о том, успешно ли прошел процесс резервного копирования

Если проверить папку `/backups/files1/` то там можно увидеть файл бекапа

Также нужно запомнить `JobId`, он понадобится дальше во время восстановления

<details><summary>тык</summary>

```bash
adminstd@kmsserver ~ $ sudo bconsole 
Connecting to Director 192.168.122.13:9101
1000 OK: 103 bacula-dir Version: 9.6.7 (10 December 2020)
Enter a period to cancel a command.
*run
Automatically selected Catalog: BaculaCatalog
Using Catalog "BaculaCatalog"
A job name must be specified.
The defined Job resources are:
     1: BackupClient1
     2: RestoreFiles
Select Job resource (1-2): 1
Run Backup job
JobName:  BackupClient1
Level:    Incremental
Client:   dir-fd
FileSet:  Full Set
Pool:     File (From Job resource)
Storage:  stor-sd (From Job resource)
When:     2024-11-16 22:12:21
Priority: 10
OK to run? (yes/mod/no): mod
Parameters to modify:
     1: Level
     2: Storage
     3: Job
     4: FileSet
     5: Client
     6: When
     7: Priority
     8: Pool
     9: Plugin Options
Select parameter to modify (1-9): 1
Levels:
     1: Full
     2: Incremental
     3: Differential
     4: Since
     5: VirtualFull
Select level (1-5): 1
Run Backup job
JobName:  BackupClient1
Level:    Full
Client:   dir-fd
FileSet:  Full Set
Pool:     File (From Job resource)
Storage:  stor-sd (From Job resource)
When:     2024-11-16 22:12:21
Priority: 10
OK to run? (yes/mod/no): yes
Job queued. JobId=26
You have messages.
*messages
16-ноя 22:12 bacula-dir JobId 26: Bacula bacula-dir 9.6.7 (10Dec20):
  Build OS:               x86_64-pc-linux-gnu AstraLinux 1.8_x86-64
  JobId:                  26
  Job:                    BackupClient1.2024-11-16_22.12.33_12
  Backup Level:           Full
  Client:                 "dir-fd" 9.6.7 (10Dec20) x86_64-pc-linux-gnu,AstraLinux,1.8_x86-64
  FileSet:                "Full Set" 2024-11-15 22:38:12
  Pool:                   "File" (From Job resource)
  Catalog:                "BaculaCatalog" (From Client resource)
  Storage:                "stor-sd" (From Job resource)
  Scheduled time:         16-ноя-2024 22:12:21
  Start time:             16-ноя-2024 22:12:35
  End time:               16-ноя-2024 22:12:37
  Elapsed time:           2 secs
  Priority:               10
  FD Files Written:       1,199
  SD Files Written:       1,199
  FD Bytes Written:       24,872,743 (24.87 MB)
  SD Bytes Written:       25,067,385 (25.06 MB)
  Rate:                   12436.4 KB/s
  Software Compression:   74.7% 4.0:1
  Comm Line Compression:  None
  Snapshot/VSS:           no
  Encryption:             no
  Accurate:               no
  Volume name(s):         Vol-0007
  Volume Session Id:      6
  Volume Session Time:    1731782799
  Last Volume Bytes:      151,002,346 (151.0 MB)
  Non-fatal FD errors:    0
  SD Errors:              0
  FD termination status:  OK
  SD termination status:  OK
  Termination:            Backup OK

16-ноя 22:12 bacula-dir JobId 26: Begin pruning Jobs older than 6 months .
16-ноя 22:12 bacula-dir JobId 26: No Jobs found to prune.
16-ноя 22:12 bacula-dir JobId 26: Begin pruning Files.
16-ноя 22:12 bacula-dir JobId 26: No Files found to prune.
16-ноя 22:12 bacula-dir JobId 26: End auto prune.

```

</details>

Далее восстановим резервную копию

`restore` 

Вводим `JobId`

`ls`

`mark home/`

`done`

Выбираем нужный File Daemon

`yes`

<details><summary>тык</summary>

```bash
*restore
Using Catalog "BaculaCatalog"

First you select one or more JobIds that contain files
to be restored. You will be presented several methods
of specifying the JobIds. Then you will be allowed to
select which files from those JobIds are to be restored.

To select the JobIds, you have the following choices:
     1: List last 20 Jobs run
     2: List Jobs where a given File is saved
     3: Enter list of comma separated JobIds to select
     4: Enter SQL list command
     5: Select the most recent backup for a client
     6: Select backup for a client before a specified time
     7: Enter a list of files to restore
     8: Enter a list of files to restore before a specified time
     9: Find the JobIds of the most recent backup for a client
    10: Find the JobIds for a backup for a client before a specified time
    11: Enter a list of directories to restore for found JobIds
    12: Select full restore to a specified Job date
    13: Cancel
Select item:  (1-13): 3
Enter JobId(s), comma separated, to restore: 26
You have selected the following JobId: 26

Building directory tree for JobId(s) 26 ...  +++++++++++++++++++++++++++++++++++++++++
995 files inserted into the tree.

You are now entering file selection mode where you add (mark) and
remove (unmark) files to be restored. No files are initially added, unless
you used the "all" keyword on the command line.
Enter "done" to leave this mode.

cwd is: /
ls
home/
$ mark home/ 
1,199 files marked.
$ done
Bootstrap records written to /var/lib/bacula/bacula-dir.restore.3.bsr

The Job will require the following (*=>InChanger):
   Volume(s)                 Storage(s)                SD Device(s)
===========================================================================
   
    Vol-0007                  stor-sd                   DevStorage               

Volumes marked with "*" are in the Autochanger.


1,199 files selected to be restored.

Defined Clients:
     1: bacula-fd
     2: dir-fd
Select the Client (1-2): 2
Run Restore job
JobName:         RestoreFiles
Bootstrap:       /var/lib/bacula/bacula-dir.restore.3.bsr
Where:           /home2
Replace:         Always
FileSet:         Full Set
Backup Client:   dir-fd
Restore Client:  dir-fd
Storage:         stor-sd
When:            2024-11-16 22:14:56
Catalog:         BaculaCatalog
Priority:        10
Plugin Options:  *None*
OK to run? (yes/mod/no): yes
Job queued. JobId=27
*
You have messages.
*messages
16-ноя 22:15 bacula-dir JobId 27: Start Restore Job RestoreFiles.2024-11-16_22.14.58_13
16-ноя 22:15 bacula-dir JobId 27: Restoring files from JobId(s) 26
16-ноя 22:15 bacula-dir JobId 27: Using Device "DevStorage" to read.
16-ноя 22:15 stor-sd JobId 27: Ready to read from volume "Vol-0007" on File device "DevStorage" (/backups/files1).
16-ноя 22:15 dir-fd JobId 27: Warning: bxattr_linux.c:280 setxattr error on file "/home2/home/.pdp/": ERR=Отказано в доступе
16-ноя 22:15 stor-sd JobId 27: Forward spacing Volume "Vol-0007" to addr=125867175
16-ноя 22:15 stor-sd JobId 27: End of Volume "Vol-0007" at addr=151002346 on device "DevStorage" (/backups/files1).
16-ноя 22:15 stor-sd JobId 27: Elapsed time=00:00:01, Transfer rate=25.06 M Bytes/second
16-ноя 22:15 dir-fd JobId 27: Warning: bxattr_linux.c:280 setxattr error on file "/home2/home/": ERR=Отказано в доступе
16-ноя 22:15 bacula-dir JobId 27: Bacula bacula-dir 9.6.7 (10Dec20):
  Build OS:               x86_64-pc-linux-gnu AstraLinux 1.8_x86-64
  JobId:                  27
  Job:                    RestoreFiles.2024-11-16_22.14.58_13
  Restore Client:         dir-fd
  Where:                  /home2
  Replace:                Always
  Start time:             16-ноя-2024 22:15:00
  End time:               16-ноя-2024 22:15:01
  Elapsed time:           1 sec
  Files Expected:         1,199
  Files Restored:         1,199
  Bytes Restored:         99,028,909 (99.02 MB)
  Rate:                   99028.9 KB/s
  FD Errors:              0
  FD termination status:  OK
  SD termination status:  OK
  Termination:            Restore OK

16-ноя 22:15 bacula-dir JobId 27: Begin pruning Jobs older than 6 months .
16-ноя 22:15 bacula-dir JobId 27: No Jobs found to prune.
16-ноя 22:15 bacula-dir JobId 27: Begin pruning Files.
16-ноя 22:15 bacula-dir JobId 27: No Files found to prune.
16-ноя 22:15 bacula-dir JobId 27: End auto prune.
```

</details>

Далее можно проверить папку `/home2` на клиенте и увидеть, что там появилась восстановленная резервная копия