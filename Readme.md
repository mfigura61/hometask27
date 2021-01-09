# Otus ДЗ PostgreSQL Replication (Centos 7)

```
PostgreSQL
Цель: Студент получил навыки работы сhot_standby.
- Настроить hot_standby репликацию с использованием слотов
- Настроить правильное резервное копирование

Для сдачи работы присылаем ссылку на репозиторий, в котором должны обязательно быть Vagranfile и плейбук Ansible, конфигурационные файлы postgresql.conf, pg_hba.conf и recovery.conf, а так же конфиг barman, либо скрипт резервного копирования. Команда "vagrant up" должна поднимать машины с настроенной репликацией и резервным копированием. Рекомендуется в README.md файл вложить результаты (текст или скриншоты) проверки работы репликации и резервного копирования.
```

## Запуск

1. ```vagrant up```

## Проверка работы

### Создание БД:

    ```
    vagrant ssh master
    sudo su postgres
    psql
    CREATE DATABASE "testDB";
    \c testDB
    CREATE TABLE users (id serial PRIMARY KEY, name VARCHAR (255) UNIQUE NOT NULL);
    INSERT INTO users (name) values ('testname');
    ```

1. Master
Поначалу в лог файле ```/var/lib/pgsql/11/data/log/postgresql-*.log``` могут быть ошибки, но они пройдут после развертывания серверов.

```
[vagrant@master ~]$ sudo su
[root@master vagrant]# less /var/lib/pgsql/11/data/log/postgresql-Thu.log
[root@master vagrant]# sudo su postgres
bash-4.2$ psql
could not change directory to "/home/vagrant": Permission denied
psql (11.10)
Type "help" for help.
postgres=# \d
Did not find any relations.
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 testDB    | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
(4 rows)

postgres=#
postgres=# CREATE DATABASE myDB;
CREATE DATABASE
postgres=#

```

2. переключимся на сервер Slave
```
[vagrant@slave ~]$ sudo su
[root@slave vagrant]# sudo su postgres
bash-4.2$ psql
could not change directory to "/home/vagrant": Permission denied
psql (11.10)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 mydb      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 testDB    | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
(5 rows)

postgres=# quit


[root@slave vagrant]# tail -f /var/lib/pgsql/11/data/log/postgresql-Thu.log
2021-01-07 13:10:08.199 UTC [5370] LOG:  database system was shut down at 2021-01-07 13:10:05 UTC
2021-01-07 13:10:08.202 UTC [5367] LOG:  database system is ready to accept connections
2021-01-07 13:11:15.330 UTC [4225] LOG:  database system was interrupted; last known up at 2021-01-07 13:11:13 UTC
2021-01-07 13:11:15.394 UTC [4225] LOG:  entering standby mode
2021-01-07 13:11:15.396 UTC [4225] LOG:  redo starts at 0/2000028
2021-01-07 13:11:15.397 UTC [4225] LOG:  consistent recovery state reached at 0/2000130
2021-01-07 13:11:15.397 UTC [4222] LOG:  database system is ready to accept read only connections
2021-01-07 13:11:15.415 UTC [4229] LOG:  started streaming WAL from primary at 0/3000000 on timeline 1

```

3. Backup

