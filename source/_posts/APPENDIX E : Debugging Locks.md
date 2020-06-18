

```
APPENDIX E
```
```
Debugging Locks
```
Any system that uses locks to control shared access to resources can be hard to debug
when a lock contention issue crops up. Perhaps you’re trying to add a column to a table,
or just trying to run a query, when suddenly you find that your queries are blocked
because something else is locking the table or rows you’re trying to use. Often all you
will want to do is find out why your query is blocked, but sometimes you will want to
know what’s blocking it, so you know which process to kill. This appendix shows you
how to achieve both goals.

The MySQL server itself uses several types of locks. If a query is waiting for a lock at
the server level, you can see evidence of it in the output of SHOW PROCESSLIST. In addition
to server-level locks, any storage engine that supports row-level locks, such as InnoDB,
implements its own locks. In MySQL 5.0 and earlier versions, the server is unaware of
such locks, and they’re mostly hidden from users and database administrators. There’s
more visibility in MySQL 5.1 and later versions.

### Lock Waits at the Server Level

A lock wait can happen at either the server level or the storage engine level.^1
(Application-level locks could be a problem too, but we’re focusing on MySQL.) Here
are the kinds of locks the MySQL server uses:

_Table locks_
Tables can be locked with explicit read and write locks. There are a couple of
variations on these locks, such as local read locks. You can learn about the varia-
tions in the LOCK TABLES section of the MySQL manual. In addition to these explicit
locks, queries acquire implicit locks on tables for their durations.

1. Refer to Figure 1-1 in Chapter 1 if you need to refresh your memory on the separation between the server
    and the storage engines.

```
735
```

_Global locks_
There is a single global read lock that can be acquired with FLUSH TABLES WITH READ
LOCK or by setting read_only=1. This conflicts with any table locks.

_Name locks_
Name locks are a type of table lock that the server creates when it renames or
drops a table.

_String locks_
You can lock and release an arbitrary string server-wide with GET_LOCK() and its
associated functions.

We examine each of these lock types in more detail in the following sections.

#### Table Locks

Table locks can be either explicit or implicit. You create explicit locks with LOCK
TABLES. For example, if you execute the following command in a _mysql_ session, you’ll
have an explicit lock on sakila.film:

```
mysql> LOCK TABLES sakila.film READ;
```
If you then execute the following command in a different session, the query will hang
and not complete:

```
mysql> LOCK TABLES sakila.film WRITE;
```
You can see the waiting thread in the first connection:

```
mysql> SHOW PROCESSLIST\G
*************************** 1. row ***************************
Id: 7
User: baron
Host: localhost
db: NULL
Command: Query
Time: 0
State: NULL
Info: SHOW PROCESSLIST
*************************** 2. row ***************************
Id: 11
User: baron
Host: localhost
db: NULL
Command: Query
Time: 4
State: Locked
Info: LOCK TABLES sakila.film WRITE
2 rows in set (0.01 sec)
```
Notice that thread 11’s state is Locked. There is only one place in the MySQL server’s
code where a thread enters that state: when it tries to acquire a table lock and another

**736 | Appendix E: Debugging Locks**


thread has the table locked. Thus, if you see this, you know the thread is waiting for a
lock in the MySQL server, not in the storage engine.

Explicit locks, however, are not the only type of lock that might block such an opera-
tion. As we mentioned earlier, the server implicitly locks tables during queries. An easy
way to show this is with a long-running query, which you can create easily with the
SLEEP() function:

```
mysql> SELECT SLEEP(30) FROM sakila.film LIMIT 1;
```
If you try again to lock sakila.film while that query is running, the operation will hang
because of the implicit lock, just as it did when you had the explicit lock. You’ll be able
to see the effects in the process list, as before:

```
mysql> SHOW PROCESSLIST\G
*************************** 1. row ***************************
Id: 7
User: baron
Host: localhost
db: NULL
Command: Query
Time: 12
State: Sending data
Info: SELECT SLEEP(30) FROM sakila.film LIMIT 1
*************************** 2. row ***************************
Id: 11
User: baron
Host: localhost
db: NULL
Command: Query
Time: 9
State: Locked
Info: LOCK TABLES sakila.film WRITE
```
In this example, the implicit read lock for the SELECT query blocks the explicit write
lock requested by LOCK TABLES. Implicit locks can block each other, too.

