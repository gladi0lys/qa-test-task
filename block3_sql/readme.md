# Блок 3. Анализ данных (SQL)

Диалект PostgreSQL. Ниже рабочие запросы и пояснения, почему сделал именно так и на какие грабли тут можно наступить.

## Задача 1. Вторая по величине уникальная зарплата

Таблица `Employee (id int PK, salary int)`. Нужно вернуть вторую по величине уникальную зарплату, а если её нет, то NULL.

```sql
SELECT MAX(salary) AS SecondHighestSalary
FROM Employee
WHERE salary < (SELECT MAX(salary) FROM Employee);
```

Почему так. Ключевое слово в условии это "уникальную". Если максимальная зарплата встречается у нескольких сотрудников (например, две по 9000), то `MAX(... WHERE salary < MAX)` отбросит весь верхний уровень целиком и вернёт следующее значение. Это надёжнее, чем `OFFSET 1`.

Почему не `LIMIT 1 OFFSET 1`. При дубликатах максимума запрос `ORDER BY salary DESC LIMIT 1 OFFSET 1` вернёт то же самое максимальное значение (вторую строку с тем же числом), а не вторую уникальную зарплату. Частая ошибка.

Про NULL. Внешний `MAX()` по пустому набору (когда уникальное значение всего одно или таблица пустая) по стандарту SQL сам вернёт NULL, так что отдельный `CASE` или `COALESCE` не нужен.

Если же задачу захотят расширить до "N-й зарплаты", удобнее оконка:

```sql
SELECT salary AS SecondHighestSalary
FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM Employee
) t
WHERE rnk = 2;
```

Тут есть нюанс: при отсутствии второй зарплаты этот вариант вернёт 0 строк, а не строку с NULL. По условию нужен именно NULL, поэтому первый вариант с `MAX` предпочтительнее. `DENSE_RANK`, а не `RANK`, потому что нужны уникальные значения без пропусков из-за дубликатов.

## Задача 2. Поиск дубликатов email

Таблица `Person (id int PK, email varchar)`. Нужно найти все email, которые встречаются больше одного раза.

```sql
SELECT email
FROM Person
GROUP BY email
HAVING COUNT(*) > 1;
```

Что тут стоит держать в голове. Каждый задублированный email возвращается один раз, а не по строке на каждое вхождение, обычно нужно именно это. По регистру и пробелам: в реальной системе `Ivan@mail.ru` и `ivan@mail.ru` это один и тот же ящик. Если нужна нормализация:

```sql
SELECT LOWER(TRIM(email)) AS email
FROM Person
GROUP BY LOWER(TRIM(email))
HAVING COUNT(*) > 1;
```

Это уже продуктовое решение. Для задачи с LeetCode хватит базового варианта, но на собеседовании стоит проговорить, что "дубликат" это понятие бизнес-уровня. NULL-значения в группировку как дубли не попадут, потому что NULL не равен NULL, и это правильно.

## Задача 3. Клиенты без заказов

Таблицы `Customers (id int PK, name varchar)` и `Orders (id int PK, customerId int FK)`. Нужно вернуть клиентов, которые ни разу не делали заказов.

```sql
SELECT c.name
FROM Customers c
LEFT JOIN Orders o ON o.customerId = c.id
WHERE o.id IS NULL;
```

Почему `LEFT JOIN ... IS NULL`, а не `NOT IN`. Конструкция `NOT IN` опасна с NULL: если в `Orders.customerId` попадётся хотя бы один NULL, то `WHERE c.id NOT IN (SELECT customerId FROM Orders)` вернёт пустой результат вообще для всех строк (срабатывает трёхзначная логика, `id <> NULL` даёт UNKNOWN). Это коварный баг, на чистых данных он не виден.

Корректная и обычно самая быстрая альтернатива это `NOT EXISTS`:

```sql
SELECT c.name
FROM Customers c
WHERE NOT EXISTS (
    SELECT 1 FROM Orders o WHERE o.customerId = c.id
);
```

В PostgreSQL `LEFT JOIN / IS NULL` и `NOT EXISTS` обычно дают одинаковый план, выбор это вопрос читаемости. Оба безопасны к NULL, в отличие от `NOT IN`.
