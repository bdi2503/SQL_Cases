### Описание:
На дашборде отражен анализ экономической эфективности продукта. 

Посчитыны: Revenue, ARPU, ARPPU, AOV, TAX, gross profit и др. Со всеми посчитанными метриками можно ознакомиться по ссылке ниже.

**Ссылка на дашборд**: [Анализ экономики продукта](https://redash.public.karpov.courses/dashboards/2972--/ "Ссылка для просмотра Дашборда")

![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/12f4e874-5a5e-402b-81d6-9640a398f048)
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/f880a10f-a1b3-4d74-8a06-c4f4fc57e914)
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/d5895cde-f45b-4f3c-a74e-9898a777cb50)

**prod_2.1 - Динамика Ежедневной / Общей выручки.**
```
with rev as (
    SELECT 
        date,
        sum(price) over (partition by date order by date) as revenue,
        row_number() over(partition by date)
    FROM (
        SELECT 
            order_id,
            creation_time::date as date,
            product_ids,
            unnest(product_ids) as product_id
        FROM orders
        where order_id not in ( SELECT order_id
                                    FROM user_actions
                                    where action = 'cancel_order')) as a
    join products using(product_id)
    order by date)

SELECT 
    date,
    revenue,
    sum(revenue) over (order by date) as total_revenue,
    round((revenue - lag(revenue, 1) over ()) / lag(revenue, 1) over () * 100, 2) as revenue_change
FROM rev 
where row_number = 1
```

**prod_2.7 - Динамика валовой прибыль и доля валовой прибыли в этот день (выручка за вычетом затрат и НДС) /
Динамика суммарной валовой прибыли и доля суммарной валовой прибыли в суммарной выручке на текущий день.**
```
with sborka_arenda as (
    SELECT
        date,
        sum(sborka) as sborka,
        CASE 
            WHEN date BETWEEN '2022-08-01' and '2022-08-31' THEN 120000
            WHEN date BETWEEN '2022-09-01' and '2022-09-30' THEN 150000
            ELSE 0
        END AS arenda
    FROM (
        SELECT 
            order_id,
            creation_time::date as date,
            CASE 
                WHEN creation_time::date BETWEEN '2022-08-01' and '2022-08-31'  THEN 140
                WHEN creation_time::date BETWEEN '2022-09-01' and '2022-09-30'  THEN 115
                ELSE 0
                END AS sborka
        FROM orders
        WHERE order_id not in ( SELECT order_id
                                FROM user_actions
                                WHERE action = 'cancel_order')) AS t1
    GROUP BY date
    ORDER BY date),

salary_cour as (
    SELECT
        date,
        sum(salary_courier) as salary_couriers
    FROM (
        SELECT
            courier_id,
            time::date as date,
            count(order_id) as count_orders,
            CASE 
                WHEN count(order_id) >= 5 and time::date BETWEEN '2022-08-01' and '2022-08-31'  THEN count(order_id) * 150 + 400
                WHEN count(order_id) >= 5 and time::date BETWEEN '2022-09-01' and '2022-09-30'  THEN count(order_id) * 150 + 500
                ELSE count(order_id) * 150
            END AS salary_courier
        FROM courier_actions
        WHERE order_id not in ( SELECT order_id
                                FROM user_actions
                                WHERE action = 'cancel_order')
            and action = 'deliver_order'
        GROUP BY courier_id, time::date
        ORDER BY courier_id, date) as t1
    GROUP BY date
    ORDER BY date),

costs_total_costs as ( 
    SELECT
        date,
        sborka + arenda + salary_couriers as costs,
        sum(sborka + arenda + salary_couriers) OVER(ORDER BY date) as total_costs
    FROM sborka_arenda JOIN salary_cour USING(date)), 

prod_price as (
    SELECT 
        product_id,
        name,
        price,
        CASE 
            WHEN name in (  'сахар', 'сухарики', 'сушки', 'семечки', 
                            'масло льняное', 'виноград', 'масло оливковое', 
                            'арбуз', 'батон', 'йогурт', 'сливки', 'гречка', 
                            'овсянка', 'макароны', 'баранина', 'апельсины', 
                            'бублики', 'хлеб', 'горох', 'сметана', 'рыба копченая', 
                            'мука', 'шпроты', 'сосиски', 'свинина', 'рис', 
                            'масло кунжутное', 'сгущенка', 'ананас', 'говядина', 
                            'соль', 'рыба вяленая', 'масло подсолнечное', 'яблоки', 
                            'груши', 'лепешка', 'молоко', 'курица', 'лаваш', 'вафли', 'мандарины') 
            THEN round(price * 10 / 110, 2)
            ELSE round(price * 20 / 120, 2)
        END AS nds
    FROM products),

revenue_tax as (
    SELECT
        date, revenue, tax,
        sum(revenue) OVER(ORDER BY date) as total_revenue,
        sum(tax) OVER(ORDER BY date) as total_tax	
    FROM (
        SELECT 
            date,
            sum(price) as revenue,
            sum(nds) as tax
        FROM (
            SELECT 
                order_id,
                creation_time::date as date,
                product_ids,
                unnest(product_ids) as product_id
            FROM orders
            WHERE order_id not in ( SELECT order_id
                                    FROM user_actions
                                    WHERE action = 'cancel_order')) as a
        JOIN prod_price USING(product_id)
        GROUP BY date
        ORDER by date) as t1)

SELECT
    date,
    revenue,
    costs,
    tax,
    revenue - costs - tax as gross_profit,
    total_revenue,
    total_costs,
    total_tax,
    sum(revenue - costs - tax) OVER(ORDER BY date) as total_gross_profit,
    round((revenue - costs - tax) / revenue * 100, 2) as gross_profit_ratio,
    round(sum(revenue - costs - tax) OVER(ORDER BY date) / total_revenue * 100, 2) as total_gross_profit_ratio
FROM costs_total_costs JOIN revenue_tax USING(date)
```

