# **Домашнее задание №4**
## Работа с базами данных, пользователями и правами

**Цель:**
 * создание новой базы данных, схемы и таблицы
 * создание роли для чтения данных из созданной схемы созданной базы данных
 * создание роли для чтения и записи из созданной схемы созданной базы данных

**Описание/Пошаговая инструкция выполнения домашнего задания:**

1. создайте новый кластер PostgresSQL 15

	```bash
	ubuntu@vm-ubuntu:~$ sudo pg_createcluster 15 test
	Creating new PostgreSQL cluster 15/test ...
	/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/test --auth-local peer --auth-host scram-sha-256 --no-instructions
	The files belonging to this database system will be owned by user "postgres".
	This user must also own the server process.

	The database cluster will be initialized with locale "en_US.UTF-8".
	The default database encoding has accordingly been set to "UTF8".
	The default text search configuration will be set to "english".

	Data page checksums are disabled.

	fixing permissions on existing directory /var/lib/postgresql/15/test ... ok
	creating subdirectories ... ok
	selecting dynamic shared memory implementation ... posix
	selecting default max_connections ... 100
	selecting default shared_buffers ... 128MB
	selecting default time zone ... Etc/UTC
	creating configuration files ... ok
	running bootstrap script ... ok
	performing post-bootstrap initialization ... ok
	syncing data to disk ... ok
	Ver Cluster Port Status Owner    Data directory              Log file
	15  test    5433 down   postgres /var/lib/postgresql/15/test /var/log/postgresql/postgresql-15-test.log
	```
2. зайдите в созданный кластер под пользователем postgres
	```sql
	ubuntu@vm-ubuntu:~$ sudo -u postgres pg_ctlcluster 15 test start
	Warning: the cluster will not be running as a systemd service. Consider using systemctl:
	  sudo systemctl start postgresql@15-test
	
	ubuntu@vm-ubuntu:~$ sudo -u postgres psql -p 5433
	could not change directory to "/home/ubuntu": Permission denied
	psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
	Type "help" for help.

	postgres=#	
	```
3. создайте новую базу данных testdb
	```sql
	postgres=# create database testdb;
	CREATE DATABASE
	postgres=#
	```
	
4. зайдите в созданную базу данных под пользователем postgres
    ```sql
	postgres=# \c testdb
	You are now connected to database "testdb" as user "postgres".	
	```
5. создайте новую схему testnm
	```sql
	testdb=# create schema testnm;
	CREATE SCHEMA
	```
6. создайте новую таблицу t1 с одной колонкой c1 типа integer
	```sql
	testdb=# create table t1 (c1 integer);
	CREATE TABLE
	```
7. вставьте строку со значением c1=1
	```sql
	testdb=# insert into t1(c1) values(1);
	INSERT 0 1
	```
8. создайте новую роль readonly
	```sql
	testdb=# create role readonly;
	CREATE ROLE
	```
	
9. дайте новой роли право на подключение к базе данных testdb
	```sql
	testdb=# grant connect on database testdb to readonly;
	GRANT
	testdb=# \l testdb
													List of databases
	  Name  |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges
	--------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
	 testdb | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =Tc/postgres         +
			|          |          |             |             |            |                 | postgres=CTc/postgres+
			|          |          |             |             |            |                 | readonly=c/postgres
	(1 row)

	testdb=#
	
	```
10. дайте новой роли право на использование схемы testnm
	```sql
	testdb=# grant USAGE on schema testnm to readonly;
	GRANT	
	testdb=# \dn+ testnm
						List of schemas
	  Name  |  Owner   |  Access privileges   | Description
	--------+----------+----------------------+-------------
	 testnm | postgres | postgres=UC/postgres+|
			|          | readonly=U/postgres  |
	(1 row)	
	```
11. дайте новой роли право на select для всех таблиц схемы testnm
	```sql
	testdb=# grant SELECT ON ALL TABLES IN SCHEMA testnm to readonly;
	GRANT
	```
12. создайте пользователя testread с паролем test123
13. дайте роль readonly пользователю testread
	* Создам роль сразу включая её в роль readonly(объединим пункты 12 и 13)
	```sql
	testdb=# create user testread password 'test123' in role readonly;
	CREATE ROLE
	testdb=# \du
										List of roles
	 Role name |                         Attributes                         | Member of
	-----------+------------------------------------------------------------+------------
	 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
	 readonly  | Cannot login                                               | {}
	 testread  |                                                            | {readonly}
	```
14. зайдите под пользователем testread в базу данных testdb
15. сделайте select * from t1;
	```sql
	ubuntu@vm-ubuntu:~$ sudo psql -h localhost -p5433 -U testread -d testdb
	Password for user testread:
	psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
	SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
	Type "help" for help.

	testdb=> select * from t1;
	ERROR:  permission denied for table t1
	```
16. получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
	* Не получилось, т.к. таблица создалась в схеме puclic, т.к. при создании таблицы не указал схему для размещения таблицы, хотя думал о том, что, вероятно, нужно указать схему.
17. напишите что именно произошло в тексте домашнего задания
	* Не получилось, т.к. таблица создалась в схеме puclic, т.к. при создании таблицы не указал схему размещения таблицы.
18. у вас есть идеи почему? ведь права то дали?
	* Нужно, либо сменить схему таблицы (ALTER TABLE .. SET SCHEMA ..), либо пересоздать таблицу с размещением в нужной схеме
19. посмотрите на список таблиц
	```sql
	testdb=> \d
			List of relations
	 Schema | Name | Type  |  Owner
	--------+------+-------+----------
	 public | t1   | table | postgres
	(1 row)
	```
20. подсказка в шпаргалке под пунктом 20
21. а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)
	* Схема по умолчанию public, search_path для пользователя postgres я не менял после добавления схемы testnm и явно не указал схему при создании таблицы, для роли readonly разрешено исользовать только схему testnm. Пользователь testread имеет аналогичный доступ к схемам, 
	что и readonly,	т.к. входит только в readonly;
	* Переключусь на пользователя postgres и выполню проверку схем
	
	```sql
	postgres=# \c testdb
	SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
	You are now connected to database "testdb" as user "postgres".	

	testdb=# select * From current_user;
	 current_user
	--------------
	 postgres
	(1 row)

	testdb=# select * from current_schema;
	 current_schema
	----------------
	 public
	(1 row)	
	
	testdb=# SHOW search_path;
	   search_path
	-----------------
	 "$user", public
	(1 row)
	
	testdb=# \dn
		  List of schemas
	  Name  |       Owner
	--------+-------------------
	 public | pg_database_owner
	 testnm | postgres
	(2 rows)

	testdb=#	
	```
	
22. вернитесь в базу данных testdb под пользователем postgres
23. удалите таблицу t1
	```
	testdb=# drop table t1;
	DROP TABLE
	```
24. создайте ее заново но уже с явным указанием имени схемы testnm
	* Можно либо явно указать схему при создании объекта schema.object_name, либо изменить search_path, указав нужную схему первой
	и не прибегать к обращению/указанию через schema.object_name
	```
	testdb=# create table testnm.t1 (c1 integer);
	CREATE TABLE
	```
	
25. вставьте строку со значением c1=1
	```
	testdb=# insert into t1 values(1);
	INSERT 0 1
	```
26. зайдите под пользователем testread в базу данных testdb
	```
	ubuntu@vm-ubuntu:~$ sudo psql -h localhost -p5433 -U testread -d testdb
	Password for user testread:
	psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
	SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
	Type "help" for help.
	```
27. сделайте select * from testnm.t1;
	```sql
	testdb=> select * from testnm.t1;
	ERROR:  permission denied for schema testnm
	```
28. получилось?

	* <span style="background:yellow">**Нет прав, как так, не понял, разве права не наследуются на объекты, создаваемые в схеме testnm?**
29. есть идеи почему? если нет - смотрите шпаргалку
	* <span style="background:yellow"> Не понял, почему слетело, но повторил назначение прав на чтение всех таблиц в схеме testnm роли readonly
	```
	testdb=# grant SELECT ON ALL TABLES IN SCHEMA testnm to readonly;
	GRANT
	```
30. как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку
31. сделайте select * from testnm.t1;
	```
	testdb=> select * from testnm.t1;
	 c1
	----
	  1
	(1 row)
	```
32. получилось?
	* Да
33. есть идеи почему? если нет - смотрите шпаргалку
	* Потому, что переназначил права для роли readonly, но так и не понял, почему они слетели
31. сделайте select * from testnm.t1;
32. получилось?
	* Да
33. ура!
34. теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
	```sql
	testdb=> create table t2(c1 integer);
	ERROR:  permission denied for schema public
	LINE 1: create table t2(c1 integer);
	
	testdb=> create table testnm.t2(c1 integer);
	ERROR:  permission denied for schema testnm
	LINE 1: create table testnm.t2(c1 integer);
	```
	* <span style="background:yellow">**Следуя назначенным првам, для пользователя testread ничего кроме connect и "SELECT в схеме testnm" и не положено. Всё логично.**
35. а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?
	* <span style="background:yellow">**Что-то, видимо, у меня идет в разрез с предполагаемым ходом событий по заданию (судя по изложению текста), возможно, начиная с пункта 27**
36. есть идеи как убрать эти права? если нет - смотрите шпаргалку
37. если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
38. теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
39. расскажите что получилось и почему

	* <span style="background:yellow">**См.шпаргалку - под шпаргалкой имеется ввиду этот лист задач, или файлик, прилагавшийся к лекции?**