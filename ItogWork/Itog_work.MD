# Итоговая работа по курсу.
## Цель - провести оптимизацию аналитического запроса по кол-ву пассажирских перевозок, общим числом мест на рейс и числом проданных билетов на каждую поездку, с информацией о месте отправления (Город - автовокзал) на базе с 60 млн.записей 
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
	work_mem                        |52428  |	 
	 
	 
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
	work_mem                        |52428  |	
	```  
# Решение:
* **!!!Исходный запрос подсчитывает неверные кол-ва билетов**
* В моём решении общее кол-во мест на рейс и проданное кол-во билетов, корректно. Если по моему решению еще посчитать Итоговое sum(order_place), то  как раз и получится общее число записей в таблице tickets.
* Моё решение через CTE + доп.индексы.
* Сначала тестировал разные варианты запросов, анализируя план, время, стоимость, порядок обхода, использование индексов. Секционировал методом by_hash таблицу tickets (Время только увеличивалось, что на SSD, что на HDD). Также пробовал на стенде с HDD отсаживать таблицу tickets в отдельное табличное пространство на отдельном диске, время также затрачивалось больше (скорее всего это связано с эмуляцией HDD на YC, т.к. на реальных HDD вероятнее всего, чтение было бы лучше с разных HDD).
* Самым тяжелым местом было соединение ride.fkbus = seat.fkbus, (оно же было не правильным), избавиться от которого для подсчета общего кол-ва посадочных мест решил в первом же запросе. плюс оптимизатор использовал мой первый доп.индекс (см.ниже). В итоге остановился на разбивке запроса на части, продумывая каждый следующий запрос

  --- 

- Сначала (q_one) - выбрал из таблицы ride все поездки с датой, ссылкой на план маршрута и здесь же сразу подсчитал общее кол-во мест в автобусе. ( в помощь первый доп. индекс Index Scan using **seat_fkbus_idx** on seat st)
- Далее (q_two) - выбрал все билеты на каждую поездку. ( в помощь второй доп.индекс **tickets_fkride_idx** Parallel Index Scan using tickets_fkride_idx)
- Далее (q_three) - добавил ко второму запросу ключ маршрута 
- Далее (q_four) - добавил ссылку на автостанцию и подсчитал кол-во проданных билетов (count(ticket.id)), без конкатенации и, тем более, её группировки.
- В последнем результирующем запросе добавил конкатенацию Город ||', '|| Автовокзал и отсортировал по дате выезда.

В процессе тестирования подзапросов пробовал подключать индексы, по оставил только два реально помогающих:
```q
create index CONCURRENTLY seat_fkbus_idx on book.seat (fkbus);
create index CONCURRENTLY tickets_fkride_idx on book.tickets (fkride);
```
Далее тесты:

----- Стенд с SSD -------
-------------------------

**Исходный запрос:**
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
Sort  (cost=79981274.91..80007524.91 rows=10500000 width=56) (actual time=573957.860..574076.002 rows=1500000 loops=1)                                                       |
  Sort Key: r.startdate                                                                                                                                                      |
  Sort Method: external merge  Disk: 81616kB                                                                                                                                 |
  ->  GroupAggregate  (cost=8710415.92..78546563.90 rows=10500000 width=56) (actual time=16363.160..573279.473 rows=1500000 loops=1)                                         |
        Group Key: r.id, ((bs.city || ', '::text) || bs.name)                                                                                                                |
        ->  Merge Join  (cost=8710415.92..56790298.74 rows=2159876516 width=52) (actual time=16362.727..430971.849 rows=2159899000 loops=1)                                  |
              Merge Cond: (r.id = t.fkride)                                                                                                                                  |
              ->  Nested Loop  (cost=81313.24..4756017.46 rows=60000001 width=28) (actual time=696.908..25612.968 rows=60000000 loops=1)                                     |
                    Join Filter: (r.fkbus = st.fkbus)                                                                                                                        |
                    Rows Removed by Join Filter: 240000000                                                                                                                   |
                    ->  Gather Merge  (cost=81313.24..256012.96 rows=1500000 width=28) (actual time=696.863..1710.977 rows=1500000 loops=1)                                  |
                          Workers Planned: 2                                                                                                                                 |
                          Workers Launched: 2                                                                                                                                |
                          ->  Sort  (cost=80313.22..81875.72 rows=625000 width=28) (actual time=643.983..719.735 rows=500000 loops=3)                                        |
                                Sort Key: r.id, (((bs.city || ', '::text) || bs.name))                                                                                       |
                                Sort Method: external merge  Disk: 28184kB                                                                                                   |
                                Worker 0:  Sort Method: external merge  Disk: 28192kB                                                                                        |
                                Worker 1:  Sort Method: external merge  Disk: 27656kB                                                                                        |
                                ->  Hash Join  (cost=49.33..20146.04 rows=625000 width=28) (actual time=240.271..483.170 rows=500000 loops=3)                                |
                                      Hash Cond: (br.fkbusstationfrom = bs.id)                                                                                               |
                                      ->  Hash Join  (cost=48.10..17808.88 rows=625000 width=16) (actual time=0.464..162.858 rows=500000 loops=3)                            |
                                            Hash Cond: (s.fkroute = br.id)                                                                                                   |
                                            ->  Hash Join  (cost=45.75..16050.01 rows=625000 width=16) (actual time=0.413..109.204 rows=500000 loops=3)                      |
                                                  Hash Cond: (r.fkschedule = s.id)                                                                                           |
                                                  ->  Parallel Seq Scan on ride r  (cost=0.00..14359.00 rows=625000 width=16) (actual time=0.039..37.966 rows=500000 loops=3)|
                                                  ->  Hash  (cost=27.00..27.00 rows=1500 width=8) (actual time=0.361..0.363 rows=1500 loops=3)                               |
                                                        Buckets: 2048  Batches: 1  Memory Usage: 75kB                                                                        |
                                                        ->  Seq Scan on schedule s  (cost=0.00..27.00 rows=1500 width=8) (actual time=0.026..0.215 rows=1500 loops=3)        |
                                            ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.033..0.033 rows=60 loops=3)                                           |
                                                  Buckets: 1024  Batches: 1  Memory Usage: 11kB                                                                              |
                                                  ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.021..0.025 rows=60 loops=3)                  |
                                      ->  Hash  (cost=1.10..1.10 rows=10 width=20) (actual time=239.702..239.702 rows=10 loops=3)                                            |
                                            Buckets: 1024  Batches: 1  Memory Usage: 9kB                                                                                     |
                                            ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=20) (actual time=239.681..239.686 rows=10 loops=3)                 |
                    ->  Materialize  (cost=0.00..5.00 rows=200 width=8) (actual time=0.000..0.007 rows=200 loops=1500000)                                                    |
                          ->  Seq Scan on seat st  (cost=0.00..4.00 rows=200 width=8) (actual time=0.017..0.038 rows=200 loops=1)                                            |
              ->  Materialize  (cost=8628992.24..8898976.80 rows=53996912 width=12) (actual time=15665.782..99986.501 rows=2159898961 loops=1)                               |
                    ->  Sort  (cost=8628992.24..8763984.52 rows=53996912 width=12) (actual time=15665.779..24442.321 rows=53997475 loops=1)                                  |
                          Sort Key: t.fkride                                                                                                                                 |
                          Sort Method: external merge  Disk: 1162808kB                                                                                                       |
                          ->  Seq Scan on tickets t  (cost=0.00..1153571.12 rows=53996912 width=12) (actual time=0.027..4816.607 rows=53997475 loops=1)                      |
Planning Time: 0.511 ms                                                                                                                                                      |
JIT:                                                                                                                                                                         |
  Functions: 98                                                                                                                                                              |
  Options: Inlining true, Optimization true, Expressions true, Deforming true                                                                                                |
  Timing: Generation 3.777 ms, Inlining 142.838 ms, Optimization 346.199 ms, Emission 230.029 ms, Total 722.843 ms                                                           |
Execution Time: 574365.324 ms                                                                                                                                                |
```

