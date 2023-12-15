# Итоговая работа по курсу.
## Цель - провести оптимизацию аналитического запроса по кол-ву пассажирских перевозок, с числом посадочных мест на рейс и числом проданных билетов на каждую поездку, с информацией о месте отправления (Город - автовокзал) на базе с 60 млн.записей 

* Схема базы данных
<kbd>
  <img src="https://github.com/AleksMinin/otus-pgsql-dba-dev/raw/main/ItogWork/images/schema_db.png" alt="Схема БД" caption="Схема БД"/>
</kbd>

```sql
/* Запрос для оптимизации */
explain analyze 
SELECT r.id, r.startdate as depart_date, bs.city || ', ' || bs.name as busstation, 
       count(t.id) as order_place, count(st.id) as all_place
  FROM book.ride as r
  JOIN book.schedule as s    on r.fkschedule = s.id
  JOIN book.seat as st       on r.fkbus = st.fkbus
  JOIN book.busroute as br   on s.fkroute = br.id
  JOIN book.busstation as bs on br.fkbusstationfrom = bs.id
  JOIN book.tickets as t     on t.fkride = r.id
 GROUP BY r.id, r.startdate, bs.city || ', ' || bs.name 
 ORDER BY r.startdate
 
 
-- Исходный запрос показывает неверные данные по общему кол-ву мест в автобусе на рейсе и кол-ву проданных билетов
-- по 1440 мест в автобусе быть не может
d|depart_date|busstation            |order_place|all_place|
-+-----------+----------------------+-----------+---------+
2| 2000-01-01|Bankgkok, Suvarnabhumi|       1440|     1440|
3| 2000-01-01|Bankgkok, Suvarnabhumi|       1400|     1400|
4| 2000-01-01|Bankgkok, Eastern     |       1400|     1400|
5| 2000-01-01|Bankgkok, Eastern     |       1440|     1440|

```

* Предварительно выполняю подготовительные операции (Вакуум и сбор статистики)
* VACUUM FULL;
* ANALYZE;

* Запуск каждого запроса выполняю два раза, первый прогревочный, второй раз контрольный (замеры прилагаются со второго запуска).
* Тестирование запросов проводил на двух виртуальных машинах: 
  - CPU 4-core, RAM 16Gb, SSDnvme 40Gb.
	```sql
	aminin@aminin-VirtualBox:~$ sudo hdparm -t /dev/nvme0n1

	/dev/nvme0n1:
	 Timing buffered disk reads: 2092 MB in  3.00 seconds = 696.43 MB/sec

	name                            |setting|
	--------------------------------+-------+
	checkpoint_completion_target    |0.9    |
	default_statistics_target       |500    |
	effective_cache_size            |1572864|
	effective_io_concurrency        |200    |
	huge_pages                      |off    |
	maintenance_work_mem            |2097152|
	max_connections                 |20     |
	max_parallel_maintenance_workers|2      |
	max_parallel_workers            |4      |
	max_parallel_workers_per_gather |2      |
	max_wal_size                    |16384  |
	max_worker_processes            |4      |
	min_wal_size                    |4096   |
	random_page_cost                |4      |
	shared_buffers                  |524288 |
	wal_buffers                     |2048   |
	work_mem                        |200MB  |	 
	 
	 
	```    

  - CPU 4-core, RAM 16Gb, HDD 40Gb.
	```sql
	ubuntu@vm-ubuntu:~$ sudo hdparm -t /dev/vda

	/dev/vda:
	 Timing buffered disk reads: 100 MB in  3.07 seconds =  32.61 MB/sec
	 
	# Настройки кластера PG 15
	name                            |setting|
	--------------------------------+-------+
	checkpoint_completion_target    |0.9    |
	default_statistics_target       |500    |
	effective_cache_size            |1572864|
	effective_io_concurrency        |2      |
	huge_pages                      |off    |
	maintenance_work_mem            |2097152|
	max_connections                 |20     |
	max_parallel_maintenance_workers|2      |
	max_parallel_workers            |4      |
	max_parallel_workers_per_gather |2      |
	max_wal_size                    |16384  |
	max_worker_processes            |4      |
	min_wal_size                    |4096   |
	random_page_cost                |4      |
	shared_buffers                  |524288 |
	wal_buffers                     |2048   |
	work_mem                        |200MB  |	
	```  
# Решение:
* **!!!Исходный запрос подсчитывает неверные кол-ва билетов**
* В моих решениях общее кол-во мест на рейс и проданное кол-во билетов, корректно. Если по моему решению еще посчитать Итоговое sum(order_place), то  как раз и получится общее число записей в таблице tickets.
* Увеличил work_mem с 55MB, предложенных PGTUNES для типа ДБ "DWH" до 200MB  на обеих виртуалках, т.к. в тестах сортировки задействовали диск из-за недостатка выделенной для сортировок памяти.
* 1-е моё решение через CTE + доп.индексы.
* 2-е моё решение через UNION ALL.
* Сначала тестировал разные варианты запросов, анализируя план, время, стоимость, порядок обхода, использование индексов. Секционировал методом by_hash таблицу tickets (Время только увеличивалось, что на SSD, что на HDD). Также пробовал на стенде с HDD отсаживать таблицу tickets в отдельное табличное пространство на отдельном диске, время также затрачивалось больше (скорее всего это связано с эмуляцией HDD на YC, т.к. на реальных HDD вероятнее всего, чтение было бы лучше с разных HDD).
* Самым тяжелым местом было соединение ride.fkbus = seat.fkbus, (оно же было не правильным), избавиться от которого для подсчета общего кол-ва посадочных мест решил в первом же запросе. плюс оптимизатор использовал мой первый доп.индекс (см.ниже). В итоге остановился на разбивке запроса на части, продумывая каждый следующий запрос

  --- 

Далее тесты:

----- Стенд с SSD -------
-------------------------

