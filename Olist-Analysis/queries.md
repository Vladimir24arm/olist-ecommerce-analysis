1)Топ 10 категорий по выручке.

Бизнес-вопрос: Какие категории товаров приносят больше всего денег?
Что делает: Соединяет 4 таблицы, фильтрует доставленные заказы, считает сумму выручки и количество позиций по каждой категории, выводит топ-10.
Использовал функции WITH, JOIN (4 таблицы), WHERE, SUM, COUNT, GROUP BY, ORDER BY, LIMIT:
```sql
with a as(select o.product_id, pt.product_category_name_english, orr.order_status, oi.price, oi.freight_value,
o.product_name_lenght, o.product_description_lenght, o.product_photos_qty, o.product_weight_g, o.product_length_cm,
o.product_height_cm, o.product_width_cm 
from olist_products o
join olist_order_items oi on oi.product_id = o.product_id
join olist_orders orr on orr.order_id = oi.order_id
join product_category_translation pt on pt.product_category_name = o.product_category_name)
select product_category_name_english, count(*) as cnt, sum(price)
from a
where  order_status = 'delivered'
group by product_category_name_english
order by sum desc
limit 10;

Так же написал более читаемый вариант запроса:

 select pct.product_category_name_english, sum(oi.price) as total, oo.order_status
 from olist_order_items oi
 join olist_orders oo on oo.order_id = oi.order_id
 join olist_products op on oi.product_id = op.product_id
 join product_category_translation pct on pct.product_category_name = op.product_category_name
 where oo.order_status = 'delivered'
 group by pct.product_category_name_english, oo.order_status
 order by total desc;

2) Средняя цена позиции по каждому заказу

Бизнес-вопрос: Сколько в среднем стоит одна позиция в каждом заказе?
Что делает: Соединяет товары и заказы, фильтрует доставленные, группирует по номеру заказа, считает количество позиций и среднюю цену, сортирует по убыванию.
Использовал функции JOIN (2 таблицы), COUNT, AVG, ROUND, GROUP BY, ORDER BY:

select count(*) as cnt,oi.order_id, round(avg(oi.price),2) as summ    
 from olist_order_items oi
 join olist_orders oo on oi.order_id = oo.order_id
 where oo.order_status = 'delivered'
 group by oi.order_id
 order by summ desc;


3) Топ-10 штатов по выручке

Бизнес-вопрос: Из какого региона приходит больше всего денег?
Что делает: Соединяет клиентов, заказы и товары, фильтрует доставленные, группирует по штату, считает сумму выручки, сортирует по убыванию и выводит топ-10.
Использовал функции JOIN (3 таблицы), SUM, GROUP BY, ORDER BY, LIMIT:

 select oc.customer_state, sum(oi.price) as total 
 from olist_customers oc
 join olist_orders oo on oo.customer_id = oc.customer_id
 join olist_order_items oi on oi.order_id = oo.order_id
 group by oc.customer_state 
 order by total desc
 limit 10;

4) Динамика выручки по месяцам (базовая)

Бизнес-вопрос: Как менялась выручка по месяцам? В какие месяцы были пики продаж?
Что делает: Соединяет товары и заказы, фильтрует доставленные, вытаскивает месяц и год из даты, группирует по ним и считает сумму выручки. Сортирует по убыванию.
Использовал функции JOIN (2 таблицы), EXTRACT, SUM, GROUP BY (2 колонки), ORDER BY:

 select extract (month from oo.order_purchase_timestamp) as month, extract (year from oo.order_purchase_timestamp)as year,
 sum(oi.price) as total
 from olist_order_items oi 
 join olist_orders oo on oo.order_id = oi.order_id
 where oo.order_status = 'delivered'
 group by month,year
 order by total desc;


5) Динамика выручки с расчетом разницы (LAG)

Бизнес-вопрос: На сколько выросла или упала выручка по сравнению с предыдущим месяцем?
Что делает: Внутри CTE считает выручку по месяцам, а снаружи использует LAG, чтобы получить значение прошлого месяца и вычислить разницу (diff). Отфильтровывает 2016 год.
Использовал функции WITH (CTE), LAG (оконная функция), OVER, ORDER BY:

 with a as(select extract (month from oo.order_purchase_timestamp) as month, extract (year from oo.order_purchase_timestamp)as year,
 sum(oi.price) as total
 from olist_order_items oi 
 join olist_orders oo on oo.order_id = oi.order_id
 where oo.order_status = 'delivered'
 group by month,year)
 select year, month, total, lag(total) over(order by year, month) as prev_month_total, total -  lag(total) over(order by year, month) as diff
 from a
 where year > 2016
 order by year, month;

6) Влияние задержки доставки на оценку

Бизнес-вопрос: Правда ли, что если заказ опаздывает, клиенты ставят более низкие оценки?
Что делает: Сравнивает фактическую и плановую даты доставки через CASE WHEN, делит заказы на «Опоздал» и «Вовремя», считает среднюю оценку для каждой группы.
Использовал функции CASE WHEN, JOIN, AVG, ROUND, GROUP BY, IS NOT NULL:

select case when oo.order_delivered_customer_date>oo.order_estimated_delivery_date then 'Опоздал'
 else 'Вовремя'end as delivery_status, round(avg(orr.review_score),2) as avg_score
 from olist_orders oo
join olist_order_reviews orr on oo.order_id = orr.order_id
where oo.order_status = 'delivered'
and oo.order_delivered_customer_date is not null
and oo.order_estimated_delivery_date is not null
group by delivery_status;

7) Процент отмененных заказов по месяцам

Бизнес-вопрос: В каком месяце был самый высокий процент отмен?
Что делает: Вытаскивает год и месяц, считает общее количество заказов и отмен, вычисляет процент отмен с округлением. Фильтрует 2016 год.
Использовал функции EXTRACT, COUNT, SUM(CASE WHEN...), ROUND, GROUP BY (2 колонки), ORDER BY:

select extract(year from order_purchase_timestamp)as year, extract(month from order_purchase_timestamp) as month,count(*) as cnt,
sum(case when order_status = 'canceled' then 1 else 0 end) as canceled_orders,
round((sum(case when order_status = 'canceled' then 1 else 0 end)*100.0)/ count(*),2) as cancel_percentage
from olist_orders
where extract(year from order_purchase_timestamp) > 2016
group by year, month
order by year, month asc;

8) Распределение выручки по способам оплаты

Бизнес-вопрос: Каким способом клиенты платят чаще всего и какой способ приносит больше всего денег?
Что делает: Соединяет платежи и заказы, фильтрует доставленные, группирует по типу оплаты, считает сумму и количество заказов, сортирует по убыванию.
Использовал функции JOIN (2 таблицы), SUM, COUNT, GROUP BY, ORDER BY:

select p.payment_type, sum(p.payment_value) as total_payments, count(*) as total_orders
from olist_order_payments p
join olist_orders o on p.order_id = o.order_id
where o.order_status = 'delivered'
group by p.payment_type
order by total_payments desc;