You might be wondering about the difference between implicit and explicit locks. In-
ternally, they are the same type of structure, and the same MySQL server code controls
them. Externally, you can control explicit locks yourself with LOCK TABLES and UNLOCK
TABLES.

When it comes to storage engines other than MyISAM, however, there’s one very im-
portant difference between them. When you create a lock explicitly, it does what you
tell it to, but implicit locks are hidden and “magical.” The server creates and releases
implicit locks automatically as needed, and it tells the storage engine about them. Stor-
age engines “convert” these locks as they see fit. For example, InnoDB has rules about
what type of InnoDB table lock it should create for a given server-level table lock. This
can make it hard to understand what locks InnoDB is really creating behind the scenes.

```
Lock Waits at the Server Level | 737
```

**Finding out who holds a lock**

If you see a lot of processes in the Locked state, your problem might be that you’re trying
to use MyISAM or a similar storage engine for a high-concurrency workload. This can
block you from performing an operation manually, such as adding an index to a table.
If an UPDATE query is queued and waiting for a lock on a MyISAM table, even a SELECT
query won’t be allowed to run. (You can read more about MySQL’s lock queuing and
priorities in the MySQL manual.)

In some cases, it can become clear that some connection has been holding a lock on a
table for a very long time and just needs to be killed (or a user needs to be admonished
not to hold up the works!). But how can you find out which connection that is?

There’s currently no SQL command that can show you which thread holds the table
locks that are blocking your query. If you run SHOW PROCESSLIST, you can see the pro-
cesses that are waiting for locks, but not which processes hold those locks. Fortunately,
there’s a _debug_ command that can print information about locks into the server’s error
log. You can use the _mysqladmin_ utility to run the command:

```
$ mysqladmin debug
```
The output in the error log includes a lot of debugging information, but near the end
you’ll see something like the following. We created this output by locking the table in
one connection, then trying to lock it again in another:

```
Thread database.table_name Locked/Waiting Lock_type
7 sakila.film Locked - read Read lock without concurrent inserts
8 sakila.film Waiting - write Highest priority write lock
```
You can see that thread 8 is waiting for the lock thread 7 holds.

#### The Global Read Lock

The MySQL server also implements a global read lock. You can obtain this lock as
follows:

```
mysql> FLUSH TABLES WITH READ LOCK;
```
If you now try to lock a table in another session, it will hang as before:

```
mysql> LOCK TABLES sakila.film WRITE;
```
How can you tell that this query is waiting for the global read lock and not a table-level
lock? Look at the output of SHOW PROCESSLIST:

```
mysql> SHOW PROCESSLIST\G
...
*************************** 2. row ***************************
Id: 22
User: baron
Host: localhost
db: NULL
Command: Query
```
**738 | Appendix E: Debugging Locks**


```
Time: 9
State: Waiting for release of readlock
Info: LOCK TABLES sakila.film WRITE
```
Notice that the query’s state is Waiting for release of readlock. This is your clue that
the query is waiting for the global read lock, not a table-level lock.

MySQL provides no way to find out who holds the global read lock.

#### Name Locks

Name locks are a type of table lock that the server creates when it renames or drops a
table. A name lock conflicts with an ordinary table lock, whether implicit or explicit.
For example, if we use LOCK TABLES as before, and then in another session try to rename
the locked table, the query will hang, but this time not in the Locked state:

```
mysql> RENAME TABLE sakila.film2 TO sakila.film;
```
As before, the process list is the place to see the locked query, which is in the Waiting
for table state:

```
mysql> SHOW PROCESSLIST\G
...
*************************** 2. row ***************************
Id: 27
User: baron
Host: localhost
db: NULL
Command: Query
Time: 3
State: Waiting for table
Info: rename table sakila.film to sakila.film 2
```
You can see the effects of a name lock in the output of SHOW OPEN TABLES, too:

