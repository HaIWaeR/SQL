# DBeaver
- Ctrl+Enter — выполнить один запрос (под курсором)
- Alt+X — выполнить весь скрипт
- F5 — обновить дерево

# Создание 

```sql
CREATE SCHEMA test; -- схема (папка для таблиц)

CREATE TABLE test.phones (
    model VARCHAR(100),
    brand VARCHAR(50),
    year  INT -- после последней колонки запятой НЕТ
);

```
## Типы данных
| Тип | Для чего |
|---|---|
| `INT` | целые: количество, размер, год, id |
| `SERIAL` | автоматический id (1, 2, 3…) |
| `VARCHAR(n)` | короткий текст: название (255), логин (50), телефон (20), имя файла (255) |
| `TEXT` | длинный текст: описание, состав |
| `NUMERIC(10,2)` | деньги |
| `DATE` | дата: `'2026-10-21'` |
| `BOOLEAN` | да/нет |

# Вставка 
### Одна строка
```sql
INSERT INTO test.cats (name, age) VALUES ('Барсик', 3);
```

### Несколько строк
```sql
INSERT INTO test.cats (name, age) VALUES
    ('Мурка', 5),
    ('Пушок', 1);
``` 

### не все колонки (остальные будут NULL)
```sql
INSERT INTO test.cats (name) VALUES ('Без возраста');
```

# Выборка

### SELECT
```sql
-- все колонки
SELECT * FROM test.phones;
-- только нужные колонки
SELECT brand, model FROM test.phones;
-- сколько строк
SELECT count(*) FROM test.phones;
```

### WHERE
```sql
-- одно условие
SELECT * FROM test.phones WHERE year > 2000;
-- оба условия AND или OR
SELECT * FROM test.phones WHERE year > 1500 AND brand = 'samsung'; 
-- без учёта регистра
SELECT * FROM test.phones WHERE brand ILIKE 'Nokia';
```

### ORDER BY
```sql
-- по возрастанию
SELECT * FROM test.phones ORDER BY year;
-- по убыванию DESC
SELECT * FROM test.phones ORDER BY year DESC;
```