**Исходный запрос:**
---
```sql
explain analyze 
SELECT r.id, r.startdate as depart_date, bs.city || ', ' || bs.name as busstation, count(t.id) as order_place, count(st.id) as all_place
  FROM book.ride as r
  JOIN book.schedule as s    on r.fkschedule = s.id
  JOIN book.seat as st       on r.fkbus = st.fkbus
  JOIN book.busroute as br   on s.fkroute = br.id
  JOIN book.busstation as bs on br.fkbusstationfrom = bs.id
  JOIN book.tickets as t     on t.fkride = r.id
 GROUP BY r.id, r.startdate, bs.city || ', ' || bs.name 
 ORDER BY r.startdate

QUERY PLAN                                                                                                                                                                   |
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
Sort  (cost=64137403.40..64163653.40 rows=10500000 width=56) (actual time=494850.868..494909.358 rows=1500000 loops=1)                                                       |
  Sort Key: r.startdate                                                                                                                                                      |
  Sort Method: quicksort  Memory: 170238kB                                                                                                                                   |
  ->  GroupAggregate  (cost=81315.26..62702692.38 rows=10500000 width=56) (actual time=614.566..494399.765 rows=1500000 loops=1)                                             |
        Group Key: r.id, ((bs.city || ', '::text) || bs.name)                                                                                                                |
        ->  Nested Loop  (cost=81315.26..40946196.82 rows=2159899556 width=52) (actual time=614.069..357528.214 rows=2159899000 loops=1)                                     |
              ->  Merge Join  (cost=81315.11..2472983.17 rows=53997488 width=36) (actual time=614.020..32619.372 rows=53997475 loops=1)                                      |
                    Merge Cond: (r.id = t.fkride)                                                                                                                            |
                    ->  Gather Merge  (cost=81313.24..256012.96 rows=1500000 width=28) (actual time=613.401..1910.316 rows=1500000 loops=1)                                  |
                          Workers Planned: 2                                                                                                                                 |
                          Workers Launched: 2                                                                                                                                |
                          ->  Sort  (cost=80313.22..81875.72 rows=625000 width=28) (actual time=593.317..630.824 rows=500000 loops=3)                                        |
                                Sort Key: r.id, (((bs.city || ', '::text) || bs.name))                                                                                       |
                                Sort Method: quicksort  Memory: 55034kB                                                                                                      |
                                Worker 0:  Sort Method: quicksort  Memory: 74259kB                                                                                           |
                                Worker 1:  Sort Method: quicksort  Memory: 54040kB                                                                                           |
                                ->  Hash Join  (cost=49.33..20146.04 rows=625000 width=28) (actual time=216.568..497.943 rows=500000 loops=3)                                |
                                      Hash Cond: (br.fkbusstationfrom = bs.id)                                                                                               |
                                      ->  Hash Join  (cost=48.10..17808.88 rows=625000 width=16) (actual time=0.448..188.647 rows=500000 loops=3)                            |
                                            Hash Cond: (s.fkroute = br.id)                                                                                                   |
                                            ->  Hash Join  (cost=45.75..16050.01 rows=625000 width=16) (actual time=0.400..130.941 rows=500000 loops=3)                      |
                                                  Hash Cond: (r.fkschedule = s.id)                                                                                           |
                                                  ->  Parallel Seq Scan on ride r  (cost=0.00..14359.00 rows=625000 width=16) (actual time=0.021..59.760 rows=500000 loops=3)|
                                                  ->  Hash  (cost=27.00..27.00 rows=1500 width=8) (actual time=0.367..0.369 rows=1500 loops=3)                               |
                                                        Buckets: 2048  Batches: 1  Memory Usage: 75kB                                                                        |
                                                        ->  Seq Scan on schedule s  (cost=0.00..27.00 rows=1500 width=8) (actual time=0.015..0.201 rows=1500 loops=3)        |
                                            ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.037..0.038 rows=60 loops=3)                                           |
                                                  Buckets: 1024  Batches: 1  Memory Usage: 11kB                                                                              |
                                                  ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.025..0.029 rows=60 loops=3)                  |
                                      ->  Hash  (cost=1.10..1.10 rows=10 width=20) (actual time=216.042..216.042 rows=10 loops=3)                                            |
                                            Buckets: 1024  Batches: 1  Memory Usage: 9kB                                                                                     |
                                            ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=20) (actual time=216.020..216.025 rows=10 loops=3)                 |
                    ->  Index Scan using tickets_fkride_idx on tickets t  (cost=0.44..1538264.90 rows=53997488 width=12) (actual time=0.021..25017.006 rows=53997475 loops=1)|
              ->  Memoize  (cost=0.15..0.86 rows=40 width=8) (actual time=0.000..0.001 rows=40 loops=53997475)                                                               |
                    Cache Key: r.fkbus                                                                                                                                       |
                    Cache Mode: logical                                                                                                                                      |
                    Hits: 53997472  Misses: 3  Evictions: 0  Overflows: 0  Memory Usage: 5kB                                                                                 |
                    ->  Index Scan using seat_fkbus_idx on seat st  (cost=0.14..0.85 rows=40 width=8) (actual time=0.013..0.017 rows=40 loops=3)                             |
                          Index Cond: (fkbus = r.fkbus)                                                                                                                      |
Planning Time: 0.749 ms                                                                                                                                                      |
JIT:                                                                                                                                                                         |
  Functions: 100                                                                                                                                                             |
  Options: Inlining true, Optimization true, Expressions true, Deforming true                                                                                                |
  Timing: Generation 4.520 ms, Inlining 114.633 ms, Optimization 327.500 ms, Emission 206.554 ms, Total 653.208 ms                                                           |
Execution Time: 494954.648 ms                                                                                                                                                |
```

**Решение №1 стенд с SSD (CTE + с доп. индексы):**
---
- Сначала (q_one) - выбрал из таблицы ride все поездки с датой, ссылкой на план маршрута и здесь же сразу подсчитал общее кол-во мест в автобусе. ( в помощь первый доп. индекс Index Scan using **seat_fkbus_idx** on seat st)
- Далее (q_two) - выбрал все билеты на каждую поездку. ( в помощь второй доп.индекс **tickets_fkride_idx** Parallel Index Scan using tickets_fkride_idx)
- Далее (q_three) - добавил ко второму запросу ключ маршрута 
- Далее (q_four) - добавил ссылку на автостанцию и подсчитал кол-во проданных билетов (count(ticket.id)), без конкатенации и, тем более, её группировки.
- В последнем результирующем запросе добавил конкатенацию Город ||', '|| Автовокзал и отсортировал по дате выезда.
В процессе тестирования подзапросов пробовал подключать доп.индексы, оставил только два реально помогающих:
```q
create index CONCURRENTLY seat_fkbus_idx on book.seat (fkbus);
create index CONCURRENTLY tickets_fkride_idx on book.tickets (fkride);
```


