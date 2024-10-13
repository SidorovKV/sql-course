# ДЗ 4

### Создать таблицу accounts(id integer, amount numeric);

```postgresql
CREATE TABLE accounts(id integer, amount numeric);
```

### Добавить несколько записей и подключившись через 2 терминала добиться ситуации взаимоблокировки (deadlock).

```postgresql
INSERT INTO accounts VALUES (1,10.0), (2,30.0), (3,50.0);
```

Первый терминал

```postgresql
BEGIN;
UPDATE accounts SET amount = amount + 15.0 WHERE id = 1;
```

Второй терминал

```postgresql
BEGIN;
UPDATE accounts SET amount = amount - 7.0 WHERE id = 2;
```

Первый терминал

```postgresql
UPDATE accounts SET amount = amount + 5.0 WHERE id = 2;
```

Транзакция в первом терминале зависла, так как ждёт разблокировки записи во втором терминале.

Второй терминал

```postgresql
UPDATE accounts SET amount = amount - 18.0 WHERE id = 1;
```

А вот и дедлок

```
SQL Error [40P01]: ERROR: deadlock detected
  Detail: Process 133401 waits for ShareLock on transaction 874; blocked by process 133399.
Process 133399 waits for ShareLock on transaction 875; blocked by process 133401.
  Hint: See server log for query details.
  Where: while updating tuple (0,8) in relation "accounts"
```

### Посмотреть логи и убедиться, что информация о дедлоке туда попала.

```
2024-10-13 11:26:27.481 MSK [133401] kostik@course ERROR: deadlock detected
2024-10-13 11:26:27.481 MSK [133401] kostik@course DETAIL: Process 133101 waits for ShareLock on transaction 874: blocked by process 133399.
Process 133399 waits for Sharelock on transaction 875: blocked by process 133401. Process 133401: UPDATE accounts SET amount = amount - 18.0 WHERE id = 1 Process 133399: UPDATE accounts SET amount = amount + 5.0 WHERE id = 2
2024-10-13 11:26:27.481 MSK [133401] kostik@course HINT: See server log for query details. 
2024-10-13 11:26:27.481 MSK [133401] kostik@course CONTEXT: while updating tuple (0,8) in relation "accounts" 
2024-10-13 11:26:27.481 MSK [133401] kostik@course STATEMENT: UPDATE accounts SET amount = amount - 18.0 WHERE id = 1 
```