**Моё решение:**

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
       bs.city || ', ' || bs.name as busstation, 
       f.order_place, f.all_place
  from book.busstation bs
  join q_four f on bs.id = f.fkbusstationfrom  
 order by f.depart_date; 

QUERY PLAN                                                                                                                                                                                                   |
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
Sort  (cost=4826569.76..4826757.26 rows=75000 width=56) (actual time=31353.369..31477.566 rows=1500000 loops=1)                                                                                              |
  Sort Key: r.startdate                                                                                                                                                                                      |
  Sort Method: external merge  Disk: 81616kB                                                                                                                                                                 |
  ->  Hash Join  (cost=4745085.22..4820496.79 rows=75000 width=56) (actual time=29977.202..30789.763 rows=1500000 loops=1)                                                                                   |
        Hash Cond: (br.fkbusstationfrom = bs.id)                                                                                                                                                             |
        ->  Finalize HashAggregate  (cost=4745084.00..4801099.62 rows=1500000 width=28) (actual time=29687.098..30252.573 rows=1500000 loops=1)                                                              |
              Group Key: r.id, r.startdate, br.fkbusstationfrom, (count(st.id))                                                                                                                              |
              Planned Partitions: 4  Batches: 5  Memory Usage: 112689kB  Disk Usage: 48552kB                                                                                                                 |
              ->  Gather  (cost=3973857.98..4602466.81 rows=3000000 width=28) (actual time=28773.563..29008.052 rows=1500000 loops=1)                                                                        |
                    Workers Planned: 2                                                                                                                                                                       |
                    Workers Launched: 2                                                                                                                                                                      |
                    ->  Partial HashAggregate  (cost=3972857.98..4301466.81 rows=1500000 width=28) (actual time=28697.132..28815.986 rows=500000 loops=3)                                                    |
                          Group Key: r.id, r.startdate, br.fkbusstationfrom, (count(st.id))                                                                                                                  |
                          Planned Partitions: 4  Batches: 1  Memory Usage: 81937kB                                                                                                                           |
                          Worker 0:  Batches: 1  Memory Usage: 79897kB                                                                                                                                       |
                          Worker 1:  Batches: 1  Memory Usage: 79897kB                                                                                                                                       |
                          ->  Hash Join  (cost=49.12..2882395.27 rows=22938246 width=28) (actual time=160.088..26038.538 rows=17999158 loops=3)                                                              |
                                Hash Cond: (s.fkroute = br.id)                                                                                                                                               |
                                ->  Hash Join  (cost=46.77..2817926.89 rows=22938246 width=28) (actual time=16.685..23854.384 rows=17999158 loops=3)                                                         |
                                      Hash Cond: (r.fkschedule = s.id)                                                                                                                                       |
                                      ->  Merge Join  (cost=1.02..2757498.12 rows=22938246 width=28) (actual time=15.742..21700.365 rows=17999158 loops=3)                                                   |
                                            Merge Cond: (t.fkride = r.id)                                                                                                                                    |
                                            ->  Parallel Index Scan using tickets_fkride_idx on tickets t  (cost=0.44..1223286.16 rows=22499147 width=12) (actual time=0.061..7870.166 rows=17999158 loops=3)|
                                            ->  Materialize  (cost=0.58..1137640.40 rows=1500000 width=20) (actual time=0.608..10850.978 rows=18999132 loops=3)                                              |
                                                  ->  GroupAggregate  (cost=0.58..1118890.40 rows=1500000 width=20) (actual time=0.606..9879.183 rows=1499986 loops=3)                                       |
                                                        Group Key: r.id                                                                                                                                      |
                                                        ->  Nested Loop  (cost=0.58..803890.41 rows=59999999 width=16) (actual time=0.579..6916.682 rows=59999454 loops=3)                                   |
                                                              ->  Index Scan using ride_pkey on ride r  (cost=0.43..35137.13 rows=1500000 width=16) (actual time=0.282..705.325 rows=1499987 loops=3)        |
                                                              ->  Memoize  (cost=0.15..0.86 rows=40 width=8) (actual time=0.000..0.001 rows=40 loops=4499961)                                                |
                                                                    Cache Key: r.fkbus                                                                                                                       |
                                                                    Cache Mode: logical                                                                                                                      |
                                                                    Hits: 1499997  Misses: 3  Evictions: 0  Overflows: 0  Memory Usage: 5kB                                                                  |
                                                                    Worker 0:  Hits: 1499993  Misses: 3  Evictions: 0  Overflows: 0  Memory Usage: 5kB                                                       |
                                                                    Worker 1:  Hits: 1499962  Misses: 3  Evictions: 0  Overflows: 0  Memory Usage: 5kB                                                       |
                                                                    ->  Index Scan using seat_fkbus_idx on seat st  (cost=0.14..0.85 rows=40 width=8) (actual time=0.075..0.080 rows=40 loops=9)             |
                                                                          Index Cond: (fkbus = r.fkbus)                                                                                                      |
                                      ->  Hash  (cost=27.00..27.00 rows=1500 width=8) (actual time=0.928..0.929 rows=1500 loops=3)                                                                           |
                                            Buckets: 2048  Batches: 1  Memory Usage: 75kB                                                                                                                    |
                                            ->  Seq Scan on schedule s  (cost=0.00..27.00 rows=1500 width=8) (actual time=0.322..0.770 rows=1500 loops=3)                                                    |
                                ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=143.384..143.385 rows=60 loops=3)                                                                                   |
                                      Buckets: 1024  Batches: 1  Memory Usage: 11kB                                                                                                                          |
                                      ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=143.362..143.367 rows=60 loops=3)                                                          |
        ->  Hash  (cost=1.10..1.10 rows=10 width=20) (actual time=290.085..290.086 rows=10 loops=1)                                                                                                          |
              Buckets: 1024  Batches: 1  Memory Usage: 9kB                                                                                                                                                   |
              ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=20) (actual time=290.067..290.071 rows=10 loops=1)                                                                               |
