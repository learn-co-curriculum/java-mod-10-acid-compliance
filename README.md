# ACID Compliance

## Learning Goals

- Understanding guarantees of ACID compliance
- Understanding potential limitations and edge cases

## Instructions

In this lab, we will be exploring hands on the data guarantees that ACID compliant database systems provide. We will be utilizing Postgres again in this situation, which can be considered a fairly hard ACID compliant system.
Once we have a grasp of how ACID compliance interacts with data requests in the general case, we will look at a few edge cases you will need to be aware of in any future use of these systems.

## Guarantees of ACID compliance

ACID stands for the following:

- Atomicity
- Consistency
- Isolation
- Durability

Let's start demoing these from the Durability standpoint

### Durability

Postgres approaches the durability guarantee by using what is called the Write Ahead Log(WAL). For every change that is made to the database system, the WAL will be updated with serializable
changes in a sequential order, before the database files are updated in place on disk. Due to this archetecture choice, there can be a continuous stream of data being written to the slow
storage, which is recoverable in the case of a system crash or power failure.

Let's see what these look like:

``` text
docker exec -it postgres-lab ls /var/lib/postgresql/data/pg_wal    
000000010000000000000004  000000010000000000000005  archive_status

docker exec -it postgres-lab pg_waldump -f 000000010000000000000004
...
rmgr: XLOG        len (rec/tot):    114/   114, tx:          0, lsn: 0/0409F040, prev 0/0409F008, desc: CHECKPOINT_ONLINE redo 0/409F008; tli 1; prev tli 1; fpw true; xid 0:122669; oid 33216; multi 1; offset 0; oldest xid 727 in DB 1; oldest multi 1 in DB 1; oldest/newest commit timestamp xid: 0/0; oldest running xid 122669; online
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/0409F0B8, prev 0/0409F040, desc: RUNNING_XACTS nextXid 122669 latestCompletedXid 122668 oldestRunningXid 122669
```

If we rerun the previous lab's benchmark at this point, we should be able to see this log output streaming in real-time.


Let us try interrupting the database operations now to see how that is handled. We'll run a longer benchmark, and while that is running, forcefully terminate the Postgres container

TODO - Previous lab needs to be completely implemented before students run the below correctly

``` text
curl "http://localhost:8080/benchmark?count=10000"

docker kill postgres-lab # In a different terminal

{"timestamp":"2022-07-13T20:12:19.398+00:00","status":500,"error":"Internal Server Error","path":"/benchmark"}
```

If you start up the container again and view the Postgres logs, you'll be able to see that the system has automatically recovered from where it had crashed.

``` text
docker start postgres-lab
docker logs postgres-lab
...
2022-07-13 20:26:08.787 UTC [1] LOG:  starting PostgreSQL 14.4 (Ubuntu 14.4-0ubuntu0.22.04.1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.2.0-19ubuntu1) 11.2.0, 64-bit
2022-07-13 20:26:08.788 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2022-07-13 20:26:08.788 UTC [1] LOG:  listening on IPv6 address "::", port 5432
2022-07-13 20:26:08.801 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2022-07-13 20:26:08.816 UTC [26] LOG:  database system was interrupted; last known up at 2022-07-13 20:10:25 UTC
2022-07-13 20:26:08.916 UTC [26] LOG:  database system was not properly shut down; automatic recovery in progress
2022-07-13 20:26:08.945 UTC [26] LOG:  redo starts at 0/41C3270
2022-07-13 20:26:08.966 UTC [26] LOG:  invalid record length at 0/42C7730: wanted 24, got 0
2022-07-13 20:26:08.966 UTC [26] LOG:  redo done at 0/42C7708 system usage: CPU: user: 0.01 s, system: 0.00 s, elapsed: 0.02 s
2022-07-13 20:26:09.046 UTC [1] LOG:  database system is ready to accept connections
```

And let us see just how much of the application client data was lost due to this "crash"