**prod_2.2 - Динамика arpu, arppu, aov по дням.**
```
with rev as (
    SELECT 
        date,
        sum(price) as revenue
    FROM (
        SELECT 
            order_id,
            creation_time::date as date,
            product_ids,
            unnest(product_ids) as product_id
        FROM orders
        where order_id not in ( SELECT order_id
                                    FROM user_actions
                                    where action = 'cancel_order')) as a
    join products using(product_id)
    group by date
    order by date),

count_users_orders as (
    SELECT
        *
    FROM (
        select
            time::date as date,
            count(DISTINCT user_id) as uniq_users
        from user_actions
        group by time::date) as un_us
    join 
        (select 
            time::date as date,
            count(DISTINCT user_id) as activ_users,
            count(order_id) as orders
        from user_actions
        where order_id not in ( SELECT order_id
                                FROM user_actions
                                where action = 'cancel_order')
        group by time::date) as or_us
    using(date))
    
    
SELECT
    date,
    round(revenue / uniq_users, 2) as arpu,
    round(revenue / activ_users, 2) as arppu,
    round(revenue / orders, 2) as aov
FROM rev join count_users_orders using(date)
```

**prod_2.4 - Динамика arpu, arppu, aov по дням недели.**
```
with rev as (
    SELECT 
        weekday_number,
        weekday,
        sum(price) as revenue
    FROM (
        SELECT 
            order_id,
            creation_time::date as date,
            to_char(creation_time, 'Day') as weekday,
            DATE_PART('isodow', creation_time) as weekday_number,
            product_ids,
            unnest(product_ids) as product_id
        FROM orders
        where order_id not in ( SELECT order_id
                                    FROM user_actions
                                    where action = 'cancel_order')
              and creation_time::date BETWEEN '2022-08-26' and '2022-09-08') as a
    join products using(product_id)
    group by weekday_number, weekday
    order by weekday_number),

count_users_orders as (
    SELECT
        un_us.weekday as weekday, weekday_number, uniq_users, activ_users, orders
    FROM (
        select
            to_char(time, 'Day') as weekday,
            DATE_PART('isodow', time) as weekday_number,
            count(DISTINCT user_id) as uniq_users
        from user_actions
        WHERE time::date BETWEEN '2022-08-26' and '2022-09-08'
        group by DATE_PART('isodow', time), to_char(time, 'Day')) as un_us
    join 
        (select 
            to_char(time, 'Day') as weekday,
            DATE_PART('isodow', time) as weekday_number,
            count(DISTINCT user_id) as activ_users,
            count(order_id) as orders
        from user_actions
        where order_id not in ( SELECT order_id
                                FROM user_actions
                                where action = 'cancel_order')
              and time::date BETWEEN '2022-08-26' and '2022-09-08'
        group by DATE_PART('isodow', time), to_char(time, 'Day')) as or_us
    using(weekday_number))
    
  
SELECT
    rev.weekday as weekday,
    weekday_number,
    round(revenue / uniq_users, 2) as arpu,
    round(revenue / activ_users, 2) as arppu,
    round(revenue / orders, 2) as aov
FROM rev join count_users_orders using(weekday_number)
```