``` sql 

-- Создание двух доп.индексов
create index CONCURRENTLY seat_fkbus_idx on book.seat (fkbus);
create index CONCURRENTLY tickets_fkride_idx on book.tickets (fkride);


explain analyze
with
q_one as
(select r.id as ride_id, r.startdate as depart_date, r.fkschedule, count(st.id) as all_place 
   from book.ride r
   join book.seat st on r.fkbus = st.fkbus
  group by r.id, r.startdate, r.fkschedule /*4.46*/
),
q_two as
(select r.ride_id, r.depart_date, r.all_place, r.fkschedule, t.id as ticket_id  
   from book.tickets as t
   join q_one r on t.fkride = r.ride_id /*26.4*/
),
q_three as
(select s.fkroute, tw.ride_id, tw.depart_date, tw.all_place, tw.ticket_id
   FROM book.schedule as s
   join q_two as tw on s.id = tw.fkschedule /*32.3*/
),
q_four as
(select th.ride_id as id, th.depart_date, br.fkbusstationfrom, 
        th.all_place,
	    count(th.ticket_id) as order_place
   from book.busroute as br 
   join q_three th on br.id = th.fkroute
  group by th.ride_id, th.depart_date, br.fkbusstationfrom, th.all_place 
)  
select f.id, f.depart_date, 
       CONCAT(bs.city, ', ', bs."name") as busstation, 
       f.order_place, f.all_place
  from book.busstation bs
  join q_four f on bs.id = f.fkbusstationfrom  
 order by f.depart_date; 

QUERY PLAN                                                                                                                                                                                                   |
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
Sort  (cost=3560944.75..3561132.25 rows=75000 width=56) (actual time=30790.133..31037.933 rows=1500000 loops=1)                                                                                              |
  Sort Key: r.startdate                                                                                                                                                                                      |
  Sort Method: quicksort  Memory: 170238kB                                                                                                                                                                   |
  ->  Hash Join  (cost=3520663.33..3554871.77 rows=75000 width=56) (actual time=29883.878..30551.980 rows=1500000 loops=1)                                                                                   |
        Hash Cond: (br.fkbusstationfrom = bs.id)                                                                                                                                                             |
        ->  Finalize HashAggregate  (cost=3520662.11..3535662.11 rows=1500000 width=28) (actual time=29586.564..29894.742 rows=1500000 loops=1)                                                              |
              Group Key: r.id, r.startdate, br.fkbusstationfrom, (count(st.id))                                                                                                                              |
              Batches: 1  Memory Usage: 270353kB                                                                                                                                                             |
              ->  Gather  (cost=3168162.11..3483162.11 rows=3000000 width=28) (actual time=28906.772..29117.576 rows=1500000 loops=1)                                                                        |
                    Workers Planned: 2                                                                                                                                                                       |
                    Workers Launched: 2                                                                                                                                                                      |
                    ->  Partial HashAggregate  (cost=3167162.11..3182162.11 rows=1500000 width=28) (actual time=28867.561..28992.362 rows=500000 loops=3)                                                    |
                          Group Key: r.id, r.startdate, br.fkbusstationfrom, (count(st.id))                                                                                                                  |
                          Batches: 1  Memory Usage: 106513kB                                                                                                                                                 |
                          Worker 0:  Batches: 1  Memory Usage: 106521kB                                                                                                                                      |
                          Worker 1:  Batches: 1  Memory Usage: 106521kB                                                                                                                                      |
                          ->  Hash Join  (cost=49.12..2881175.52 rows=22878927 width=28) (actual time=161.015..26136.790 rows=17999158 loops=3)                                                              |
                                Hash Cond: (s.fkroute = br.id)                                                                                                                                               |
                                ->  Hash Join  (cost=46.77..2816873.86 rows=22878927 width=28) (actual time=14.393..23968.892 rows=17999158 loops=3)                                                         |
                                      Hash Cond: (r.fkschedule = s.id)                                                                                                                                       |
                                      ->  Merge Join  (cost=1.02..2756601.23 rows=22878927 width=28) (actual time=13.985..21725.547 rows=17999158 loops=3)                                                   |
                                            Merge Cond: (t.fkride = r.id)                                                                                                                                    |
                                            ->  Parallel Index Scan using tickets_fkride_idx on tickets t  (cost=0.44..1223279.55 rows=22498953 width=12) (actual time=0.143..8156.867 rows=17999158 loops=3)|
                                            ->  Materialize  (cost=0.58..1137640.41 rows=1500000 width=20) (actual time=0.113..10527.255 rows=18999132 loops=3)                                              |
                                                  ->  GroupAggregate  (cost=0.58..1118890.41 rows=1500000 width=20) (actual time=0.110..9568.063 rows=1499986 loops=3)                                       |
                                                        Group Key: r.id                                                                                                                                      |
                                                        ->  Nested Loop  (cost=0.58..803890.41 rows=60000001 width=16) (actual time=0.084..6564.653 rows=59999454 loops=3)                                   |
                                                              ->  Index Scan using ride_pkey on ride r  (cost=0.43..35137.13 rows=1500000 width=16) (actual time=0.041..303.078 rows=1499987 loops=3)        |
                                                              ->  Memoize  (cost=0.15..0.86 rows=40 width=8) (actual time=0.000..0.001 rows=40 loops=4499961)                                                |
                                                                    Cache Key: r.fkbus                                                                                                                       |
                                                                    Cache Mode: logical                                                                                                                      |
                                                                    Hits: 1499993  Misses: 3  Evictions: 0  Overflows: 0  Memory Usage: 5kB                                                                  |
                                                                    Worker 0:  Hits: 1499962  Misses: 3  Evictions: 0  Overflows: 0  Memory Usage: 5kB                                                       |
                                                                    Worker 1:  Hits: 1499997  Misses: 3  Evictions: 0  Overflows: 0  Memory Usage: 5kB                                                       |
                                                                    ->  Index Scan using seat_fkbus_idx on seat st  (cost=0.14..0.85 rows=40 width=8) (actual time=0.012..0.017 rows=40 loops=9)             |
                                                                          Index Cond: (fkbus = r.fkbus)                                                                                                      |
                                      ->  Hash  (cost=27.00..27.00 rows=1500 width=8) (actual time=0.389..0.390 rows=1500 loops=3)                                                                           |
                                            Buckets: 2048  Batches: 1  Memory Usage: 75kB                                                                                                                    |
                                            ->  Seq Scan on schedule s  (cost=0.00..27.00 rows=1500 width=8) (actual time=0.032..0.236 rows=1500 loops=3)                                                    |
                                ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=146.603..146.603 rows=60 loops=3)                                                                                   |
                                      Buckets: 1024  Batches: 1  Memory Usage: 11kB                                                                                                                          |
                                      ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=146.578..146.584 rows=60 loops=3)                                                          |
        ->  Hash  (cost=1.10..1.10 rows=10 width=20) (actual time=297.292..297.293 rows=10 loops=1)                                                                                                          |
              Buckets: 1024  Batches: 1  Memory Usage: 9kB                                                                                                                                                   |
              ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=20) (actual time=297.273..297.279 rows=10 loops=1)                                                                               |
Planning Time: 0.544 ms                                                                                                                                                                                      |
JIT:                                                                                                                                                                                                         |
  Functions: 139                                                                                                                                                                                             |
  Options: Inlining true, Optimization true, Expressions true, Deforming true                                                                                                                                |
  Timing: Generation 5.322 ms, Inlining 110.745 ms, Optimization 365.051 ms, Emission 261.380 ms, Total 742.497 ms                                                                                           |
Execution Time: 31095.827 ms                                                                                                                                                                                 |                                                                                                                                                                              |                                                                                                                                                                               |

```

