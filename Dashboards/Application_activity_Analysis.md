### Описание:
На дашборде отражен анализ активности пользователей и курьеров в приложении. 

Посчитыны: Количество пользователей, количество платящих пользователей, количество задействованых курьеров и др. Со всеми посчитанными метриками можно ознакомиться по ссылке ниже.

**Ссылка на дашборд**: [Анализ активности приложения](https://redash.public.karpov.courses/dashboards/2970--/ "Ссылка для просмотра Дашборда")

![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/e631d547-0c1d-47af-812b-76d16ee18204)
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/32548c09-5668-4f9d-9a4a-1cfbc26ab985)
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/0d79fa63-7095-4278-b57a-2cbb679d912b)

**1.2_post_dash - Динамика Новых пользователей и курьеров / Общего числа пользователей.**
```
with t1 as (
    SELECT 
        user_id,
        min(time)::date as date
    FROM user_actions
    group BY user_id
    order BY user_id),
    
t2 as (
    SELECT 
        courier_id,
        min(time)::date as date
    FROM courier_actions
    group BY courier_id
    order BY courier_id)

SELECT
    *,
    sum(new_users) over(order by date)::integer as total_users,
    sum(new_couriers) over(order by date)::integer as total_couriers
FROM (
    SELECT
        date,
        count(user_id) as new_users
    FROM t1
    group by date
    ORDER BY date) as a
join (
    SELECT
        date,
        count(courier_id) as new_couriers
    FROM t2
    group by date
    ORDER BY date) as b
using (date)
```

**1.3_post_dash - Динамика прироста Числа новых пользователей и курьеров / Общего числа пользователей и курьеров.**
```
with t1 as (
    SELECT 
        user_id,
        min(time)::date as date
    FROM user_actions
    group BY user_id
    order BY user_id),
    
t2 as (
    SELECT 
        courier_id,
        min(time)::date as date
    FROM courier_actions
    group BY courier_id
    order BY courier_id),

t3 as (
    SELECT
        *,
        sum(new_users) over(order by date)::integer as total_users,
        sum(new_couriers) over(order by date)::integer as total_couriers
    FROM (
        SELECT
            date,
            count(user_id) as new_users
        FROM t1
        group by date
        ORDER BY date) as a
    join (
        SELECT
            date,
            count(courier_id) as new_couriers
        FROM t2
        group by date
        ORDER BY date) as b
    using (date) )
    
SELECT 
    *,
    round((new_users - lag(new_users, 1) over()) / ((lag(new_users, 1) over())*0.01), 2) as new_users_change,
    round((new_couriers - lag(new_couriers, 1) over()) / ((lag(new_couriers, 1) over())*0.01), 2) as new_couriers_change,
    round((total_users - lag(total_users, 1) over()) / ((lag(total_users, 1) over())*0.01), 2) as total_users_growth,
    round((total_couriers - lag(total_couriers, 1) over()) / ((lag(total_couriers, 1) over())*0.01), 2) as total_couriers_growth
FROM t3 
```

**1.4_post_dash - Динамика Платящих пользователей и активных курьеров / Долей платящих пользователей и активных курьеров.**
```
with pay_us as (
    SELECT 
        time::date as date,
        count(DISTINCT user_id) as paying_users
    FROM user_actions
    where order_id not in ( SELECT order_id
                            FROM user_actions
                            where action = 'cancel_order')
    group by time::date),

total_us as (
    SELECT
        *,
        sum(new_users) over(order by date)::integer as total_users
    FROM (
        SELECT
            date,
            count(user_id) as new_users
        FROM (
            SELECT 
                user_id,
                min(time)::date as date
            FROM user_actions
            group BY user_id
            order BY user_id) as t1
        group by date
        ORDER BY date) as a),

act_cour as (
    SELECT 
        time::date as date,
        count(DISTINCT courier_id) as active_couriers 
    FROM courier_actions
    where order_id not in ( SELECT order_id
                            FROM user_actions
                            where action = 'cancel_order') 
    group by time::date),

total_cour as (
    SELECT
        *,
        sum(new_couriers) over(order by date)::integer as total_couriers
    FROM (
       SELECT
            date,
            count(courier_id) as new_couriers
        FROM (
            SELECT 
                courier_id,
                min(time)::date as date
            FROM courier_actions
            group BY courier_id
            order BY courier_id) as t2
        group by date
        ORDER BY date) as b)

SELECT 
    date,
    paying_users,
    active_couriers,
    round(paying_users / (total_users * 0.01), 2) as paying_users_share,
    round(active_couriers / (total_couriers * 0.01), 2) as active_couriers_share
FROM total_us 
join pay_us using(date)
join act_cour using(date)
join total_cour using(date)
```
**1.6_post_dash - Динамика общего числа заказов, числа первых заказов и числа заказов новых пользователей / 
Динамика доли первых заказов и доли заказов новых пользователей в общем числе заказов.**
```
with all_orders as (
    SELECT 
        time::date as date,
        count(order_id) as orders
    FROM user_actions
    where order_id not in ( SELECT order_id
                            FROM user_actions
                            where action = 'cancel_order')
    group by time::date
    ORDER BY date),

order_num as (
    SELECT 
        user_id, order_id, time::date as date,
        min(time::date) over(partition by user_id) as first_date,
        row_number() over(partition by user_id order by time::date) as order_number_one
    FROM user_actions
    where order_id not in ( SELECT order_id
                            FROM user_actions
                            where action = 'cancel_order')),

first_ord as ( 
    SELECT
        date,
        count(order_id) as first_orders
    FROM order_num
    where order_number_one = 1
    group by date),
    
first_activ as (
    SELECT 
        time::date as date,
        user_id
    FROM (
        SELECT time,
                user_id,
                order_id,
                row_number() OVER(PARTITION BY user_id ORDER BY time) as rank
            FROM   user_actions
            ORDER BY user_id, time) as t1
    WHERE  rank = 1
    ORDER BY date, user_id),

count_orders_day as (
    SELECT 
        time::date as date,
        user_id,
        count(order_id) filter( WHERE order_id not in (SELECT order_id
                                FROM   user_actions
                                WHERE  action = 'cancel_order')) as sum_order
    FROM   user_actions
    GROUP BY time::date, user_id
    ORDER BY user_id, date),

new_us_or as (
    SELECT first_activ.date as date,
           sum(sum_order)::integer as new_users_orders
    FROM   first_activ
    LEFT JOIN count_orders_day
            ON first_activ.date = count_orders_day.date and
               first_activ.user_id = count_orders_day.user_id
    GROUP BY first_activ.date
    ORDER BY date)


SELECT
    date,
    orders,
    first_orders,
    new_users_orders,
    round(100 * first_orders::decimal / orders, 2) as first_orders_share,
    round(100 * new_users_orders::decimal / orders, 2) as new_users_orders_share
FROM all_orders 
join first_ord using(date)
join new_us_or using(date)
```

