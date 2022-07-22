# Isolations & locks

Spin up Percona and Postgres containers and create an InnoDB table.
By changing isolation levels and making parallel queries, reproduce the main problems of parallel access: lost updates, dirty reads, non-repeatable reads, phantom reads.

## Percona Session 1
```
SET autocommit=0;
SET GLOBAL innodb_status_output=ON;
SET GLOBAL innodb_status_output_locks=ON;

show engine innodb status;

SELECT @@transaction_ISOLATION;
```

### Dirty read
```
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

SELECT age FROM users WHERE id = 1;
-- run Session 2
SELECT age FROM users WHERE id = 1; -- Here is wrong value
-- run Session 2
SELECT age FROM users WHERE id = 1;
```

### Non-repeatable reads
```
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

START TRANSACTION;
SELECT * FROM users WHERE id = 1;
-- run Session 2
SELECT * FROM users WHERE id = 1;
COMMIT;
```

### Phantom reads
```
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ ;

START TRANSACTION;
SELECT * FROM users
WHERE age BETWEEN 10 AND 30;
-- run Session 2
SELECT * FROM users
WHERE age BETWEEN 10 AND 30;
COMMIT;
```

### Lost updates
```
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED ;

START TRANSACTION;
SELECT * FROM users WHERE id = 1;
-- run Session 2
UPDATE users SET name='y' WHERE id=1;
COMMIT;
```

## Percona Session 2

```
SET autocommit=0;
SET GLOBAL innodb_status_output=ON;
SET GLOBAL innodb_status_output_locks=ON;

show engine innodb status;

SELECT @@transaction_ISOLATION;
```

### Dirty read
```
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

START TRANSACTION;
UPDATE users SET age = 21 WHERE id = 1;
-- run Session 1
ROLLBACK;
-- run Session 1
```

### Non-repeatable reads
```
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED ;

-- run Session 1
UPDATE users SET age = 21 WHERE id = 1;
COMMIT;
-- run Session 1
```

### Phantom reads
```
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ ;
-- run Session 1
INSERT INTO users(id, name, age) VALUES (3, 'Bob', 27); # Error after some time(~60s)
COMMIT;
-- run Session 1
```

### Lost updates
```
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED ;

START TRANSACTION;
-- run Session 1
SELECT * FROM users WHERE id = 1;
UPDATE users SET name='x' WHERE id=1; # Error after some time(~60s)
COMMIT;
-- run Session 1
```

## Postgres Session 1
```
SELECT current_setting('transaction_isolation');
```

###  Dirty read
```
START TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT age FROM users WHERE id = 1;
-- run Session 2
SELECT age FROM users WHERE id = 1; -- Here is wrong value
-- run Session 2
SELECT age FROM users WHERE id = 1;
COMMIT ;
```

###  Non-repeatable reads
```
START TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT * FROM users WHERE id = 1;
-- run Session 2
SELECT * FROM users WHERE id = 1;
COMMIT;
```

### Phantom reads
```
START TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT * FROM users
WHERE age BETWEEN 10 AND 30;
-- run Session 2
SELECT * FROM users
WHERE age BETWEEN 10 AND 30;
COMMIT;
```

###  Lost updates
```
START TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT * FROM users WHERE id = 1;
-- run Session 2
UPDATE users SET name='y' WHERE id=1;
COMMIT;
```

### Lost updates 2
```
START TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT * FROM users WHERE id = 1; -- age = 20
-- run Session 2
UPDATE users SET age=20+1 WHERE id=1;
COMMIT;
SELECT * FROM users WHERE id = 1; -- age = 21
```

## Postgres Session 2
```
SELECT current_setting('transaction_isolation');
```
###  Dirty read
```
START TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
UPDATE users SET age = 21 WHERE id = 1;
-- run Session 1
ROLLBACK;
-- run Session 1
```

###  Non-repeatable reads
```
-- run Session 1
START TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
UPDATE users SET age = 21 WHERE id = 1;
COMMIT;
-- run Session 1
```

###  Phantom reads
```
-- run Session 1
START TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
INSERT INTO users(id, name, age) VALUES (3, 'Bob', 27);  -- Error after some time(~60s)
COMMIT;
-- run Session 1
```

###  Lost updates
```
START TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
-- run Session 1
SELECT * FROM users WHERE id = 1;
UPDATE users SET name='x' WHERE id=1; -- Error after some time(~60s)
COMMIT;
-- run Session 1
```

### Lost updates 2
```
START TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
-- run Session 1
SELECT * FROM users WHERE id = 1; -- age = 20
UPDATE users SET age=20+1 WHERE id=1; --
COMMIT;
-- run Session 1
SELECT * FROM users WHERE id = 1; -- age = 21
```