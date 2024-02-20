## Задание:
На основе информации в таблицах рассчитайте ежедневную выручку сервиса и отразите её в колонке daily_revenue. Затем посчитайте ежедневный прирост выручки. Прирост выручки отразите как в абсолютных значениях, так и в % относительно предыдущего дня.
### Код:
```
with orders_prices as (SELECT order_id,
                              creation_time,
                              sum(price) as order_price
                       FROM   (SELECT order_id,
                                      creation_time,
                                      product_ids,
                                      unnest(product_ids) as product_id
                               FROM   orders
                               WHERE  order_id not in (SELECT order_id
                                                       FROM   user_actions
                                                       WHERE  action = 'cancel_order')) as a join (SELECT product_id,
                                                                          price
                                                                   FROM   products) as b using (product_id)
                       GROUP BY order_id, creation_time), growth_revenue as (SELECT creation_time::date as date,
                                                             sum(order_price) as daily_revenue
                                                      FROM   orders_prices
                                                      GROUP BY creation_time::date
                                                      ORDER BY creation_time::date)
SELECT *,
       coalesce(daily_revenue - lag(daily_revenue, 1) OVER(ORDER BY date),
                0) as revenue_growth_abs,
       round((coalesce(daily_revenue - lag(daily_revenue, 1) OVER(ORDER BY date), 0) / coalesce((lag(daily_revenue*0.01, 1) OVER(ORDER BY date)) , 1)),
             1) as revenue_growth_percentage
FROM   growth_revenue
```
### Результат:
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/4c7c3fde-5cac-4464-8a42-ac92881703f1)
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/43acc90c-2980-41d8-83b7-0cff22a772ef)
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/199c7540-6772-48ad-9175-f5dbb9dc8dad)