Planning Time: 1.522 ms                                                                                                                                                                                      |
JIT:                                                                                                                                                                                                         |
  Functions: 142                                                                                                                                                                                             |
  Options: Inlining true, Optimization true, Expressions true, Deforming true                                                                                                                                |
  Timing: Generation 19.229 ms, Inlining 103.740 ms, Optimization 381.443 ms, Emission 270.221 ms, Total 774.632 ms                                                                                          |
Execution Time: 31599.153 ms                                                                                                                                                                                 |

```

**Вывод по SSD:**

Исходый запрос:

(cost=79 981 274.91..80 007 524.91 actual time=573 957.860..574 076.002 rows=1 500 000)   

Оптимизированный запрос

(cost= 4 826 569.76..4 826 757.26 actual time=31 353.369.. 31 477.566 rows=1 500 000)

Прирост по времени 18 - кратный, по стоимости 16 кратное снижение стоимости.



----- Стенд с HDD -------
-------------------------

**Исходный запрос:**
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
 Sort  (cost=80590536.40..80616786.40 rows=10500000 width=56) (actual time=637070.144..637197.838 rows=1500000 loops=1)
   Sort Key: r.startdate
   Sort Method: external merge  Disk: 81632kB
   ->  GroupAggregate  (cost=9092845.69..79007142.38 rows=10500000 width=56) (actual time=19526.879..636634.076 rows=1500000 loops=1)
         Group Key: r.id, ((bs.city || ', '::text) || bs.name)
         ->  Merge Join  (cost=9092845.69..57250570.02 rows=2159907236 width=52) (actual time=19526.370..492884.987 rows=2159899000 loops=1)
               Merge Cond: (r.id = t.fkride)
               ->  Nested Loop  (cost=81313.24..4756017.46 rows=60000001 width=28) (actual time=1049.345..28618.857 rows=60000000 loops=1)
                     Join Filter: (r.fkbus = st.fkbus)
                     Rows Removed by Join Filter: 240000000
                     ->  Gather Merge  (cost=81313.24..256012.96 rows=1500000 width=28) (actual time=1049.284..1447.928 rows=1500000 loops=1)
                           Workers Planned: 2
                           Workers Launched: 2
                           ->  Sort  (cost=80313.22..81875.72 rows=625000 width=28) (actual time=996.018..1059.907 rows=500000 loops=3)
                                 Sort Key: r.id, (((bs.city || ', '::text) || bs.name))
                                 Sort Method: quicksort  Memory: 39084kB
                                 Worker 0:  Sort Method: quicksort  Memory: 41751kB
                                 Worker 1:  Sort Method: external merge  Disk: 47032kB
                                 ->  Hash Join  (cost=49.33..20146.04 rows=625000 width=28) (actual time=370.718..784.449 rows=500000 loops=3)
                                       Hash Cond: (br.fkbusstationfrom = bs.id)
                                       ->  Hash Join  (cost=48.10..17808.88 rows=625000 width=16) (actual time=0.673..261.461 rows=500000 loops=3)
                                             Hash Cond: (s.fkroute = br.id)
                                             ->  Hash Join  (cost=45.75..16050.01 rows=625000 width=16) (actual time=0.590..168.724 rows=500000 loops=3)
                                                   Hash Cond: (r.fkschedule = s.id)
                                                   ->  Parallel Seq Scan on ride r  (cost=0.00..14359.00 rows=625000 width=16) (actual time=0.019..57.248 rows=500000 loops=3)
                                                   ->  Hash  (cost=27.00..27.00 rows=1500 width=8) (actual time=0.554..0.556 rows=1500 loops=3)
                                                         Buckets: 2048  Batches: 1  Memory Usage: 75kB
                                                         ->  Seq Scan on schedule s  (cost=0.00..27.00 rows=1500 width=8) (actual time=0.017..0.260 rows=1500 loops=3)
                                             ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.062..0.063 rows=60 loops=3)
                                                   Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                                   ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.040..0.047 rows=60 loops=3)
                                       ->  Hash  (cost=1.10..1.10 rows=10 width=20) (actual time=369.982..369.983 rows=10 loops=3)
                                             Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                             ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=20) (actual time=369.948..369.955 rows=10 loops=3)
                     ->  Materialize  (cost=0.00..5.00 rows=200 width=8) (actual time=0.000..0.008 rows=200 loops=1500000)
                           ->  Seq Scan on seat st  (cost=0.00..4.00 rows=200 width=8) (actual time=0.024..0.052 rows=200 loops=1)
               ->  Materialize  (cost=9011421.74..9281410.14 rows=53997680 width=12) (actual time=18476.979..118403.429 rows=2159898961 loops=1)
                     ->  Sort  (cost=9011421.74..9146415.94 rows=53997680 width=12) (actual time=18476.976..28308.434 rows=53997475 loops=1)
                           Sort Key: t.fkride
                           Sort Method: external merge  Disk: 1162928kB
                           ->  Seq Scan on tickets t  (cost=0.00..1153579.80 rows=53997680 width=12) (actual time=0.037..5533.997 rows=53997475 loops=1)
 Planning Time: 0.720 ms
 JIT:
   Functions: 98
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 4.910 ms, Inlining 171.195 ms, Optimization 550.343 ms, Emission 388.391 ms, Total 1114.839 ms
 Execution Time: 637392.105 ms
(47 rows)(END)
```