**Вывод по SSD (CTE с доп.индексами):**
---

**Исходый запрос:**


(cost=64 137 403.40..64 163 653.40 actual time=494 850.868..494 909.358 rows=1 500 000)

**Оптимизированный запрос:**

(cost=3 560 944.75..3 561 132.25 actual time=30 790.133..31 037.933 rows=1 500 000)

Сокращение времени выполнения 16 - кратное, по стоимости 18 кратное снижение стоимости.



----- Стенд с HDD -------
---

**Исходный запрос:**
---
```sql
explain analyze 
SELECT r.id, r.startdate as depart_date, bs.city || ', ' || bs.name as busstation, count(t.id) as order_place, count(st.id) as all_place
  FROM book.ride as r
  JOIN book.schedule as s    on r.fkschedule = s.id
  JOIN book.seat as st       on r.fkbus = st.fkbus
  JOIN book.busroute as br   on s.fkroute = br.id
  JOIN book.busstation as bs on br.fkbusstationfrom = bs.id
  JOIN book.tickets as t     on t.fkride = r.id
 GROUP BY r.id, r.startdate, bs.city || ', ' || bs.name 
 ORDER BY r.startdate

                                                                                  QUERY PLAN                                            
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=66205753.29..66232003.29 rows=10500000 width=56) (actual time=506195.079..506257.456 rows=1500000 loops=1)
   Sort Key: r.startdate
   Sort Method: quicksort  Memory: 170238kB
   ->  GroupAggregate  (cost=81320.49..64622359.27 rows=10500000 width=56) (actual time=948.428..505806.898 rows=1500000 loops=1)
         Group Key: r.id, ((bs.city || ', '::text) || bs.name)
         ->  Nested Loop  (cost=81320.49..42865942.99 rows=2159891628 width=52) (actual time=947.876..371666.688 rows=2159899000 loops=1)
               ->  Merge Join  (cost=81320.34..4392871.48 rows=53997288 width=36) (actual time=947.837..31393.524 rows=53997475 loops=1)
                     Merge Cond: (r.id = t.fkride)
                     ->  Gather Merge  (cost=81313.24..256012.96 rows=1500000 width=28) (actual time=947.805..1343.655 rows=1500000 loops=1)
                           Workers Planned: 2
                           Workers Launched: 2
                           ->  Sort  (cost=80313.22..81875.72 rows=625000 width=28) (actual time=931.126..977.035 rows=500000 loops=3)
                                 Sort Key: r.id, (((bs.city || ', '::text) || bs.name))
                                 Sort Method: quicksort  Memory: 40984kB
                                 Worker 0:  Sort Method: quicksort  Memory: 42523kB
                                 Worker 1:  Sort Method: quicksort  Memory: 99826kB
                                 ->  Hash Join  (cost=49.33..20146.04 rows=625000 width=28) (actual time=349.012..757.794 rows=500000 loops=3)
                                       Hash Cond: (br.fkbusstationfrom = bs.id)
                                       ->  Hash Join  (cost=48.10..17808.88 rows=625000 width=16) (actual time=0.659..267.213 rows=500000 loops=3)
                                             Hash Cond: (s.fkroute = br.id)
                                             ->  Hash Join  (cost=45.75..16050.01 rows=625000 width=16) (actual time=0.587..177.059 rows=500000 loops=3)
                                                   Hash Cond: (r.fkschedule = s.id)
                                                   ->  Parallel Seq Scan on ride r  (cost=0.00..14359.00 rows=625000 width=16) (actual time=0.026..67.368 rows=500000 loops=3)
                                                   ->  Hash  (cost=27.00..27.00 rows=1500 width=8) (actual time=0.544..0.546 rows=1500 loops=3)
                                                         Buckets: 2048  Batches: 1  Memory Usage: 75kB
                                                         ->  Seq Scan on schedule s  (cost=0.00..27.00 rows=1500 width=8) (actual time=0.014..0.252 rows=1500 loops=3)
                                             ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.054..0.055 rows=60 loops=3)
                                                   Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                                   ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.032..0.039 rows=60 loops=3)
                                       ->  Hash  (cost=1.10..1.10 rows=10 width=20) (actual time=348.283..348.284 rows=10 loops=3)
                                             Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                             ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=20) (actual time=348.253..348.259 rows=10 loops=3)
                     ->  Index Scan using tickets_fkride_idx on tickets t  (cost=0.44..3458145.27 rows=53997288 width=12) (actual time=0.020..23739.737 rows=53997475 loops=1)
               ->  Memoize  (cost=0.15..0.86 rows=40 width=8) (actual time=0.000..0.001 rows=40 loops=53997475)
                     Cache Key: r.fkbus
                     Cache Mode: logical
                     Hits: 53997472  Misses: 3  Evictions: 0  Overflows: 0  Memory Usage: 5kB
                     ->  Index Scan using seat_fkbus_idx on seat st  (cost=0.14..0.85 rows=40 width=8) (actual time=0.009..0.016 rows=40 loops=3)
                           Index Cond: (fkbus = r.fkbus)
 Planning Time: 0.822 ms
 JIT:
   Functions: 100
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 4.635 ms, Inlining 163.272 ms, Optimization 518.351 ms, Emission 363.214 ms, Total 1049.472 ms
 Execution Time: 506313.658 ms
(45 rows)
```


