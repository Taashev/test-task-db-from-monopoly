# #1 Написать функцию на postgres

## Задача:

Дана таблица

```sql
CREATE TABLE public.cats (
    id_cat uuid NULL DEFAULT uuid_generate_v4(),
    "name" int8 NOT NULL GENERATED ALWAYS AS IDENTITY
);
```

Написать функцию, которая будет вставлять в таблицу заданный объём данных в Mb.
Объём должен передаваться параметром в виде числа Mb.

---

## Решение:

### Пользователь и БД

Рекомендуется перед началом работы создать нового пользователя и БД, выдать пользователю права на БД
и работать от имени созданного пользователя в созданной БД.

Создаем пользователя

```sql
CREATE USER test WITH PASSWORD 'test';
```

Создаем базу данных

```sql
CREATE DATABASE monopoly_online;
```

Выдаем пользователю права на БД monopoly_online;

```sql
GRANT ALL PRIVILEGES ON DATABASE monopoly_online TO test; 
```

### Уствновка расширения для генерации UUID

Для возможности генерировать uuid и использовать функцию uuid_generate_v4() нужно установить расширение.
Для этого потребуется выполнить команду по установке расширения от имени суперпользователя в БД monopoly_online.

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

проверим установилось ли расширение в текущей БД

```sql
SELECT * FROM pg_extension WHERE extname = 'uuid-ossp';
```

### Таблица cats

Создаем таблицу cats с необходимыми полями.

```sql
CREATE TABLE public.cats (
    id_cat uuid NULL DEFAULT uuid_generate_v4(),
    "name" int8 NOT NULL GENERATED ALWAYS AS IDENTITY
);
```

### Функция генерации котиков

Опишем функцию которая будет принимать на вход мегабайты и генерировать котиков на эти мегабайты.
(много котиков небывает)

```sql
CREATE OR REPLACE FUNCTION insert_data_mb(mb INTEGER)
RETURNS VOID
AS $$
DECLARE
    -- переводим мегабайты в байты
    count_cats BIGINT := mb * 1024 * 1024 / 48;

    -- счетчик цикла for
    i INTEGER;
BEGIN
    FOR i IN 1..count_cats LOOP
        INSERT INTO public.cats DEFAULT VALUES;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

```

### Тестируем


проверим количество записей и размер таблицы до генерации 

```sql
SELECT count(cats) as count, pg_size_pretty(pg_total_relation_size('cats')) AS total_size FROM cats;
```

генерируем

```sql
SELECT insert_data_mb(1); -- котиков на 1 мегабайт пожалуйста
```

проверим количество записей и размер таблицы после генерации 

```sql
SELECT count(cats) as count, pg_size_pretty(pg_total_relation_size('cats')) AS total_size FROM cats;
```

---

# #2 Архитектура БД

## Задача:

Предложить концепцию универсального справочника. Справочник должен отвечать требованиям:

1. Иметь возможность не ограничено расширяться под новые типы справочных данных (emp, departments, clients, etc...)
2. Каждый тип может не ограничено наращивать количество полей
3. Справочник должен состоять не более чем из 3-х таблиц.

## Решение:

У нас будет 3 таблицы: directories, directory_fields, field_values.

### Таблица directories - различные справочники (emp, departments, clients, etc...).
В таблице содежратся поля:
- id - первичный ключ
- name - название справочника

### Таблица directory_fields - поля конкретного справочника их может быть неограниченное количество.
В таблице содежратся поля:
- id - первичный ключ
- directory_id - внешний ключ для связи со справочниками
- name - название поля

### Таблица field_values - значение полей.
В таблице содежратся поля:
- id - первичный ключ
- field_id - внешний ключ для связи с полями
- value - значение поля

### Создадим таблицы

```sql
CREATE TABLE directories (
    id UUID DEFAULT uuid_generate_v4() UNIQUE NOT NULL,
    name VARCHAR NOT NULL UNIQUE
);
```

```sql
CREATE TABLE directory_fields (
    id UUID DEFAULT uuid_generate_v4() UNIQUE NOT NULL,
    directory_id UUID NOT NULL,
    name VARCHAR NOT NULL,
    FOREIGN KEY (directory_id) REFERENCES directories(id) ON DELETE CASCADE,
    CONSTRAINT unique_directory_name UNIQUE (directory_id, name)
);
```

```sql
CREATE TABLE field_values (
    id UUID DEFAULT uuid_generate_v4() UNIQUE NOT NULL,
    field_id UUID NOT NULL,
    value VARCHAR,
    FOREIGN KEY (field_id) REFERENCES directory_fields(id) ON DELETE CASCADE
);

```
