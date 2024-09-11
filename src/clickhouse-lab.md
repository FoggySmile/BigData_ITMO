# ClickHouse Lab Report
Наталья Кузьмина, J4140

## Cхемы таблиц

### Основная таблица с транзакциями

```sql
CREATE TABLE nkuzmina_411676.transactions ON CLUSTER kube_clickhouse_cluster
(
    user_id_out Int64,
    user_id_in Int64,
    important UInt8,
    amount Float32,
    datetime DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(datetime)
ORDER BY (user_id_out, user_id_in, datetime);
```

### Создание распределенной таблицы с транзакциями
```
CREATE TABLE nkuzmina_411676.distr_transactions 
ON CLUSTER kube_clickhouse_cluster AS nkuzmina_411676.transactions
ENGINE = Distributed(kube_clickhouse_cluster, 
nkuzmina_411676, transactions, xxHash64(user_id_out));
```

## Обоснование выбранного выражения шардинга
Был выбран xxHash64(user_id_out) в качестве выражения для шардинга.
Поле user_id_out является уникальным идентификатором пользователя, участвующего в транзакции. Это гарантирует, что каждая транзакция будет ассоциироваться с конкретным пользователем. 
В случае с user_id_in распределение может быть менее равномерным, так как некоторые получатели могут получать значительно больше транзакций (например, популярные аккаунты или аккаунты компаний), что может привести к дисбалансу в распределении данных по узлам.

## Список выбранных MV и запросы на их создание

### Суммы по входящим и исходящим транзакциям по месяцам для каждого пользователя

1. MV и распределенная таблица для исходящих транзакций
```
CREATE MATERIALIZED VIEW nkuzmina_411676.sum_out_trans_by_month 
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree()
ORDER BY (user_id, month)
POPULATE AS
SELECT
    user_id_out AS user_id,
    toYYYYMM(datetime) AS month,
    amount AS sum_out_amount
FROM nkuzmina_411676.transactions;
```
```
CREATE TABLE nkuzmina_411676.distr_sum_out_trans_by_month
ON CLUSTER kube_clickhouse_cluster AS nkuzmina_411676.sum_out_trans_by_month
ENGINE = Distributed(kube_clickhouse_cluster, nkuzmina_411676,
sum_out_trans_by_month, xxHash64(user_id));
```
2. MV и распределенная таблица для входящих транзакций
```
CREATE MATERIALIZED VIEW nkuzmina_411676.sum_in_trans_by_month 
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree()
ORDER BY (user_id, month)
POPULATE AS
SELECT
    user_id_in AS user_id,
    toYYYYMM(datetime) AS month,
    amount AS sum_in_amount
FROM nkuzmina_411676.transactions;
```
```
CREATE TABLE nkuzmina_411676.distr_sum_in_trans_by_month 
ON CLUSTER kube_clickhouse_cluster AS nkuzmina_411676.sum_in_trans_by_month
ENGINE = Distributed(kube_clickhouse_cluster, nkuzmina_411676,
sum_in_trans_by_month, xxHash64(user_id));
```
3. Итоговое представление для объединения данных
```
CREATE VIEW nkuzmina_411676.sum_trans_by_month AS
SELECT
    user_id,
    month,
    sum(sum_in_amount) AS sum_in_amount,
    sum(sum_out_amount) AS sum_out_amount
FROM
(
    SELECT
        user_id,
        month,
        sum_in_amount,
        0 AS sum_out_amount
    FROM
        nkuzmina_411676.distr_sum_in_trans_by_month
    UNION ALL
    SELECT
        user_id,
        month,
        0 AS sum_in_amount,
        sum_out_amount
    FROM
        nkuzmina_411676.distr_sum_out_trans_by_month
) AS combined
GROUP BY user_id, month;
```
### Баланс пользователя на текущий момент