**Решение №1 HDD (CTE + доп.индексы):**
---
```sql

create index CONCURRENTLY seat_fkbus_idx on book.seat (fkbus);
create index CONCURRENTLY tickets_fkride_idx on book.tickets (fkride);

explain analyze
with
q_one as
(select r.id as ride_id, r.startdate as depart_date, r.fkschedule, count(st.id) as all_place 
   from book.ride r
   join book.seat st on r.fkbus = st.fkbus
  group by r.id, r.startdate, r.fkschedule
),
q_two as
(select r.ride_id, r.depart_date, r.all_place, r.fkschedule, t.id as ticket_id  
   from book.tickets as t
   join q_one r on t.fkride = r.ride_id
),
q_three as
(select s.fkroute, tw.ride_id, tw.depart_date, tw.all_place, tw.ticket_id
   FROM book.schedule as s
   join q_two as tw on s.id = tw.fkschedule
),
q_four as
(select th.ride_id as id, th.depart_date, br.fkbusstationfrom, 
        th.all_place, count(th.ticket_id) as order_place
   from book.busroute as br 
   join q_three th on br.id = th.fkroute
  group by th.ride_id, th.depart_date, br.fkbusstationfrom, th.all_place 
)  
select f.id, f.depart_date, 
       CONCAT(bs.city, ', ', bs."name") as busstation, 
       f.order_place, f.all_place
  from book.busstation bs
  join q_four f on bs.id = f.fkbusstationfrom  
 order by f.depart_date;

                                                                                     QUERY PLAN                                         
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=4526250.27..4526437.77 rows=75000 width=56) (actual time=49081.757..49312.201 rows=1500000 loops=1)
   Sort Key: r.startdate
   Sort Method: quicksort  Memory: 170236kB
   ->  Hash Join  (cost=4485968.85..4520177.29 rows=75000 width=56) (actual time=47887.184..48702.891 rows=1500000 loops=1)
         Hash Cond: (br.fkbusstationfrom = bs.id)
         ->  HashAggregate  (cost=4485967.63..4500967.63 rows=1500000 width=28) (actual time=47559.602..47943.707 rows=1500000 loops=1)
               Group Key: r.id, r.startdate, br.fkbusstationfrom, (count(st.id))
               Batches: 1  Memory Usage: 221201kB
               ->  Hash Join  (cost=2845427.09..3810335.80 rows=54050546 width=28) (actual time=19318.945..39722.027 rows=53997475 loops=1)
                     Hash Cond: (r.id = t.fkride)
                     ->  Hash Join  (cost=753226.11..791390.36 rows=1500000 width=20) (actual time=6143.292..6910.084 rows=1500000 loops=1)
                           Hash Cond: (s.fkroute = br.id)
                           ->  Hash Join  (cost=753223.77..787172.39 rows=1500000 width=20) (actual time=6143.254..6729.194 rows=1500000 loops=1)
                                 Hash Cond: (r.fkschedule = s.id)
                                 ->  Finalize HashAggregate  (cost=753178.02..768178.02 rows=1500000 width=20) (actual time=6142.953..6526.990 rows=1500000 loops=1)
                                       Group Key: r.id
                                       Batches: 1  Memory Usage: 172049kB
                                       ->  Gather  (cost=423178.02..738178.02 rows=3000000 width=20) (actual time=5423.185..5614.605 rows=1500000 loops=1)
                                             Workers Planned: 2
                                             Workers Launched: 2
                                             ->  Partial HashAggregate  (cost=422178.02..437178.02 rows=1500000 width=20) (actual time=5389.881..5507.970 rows=500000 loops=3)
                                                   Group Key: r.id
                                                   Batches: 1  Memory Usage: 81937kB
                                                   Worker 0:  Batches: 1  Memory Usage: 81937kB
                                                   Worker 1:  Batches: 1  Memory Usage: 106513kB
                                                   ->  Hash Join  (cost=6.50..297178.01 rows=25000001 width=16) (actual time=124.482..2220.150 rows=20000000 loops=3)
                                                         Hash Cond: (r.fkbus = st.fkbus)
                                                         ->  Parallel Seq Scan on ride r  (cost=0.00..14359.00 rows=625000 width=16) (actual time=0.008..59.106 rows=500000 loops=3)
                                                         ->  Hash  (cost=4.00..4.00 rows=200 width=8) (actual time=124.454..124.456 rows=200 loops=3)
                                                               Buckets: 1024  Batches: 1  Memory Usage: 16kB
                                                               ->  Seq Scan on seat st  (cost=0.00..4.00 rows=200 width=8) (actual time=124.386..124.411 rows=200 loops=3)
                                 ->  Hash  (cost=27.00..27.00 rows=1500 width=8) (actual time=0.288..0.289 rows=1500 loops=1)
                                       Buckets: 2048  Batches: 1  Memory Usage: 75kB
                                       ->  Seq Scan on schedule s  (cost=0.00..27.00 rows=1500 width=8) (actual time=0.007..0.126 rows=1500 loops=1)
                           ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.025..0.026 rows=60 loops=1)
                                 Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                 ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.012..0.017 rows=60 loops=1)
                     ->  Hash  (cost=1153575.88..1153575.88 rows=53997288 width=12) (actual time=13169.968..13169.968 rows=53997475 loops=1)
                           Buckets: 8388608  Batches: 8  Memory Usage: 355686kB
                           ->  Seq Scan on tickets t  (cost=0.00..1153575.88 rows=53997288 width=12) (actual time=0.076..5137.166 rows=53997475 loops=1)
         ->  Hash  (cost=1.10..1.10 rows=10 width=20) (actual time=327.561..327.562 rows=10 loops=1)
               Buckets: 1024  Batches: 1  Memory Usage: 9kB
               ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=20) (actual time=327.537..327.542 rows=10 loops=1)
 Planning Time: 0.483 ms
 JIT:
   Functions: 90
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 4.442 ms, Inlining 141.279 ms, Optimization 324.746 ms, Emission 234.807 ms, Total 705.274 ms
 Execution Time: 49419.974 ms
(49 rows)
```
**Вывод по HDD (CTE + доп.индексы):**
---

