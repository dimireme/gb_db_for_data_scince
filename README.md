# db_for_data_scince

Домашние работы по курсу "Базы данных для аналитиков" от GeekBrains.

<details>
  <summary>Урок 1. Аналитика в бизнес-задачах.</summary>

1.  Залить в свою БД данные по продажам. Часть таблицы orders в csv, исходник [здесь](https://drive.google.com/drive/folders/1C3HqIJcABblKM2tz8vPGiXTFT7MisrML?usp=sharing).

2.  Проанализировать, какой период данных выгружен.

    ```sql
    SELECT min(o_date), max(o_date) FROM orders;
    ```

    | min(o_date) | max(o_date) |
    | ----------- | ----------- |
    | 2016-01-01  | 2017-12-31  |

3.  Посчитать количество строк, заказов и уникальных пользователей, которые совершали заказы.

    ```sql
    SELECT
     count(id_o) AS total,
     count(DISTINCT id_o) AS unique_orders,
     count(DISTINCT user_id) AS unique_users
    FROM orders;
    ```

    | total   | unique_orders | unique_users |
    | ------- | ------------- | ------------ |
    | 2002804 | 2002804       | 1015119      |

4.  По годам посчитать средний чек, среднее количество заказов на пользователя, сделать вывод, как изменялись эти показатели год от года.

    ```sql
    SELECT
      YEAR(o_date) AS 'year',
      round(avg(price), 0) AS avg_price,
      count(id_o) / count(DISTINCT user_id) AS avg_orders
    FROM orders
    GROUP BY YEAR(o_date);
    ```

    | year | avg_price | avg_orders |
    | ---- | --------- | ---------- |
    | 2016 | 2096      | 1.9352     |
    | 2017 | 2398      | 1.7430     |

5.  Найти количество пользователей, которые покупали в одном году и перестали покупать в следующем.

    ```sql
    SELECT count(t16.user_id) AS 'count' FROM
      (SELECT DISTINCT user_id FROM orders WHERE YEAR(o_date) = 2016) t16
    LEFT JOIN
      (SELECT DISTINCT user_id FROM orders WHERE YEAR(o_date) = 2017) t17
    ON t16.user_id = t17.user_id
    WHERE t17.user_id IS NULL;
    ```

    | count  |
    | ------ |
    | 360225 |

6.  Найти ID самого активного по количеству покупок пользователя.

    ```sql
    SELECT
      user_id,
      count(id_o) AS orders
    FROM orders
    GROUP BY user_id
    ORDER BY orders DESC LIMIT 1;
    ```

    | user_id | orders |
    | ------- | ------ |
    | 765861  | 3183   |

</details>

<details>
  <summary>Урок 3. Типовые методы анализа данных. RFM-анализ.</summary>

Главная задача: сделать RFM-анализ на основе данных по продажам за 2 года.

1.  Определяем критерии для каждой буквы R, F, M (т.е. к примеру, R=3 для клиентов, которые покупали <= 30 дней от последней даты в базе, R=2 для клиентов, которые покупали > 30 и менее 60 дней от последней даты в базе и т.д.).

| номер | r               | f                | m                   |
| ----- | --------------- | ---------------- | ------------------- |
| 1     | 60 < days       | 20 <= period     | spend < 1000        |
| 2     | 30 < days <= 60 | 10 <= period <20 | 1000 <= spend <5000 |
| 3     | days <= 30      | period < 10      | 5000 <= spend       |

При этом, если пользователь совершил менее 4-х покупок, при определении периода f, он попадёт в категорию 1.

2.  Для каждого пользователя получаем набор из 3 цифр (от 111 до 333, где 333 – самые классные пользователи)

    ```sql
    DROP TABLE IF EXISTS rfm_analys;

    CREATE TABLE rfm_analys
    SELECT
      user_id,
      min(o_date) AS first_activity,
      max(o_date) AS last_activity,
      count(id_o) AS orders_count,
      sum(price) AS total_price,
      CASE
        WHEN count(id_o) < 4 THEN "1"
        ELSE (
          CASE
            WHEN (TIMESTAMPDIFF(DAY, min(o_date), max(o_date)) / (count(id_o) - 1)) < 10 THEN "3"
            WHEN (TIMESTAMPDIFF(DAY, min(o_date), max(o_date)) / (count(id_o) - 1)) >= 10 AND (TIMESTAMPDIFF(DAY, min(o_date), max(o_date)) / (count(id_o) - 1)) < 20 THEN "2"
            ELSE "1" END
        ) END AS f,
      CASE
        WHEN sum(price) < 1000 THEN "1"
        WHEN sum(price) >= 1000 AND sum(price) < 5000 THEN "2"
        ELSE "3" END AS m,
      CASE
        WHEN TIMESTAMPDIFF(DAY, max(o_date), date('2018-01-01')) >= 0 AND TIMESTAMPDIFF(DAY, max(o_date), date('2018-01-01')) < 30 THEN "1"
        WHEN TIMESTAMPDIFF(DAY, max(o_date), date('2018-01-01')) >= 30 AND TIMESTAMPDIFF(DAY, max(o_date), date('2018-01-01')) < 60 THEN "2"
        ELSE "3" END AS r
    FROM orders
    GROUP BY user_id;
    ```

3.  Вводим группировку, к примеру, 333 и 233 – это Vip, 1XX – это Lost, остальные Regular ( можете ввести боле глубокую сегментацию)

    ```sql
    SELECT
      count(user_id) AS count_users,
      sum(total_price) AS sum_price,
      r,
      f,
      m
    FROM rfm_analys
    GROUP BY r, f, m;
    ```

<details>
  <summary>результат</summary>

| count_users | sum_price      | r   | f   | m   |
| ----------- | -------------- | --- | --- | --- |
| 24396       | 15168338.500   | 1   | 1   | 1   |
| 53682       | 125086946.600  | 1   | 1   | 2   |
| 20243       | 330105237.000  | 1   | 1   | 3   |
| 44          | 159354.300     | 1   | 2   | 2   |
| 1070        | 72138007.200   | 1   | 2   | 3   |
| 1           | 842.800        | 1   | 3   | 1   |
| 68          | 243700.100     | 1   | 3   | 2   |
| 489         | 176602634.250  | 1   | 3   | 3   |
| 18771       | 11539929.100   | 2   | 1   | 1   |
| 41026       | 96279822.800   | 2   | 1   | 2   |
| 17085       | 256956597.100  | 2   | 1   | 3   |
| 24          | 88804.100      | 2   | 2   | 2   |
| 503         | 26589229.800   | 2   | 2   | 3   |
| 2           | 1253.000       | 2   | 3   | 1   |
| 39          | 134100.400     | 2   | 3   | 2   |
| 162         | 18564756.000   | 2   | 3   | 3   |
| 243762      | 143867896.900  | 3   | 1   | 1   |
| 453734      | 1039986626.700 | 3   | 1   | 2   |
| 133628      | 1605575577.900 | 3   | 1   | 3   |
| 18          | 13876.100      | 3   | 2   | 1   |
| 623         | 2081179.100    | 3   | 2   | 2   |
| 3271        | 82875090.200   | 3   | 2   | 3   |
| 76          | 43045.100      | 3   | 3   | 1   |
| 735         | 2239603.800    | 3   | 3   | 2   |
| 1667        | 64766158.800   | 3   | 3   | 3   |

</details>

Всего пользователей и потраченных ими денег:

```sql
SELECT count(user_id), sum(total_price) FROM rfm_analys;
```

| count(user_id) | sum(total_price) |
| -------------- | ---------------- |
| 1015119        | 4071108607.650   |

Добавим категории пользователей.

```sql
ALTER TABLE rfm_analys ADD category VARCHAR(10);
UPDATE rfm_analys set category = (
  CASE
    WHEN (r='3' OR r='2') AND f = '3' AND m='3' THEN 'vip'
    WHEN r='1'	THEN 'lost'
    ELSE 'regular' END
);
```

4.  Для каждой группы из п. 3 находим количество пользователей, которые попали в них и % товарооборота, которое они сделали на эти 2 года.

    ```sql
    SELECT
      sum(total_price) AS total_spend,
      concat(round(( sum(total_price)/ (SELECT sum(total_price) FROM rfm_analys) * 100 ),2),'%') AS percentage,
      count(user_id) AS users_count,
      category
    FROM rfm_analys
    GROUP BY category
    ORDER BY total_spend DESC;
    ```

    | total_spend    | percentage | users_count | category |
    | -------------- | ---------- | ----------- | -------- |
    | 3268272632.100 | 80.28%     | 913297      | regular  |
    | 719505060.750  | 17.67%     | 99993       | lost     |
    | 83330914.800   | 2.05%      | 1829        | vip      |

5.  Проверяем, что общее кол-во пользователей бьется с суммой кол-ва пользователей по группам из п. 3 (если у вас есть логические ошибки в создании групп, у вас не собьются цифры). То же самое делаем и по деньгам.

    Количество пользователей в пункте 4 `98102 + 8085 + 180 = 106367` совпадает с количеством пользователей в пункте 3.

    Количество потраченных денег в пункте 4 `265941536.70 + 27605275.60 + 7035198.80 = 300582011.1` совпадает со значением в пункте 3.

</details>

<details>
  <summary>Урок 4. Типовая аналитика маркетинговой активности. Кагортный анализ.</summary>

На основе данных по продажам за 16 и 17 год на основе когортного анализа по ГГММ первой покупки спрогнозировать товарооборот января 2018 года (с выводом кэфов поведения когротны по порядковому номеру месяца). Т.е. строим все когорты, понимаем как вымирает когорта. После 14 месяца обычно начинает мерцание на 2-5 процентов от первоначальной суммы. Итого, мы знаем как в среднем живут когорты, строим прогноз на один месяц для уже существующих когорт и предполагаем какой сформируется новая.

Запрос данных для разбивки по кагортам:

```sql
SELECT
  c.cogort,
  date_format((o.o_date), "%Y-%m") AS purchase_date,
  sum(o.price) AS revenue
FROM orders o
JOIN (
  SELECT
    user_id,
    date_format(min(o_date), "%Y-%m") AS cogort
  FROM orders
  GROUP BY user_id
) c
ON o.user_id = c.user_id
GROUP BY c.cogort, date_format((o.o_date), "%Y-%m");
```

Последние 10 строк результата:

| cogort  | purchase_date | revenue       |
| ------- | ------------- | ------------- |
| 2017-09 | 2017-09       | 114721028.800 |
| 2017-09 | 2017-10       | 5214909.700   |
| 2017-09 | 2017-11       | 4504822.000   |
| 2017-09 | 2017-12       | 3960622.400   |
| 2017-10 | 2017-10       | 138653454.800 |
| 2017-10 | 2017-11       | 6344545.200   |
| 2017-10 | 2017-12       | 5199659.500   |
| 2017-11 | 2017-11       | 163478573.300 |
| 2017-11 | 2017-12       | 6732426.400   |
| 2017-12 | 2017-12       | 191394529.900 |

Результаты расчета в файле lesson_4_cagort.csv

Получили распределение по кагортам:

| cogort  | покупка в первом месяце | коэффициент | прогноз      |
| ------- | ----------------------- | ----------- | ------------ |
| 2016-01 | 112520331.35            | 14.64%      | 16470434.68  |
| 2016-02 | 76659972.9              | 7.90%       | 6058704.63   |
| 2016-03 | 89331704                | 6.53%       | 5835736.9    |
| 2016-04 | 87505127.5              | 5.10%       | 4460832.7    |
| 2016-05 | 77422482.9              | 4.61%       | 3567982.48   |
| 2016-06 | 68918992.1              | 4.21%       | 2902944.94   |
| 2016-07 | 71512003.5              | 4.22%       | 3017073.15   |
| 2016-08 | 83235113.5              | 3.92%       | 3259241.07   |
| 2016-09 | 84694696.8              | 4.09%       | 3462294.03   |
| 2016-10 | 106447878.6             | 3.29%       | 3506519.8    |
| 2016-11 | 126087879.4             | 3.18%       | 4004945      |
| 2016-12 | 127987883.1             | 3.44%       | 4407360.63   |
| 2017-01 | 123985677.2             | 10.82%      | 13412565.81  |
| 2017-02 | 104769212.1             | 6.42%       | 6728552.35   |
| 2017-03 | 118447847.7             | 5.30%       | 6275869.32   |
| 2017-04 | 109542408.5             | 4.42%       | 4843086.94   |
| 2017-05 | 129331934.2             | 4.18%       | 5412404.33   |
| 2017-06 | 110123214.6             | 4.23%       | 4653765.93   |
| 2017-07 | 113386903               | 3.53%       | 4002233.86   |
| 2017-08 | 117063009.7             | 3.27%       | 3831333.23   |
| 2017-09 | 114721028.8             | 2.62%       | 3008166.84   |
| 2017-10 | 138653454.8             | 2.53%       | 3509525.1    |
| 2017-11 | 163478573.3             | 1.95%       | 3188852.44   |
| 2017-12 | 191394529.9             | 4.56%       | 8725097.76   |
| 2018-01 | 118253004.28            | 100.00%     | 118253004.28 |

Суммарная прибыль в январе 2018 года составит 246 798 528.19 рублей.

Для кагорт 2016-01 - 2016-11 коэффициент рассчитывался как средний процент покупки от покупки в первом месяце за период с 15 по последующие месяцы с месяца первой покупки.

Для кагорт 2016-12 - 2017-11 коэффициент рассчитывался как среднее от скорости затухания покупок за первые 14 месяцев.

Для кагорты 2017-12 ожидаенмый процент покупок в январе составил 91% от первой покупки, поэтому для этой кагорты был взят процент покупок кагортой 2016-12 в январе 2017 по отношению к покупкам в первом месяце (декабрь 2016). Он составил 4.56%.

Для ожидаемой кагорты 2018-01 объём затрат рассчитывался как средние затраты кагорт 2016-01 и 2017-01 в первом месяце покупки (январь 2016 и январь 2017 соответственно).
