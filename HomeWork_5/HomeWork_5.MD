# **Домашнее задание №5** 
## Настройка autovacuum с учетом особеностей производительности

**Цель:**\
запустить нагрузочный тест pgbench\
настроить параметры autovacuum\
проверить работу autovacuum

Описание/Пошаговая инструкция выполнения домашнего задания:
Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB
Установить на него PostgreSQL 15 с дефолтными настройками
Создать БД для тестов: выполнить pgbench -i postgres
```sql
postgres=# create database test;
CREATE DATABASE
postgres=\q
```
	
1. Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres

```sql
root@aminin-VirtualBox:/etc/postgresql/15/main# pgbench -c8 -P 6 -T 60 -h localhost -U postgres test
Password: 
pgbench (15.4 (Ubuntu 15.4-1.pgdg23.04+1))
pgbench: error: could not count number of branches: ОШИБКА:  отношение "pgbench_branches" не существует
LINE 1: select count(*) from pgbench_branches
							 ^
pgbench: hint: Perhaps you need to do initialization ("pgbench -i") in database "test".
	
root@aminin-VirtualBox:/etc/postgresql/15/main# sudo pgbench -i -h localhost -U postgres -d test
Password: 
dropping old tables...
ЗАМЕЧАНИЕ:  таблица "pgbench_accounts" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_branches" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_history" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_tellers" не существует, пропускается
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.01 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.28 s (drop tables 0.01 s, create tables 0.03 s, client-side generate 0.14 s, vacuum 0.05 s, primary keys 0.05 s).
root@aminin-VirtualBox:/etc/postgresql/15/main#

aminin@aminin-VirtualBox:~$ pgbench -c8 -P 6 -T 60 -h localhost -U postgres test
Password: 
pgbench (15.4 (Ubuntu 15.4-1.pgdg23.04+1))
starting vacuum...end.
progress: 6.0 s, 1001.8 tps, lat 7.819 ms stddev 3.578, 0 failed
progress: 12.0 s, 1053.5 tps, lat 7.580 ms stddev 3.361, 0 failed
progress: 18.0 s, 1054.5 tps, lat 7.575 ms stddev 3.621, 0 failed
progress: 24.0 s, 1046.9 tps, lat 7.615 ms stddev 4.088, 0 failed
progress: 30.0 s, 1088.3 tps, lat 7.331 ms stddev 3.647, 0 failed
progress: 36.0 s, 1044.7 tps, lat 7.646 ms stddev 3.399, 0 failed
progress: 42.0 s, 1059.4 tps, lat 7.538 ms stddev 3.290, 0 failed
progress: 48.0 s, 1057.8 tps, lat 7.551 ms stddev 3.230, 0 failed
progress: 54.0 s, 1076.5 tps, lat 7.418 ms stddev 3.090, 0 failed
progress: 60.0 s, 1064.7 tps, lat 7.502 ms stddev 3.307, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 63295
number of failed transactions: 0 (0.000%)
latency average = 7.555 ms
latency stddev = 3.472 ms
initial connection time = 111.021 ms
tps = 1056.667547 (without initial connection time)
```
2. Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла\
	* Изменил настройки в postgres.conf и командой pg_ctlcluster 15 main restart перезапустил кластер
```sql	
SELECT name, setting, unit 
  FROM pg_settings WHERE name in 
( 'shared_buffers',
  'max_connections',
  'effective_cache_size',
  'maintenance_work_mem',
  'checkpoint_completion_target',
  'wal_buffers',
  'default_statistics_target',
  'random_page_cost',
  'effective_io_concurrency',
  'work_mem',
  'min_wal_size',
  'max_wal_size');
             name             | setting | unit 
------------------------------+---------+------
 checkpoint_completion_target | 0.9     | 
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
(12 строк)	
```
3. Протестировать заново

```sql
root@aminin-VirtualBox:/etc/postgresql/15/main# pgbench -c8 -P 6 -T 60 -h localhost -U postgres test
Password: 
pgbench (15.4 (Ubuntu 15.4-1.pgdg23.04+1))
starting vacuum...end.
progress: 6.0 s, 1063.5 tps, lat 7.362 ms stddev 3.086, 0 failed
progress: 12.0 s, 1065.6 tps, lat 7.498 ms stddev 3.486, 0 failed
progress: 18.0 s, 1076.3 tps, lat 7.417 ms stddev 3.137, 0 failed
progress: 24.0 s, 1081.5 tps, lat 7.389 ms stddev 3.299, 0 failed
progress: 30.0 s, 1061.5 tps, lat 7.522 ms stddev 3.422, 0 failed
progress: 36.0 s, 1052.5 tps, lat 7.588 ms stddev 3.664, 0 failed
progress: 42.0 s, 1090.8 tps, lat 7.323 ms stddev 3.192, 0 failed
progress: 48.0 s, 1066.3 tps, lat 7.489 ms stddev 3.431, 0 failed
progress: 54.0 s, 1105.2 tps, lat 7.228 ms stddev 2.984, 0 failed
progress: 60.0 s, 1091.8 tps, lat 7.316 ms stddev 3.231, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 64538
number of failed transactions: 0 (0.000%)
latency average = 7.413 ms
latency stddev = 3.302 ms
initial connection time = 112.946 ms
tps = 1077.216033 (without initial connection time)
```