```
[vagrant@backup ~]$ sudo su
[root@backup vagrant]# barman switch-xlog --force --archive master
The WAL file 000000010000000000000003 has been closed on server 'master'
Waiting for the WAL file 000000010000000000000003 from server 'master' (max: 30 seconds)
Processing xlog segments from file archival for master
	000000010000000000000003
[root@backup vagrant]# barman check master
Server master:
	PostgreSQL: OK
	superuser or standard user with backup privileges: OK
	PostgreSQL streaming: OK
	wal_level: OK
	replication slot: OK
	directories: OK
	retention policy settings: OK
	backup maximum age: OK (no last_backup_maximum_age provided)
	compression settings: OK
	failed backups: OK (there are 0 failed backups)
	minimum redundancy requirements: OK (have 0 backups, expected at least 0)
	pg_basebackup: OK
	pg_basebackup compatible: OK
	pg_basebackup supports tablespaces mapping: OK
	systemid coherence: OK (no system Id stored on disk)
	pg_receivexlog: OK
	pg_receivexlog compatible: OK
	receive-wal running: OK
	archive_mode: OK
	archive_command: OK
	continuous archiving: OK
	archiver errors: OK
[root@backup vagrant]# barman backup master
Starting backup using postgres method for server master in /var/lib/barman/master/base/20210107T144438
Backup start at LSN: 0/4000060 (000000010000000000000004, 00000060)
Starting backup copy via pg_basebackup for 20210107T144438
Copy done (time: 2 seconds)
Finalising the backup.
This is the first backup for server master
WAL segments preceding the current backup have been found:
	000000010000000000000001 from server master has been removed
	000000010000000000000002 from server master has been removed
	000000010000000000000002.00000028.backup from server master has been removed
	000000010000000000000003 from server master has been removed
Backup size: 37.5 MiB
Backup end at LSN: 0/6000060 (000000010000000000000006, 00000060)
Backup completed (start time: 2021-01-07 14:44:38.031806, elapsed time: 3 seconds)
Processing xlog segments from streaming for master
	000000010000000000000003
	000000010000000000000004
	000000010000000000000005
Processing xlog segments from file archival for master
	000000010000000000000004
	000000010000000000000005
	000000010000000000000005.00000028.backup
	000000010000000000000006
[root@backup vagrant]# barman check master
Server master:
	PostgreSQL: OK
	superuser or standard user with backup privileges: OK
	PostgreSQL streaming: OK
	wal_level: OK
	replication slot: OK
	directories: OK
	retention policy settings: OK
	backup maximum age: OK (no last_backup_maximum_age provided)
	compression settings: OK
	failed backups: OK (there are 0 failed backups)
	minimum redundancy requirements: OK (have 1 backups, expected at least 0)
	pg_basebackup: OK
	pg_basebackup compatible: OK
	pg_basebackup supports tablespaces mapping: OK
	systemid coherence: OK
	pg_receivexlog: OK
	pg_receivexlog compatible: OK
	receive-wal running: OK
	archive_mode: OK
	archive_command: OK
	continuous archiving: OK
	archiver errors: OK
	[root@backup vagrant]# barman status master
	Server master:
		Description: PostgreSQL Backup
		Active: True
		Disabled: False
		PostgreSQL version: 11.10
		Cluster state: in production
		pgespresso extension: Not available
		Current data size: 37.8 MiB
		PostgreSQL Data directory: /var/lib/pgsql/11/data
		Current WAL segment: 000000010000000000000007
		PostgreSQL 'archive_command' setting: barman-wal-archive backup master %p
		Last archived WAL: 000000010000000000000006, at Thu Jan  7 14:44:41 2021
		Failures of WAL archiver: 9 (000000010000000000000001 at Thu Jan  7 14:25:04 2021)
		Server WAL archiving rate: 20.32/hour
		Passive node: False
		Retention policies: not enforced
		No. of available backups: 1
		First available backup: 20210107T144438
		Last available backup: 20210107T144438
		Minimum redundancy requirements: satisfied (1/0)
		[root@backup vagrant]# barman backup master
	Starting backup using postgres method for server master in /var/lib/barman/master/base/20210107T144728
	Backup start at LSN: 0/70000C8 (000000010000000000000007, 000000C8)
	Starting backup copy via pg_basebackup for 20210107T144728
	Copy done (time: 2 seconds)
	Finalising the backup.
	Backup size: 37.5 MiB
	Backup end at LSN: 0/9000000 (000000010000000000000008, 00000000)
	Backup completed (start time: 2021-01-07 14:47:28.960188, elapsed time: 2 seconds)
	Processing xlog segments from streaming for master
		000000010000000000000007
	Processing xlog segments from file archival for master
		000000010000000000000007
		000000010000000000000008
		000000010000000000000008.00000028.backup
	[root@backup vagrant]# barman replication-status master
	Status of streaming clients for server 'master':
	  Current LSN on master: 0/90000C8
	  Number of streaming clients: 2

	  1. Async standby
	     Application name: walreceiver
	     Sync stage      : 5/5 Hot standby (max)
	     Communication   : TCP/IP
	     IP Address      : 192.168.1.20 / Port: 35412 / Host: -
	     User name       : streaming_user
	     Current state   : streaming (async)
	     Replication slot: standby_slot
	     WAL sender PID  : 5766
	     Started at      : 2021-01-07 14:23:46.141096+00:00
	     Sent LSN   : 0/90000C8 (diff: 0 B)
	     Write LSN  : 0/90000C8 (diff: 0 B)
	     Flush LSN  : 0/90000C8 (diff: 0 B)
	     Replay LSN : 0/90000C8 (diff: 0 B)

	  2. Async WAL streamer
	     Application name: barman_receive_wal
	     Sync stage      : 3/3 Remote write
	     Communication   : TCP/IP
	     IP Address      : 192.168.1.30 / Port: 45796 / Host: -
	     User name       : barman_streaming_user
	     Current state   : streaming (async)
	     Replication slot: barman
	     WAL sender PID  : 5792
	     Started at      : 2021-01-07 14:26:01.985063+00:00
	     Sent LSN   : 0/90000C8 (diff: 0 B)
	     Write LSN  : 0/90000C8 (diff: 0 B)
	     Flush LSN  : 0/9000000 (diff: -200 B)

```