**Мой вариант без доп.индексов:**
```sql
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
       bs.city || ', ' || bs.name as busstation, 
       f.order_place, f.all_place
  from book.busstation bs
  join q_four f on bs.id = f.fkbusstationfrom  
 order by f.depart_date; 
                                                                                        QUERY PLAN                                                             
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=11340550.34..11340737.84 rows=75000 width=56) (actual time=53442.796..53581.585 rows=1500000 loops=1)
   Sort Key: r.startdate
   Sort Method: external merge  Disk: 81632kB
   ->  Hash Join  (cost=10550890.21..11334477.36 rows=75000 width=56) (actual time=48236.309..52840.471 rows=1500000 loops=1)
         Hash Cond: (br.fkbusstationfrom = bs.id)
         ->  HashAggregate  (cost=10550888.98..11315080.20 rows=1500000 width=28) (actual time=47883.088..52173.388 rows=1500000 loops=1)
               Group Key: r.id, r.startdate, br.fkbusstationfrom, (count(st.id))
               Planned Partitions: 4  Batches: 5  Memory Usage: 106545kB  Disk Usage: 1411040kB
               ->  Hash Join  (cost=4435780.03..5773189.58 rows=54797986 width=28) (actual time=21147.919..36171.748 rows=53997475 loops=1)
                     Hash Cond: (r.id = t.fkride)
                     ->  Hash Join  (cost=2343568.23..2746756.92 rows=1500000 width=20) (actual time=6317.208..7380.324 rows=1500000 loops=1)
                           Hash Cond: (s.fkroute = br.id)
                           ->  Hash Join  (cost=2343565.88..2742538.94 rows=1500000 width=20) (actual time=6317.126..7158.398 rows=1500000 loops=1)
                                 Hash Cond: (r.fkschedule = s.id)
                                 ->  Finalize GroupAggregate  (cost=2343520.13..2723544.57 rows=1500000 width=20) (actual time=6316.808..6918.229 rows=1500000 loops=1)
                                       Group Key: r.id
                                       ->  Gather Merge  (cost=2343520.13..2693544.57 rows=3000000 width=20) (actual time=6316.759..6586.020 rows=1500000 loops=1)
                                             Workers Planned: 2
                                             Workers Launched: 2
                                             ->  Sort  (cost=2342520.11..2346270.11 rows=1500000 width=20) (actual time=6095.862..6169.193 rows=500000 loops=3)
                                                   Sort Key: r.id
                                                   Sort Method: external merge  Disk: 24848kB
                                                   Worker 0:  Sort Method: quicksort  Memory: 35934kB
                                                   Worker 1:  Sort Method: quicksort  Memory: 35726kB
                                                   ->  Partial HashAggregate  (cost=1898740.50..2157881.12 rows=1500000 width=20) (actual time=5696.087..5836.089 rows=500000 loops=3)
                                                         Group Key: r.id
                                                         Planned Partitions: 4  Batches: 1  Memory Usage: 83985kB
                                                         Worker 0:  Batches: 1  Memory Usage: 55313kB
                                                         Worker 1:  Batches: 1  Memory Usage: 55313kB
                                                         ->  Hash Join  (cost=6.50..297178.00 rows=25000000 width=16) (actual time=161.468..2376.690 rows=20000000 loops=3)
                                                               Hash Cond: (r.fkbus = st.fkbus)
                                                               ->  Parallel Seq Scan on ride r  (cost=0.00..14359.00 rows=625000 width=16) (actual time=0.014..64.853 rows=500000 loops=3)
                                                               ->  Hash  (cost=4.00..4.00 rows=200 width=8) (actual time=161.425..161.427 rows=200 loops=3)
                                                                     Buckets: 1024  Batches: 1  Memory Usage: 16kB
                                                                     ->  Seq Scan on seat st  (cost=0.00..4.00 rows=200 width=8) (actual time=161.318..161.364 rows=200 loops=3)
                                 ->  Hash  (cost=27.00..27.00 rows=1500 width=8) (actual time=0.304..0.305 rows=1500 loops=1)
                                       Buckets: 2048  Batches: 1  Memory Usage: 75kB
                                       ->  Seq Scan on schedule s  (cost=0.00..27.00 rows=1500 width=8) (actual time=0.012..0.143 rows=1500 loops=1)
                           ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.060..0.061 rows=60 loops=1)
                                 Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                 ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.042..0.048 rows=60 loops=1)
                     ->  Hash  (cost=1153579.80..1153579.80 rows=53997680 width=12) (actual time=14807.698..14807.699 rows=53997475 loops=1)
                           Buckets: 2097152  Batches: 32  Memory Usage: 89116kB
                           ->  Seq Scan on tickets t  (cost=0.00..1153579.80 rows=53997680 width=12) (actual time=0.083..5619.586 rows=53997475 loops=1)
         ->  Hash  (cost=1.10..1.10 rows=10 width=20) (actual time=353.201..353.201 rows=10 loops=1)
               Buckets: 1024  Batches: 1  Memory Usage: 9kB
               ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=20) (actual time=353.170..353.177 rows=10 loops=1)
 Planning Time: 0.529 ms
 JIT:
   Functions: 91
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 5.600 ms, Inlining 194.994 ms, Optimization 397.213 ms, Emission 284.331 ms, Total 882.137 ms
 Execution Time: 54685.616 ms
(53 rows)
```

