### Сгенерировать таблицу с 1 млн JSONB документов

```postgresql
CREATE TABLE json_table (
    id SERIAL PRIMARY KEY,
    data JSONB
);

CREATE OR REPLACE FUNCTION generate_random_string(
    length INTEGER,
    characters TEXT default '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz'
) RETURNS TEXT AS
$$
DECLARE
    result TEXT := '';
BEGIN
    IF length < 1 then
        RAISE EXCEPTION 'Invalid length';
    END IF;
    FOR __ IN 1..length LOOP
            result := result || substr(characters, floor(random() * length(characters))::int + 1, 1);
        end loop;
    RETURN result;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION generate_random_string(
    length INTEGER,
    characters TEXT default '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz'
) RETURNS TEXT AS
$$
DECLARE
    result TEXT := '';
BEGIN
    IF length < 1 then
        RAISE EXCEPTION 'Invalid length';
    END IF;
    FOR __ IN 1..length LOOP
            result := result || substr(characters, floor(random() * length(characters))::int + 1, 1);
        end loop;
    RETURN result;
END;
$$ LANGUAGE plpgsql;

-- Заполним таблицу 1 млн JSONB документов
DO $$
BEGIN
    FOR i IN 1..1000000 LOOP
        INSERT INTO json_table (data)
        VALUES (
            jsonb_build_object(
                'field1', generate_random_string(100),
                'field2', random() * 1000,
                'field3', random() > 0.5
            )
        );
    END LOOP;
END $$;
```

### Создать индекс

```postgresql
CREATE INDEX json_idx ON json_table USING gin (data jsonb_path_ops);
```

Зафиксируем состояние

```postgresql
SELECT
    n.nspname || '.' || c.relname AS table_name,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size,
    pg_size_pretty(pg_total_relation_size(c.reltoastrelid)) AS toast_size
FROM pg_class c
JOIN pg_namespace n
    ON c.relnamespace = n.oid
WHERE
    relname = 'json_table';
```

| table_name        | total_size | toast_size |
|-------------------|------------|------------|
| public.json_table | 284 MB     | 8192 bytes |

### Обновить 1 из полей в json

Обновим 1 поле в первых 1000 записях и вернём его в +/- исходное состояние (100 символов)
```postgresql
UPDATE json_table
SET data = jsonb_set(data, '{field1}', to_jsonb(generate_random_string(1 + 100000 * random()::int)))
WHERE id <= 1000;

UPDATE json_table
SET data = jsonb_set(data, '{field1}', to_jsonb(generate_random_string(100)))
WHERE id <= 1000;
```

### Убедиться в блоатинге TOAST 

```postgresql
SELECT
    n.nspname || '.' || c.relname AS table_name,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size,
    pg_size_pretty(pg_total_relation_size(c.reltoastrelid)) AS toast_size
FROM pg_class c
JOIN pg_namespace n
    ON c.relnamespace = n.oid
WHERE
    relname = 'json_table';
```

| table_name        | total_size | toast_size |
|-------------------|------------|------------|
| public.json_table | 336 MB     | 52 MB      |

### Придумать методы избавится от него и проверить на практике

1. VACUUM FULL - надёжно, но очень медленно и блокируем таблицу на время вакуума
2. Расширение jsonb_toaster - стороннее опенсорс решение. Позволит эффективнее работать с jsonb-объектами
3. REINDEX CONCURRENTLY - попробуем его на практике, так как доступно "из коробки" и не блокирует таблицу

```postgresql
REINDEX INDEX CONCURRENTLY json_idx;
```

```postgresql
SELECT
    n.nspname || '.' || c.relname AS table_name,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size,
    pg_size_pretty(pg_total_relation_size(c.reltoastrelid)) AS toast_size
FROM pg_class c
JOIN pg_namespace n
    ON c.relnamespace = n.oid
WHERE
    relname = 'json_table';
```

| table_name        | total_size | toast_size |
|-------------------|------------|------------|
| public.json_table | 285 MB     | 632 kB     |