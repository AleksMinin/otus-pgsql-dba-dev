# Домашнее задание №11
## Работа с индексами

**Цель:**
знать и уметь применять основные виды индексов PostgreSQL
строить и анализировать план выполнения запроса
уметь оптимизировать запросы для с использованием индексов

Описание/Пошаговая инструкция выполнения домашнего задания:
Создать индексы на БД, которые ускорят доступ к данным.
В данном задании тренируются навыки:
определения узких мест
написания запросов для создания индекса
оптимизации
Необходимо:
1. Создать индекс к какой-либо из таблиц вашей БД
Прислать текстом результат команды explain,
в которой используется данный индекс
```sql
do $$
begin
   /*ПереСоздадим таблицу*/
   drop table tbl3;
   create table tbl3(nID int GENERATED ALWAYS AS IDENTITY,          
                    sVal varchar(80), 
                    nVal numeric(15,2));	
	
   /*Сгенерируем случайные данные*/
   insert into tbl3(sVal, nVal)
   select 'Один день из '||trim(to_char(random()*3000,'9999')),
          random()*5000
     from generate_series(1,3000); 
	
   /*Создам простой индекс на поле nId*/ 
   create index idx_tbl3_nId on tbl3(nId); 
end;
$$
/*Выполним запрос*/
explain
select * from tbl3 where nid < 1000;
"QUERY PLAN"
Index Scan using idx_tbl3_nid on tbl3  (cost=0.28..31.26 rows=999 width=37)
  Index Cond: (nid < 1000)

```
2. Реализовать индекс для полнотекстового поиска 
	* Реализацию индекса полнотекстового поиска и индекса с функцией объединю в один пункт. Преобразование текста to_tsvector с русскоязычного текста буду делать через функцию, которую предварительно создам
	
	
```sql
-- Пример содержимого таблицы
select t.*
  from tbl3 t limit 3

nid|sval             |nval   |
---+-----------------+-------+
  1|Один день из 2478|1480.52|
  2|Один день из 1444|3864.68|
  3|Один день из 2565|3008.19|

-- Создание функции преобразования к tsvector
create or replace function make_tsvector_ru(stxt text)
returns tsvector 
as $$
begin 
  return(to_tsvector('russian', stxt)); 
end;
$$ language 'plpgsql' immutable;

-- Сначала посмотрю план запроса без индекса
explain
select t.*
  from tbl3 t
 where make_tsvector_ru(sval) @@ to_tsquery('день');
"QUERY PLAN"
Seq Scan on tbl3 t  (cost=0.00..1562.50 rows=15 width=37)
  Filter: (to_tsvector('russian'::regconfig, (sval)::text) @@ to_tsquery('день'::text))

-- Создам полнотекстовый индекс с использованием функции
create index idx_gin_tbl3_sval on tbl3 using gin (make_tsvector_ru(sval::text));

explain
select t.*
  from tbl3 t
 where make_tsvector_ru(sval) @@ to_tsquery('день');
"QUERY PLAN"  
 Bitmap Heap Scan on tbl3 t  (cost=3.67..23.72 rows=15 width=37)
  Recheck Cond: (make_tsvector_ru((sval)::text) @@ to_tsquery('день'::text))
  ->  Bitmap Index Scan on idx_gin_tbl3_sval  (cost=0.00..3.66 rows=15 width=0)
        Index Cond: (make_tsvector_ru((sval)::text) @@ to_tsquery('день'::text)) 
```  
3. Реализовать индекс на часть таблицы или индекс на поле с функцией
* Индекс с функцией реализовал во втором пункте
```sql
--Без индекса
explain
select t.*
  from tbl3 t 
 where t.nval < 2000

"QUERY PLAN"
 Seq Scan on tbl3 t  (cost=0.00..62.50 rows=1196 width=37)
  Filter: (nval < '2000'::numeric)

-- Создаю индекс на поле nval со значением меньше 2000
create index idx_tbl3_nval_low2000 on tbl3(nval) where nval < 2000;

-- Проверка
explain
select t.*
  from tbl3 t 
 where t.nval < 2000
 
"QUERY PLAN" 
Index Scan using idx_tbl3_nval_low2000 on tbl3 t  (cost=0.28..52.31 rows=1196 width=37)

explain
select t.*
  from tbl3 t 
 where t.nval > 2000

"QUERY PLAN" 
Seq Scan on tbl3 t  (cost=0.00..62.50 rows=1803 width=37)
  Filter: (nval > '2000'::numeric)
```	
	
4. Создать индекс на несколько полей
```sql
-- Проверка
explain
select t.*
  from tbl3 t 
 where t.nval < 2000 and substr(sval,14,1) = '2'  
 limit 3;

"QUERY PLAN" 
Limit  (cost=0.00..38.75 rows=3 width=37)
  ->  Seq Scan on tbl3 t  (cost=0.00..77.50 rows=6 width=37)
        Filter: ((nval < '2000'::numeric) AND (substr((sval)::text, 14, 1) = '2'::text))
		
--Создам индекс по двум полям
create index idx_tbl3_nval_sval on tbl3(nval,sval);

--Повторю запрос
explain
select t.*
  from tbl3 t 
 where t.nval < 2000 and substr(sval,14,1) = '2';

"QUERY PLAN" 
Index Scan using idx_tbl3_nval_sval on tbl3 t  (cost=0.28..65.69 rows=6 width=37)
  Index Cond: (nval < '2000'::numeric)
  Filter: (substr((sval)::text, 14, 1) = '2'::text)

```


5. Написать комментарии к каждому из индексов. Описать что и как делали и с какими проблемами
столкнулись
	* Пришлось изучать документацию по полнотекстовым индексам, т.к. с ними никогда не сталкивался и в вэбинаре тоже их не затронули.\
	По остальным индексам проблем никаких не было.
	
	