**Мой вариант с доп.индексами:**
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
       bs.city || ', ' || bs.name as busstation, 
       f.order_place, f.all_place
  from book.busstation bs
  join q_four f on bs.id = f.fkbusstationfrom  
 order by f.depart_date;
                                                                                                  QUERY PLAN                                                   
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=9397099.93..9397287.43 rows=75000 width=56) (actual time=54682.931..54808.876 rows=1500000 loops=1)
   Sort Key: r.startdate
   Sort Method: external merge  Disk: 81632kB
   ->  Hash Join  (cost=1008.00..9391026.96 rows=75000 width=56) (actual time=500.329..54155.695 rows=1500000 loops=1)
         Hash Cond: (br.fkbusstationfrom = bs.id)
         ->  GroupAggregate  (cost=1006.78..9371629.79 rows=1500000 width=28) (actual time=170.899..53300.085 rows=1500000 loops=1)
               Group Key: r.id, r.startdate, br.fkbusstationfrom, (count(st.id))
               ->  Incremental Sort  (cost=1006.78..8676356.08 rows=54421897 width=28) (actual time=170.581..48301.714 rows=53997475 loops=1)
                     Sort Key: r.id, r.startdate, br.fkbusstationfrom, (count(st.id))
                     Presorted Key: r.id
                     Full-sort Groups: 1476900  Sort Method: quicksort  Average Memory: 27kB  Peak Memory: 27kB
                     Pre-sorted Groups: 38766  Sort Method: quicksort  Average Memory: 27kB  Peak Memory: 27kB
                     ->  Merge Join  (cost=1001.47..5540158.90 rows=54421897 width=28) (actual time=170.303..35870.564 rows=53997475 loops=1)
                           Merge Cond: (r.id = t.fkride)
                           ->  Nested Loop  (cost=1001.03..1370622.51 rows=1500000 width=20) (actual time=170.249..5089.011 rows=1500000 loops=1)
                                 ->  Nested Loop  (cost=1000.88..1333475.15 rows=1500000 width=20) (actual time=170.221..4425.222 rows=1500000 loops=1)
                                       ->  Finalize GroupAggregate  (cost=1000.61..875916.67 rows=1500000 width=20) (actual time=170.194..2149.148 rows=1500000 loops=1)
                                             Group Key: r.id
                                             ->  Gather Merge  (cost=1000.61..845916.67 rows=3000000 width=20) (actual time=170.092..1705.086 rows=1500000 loops=1)
                                                   Workers Planned: 2
                                                   Workers Launched: 2
                                                   ->  Partial GroupAggregate  (cost=0.58..498642.21 rows=1500000 width=20) (actual time=95.308..3892.636 rows=500000 loops=3)
                                                         Group Key: r.id
                                                         ->  Nested Loop  (cost=0.58..358642.21 rows=25000000 width=16) (actual time=95.268..2708.426 rows=20000000 loops=3)
                                                               ->  Parallel Index Scan using ride_pkey on ride r  (cost=0.43..38326.43 rows=625000 width=16) (actual time=0.046..162.539 rows=500000 loops=3)
                                                               ->  Memoize  (cost=0.15..0.86 rows=40 width=8) (actual time=0.000..0.002 rows=40 loops=1500000)
                                                                     Cache Key: r.fkbus
                                                                     Cache Mode: logical
                                                                     Hits: 174579  Misses: 3  Evictions: 0  Overflows: 0  Memory Usage: 5kB
                                                                     Worker 0:  Hits: 662823  Misses: 3  Evictions: 0  Overflows: 0  Memory Usage: 5kB
                                                                     Worker 1:  Hits: 662589  Misses: 3  Evictions: 0  Overflows: 0  Memory Usage: 5kB
                                                                     ->  Index Scan using seat_fkbus_idx on seat st  (cost=0.14..0.85 rows=40 width=8) (actual time=0.027..0.036 rows=40 loops=9)
                                                                           Index Cond: (fkbus = r.fkbus)
                                       ->  Index Scan using schedule_pkey on schedule s  (cost=0.28..0.30 rows=1 width=8) (actual time=0.001..0.001 rows=1 loops=1500000)
                                             Index Cond: (id = r.fkschedule)
                                 ->  Memoize  (cost=0.15..0.18 rows=1 width=8) (actual time=0.000..0.000 rows=1 loops=1500000)
                                       Cache Key: s.fkroute
                                       Cache Mode: logical
                                       Hits: 1499940  Misses: 60  Evictions: 0  Overflows: 0  Memory Usage: 7kB
                                       ->  Index Scan using busroute_pkey on busroute br  (cost=0.14..0.17 rows=1 width=8) (actual time=0.002..0.002 rows=1 loops=60)
                                             Index Cond: (id = s.fkroute)
                           ->  Index Scan using tickets_fkride_idx on tickets t  (cost=0.44..3458344.24 rows=53997696 width=12) (actual time=0.039..24289.439 rows=53997475 loops=1)
         ->  Hash  (cost=1.10..1.10 rows=10 width=20) (actual time=329.409..329.410 rows=10 loops=1)
               Buckets: 1024  Batches: 1  Memory Usage: 9kB
               ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=20) (actual time=329.377..329.384 rows=10 loops=1)
 Planning Time: 0.606 ms
 JIT:
   Functions: 78
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 3.944 ms, Inlining 122.822 ms, Optimization 275.834 ms, Emission 216.207 ms, Total 618.807 ms
 Execution Time: 54869.482 ms
(51 rows)
```
**Вывод по HDD:**

Исходный запрос:

(cost=80 590 536.40..80 616 786.40 actual time=637 070.144..637 197.838 rows=1 500 000)

Оптимизированный запрос:

(cost=9 397 099.93..9 397 287.43 actual time=54 682.931..54 808.876 rows=1 500 000

Прирост по времени 11.5 - кратный, по стоимости 9.5 - ти кратное снижение стоимости.

**Доп.информация по секционированию в процессе тестирования**
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