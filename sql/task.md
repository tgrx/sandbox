# Задача

Есть две сущности: `Customer` и `Purchase`. Связь — one-to-many.

```sql
CREATE TABLE customers
AS
SELECT * FROM (VALUES
    (1, 'Alice'),
    (2, 'Bob'),
    (3, 'Charlie')
) AS _v(cid, customer);
```

и

```sql
CREATE TABLE purchases
AS
SELECT * FROM (VALUES
    (1, 1, 'Apple'),
    (2, 1, 'Banana'),
    (3, 2, 'Phone'),
    (4, 2, 'MacBook'),
    (5, 2, 'Coca-Cola'),
    (6, 3, 'Ferrari')
) AS _v(pid, cid, good);
```

В SQL, конструкции LIMIT и OFFSET применяются к таблице-результату,
и ограничивают отступ и количество **строк**.

Все данные:

```sql
SELECT * FROM customers JOIN purchases USING (cid);
```

| cid | customer | pid |   good    |
|-----|----------|-----|-----------|
|   1 | Alice    |   1 | Apple     |
|   1 | Alice    |   2 | Banana    |
|   2 | Bob      |   3 | Phone     |
|   2 | Bob      |   4 | MacBook   |
|   2 | Bob      |   5 | Coca-Cola |
|   3 | Charlie  |   6 | Ferrari   |

Применяем `LIMIT`/`OFFSET`:

```sql
SELECT * FROM customers JOIN purchases USING (cid)
LIMIT 1 OFFSET 1;
```

| cid | customer | pid |   good    |
|-----|----------|-----|-----------|
|   1 | Alice    |   2 | Banana    |

В этом случае `LIMIT 1` означает "отдать одну **строку**",
а `OFFSET 1` означает "пропустить одну **строку**".

Но что, если нужно ограничить/пропустить не количество **строк**,
а количество **объектов**? В этом примере, объект — это `Customer`,
а `Purchase` — это его атрибут. 

Иными словами — как бы выглядел запрос:
```sql
SELECT ... -- какой запрос?
```

чтобы он возвращал аналог `LIMIT 1` / `OFFSET 1` для **объектов**
— "пропустить 1 покупателя", "отдать 1 покупателя"? Естественно, с атрибутами:

| cid | customer | pid |   good    |
|-----|----------|-----|-----------|
|   ~~1~~ | ~~Alice~~    |   ~~1~~ | ~~Apple~~     |
|   ~~1~~ | ~~Alice~~    |   ~~2~~ | ~~Banana~~    |
|   2 | Bob      |   3 | Phone     |
|   2 | Bob      |   4 | MacBook   |
|   2 | Bob      |   5 | Coca-Cola |
|   ~~3~~ | ~~Charlie~~  |   ~~6~~ | ~~Ferrari~~   |

<style>
hide {
  background-color: #d6d6d6;
  color: #d6d6d6;
}
hide:hover {
  background-color: white;
  color: black;
}
</style>

<details><summary>Подсказка:</summary>

```sql
SELECT
    row_number() OVER (), -- туть
    *
FROM
    customers
    JOIN purchases USING (cid)
;
```

| row_number | cid | customer | pid |   good    |
|------------|-----|----------|-----|-----------|
|          1 |   1 | Alice    |   1 | Apple     |
|          2 |   1 | Alice    |   2 | Banana    |
|          3 |   2 | Bob      |   3 | Phone     |
|          4 |   2 | Bob      |   4 | MacBook   |
|          5 |   2 | Bob      |   5 | Coca-Cola |
|          6 |   3 | Charlie  |   6 | Ferrari   |


</details>