``` text
# trace server logs from IntelliJ at the point of crash
...
Hibernate: insert into haystack (uuid, value) values (?, ?)
2022-07-13 15:30:02.743 TRACE 39131 --- [nio-8080-exec-1] o.h.type.descriptor.sql.BasicBinder      : binding parameter [1] as [OTHER] - [b4bfcc8b-223c-4013-90b7-2c0cc5605e40]
2022-07-13 15:30:02.743 TRACE 39131 --- [nio-8080-exec-1] o.h.type.descriptor.sql.BasicBinder      : binding parameter [2] as [VARCHAR] - [hay]
Hibernate: insert into haystackuuid (uuid) values (?)
2022-07-13 15:30:02.751 TRACE 39131 --- [nio-8080-exec-1] o.h.type.descriptor.sql.BasicBinder      : binding parameter [1] as [OTHER] - [b4bfcc8b-223c-4013-90b7-2c0cc5605e40]
2022-07-13 15:30:02.760  WARN 39131 --- [nio-8080-exec-1] com.zaxxer.hikari.pool.ProxyConnection   : HikariPool-1 - Connection org.postgresql.jdbc.PgConnection@6d88415f marked as broken because of SQLSTATE(57P01), ErrorCode(0)

org.postgresql.util.PSQLException: FATAL: terminating connection due to unexpected postmaster exit
	at org.postgresql.core.v3.QueryExecutorImpl.receiveErrorResponse(QueryExecutorImpl.java:2674) ~[postgresql-42.3.1.jar:42.3.1]
	at org.postgresql.core.v3.QueryExecutorImpl.processResults(QueryExecutorImpl.java:2364) ~[postgresql-42.3.1.jar:42.3.1]
...
```

``` text
postgres=# \c db_test

db_test=# select * from haystack order by id desc limit 10;
  id   |                 uuid                 | value 
-------+--------------------------------------+-------
 54815 | b4bfcc8b-223c-4013-90b7-2c0cc5605e40 | hay
 54814 | a492fa41-ba46-4874-9132-dd0ce0c24390 | hay
 54813 | b7a9e7e8-9cbc-4306-897c-aa9f31627609 | hay
 54812 | d559da88-42de-46da-867f-4596ef1b6e93 | hay
 54811 | 2968f00c-d376-4972-9083-2942c90a35d8 | hay
 54810 | aad4f40a-3821-4608-8612-6cfdb18c032f | hay
 54809 | ba459a73-0ece-4948-9694-ff2fae7acdbe | hay
 54808 | 04d584dd-1ee9-4d52-8f53-9a771b393310 | hay
 54807 | 528fce92-a6b2-48c9-bf21-043419fc9fea | hay
 54806 | 91636e30-d771-4089-a07a-b6e7e833aefa | hay
(10 rows)

db_test=# select * from haystackuuid order by id desc limit 10;
  id   |                 uuid                 
-------+--------------------------------------
 54814 | a492fa41-ba46-4874-9132-dd0ce0c24390
 54813 | b7a9e7e8-9cbc-4306-897c-aa9f31627609
 54812 | d559da88-42de-46da-867f-4596ef1b6e93
 54811 | 2968f00c-d376-4972-9083-2942c90a35d8
 54810 | aad4f40a-3821-4608-8612-6cfdb18c032f
 54809 | ba459a73-0ece-4948-9694-ff2fae7acdbe
 54808 | 04d584dd-1ee9-4d52-8f53-9a771b393310
 54807 | 528fce92-a6b2-48c9-bf21-043419fc9fea
 54806 | 91636e30-d771-4089-a07a-b6e7e833aefa
 54805 | 3ac8bcc1-84da-4332-af42-448eaed892e2
(10 rows)
```

It looks like in this case, the database was able to recover all data, up until the point where the client lost connectivity from the "crash". It's worth noting though that this demonstration
may not be representative of real world hardware or software failure. The intent of the Postgres Write Ahead Log is to provide the Durability guarantee from the application perspective,
but the Postgres application itself is only one component in a full Database implementation.


### Isolation


Postgres implements Isolation by using different levels of transaction isolation, and data locking. 


Let us take a look at a simple example


``` text
postgres=# \c db_test
db_test=# CREATE TABLE test(id SERIAL PRIMARY KEY, x INTEGER);
CREATE TABLE
db_test=# INSERT INTO test (id, x) VALUES (1, 0);
INSERT 0 1
db_test=# SELECT * FROM test;
 id | x 
----+---
  1 | 0
(1 row)

db_test=# UPDATE test SET x = x + 1 WHERE id = 1;
UPDATE 1
db_test=# SELECT * FROM test;
 id | x 
----+---
  1 | 1
(1 row)
```

