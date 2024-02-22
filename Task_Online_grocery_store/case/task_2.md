## Задание:
Выясните, кто заказывал и доставлял самые большие заказы. Самыми большими считайте заказы с наибольшим числом товаров.
### Код:
```
SELECT order_id,
       user_id,
       user_age,
       courier_id,
       courier_age
FROM   (SELECT order_id,
               courier_id
        FROM   courier_actions
        WHERE  action = 'deliver_order'
           and order_id not in (SELECT order_id
                             FROM   user_actions
                             WHERE  action = 'cancel_order')) as a join (SELECT order_id,
                                                   array_length(product_ids, 1) as count_products
                                            FROM   orders) as b using(order_id)
    LEFT JOIN (SELECT courier_id,
                      date_part('year', age((SELECT max(time)
                                      FROM   user_actions), birth_date))::integer as courier_age
               FROM   couriers) as c using(courier_id)
    LEFT JOIN (SELECT user_id,
                      order_id
               FROM   user_actions) as d using(order_id)
    LEFT JOIN (SELECT user_id,
                      date_part('year', age((SELECT max(time)
                                      FROM   user_actions), birth_date))::integer as user_age
               FROM   users) as e using(user_id)
WHERE  count_products = 9
ORDER BY count_products desc, order_id
```
### Результат:
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/c931f67f-558c-4392-9826-2a4b66555954)