# Удаление 
```sql
-- удалить выбранные строки 
delete from test.phones where brand = 'nokia';
-- удалить все строки таблица остаётся 
truncate test.phones;
-- удалить таблицу целиков
drop table test.phones;
-- Удалить таблицу без ошибки если таблицы нету 
drop table if exists test.phones;
-- Удалит схему со всеми таблицами
DROP SCHEMA test CASCADE;
-- очистить таблицы полностью (для перезаливки)
truncate test.sneakers, test.brands restart identity cascade;
```
- перечисленные таблицы очищаются полностью (все строки), порядок в списке не важен
- `restart identity` — сбросить serial, id снова с 1
- `cascade` — очистить ещё и всех, кто ссылается на перечисленные (даже если их не на


# PRIMARY KEY

```sql
CREATE TABLE test.shoes (
    -- первой колонкой в каждой таблице
    id   SERIAL PRIMARY KEY,
    name VARCHAR(255)
);

-- id не указываем
INSERT INTO test.shoes (name) VALUES ('Симы');
-- удалить ровно одну строку
DELETE FROM test.shoes WHERE id = 2;
```

- Сам ставит 1, 2, 3…
- Номера не переиспользуются — дыры после удаления это норма
- id руками НЕ вписывать → счётчик собьётся → `duplicate key`

# FOREIGN  KEY

```sql
-- 1. сначала справочник (на него ссылаются)
CREATE TABLE test.brands (
    id   SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE
);
```
```sql
-- 2. потом таблица, которая ссылается
CREATE TABLE test.sneakers (
    id       SERIAL PRIMARY KEY,
    name     VARCHAR(255) NOT NULL,
    brand_id INT NOT NULL REFERENCES test.brands(id)
);
```

```sql
-- 3. заполнение в том же порядке
INSERT INTO test.brands (name) VALUES ('Nike'), ('Adidas');
INSERT INTO test.sneakers (name, brand_id) VALUES ('Air Max', 1);
```

- Тип ссылки — `INT`, не `SERIAL` (SERIAL генерирует, ссылка хранит чужой id)
- Как выпадающий список — выбрать можно только существующее
- В колонке могут быть только те числа, что есть как `id` в другой таблице

# NOT NULL, UNIQUE, CHECK 

```sql
CREATE TABLE test.users (
    id SERIAL PRIMARY KEY,
    login VARCHAR(50)  NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL CHECK (name <> ''),
    age INT CHECK (age >= 14 AND age <= 120),
    -- без правил: может быть пустым и повторяться
    phone VARCHAR(20)
);
```
- `NOT NULL` — нельзя оставить пустым (название, цена, логин)
- `UNIQUE` — нельзя повторить (логин, email, название в справочнике)
- `CHECK (условие)` — любое условие как в WHERE (цена ≥ 0, количество ≥ 0)  

# DISTINCT 

```sql
-- каждая категория один раз
select distinct category from test.raw_shoes;
-- уникальные пары
select distinct category, manufacturer from test.raw_shoes;
```

- Убирает одинаковые строки из результата
- Таблицу не меняет — только вывод
- Пишется сразу после `select`
- Несколько колонок → убирает строки, где совпадают ВСЕ колонки


# Чистка текста: TRIM, REPLACE, ::
```sql
-- 'Барбари ' → 'Барбари'
select distinct trim(manufacturer) from test.raw_shoes;
-- '25000'
select replace('25 000', ' ', '');
-- '6550.50'
select replace('6550,50', ',', '.');
-- неразрывный пробел → обычный
select replace(name, chr(160), ' ') from ...;
-- текст → число
select price::numeric from test.raw_shoes;
-- вложенные: 25000.50
select replace(replace('25 000,50', ' ', ''), ',', '.')::numeric; 
```

- `trim(x)` — убирает пробелы по краям (внутри не трогает)
- `replace(где, что, на_что)` — заменяет кусок текста
- `x::numeric`, `x::int`, `x::date` — переводит текст в тип

# INSERT ... SELECT — вставить результат запроса
```sql
insert into test.makers (name)
select distinct trim(manufacturer) from test.raw_shoes;
```

- Сначала `select` → проверить результат глазами
- Сверху дописать `insert into таблица (колонки)`
- `values` не пишется, `id` не указывается

# Join 
```sql
select trim(r.name), c.id, s.id, m.id, r.price::numeric
from test.raw_products r
join test.categories    c on c.name = trim(r.category)
join test.subcategories s on s.name = trim(r.subcategory)
join test.manufacturers m on m.name = trim(r.manufacturer);

-- вставка: то же самое + insert сверху
insert into test.products (name, category_id, subcategory_id, manufacturer_id, price, image, description, composition)
select trim(r.name), c.id, s.id, m.id, r.price::numeric, trim(r.image), trim(r.description), trim(r.composition)
from test.raw_products r
join test.categories    c on c.name = trim(r.category)
join test.subcategories s on s.name = trim(r.subcategory)
join test.manufacturers m on m.name = trim(r.manufacturer);
```
- `from` — главная таблица, идём по ней построчно (raw_products)
- каждый `join` пристёгивает один справочник
- `on` — правило пары: название в справочнике = очищенный текст из raw

# LEFT JOIN — найти, что не склеилось

```sql
-- все строки raw, у ненайденных m.id = null
select r.name, r.manufacturer, m.id
from test.raw_products r
left join test.manufacturers m on m.name = trim(r.manufacturer);

-- только потерявшиеся
select r.name, r.manufacturer
from test.raw_products r
left join test.manufacturers m on m.name = trim(r.manufacturer)
where m.id is null;
```# SQL