What exactly is happening here? While this may look like a single atomic operation, it is actually a multi-step process, with Postgres
automatically wrapping it in a Transaction. This actually is the default behavior for all SQL operations. Let's slow things down and
see what is actually going on:


``` text
db_test=# UPDATE test SET x = x + 1 WHERE id = 1 RETURNING pg_sleep(30);

# In another terminal

db_test=# select locktype, database, relation, virtualxid, transactionid, virtualtransaction, pid, mode, granted, fastpath from pg_locks;
   locktype    | database | relation | virtualxid | transactionid | virtualtransaction | pid |       mode       | granted | fastpath 
---------------+----------+----------+------------+---------------+--------------------+-----+------------------+---------+----------
 relation      |    16384 |    41413 |            |               | 15/4               | 173 | RowExclusiveLock | t       | t
 relation      |    16384 |    41409 |            |               | 15/4               | 173 | RowExclusiveLock | t       | t
 virtualxid    |          |          | 15/4       |               | 15/4               | 173 | ExclusiveLock    | t       | t
 relation      |    16384 |    12290 |            |               | 5/32               |  46 | AccessShareLock  | t       | t
 virtualxid    |          |          | 5/32       |               | 5/32               |  46 | ExclusiveLock    | t       | t
 transactionid |          |          |            |        134639 | 15/4               | 173 | ExclusiveLock    | t       | f
(6 rows)
```

We can see that our slowed process (pid 173 here) actually has several ExclusiveLocks and RowExclusiveLocks in this table it is accessing.

``` text
db_test=# UPDATE test SET x = x + 1 WHERE id = 1 RETURNING pg_sleep(30);

# In a second terminal

db_test=# UPDATE test SET x = x + 1 WHERE id = 1;

# In a third terminal

db_test=#  select datid, datname, pid, application_name, wait_event_type, wait_event, state, query from pg_stat_activity;
 datid | datname | pid |    application_name    | wait_event_type |     wait_event      | state  |                                                          query                                                           
-------+---------+-----+------------------------+-----------------+---------------------+--------+--------------------------------------------------------------------------------------------------------------------------
       |         |  31 |                        | Activity        | AutoVacuumMain      |        | 
       |         |  33 |                        | Activity        | LogicalLauncherMain |        | 
 16384 | db_test | 200 | PostgreSQL JDBC Driver | Client          | ClientRead          | idle   | COMMIT
 16384 | db_test | 184 | PostgreSQL JDBC Driver | Client          | ClientRead          | idle   | COMMIT
 16384 | db_test |  46 | psql                   | Lock            | transactionid       | active | UPDATE test SET x = x + 1 WHERE id = 1;
 16384 | db_test | 189 | PostgreSQL JDBC Driver | Client          | ClientRead          | idle   | COMMIT
 16384 | db_test | 183 | PostgreSQL JDBC Driver | Client          | ClientRead          | idle   | COMMIT
 16384 | db_test | 182 | PostgreSQL JDBC Driver | Client          | ClientRead          | idle   | COMMIT
 16384 | db_test | 187 | PostgreSQL JDBC Driver | Client          | ClientRead          | idle   | COMMIT
 16384 | db_test | 191 | PostgreSQL JDBC Driver | Client          | ClientRead          | idle   | COMMIT
 16384 | db_test | 185 | PostgreSQL JDBC Driver | Client          | ClientRead          | idle   | COMMIT
 16384 | db_test | 188 | PostgreSQL JDBC Driver | Client          | ClientRead          | idle   | COMMIT
 16384 | db_test | 190 | PostgreSQL JDBC Driver | Client          | ClientRead          | idle   | COMMIT
 16384 | db_test | 173 | psql                   | Timeout         | PgSleep             | active | UPDATE test SET x = x + 1 WHERE id = 1 RETURNING pg_sleep(30);
 16384 | db_test | 221 | psql                   |                 |                     | active | select datid, datname, pid, application_name, wait_event_type, wait_event, state, query_id, query from pg_stat_activity;
       |         |  29 |                        | Activity        | BgWriterHibernate   |        | 
       |         |  28 |                        | Activity        | CheckpointerMain    |        | 
       |         |  30 |                        | Activity        | WalWriterMain       |        | 
(18 rows)

```

