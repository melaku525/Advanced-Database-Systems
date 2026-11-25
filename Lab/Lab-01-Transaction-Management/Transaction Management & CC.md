# Lab 01: Transaction Management and Concurrency Control

## 1. Introduction

A database transaction is a sequence of one or more SQL operations executed as a single logical unit of work. Transactions ensure that database operations are performed reliably and consistently, even when errors occur or multiple users access the database simultaneously.

In real-world applications such as banking, university registration, hospital management, and online shopping, multiple database operations often need to succeed together. For example, transferring money between two bank accounts requires deducting money from one account and adding the same amount to another account.

If one operation succeeds but the other fails, the database may become inconsistent. Transaction management prevents such problems by providing mechanisms to commit or roll back database operations.

In this laboratory, students will use PostgreSQL to explore transaction management, ACID properties, COMMIT, ROLLBACK, transaction isolation levels, and concurrency control.

## 2. Objectives

At the end of this laboratory, students should be able to:

1. Explain the concept of database transactions.
2. Describe and demonstrate the ACID properties.
3. Start and manage transactions in PostgreSQL.
4. Apply COMMIT and ROLLBACK commands.
5. Understand transaction isolation levels.
6. Demonstrate concurrent database access using multiple sessions.
7. Apply row-level locking to control concurrent transactions.

## 3. Required Software

| Software | Purpose |
|---|---|
| PostgreSQL | Database management system |
| pgAdmin 4 | Graphical database interface |

**Note:** PostgreSQL must be installed and running before starting the exercises.

## 4. Background Theory

### 4.1 What Is a Transaction?

A transaction is a logical unit of database operations that must be completed successfully as a whole.

Consider transferring 100 Birr from Account A to Account B:

1. Deduct 100 Birr from Account A.
2. Add 100 Birr to Account B.

Both operations must succeed. If either operation fails, the entire transaction should be cancelled.

### 4.2 ACID Properties

Database transactions follow four important properties known as ACID.

| Property | Description | Example |
|---|---|---|
| Atomicity | All operations succeed or none are applied. | Both account updates must succeed. |
| Consistency | A transaction preserves database rules and constraints. | Account balances cannot become negative. |
| Isolation | Concurrent transactions should not interfere improperly. | Two users updating the same account. |
| Durability | Committed changes remain stored. | A successful transfer remains after a restart. |

### 4.3 Transaction Commands

| Command | Description |
|---|---|
| `BEGIN` | Starts a transaction. |
| `COMMIT` | Permanently saves transaction changes. |
| `ROLLBACK` | Cancels uncommitted changes. |
| `SAVEPOINT` | Creates a point within a transaction to which you can roll back. |
| `ROLLBACK TO SAVEPOINT` | Reverses changes made after a savepoint. |

### 4.4 Transaction Isolation Levels

PostgreSQL supports the following standard isolation levels:

| Isolation Level | Description |
|---|---|
| READ UNCOMMITTED | In PostgreSQL, behaves like READ COMMITTED. |
| READ COMMITTED | Each statement sees data committed before that statement begins. |
| REPEATABLE READ | A transaction sees a consistent snapshot of data. |
| SERIALIZABLE | Concurrent transactions must produce results equivalent to sequential execution. Provides the strongest isolation and may abort transactions that cannot be safely serialized. |

## 5. Practical Activities

### Activity 1: Create a Database

**Step 1:** Open pgAdmin 4 and connect to your PostgreSQL server.

**Step 2:** Open the Query Tool.

**Step 3:** Execute the following SQL statement:

```sql
CREATE DATABASE transaction_lab;
```

**Step 4:** Connect to the newly created `transaction_lab` database and open its Query Tool.

### Activity 2: Create a Bank Account Table

Create a table named `accounts` to store customer account information.

```sql
CREATE TABLE accounts (
    account_id SERIAL PRIMARY KEY,
    account_name VARCHAR(100) NOT NULL,
    balance NUMERIC(12, 2) NOT NULL
        CHECK (balance >= 0)
);
```

Insert two sample accounts:

```sql
INSERT INTO accounts (account_name, balance)
VALUES
    ('Account A', 1000.00),
    ('Account B', 500.00);
```

Retrieve the account records:

```sql
SELECT * FROM accounts;
```

**Expected Output**

| account_id | account_name | balance |
|---:|---|---:|
| 1 | Account A | 1000.00 |
| 2 | Account B | 500.00 |

### Activity 3: Perform a Transaction Using COMMIT

Transfer 100 Birr from Account A to Account B.

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE account_id = 1;

UPDATE accounts
SET balance = balance + 100
WHERE account_id = 2;

COMMIT;
```

Verify the results:

```sql
SELECT * FROM accounts;
```

**Expected Output**

| account_id | account_name | balance |
|---:|---|---:|
| 1 | Account A | 900.00 |
| 2 | Account B | 600.00 |

**Observation:** The transaction successfully transfers 100 Birr. The total balance remains 1,500 Birr.

```sql
SELECT SUM(balance) AS total_balance
FROM accounts;
```

Expected total balance: `1500.00`.

### Activity 4: Perform a Transaction Using ROLLBACK

In this activity, students will observe how PostgreSQL cancels uncommitted changes.

Execute:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 200
WHERE account_id = 1;

UPDATE accounts
SET balance = balance + 200
WHERE account_id = 2;

ROLLBACK;
```