1. MV и распределенная таблица для исходящих транзакций
```
CREATE MATERIALIZED VIEW nkuzmina_411676.sum_out_for_saldo
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree()
ORDER BY user_id
POPULATE AS
SELECT
    user_id_out AS user_id,
    -sum(amount) AS balance
FROM nkuzmina_411676.transactions
GROUP BY user_id_out;
```
```
CREATE TABLE nkuzmina_411676.distr_sum_out_for_saldo
ON CLUSTER kube_clickhouse_cluster AS nkuzmina_411676.sum_out_for_saldo
ENGINE = Distributed(kube_clickhouse_cluster, nkuzmina_411676,
sum_out_for_saldo, xxHash64(user_id));
```
2. MV и распределенная таблица для входящих транзакций
```
CREATE MATERIALIZED VIEW nkuzmina_411676.sum_in_for_saldo 
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree()
ORDER BY user_id
POPULATE AS
SELECT
    user_id_in AS user_id,
    sum(amount) AS balance
FROM nkuzmina_411676.transactions
GROUP BY user_id_in;
```
```
CREATE TABLE nkuzmina_411676.distr_sum_in_for_saldo
ON CLUSTER kube_clickhouse_cluster AS nkuzmina_411676.sum_in_for_saldo
ENGINE = Distributed(kube_clickhouse_cluster, nkuzmina_411676,
sum_in_for_saldo, xxHash64(user_id));
```
3. Итоговое представление для объединения данных
```
CREATE VIEW nkuzmina_411676.user_saldo AS
SELECT
    user_id,
    sum(balance) AS saldo
FROM
(
    SELECT
        user_id,
        balance
    FROM
        nkuzmina_411676.distr_sum_in_for_saldo
    UNION ALL
    SELECT
        user_id,
        balance
    FROM
        nkuzmina_411676.distr_sum_out_for_saldo
) AS combined
GROUP BY user_id;
```

### Количество важных транзакций для входящих и исходящих транзакций по месяцам и дням для каждого пользователя.