**1.5_post_dash - Доли пользователей с одним и несколькими заказами.**
```
with 

pay_us as (
    SELECT 
        time::date as date,
        count(DISTINCT user_id) as paying_users
    FROM user_actions
    where order_id not in ( SELECT order_id
                            FROM user_actions
                            where action = 'cancel_order')
    group by time::date),

several_orders as (
    SELECT 
        date,
        count(user_id) as several_orders_users
    FROM (
        SELECT 
            user_id,
            time::date as date
        FROM user_actions
        where order_id not in ( SELECT order_id
                                FROM user_actions
                                where action = 'cancel_order')
        group by user_id, time::date
        having count(order_id) > 1) as a
    group by date
    ORDER BY date)
        
SELECT
   date,
   round(100 * (paying_users - several_orders_users)::decimal / paying_users, 2) as single_order_users_share,
   round(100 * several_orders_users::decimal / paying_users, 2) as several_orders_users_share
FROM pay_us join several_orders using(date)
```

**1.9_post_dash - Динамика показателя cancel rate и числа успешных/отменённых заказов.**
```
with hour_ac as (
    SELECT
        DATE_PART('hour', creation_time)::integer as hour,
        count(order_id) as successful_orders
    FROM orders
    WHERE  order_id not in (SELECT order_id
                            FROM   user_actions
                            WHERE  action = 'cancel_order')
    group by DATE_PART('hour', creation_time)::integer
    ORDER BY hour),

hour_can as (
    SELECT
        DATE_PART('hour', creation_time)::integer as hour,
        count(order_id) as canceled_orders
    FROM orders
    WHERE  order_id in (SELECT order_id
                            FROM   user_actions
                            WHERE  action = 'cancel_order')
    group by DATE_PART('hour', creation_time)::integer
    ORDER BY hour)
    
SELECT
    *,
    round(canceled_orders::decimal / (successful_orders + canceled_orders), 3) as cancel_rate
FROM hour_ac join hour_can using(hour)
```

**1.7_post_dash - Динамика числа пользователей и заказов на одного курьера.**
```
with active_cours as (
    SELECT
        time::date as date,
        count(DISTINCT courier_id) as count_couriers
    FROM courier_actions
    where 
          order_id not in ( SELECT order_id
                            FROM user_actions
                            where action = 'cancel_order') 
    group by time::date
    ORDER BY date),

orders as (
    SELECT
        time::date as date,
        count(DISTINCT order_id) as count_orders
    FROM courier_actions
    where 
          action = 'accept_order' and
          order_id not in ( SELECT order_id
                            FROM user_actions
                            where action = 'cancel_order') 
    group by time::date
    ORDER BY date),

active_users as (
    SELECT
    courier_actions.time::date as date, 
    count(DISTINCT user_id) as count_users
FROM courier_actions join user_actions using(order_id)
where order_id not in ( SELECT order_id
                        FROM user_actions
                        where action = 'cancel_order')
      and courier_actions.action = 'accept_order'
group by courier_actions.time::date
ORDER BY date)
                            
SELECT 
    date,
    round(count_users::decimal / count_couriers, 2) as users_per_courier,
    round(count_orders::decimal / count_couriers , 2) as orders_per_courier
FROM active_cours 
join orders using(date)
join active_users using(date)
```

**1.8_post_dash - Динамика среднего времени доставки заказов.**
```
SELECT date,
       round(avg(delivery_time))::int as minutes_to_deliver
FROM   (SELECT order_id,
               max(time::date) as date,
               extract(epoch
        FROM   max(time) - min(time))/60 as delivery_time
        FROM   courier_actions
        WHERE  order_id not in (SELECT order_id
                                FROM   user_actions
                                WHERE  action = 'cancel_order')
        GROUP BY order_id) t
GROUP BY date
ORDER BY date
```


![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/e631d547-0c1d-47af-812b-76d16ee18204)
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/32548c09-5668-4f9d-9a4a-1cfbc26ab985)
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/0d79fa63-7095-4278-b57a-2cbb679d912b)



