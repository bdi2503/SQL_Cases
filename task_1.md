## Задание:
Произведите замену списков с id товаров на списки с наименованиями товаров.
### Код:

SELECT order_id,
       array_agg(name) as product_names
FROM   (SELECT order_id,
               product_ids,
               unnest(product_ids) as product_id
        FROM   orders) as a
    LEFT JOIN (SELECT product_id,
                      name
               FROM   products) as b using(product_id)
GROUP BY order_id limit 1000

### Результат:
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/da1cb210-8599-47b0-9e29-ae693d5a8b4e)





