---
title: "MySQL :: MySQL 8.4 Reference Manual :: 17.7.2.1 Transaction Isolation Levels"
source: "https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html"
author:
published:
created: 2026-05-14
description:
tags:
  - "clippings"
---
#### 17.7.2.1 Transaction Isolation Levels

Transaction isolation is one of the foundations of database processing. Isolation is the I in the acronym [ACID](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_acid "ACID"); the isolation level is the setting that fine-tunes the balance between performance and reliability, consistency, and reproducibility of results when multiple transactions are making changes and performing queries at the same time.

`InnoDB` offers all four transaction isolation levels described by the SQL:1992 standard: [`READ UNCOMMITTED`](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html#isolevel_read-uncommitted), [`READ COMMITTED`](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html#isolevel_read-committed), [`REPEATABLE READ`](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read), and [`SERIALIZABLE`](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html#isolevel_serializable). The default isolation level for `InnoDB` is [`REPEATABLE READ`](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read).

A user can change the isolation level for a single session or for all subsequent connections with the [`SET         TRANSACTION`](https://dev.mysql.com/doc/refman/8.4/en/set-transaction.html "15.3.7 SET TRANSACTION Statement") statement. To set the server's default isolation level for all connections, use the [`--transaction-isolation`](https://dev.mysql.com/doc/refman/8.4/en/server-options.html#option_mysqld_transaction-isolation) option on the command line or in an option file. For detailed information about isolation levels and level-setting syntax, see [Section 15.3.7, “SET TRANSACTION Statement”](https://dev.mysql.com/doc/refman/8.4/en/set-transaction.html "15.3.7 SET TRANSACTION Statement").

`InnoDB` supports each of the transaction isolation levels described here using different [locking](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_locking "locking") strategies. You can enforce a high degree of consistency with the default [`REPEATABLE READ`](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read) level, for operations on crucial data where [ACID](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_acid "ACID") compliance is important. Or you can relax the consistency rules with [`READ COMMITTED`](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html#isolevel_read-committed) or even [`READ UNCOMMITTED`](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html#isolevel_read-uncommitted), in situations such as bulk reporting where precise consistency and repeatable results are less important than minimizing the amount of overhead for locking. [`SERIALIZABLE`](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html#isolevel_serializable) enforces even stricter rules than [`REPEATABLE         READ`](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read), and is used mainly in specialized situations, such as with [XA](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_xa "XA") transactions and for troubleshooting issues with concurrency and [deadlocks](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_deadlock "deadlock").

The following list describes how MySQL supports the different transaction levels. The list goes from the most commonly used level to the least used.

- `REPEATABLE READ`
	This is the default isolation level for `InnoDB`. [Consistent reads](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_consistent_read "consistent read") within the same transaction read the [snapshot](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_snapshot "snapshot") established by the first read. This means that if you issue several plain (nonlocking) [`SELECT`](https://dev.mysql.com/doc/refman/8.4/en/select.html "15.2.13 SELECT Statement") statements within the same transaction, these [`SELECT`](https://dev.mysql.com/doc/refman/8.4/en/select.html "15.2.13 SELECT Statement") statements are consistent also with respect to each other. See [Section 17.7.2.3, “Consistent Nonlocking Reads”](https://dev.mysql.com/doc/refman/8.4/en/innodb-consistent-read.html "17.7.2.3 Consistent Nonlocking Reads").
	For [locking reads](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_locking_read "locking read") ([`SELECT`](https://dev.mysql.com/doc/refman/8.4/en/select.html "15.2.13 SELECT Statement") with `FOR             UPDATE` or `FOR SHARE`), [`UPDATE`](https://dev.mysql.com/doc/refman/8.4/en/update.html "15.2.17 UPDATE Statement"), and [`DELETE`](https://dev.mysql.com/doc/refman/8.4/en/delete.html "15.2.2 DELETE Statement") statements, locking depends on whether the statement uses a unique index with a unique search condition, or a range-type search condition.
	- For a unique index with a unique search condition, `InnoDB` locks only the index record found, not the [gap](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_gap "gap") before it.
	- For other search conditions, `InnoDB` locks the index range scanned, using [gap locks](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_gap_lock "gap lock") or [next-key locks](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_next_key_lock "next-key lock") to block insertions by other sessions into the gaps covered by the range. For information about gap locks and next-key locks, see [Section 17.7.1, “InnoDB Locking”](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html "17.7.1 InnoDB Locking").
	It is not recommended to mix locking statements ([`UPDATE`](https://dev.mysql.com/doc/refman/8.4/en/update.html "15.2.17 UPDATE Statement"), [`INSERT`](https://dev.mysql.com/doc/refman/8.4/en/insert.html "15.2.7 INSERT Statement"), [`DELETE`](https://dev.mysql.com/doc/refman/8.4/en/delete.html "15.2.2 DELETE Statement"), or `SELECT             ... FOR ...`) with non-locking [`SELECT`](https://dev.mysql.com/doc/refman/8.4/en/select.html "15.2.13 SELECT Statement") statements in a single [`REPEATABLE READ`](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read) transaction, because typically in such cases you want [`SERIALIZABLE`](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html#isolevel_serializable). This is because a non-locking [`SELECT`](https://dev.mysql.com/doc/refman/8.4/en/select.html "15.2.13 SELECT Statement") statement presents the state of the database from a read view which consists of transactions committed before the read view was created, and before the current transaction's own writes, while the locking statements use the most recent state of the database to use locking. In general, these two different table states are inconsistent with each other and difficult to parse.
- `READ COMMITTED`
	Each consistent read, even within the same transaction, sets and reads its own fresh snapshot. For information about consistent reads, see [Section 17.7.2.3, “Consistent Nonlocking Reads”](https://dev.mysql.com/doc/refman/8.4/en/innodb-consistent-read.html "17.7.2.3 Consistent Nonlocking Reads").
	For locking reads ([`SELECT`](https://dev.mysql.com/doc/refman/8.4/en/select.html "15.2.13 SELECT Statement") with `FOR UPDATE` or `FOR             SHARE`), [`UPDATE`](https://dev.mysql.com/doc/refman/8.4/en/update.html "15.2.17 UPDATE Statement") statements, and [`DELETE`](https://dev.mysql.com/doc/refman/8.4/en/delete.html "15.2.2 DELETE Statement") statements, `InnoDB` locks only index records, not the gaps before them, and thus permits the free insertion of new records next to locked records. Gap locking is only used for foreign-key constraint checking and duplicate-key checking.
	Because gap locking is disabled, phantom row problems may occur, as other sessions can insert new rows into the gaps. For information about phantom rows, see [Section 17.7.4, “Phantom Rows”](https://dev.mysql.com/doc/refman/8.4/en/innodb-next-key-locking.html "17.7.4 Phantom Rows").
	Only row-based binary logging is supported with the `READ COMMITTED` isolation level. If you use `READ COMMITTED` with [`binlog_format=MIXED`](https://dev.mysql.com/doc/refman/8.4/en/replication-options-binary-log.html#sysvar_binlog_format), the server automatically uses row-based logging.
	Using `READ COMMITTED` has additional effects:
	- For [`UPDATE`](https://dev.mysql.com/doc/refman/8.4/en/update.html "15.2.17 UPDATE Statement") or [`DELETE`](https://dev.mysql.com/doc/refman/8.4/en/delete.html "15.2.2 DELETE Statement") statements, `InnoDB` holds locks only for rows that it updates or deletes. Record locks for nonmatching rows are released after MySQL has evaluated the `WHERE` condition. This greatly reduces the probability of deadlocks, but they can still happen.
	- For [`UPDATE`](https://dev.mysql.com/doc/refman/8.4/en/update.html "15.2.17 UPDATE Statement") statements, if a row is already locked, `InnoDB` performs a “semi-consistent” read, returning the latest committed version to MySQL so that MySQL can determine whether the row matches the `WHERE` condition of the [`UPDATE`](https://dev.mysql.com/doc/refman/8.4/en/update.html "15.2.17 UPDATE Statement"). If the row matches (must be updated), MySQL reads the row again and this time `InnoDB` either locks it or waits for a lock on it.
	Consider the table created and populated like this:
	```sql
	CREATE TABLE t (a INT NOT NULL, b INT) ENGINE = InnoDB;
	INSERT INTO t VALUES (1,2),(2,3),(3,2),(4,3),(5,2);
	COMMIT;
	```
	In this case, the table has no indexes, so searches and index scans use the hidden clustered index for record locking (see [Section 17.6.2.1, “Clustered and Secondary Indexes”](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html "17.6.2.1 Clustered and Secondary Indexes")) rather than indexed columns.
	Suppose that one session performs an [`UPDATE`](https://dev.mysql.com/doc/refman/8.4/en/update.html "15.2.17 UPDATE Statement") using these statements:
	```sql
	# Session A
	START TRANSACTION;
	UPDATE t SET b = 5 WHERE b = 3;
	```
	Suppose also that a second session performs an [`UPDATE`](https://dev.mysql.com/doc/refman/8.4/en/update.html "15.2.17 UPDATE Statement") by executing these statements following those of the first session:
	```sql
	# Session B
	UPDATE t SET b = 4 WHERE b = 2;
	```
	As [`InnoDB`](https://dev.mysql.com/doc/refman/8.4/en/innodb-storage-engine.html "Chapter 17 The InnoDB Storage Engine") executes each [`UPDATE`](https://dev.mysql.com/doc/refman/8.4/en/update.html "15.2.17 UPDATE Statement"), it first acquires an exclusive lock for each row, and then determines whether to modify it. If [`InnoDB`](https://dev.mysql.com/doc/refman/8.4/en/innodb-storage-engine.html "Chapter 17 The InnoDB Storage Engine") does not modify the row, it releases the lock. Otherwise, [`InnoDB`](https://dev.mysql.com/doc/refman/8.4/en/innodb-storage-engine.html "Chapter 17 The InnoDB Storage Engine") retains the lock until the end of the transaction. This affects transaction processing as follows.
	When using the default `REPEATABLE READ` isolation level, the first [`UPDATE`](https://dev.mysql.com/doc/refman/8.4/en/update.html "15.2.17 UPDATE Statement") acquires an x-lock on each row that it reads and does not release any of them:
	```svg
	x-lock(1,2); retain x-lock
	x-lock(2,3); update(2,3) to (2,5); retain x-lock
	x-lock(3,2); retain x-lock
	x-lock(4,3); update(4,3) to (4,5); retain x-lock
	x-lock(5,2); retain x-lock
	```
	The second [`UPDATE`](https://dev.mysql.com/doc/refman/8.4/en/update.html "15.2.17 UPDATE Statement") blocks as soon as it tries to acquire any locks (because first update has retained locks on all rows), and does not proceed until the first [`UPDATE`](https://dev.mysql.com/doc/refman/8.4/en/update.html "15.2.17 UPDATE Statement") commits or rolls back:
	```svg
	Press ⌘+C to copy x-lock(1,2); block and wait for first UPDATE to commit or roll back
	```
	If `READ COMMITTED` is used instead, the first [`UPDATE`](https://dev.mysql.com/doc/refman/8.4/en/update.html "15.2.17 UPDATE Statement") acquires an x-lock on each row that it reads and releases those for rows that it does not modify:
	```svg
	Press ⌘+C to copy x-lock(1,2); unlock(1,2)
	x-lock(2,3); update(2,3) to (2,5); retain x-lock
	x-lock(3,2); unlock(3,2)
	x-lock(4,3); update(4,3) to (4,5); retain x-lock
	x-lock(5,2); unlock(5,2)
	```
	For the second `UPDATE`, `InnoDB` does a “semi-consistent” read, returning the latest committed version of each row that it reads to MySQL so that MySQL can determine whether the row matches the `WHERE` condition of the [`UPDATE`](https://dev.mysql.com/doc/refman/8.4/en/update.html "15.2.17 UPDATE Statement"):
	```svg
	Press ⌘+C to copy x-lock(1,2); update(1,2) to (1,4); retain x-lock
	x-lock(2,3); unlock(2,3)
	x-lock(3,2); update(3,2) to (3,4); retain x-lock
	x-lock(4,3); unlock(4,3)
	x-lock(5,2); update(5,2) to (5,4); retain x-lock
	```
	However, if the `WHERE` condition includes an indexed column, and `InnoDB` uses the index, only the indexed column is considered when taking and retaining record locks. In the following example, the first [`UPDATE`](https://dev.mysql.com/doc/refman/8.4/en/update.html "15.2.17 UPDATE Statement") takes and retains an x-lock on each row where b = 2. The second [`UPDATE`](https://dev.mysql.com/doc/refman/8.4/en/update.html "15.2.17 UPDATE Statement") blocks when it tries to acquire x-locks on the same records, as it also uses the index defined on column b.
	```sql
	Press ⌘+C to copy CREATE TABLE t (a INT NOT NULL, b INT, c INT, INDEX (b)) ENGINE = InnoDB;
	INSERT INTO t VALUES (1,2,3),(2,2,4);
	COMMIT;
	# Session A
	START TRANSACTION;
	UPDATE t SET b = 3 WHERE b = 2 AND c = 3;
	# Session B
	UPDATE t SET b = 4 WHERE b = 2 AND c = 4;
	```
	The `READ COMMITTED` isolation level can be set at startup or changed at runtime. At runtime, it can be set globally for all sessions, or individually per session.
- `READ UNCOMMITTED`
	[`SELECT`](https://dev.mysql.com/doc/refman/8.4/en/select.html "15.2.13 SELECT Statement") statements are performed in a nonlocking fashion, but a possible earlier version of a row might be used. Thus, using this isolation level, such reads are not consistent. This is also called a [dirty read](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_dirty_read "dirty read"). Otherwise, this isolation level works like [`READ COMMITTED`](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html#isolevel_read-committed).
- `SERIALIZABLE`
	This level is like [`REPEATABLE             READ`](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read), but `InnoDB` implicitly converts all plain [`SELECT`](https://dev.mysql.com/doc/refman/8.4/en/select.html "15.2.13 SELECT Statement") statements to [`SELECT             ... FOR SHARE`](https://dev.mysql.com/doc/refman/8.4/en/select.html "15.2.13 SELECT Statement") if [`autocommit`](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html#sysvar_autocommit) is disabled. If [`autocommit`](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html#sysvar_autocommit) is enabled, the [`SELECT`](https://dev.mysql.com/doc/refman/8.4/en/select.html "15.2.13 SELECT Statement") is its own transaction. It therefore is known to be read only and can be serialized if performed as a consistent (nonlocking) read and need not block for other transactions. (To force a plain [`SELECT`](https://dev.mysql.com/doc/refman/8.4/en/select.html "15.2.13 SELECT Statement") to block if other transactions have modified the selected rows, disable [`autocommit`](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html#sysvar_autocommit).)
	DML operations that read data from MySQL grant tables (through a join list or subquery) but do not modify them do not acquire read locks on the MySQL grant tables, regardless of the isolation level. For more information, see [Grant Table Concurrency](https://dev.mysql.com/doc/refman/8.4/en/grant-tables.html#grant-tables-concurrency "Grant Table Concurrency").