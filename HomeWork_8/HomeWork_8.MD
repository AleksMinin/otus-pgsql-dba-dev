# Домашнее задание №8
## Нагрузочное тестирование и тюнинг PostgreSQL

**Цель:**
сделать нагрузочное тестирование PostgreSQL
настроить параметры PostgreSQL для достижения максимальной производительности

Описание/Пошаговая инструкция выполнения домашнего задания:
1. развернуть виртуальную машину любым удобным способом
2. поставить на неё PostgreSQL 15 любым способом
3. настроить кластер PostgreSQL 15 на максимальную производительность не
обращая внимание на возможные проблемы с надежностью в случае
аварийной перезагрузки виртуальной машины
  * Тестовый стенд VM VirtualBox: CPU 2core single core 4.3Ghz Ryzen 3700x, RAM 4Gb ddr4 3200, SSD_NVME 80GB(RW~3300mb/s)
  * Настройки буду менять в AUTO.CONF. Сначала укажу только максимальное кол-во подключений 30 и прогоню тест на проверку производительности при настройках PG по умолчанию (чтобы было с чем сравнивать далее). Тесты буду запускать на максимально допустимом кол-ве сессиий = 27, т.к. еще три резервирует PG

4. нагрузить кластер через утилиту через утилиту pgbench (https://postgrespro.ru/docs/postgrespro/14/pgbench)
5. написать какого значения tps удалось достичь, показать какие параметры в
какие значения устанавливали и почему
* Буду прогонять тесты, менять настройки параметров, снова прогонять тесты, и попутно буду комментировать свои действия.
  
```sql
test=# alter system set max_connections = 30;
test=#\q
root@aminin-VirtualBox:/etc/postgresql/15/main# sudo pg_ctlcluster 15 main restart;
root@aminin-VirtualBox:/etc/postgresql/15/main# sudo psql -h localhost -p5432 -U postgres -d test
Пароль пользователя postgres: 
psql (15.4 (Ubuntu 15.4-2.pgdg23.04+1))
SSL-соединение (протокол: TLSv1.3, шифр: TLS_AES_256_GCM_SHA384, сжатие: выкл.)
Введите "help", чтобы получить справку.

test=# show max_connections;
max_connections 
-----------------
30
(1 строка)

root@aminin-VirtualBox:/etc/postgresql/15/main# sudo pgbench -i -h localhost -p 5432 -d test -U postgres 
Password: 
dropping old tables...
ЗАМЕЧАНИЕ:  таблица "pgbench_accounts" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_branches" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_history" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_tellers" не существует, пропускается
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.07 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.27 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.15 s, vacuum 0.04 s, primary keys 0.07 s).
root@aminin-VirtualBox:/etc/postgresql/15/main# pgbench -c27 -j 2 -P 20 -T 60 -h localhost -U postgres test
Password: 
pgbench (15.4 (Ubuntu 15.4-2.pgdg23.04+1))
starting vacuum...end.
progress: 20.0 s, 998.6 tps, lat 26.714 ms stddev 18.618, 0 failed
progress: 40.0 s, 1026.1 tps, lat 26.301 ms stddev 18.190, 0 failed
progress: 60.0 s, 1041.5 tps, lat 25.930 ms stddev 19.183, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 27
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 61352
number of failed transactions: 0 (0.000%)
latency average = 26.317 ms
latency stddev = 18.681 ms
initial connection time = 214.532 ms
tps = 1024.449190 (without initial connection time)
```
  *  Теперь воспользуюсь калькулятором распеределения ресурсов PGTUNE для приложения со смешанным режимом использования, чтобы распределить ресурсы по общим первоначальным настройкам кластера PG, основываясь на [Описание по распределению ресурсов сервера](https://www.postgrespro.ru/docs/postgresql/15/runtime-config-resource), этого инструмента достаточно для начала работ с пустой БД, чтобы не тратить время на подсчет значений параметров вручную по рекомендуемым формулам/соотношениям, а с уже реальными данными и реальными нагрузками можно менять параметры использования ресурсов сервера для достижения максимально возможной производительности.
  ```sql
  root@aminin-VirtualBox:/etc/postgresql/15/main# sudo psql -h localhost -p5432 -U postgres
   Пароль пользователя postgres: 
   psql (15.4 (Ubuntu 15.4-2.pgdg23.04+1))
   SSL-соединение (протокол: TLSv1.3, шифр: TLS_AES_256_GCM_SHA384, сжатие: выкл.)
   Введите "help", чтобы получить справку.

   postgres=# ALTER SYSTEM SET
   max_connections = '30';
   ALTER SYSTEM SET
   shared_buffers = '1GB';
   ALTER SYSTEM SET
   effective_cache_size = '3GB';
   ALTER SYSTEM SET
   maintenance_work_mem = '256MB';
   ALTER SYSTEM SET
   checkpoint_completion_target = '0.9';
   ALTER SYSTEM SET
   wal_buffers = '16MB';
   ALTER SYSTEM SET
   default_statistics_target = '100';
   ALTER SYSTEM SET
   random_page_cost = '1.1';
   ALTER SYSTEM SET
   effective_io_concurrency = '200';
   ALTER SYSTEM SET
   work_mem = '8738kB';
   ALTER SYSTEM SET
   huge_pages = 'off';
   ALTER SYSTEM SET
   min_wal_size = '1GB';
   ALTER SYSTEM SET
   max_wal_size = '4GB';
   ALTER SYSTEM
   ALTER SYSTEM
   ALTER SYSTEM
   ALTER SYSTEM
   ALTER SYSTEM
   ALTER SYSTEM
   ALTER SYSTEM
   ALTER SYSTEM
   ALTER SYSTEM
   ALTER SYSTEM
   ALTER SYSTEM
   ALTER SYSTEM
   ALTER SYSTEM
   postgres=# \q
   root@aminin-VirtualBox:/etc/postgresql/15/main# sudo pg_ctlcluster 15 main restart;
   root@aminin-VirtualBox:/etc/postgresql/15/main# pgbench -c27 -j 2 -P 20 -T 60 -h localhost -U postgres test
   Password: 
   pgbench (15.4 (Ubuntu 15.4-2.pgdg23.04+1))
   starting vacuum...end.
   progress: 20.0 s, 1070.4 tps, lat 24.935 ms stddev 18.179, 0 failed
   progress: 40.0 s, 1115.6 tps, lat 24.198 ms stddev 18.223, 0 failed
   progress: 60.0 s, 1043.0 tps, lat 25.873 ms stddev 18.434, 0 failed
   transaction type: <builtin: TPC-B (sort of)>
   scaling factor: 1
   query mode: simple
   number of clients: 27
   number of threads: 2
   maximum number of tries: 1
   duration: 60 s
   number of transactions actually processed: 64607
   number of failed transactions: 0 (0.000%)
   latency average = 24.998 ms
   latency stddev = 18.310 ms
   initial connection time = 209.950 ms
   tps = 1078.833909 (without initial connection time)
  ```  
* Почитал на postgrespro [Оптимизация, угрожающая стабильности](https://postgrespro.ru/docs/postgresql/15/non-durability). Следующие настройки могут приести пользу (ускорить процесс), например: при первоначальной загрузке данных в БД при миграции на PG.

* Включу режим асинхронной записи WAL, не будет ожидания локального сброса WAL на диск. Восстановление после сбоя может быть с потерей последних транзакций, но данные останутся согласованными. И снова прогоню тест
```sql
postgres=# show synchronous_commit;
 synchronous_commit 
--------------------
 on
(1 строка)

postgres=#alter system set synchronous_commit=off;
postgres=#ALTER
postgres=# \q
root@aminin-VirtualBox:/etc/postgresql/15/main# sudo pg_ctlcluster 15 main restart;
root@aminin-VirtualBox:/etc/postgresql/15/main# sudo psql -h localhost -p5432 -U postgres -c 'show synchronous_commit;'
Пароль пользователя postgres: 
 synchronous_commit 
--------------------
 off
(1 строка)

root@aminin-VirtualBox:/etc/postgresql/15/main# pgbench -c27 -P 20 -j 2 -T 60 -h localhost -U postgres test
Password: 
pgbench (15.4 (Ubuntu 15.4-2.pgdg23.04+1))
starting vacuum...end.
progress: 20.0 s, 1547.4 tps, lat 17.258 ms stddev 11.846, 0 failed
progress: 40.0 s, 1562.2 tps, lat 17.280 ms stddev 12.229, 0 failed
progress: 60.0 s, 1542.9 tps, lat 17.491 ms stddev 11.135, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 27
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 93076
number of failed transactions: 0 (0.000%)
latency average = 17.355 ms
latency stddev = 11.784 ms
initial connection time = 199.801 ms
tps = 1553.971769 (without initial connection time)
```

* Стало еще лучше. Теперь отключу принудительную запись изменений на диск fsync, что грозит потерей данных в случае сбоя. И выполню очередное тестирование
```sql
root@aminin-VirtualBox:/etc/postgresql/15/main# sudo psql -h localhost -p5432 -U postgres -c 'alter system set fsync=off;'
Пароль пользователя postgres: 
ALTER SYSTEM
root@aminin-VirtualBox:/etc/postgresql/15/main# sudo pg_ctlcluster 15 main restart;
root@aminin-VirtualBox:/etc/postgresql/15/main# pgbench -c27 -P 20 -j 2 -T 60 -h localhost -U postgres test
Password: 
pgbench (15.4 (Ubuntu 15.4-2.pgdg23.04+1))
starting vacuum...end.
progress: 20.0 s, 1528.4 tps, lat 17.455 ms stddev 11.002, 0 failed
progress: 40.0 s, 1551.2 tps, lat 17.401 ms stddev 10.737, 0 failed
progress: 60.0 s, 1516.2 tps, lat 17.800 ms stddev 11.581, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 27
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 91943
number of failed transactions: 0 (0.000%)
latency average = 17.561 ms
latency stddev = 11.138 ms
initial connection time = 216.573 ms
tps = 1535.454550 (without initial connection time)
```
*============================================
* Попробую создать аналогичную машину на YC, там выберу ssd помедленней и проведу тесты с той же последовательностью шагов изменения настроек

*============================================
```sql
root@vm-ubuntu:/home/ubuntu# pgbench -c27 -j 2 -P 20 -T 60 -h localhost -U postgres test
Password:
pgbench (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
starting vacuum...end.
progress: 20.0 s, 358.1 tps, lat 73.530 ms stddev 68.821, 0 failed
progress: 40.0 s, 409.5 tps, lat 66.130 ms stddev 57.329, 0 failed
progress: 60.0 s, 406.1 tps, lat 66.513 ms stddev 61.038, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 27
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 23500
number of failed transactions: 0 (0.000%)
latency average = 68.511 ms
latency stddev = 62.358 ms
initial connection time = 394.994 ms
tps = 393.902133 (without initial connection time)
```

* Изменил параметры на предложенные калькулятором PGTUNE
```sql 
root@vm-ubuntu:/home/ubuntu# pgbench -c27 -j 2 -P 20 -T 60 -h localhost -U postgres test
Password:
pgbench (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
starting vacuum...end.
progress: 20.0 s, 415.8 tps, lat 63.692 ms stddev 55.707, 0 failed
progress: 40.0 s, 413.9 tps, lat 64.999 ms stddev 58.923, 0 failed
progress: 60.0 s, 548.6 tps, lat 49.461 ms stddev 38.310, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 27
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 27593
number of failed transactions: 0 (0.000%)
latency average = 58.425 ms
latency stddev = 51.132 ms
initial connection time = 324.537 ms
tps = 461.829027 (without initial connection time)
```

* Включу режим асинхронной записи WAL, не будет ожидания локального сброса WAL на диск. Восстановление после сбоя может быть с потерей последних транзакций. но еще не очень высокий риск. И снова прогоню тест
```sql
postgres=# show synchronous_commit;
 synchronous_commit 
--------------------
 on
(1 строка)

postgres=#alter system set synchronous_commit=off;
postgres=#ALTER
postgres=# \q
root@vm-ubuntu:/home/ubuntu# sudo pg_ctlcluster 15 main restart;
root@vm-ubuntu:/home/ubuntu# sudo psql -h localhost -p5432 -U postgres -c 'show synchronous_commit;'
Пароль пользователя postgres: 
 synchronous_commit 
--------------------
 off
(1 строка)

root@vm-ubuntu:/home/ubuntu# pgbench -c27 -j 2 -P 20 -T 60 -h localhost -U postgres test
Password:
pgbench (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
starting vacuum...end.
progress: 20.0 s, 1773.7 tps, lat 14.953 ms stddev 7.018, 0 failed
progress: 40.0 s, 1865.4 tps, lat 14.474 ms stddev 7.151, 0 failed
progress: 60.0 s, 1838.2 tps, lat 14.686 ms stddev 6.791, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 27
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 109574
number of failed transactions: 0 (0.000%)
latency average = 14.706 ms
latency stddev = 7.004 ms
initial connection time = 340.531 ms
tps = 1834.855595 (without initial connection time)
```
* Стало сильно лучше. Разница стала ощутимой между синхронным и асинхронным методом записи WAL четырехкратной при использовании менее быстрого SSD(скорость RW ~400 mb/s), а в прошлом тесте скорость ssd была ~3500 mb/s, при такой скорости результат разрыв в настройках был не большим.
* Теперь отключу принудительную запись изменений на диск fsync, что грозит потерей данных в случае сбоя. И выполню очередное тестирование

```sql
postgres=# show fsync;
 fsync
-------
 on
(1 row)

postgres=# alter system set fsync=off;
ALTER SYSTEM
postgres=# \q
root@vm-ubuntu:/home/ubuntu# pg_ctlcluster 15 main restart
root@vm-ubuntu:/home/ubuntu# pgbench -c27 -j 2 -P 20 -T 60 -h localhost -U postgres test
Password:
pgbench (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
starting vacuum...end.
progress: 20.0 s, 1776.7 tps, lat 14.875 ms stddev 7.356, 0 failed
progress: 40.0 s, 1800.0 tps, lat 14.999 ms stddev 6.986, 0 failed
progress: 60.0 s, 1838.6 tps, lat 14.684 ms stddev 7.216, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 27
number of threads: 2ключ
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 108331
number of failed transactions: 0 (0.000%)
latency average = 14.857 ms
latency stddev = 7.202 ms
initial connection time = 411.557 ms
tps = 1816.179365 (without initial connection time)
```

* Пробовал также играться с wal_compression = on/off, wal_level = minimal и max_wal_senders = 0, но прироста tps не увидел, по сравнению с wal_compression = off, либо wal_level = replica и max_wal_senders = 10

Задание со *: аналогично протестировать через утилиту https://github.com/Percona-Lab/sysbench-tpcc (требует установки
https://github.com/akopytov/sysbench)

* Поставил софт, попробовал запустить, основываясь на статью [PERCONA](https://www.percona.com/blog/tuning-postgresql-for-sysbench-tpcc/), но пошли ошибки: места на диске недостаточно. Пока отложил это, т.к. времени очень мало сейчас, хоть и любопытно попробовать, пока отложил задание со *.