```
mysql> SHOW OPEN TABLES;
+----------+-----------+--------+-------------+
| Database | Table | In_use | Name_locked |
+----------+-----------+--------+-------------+
| sakila | film_text | 3 | 0 |
| sakila | film | 2 | 1 |
| sakila | film2 | 1 | 1 |
+----------+-----------+--------+-------------+
3 rows in set (0.00 sec)
```
Notice that both names (the original and the new name) are locked. sakila
.film_text is locked because there’s a trigger on sakila.film that refers to it, which
illustrates another way locks can insinuate themselves into places you might not expect.
If you query sakila.film, the trigger causes you to implicitly touch sakila.film_text,
and therefore to implicitly lock it. It’s true that the trigger really doesn’t need to fire for
the rename, and thus technically the lock isn’t required, but that’s the way it is:
MySQL’s locking is sometimes not as fine-grained as you might like.

```
Lock Waits at the Server Level | 739
```

MySQL doesn’t provide any way to find out who holds name locks, but this usually
isn’t a problem because they’re generally held for only a very short time. When there’s
a conflict, it is generally because a name lock is waiting for a normal table lock, which
you can view with _mysqladmin debug_ , as shown earlier.

#### User Locks

The final type of lock implemented in the server is the user lock, which is basically a
named mutex. You specify the string to lock and the number of seconds to wait before
the lock attempt should time out:

```
mysql> SELECT GET_LOCK('my lock', 100);
+--------------------------+
| GET_LOCK('my lock', 100) |
+--------------------------+
| 1 |
+--------------------------+
1 row in set (0.00 sec)
```
This attempt returned success immediately, so this thread now has a lock on that
named mutex. If another thread tries to lock the same string, it will hang until it times
out. This time the process list shows a different state:

```
mysql> SHOW PROCESSLIST\G
*************************** 1. row ***************************
Id: 22
User: baron
Host: localhost
db: NULL
Command: Query
Time: 9
State: User lock
Info: SELECT GET_LOCK('my lock', 100)
```
The User lock state is unique to this type of lock. MySQL provides no way to find out
who holds a user lock.

### Lock Waits in InnoDB

Locks at the server level can be quite a bit easier to debug than locks in storage engines.
Storage engine locks differ from one storage engine to the next, and the engines might
not provide any means to inspect their locks. We focus on InnoDB in this appendix.

InnoDB exposes some lock information in the output of SHOW INNODB STATUS. If a trans-
action is waiting for a lock, the lock will appear in the TRANSACTIONS section of the output
from SHOW INNODB STATUS. For example, if you execute the following commands in one
session, you will acquire a write lock on the first row in the table:

```
mysql> SET AUTOCOMMIT=0;
mysql> BEGIN;
mysql> SELECT film_id FROM sakila.film LIMIT 1 FOR UPDATE;
```
**740 | Appendix E: Debugging Locks**


If you now run the same commands in another session, your query will block on the
lock the first session acquired on that row. You can see the effects in SHOW INNODB
STATUS (we’ve abbreviated the results for clarity):

```
1 LOCK WAIT 2 lock struct(s), heap size 1216
2 MySQL thread id 8, query id 89 localhost baron Sending data
3 SELECT film_id FROM sakila.film LIMIT 1 FOR UPDATE
4 ------- TRX HAS BEEN WAITING 9 SEC FOR THIS LOCK TO BE GRANTED:
5 RECORD LOCKS space id 0 page no 194 n bits 1072 index `idx_fk_language_id` of table
`sakila/film` trx id 0 61714 lock_mode X waiting
```
The last line shows that the query is waiting for an exclusive (lock_mode X) lock on page
194 of the table’s idx_fk_language_id index. Eventually, the lock wait timeout will be
exceeded, and the query will return an error:

```
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```
Unfortunately, without seeing who holds the locks, it’s hard to figure out which trans-
action is causing the problem. You can often make an educated guess by looking at
which transactions have been open a very long time; alternatively, you can activate the
InnoDB lock monitor, which will show up to 10 of the locks each transaction holds.
To activate the monitor, you create a magically named table with the InnoDB storage
engine:^2

