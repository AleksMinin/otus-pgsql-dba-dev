# **Домашнее задание №1** 
## Работа с уровнями изоляции транзакции в PostgreSQL

**Цель:**
* Научиться работать с Google Cloud Platform на уровне Google Compute Engine (IaaS)
 * Научиться управлять уровнем изолции транзации в PostgreSQL и понимать особенность работы уровней read commited и repeatable read

**Описание/Пошаговая инструкция выполнения домашнего задания/Выполнение:**

1. Создать новый проект в Google Cloud Platform, Яндекс облако или на любых ВМ, докере.
2. Далее создать инстанс виртуальной машины с дефолтными параметрами.
3. Добавить свой ssh ключ в metadata ВМ.
4. 3айти удаленным ssh (первая сессия), не забывайте про ssh-add.
5. Поставить PostgreSQL.
6. Зайти вторым ssh (вторая сессия).
7. Запустить везде psql из под пользователя postgres.
8. Выключить auto commit.
	```sql
	\set autocommit off
	```	
9. Сделать в первой сессии новую таблицу и наполнить ее данными
	*__create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;__*
10. Посмотреть текущий уровень изоляции: *__show transaction isolation level;__*
	```sql
	hw=# show transaction isolation level;
	 transaction_isolation
	-----------------------
	 read committed
	(1 row)
	```
11. Начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
12. В первой сессии добавить новую запись *__insert into persons(first_name, second_name) values('sergey', 'sergeev');__*
13. Сделать __*select * from persons;*__ во второй сессии.
	> __*<u>Вопрос:</u> Видите ли вы новую запись и если да то почему?*__
	>
	> __*<u>Ответ:</u> Новой записи нет, т.к. при уровне изоляции read commited будут видны только те данные, которые были зафиксированы другими транзакциями до начала выполнения запроса в текущей транзакции (исключено грязное чтение)*__
	```sql
	 id | first_name | second_name
	----+------------+-------------
	  1 | ivan       | ivanov
	  2 | petr       | petrov
	(2 rows)
	```
14. Завершить первую транзакцию - __*commit;*__
15. Сделать __*select * from persons;*__ во второй сессии
	> __*<u>Вопрос:</u> Видите ли вы новую запись и если да то почему?*__
	>
	>__*<u>Ответ:</u> Новая запись видна, т.к. первая транзакция завершилась фиксацией изменений commit и поврторное чтение в текущей транзакции даст другой результат (присутствует эффект Фантомного чтения), а также есть эффект неповторяющегося чтения (если бы мы выполняли UPDATE в другой транзакции зафиксировали и снова выполнили бы выборку в текущей, то увидели бы уже измененные данные);*__	
	
	```sql
	 id | first_name | second_name
	----+------------+-------------
	  1 | ivan       | ivanov
	  2 | petr       | petrov
	  3 | sergey     | sergeev
	(3 rows)
	```
16. Завершите транзакцию во второй сессии.
17. Начать новые но уже repeatable read транзации - *__set transaction isolation level repeatable read;__*
18. В первой сессии добавить новую запись *__insert into persons(first_name, second_name) values('sveta', 'svetova');__*
19. Сделать *__select * from persons;__* во второй сессии
	> __*<u>Вопрос:</u> Видите ли вы новую запись и если да то почему?*__
	>
	> __*<u>Ответ:</u> Нет, еще не завершены транзакции фиксацией изменений, грязное чтение на этом уровне изоляции недопустимо*__
	```sql
	 id | first_name | second_name
	----+------------+-------------
	  1 | ivan       | ivanov
	  2 | petr       | petrov
	  3 | sergey     | sergeev
	(3 rows)
	```
20. Завершить первую транзакцию - commit;
21. Сделать *__select * from persons;__* во второй сессии
	> __*<u>Вопрос:</u> Видите ли вы новую запись и если да то почему?*__
	>
	> __*<u>Ответ:</u> Нет, т.к. эффект фантомного чтения на данном уровне изоляции в PG отсутствует. Данная транзакция повесила блокировку на читаемые данные и пока она не завершится, другие транзакции не смогут применить изменения.*__
	```sql
	 id | first_name | second_name
	----+------------+-------------
	  1 | ivan       | ivanov
	  2 | petr       | petrov
	  3 | sergey     | sergeev
	(3 rows)
	```
22. Завершить вторую транзакцию
23. Сделать *__select * from persons;__* во второй сессии
	> __*<u>Вопрос:</u> Видите ли вы новую запись и если да то почему?*__
	>
	> __*<u>Ответ:</u> Да, теперь изменения видны. Вторая транзакция завершилась и позволила другим транзакциям записать изменение/добавление данных. До завершения текущей транзакции другие ждали снятия блокировки читаемых текущей транзакцией данных.*__
	```sql
	select * from persons;
	 id | first_name | second_name
	----+------------+-------------
	  1 | ivan       | ivanov
	  2 | petr       | petrov
	  3 | sergey     | sergeev
	  4 | sveta      | svetova
	(4 rows)
	```