**prod_2.3 - Динамика накопительной выручки на пользователя / платящего пользователя и с заказа.**
```
with sub1 as(
    SELECT DISTINCT date,
           sum(order_price) OVER(PARTITION BY date) as revenue,
           count(order_id) OVER(PARTITION BY date) as num_orders
    FROM (
        SELECT order_id, date, sum(price) as order_price
        FROM ( 
            SELECT 
                order_id, 
                product_id, 
                price, 
                date
            FROM products
            JOIN (SELECT order_id,
                         unnest(product_ids) as product_id,
                         creation_time::date as date
                  FROM orders) as t1 
            using(product_id)
            WHERE order_id not in (SELECT order_id FROM user_actions WHERE action = 'cancel_order')) as aaaaaaa
        GROUP BY order_id, date) as b
    order by date
),

sub2 AS(
    SELECT time::date as date,
           count(distinct user_id) as num_users
    FROM user_actions 
    JOIN (
        SELECT DISTINCT user_id,
               min(time::date) OVER(PARTITION BY user_id) as user_first_date
        FROM user_actions) as t1 
    USING(user_id)
    WHERE user_first_date = time::date
    GROUP BY time::date
),

sub3 as (
    SELECT date,
           round(sum(revenue) OVER(ORDER BY date)::decimal / sum(num_users) OVER(ORDER BY date), 2) as running_arpu,
           round(sum(revenue) OVER(ORDER BY date)::decimal / sum(num_orders) OVER(ORDER BY date), 2) as running_aov
    FROM sub1 INNER JOIN sub2 using(date) 
    ORDER BY date),
    
sub4 as (
    SELECT 
        time::date as date,
        count(distinct user_id) as num_paid_users
    FROM user_actions 
    JOIN (
        SELECT 
            DISTINCT user_id,
            min(time::date) OVER(PARTITION BY user_id) as user_first_date
        FROM user_actions
        where order_id not in ( SELECT order_id
                                FROM user_actions
                                where action = 'cancel_order')) as t2 
    USING(user_id)
    WHERE user_first_date = time::date
    GROUP BY time::date
),

sub5 as (
    SELECT date,
           round(sum(revenue) OVER(ORDER BY date)::decimal / sum(num_paid_users) OVER(ORDER BY date), 2) as running_arppu
    FROM sub1 INNER JOIN sub4 using(date) 
    ORDER BY date
)

SELECT
    date,
    running_arpu,
    running_arppu,
    running_aov
FROM sub3 JOIN sub5 using(date)
```

**prod_2.5 - Динамика долей выручки с заказов новых пользователей в общей выручке и долей выручки с заказов остальных пользователей в общей выручке полученных за этот день.**
```
with rev as (
    SELECT 
        date,
        sum(price) as revenue
    FROM (
        SELECT 
            order_id,
            creation_time::date as date,
            product_ids,
            unnest(product_ids) as product_id
        FROM orders
        where order_id not in ( SELECT order_id
                                    FROM user_actions
                                    where action = 'cancel_order')) as a
    join products using(product_id)
    group by date
    order by date),

new_rev as (
    SELECT
        date,
        sum(price) as new_users_revenue
    FROM (
        SELECT 
            date,
            order_id,
            product_id,
            price
        FROM (
            SELECT 
                order_id,
                creation_time::date as date,
                product_ids,
                unnest(product_ids) as product_id
            FROM orders
            where order_id not in ( SELECT order_id
                                    FROM user_actions
                                    where action = 'cancel_order')) as a
        join products using(product_id) ) AS t1
    JOIN (
        SELECT 
            DISTINCT user_id,
            order_id,
            min(time::date) OVER(PARTITION BY user_id) as user_first_date
        FROM user_actions) as t2
    ON t1.order_id = t2.order_id 
    AND t1.date = t2.user_first_date
    GROUP BY date
    ORDER BY date) 
    
SELECT
    date,
    revenue,
    new_users_revenue,
    round(100 * new_users_revenue / revenue, 2) as new_users_revenue_share,
    round(100 - (100 * new_users_revenue / revenue), 2) as old_users_revenue_share
FROM rev JOIN new_rev USING(date)
```

**prod_2.6 - Динамика долей выручки от продаж товара в общей выручке, полученной за весь период.**
```
with rev as (
    SELECT 
        sum(price) as all_rev
    FROM (
        SELECT 
            order_id,
            creation_time::date as date,
            product_ids,
            unnest(product_ids) as product_id
        FROM orders
        where order_id not in ( SELECT order_id
                                    FROM user_actions
                                    where action = 'cancel_order')) as a
    join products using(product_id)),

prod_names as ( 
    SELECT 
        CASE 
            WHEN round(100 * sum(price) / (SELECT all_rev FROM rev), 2) < 0.5 THEN 'ДРУГОЕ'
            ELSE name
        END AS product_name,
        sum(price) as revenue,
        round(100 * sum(price) / (SELECT all_rev FROM rev), 2) as share_in_revenue
        
    FROM (
        SELECT 
            order_id,
            creation_time::date as date,
            product_ids,
            unnest(product_ids) as product_id
        FROM orders
        where order_id not in ( SELECT order_id
                                    FROM user_actions
                                    where action = 'cancel_order')) as a
    join products using(product_id)
    group by name
    ORDER BY revenue desc)

SELECT
    product_name,
    sum(revenue) as revenue,
    sum(share_in_revenue) as share_in_revenue
FROM prod_names
GROUP BY product_name
ORDER BY revenue desc
```

![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/12f4e874-5a5e-402b-81d6-9640a398f048)
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/f880a10f-a1b3-4d74-8a06-c4f4fc57e914)
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/d5895cde-f45b-4f3c-a74e-9898a777cb50)
