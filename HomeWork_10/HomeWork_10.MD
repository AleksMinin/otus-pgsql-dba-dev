# Домашнее задание
## Репликация

**Цель: реализовать свой миникластер на 3 ВМ.**


Описание/Пошаговая инструкция выполнения домашнего задания:
	*Во всех БД будет создаваться роль для использования в репликациях и права по заданиям ДЗ буду давать именно этой роли, как и настройку репликаций в pg_hba.conf (аналогичная настройка во всех БД ДЗ)
```sql
# TYPE  DATABASE        USER            ADDRESS     METHOD

local   all             postgres                    peer
# "local" is for Unix domain socket connections only
local   all             all                         scram-sha-256
# IPv4 local connections:
host    test            urepl     0.0.0.0/0         scram-sha-256
host    all             all       127.0.0.1/32      scram-sha-256
host    all             all       localhost      	scram-sha-256
# IPv6 local connections:
#host   all             all       ::1/128           scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                         peer
host    replication     urepl     0.0.0.0/0         scram-sha-256
#host   replication     all       ::1/128           scram-sha-256
```
* Создание роли для всех БД ДЗ
```sql
create role urepl WITH PASSWORD '123urepl' LOGIN REPLICATION;
```
1. На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение.
```sql
--Подключение к БД на ВМ1
ubuntu@vm-ubuntu:~$ psql -hlocalhost -p5432 -Upostgres -dtest

-- Создание роли urepl для использования в репликациях
test=#create role urepl WITH PASSWORD '123urepl' LOGIN REPLICATION;

-- Создание таблицы test
create table test as (select generate_series(1,10)::integer as nid);
test=# grant insert,update,delete,select on test to urepl;

-- Создание таблицы test2
test=# create table test2 (nid integer);
CREATE TABLE

-- Права на чтение таблицы test2 для роли urepl
test=# grant select on table test2 to urepl;
GRANT

-- Проверка прав доступа
test=# \dp
                                 Access privileges
 Schema | Name  | Type  |     Access privileges     | Column privileges | Policies
--------+-------+-------+---------------------------+-------------------+----------
 public | test  | table | postgres=arwdDxt/postgres+|                   |
        |       |       | urepl=awd/postgres        |                   |
 public | test2 | table | postgres=arwdDxt/postgres+|                   |
        |       |       | urepl=r/postgres          |                   |
(2 rows)

```
Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.
```sql
-- Переключение на режим логической репликации
test=# alter system set wal_level = logical;
ALTER SYSTEM
test=#\q

-- Перезапуск кластера для применения изменений
ubuntu@vm-ubuntu:~$ sudo pg_ctlcluster 15 main restart

--Подключение к БД на ВМ1
ubuntu@vm-ubuntu:~$ psql -hlocalhost -p5432 -Upostgres -dtes

--Создание публикации для таблицы test на ВМ1
test=# CREATE PUBLICATION test_pubvm1 FOR TABLE test;
CREATE PUBLICATION
test=# \dRp+
                          Publication test_pubvm1
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test"
test=#

-- Создание подписки для таблицы test2 ВМ2
CREATE SUBSCRIPTION test2_pubvm2 CONNECTION 'host=158.160.85.87 port=5432 user=urepl password=123urepl dbname=test' PUBLICATION test2_pubvm2 WITH (copy_data = true);

test=# \dRs
               List of subscriptions
     Name     |  Owner   | Enabled |  Publication
--------------+----------+---------+----------------
 test2_pubvm2 | postgres | t       | {test2_pubvm2}
(1 row)
```