```
mysql> CREATE TABLE innodb_lock_monitor(a int) ENGINE=INNODB;
```
When you issue this query, InnoDB begins printing a slightly enhanced version of the
output of SHOW INNODB STATUS to standard output at intervals (the interval varies, but
it’s usually several times per minute). On most systems, this output is redirected to the
server’s error log; you can examine it to see which transactions hold which locks. To
stop the lock monitor, drop the table.

Here’s the relevant sample of the lock monitor output:

```
1 ---TRANSACTION 0 61717, ACTIVE 3 sec, process no 5102, OS thread id 1141152080
2 3 lock struct(s), heap size 1216
3 MySQL thread id 11, query id 108 localhost baron
4 show innodb status
5 TABLE LOCK table `sakila/film` trx id 0 61717 lock mode IX
6 RECORD LOCKS space id 0 page no 194 n bits 1072 index `idx_fk_language_id` of table
`sakila/film` trx id 0 61717 lock_mode X
7 Record lock, heap no 2 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
8 ... omitted ...
9
10 RECORD LOCKS space id 0 page no 231 n bits 168 index `PRIMARY` of table `sakila/film`
trx id 0 61717 lock_mode X locks rec but not gap
11 Record lock, heap no 2 PHYSICAL RECORD: n_fields 15; compact format; info bits 0
12 ... omitted ...
```
2. InnoDB honors several “magical” table names as instructions. Current practice is to use dynamically
    settable server variables, but InnoDB has been around a long time, so it still has some old behaviors.

```
Lock Waits in InnoDB | 741
```

Notice that line 3 shows the MySQL thread ID, which is the same as the value in the
Id column in the process list. Line 5 shows that the transaction has an implicit exclusive
table lock (IX) on the table. Lines 6 through 8 show the lock on the index. We’ve omitted
the information on line 8 because it’s a dump of the locked record and is pretty verbose.
Lines 9 through 11 show the corresponding lock on the primary key (a FOR UPDATE lock
must lock the row, not just the index).

When the lock monitor is activated the extra information appears in the output of SHOW
INNODB STATUS too, so you don’t actually have to look in the server’s error log to see the
lock information.

The lock monitor is not optimal, for several reasons. The primary problem is that the
lock information is very verbose, because it includes hex and ASCII dumps of the re-
cords that are locked. It fills up the error log, and it can easily overflow the fixed-size
output of SHOW INNODB STATUS. This means you might not get the information you’re
looking for in later sections of the output. InnoDB also has a hardcoded limit to the
number of locks it prints per transaction—after printing 10 locks, it will not print any
more, which means you might not even see any information on the lock you want. To
top it all off, even if what you’re looking for is there, it’s hard to find it in all that lock
output. (Just try it on a busy server, and you’ll see!)

Two things can make the lock output more usable. The first is a patch one of this book’s
authors wrote for InnoDB and the MySQL server, which is included in Percona Server
and MariaDB. The patch removes the verbose record dumps from the output, includes
the lock information in the output of SHOW INNODB STATUS by default (so the lock monitor
doesn’t need to be activated), and adds dynamically settable server variables to control
the verbosity and how many locks should be printed per transaction.

The second option is to use _innotop_ to parse and format the output. Its Lock mode
shows locks, aggregated neatly by connection and table, so you can see quickly which
transactions hold locks on a given table. This is not a foolproof method of finding which
transaction is blocking a lock, because that would require examining the dumped re-
cords to find the precise record that’s locked. However, it’s much better than the usual
alternatives, and it’s good enough for many purposes.

#### Using the INFORMATION_SCHEMA Tables

Using SHOW INNODB STATUS to look at locks is definitely old-school, now that InnoDB
has INFORMATION_SCHEMA tables that expose its transactions and locks.

If you don’t see the tables, you are not using a new enough version of InnoDB. You
need at least MySQL 5.1 and the InnoDB plugin. If you’re using MySQL 5.1 and you
don’t see the INNODB_LOCKS table, check SHOW VARIABLES for the innodb_version variable.
If you don’t see the variable, you’re not using the InnoDB plugin, and you should be!
If you see the variable but you don’t have the tables, you need to ensure that the