1. MV и распределенная таблица для количества важных исходящих транзакций по месяцам
```
CREATE MATERIALIZED VIEW nkuzmina_411676.important_out_trans_by_month 
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree()
ORDER BY (user_id, month)
POPULATE AS
SELECT
    user_id_out AS user_id,
    toYYYYMM(datetime) AS month,
    count() AS important_out_count
FROM nkuzmina_411676.transactions
WHERE important = 1
GROUP BY user_id_out, month;
```
```
CREATE TABLE nkuzmina_411676.distr_important_out_trans_by_month 
ON CLUSTER kube_clickhouse_cluster AS nkuzmina_411676.important_out_trans_by_month
ENGINE = Distributed(kube_clickhouse_cluster, nkuzmina_411676,
important_out_trans_by_month, xxHash64(user_id));
```
2. MV и распределенная таблица для количества важных входящих транзакций по месяцам
```
CREATE MATERIALIZED VIEW nkuzmina_411676.important_in_trans_by_month 
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree()
ORDER BY (user_id, month)
POPULATE AS
SELECT
    user_id_in AS user_id,
    toYYYYMM(datetime) AS month,
    count() AS important_in_count
FROM nkuzmina_411676.transactions
WHERE important = 1
GROUP BY user_id_in, month;
```
```
CREATE TABLE nkuzmina_411676.distr_important_in_trans_by_month 
ON CLUSTER kube_clickhouse_cluster AS nkuzmina_411676.important_in_trans_by_month
ENGINE = Distributed(kube_clickhouse_cluster, nkuzmina_411676,
important_in_trans_by_month, xxHash64(user_id));
```
3. MV и распределенная таблица для количества важных исходящих транзакций по дням
```
CREATE MATERIALIZED VIEW nkuzmina_411676.important_out_trans_by_day 
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree()
ORDER BY (user_id, day)
POPULATE AS
SELECT
    user_id_out AS user_id,
    toYYYYMMDD(datetime) AS day,
    count() AS important_out_count
FROM nkuzmina_411676.transactions
WHERE important = 1
GROUP BY user_id_out, day;
```
```
CREATE TABLE nkuzmina_411676.distr_important_out_trans_by_day 
ON CLUSTER kube_clickhouse_cluster AS nkuzmina_411676.important_out_trans_by_day
ENGINE = Distributed(kube_clickhouse_cluster, nkuzmina_411676,
important_out_trans_by_day, xxHash64(user_id));
```
4. MV и распределенная таблица для количества важных входящих транзакций по дням
```
CREATE MATERIALIZED VIEW nkuzmina_411676.important_in_trans_by_day 
ON CLUSTER kube_clickhouse_cluster
ENGINE = SummingMergeTree()
ORDER BY (user_id, day)
POPULATE AS
SELECT
    user_id_in AS user_id,
    toYYYYMMDD(datetime) AS day,
    count() AS important_in_count
FROM nkuzmina_411676.transactions
WHERE important = 1
GROUP BY user_id_in, day;
```
```
CREATE TABLE nkuzmina_411676.distr_important_in_trans_by_day 
ON CLUSTER kube_clickhouse_cluster AS nkuzmina_411676.important_in_trans_by_day
ENGINE = Distributed(kube_clickhouse_cluster, nkuzmina_411676,
important_in_trans_by_day, xxHash64(user_id));
```
5. Итоговое представление для объединения данных (по месяцам)
```
CREATE VIEW nkuzmina_411676.important_trans_by_month AS
SELECT
    user_id,
    month,
    sum(important_in_count) AS important_in_count,
    sum(important_out_count) AS important_out_count
FROM
(
    SELECT
        user_id,
        month,
        important_in_count,
        0 AS important_out_count
    FROM
        nkuzmina_411676.distr_important_in_trans_by_month
    UNION ALL
    SELECT
        user_id,
        month,
        0 AS important_in_count,
        important_out_count
    FROM
        nkuzmina_411676.distr_important_out_trans_by_month
) AS combined
GROUP BY user_id, month;
```
6. Итоговое представление для объединения данных (по дням)
```
CREATE VIEW nkuzmina_411676.important_trans_by_day AS
SELECT
    user_id,
    day,
    sum(important_in_count) AS important_in_count,
    sum(important_out_count) AS important_out_count
FROM
(
    SELECT
        user_id,
        day,
        important_in_count,
        0 AS important_out_count
    FROM
        nkuzmina_411676.distr_important_in_trans_by_day
    UNION ALL
    SELECT
        user_id,
        day,
        0 AS important_in_count,
        important_out_count
    FROM
        nkuzmina_411676.distr_important_out_trans_by_day
) AS combined
GROUP BY user_id, day;
```
## Полученная DWH
```
SHOW TABLES FROM nkuzmina_411676
```
```
┌─name───────────────────────────────────────────┐
│ .inner_id.349df8d0-c43f-49e8-b49d-f8d0c43f79e8 │
│ .inner_id.61c4cb09-2656-46fb-a1c4-cb09265636fb │
│ .inner_id.817b3487-4f90-4263-817b-34874f90c263 │
│ .inner_id.93123474-4395-4c56-9312-347443958c56 │
│ .inner_id.990956b5-7914-4052-9909-56b579142052 │
│ .inner_id.b1da2f14-2a5e-4c2e-b1da-2f142a5e4c2e │
│ .inner_id.fb3fb2ed-fc86-4230-bb3f-b2edfc865230 │
│ .inner_id.fcb0f4fb-9770-4b4e-bcb0-f4fb9770cb4e │
│ distr_important_in_trans_by_day                │
│ distr_important_in_trans_by_month              │
│ distr_important_out_trans_by_day               │
│ distr_important_out_trans_by_month             │
│ distr_sum_in_for_saldo                         │
│ distr_sum_in_trans_by_month                    │
│ distr_sum_out_for_saldo                        │
│ distr_sum_out_trans_by_month                   │
│ distr_transactions                             │
│ important_in_trans_by_day                      │
│ important_in_trans_by_month                    │
│ important_out_trans_by_day                     │
│ important_out_trans_by_month                   │
│ important_trans_by_day                         │
│ important_trans_by_month                       │
│ sum_in_for_saldo                               │
│ sum_in_trans_by_month                          │
│ sum_out_for_saldo                              │
│ sum_out_trans_by_month                         │
│ sum_trans_by_month                             │
│ transactions                                   │
│ user_saldo                                     │
└────────────────────────────────────────────────┘
```
__________________________________
SELECT
    user_id,
    month,
    SUM(amount_received) AS sum_received,
    SUM(amount_outgoing) AS sum_outgoing
FROM
(
    SELECT
        user_id_in AS user_id,
        toYYYYMM(datetime) AS month,
        SUM(amount) AS amount_received,
        0 AS amount_outgoing
    FROM
        nkuzmina_411676.distr_transactions
    GROUP BY
        user_id, month

    UNION ALL

    SELECT
        user_id_out AS user_id,
        toYYYYMM(datetime) AS month,
        0 AS amount_received,
        SUM(amount) AS amount_outgoing
    FROM
        nkuzmina_411676.distr_transactions
    GROUP BY
        user_id, month
) AS monthly_transactions
GROUP BY
    user_id, month
ORDER BY
    user_id, month limit 10;

select * from nkuzmina_411676.sum_trans_by_month where user_id = 2902