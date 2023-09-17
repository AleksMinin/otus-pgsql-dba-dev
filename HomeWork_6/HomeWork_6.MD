#Домашнее задание
##Работа с журналами

**Цель:**
 - уметь работать с журналами и контрольными точками
 - уметь настраивать параметры журналов

Описание/Пошаговая инструкция выполнения домашнего задания:
1. Настройте выполнение контрольной точки раз в 30 секунд.
```sql
test=# ALTER SYSTEM set checkpoint_timeout = '30s';
ALTER SYSTEM
test=#\q
aminin@aminin-VirtualBox:~$ sudo pg_ctlcluster 15 main restart;
aminin@aminin-VirtualBox:~$ psql -h localhost -p5432 -U postgres -d test
test=#
test=# select setting || ' x ' || coalesce(unit, 'units')
from pg_settings
where name = 'checkpoint_timeout';
 ?column? 
----------
 30 x s
(1 строка)

SELECT name, setting, unit 
  FROM pg_settings WHERE name in .....
             name             | setting | unit 
------------------------------+---------+------
 checkpoint_completion_target | 0.9     | 
 checkpoint_timeout           | 30      | s
 default_statistics_target    | 500     | 
 effective_cache_size         | 393216  | 8kB
 effective_io_concurrency     | 2       | 
 maintenance_work_mem         | 524288  | kB
 max_connections              | 40      | 
 max_wal_size                 | 16384   | MB
 min_wal_size                 | 4096    | MB
 random_page_cost             | 4       | 
 shared_buffers               | 131072  | 8kB
 wal_buffers                  | 2048    | 8kB
 work_mem                     | 6553    | kB


```
2. 10 минут c помощью утилиты pgbench подавайте нагрузку.
	- Сначала сброшу ранее накопленную статистику командами pg_stat_reset() и pg_stat_reset_shared('bgwriter') и после запущу тест на 10 минут

	
```sql
test=# select pg_stat_reset();SELECT pg_stat_reset_shared('bgwriter');
 pg_stat_reset 
---------------
(1 строка)

 pg_stat_reset_shared 
----------------------
(1 строка)
test=#\q

aminin@aminin-VirtualBox:~$ pgbench -T 600 -h localhost -U postgres test
Password: 
pgbench (15.4 (Ubuntu 15.4-2.pgdg23.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 226599
number of failed transactions: 0 (0.000%)
latency average = 2.648 ms
initial connection time = 20.273 ms
tps = 377.677499 (without initial connection time)
```

3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
```sql
test=# SELECT sum(size)/1024/1024 as sise_MB, 
              count(*) as cnt_files 
         FROM pg_ls_waldir()
		WHERE date_trunc('minute', modification) between timestamp '2023-09-17 02:38:00'
									and timestamp '2023-09-17 02:53:00';
       sise_mb       | cnt_files 
---------------------+-----------
 80.0000000000000000 |         5
(1 строка)

```

- 80mb/20(где 20 = 600сек общего времени теста/ 30сек время между контрольными точками) =  4mb дискового пространства за контрольную точку