Retrieve the account records:

```sql
SELECT * FROM accounts;
```

**Expected Output**

| account_id | account_name | balance |
|---:|---|---:|
| 1 | Account A | 900.00 |
| 2 | Account B | 600.00 |

**Observation:** The balances remain unchanged because the transaction was rolled back instead of committed.

### Activity 5: Use SAVEPOINT

A savepoint allows a transaction to partially reverse its operations without cancelling the entire transaction.

Execute:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 50
WHERE account_id = 1;

SAVEPOINT transfer_point;

UPDATE accounts
SET balance = balance + 50
WHERE account_id = 2;

ROLLBACK TO SAVEPOINT transfer_point;

COMMIT;
```

Verify the results:

```sql
SELECT * FROM accounts;
```

**Expected Output**

| account_id | account_name | balance |
|---:|---|---:|
| 1 | Account A | 850.00 |
| 2 | Account B | 600.00 |

**Explanation:** The first update is committed, while the second update is reversed by rolling back to the savepoint.

### Activity 6: Demonstrate Transaction Isolation

Transaction isolation levels determine how multiple users or transactions access and modify the same database data simultaneously without interfering with one another.
PostgreSQL supports different transaction isolation levels. In this activity, students will examine how isolation levels affect concurrent database operations.

First, reset the account balances:

```sql
UPDATE accounts
SET balance = 1000.00
WHERE account_id = 1;

UPDATE accounts
SET balance = 500.00
WHERE account_id = 2;
```
#### Check current transaction Isolation Level


```sql
show transaction_isolation;
```

#### Experiment A: READ COMMITTED

Open two separate Query Tool sessions connected to `transaction_lab`.

**Session 1**

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;

SELECT balance
FROM accounts
WHERE account_id = 1;
```

Expected balance: `1000.00`.

**Session 2**

```sql
BEGIN;
UPDATE accounts
SET balance = 1200.00
WHERE account_id = 1;

COMMIT;
```

**Session 1**

Execute the same SELECT statement again:

```sql
SELECT balance
FROM accounts
WHERE account_id = 1;
```

Expected balance: `1200.00`.

Finish Session 1:

```sql
COMMIT;
```

**Observation:** Under READ COMMITTED, each SELECT statement can see changes committed by other transactions before that statement begins.

#### Experiment B: REPEATABLE READ

Reset the account balance:

```sql
UPDATE accounts
SET balance = 1000.00
WHERE account_id = 1;
```

**Session 1**

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;

SELECT balance
FROM accounts
WHERE account_id = 1;
```

Expected balance: `1000.00`.

**Session 2**

```sql
BEGIN;
UPDATE accounts
SET balance = 1200.00
WHERE account_id = 1;

COMMIT;
```

**Session 1**

Execute:

```sql
SELECT balance
FROM accounts
WHERE account_id = 1;
```

Expected balance: `1000.00`.

Finish Session 1:

```sql
COMMIT;
```

**Observation:** REPEATABLE READ provides a consistent snapshot for the transaction. The second SELECT does not see the committed update from Session 2.

### Activity 7: Demonstrate Row-Level Locking

1. Row-level locking: PostgreSQL provides row-level locking to control concurrent access to records.

**Session 1**

```sql
BEGIN;

SELECT *
FROM accounts
WHERE account_id = 1
FOR UPDATE;
```

The selected row is locked by Session 1.

**Session 2**

Execute:

```sql
UPDATE accounts
SET balance = balance + 100
WHERE account_id = 1;
```

**Observation:** Session 2 waits because Session 1 holds a conflicting row-level lock.

**Session 1**

```sql
COMMIT;
```

After Session 1 commits, Session 2 can continue.

Finish Session 2 by committing:

```sql
COMMIT;
```
Purpose: Prevents other transactions from modifying the selected row until the transaction finishes.
**Discussion:** Explain why row-level locking is important in banking and other multi-user applications.

2. Table-Level Locking: Locks an entire table to control access by other transactions.

**Session 1**

```sql
BEGIN;

LOCK TABLE accounts
IN EXCLUSIVE MODE;

UPDATE accounts
SET balance = balance + 500
WHERE account_id = 1;
```

The entire accounts table is locked by Session 1.

**Session 2**

Execute:

```sql
BEGIN;

UPDATE accounts
SET balance = balance + 100
WHERE account_id = 2;
```

**Observation:** Session 2 waits because Session 1 holds a conflicting table-level lock.
Notice that Session 1 updates account ID 1, while Session 2 attempts to update account ID 2. Even though they are different records, Session 2 must wait because the lock applies to the entire accounts table.

**Session 1**

```sql
COMMIT;
```

After Session 1 commits, Session 2 can continue.

Finish Session 2 by committing:

```sql
COMMIT;