**Исходный запрос:**

(cost=66 205 753.29..66 232 003.29 actual time=506 195.079..506 257.456 rows=1 500 000)

**Оптимизированный запрос:**

(cost=4 526 250.27..4 526 437.77 actual time=49 081.757..49 312.201 rows=1 500 000)

Сокращение времени выполнения 10 - кратное, по стоимости 14.6 - ти кратное снижение стоимости.


----

**Решение №2 с использованием запросов с UNION c SSD диском - получился еще быстрее предыдущего**
---
* 0 запрос - Выбираю кол-во проданных билетов по каждому рейсу, группировкой по ссылке на рейс.
* 1 запрос - Выбираю общее число посадочных мест в каждом рейсе, ключ каждого рейса и ссылку на маршрут.
* 2 запрос - Выбираю маршруты со ссылками на место отправления.
* 3 запрос - Выбираю всёё из первого запроса + ссылку на таблицу Автовокзалов.
* 4 запрос - Выбираю всё из нулевого запроса UNION ALL всё из 3 запроса.
* 5 запрос - группировка по рейсам и подсчет итогов по 4 запросу.
* 6 итоговый запрос - добавляю к результату пятого запроса сведения о городе + автовокзале отбытия и провожу сортировку по дате выезда.


* Поднял настройки work_mem = 200MB вместо 55MB исходных. т.к. следующему запросу не хватало выделенной RAM для быстрой сортировки в памяти.
```sql
-- без поднятия work_mem сортировке не хватало памяти, было так Sort Method: external merge  Disk: 67704kB   

explain analyze
with
q_null as
(select t2.fkride as ride_id, date'1900-01-01' as depart_date, 0::numeric as nbusstation, 
        count(t2.id) as order_place, 0::numeric as all_place
   from book.tickets t2 
  group by t2.fkride
),
q_one as
(select r.id as ride_id, r.startdate as depart_date, r.fkschedule, count(st.id) as all_place
   from book.ride r
   join book.seat st on r.fkbus = st.fkbus
  group by r.id, r.startdate, r.fkschedule
),  
q_two as 
(select s.id, br.fkbusstationfrom as nbusstation
   FROM book.schedule as s
   join book.busroute as br on s.fkroute = br.id
),
q_three as
(select o.ride_id, o.depart_date, t.nbusstation, 0::numeric as order_place, o.all_place
  from q_one o
  join q_two t on o.fkschedule = t.id
),
q_four as 
(select * from q_null qn
union all
select * from q_three qt
), q_five as
(select q.ride_id, 
        max(q.depart_date) as depart_date, 
        max(q.nbusstation) as nbusstation, 
        sum(q.order_place) as order_place, 
        sum(q.all_place) as all_place
  from q_four q
 group by q.ride_id
)
select q.ride_id, q.depart_date, q.order_place, 
       q.all_place, concat(b.city, ', ',b."name") as busstation
  from q_five q
  join book.busstation b on q.nbusstation = b.id 
  order by q.depart_date;
  
  QUERY PLAN                                                                                                                                                                               |
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
Sort  (cost=2220258.80..2220258.83 rows=10 width=104) (actual time=16042.159..16251.034 rows=1500000 loops=1)                                                                            |
  Sort Key: (max("*SELECT* 1".depart_date))                                                                                                                                              |
  Sort Method: quicksort  Memory: 165397kB                                                                                                                                               |
  ->  Hash Join  (cost=2220252.51..2220258.64 rows=10 width=104) (actual time=14402.380..15750.355 rows=1500000 loops=1)                                                                 |
        Hash Cond: ((max("*SELECT* 1".nbusstation)) = (b.id)::numeric)                                                                                                                   |
        ->  HashAggregate  (cost=2220251.29..2220254.29 rows=200 width=104) (actual time=14085.147..14959.383 rows=1500000 loops=1)                                                      |
              Group Key: "*SELECT* 1".ride_id                                                                                                                                            |
              Batches: 5  Memory Usage: 819265kB  Disk Usage: 40584kB                                                                                                                    |
              ->  Append  (cost=1276605.66..2153311.81 rows=2975088 width=104) (actual time=7723.192..12956.887 rows=3000000 loops=1)                                                    |
                    ->  Subquery Scan on "*SELECT* 1"  (cost=1276605.66..1324546.02 rows=1475088 width=104) (actual time=7723.190..8221.153 rows=1500000 loops=1)                        |
                          ->  Finalize HashAggregate  (cost=1276605.66..1291356.54 rows=1475088 width=80) (actual time=7723.185..8051.737 rows=1500000 loops=1)                          |
                                Group Key: t2.fkride                                                                                                                                     |
                                Batches: 1  Memory Usage: 172049kB                                                                                                                       |
                                ->  Gather  (cost=952086.30..1261854.78 rows=2950176 width=12) (actual time=5916.979..6485.417 rows=4500000 loops=1)                                     |
                                      Workers Planned: 2                                                                                                                                 |
                                      Workers Launched: 2                                                                                                                                |
                                      ->  Partial HashAggregate  (cost=951086.30..965837.18 rows=1475088 width=12) (actual time=5885.491..6201.512 rows=1500000 loops=3)                 |
                                            Group Key: t2.fkride                                                                                                                         |
                                            Batches: 1  Memory Usage: 172049kB                                                                                                           |
                                            Worker 0:  Batches: 1  Memory Usage: 172049kB                                                                                                |
                                            Worker 1:  Batches: 1  Memory Usage: 172049kB                                                                                                |
                                            ->  Parallel Seq Scan on tickets t2  (cost=0.00..838591.53 rows=22498953 width=12) (actual time=0.075..1579.616 rows=17999158 loops=3)       |
                    ->  Subquery Scan on "*SELECT* 2"  (cost=753226.10..813890.35 rows=1500000 width=104) (actual time=3739.377..4565.066 rows=1500000 loops=1)                          |
                          ->  Hash Join  (cost=753226.10..791390.35 rows=1500000 width=52) (actual time=3739.374..4352.765 rows=1500000 loops=1)                                         |
                                Hash Cond: (s.fkroute = br.id)                                                                                                                           |
                                ->  Hash Join  (cost=753223.75..787172.38 rows=1500000 width=20) (actual time=3739.296..4191.912 rows=1500000 loops=1)                                   |
                                      Hash Cond: (r.fkschedule = s.id)                                                                                                                   |
                                      ->  Finalize HashAggregate  (cost=753178.00..768178.00 rows=1500000 width=20) (actual time=3739.041..4010.625 rows=1500000 loops=1)                |
                                            Group Key: r.id                                                                                                                              |
                                            Batches: 1  Memory Usage: 172049kB                                                                                                           |
                                            ->  Gather  (cost=423178.00..738178.00 rows=3000000 width=20) (actual time=3107.172..3299.312 rows=1500000 loops=1)                          |
                                                  Workers Planned: 2                                                                                                                     |
                                                  Workers Launched: 2                                                                                                                    |
                                                  ->  Partial HashAggregate  (cost=422178.00..437178.00 rows=1500000 width=20) (actual time=3076.614..3179.386 rows=500000 loops=3)      |
                                                        Group Key: r.id                                                                                                                  |
                                                        Batches: 1  Memory Usage: 90129kB                                                                                                |
                                                        Worker 0:  Batches: 1  Memory Usage: 90129kB                                                                                     |
                                                        Worker 1:  Batches: 1  Memory Usage: 90129kB                                                                                     |
                                                        ->  Hash Join  (cost=6.50..297178.00 rows=25000000 width=16) (actual time=76.663..1334.944 rows=20000000 loops=3)                |
                                                              Hash Cond: (r.fkbus = st.fkbus)                                                                                            |
                                                              ->  Parallel Seq Scan on ride r  (cost=0.00..14359.00 rows=625000 width=16) (actual time=0.011..37.279 rows=500000 loops=3)|
                                                              ->  Hash  (cost=4.00..4.00 rows=200 width=8) (actual time=76.638..76.639 rows=200 loops=3)                                 |
                                                                    Buckets: 1024  Batches: 1  Memory Usage: 16kB                                                                        |
                                                                    ->  Seq Scan on seat st  (cost=0.00..4.00 rows=200 width=8) (actual time=76.582..76.601 rows=200 loops=3)            |
                                      ->  Hash  (cost=27.00..27.00 rows=1500 width=8) (actual time=0.244..0.245 rows=1500 loops=1)                                                       |
                                            Buckets: 2048  Batches: 1  Memory Usage: 75kB                                                                                                |
                                            ->  Seq Scan on schedule s  (cost=0.00..27.00 rows=1500 width=8) (actual time=0.008..0.110 rows=1500 loops=1)                                |
                                ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.043..0.044 rows=60 loops=1)                                                                   |
                                      Buckets: 1024  Batches: 1  Memory Usage: 11kB                                                                                                      |
                                      ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.030..0.034 rows=60 loops=1)                                          |
        ->  Hash  (cost=1.10..1.10 rows=10 width=20) (actual time=317.216..317.217 rows=10 loops=1)                                                                                      |
              Buckets: 1024  Batches: 1  Memory Usage: 9kB                                                                                                                               |
              ->  Seq Scan on busstation b  (cost=0.00..1.10 rows=10 width=20) (actual time=317.196..317.200 rows=10 loops=1)                                                            |
Planning Time: 0.377 ms                                                                                                                                                                  |
JIT:                                                                                                                                                                                     |
  Functions: 113                                                                                                                                                                         |
  Options: Inlining true, Optimization true, Expressions true, Deforming true                                                                                                            |
  Timing: Generation 5.409 ms, Inlining 198.155 ms, Optimization 337.672 ms, Emission 248.309 ms, Total 789.545 ms                                                                       |
Execution Time: 16410.901 ms                                                                                                                                                             |
  
```
**Вывод по SSD (CTE - запросы с UNION):**
---
**Исходый запрос:**

