# Домашнее задание №9 - Бэкапы
## Цель: Применить логический бэкап. Восстановиться из бэкапа

Описание/Пошаговая инструкция выполнения домашнего задания:
1. Создаем ВМ/докер c ПГ.
2. Создаем БД, схему и в ней таблицу.
3. Заполним таблицы автосгенерированными 100 записями.
```sql
postgres=# create database test;
CREATE DATABASE
test=# create schema schema_test;
CREATE SCHEMA
test=# grant USAGE on schema sch to postgres;
GRANT
test=# create table schema_test.tbl1 as select 'Hello!!!' from generate_series(1, 100);
SELECT 100
```
4. Под линукс пользователем Postgres создадим каталог для бэкапов
```sql
postgres@aminin-VirtualBox:/home/aminin$ mkdir /tmp/backups
```
5. Сделаем логический бэкап используя утилиту COPY
```sql
postgres@aminin-VirtualBox:/tmp$ psql -hlocalhost -p5432 -d test
Пароль пользователя postgres: 
psql (15.4 (Ubuntu 15.4-2.pgdg23.04+1))
SSL-соединение (протокол: TLSv1.3, шифр: TLS_AES_256_GCM_SHA384, сжатие: выкл.)
Введите "help", чтобы получить справку.
test=# \copy sch.tbl1 to '/tmp/backups/mm.sql'
COPY 100
test=#
```
6. Восстановим в 2 таблицу данные из бэкапа.
```sql
test=# create table sch.tbl2(cval text);
CREATE TABLE
test=#
test=# copy tbl2 from '/tmp/backups/mm.sql';
COPY 100
test=# select * from tbl2 limit 5;
   cval   
----------
 Hello!!!
 Hello!!!
 Hello!!!
 Hello!!!
 Hello!!!
(5 строк)
```
7. Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц
```sql
postgres@aminin-VirtualBox:/tmp$ pg_dump -d test --create -U postgres -Fc > /tmp/backups/arh.gz
```
8. Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!
```sql
postgres=# create database test2;
CREATE DATABASE
postgres=#\c test2

test2=# create schema sch;
CREATE SCHEMA
test2=# grant usage on schema sch to postgres;
GRANT
test2=#\q
postgres@aminin-VirtualBox:/tmp/backups$ pg_restore -d test2 -U postgres -t tbl2 /tmp/backups/arh.gz
postgres@aminin-VirtualBox:/tmp/backups$ psql -hlocalhost -p5432 -dtest2
Пароль пользователя postgres: 
psql (15.4 (Ubuntu 15.4-2.pgdg23.04+1))
SSL-соединение (протокол: TLSv1.3, шифр: TLS_AES_256_GCM_SHA384, сжатие: выкл.)
Введите "help", чтобы получить справку.
test2=# \dt
Отношения не найдены.
test2=# set search_path = sch,"$user",public;
SET
test2=# \dt
         Список отношений
 Схема | Имя  |   Тип   | Владелец 
-------+------+---------+----------
 sch   | tbl2 | таблица | postgres
(1 строка)

test2=# select * from tbl2 limit 3; 
   cval   
----------
 Hello!!!
 Hello!!!
 Hello!!!
(3 строки)
```
* Проблем при решении не возникало, кроме того, что забыл схему добавить в search_path