4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
```sql
test=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 20
checkpoints_req       | 0
checkpoint_write_time | 512185
checkpoint_sync_time  | 69
buffers_checkpoint    | 35120
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 1462
buffers_backend_fsync | 0
buffers_alloc         | 1453
stats_reset           | 2023-09-16 12:21:18.377871+05
```
- SELECT * FROM pg_stat_bgwriter выполнил сразу после теста, чтобы не успели сработать следующие котрольные точки. Выполнено 20 контрольных точек, как и предполагалось. Попробовал перезапустить тест с количеством коннектов = 30 pgbench -c30 -T 600 -h localhost -U postgres test, но результат остался тем же.
Но в задании, вероятно, предполагалось, что контрольных точек должно было пройти меньше, предположу, что это должно быть связано с временем на синхронизацию и запись.
Первые запуски тестов были не очень чистыми и было обозначено, что последний сброс статистики был довольно давно, как делать тесты чище помогла статья [Статистика работы PostgreSQL](https://sysadminium.ru/statistika_raboty_postgresql/)

5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
```sql
test=#show synchronous_commit;

 synchronous_commit 
--------------------
 on
(1 строка)

minin@aminin-VirtualBox:~$ pgbench -c30 -T 15 -h localhost -U postgres test
Password: 
pgbench (15.4 (Ubuntu 15.4-2.pgdg23.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 30
number of threads: 1
maximum number of tries: 1
duration: 15 s
number of transactions actually processed: 13587
number of failed transactions: 0 (0.000%)
latency average = 32.407 ms
initial connection time = 383.655 ms
tps = 925.726951 (without initial connection time)
```
- далее отключаю синхронное выполнение, работа будет выполняться в асинхронном режиме, не будет ожидания локального сброса WAL на диск.
```sql
test=# ALTER SYSTEM SET synchronous_commit = off;
ALTER SYSTEM
test=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 строка)


minin@aminin-VirtualBox:~$ pgbench -c30 -T 15 -h localhost -U postgres test
Password: 
pgbench (15.4 (Ubuntu 15.4-2.pgdg23.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 30
number of threads: 1
maximum number of tries: 1
duration: 15 s
number of transactions actually processed: 19315
number of failed transactions: 0 (0.000%)
latency average = 22.793 ms
initial connection time = 394.353 ms
tps = 1316.186856 (without initial connection time)

```
- Количество tps увеличилось. Но появился риск потери последних транзакций при сбое сервера.

6. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. 
```sql
aminin@aminin-VirtualBox:~$ sudo pg_createcluster 15 test -- --data-checksums
Creating new PostgreSQL cluster 15/test ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/test --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
Файлы, относящиеся к этой СУБД, будут принадлежать пользователю "postgres".
От его имени также будет запускаться процесс сервера.

Кластер баз данных будет инициализирован с локалью "ru_RU.UTF-8".
Кодировка БД по умолчанию, выбранная в соответствии с настройками: "UTF8".
Выбрана конфигурация текстового поиска по умолчанию "russian".

Контроль целостности страниц данных включён.


minin@aminin-VirtualBox:~$ sudo -u postgres psql -p 5433
could not change directory to "/home/aminin": Отказано в доступе
psql (15.4 (Ubuntu 15.4-2.pgdg23.04+1))
Type "help" for help.

postgres=# \password postgres
Enter new password for user "postgres": 
Enter it again: 
postgres=# create database test5433;
CREATE DATABASE
postgres=#
postgres=# \c test5433
You are now connected to database "test5433" as user "postgres".
test5433=# create table t(cval char(10));
CREATE TABLE
test5433=#
test5433=# insert into t select 'Hello!!!' from generate_series(1, 100);
INSERT 0 100
test5433=# SELECT pg_relation_filepath('t');
 pg_relation_filepath 
----------------------
 base/16388/16389
(1 строка)
test5433=#\q
aminin@aminin-VirtualBox:~$ sudo pg_ctlcluster 15 test stop;

aminin@aminin-VirtualBox:~$ sudo dd if=/dev/zero of=/var/lib/postgresql/15/test/base/16388/16389 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0,0130264 s, 0,6 kB/s

```

7. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?
```sql
aminin@aminin-VirtualBox:~$ sudo pg_ctlcluster 15 test start;
aminin@aminin-VirtualBox:~$ psql -h localhost -p 5433 -U postgres -d test5433
Пароль пользователя postgres: 
psql (15.4 (Ubuntu 15.4-2.pgdg23.04+1))
SSL-соединение (протокол: TLSv1.3, шифр: TLS_AES_256_GCM_SHA384, сжатие: выкл.)
Введите "help", чтобы получить справку.

test5433=# select * from t;
ПРЕДУПРЕЖДЕНИЕ:  ошибка проверки страницы: получена контрольная сумма 39139, а ожидалась - 43884
ОШИБКА:  неверная страница в блоке 0 отношения base/16388/16389
test5433=# 
```
- Включу игнор ошибки контрольной суммы
```sqltest5433=# SHOW ignore_checksum_failure;
 ignore_checksum_failure 
-------------------------
 off
(1 строка)

test5433=# ALTER SYSTEM SET ignore_checksum_failure=on;
ALTER SYSTEM
test5433=# select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 строка)

test5433=# SHOW ignore_checksum_failure;
 ignore_checksum_failure 
-------------------------
 on
(1 строка)

test5433=# select * from t;
ПРЕДУПРЕЖДЕНИЕ:  ошибка проверки страницы: получена контрольная сумма 39139, а ожидалась - 43884
test5433=# select * from t limit 1;
    cval    
------------
 Hello!!!  
(1 строка)
```