(cost=79 981 274.91..80 007 524.91 actual time=573 957.860..574 076.002 rows=1 500 000)   

**Оптимизированный запрос**

(cost=2 220 258.80..2 220 258.83 actual time=16 042.159..16 251.034 rows=1 500 000)

Сокращение времени выполнения 35 - кратное, по стоимости 36 кратное снижение стоимости.


**Решение №2 с использованием запросов с UNION c HDD диском - тоже вышло достаточно хорошо**
---

```sql

explain analyze
with
q_null as
(select t2.fkride as ride_id, date'1900-01-01' as depart_date, 0::numeric as nbusstation, 
        count(t2.id) as order_place, 0::numeric as all_place
   from book.tickets t2 
  group by t2.fkride
),
q_one as
(select r.id as ride_id, r.startdate as depart_date, r.fkschedule, count(st.id) as all_place
   from book.ride r
   join book.seat st on r.fkbus = st.fkbus
  group by r.id, r.startdate, r.fkschedule
),  
q_two as 
(select s.id, br.fkbusstationfrom as nbusstation
   FROM book.schedule as s
   join book.busroute as br on s.fkroute = br.id
),
q_three as
(select o.ride_id, o.depart_date, t.nbusstation, 0::numeric as order_place, o.all_place
  from q_one o
  join q_two t on o.fkschedule = t.id
),
q_four as 
(select * from q_null qn
union all
select * from q_three qt
), q_five as
(select q.ride_id, 
        max(q.depart_date) as depart_date, 
        max(q.nbusstation) as nbusstation, 
        sum(q.order_place) as order_place, 
        sum(q.all_place) as all_place
  from q_four q
 group by q.ride_id
)
select q.ride_id, q.depart_date, q.order_place, 
       q.all_place, concat(b.city, ', ',b."name") as busstation
  from q_five q
  join book.busstation b on q.nbusstation = b.id 
  order by q.depart_date;

                                                                                        QUERY PLAN                                      
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=2226820.09..2226820.12 rows=10 width=104) (actual time=22118.558..22346.421 rows=1500000 loops=1)
   Sort Key: (max("*SELECT* 1".depart_date))
   Sort Method: quicksort  Memory: 165397kB
   ->  Hash Join  (cost=2226813.80..2226819.93 rows=10 width=104) (actual time=19714.797..21767.791 rows=1500000 loops=1)
         Hash Cond: ((max("*SELECT* 1".nbusstation)) = (b.id)::numeric)
         ->  HashAggregate  (cost=2226812.58..2226815.58 rows=200 width=104) (actual time=19332.990..20777.868 rows=1500000 loops=1)
               Group Key: "*SELECT* 1".ride_id
               Batches: 5  Memory Usage: 409665kB  Disk Usage: 81016kB
               ->  Append  (cost=1281760.89..2159345.83 rows=2998522 width=104) (actual time=10299.863..18369.278 rows=3000000 loops=1)
                     ->  Subquery Scan on "*SELECT* 1"  (cost=1281760.89..1330462.85 rows=1498522 width=104) (actual time=10299.861..10830.313 rows=1500000 loops=1)
                           ->  Finalize HashAggregate  (cost=1281760.89..1296746.11 rows=1498522 width=80) (actual time=10299.855..10637.482 rows=1500000 loops=1)
                                 Group Key: t2.fkride
                                 Batches: 1  Memory Usage: 172049kB
                                 ->  Gather  (cost=952086.05..1266775.67 rows=2997044 width=12) (actual time=8189.595..8808.518 rows=4499991 loops=1)
                                       Workers Planned: 2
                                       Workers Launched: 2
                                       ->  Partial HashAggregate  (cost=951086.05..966071.27 rows=1498522 width=12) (actual time=8157.997..8477.521 rows=1499997 loops=3)
                                             Group Key: t2.fkride
                                             Batches: 1  Memory Usage: 221201kB
                                             Worker 0:  Batches: 1  Memory Usage: 172049kB
                                             Worker 1:  Batches: 1  Memory Usage: 172049kB
                                             ->  Parallel Seq Scan on tickets t2  (cost=0.00..838591.70 rows=22498870 width=12) (actual time=0.044..2252.037 rows=17999158 loops=3)
                     ->  Subquery Scan on "*SELECT* 2"  (cost=753226.11..813890.36 rows=1500000 width=104) (actual time=6393.430..7361.503 rows=1500000 loops=1)
                           ->  Hash Join  (cost=753226.11..791390.36 rows=1500000 width=52) (actual time=6393.424..7097.137 rows=1500000 loops=1)
                                 Hash Cond: (s.fkroute = br.id)
                                 ->  Hash Join  (cost=753223.77..787172.39 rows=1500000 width=20) (actual time=6393.338..6901.143 rows=1500000 loops=1)
                                       Hash Cond: (r.fkschedule = s.id)
                                       ->  Finalize HashAggregate  (cost=753178.02..768178.02 rows=1500000 width=20) (actual time=6393.031..6705.047 rows=1500000 loops=1)
                                             Group Key: r.id
                                             Batches: 1  Memory Usage: 172049kB
                                             ->  Gather  (cost=423178.02..738178.02 rows=3000000 width=20) (actual time=5646.599..5837.922 rows=1500000 loops=1)
                                                   Workers Planned: 2
                                                   Workers Launched: 2
                                                   ->  Partial HashAggregate  (cost=422178.02..437178.02 rows=1500000 width=20) (actual time=5614.626..5728.801 rows=500000 loops=3)
                                                         Group Key: r.id
                                                         Batches: 1  Memory Usage: 81937kB
                                                         Worker 0:  Batches: 1  Memory Usage: 81937kB
                                                         Worker 1:  Batches: 1  Memory Usage: 106513kB
                                                         ->  Hash Join  (cost=6.50..297178.01 rows=25000001 width=16) (actual time=128.130..2330.713 rows=20000000 loops=3)
                                                               Hash Cond: (r.fkbus = st.fkbus)
                                                               ->  Parallel Seq Scan on ride r  (cost=0.00..14359.00 rows=625000 width=16) (actual time=0.012..62.893 rows=500000 loops=3)
                                                               ->  Hash  (cost=4.00..4.00 rows=200 width=8) (actual time=128.098..128.099 rows=200 loops=3)
                                                                     Buckets: 1024  Batches: 1  Memory Usage: 16kB
                                                                     ->  Seq Scan on seat st  (cost=0.00..4.00 rows=200 width=8) (actual time=128.014..128.047 rows=200 loops=3)
                                       ->  Hash  (cost=27.00..27.00 rows=1500 width=8) (actual time=0.293..0.294 rows=1500 loops=1)
                                             Buckets: 2048  Batches: 1  Memory Usage: 75kB
                                             ->  Seq Scan on schedule s  (cost=0.00..27.00 rows=1500 width=8) (actual time=0.010..0.136 rows=1500 loops=1)
                                 ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.050..0.051 rows=60 loops=1)
                                       Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                       ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.033..0.038 rows=60 loops=1)
         ->  Hash  (cost=1.10..1.10 rows=10 width=20) (actual time=381.786..381.788 rows=10 loops=1)
               Buckets: 1024  Batches: 1  Memory Usage: 9kB
               ->  Seq Scan on busstation b  (cost=0.00..1.10 rows=10 width=20) (actual time=381.759..381.765 rows=10 loops=1)
 Planning Time: 0.483 ms
 JIT:
   Functions: 113
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 6.767 ms, Inlining 306.355 ms, Optimization 469.674 ms, Emission 327.030 ms, Total 1109.826 ms
 Execution Time: 22439.049 ms
(59 rows)
```

