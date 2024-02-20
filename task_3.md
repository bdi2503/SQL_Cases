## Задание:
Выясните, какие пары товаров покупают вместе чаще всего.
### Код:
```
SELECT array_cat(array[name_1], array[name_2]) as pair,
       count(array_cat(array[name_1], array[name_2])) as count_pair
FROM   (SELECT DISTINCT order_id,
                        name_1,
                        name_2
        FROM   (SELECT order_id,
                       a.name as name_1,
                       b.name as name_2
                FROM   (SELECT order_id,
                               name
                        FROM   (SELECT DISTINCT order_id,
                                                unnest(product_ids) as product_id
                                FROM   orders
                                WHERE  order_id not in (SELECT order_id
                                                        FROM   user_actions
                                                        WHERE  action = 'cancel_order')) as tt1
                            LEFT JOIN products using (product_id)) as a
                    INNER JOIN (SELECT order_id,
                                       name
                                FROM   (SELECT DISTINCT order_id,
                                                        unnest(product_ids) as product_id
                                        FROM   orders
                                        WHERE  order_id not in (SELECT order_id
                                                                FROM   user_actions
                                                                WHERE  action = 'cancel_order')) as tt2
                                    LEFT JOIN products using (product_id)) as b using (order_id)
                WHERE  a.name != b.name) as t1
        WHERE  name_1 < name_2
        ORDER BY order_id, name_1, name_2) ppp
GROUP BY array_cat(array[name_1], array[name_2])
ORDER BY count_pair desc, pair
```
### Результат:
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/109aee11-58fa-4aaa-a052-79b9023f66ab)


