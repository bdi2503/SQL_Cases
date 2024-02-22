## Задание:
Для каждого пользователя в каждый момент времени посчитайте две накопительные суммы — числа оформленных и числа отменённых заказов. Если пользователь оформляет заказ, то число оформленных им заказов увеличивайте на 1, если отменяет — увеличивайте на 1 количество отмен.
### Код:
```
SELECT *,
       count(order_id) filter(WHERE action = 'create_order') OVER(PARTITION BY user_id
                                                                  ORDER BY time) as created_orders,
       count(order_id) filter(WHERE action = 'cancel_order') OVER(PARTITION BY user_id
                                                                  ORDER BY time) as canceled_orders,
       round((((count(order_id) filter(WHERE action = 'cancel_order') OVER(PARTITION BY user_id
                                                                           ORDER BY time))::numeric) / ((count(order_id) filter(WHERE action = 'create_order') OVER(PARTITION BY user_id ORDER BY time))::numeric)), 2) as cancel_rate
FROM   user_actions
ORDER BY user_id, order_id, time
LIMIT 1000
```
### Результат:
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/d16eb817-ac8e-4386-84b4-418a56f41f51)
![image](https://github.com/bdi2503/SQL_works_online_grocery_store/assets/142053096/4ba19844-8be7-4878-b26d-520f2484264d)