**Вывод по HDD (CTE - запросы с UNION):**
---
**Исходный запрос:**

(cost=80 590 536.40..80 616 786.40 actual time=637 070.144..637 197.838 rows=1 500 000)

**Оптимизированный запрос:**

(cost=2 226 820.09..2 226 820.12 actual time=22 118.558..22 346.421 rows=1 500 000)

Сокращение времени выполнения 28.5 - кратное, по стоимости 36 - ти кратное снижение стоимости.





**Доп.информация по секционированию в процессе тестирования, от которого отказался в итоге:**
```sql
create schema by_hash;

 CREATE TABLE by_hash.tickets
	(
	id      bigserial NOT NULL,
	fkride  int4 NULL,
	fio     text NULL,
	contact jsonb NULL,
	fkseat  int4 null,
	CONSTRAINT tickets_pkey PRIMARY KEY (id)
	) PARTITION BY HASH (id);

-- Создание секций
do 
$$
declare query text;
begin
  FOR i IN 0 .. 29
  LOOP
    query = format ($fmt$
                       CREATE TABLE by_hash.tickets_%s
                       PARTITION OF by_hash.tickets
                       FOR VALUES WITH (MODULUS 30, REMAINDER %s);
                    $fmt$, i, i);
    RAISE NOTICE '%', query;
    EXECUTE query;
  END LOOP;	
end;
$$
-- Наполнение таблицы
insert into by_hash.tickets 
(select * from book.tickets order by id);
```