**742 | Appendix E: Debugging Locks**


plugin_load setting in the server configuration file includes the tables explicitly. Check
the MySQL manual for details.

Fortunately, in MySQL 5.5 you don’t need to worry about all of this; the modern version
of InnoDB is built right into the server.

The MySQL and InnoDB manuals have sample queries you can use against these tables,
which we won’t repeat here, but we’ll add a couple of our own. For example, here is a
query that shows who’s blocking and who’s waiting, and for how long:

```
SELECT r.trx_id AS waiting_trx_id, r.trx_mysql_thread_id AS waiting_thread,
TIMESTAMPDIFF(SECOND, r.trx_wait_started, CURRENT_TIMESTAMP) AS wait_time,
r.trx_query AS waiting_query,
l.lock_table AS waiting_table_lock,
b.trx_id AS blocking_trx_id, b.trx_mysql_thread_id AS blocking_thread,
SUBSTRING(p.host, 1, INSTR(p.host, ':') - 1) AS blocking_host,
SUBSTRING(p.host, INSTR(p.host, ':') +1) AS blocking_port,
IF(p.command = "Sleep", p.time, 0) AS idle_in_trx,
b.trx_query AS blocking_query
FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS AS w
INNER JOIN INFORMATION_SCHEMA.INNODB_TRX AS b ON b.trx_id = w.blocking_trx_id
INNER JOIN INFORMATION_SCHEMA.INNODB_TRX AS r ON r.trx_id = w.requesting_trx_id
INNER JOIN INFORMATION_SCHEMA.INNODB_LOCKS AS l ON w.requested_lock_id = l.lock_id
LEFT JOIN INFORMATION_SCHEMA.PROCESSLIST AS p ON p.id = b.trx_mysql_thread_id
ORDER BY wait_time DESC\G
*************************** 1. row ***************************
waiting_trx_id: 5D03
waiting_thread: 3
wait_time: 6
waiting_query: select * from store limit 1 for update
waiting_table_lock: `sakila`.`store`
blocking_trx_id: 5D02
blocking_thread: 2
blocking_host: localhost
blocking_port: 40298
idle_in_trx: 8
blocking_query: NULL
```
The result shows that thread 3 has been waiting for 6 seconds to lock a row in the
store table. It is blocked on thread 2, which has been idle for 8 seconds.

If you’re suffering from a lot of locking due to threads that are idle in a transaction, the
following variation can show you how many queries are blocked on which threads,
without all the verbosity:

```
SELECT CONCAT('thread ', b.trx_mysql_thread_id, ' from ', p.host) AS who_blocks,
IF(p.command = "Sleep", p.time, 0) AS idle_in_trx,
MAX(TIMESTAMPDIFF(SECOND, r.trx_wait_started, NOW())) AS max_wait_time,
COUNT(*) AS num_waiters
FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS AS w
INNER JOIN INFORMATION_SCHEMA.INNODB_TRX AS b ON b.trx_id = w.blocking_trx_id
INNER JOIN INFORMATION_SCHEMA.INNODB_TRX AS r ON r.trx_id = w.requesting_trx_id
LEFT JOIN INFORMATION_SCHEMA.PROCESSLIST AS p ON p.id = b.trx_mysql_thread_id
GROUP BY who_blocks ORDER BY num_waiters DESC\G
*************************** 1. row ***************************
```
```
Lock Waits in InnoDB | 743
```

```
who_blocks: thread 2 from localhost:40298
idle_in_trx: 1016
max_wait_time: 37
num_waiters: 8
```
The result shows that thread 2 has now been idle for a much longer time, and at least
one thread has been waiting for up to 37 seconds for it to release its locks. There are
eight threads waiting for thread 2 to finish its work and commit.

We’ve found that idle-in-transaction locking is a common cause of emergency prob-
lems, and is sometimes difficult for people to diagnose. The _pt-kill_ tool from Percona
Toolkit can be configured to kill long-running idle transactions to prevent this situation.
Percona Server itself also supports an idle transaction timeout parameter to accomplish
the same thing.

**744 | Appendix E: Debugging Locks**
