# ДЗ 8

### Создать таблицу с продажами.

```postgresql
CREATE TABLE IF NOT EXISTS sales (
	id int8 NOT NULL UNIQUE,
	volume int4 NOT NULL,
	income numeric NOT NULL,
	date timestamp NOT NULL
);

INSERT INTO public.sales
    (volume, income, date)
VALUES (100, 6000, '2023-01-15'),
       (143, 4754, '2024-01-15'),
       (250, 6387, '2024-02-23'),
       (654, 12075, '2024-03-15'),
       (744, 15954, '2024-04-25'),
       (4433, 600400, '2024-05-15'),
       (537, 9542, '2024-06-15'),
       (428, 4678, '2024-07-15'),
       (3567, 36478, '2024-08-15'),
       (36789, 6083600, '2024-09-15'),
       (4567, 506000, '2024-10-15'),
       (1009, 26000, '2024-11-15'),
       (544, 16000, '2024-12-15')
;
```

### Реализовать функцию выбор трети года (1-4 месяц - первая треть, 5-8 - вторая и тд)
   
#### через case

```postgresql
CREATE OR REPLACE FUNCTION select_third(part int) RETURNS TABLE (volume int4, income numeric, "date" timestamp) AS
$BODY$
BEGIN
	IF part > 3 OR part < 1 THEN
		RETURN;
	END IF;

	RETURN QUERY 
		SELECT s.volume, s.income, s."date" FROM public.sales s
		WHERE CASE
			WHEN part = 1 THEN EXTRACT(MONTH FROM s.date) IN (1,2,3,4)
			WHEN part = 2 THEN EXTRACT(MONTH FROM s.date) IN (5,6,7,8)
			WHEN part = 3 THEN EXTRACT(MONTH FROM s.date) IN (9,10,11,12)
		END;
END;
$BODY$
LANGUAGE plpgsql;
```


### используя математическую операцию  и предусмотреть NULL на входе

```postgresql
CREATE OR REPLACE FUNCTION select_third_v2(part int default 0) RETURNS TABLE (volume int4, income numeric, "date" timestamp) AS
$BODY$
BEGIN
	IF part > 3 OR part < 1 THEN
		RETURN;
	END IF;

	RETURN QUERY 
		SELECT s.volume, s.income, s."date" FROM public.sales s
		WHERE EXTRACT(MONTH FROM s.date) > (part - 1) * 4 AND EXTRACT(MONTH FROM s.date) <= part * 4;
END;
$BODY$
LANGUAGE plpgsql;
```

### Вызвать эту функцию в SELECT из таблицы с продажами, уведиться, что всё отработало

```postgresql
select volume, income, "date"::date from select_third(1);
```

| volume | income | date       |
|--------|--------|------------|
| 100    | 6000   | 2023-01-15 |
| 143    | 4754   | 2024-01-15 |
| 250    | 6387   | 2024-02-23 |
| 654    | 12075  | 2024-03-15 |
| 744    | 15954  | 2024-04-25 |

```postgresql
select volume, income, "date"::date from select_third(2);
```

| volume | income | date       |
|--------|--------|------------|
| 4433   | 600400 | 2024-05-15 |
| 537    | 9542   | 2024-06-15 |
| 428    | 4678   | 2024-07-15 |
| 3567   | 36478  | 2024-08-15 |

-----

Вторая функция

```postgresql
select volume, income, "date"::date from select_third_v2(3);
```

| volume | income  | date       |
|--------|---------|------------|
| 36789  | 6083600 | 2024-09-15 |
| 4567   | 506000  | 2024-10-15 |
| 1009   | 26000   | 2024-11-15 |
| 544    | 16000   | 2024-12-15 |

```postgresql
select volume, income, "date"::date from select_third_v2();
```

| volume | income  | date       |
|--------|---------|------------|