2. На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение.
```sql
-- Создание роли urepl для использования в репликациях
test=#create role urepl WITH PASSWORD '123urepl' LOGIN REPLICATION;

-- Cоздание таблицы test на ВМ2
test=# create table test (nid integer);
CREATE TABLE

-- Создание таблицы с test2 на ВМ2
test=# create table test2 as (select generate_series(51,60)::integer as nid);
SELECT 10

-- Дать права на чтение в таблице test на ВМ2 для пользователя urepl
test=# grant select on test to urepl;
GRANT

-- Дать права на запись в таблице test2 на ВМ2 для пользователя urepl
test=# grant insert,update,delete,select on test2 to urepl;
GRANT

test=# grant select on table test2 to urepl;
GRANT
test=# \dp
                                 Access privileges
 Schema | Name  | Type  |     Access privileges     | Column privileges | Policies
--------+-------+-------+---------------------------+-------------------+----------
 public | test  | table | urepl=r/postgres         +|                   |
        |       |       | postgres=arwdDxt/postgres |                   |
 public | test2 | table | urepl=arwd/postgres      +|                   |
        |       |       | postgres=arwdDxt/postgres |                   |
(2 rows)
```

Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.
```sql
-- Переключение на режим логической репликации
test=# alter system set wal_level = logical;
ALTER SYSTEM
test=#\q

-- Перезапуск кластера для применения изменений
ubuntu@vm-ubuntu2:~$ sudo pg_ctlcluster 15 main restart

-- Подключение к БД test на ВМ2
ubuntu@vm-ubuntu2:~$ psql -hlocalhost -p5432 -Upostgres -dtest

--Создание публикации для таблицы test на ВМ1
test=# CREATE PUBLICATION test2_pubvm2 FOR TABLE test2;
CREATE PUBLICATION

test=# \dRp+
                          Publication test2_pubvm2
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test2"
	
-- Создание подписки для таблицы test ВМ1
CREATE SUBSCRIPTION test_pubvm1 CONNECTION 'host=158.160.81.138 port=5432 user=urepl password=123urepl dbname=test' PUBLICATION test_pubvm1 WITH (copy_data = true);

test=# \dRs
              List of subscriptions
    Name     |  Owner   | Enabled |  Publication
-------------+----------+---------+---------------
 test_pubvm1 | postgres | t       | {test_pubvm1}
(1 row)	
```

3. 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).
```sql
--Подключение к БД на ВМ3
ubuntu@vm-ubuntu3:~$ psql -hlocalhost -p5432 -Upostgres -dtest

-- Создание роли urepl для использования в репликациях
test=#create role urepl WITH PASSWORD '123urepl' LOGIN REPLICATION;

-- Создание таблицы test на ВМ3
test=# create table test (nid integer);

-- Создание таблицы test2 на ВМ3
test=# create table test2 (nid integer);
CREATE TABLE

-- Права на чтение таблицы test и test2 для роли urepl
test=# grant select on table test,test2 to urepl;
GRANT

-- Проверка прав доступа
test=# \dp
                                 Access privileges
 Schema | Name  | Type  |     Access privileges     | Column privileges | Policies
--------+-------+-------+---------------------------+-------------------+----------
 public | test  | table | postgres=arwdDxt/postgres+|                   |
        |       |       | urepl=r/postgres          |                   |
 public | test2 | table | postgres=arwdDxt/postgres+|                   |
        |       |       | urepl=r/postgres          |                   |
(2 rows)

```
Подписываемся на публикацию таблицы test с ВМ №1 и на на публикацию таблицы test2 с ВМ №2
```sql
-- Переключение на режим логической репликации
test=# alter system set wal_level = logical;
ALTER SYSTEM
test=#\q

-- Перезапуск кластера для применения изменений
ubuntu@vm-ubuntu3:~$ sudo pg_ctlcluster 15 main restart

--Подключение к БД на ВМ1
ubuntu@vm-ubuntu3:~$ psql -hlocalhost -p5432 -Upostgres -dtest

-- Создание подписок для таблицы test ВМ1 и таблицы test2 ВМ2
-- Создание подписки для таблицы test ВМ1
CREATE SUBSCRIPTION test_pubvm11 CONNECTION 'host=158.160.81.138 port=5432 user=urepl password=123urepl dbname=test' PUBLICATION test_pubvm1 WITH (copy_data = true);

-- Создание подписки для таблицы test2 ВМ2
CREATE SUBSCRIPTION test2_pubvm22 CONNECTION 'host=158.160.85.87 port=5432 user=urepl password=123urepl dbname=test' PUBLICATION test2_pubvm2 WITH (copy_data = true);
```

ДЗ сдается в виде миниотчета на гитхабе с описанием шагов
* __С какими проблемами столкнулись?__
* При создании публикаций на таблицы соседних ВМ указал WITH (copy_data = true), но возникала ошибка, не копировало уже \
  существующие значения, пока не добавил право SELECT тем таблицам, которым нужно ранее было назначить права на запись, а именно
