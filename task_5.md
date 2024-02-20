## Задание:
На основе информации в таблицах рассчитайте стоимость каждого заказа, ежедневную выручку сервиса и долю стоимости каждого заказа в ежедневной выручке, выраженную в процентах.
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
                       GROUP BY order_id, creation_time)
SELECT order_id,
       creation_time,
       order_price,
       sum(order_price) OVER(PARTITION BY creation_time::date) as daily_revenue,
       round(((order_price)*100 / (sum(order_price) OVER(PARTITION BY creation_time::date))),
             3) as percentage_of_daily_revenue
FROM   orders_prices
ORDER BY creation_time::date desc, percentage_of_daily_revenue desc, order_id
LIMIT 1000
```
### Результат:
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/b3a54037-270b-4cea-8882-0fd098dbfc71)
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/fafc164d-641c-4332-baf4-5483d48fcc14)