Что изменилось и почему?\
	* Немного увеличилось кол-во tps, значительного прироста не произошло, возможно потому, что ВМ развернута 
	в VirtualBox c полной Ubuntu 22 и кол-во выделенных ресурсов для ВМ достаточно скромное. Хотя утилита TOP 
	не показала пиковой загрузки CPU и RAM при выполнении pgbench, да и диск nvme со скоростью RW~3500
	
4. Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк
```sql
test=# create table t(cval char(80));
CREATE TABLE
test=# insert into t select 'default_value' from generate_series(1, 1000000);
INSERT 0 1000000
test=# select cval, count(cval) from t group by cval;
                                       cval                                       |  count  
----------------------------------------------------------------------------------+---------
 default_value                                                                    | 1000000
(1 строка)

test=# 
```

5. Посмотреть размер файла с таблицей
```sql
test=# SELECT pg_size_pretty(pg_total_relation_size('t'));
 pg_size_pretty 
----------------
 112 MB
(1 строка)

```
6. 5 раз обновить все строчки и добавить к каждой строчке любой символ
```sql
test=# update t set cval = cval||'=';
UPDATE 1000000
test=# update t set cval = cval||'=';
UPDATE 1000000
test=# update t set cval = cval||'=';
UPDATE 1000000
test=# update t set cval = cval||'=';
UPDATE 1000000
test=# update t set cval = cval||'=';
UPDATE 1000000
test=# SELECT pg_size_pretty(pg_total_relation_size('t'));
 pg_size_pretty 
----------------
 670 MB
(1 строка)

```
7. Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
```sql
test=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 't';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 t       |    1000000 |    4999923 |    499 | 2023-09-07 23:57:52.215648+05
(1 строка)
```
8. Подождать некоторое время, проверяя, пришел ли автовакуум
```sql
test=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 't';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 t       |    1000000 |          0 |      0 | 2023-09-07 23:58:54.538074+05
(1 строка)

test=# SELECT pg_size_pretty(pg_total_relation_size('t'));
 pg_size_pretty 
----------------
 670 MB
(1 строка)
```

9. 5 раз обновить все строчки и добавить к каждой строчке любой символ
```sql
test=# update t set cval = cval||'=';update t set cval = cval||'=';update t set cval = cval||'=';update t set cval = cval||'=';update t set cval = cval||'=';
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
```
10. Посмотреть размер файла с таблицей
```sql
test=# SELECT pg_size_pretty(pg_total_relation_size('t'));
 pg_size_pretty 
----------------
 670 MB
```

11. Отключить Автовакуум на конкретной таблице\
10 раз обновить все строчки и добавить к каждой строчке любой символ\
Посмотреть размер файла с таблицей
```sql
test=# ALTER TABLE t SET (autovacuum_enabled = off);
ALTER TABLE
test=#
test=# update t set cval = cval||'=';update t set cval = cval||'=';update t set cval = cval||'=';update t set cval = cval||'=';update t set cval = cval||'=';update t set cval = cval||'=';update t set cval = cval||'=';update t set cval = cval||'=';update t set cval = cval||'=';update t set cval = cval||'=';
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
UPDATE 1000000
test=#
test=# SELECT pg_size_pretty(pg_total_relation_size('t'));
 pg_size_pretty 
----------------
 1228 MB
(1 строка)

test=# 

```
Объясните полученный результат.\
* После генерации 1млн записей таблица стала занимать 112мб, при каждом обновлении из пяти, исходные строки помечаются на удаление (физически не удаляются) и создается новая версия строк с новыми значениями, и так при каждом UPDATE. Итого при 5-ти UPDATE места стало 112 + 112 х 5 = примерно 670
Далее автовакуум в течение ~минуты удалил мертвые строки и освободил 560 мегабайт данных в виде резерва под будущие данные. размер таблицы при этом не уменьшая.\
Далее отключаем автовакуум таблицы и делаем 10 UPDATE, пять из них занимают освободившееся ранее место 560 мб, а еще пять занимают новые 560 мб места на диске. Итого 560 х 2 + 112 (исходная строка до 10-кратного UPDATE) = примерно 1228 мб (на калькуляторе 1232). Смотрим процентное соотношение мертвых строк к живым:
```sql
test=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 't';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 t       |    1000000 |    9998636 |    999 | 2023-09-08 00:23:54.500842+05

```

Не забудьте включить автовакуум)
```sql
test=# ALTER TABLE t SET (autovacuum_enabled = on);
```


__12. Задание со *:__
Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице.
Не забыть вывести номер шага цикла.
```sql
test=# do $$ declare i integer;
begin 
for i in 1..10 
loop
  update t set cval = cval || '=';
  raise info 'step = %', cast( i as character varying);
end loop;
end
$$;  
ИНФОРМАЦИЯ:  step = 1
ИНФОРМАЦИЯ:  step = 2
ИНФОРМАЦИЯ:  step = 3
ИНФОРМАЦИЯ:  step = 4
ИНФОРМАЦИЯ:  step = 5
ИНФОРМАЦИЯ:  step = 6
ИНФОРМАЦИЯ:  step = 7
ИНФОРМАЦИЯ:  step = 8
ИНФОРМАЦИЯ:  step = 9
ИНФОРМАЦИЯ:  step = 10
DO
test=# 
```




