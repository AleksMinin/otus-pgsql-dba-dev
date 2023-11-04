# Домашнее задание №12
## Секционирование таблицы

**Цель:**
научиться секционировать таблицы.

Описание/Пошаговая инструкция выполнения домашнего задания:
Секционировать большую таблицу из демо базы flights

```sql
do
$$ 
declare
    sQuery   text;        -- Текст запроса
    sValFrom varchar(10); -- Нижний граница интервала
    sValTo   varchar(10); -- Верхняя граница интервала
begin
    drop table if exists flights_range;
    -- Создание головной секционированной таблицы 
    CREATE TABLE flights_range 
    (   flight_id serial4 NOT NULL,
        flight_no bpchar(6) NOT NULL,
        scheduled_departure timestamptz NOT NULL,
        scheduled_arrival timestamptz NOT NULL,
        departure_airport bpchar(3) NOT NULL,
        arrival_airport bpchar(3) NOT NULL,
        status varchar(20) NOT NULL,
        aircraft_code bpchar(3) NOT NULL,
        actual_departure timestamptz NULL,
        actual_arrival timestamptz NULL,
        PRIMARY KEY (flight_id, scheduled_departure)
    ) PARTITION BY RANGE(scheduled_departure);

    -- Цикл по год, месяц из исходной таблицы flights
    declare cr record;
    begin
      for cr in (select distinct
                        date_trunc('month', f.scheduled_departure) as dBgnRange
                   from flights f  
                  order by dBgnRange)
      loop
          sValFrom := to_char(cr.dBgnRange, 'YYYY-MM-DD');  
          sValTo	 := to_char(cr.dBgnRange + interval '1 month', 'YYYY-MM-DD');
          /* Формирование имени секции и её границ 
             значений плановой даты вылета */
          sQuery := format($que$
                             CREATE TABLE flights_range_%s
                             PARTITION OF flights_range
                             FOR VALUES FROM ('%s') TO ('%s');
                           $que$, to_char(cr.dBgnRange, 'YYYY_MM'), sValFrom, sValTo);
          execute sQuery; 				
     end loop;
   end;
   -- Заполнение секционированной таблицы flights_range
   insert into bookings.flights_range
   select * from bookings.flights;
  end;
$$
-- Проведение тестов

-- 1. Запрос кол-ва записей из НЕ секционированной таблицы за май 2017
explain analyze
select count(*) from flights f 
 where f.scheduled_departure between date'2017-05-01' and date'2017-05-31'

Aggregate  (cost=1049.45..1049.46 rows=1 width=8) (actual time=5.973..5.974 rows=1 loops=1)
  ->  Index Only Scan using flights_flight_no_scheduled_departure_key on flights f  (cost=0.42..1029.98 rows=7786 width=0) (actual time=0.032..5.681 rows=7596 loops=1)
        Index Cond: ((scheduled_departure >= '2017-05-01'::date) AND (scheduled_departure <= '2017-05-31'::date))
        Heap Fetches: 6
Planning Time: 0.069 ms
Execution Time: 5.992 ms

 -- 2. Запрос кол-ва записей из секционированной таблицы за май 2017
explain analyze
select count(*) from flights_range f 
 where f.scheduled_departure between date'2017-05-01' and date'2017-05-31'

 Aggregate  (cost=1081.97..1081.98 rows=1 width=8) (actual time=1.736..1.736 rows=1 loops=1)
  ->  Append  (cost=0.28..1062.90 rows=7629 width=0) (actual time=0.031..1.459 rows=7596 loops=1)
        Subplans Removed: 4
        ->  Index Only Scan using flights_range_2017_05_pkey on flights_range_2017_05 f_1  (cost=0.28..195.37 rows=7625 width=0) (actual time=0.031..1.067 rows=7596 loops=1)
              Index Cond: ((scheduled_departure >= '2017-05-01'::date) AND (scheduled_departure <= '2017-05-31'::date))
              Heap Fetches: 0
Planning Time: 0.168 ms
Execution Time: 1.758 ms
```
* Сокращение времени запроса в 3,5 раза при секционировании таблицы flights методом BY_RANGE

