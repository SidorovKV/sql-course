# ДЗ 3

### Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1 млн строк

```postgresql
CREATE TABLE IF NOT EXISTS public.homework3 (
	id serial4 PRIMARY KEY,
	value text NOT NULL
);

BEGIN;
INSERT INTO public.homework3 (
    value
)
SELECT
    left(md5(random()::text), 2)
FROM generate_series(1, 1000000);
COMMIT;
```

Проверим, есть ли записи

```postgresql
SELECT * FROM public.homework3 LIMIT 5;
```

| id | value |
|----|-------|
| 1  | 75    |
| 2  | ff    |
| 3  | 8f    |
| 4  | 0c    |
| 5  | 3a    |

### Посмотреть размер файла с таблицей

```postgresql
SELECT
    pg_size_pretty (
        pg_total_relation_size ('homework3')
    ) size;
```

| size  |
|-------|
| 56 MB |

### 5 раз обновить все строчки и добавить к каждой строчке любой символ

```postgresql
DO
$$
BEGIN 
   FOR i IN 1..5 LOOP
      UPDATE public.homework3 SET value = value || left(md5(random()::text), 1);
   END LOOP;
END
$$;
```

Проверим, поменялись ли записи и сколько символов добавилось

```postgresql
SELECT * FROM public.homework3 LIMIT 5;
```

| id | value   |
|----|---------|
| 1  | 75e9e2d |
| 2  | ff41734 |
| 3  | 8f55b95 |
| 4  | 0c22262 |
| 5  | 3aa3523 |

### Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум

```postgresql
SELECT n_dead_tup, last_autovacuum, relname FROM pg_stat_user_tables WHERE relname = 'homework3';
```

| n_dead_tup | last_autovacuum               | relname   |
|------------|-------------------------------|-----------|
| 0          | 2024-10-12 21:28:48.731 +0300 | homework3 |

Автовакуум приходил 20 минут назад. Я отходил, пока строки обновлялись. Видимо слишком надолго.

### Подождать некоторое время, проверяя, пришел ли автовакуум

Приходил, приходил.

### 5 раз обновить все строчки и добавить к каждой строчке любой символ

```postgresql
DO
$$
BEGIN 
   FOR i IN 1..5 LOOP
      UPDATE public.homework3 SET value = value || left(md5(random()::text), 1);
   END LOOP;
END
$$;
```

Проверим, поменялись ли записи и сколько символов добавилось

```postgresql
SELECT * FROM public.homework3 LIMIT 5;
```

| id | value        |
|----|--------------|
| 1  | 75e9e2deaca2 |
| 2  | ff417341a02c |
| 3  | 8f55b9576335 |
| 4  | 0c2226216f0f |
| 5  | 3aa3523eaf6f |

### Посмотреть размер файла с таблицей

```postgresql
SELECT
    pg_size_pretty (
        pg_total_relation_size ('homework3')
    ) size;
```

| size   |
|--------|
| 347 MB |

### Отключить Автовакуум на конкретной таблице

```postgresql
ALTER TABLE public.homework3 SET (autovacuum_enabled = off);
```

### 10 раз обновить все строчки и добавить к каждой строчке любой символ

```postgresql
DO
$$
BEGIN 
   FOR i IN 1..10 LOOP
      UPDATE public.homework3 SET value = value || left(md5(random()::text), 1);
   END LOOP;
END
$$;
```

Проверим, поменялись ли записи и сколько символов добавилось

```postgresql
SELECT * FROM public.homework3 LIMIT 5;
```

| id | value                  |
|----|------------------------|
 1  | 75e9e2deaca2d99e03faec |
 2  | ff417341a02cb7b6b56368 |
 3  | 8f55b95763350bd4c9f123 |
 4  | 0c2226216f0f087b9ed82d |
 5  | 3aa3523eaf6f5e10f37591 |

### Посмотреть размер файла с таблицей

```postgresql
SELECT
    pg_size_pretty (
        pg_total_relation_size ('homework3')
    ) size;
```

| size   |
|--------|
| 699 MB |

### Объясните полученный результат

В первую очередь размер увеличился из-за роста самих строк.
Но основную роль сыграли создающиеся "странички" и мёртвые записи. Давайте проверим кол-во мёртвых записей.

```postgresql
SELECT n_dead_tup, last_autovacuum, relname FROM pg_stat_user_tables WHERE relname = 'homework3';
```

| n_dead_tup | last_autovacuum               | relname   |
|------------|-------------------------------|-----------|
| 9999979    | 2024-10-12 21:28:48.731 +0300 | homework3 |

Проведём вакуум вручную.

```postgresql
VACUUM public.homework3;
```

```postgresql
SELECT
    pg_size_pretty (
        pg_total_relation_size ('homework3')
    ) size;
```

| size   |
|--------|
| 699 MB |

Размер не поменялся. Обидно будет, если нужно было смотреть pg_relation_size ('homework3').

```postgresql
SELECT n_dead_tup, last_autovacuum, relname FROM pg_stat_user_tables WHERE relname = 'homework3';
```

| n_dead_tup | last_autovacuum               | relname   |
|------------|-------------------------------|-----------|
| 0          | 2024-10-12 21:28:48.731 +0300 | homework3 |

Но мёртвые записи отсосало. Хоть в файловой системе место до сих пор занято, но для новых инсертов/апдейтов ПГ
сможет использовать уже созданные "странички".

### Не забудьте включить автовакуум)

```postgresql
ALTER TABLE public.homework3 SET (autovacuum_enabled = on);
```

Ради интереса сделал 

```postgresql
VACUUM FULL public.homework3;

SELECT
    pg_size_pretty (
            pg_total_relation_size ('homework3')
    ) size;
```

| size  |
|-------|
| 79 MB |

Получается около 90% до полного вакуума занимало место в страничках "про запас".
