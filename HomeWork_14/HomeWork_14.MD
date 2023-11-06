# Домашнее задание №14
## Триггеры, поддержка заполнения витрин

**Цель:**
Создать триггер для поддержки витрины в актуальном состоянии.


Описание/Пошаговая инструкция выполнения домашнего задания:
* Скрипт и развернутое описание задачи – в ЛК (файл hw_triggers.sql) или по ссылке: https://disk.yandex.ru/d/l70AvknAepIJXQ
В БД создана структура, описывающая товары (таблица goods) и продажи (таблица sales).
Есть запрос для генерации отчета – сумма продаж по каждому товару.
БД была денормализована, создана таблица (витрина), структура которой повторяет структуру отчета.
Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)
Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE

```sql
-- Создание триггера и функции 
create trigger t_sales_binsert before insert on sales 
for each row 
execute procedure t_sales_binsert();

create or replace function t_sales_binsert()
returns trigger
language plpgsql
security definer
set search_path = pract_functions
as $function$

declare
  sGood_name  goods.good_name%type;   -- Наименование товара
  nGood_price goods.good_price%type;  -- Цена товара
begin
  /* Выборка наименования и цены товара*/	
  select g.good_name, 
         g.good_price
    into strict 
    	 sGood_name,
    	 nGood_price
    from goods g 
   where g.goods_id = new.good_id;
  
  /* Попытка сразу обновить данные, если нужные данные присутствуют */
  update good_sum_mart m 
     set sum_sale = sum_sale + (new.sales_qty * nGood_price)
   where m.good_name = sGood_name; 								 
  /* Если запись с указанным именем товара не найдена, тогда
     произведем вставку новой записи и суммы продажи */
  if not found then
    insert into good_sum_mart (good_name, sum_sale)
    values (sGood_name, new.sales_qty * nGood_price);
  end if;
return new; 
end;
$function$;

-- Проверка работоспособности
INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

select * from good_sum_mart;

good_name               |sum_sale    |
------------------------+------------+
Спички хозайственные    |       65.50|
Автомобиль Ferrari FXX K|185000000.01|

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

select * from good_sum_mart;

good_name               |sum_sale    |
------------------------+------------+
Спички хозайственные    |      131.00|
Автомобиль Ferrari FXX K|370000000.02|
```
__Задание со звездочкой*__

* Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
Подсказка: В реальной жизни возможны изменения цен.

**Ответ:** С использованием схемы (витрина+триггер) можно не бояться изменения цен, 
сумма продажи рассчитывается в момент добавления продажи по актуальной цене, 
но это если говорить только про INSERT в таблицу Sales, а если добавлять функционал 
отмены продажи (delete from Sales), или update Sales, то тут уже нужно будет реализовывать хронику изменения цен товаров.

А вот отчет будет неправильным (сумма исказится) при первом же изменении цены товара в таблице Goods в текущей реализации БД (тут как минимум нужна реализация хроники изменения цен и в отчете отталкиваться при подсчете суммы на цену на дату продажи).