* Сначала не понял, что нужно не таблицу flights секционировать, а просто большую таблицу секционировать. 
* Но в качестве прмера для практики подойдет и таблица flights
* Раз сначала ошибся с выбором, секционирую самую большую таблицу "ticket_flights" демо-базы, и т.к. она больше претендует на BY LIST, сделаю именно BY_LIST
```sql
do
$$ 
declare
    sQuery   text;	-- Текст запроса
begin
	
    -- Создание секционированной таблицы "BY LIST"
    CREATE TABLE ticket_flights_bylst
    (  ticket_no bpchar(13) NOT NULL,
       flight_id int4 NOT NULL,
       fare_conditions varchar(10) NOT NULL,
       amount numeric(10, 2) NOT null,
       primary key (ticket_no, flight_id, fare_conditions)
    )  PARTITION BY LIST (fare_conditions);

    declare cr record;
    begin
	    /* Пройти по Условиям перелета - Класс 
	       и создать секции по Классам */
         for cr in (select distinct 
                           f.fare_conditions
                      from ticket_flights f
                     order by 1)
        loop
            sQuery := format($que$
                               CREATE TABLE ticket_flights_bylst_%s
                               PARTITION OF ticket_flights_bylst
                               FOR VALUES IN ('%s');
                             $que$, cr.fare_conditions, cr.fare_conditions);
            execute sQuery; 				
        end loop;
    end;
    -- Заполнение секционированной таблицы
    insert into ticket_flights_bylst
    select * from ticket_flights;
end;
$$

-- Тестирование
-- 1. Запрос кол-ва записей "Бизнес-класс" по не секционированной таблице
explain analyze
select count(*) from ticket_flights f where f.fare_conditions = 'Business';

Finalize Aggregate  (cost=33218.41..33218.42 rows=1 width=8) (actual time=112.408..112.702 rows=1 loops=1)
  ->  Gather  (cost=33218.20..33218.41 rows=2 width=8) (actual time=95.829..112.468 rows=3 loops=1)
        Workers Planned: 2
        Workers Launched: 2
        ->  Partial Aggregate  (cost=32218.20..32218.21 rows=1 width=8) (actual time=93.346..93.347 rows=1 loops=3)
              ->  Parallel Seq Scan on ticket_flights f  (cost=0.00..31963.07 rows=102049 width=0) (actual time=0.583..86.638 rows=80735 loops=3)
                    Filter: ((fare_conditions)::text = 'Business'::text)
                    Rows Removed by Filter: 706044
Planning Time: 0.052 ms
Execution Time: 112.722 ms

-- 2. Запрос кол-ва записей "Бизнес-класс" по секционированной таблице
explain analyze
select count(*) from ticket_flights_bylst f where f.fare_conditions = 'Business';

Finalize Aggregate  (cost=5156.21..5156.22 rows=1 width=8) (actual time=30.262..30.564 rows=1 loops=1)
  ->  Gather  (cost=5156.09..5156.20 rows=1 width=8) (actual time=24.876..30.556 rows=2 loops=1)
        Workers Planned: 1
        Workers Launched: 1
        ->  Partial Aggregate  (cost=4156.09..4156.10 rows=1 width=8) (actual time=20.651..20.652 rows=1 loops=2)
              ->  Parallel Seq Scan on ticket_flights_bylst_business f  (cost=0.00..3799.91 rows=142473 width=0) (actual time=0.012..13.718 rows=121102 loops=2)
                    Filter: ((fare_conditions)::text = 'Business'::text)
Planning Time: 0.079 ms
Execution Time: 30.585 ms
```
* Сокращение времени выборки примерно 4 раза при секционировании таблицы ticket_flights методом BY_LIST
Критерии оценки:
Выполнение ДЗ: 10 баллов
плюс 2 балл за красивое решение
минус 2 балл за рабочее решение, и недостатки указанные преподавателем не устранены