You can see here that the first query (pid 173) waits for 30 seconds before completing, the second query (pid 46) waits for completion of the first, and the third captures this state of the second query actively waiting on a Transaction
Lock.

All this is just to show that even simple transactions get executed in isolation. When you start thinking about production systems running thousands of transactions a second, you can see where bottlenecks may start to form, and how
data model design quickly becomes a limiting factor.


### Consistency


Consistency of data is supported by all the other guarantees, as well as data local constraints and typing. This is integral to the database in some instances, but also requires a supporting data model
with proper constraints.

Let's look at some examples



``` text
db_test=# INSERT INTO test (id, x) VALUES (2, 2147483648);
ERROR:  integer out of range
db_test=# INSERT INTO test (id, x) VALUES (2, 2147483647);
INSERT 0 1
db_test=# INSERT INTO test (id, x) VALUES (3, 'string');
ERROR:  invalid input syntax for type integer: "string"
LINE 1: INSERT INTO test (id, x) VALUES (3, 'string');
```

These show native constraints being applied by the data types selected.
Let's look at additional constraints that can be used.

``` text
db_test=# CREATE TABLE constraint_test(id SERIAL PRIMARY KEY, x INTEGER CHECK ( x > 0 ));
CREATE TABLE
db_test=# INSERT INTO constraint_test (id, x) VALUES (1, -1);
ERROR:  new row for relation "constraint_test" violates check constraint "constraint_test_x_check"
DETAIL:  Failing row contains (1, -1).
db_test=# INSERT INTO constraint_test (id, x) VALUES (1, 1);
INSERT 0 1
db_test=# UPDATE constraint_test SET x = x - 2 WHERE id = 1;
ERROR:  new row for relation "constraint_test" violates check constraint "constraint_test_x_check"
DETAIL:  Failing row contains (1, -1).
db_test=# SELECT * FROM constraint_test;
 id | x 
----+---
  1 | 1
(1 row)

```

We can see that illegal operations will be halted before placing a database into an inconsistent state.


### Atomicity

Atomicity in Postgres is guaranteed by all SQL operations being handled as transactions internally. And for multi-query operations, the capability to bundle all into a multi-statement transaction.
If any operation bundled inside a transaction fails, all operations completed up until that point will be reverted.
Let's take a look.
       
``` text
db_test=# SELECT * FROM test;
 id |     x      
----+------------
  1 |          4
  2 | 2147483647
(2 rows)

db_test=# BEGIN;
BEGIN
db_test=*# UPDATE test SET x = x + 1 WHERE id = 1;
UPDATE 1
db_test=*# UPDATE test SET x = x - 1 WHERE id = 2;
UPDATE 1
db_test=*# COMMIT;
COMMIT
db_test=# SELECT * FROM test;
 id |     x      
----+------------
  1 |          5
  2 | 2147483646
(2 rows)
```

You can imagine this behavior would be mission critical in various different sectors. Imagine for a second poorly timed failures or bugs occurring between a debit from one account, and the associated credit on the other side.

Let's see what happens when there is a failure of some kind during a transaction.

``` text
db_test=# SELECT * FROM test;
 id |     x      
----+------------
  1 |          5
  2 | 2147483646
(2 rows)

db_test=# BEGIN; UPDATE test SET x = x - 2 WHERE id = 1; SELECT * FROM test; UPDATE test SET x = x + 2 WHERE id = 2; COMMIT;
BEGIN
UPDATE 1
 id |     x      
----+------------
  2 | 2147483646
  1 |          3
(2 rows)

ERROR:  integer out of range
ROLLBACK
db_test=# SELECT * FROM test;
 id |     x      
----+------------
  1 |          5
  2 | 2147483646
(2 rows)
```

You can see that the transaction started moving values from one item to another, but errored and rolled back when one of the values overflowed.


## Limitations and Edge Cases

While ACID Compliance does mean that your data is guaranteed to be safer from loss, and act in more consistent ways, this doesn't mean that you can throw data into the ACID compliant system without concern
to data models and access patterns. Now that we've had an overview of how Postgres manages this, take some time to explore and try to implement some edge cases which can occur when using these systems. Feel free to research
anything you've heard of, or are interested in finding out some more about. Here are a few keywords to get started with.

- Deadlocks
- Phantom Reads
- Divide by Zero
