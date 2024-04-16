# Домашнее задание № 26 по теме: "Postgres: Backup + Репликация". К курсу Administrator Linux. Professional

## Задание

- Настроить hot\_standby репликацию с использованием слотов
- Настроить правильное резервное копирование

## Выполнение

### 1. Настройка hot\_standby репликации с использованием слотов

Для проверки после разворачивания стенда выполнить:

- Список существующих баз данных на node1:
```bash
vagrant ssh node1 -c "sudo -u postgres psql -c '\l'"
```

```
                                                       List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | ICU Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+-------------+-------------+------------+-----------+-----------------------
 postgres  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |
 template0 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
(3 rows)
```

- Список существующих баз данных на node2:
```bash
vagrant ssh node2 -c "sudo -u postgres psql -c '\l'"
```

```
                                                       List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | ICU Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+-------------+-------------+------------+-----------+-----------------------
 postgres  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |
 template0 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
(3 rows)
```

- Создать на master (node1) базу данных **otus_test**
```bash
vagrant ssh node1 -c "sudo -u postgres psql -c 'CREATE DATABASE otus_test'"
```

- Проверить реплику (node2)
```bash
vagrant ssh node2 -c "sudo -u postgres psql -c '\l'"
```

```
                                                       List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | ICU Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+-------------+-------------+------------+-----------+-----------------------
 otus_test | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |
 postgres  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |
 template0 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
(4 rows)
```

*Убедились что репликация настроена верно - база данных присутствует на обоих серверах*


### 2. Настройка резервного копирования

- Для проверки нстройки бэкапа необходимо выполнить под пользователем barman:

```bash
vagrant ssh barman -c "sudo -u barman barman switch-wal node1"
```
```
The WAL file 000000010000000000000004 has been closed on server 'node1'
```

```bash
vagrant ssh barman -c "sudo -u barman barman cron"
```
```
Starting WAL archiving for server node1
```

```bash
vagrant ssh barman -c "sudo -u barman barman check node1"
```
```
Server node1:
        PostgreSQL: OK
        superuser or standard user with backup privileges: OK
        PostgreSQL streaming: OK
        wal_level: OK
        replication slot: OK
        directories: OK
        retention policy settings: OK
        backup maximum age: FAILED (interval provided: 4 days, latest backup age: No available backups)
        backup minimum size: OK (0 B)
        wal maximum age: OK (no last_wal_maximum_age provided)
        wal size: OK (0 B)
        compression settings: OK
        failed backups: OK (there are 0 failed backups)
        minimum redundancy requirements: FAILED (have 0 backups, expected at least 1)
        pg_basebackup: OK
        pg_basebackup compatible: OK
        pg_basebackup supports tablespaces mapping: OK
        systemid coherence: OK (no system Id stored on disk)
        pg_receivexlog: OK
        pg_receivexlog compatible: OK
        receive-wal running: OK
        archiver errors: OK
```

- Запустить процесс резервного копирования

```bash
vagrant ssh barman -c "sudo -u barman barman backup node1"
```
```
Starting backup using postgres method for server node1 in /var/lib/barman/node1/base/20240416T235430
Backup start at LSN: 0/5000148 (000000010000000000000005, 00000148)
Starting backup copy via pg_basebackup for 20240416T235430
Copy done (time: less than one second)
Finalising the backup.
This is the first backup for server node1
WAL segments preceding the current backup have been found:
        000000010000000000000004 from server node1 has been removed
Backup size: 29.5 MiB
Backup end at LSN: 0/7000000 (000000010000000000000006, 00000000)
Backup completed (start time: 2024-04-16 23:54:30.575691, elapsed time: less than one second)
Processing xlog segments from streaming for node1
        000000010000000000000005
        000000010000000000000006
```

- Для проверки восстановления из резервной копии удалить базу данных **otus**
```bash
vagrant ssh node1 -c "sudo -u postgres psql -c 'DROP DATABASE otus;'"
```

- Просмотреть имеющиеся резервные копии
```bash
vagrant ssh barman -c "sudo -u barman barman list-backup node1"
```
```
node1 20240416T235430 - Tue Apr 16 23:54:31 2024 - Size: 29.5 MiB - WAL Size: 0 B
```

- Остановить postgresql на node1
```bash
vagrant ssh node1 -c "sudo systemctl stop postgresql-16"
```

- Восстановить из резервной копии:
```bash
vagrant ssh barman -c "sudo -u barman barman recover node1 20240416T235430 /var/lib/pgsql/16/data/ --remote-ssh-comman 'ssh postgres@192.168.56.11'"
```
```
Starting remote restore for server node1 using backup 20240416T235430
Destination directory: /var/lib/pgsql/16/data/
Remote command: ssh postgres@192.168.56.11
Using safe horizon time for smart rsync copy: 2024-04-16 23:54:30.576969+03:00
Copying the base backup.
Copying required WAL segments.
Generating archive status files
Identify dangerous settings in destination directory.

Recovery completed (start time: 2024-04-17 00:16:15.828065+03:00, elapsed time: 4 seconds)
Your PostgreSQL server has been successfully prepared for recovery!
```

- Запустить PostgreSQL
```bash
vagrant ssh node1 -c "sudo systemctl start postgresql-16"
```

- Убедиться, что удаленная база присутствует
```bash
vagrant ssh node1 -c "sudo -u postgres psql -c '\l'"
```
```
                                                       List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | ICU Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+-------------+-------------+------------+-----------+-----------------------
 otus      | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |
 postgres  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |
 template0 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
           |          |          |                 |             |             |            |           | postgres=CTc/postgres
(4 rows)
```