```sql
-- На ВМ1
-- Дать права на запись в таблице test на ВМ1 для пользователя urepl
test=# grant insert,update,delete,select on test to urepl;

-- На ВМ2
-- Дать права на запись в таблице test2 на ВМ2 для пользователя urepl
test=# grant insert,update,delete,select on test2 to urepl;
```	
* Также столкнулся с ошибкой при создании подписки для ВМ3, сначала писал CREATE SUBSCRIPTION test_pubvm1, выходила ошибка о уже существующем сокете с таким именем,
сначала не понял, почитал разобрался, создавал уже с другим именем CREATE SUBSCRIPTION test_pubvm11, т.к. test_pubvm1 уже использовала ВМ2


4. реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. 
* На ВМ3 укажу в параметрах postgressql.conf
```sql
listen_addresses = 'localhost, 84.201.143.194'
```
* В pg_hba.conf необходимые параметры
```sql
local   all             postgres                                peer

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                  scram-sha-256
# IPv4 local connection
host    all             all             84.201.143.194/32    scram-sha-256
host    all             all             localhost            scram-sha-256
host    all             all             127.0.0.1/32         scram-sha-256

# IPv6 local connections:
#host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.x
local   replication     all                                     peer
host    replication     urepl            84.201.143.194/32       scram-sha-256
#host    replication     all             ::1/128                 scram-sha-256
```
* Далее проверю и при необходимости переключусь на режим физической репликации
```sql
-- Переключение на режим физической репликации
test=# alter system set wal_level = replica;
ALTER SYSTEM
test=#\q
ubuntu@vm-ubuntu3:~$ sudo pg_ctlcluster 15 main restart
```
* На ВМ4 создаю пользователя для репликаций как и ранее создавал для репликаций на прошлых ВМ
```sql
-- Создание роли urepl для использования в репликациях
postgres=#create role urepl WITH PASSWORD '123urepl' LOGIN REPLICATION;

-- Остановлю кластер на ВМ4 и удалю файлы данных
ubuntu@vm-ubuntu4:~$ sudo pg_ctlcluster 15 main restart
sudo rm -rf /var/lib/postgresql/15/main

-- Сделаю бэкап БД с ВМ3. Ключ -R создаст заготовку управляющего файла recovery.conf 
ubuntu@vm-ubuntu4:~$ sudo -u postgres pg_basebackup -R -h 158.160.69.208 -p 5432 -U urepl -D /var/lib/postgresql/15/main -P
Password:
30677/30677 kB (100%), 1/1 tablespace
```
* На ВМ1 добавляю значение в таблицу test, оно должно появиться на ВМ3 и c ВМ3 скопироваться на ВМ4
```sql
--добавляю запись на ВМ1
ubuntu@vm-ubuntu:~$ psql -hlocalhost -p5432 -Upostgres -dtest
Password for user postgres:
psql (16.0 (Ubuntu 16.0-1.pgdg22.04+1), server 15.4 (Ubuntu 15.4-2.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

test=# insert into test values(53);
INSERT 0 1
```
* Проверю на ВМ4
```sql
ubuntu@vm-ubuntu4:~$ psql -h localhost -p5432 -d test -Upostgres
Password for user postgres:
psql (15.4 (Ubuntu 15.4-2.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

test=# select * from test order by nid desc limit 5;
 nid
-----
  53
  52
  51
  50
  15
(5 rows)
```

* __Написать с какими проблемами столкнулись.__
* Долго разбирался с pg_hba.conf c настройкой для репликаций, думал, что имя БД replication нужно менять на имя своей БД, оказалось, не нужно менять.
* Не получалось корректно забить параметры подключения для создания резервной копии при началоной настройке кластера на ВМ4, почитал, кстати, подсмотрел в [статье ОТУСа на Хабре](https://habr.com/ru/companies/otus/articles/710956/)
* Просидел с этим заданием прям достаточно прилично, теперь можно и за patroni браться :)

Критерии оценки:
Выполнение ДЗ: 10 баллов

5 баллов за задание со *
плюс 2 балл за красивое решение
минус 2 балл за рабочее решение, и недостатки указанные преподавателем не устранены
