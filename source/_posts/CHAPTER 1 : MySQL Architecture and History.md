MySQL is very different from other database servers, and its architectural characteristics
make it useful for a wide range of purposes as well as making it a poor choice for
others. MySQL is not perfect, but it is flexible enough to work well in very demanding
environments, such as web applications. At the same time, MySQL can power embedded
applications, data warehouses, content indexing and delivery software, highly
available redundant systems, online transaction processing (OLTP), and much more.

To get the most from MySQL, you need to understand its design so that you can
work with it, not against it. MySQL is flexible in many ways. For example, you can
configure it to run well on a wide range of hardware, and it supports a variety of data
types. However, MySQL’s most unusual and important feature is its storage-engine
architecture, whose design separates query processing and other server tasks from data
storage and retrieval. This separation of concerns lets you choose how your data is
stored and what performance, features, and other characteristics you want.

This chapter provides a high-level overview of the MySQL server architecture, the major
differences between the storage engines, and why those differences are important. We’ll
finish with some historical context and benchmarks. We’ve tried to explain MySQL by
simplifying the details and showing examples. This discussion will be useful for those
new to database servers as well as readers who are experts with other database servers.

### MySQL’s Logical Architecture

A good mental picture of how MySQL’s components work together will help you un-
derstand the server. Figure 1-1 shows a logical view of MySQL’s architecture.

The topmost layer contains the services that aren’t unique to MySQL. They’re services
most network-based client/server tools or servers need: connection handling, authen-
tication, security, and so forth.

The second layer is where things get interesting. Much of MySQL’s brains are here,
including the code for query parsing, analysis, optimization, caching, and all the

```
1
```

built-in functions (e.g., dates, times, math, and encryption). Any functionality provided
across storage engines lives at this level: stored procedures, triggers, and views, for
example.

The third layer contains the storage engines. They are responsible for storing and
retrieving all data stored “in” MySQL. Like the various filesystems available for GNU/
Linux, each storage engine has its own benefits and drawbacks. The server communi-
cates with them through the _storage engine API_. This interface hides differences
between storage engines and makes them largely transparent at the query layer. The
API contains a couple of dozen low-level functions that perform operations such as
“begin a transaction” or “fetch the row that has this primary key.” The storage engines
don’t parse SQL^1 or communicate with each other; they simply respond to requests
from the server.

#### Connection Management and Security

Each client connection gets its own thread within the server process. The connection’s
queries execute within that single thread, which in turn resides on one core or CPU.
The server caches threads, so they don’t need to be created and destroyed for each new
connection.^2

When clients (applications) connect to the MySQL server, the server needs to authen-
ticate them. Authentication is based on username, originating host, and password.

_Figure 1-1. A logical view of the MySQL server architecture_

1. One exception is InnoDB, which does parse foreign key definitions, because the MySQL server doesn’t
    yet implement them itself.
2. MySQL 5.5 and newer versions support an API that can accept thread-pooling plugins, so a small pool
    of threads can service many connections.

**2 | Chapter 1: MySQL Architecture and History**


X.509 certificates can also be used across an SSL (Secure Sockets Layer) connection.
Once a client has connected, the server verifies whether the client has privileges for
each query it issues (e.g., whether the client is allowed to issue a SELECT statement that
accesses the Country table in the world database).

#### Optimization and Execution

MySQL parses queries to create an internal structure (the parse tree), and then applies
a variety of optimizations. These can include rewriting the query, determining the order
in which it will read tables, choosing which indexes to use, and so on. You can pass
hints to the optimizer through special keywords in the query, affecting its decision-
making process. You can also ask the server to explain various aspects of optimization.
This lets you know what decisions the server is making and gives you a reference point
for reworking queries, schemas, and settings to make everything run as efficiently as
possible. We discuss the optimizer in much more detail in Chapter 6.

The optimizer does not really care what storage engine a particular table uses, but the
storage engine does affect how the server optimizes the query. The optimizer asks
the storage engine about some of its capabilities and the cost of certain operations, and
for statistics on the table data. For instance, some storage engines support index types
that can be helpful to certain queries. You can read more about indexing and schema
optimization in Chapter 4 and Chapter 5.

Before even parsing the query, though, the server consults the query cache, which can
store only SELECT statements, along with their result sets. If anyone issues a query that’s
identical to one already in the cache, the server doesn’t need to parse, optimize, or
execute the query at all—it can simply pass back the stored result set. We write more
about that in Chapter 7.

### Concurrency Control

Anytime more than one query needs to change data at the same time, the problem of
concurrency control arises. For our purposes in this chapter, MySQL has to do this at
two levels: the server level and the storage engine level. Concurrency control is a big
topic to which a large body of theoretical literature is devoted, so we will just give you
a simplified overview of how MySQL deals with concurrent readers and writers, so you
have the context you need for the rest of this chapter.

We’ll use an email box on a Unix system as an example. The classic _mbox_ file format
is very simple. All the messages in an _mbox_ mailbox are concatenated together, one
after another. This makes it very easy to read and parse mail messages. It also makes
mail delivery easy: just append a new message to the end of the file.

```
Concurrency Control | 3
```

But what happens when two processes try to deliver messages at the same time to the
same mailbox? Clearly that could corrupt the mailbox, leaving two interleaved mes-
sages at the end of the mailbox file. Well-behaved mail delivery systems use locking to
prevent corruption. If a client attempts a second delivery while the mailbox is locked,
it must wait to acquire the lock itself before delivering its message.

This scheme works reasonably well in practice, but it gives no support for concurrency.
Because only a single process can change the mailbox at any given time, this approach
becomes problematic with a high-volume mailbox.

#### Read/Write Locks

Reading from the mailbox isn’t as troublesome. There’s nothing wrong with multiple
clients reading the same mailbox simultaneously; because they aren’t making changes,
nothing is likely to go wrong. But what happens if someone tries to delete message
number 25 while programs are reading the mailbox? It depends, but a reader could
come away with a corrupted or inconsistent view of the mailbox. So, to be safe, even
reading from a mailbox requires special care.

If you think of the mailbox as a database table and each mail message as a row, it’s easy
to see that the problem is the same in this context. In many ways, a mailbox is really
just a simple database table. Modifying rows in a database table is very similar to re-
moving or changing the content of messages in a mailbox file.

The solution to this classic problem of concurrency control is rather simple. Systems
that deal with concurrent read/write access typically implement a locking system that
consists of two lock types. These locks are usually known as _shared locks_ and _exclusive
locks_ , or _read locks_ and _write locks_.

Without worrying about the actual locking technology, we can describe the concept as
follows. Read locks on a resource are shared, or mutually nonblocking: many clients
can read from a resource at the same time and not interfere with each other. Write
locks, on the other hand, are exclusive—i.e., they block both read locks and other write
locks—because the only safe policy is to have a single client writing to the resource at
a given time and to prevent all reads when a client is writing.

In the database world, locking happens all the time: MySQL has to prevent one client
from reading a piece of data while another is changing it. It performs this lock man-
agement internally in a way that is transparent much of the time.

#### Lock Granularity

One way to improve the concurrency of a shared resource is to be more selective about
what you lock. Rather than locking the entire resource, lock only the part that contains
the data you need to change. Better yet, lock only the exact piece of data you plan to

**4 | Chapter 1: MySQL Architecture and History**


change. Minimizing the amount of data that you lock at any one time lets changes to
a given resource occur simultaneously, as long as they don’t conflict with each other.

The problem is locks consume resources. Every lock operation—getting a lock, check-
ing to see whether a lock is free, releasing a lock, and so on—has overhead. If the system
spends too much time managing locks instead of storing and retrieving data, perfor-
mance can suffer.

A locking strategy is a compromise between lock overhead and data safety, and that
compromise affects performance. Most commercial database servers don’t give you
much choice: you get what is known as row-level locking in your tables, with a variety
of often complex ways to give good performance with many locks.

MySQL, on the other hand, does offer choices. Its storage engines can implement their
own locking policies and lock granularities. Lock management is a very important de-
cision in storage engine design; fixing the granularity at a certain level can give better
performance for certain uses, yet make that engine less suited for other purposes. Be-
cause MySQL offers multiple storage engines, it doesn’t require a single general-
purpose solution. Let’s have a look at the two most important lock strategies.

**Table locks**

The most basic locking strategy available in MySQL, and the one with the lowest over-
head, is _table locks_. A table lock is analogous to the mailbox locks described earlier: it
locks the entire table. When a client wishes to write to a table (insert, delete, update,
etc.), it acquires a write lock. This keeps all other read and write operations at bay.
When nobody is writing, readers can obtain read locks, which don’t conflict with other
read locks.

Table locks have variations for good performance in specific situations. For example,
READ LOCAL table locks allow some types of concurrent write operations. Write locks
also have a higher priority than read locks, so a request for a write lock will advance to
the front of the lock queue even if readers are already in the queue (write locks can
advance past read locks in the queue, but read locks cannot advance past write locks).

Although storage engines can manage their own locks, MySQL itself also uses a variety
of locks that are effectively table-level for various purposes. For instance, the server
uses a table-level lock for statements such as ALTER TABLE, regardless of the storage
engine.

**Row locks**

The locking style that offers the greatest concurrency (and carries the greatest overhead)
is the use of _row locks_. Row-level locking, as this strategy is commonly known, is
available in the InnoDB and XtraDB storage engines, among others. Row locks are
implemented in the storage engine, not the server (refer back to the logical architecture
diagram if you need to). The server is completely unaware of locks implemented in the

```
Concurrency Control | 5
```

storage engines, and as you’ll see later in this chapter and throughout the book, the
storage engines all implement locking in their own ways.

### Transactions

You can’t examine the more advanced features of a database system for very long before
_transactions_ enter the mix. A transaction is a group of SQL queries that are treated
_atomically_ , as a single unit of work. If the database engine can apply the entire group
of queries to a database, it does so, but if any of them can’t be done because of a crash
or other reason, none of them is applied. It’s all or nothing.

Little of this section is specific to MySQL. If you’re already familiar with ACID trans-
actions, feel free to skip ahead to “Transactions in MySQL” on page 10.

A banking application is the classic example of why transactions are necessary. Imagine
a bank’s database with two tables: checking and savings. To move $200 from Jane’s
checking account to her savings account, you need to perform at least three steps:

1. Make sure her checking account balance is greater than $200.
2. Subtract $200 from her checking account balance.
3. Add $200 to her savings account balance.

The entire operation should be wrapped in a transaction so that if any one of the steps
fails, any completed steps can be rolled back.

You start a transaction with the START TRANSACTION statement and then either make its
changes permanent with COMMIT or discard the changes with ROLLBACK. So, the SQL for
our sample transaction might look like this:

```
1 START TRANSACTION;
2 SELECT balance FROM checking WHERE customer_id = 10233276;
3 UPDATE checking SET balance = balance - 200.00 WHERE customer_id = 10233276;
4 UPDATE savings SET balance = balance + 200.00 WHERE customer_id = 10233276;
5 COMMIT;
```
But transactions alone aren’t the whole story. What happens if the database server
crashes while performing line 4? Who knows? The customer probably just lost $200.
And what if another process comes along between lines 3 and 4 and removes the entire
checking account balance? The bank has given the customer a $200 credit without even
knowing it.

Transactions aren’t enough unless the system passes the _ACID test_. ACID stands for
Atomicity, Consistency, Isolation, and Durability. These are tightly related criteria that
a well-behaved transaction processing system must meet:

_Atomicity_
A transaction must function as a single indivisible unit of work so that the entire
transaction is either applied or rolled back. When transactions are atomic, there is
no such thing as a partially completed transaction: it’s all or nothing.

**6 | Chapter 1: MySQL Architecture and History**


_Consistency_
The database should always move from one consistent state to the next. In our
example, consistency ensures that a crash between lines 3 and 4 doesn’t result in
$200 disappearing from the checking account. Because the transaction is never
committed, none of the transaction’s changes are ever reflected in the database.

_Isolation_
The results of a transaction are usually invisible to other transactions until the
transaction is complete. This ensures that if a bank account summary runs after
line 3 but before line 4 in our example, it will still see the $200 in the checking
account. When we discuss isolation levels, you’ll understand why we said _usu-
ally_ invisible.

_Durability_
Once committed, a transaction’s changes are permanent. This means the changes
must be recorded such that data won’t be lost in a system crash. Durability is a
slightly fuzzy concept, however, because there are actually many levels. Some du-
rability strategies provide a stronger safety guarantee than others, and nothing is
ever 100% durable (if the database itself were truly durable, then how could back-
ups increase durability?). We discuss what durability _really_ means in MySQL in
later chapters.

ACID transactions ensure that banks don’t lose your money. It is generally extremely
difficult or impossible to do this with application logic. An ACID-compliant database
server has to do all sorts of complicated things you might not realize to provide ACID
guarantees.

Just as with increased lock granularity, the downside of this extra security is that the
database server has to do more work. A database server with ACID transactions also
generally requires more CPU power, memory, and disk space than one without them.
As we’ve said several times, this is where MySQL’s storage engine architecture works
to your advantage. You can decide whether your application needs transactions. If you
don’t really need them, you might be able to get higher performance with a nontran-
sactional storage engine for some kinds of queries. You might be able to use LOCK
TABLES to give the level of protection you need without transactions. It’s all up to you.

#### Isolation Levels

Isolation is more complex than it looks. The SQL standard defines four isolation levels,
with specific rules for which changes are and aren’t visible inside and outside a trans-
action. Lower isolation levels typically allow higher concurrency and have lower
overhead.

```
Transactions | 7
```

```
Each storage engine implements isolation levels slightly differently, and
they don’t necessarily match what you might expect if you’re used to
another database product (thus, we won’t go into exhaustive detail in
this section). You should read the manuals for whichever storage en-
gines you decide to use.
```
Let’s take a quick look at the four isolation levels:

READ UNCOMMITTED
In the READ UNCOMMITTED isolation level, transactions can view the results of un-
committed transactions. At this level, many problems can occur unless you really,
really know what you are doing and have a good reason for doing it. This level is
rarely used in practice, because its performance isn’t much better than the other
levels, which have many advantages. Reading uncommitted data is also known as
a _dirty read_.

READ COMMITTED
The default isolation level for most database systems (but not MySQL!) is READ
COMMITTED. It satisfies the simple definition of isolation used earlier: a transaction
will see only those changes made by transactions that were already committed
when it began, and its changes won’t be visible to others until it has committed.
This level still allows what’s known as a _nonrepeatable read_. This means you can
run the same statement twice and see different data.

REPEATABLE READ
REPEATABLE READ solves the problems that READ UNCOMMITTED allows. It guarantees
that any rows a transaction reads will “look the same” in subsequent reads within
the same transaction, but in theory it still allows another tricky problem: _phantom
reads_. Simply put, a phantom read can happen when you select some range of rows,
another transaction inserts a new row into the range, and then you select the same
range again; you will then see the new “phantom” row. InnoDB and XtraDB solve
the phantom read problem with multiversion concurrency control, which we ex-
plain later in this chapter.
REPEATABLE READ is MySQL’s default transaction isolation level.

SERIALIZABLE
The highest level of isolation, SERIALIZABLE, solves the phantom read problem by
forcing transactions to be ordered so that they can’t possibly conflict. In a nutshell,
SERIALIZABLE places a lock on every row it reads. At this level, a lot of timeouts and
lock contention can occur. We’ve rarely seen people use this isolation level, but
your application’s needs might force you to accept the decreased concurrency in
favor of the data stability that results.

Table 1-1 summarizes the various isolation levels and the drawbacks associated with
each one.

**8 | Chapter 1: MySQL Architecture and History**


_Table 1-1. ANSI SQL isolation levels_

```
Isolation level Dirty reads possible
```
```
Nonrepeatable reads
possible
```
```
Phantom reads
possible Locking reads
READ UNCOMMITTED Yes Yes Yes No
READ COMMITTED No Yes Yes No
REPEATABLE READ No No Yes No
SERIALIZABLE No No No Yes
```
#### Deadlocks

A _deadlock_ is when two or more transactions are mutually holding and requesting locks
on the same resources, creating a cycle of dependencies. Deadlocks occur when trans-
actions try to lock resources in a different order. They can happen whenever multiple
transactions lock the same resources. For example, consider these two transactions
running against the StockPrice table:

_Transaction #1_

```
START TRANSACTION;
UPDATE StockPrice SET close = 45.50 WHERE stock_id = 4 and date = '2002-05-01';
UPDATE StockPrice SET close = 19.80 WHERE stock_id = 3 and date = '2002-05-02';
COMMIT;
```
_Transaction #2_

```
START TRANSACTION;
UPDATE StockPrice SET high = 20.12 WHERE stock_id = 3 and date = '2002-05-02';
UPDATE StockPrice SET high = 47.20 WHERE stock_id = 4 and date = '2002-05-01';
COMMIT;
```
If you’re unlucky, each transaction will execute its first query and update a row of data,
locking it in the process. Each transaction will then attempt to update its second row,
only to find that it is already locked. The two transactions will wait forever for each
other to complete, unless something intervenes to break the deadlock.

To combat this problem, database systems implement various forms of deadlock de-
tection and timeouts. The more sophisticated systems, such as the InnoDB storage
engine, will notice circular dependencies and return an error instantly. This can be a
good thing—otherwise, deadlocks would manifest themselves as very slow queries.
Others will give up after the query exceeds a lock wait timeout, which is not always
good. The way InnoDB currently handles deadlocks is to roll back the transaction that
has the fewest exclusive row locks (an approximate metric for which will be the easiest
to roll back).

Lock behavior and order are storage engine–specific, so some storage engines might
deadlock on a certain sequence of statements even though others won’t. Deadlocks
have a dual nature: some are unavoidable because of true data conflicts, and some are
caused by how a storage engine works.

```
Transactions | 9
```

Deadlocks cannot be broken without rolling back one of the transactions, either par-
tially or wholly. They are a fact of life in transactional systems, and your applications
should be designed to handle them. Many applications can simply retry their transac-
tions from the beginning.

#### Transaction Logging

Transaction logging helps make transactions more efficient. Instead of updating the
tables on disk each time a change occurs, the storage engine can change its in-memory
copy of the data. This is very fast. The storage engine can then write a record of the
change to the transaction log, which is on disk and therefore durable. This is also a
relatively fast operation, because appending log events involves sequential I/O in one
small area of the disk instead of random I/O in many places. Then, at some later time,
a process can update the table on disk. Thus, most storage engines that use this tech-
nique (known as _write-ahead logging_ ) end up writing the changes to disk twice.

If there’s a crash after the update is written to the transaction log but before the changes
are made to the data itself, the storage engine can still recover the changes upon restart.
The recovery method varies between storage engines.

#### Transactions in MySQL

MySQL provides two transactional storage engines: InnoDB and NDB Cluster. Several
third-party engines are also available; the best-known engines right now are XtraDB
and PBXT. We discuss some specific properties of each engine in the next section.

**AUTOCOMMIT**

MySQL operates in AUTOCOMMIT mode by default. This means that unless you’ve ex-
plicitly begun a transaction, it automatically executes each query in a separate trans-
action. You can enable or disable AUTOCOMMIT for the current connection by setting a
variable:

```
mysql> SHOW VARIABLES LIKE 'AUTOCOMMIT';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit | ON |
+---------------+-------+
1 row in set (0.00 sec)
mysql> SET AUTOCOMMIT = 1;
```
The values 1 and ON are equivalent, as are 0 and OFF. When you run with AUTOCOMMIT
=0, you are always in a transaction, until you issue a COMMIT or ROLLBACK. MySQL then
starts a new transaction immediately. Changing the value of AUTOCOMMIT has no effect
on nontransactional tables, such as MyISAM or Memory tables, which have no notion
of committing or rolling back changes.

**10 | Chapter 1: MySQL Architecture and History**


Certain commands, when issued during an open transaction, cause MySQL to commit
the transaction before they execute. These are typically Data Definition Language
(DDL) commands that make significant changes, such as ALTER TABLE, but LOCK
TABLES and some other statements also have this effect. Check your version’s docu-
mentation for the full list of commands that automatically commit a transaction.

MySQL lets you set the isolation level using the SET TRANSACTION ISOLATION LEVEL
command, which takes effect when the next transaction starts. You can set the isolation
level for the whole server in the configuration file, or just for your session:

```
mysql> SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```
MySQL recognizes all four ANSI standard isolation levels, and InnoDB supports all of
them.

**Mixing storage engines in transactions**

MySQL doesn’t manage transactions at the server level. Instead, the underlying storage
engines implement transactions themselves. This means you can’t reliably mix different
engines in a single transaction.

If you mix transactional and nontransactional tables (for instance, InnoDB and
MyISAM tables) in a transaction, the transaction will work properly if all goes well.

However, if a rollback is required, the changes to the nontransactional table can’t be
undone. This leaves the database in an inconsistent state from which it might be difficult
to recover and renders the entire point of transactions moot. This is why it is really
important to pick the right storage engine for each table.

MySQL will not usually warn you or raise errors if you do transactional operations on
a nontransactional table. Sometimes rolling back a transaction will generate the warn-
ing “Some nontransactional changed tables couldn’t be rolled back,” but most of the
time, you’ll have no indication you’re working with nontransactional tables.

**Implicit and explicit locking**

InnoDB uses a two-phase locking protocol. It can acquire locks at any time during a
transaction, but it does not release them until a COMMIT or ROLLBACK. It releases all the
locks at the same time. The locking mechanisms described earlier are all implicit.
InnoDB handles locks automatically, according to your isolation level.

However, InnoDB also supports explicit locking, which the SQL standard does not
mention at all:^3

- SELECT ... LOCK IN SHARE MODE
- SELECT ... FOR UPDATE
    3. These locking hints are frequently abused and should usually be avoided; see Chapter 6 for more details.

```
Transactions | 11
```

MySQL also supports the LOCK TABLES and UNLOCK TABLES commands, which are im-
plemented in the server, not in the storage engines. These have their uses, but they are
not a substitute for transactions. If you need transactions, use a transactional storage
engine.

We often see applications that have been converted from MyISAM to InnoDB but are
still using LOCK TABLES. This is no longer necessary because of row-level locking, and it
can cause severe performance problems.

```
The interaction between LOCK TABLES and transactions is complex, and
there are unexpected behaviors in some server versions. Therefore, we
recommend that you never use LOCK TABLES unless you are in a trans-
action and AUTOCOMMIT is disabled, no matter what storage engine you
are using.
```
### Multiversion Concurrency Control

Most of MySQL’s transactional storage engines don’t use a simple row-locking mech-
anism. Instead, they use row-level locking in conjunction with a technique for increas-
ing concurrency known as _multiversion concurrency control_ (MVCC). MVCC is not
unique to MySQL: Oracle, PostgreSQL, and some other database systems use it too,
although there are significant differences because there is no standard for how MVCC
should work.

You can think of MVCC as a twist on row-level locking; it avoids the need for locking
at all in many cases and can have much lower overhead. Depending on how it is im-
plemented, it can allow nonlocking reads, while locking only the necessary rows during
write operations.

MVCC works by keeping a snapshot of the data as it existed at some point in time.
This means transactions can see a consistent view of the data, no matter how long they
run. It also means different transactions can see different data in the same tables at the
same time! If you’ve never experienced this before, it might be confusing, but it will
become easier to understand with familiarity.

Each storage engine implements MVCC differently. Some of the variations include
_optimistic_ and _pessimistic_ concurrency control. We’ll illustrate one way MVCC works
by explaining a simplified version of InnoDB’s behavior.

InnoDB implements MVCC by storing with each row two additional, hidden values
that record when the row was created and when it was expired (or deleted). Rather
than storing the actual times at which these events occurred, the row stores the system
version number at the time each event occurred. This is a number that increments each
time a transaction begins. Each transaction keeps its own record of the current system
version, as of the time it began. Each query has to check each row’s version numbers

**12 | Chapter 1: MySQL Architecture and History**


against the transaction’s version. Let’s see how this applies to particular operations
when the transaction isolation level is set to REPEATABLE READ:

SELECT
InnoDB must examine each row to ensure that it meets two criteria:
a. InnoDB must find a version of the row that is at least as old as the transaction
(i.e., its version must be less than or equal to the transaction’s version). This
ensures that either the row existed before the transaction began, or the trans-
action created or altered the row.
b. The row’s deletion version must be undefined or greater than the transaction’s
version. This ensures that the row wasn’t deleted before the transaction began.
Rows that pass both tests may be returned as the query’s result.

INSERT
InnoDB records the current system version number with the new row.

DELETE
InnoDB records the current system version number as the row’s deletion ID.

UPDATE
InnoDB writes a new copy of the row, using the system version number for the new
row’s version. It also writes the system version number as the old row’s deletion
version.

The result of all this extra record keeping is that most read queries never acquire locks.
They simply read data as fast as they can, making sure to select only rows that meet
the criteria. The drawbacks are that the storage engine has to store more data with each
row, do more work when examining rows, and handle some additional housekeeping
operations.

MVCC works only with the REPEATABLE READ and READ COMMITTED isolation levels. READ
UNCOMMITTED isn’t MVCC-compatible^4 because queries don’t read the row version
that’s appropriate for their transaction version; they read the newest version, no matter
what. SERIALIZABLE isn’t MVCC-compatible because reads lock every row they return.

### MySQL’s Storage Engines

This section gives an overview of MySQL’s storage engines. We won’t go into great
detail here, because we discuss storage engines and their particular behaviors through-
out the book. Even this book, though, isn’t a complete source of documentation; you
should read the MySQL manuals for the storage engines you decide to use.

MySQL stores each database (also called a _schema_ ) as a subdirectory of its data directory
in the underlying filesystem. When you create a table, MySQL stores the table definition

4. There is no formal standard that defines MVCC, so different engines and databases implement it very
    differently, and no one can say any of them is wrong.

```
MySQL’s Storage Engines | 13
```

in a _.frm_ file with the same name as the table. Thus, when you create a table named
MyTable, MySQL stores the table definition in _MyTable.frm_. Because MySQL uses the
filesystem to store database names and table definitions, case sensitivity depends on
the platform. On a Windows MySQL instance, table and database names are case
insensitive; on Unix-like systems, they are case sensitive. Each storage engine stores the
table’s data and indexes differently, but the server itself handles the table definition.

You can use the SHOW TABLE STATUS command (or in MySQL 5.0 and newer versions,
query the INFORMATION_SCHEMA tables) to display information about tables. For example,
to examine the user table in the mysql database, execute the following:

```
mysql> SHOW TABLE STATUS LIKE 'user' \G
*************************** 1. row ***************************
Name: user
Engine: MyISAM
Row_format: Dynamic
Rows: 6
Avg_row_length: 59
Data_length: 356
Max_data_length: 4294967295
Index_length: 2048
Data_free: 0
Auto_increment: NULL
Create_time: 2002-01-24 18:07:17
Update_time: 2002-01-24 21:56:29
Check_time: NULL
Collation: utf8_bin
Checksum: NULL
Create_options:
Comment: Users and global privileges
1 row in set (0.00 sec)
```
The output shows that this is a MyISAM table. You might also notice a lot of other
information and statistics in the output. Let’s look briefly at what each line means:

Name
The table’s name.

Engine
The table’s storage engine. In old versions of MySQL, this column was named
Type, not Engine.

Row_format
The row format. For a MyISAM table, this can be Dynamic, Fixed, or Compressed.
Dynamic rows vary in length because they contain variable-length fields such as
VARCHAR or BLOB. Fixed rows, which are always the same size, are made up of fields
that don’t vary in length, such as CHAR and INTEGER. Compressed rows exist only in
compressed tables; see “Compressed MyISAM tables” on page 19.

Rows
The number of rows in the table. For MyISAM and most other engines, this number
is always accurate. For InnoDB, it is an estimate.

**14 | Chapter 1: MySQL Architecture and History**


Avg_row_length
How many bytes the average row contains.

Data_length
How much data (in bytes) the entire table contains.

Max_data_length
The maximum amount of data this table can hold. This is engine-specific.

Index_length
How much disk space the index data consumes.

Data_free
For a MyISAM table, the amount of space that is allocated but currently unused.
This space holds previously deleted rows and can be reclaimed by future INSERT
statements.

Auto_increment
The next AUTO_INCREMENT value.

Create_time
When the table was first created.

Update_time
When data in the table last changed.

Check_time
When the table was last checked using CHECK TABLE or _myisamchk_.

Collation
The default character set and collation for character columns in this table.

Checksum
A live checksum of the entire table’s contents, if enabled.

Create_options
Any other options that were specified when the table was created.

Comment
This field contains a variety of extra information. For a MyISAM table, it contains
the comments, if any, that were set when the table was created. If the table uses
the InnoDB storage engine, the amount of free space in the InnoDB tablespace
appears here. If the table is a view, the comment contains the text “VIEW.”

#### The InnoDB Engine

InnoDB is the default transactional storage engine for MySQL and the most important
and broadly useful engine overall. It was designed for processing many short-lived
transactions that usually complete rather than being rolled back. Its performance and
automatic crash recovery make it popular for nontransactional storage needs, too. _You
should use InnoDB for your tables unless you have a compelling need to use a different
engine_. If you want to study storage engines, it is also well worth your time to study

```
MySQL’s Storage Engines | 15
```

InnoDB in depth to learn as much as you can about it, rather than studying all storage
engines equally.

**InnoDB’s history**

InnoDB has a complex release history, but it’s very helpful to understand it. In 2008,
the so-called InnoDB plugin was released for MySQL 5.1. This was the next generation
of InnoDB created by Oracle, which at that time owned InnoDB but not MySQL. For
various reasons that are great to discuss over beers, MySQL continued shipping the
older version of InnoDB, compiled into the server. But you could disable this and install
the newer, better-performing, more scalable InnoDB plugin if you wished. Eventually,
Oracle acquired Sun Microsystems and thus MySQL, and removed the older codebase,
replacing it with the “plugin” by default in MySQL 5.5. (Yes, this means that now the
“plugin” is actually compiled in, not installed as a plugin. Old terminology dies hard.)

The modern version of InnoDB, introduced as the InnoDB plugin in MySQL 5.1, sports
new features such as building indexes by sorting, the ability to drop and add indexes
without rebuilding the whole table, and a new storage format that offers compression,
a new way to store large values such as BLOB columns, and file format management.
Many people who use MySQL 5.1 don’t use the plugin, sometimes because they aren’t
aware of it. If you’re using MySQL 5.1, please ensure that you’re using the InnoDB
plugin. It’s much better than the older version of InnoDB.

InnoDB is such an important engine that many people and companies have invested
in developing it, not just Oracle’s team. Notable contributions have come from Google,
Yasufumi Kinoshita, Percona, and Facebook, among others. Some of these improve-
ments have been included into the official InnoDB source code, and many others have
been reimplemented in slightly different ways by the InnoDB team. In general,
InnoDB’s development has accelerated greatly in the last few years, with major im-
provements to instrumentation, scalability, configurability, performance, features, and
support for Windows, among other notable items. MySQL 5.6 lab previews and mile-
stone releases include a remarkable palette of new features for InnoDB, too.

Oracle is investing tremendous resources in improving InnoDB performance, and doing
a great job of it (a considerable amount of external contribution has helped with this,
too). In the second edition of this book, we noted that InnoDB failed pretty miserably
beyond four CPU cores. It now scales well to 24 CPU cores, and arguably up to 32 or
even more cores depending on the scenario. Many improvements are slated for the
upcoming 5.6 release, but there are still opportunities for enhancement.

**InnoDB overview**

InnoDB stores its data in a series of one or more data files that are collectively known
as a _tablespace_. A tablespace is essentially a black box that InnoDB manages all by itself.
In MySQL 4.1 and newer versions, InnoDB can store each table’s data and indexes in

**16 | Chapter 1: MySQL Architecture and History**


separate files. InnoDB can also use raw disk partitions for building its tablespace, but
modern filesystems make this unnecessary.

InnoDB uses MVCC to achieve high concurrency, and it implements all four SQL stan-
dard isolation levels. It defaults to the REPEATABLE READ isolation level, and it has a
_next-key locking_ strategy that prevents phantom reads in this isolation level: rather than
locking only the rows you’ve touched in a query, InnoDB locks gaps in the index struc-
ture as well, preventing phantoms from being inserted.

InnoDB tables are built on a _clustered index_ , which we will cover in detail in later chap-
ters. InnoDB’s index structures are very different from those of most other MySQL
storage engines. As a result, it provides very fast primary key lookups. However, _sec-
ondary indexes_ (indexes that aren’t the primary key) contain the primary key columns,
so if your primary key is large, other indexes will also be large. You should strive for a
small primary key if you’ll have many indexes on a table. The storage format is platform-
neutral, meaning you can copy the data and index files from an Intel-based server to a
PowerPC or Sun SPARC without any trouble.

InnoDB has a variety of internal optimizations. These include predictive read-ahead for
prefetching data from disk, an adaptive hash index that automatically builds hash in-
dexes in memory for very fast lookups, and an insert buffer to speed inserts. We cover
these later in this book.

InnoDB’s behavior is very intricate, and we highly recommend reading the “InnoDB
Transaction Model and Locking” section of the MySQL manual if you’re using
InnoDB. There are many subtleties you should be aware of before building an appli-
cation with InnoDB, because of its MVCC architecture. Working with a storage engine
that maintains consistent views of the data for all users, even when some users are
changing data, can be complex.

As a transactional storage engine, InnoDB supports truly “hot” online backups through
a variety of mechanisms, including Oracle’s proprietary MySQL Enterprise Backup and
the open source Percona XtraBackup. MySQL’s other storage engines can’t take hot
backups—to get a consistent backup, you have to halt all writes to the table, which in
a mixed read/write workload usually ends up halting reads too.

#### The MyISAM Engine

As MySQL’s default storage engine in versions 5.1 and older, MyISAM provides a large
list of features, such as full-text indexing, compression, and spatial (GIS) functions.
MyISAM doesn’t support transactions or row-level locks. Its biggest weakness is un-
doubtedly the fact that it isn’t even remotely crash-safe. MyISAM is why MySQL still
has the reputation of being a nontransactional database management system, more
than a decade after it gained transactions! Still, MyISAM isn’t all that bad for a non-
transactional, non-crash-safe storage engine. If you need read-only data, or if your

```
MySQL’s Storage Engines | 17
```

tables aren’t large and won’t be painful to repair, it isn’t out of the question to use it.
(But please, don’t use it by default. Use InnoDB instead.)

**Storage**

MyISAM typically stores each table in two files: a data file and an index file. The two
files bear _.MYD_ and _.MYI_ extensions, respectively. MyISAM tables can contain either
dynamic or static (fixed-length) rows. MySQL decides which format to use based on
the table definition. The number of rows a MyISAM table can hold is limited primarily
by the available disk space on your database server and the largest file your operating
system will let you create.

MyISAM tables created in MySQL 5.0 with variable-length rows are configured by
default to handle 256 TB of data, using 6-byte pointers to the data records. Earlier
MySQL versions defaulted to 4-byte pointers, for up to 4 GB of data. All MySQL ver-
sions can handle a pointer size of up to 8 bytes. To change the pointer size on a MyISAM
table (either up or down), you must alter the table with new values for the MAX_ROWS
and AVG_ROW_LENGTH options that represent ballpark figures for the amount of space you
need. This will cause the entire table and all of its indexes to be rewritten, which might
take a long time.

**MyISAM features**

As one of the oldest storage engines included in MySQL, MyISAM has many features
that have been developed over years of use to fill niche needs:

_Locking and concurrency_
MyISAM locks entire tables, not rows. Readers obtain shared (read) locks on all
tables they need to read. Writers obtain exclusive (write) locks. However, you can
insert new rows into the table while select queries are running against it (concurrent
inserts).

_Repair_
MySQL supports manual and automatic checking and repairing of MyISAM tables,
but don’t confuse this with transactions or crash recovery. After repairing a table,
you’ll likely find that some data is simply gone. Repairing is slow, too. You can use
the CHECK TABLE mytable and REPAIR TABLE mytable commands to check a table for
errors and repair them. You can also use the _myisamchk_ command-line tool to
check and repair tables when the server is offline.

_Index features_
You can create indexes on the first 500 characters of BLOB and TEXT columns in
MyISAM tables. MyISAM supports full-text indexes, which index individual words
for complex search operations. For more information on indexing, see Chapter 5.

**18 | Chapter 1: MySQL Architecture and History**


_Delayed key writes_
MyISAM tables marked with the DELAY_KEY_WRITE create option don’t write
changed index data to disk at the end of a query. Instead, MyISAM buffers the
changes in the in-memory key buffer. It flushes index blocks to disk when it prunes
the buffer or closes the table. This can boost performance, but after a server or
system crash, the indexes will definitely be corrupted and will need repair. You can
configure delayed key writes globally, as well as for individual tables.

**Compressed MyISAM tables**

Some tables never change once they’re created and filled with data. These might be
well suited to compressed MyISAM tables.

You can compress (or “pack”) tables with the _myisampack_ utility. You can’t modify
compressed tables (although you can uncompress, modify, and recompress tables if
you need to), but they generally use less space on disk. As a result, they offer faster
performance, because their smaller size requires fewer disk seeks to find records. Com-
pressed MyISAM tables can have indexes, but they’re read-only.

The overhead of decompressing the data to read it is insignificant for most applications
on modern hardware, where the real gain is in reducing disk I/O. The rows are com-
pressed individually, so MySQL doesn’t need to unpack an entire table (or even a page)
just to fetch a single row.

**MyISAM performance**

Because of its compact data storage and low overhead due to its simpler design,
MyISAM can provide good performance for some uses. It does have some severe scal-
ability limitations, including mutexes on key caches. MariaDB offers a segmented key
cache that avoids this problem. The most common MyISAM performance problem we
see, however, is table locking. If your queries are all getting stuck in the “Locked” status,
you’re suffering from table-level locking.

#### Other Built-in MySQL Engines

MySQL has a variety of special-purpose storage engines. Many of them are somewhat
deprecated in newer versions, for various reasons. Some of these are still available in
the server, but must be enabled specially.

**The Archive engine**

The Archive engine supports only INSERT and SELECT queries, and it does not support
indexes until MySQL 5.1. It causes much less disk I/O than MyISAM, because it buffers
data writes and compresses each row with _zlib_ as it’s inserted. Also, each SELECT query
requires a full table scan. Archive tables are thus best for logging and data acquisition,
where analysis tends to scan an entire table, or where you want fast INSERT queries.

```
MySQL’s Storage Engines | 19
```

Archive supports row-level locking and a special buffer system for high-concurrency
inserts. It gives consistent reads by stopping a SELECT after it has retrieved the number
of rows that existed in the table when the query began. It also makes bulk inserts
invisible until they’re complete. These features emulate some aspects of transactional
and MVCC behaviors, but Archive is not a transactional storage engine. It is simply a
storage engine that’s optimized for high-speed inserting and compressed storage.

**The Blackhole engine**

The Blackhole engine has no storage mechanism at all. It discards every INSERT instead
of storing it. However, the server writes queries against Blackhole tables to its logs, so
they can be replicated or simply kept in the log. That makes the Blackhole engine
popular for fancy replication setups and audit logging, although we’ve seen enough
problems caused by such setups that we don’t recommend them.

**The CSV engine**

The CSV engine can treat comma-separated values (CSV) files as tables, but it does not
support indexes on them. This engine lets you copy files into and out of the database
while the server is running. If you export a CSV file from a spreadsheet and save it in
the MySQL server’s data directory, the server can read it immediately. Similarly, if you
write data to a CSV table, an external program can read it right away. CSV tables are
thus useful as a data interchange format.

**The Federated engine**

This storage engine is sort of a proxy to other servers. It opens a client connection to
another server and executes queries against a table there, retrieving and sending rows
as needed. It was originally marketed as a competitor to features supported in many
enterprise-grade proprietary database servers, such as Microsoft SQL Server and
Oracle, but that was always a stretch, to say the least. Although it seemed to enable a
lot of flexibility and neat tricks, it has proven to be a source of many problems and is
disabled by default. A successor to it, FederatedX, is available in MariaDB.

**The Memory engine**

Memory tables (formerly called HEAP tables) are useful when you need fast access to
data that either never changes or doesn’t need to persist after a restart. Memory tables
can be up to an order of magnitude faster than MyISAM tables. All of their data is stored
in memory, so queries don’t have to wait for disk I/O. The table structure of a Memory
table persists across a server restart, but no data survives.

Here are some good uses for Memory tables:

- For “lookup” or “mapping” tables, such as a table that maps postal codes to state
    names

**20 | Chapter 1: MySQL Architecture and History**


- For caching the results of periodically aggregated data
- For intermediate results when analyzing data

Memory tables support HASH indexes, which are very fast for lookup queries. Although
Memory tables are very fast, they often don’t work well as a general-purpose
replacement for disk-based tables. They use table-level locking, which gives low write
concurrency. They do not support TEXT or BLOB column types, and they support only
fixed-size rows, so they really store VARCHARs as CHARs, which can waste memory. (Some
of these limitations are lifted in Percona Server.)

MySQL uses the Memory engine internally while processing queries that require a
temporary table to hold intermediate results. If the intermediate result becomes too
large for a Memory table, or has TEXT or BLOB columns, MySQL will convert it to a
MyISAM table on disk. We say more about this in later chapters.

```
People often confuse Memory tables with temporary tables, which are
ephemeral tables created with CREATE TEMPORARY TABLE. Temporary
tables can use any storage engine; they are not the same thing as tables
that use the Memory storage engine. Temporary tables are visible only
to a single connection and disappear entirely when the connection
closes.
```
**The Merge storage engine**

The Merge engine is a variation of MyISAM. A Merge table is the combination of several
identical MyISAM tables into one virtual table. This can be useful when you use MySQL
in logging and data warehousing applications, but it has been deprecated in favor of
partitioning (see Chapter 7).

**The NDB Cluster engine**

MySQL AB acquired the NDB database from Sony Ericsson in 2003 and built the NDB
Cluster storage engine as an interface between the SQL used in MySQL and the native
NDB protocol. The combination of the MySQL server, the NDB Cluster storage engine,
and the distributed, shared-nothing, fault-tolerant, highly available NDB database is
known as MySQL Cluster. We discuss MySQL Cluster later in this book.

#### Third-Party Storage Engines

Because MySQL offers a pluggable storage engine API, beginning around 2007 a be-
wildering array of storage engines started springing up to serve special purposes. Some
of these were included with the server, but most were third-party products or open
source projects. We’ll discuss a few of the storage engines that we’ve observed to be
useful enough that they remain relevant even as the diversity has thinned out a bit.

```
MySQL’s Storage Engines | 21
```

**OLTP storage engines**

Percona’s XtraDB storage engine, which is included with Percona Server and MariaDB,
is a modified version of InnoDB. Its improvements are targeted at performance, meas-
urability, and operational flexibility. It is a drop-in replacement for InnoDB with the
ability to read and write InnoDB’s data files compatibly, and to execute all queries that
InnoDB can execute.

There are several other OLTP storage engines that are roughly similar to InnoDB in
some important ways, such as offering ACID compliance and MVCC. One is PBXT,
the creation of Paul McCullagh and Primebase GMBH. It sports engine-level replica-
tion, foreign key constraints, and an intricate architecture that positions it for solid-
state storage and efficient handling of large values such as BLOBs. PBXT is widely
regarded as a community storage engine and is included with MariaDB.

TokuDB uses a new index data structure called Fractal Trees, which are cache-
oblivious, so they don’t slow down as they get larger than memory, nor do they age or
fragment. TokuDB is marketed as a Big Data storage engine, because it has high com-
pression ratios and can support lots of indexes on large data volumes. At the time of
writing it is in early production release status, and has some important limitations
around concurrency. This makes it best suited for use cases such as analytical datasets
with high insertion rates, but that could change in future versions.

RethinkDB was originally positioned as a storage engine designed for solid-state
storage, although it seems to have become less niched as time has passed. Its most
distinctive technical characteristic could be said to be its use of an append-only copy-
on-write B-Tree index data structure. It is still in early development, and we’ve neither
evaluated it nor seen it in use.

Falcon was promoted as the next-generation transactional storage engine for MySQL
around the time of Sun’s acquisition of MySQL AB, but it has long since been canceled.
Jim Starkey, the primary architect of Falcon, has founded a new company to build a
cloud-enabled NewSQL database called NuoDB (formerly NimbusDB).

**Column-oriented storage engines**

MySQL is row-oriented by default, meaning that each row’s data is stored together,
and the server works in units of rows as it executes queries. But for very large volumes
of data, a column-oriented approach can be more efficient; it allows the engine to re-
trieve less data when full rows aren’t needed, and when each column is stored sepa-
rately, it can often be compressed more effectively.

The leading column-oriented storage engine is Infobright, which works well at very
large sizes (tens of terabytes). It is designed for analytical and data warehousing use
cases. It works by storing data in blocks, which are highly compressed. It maintains a
set of metadata for each block, which allows it to skip blocks or even to complete queries
simply by looking at the metadata. It has no indexes—that’s the point; at such huge

**22 | Chapter 1: MySQL Architecture and History**


sizes, indexes are useless, and the block structure is a kind of quasi-index. Infobright
requires a customized version of the server, because portions of the server have to be
rewritten to work with column-oriented data. Some queries can’t be executed by the
storage engine in column-oriented mode, and cause the server to fall back to row-by-
row mode, which is slow. Infobright is available in both open source–community and
proprietary commercial versions.

Another column-oriented storage engine is Calpont’s InfiniDB, which is also available
in commercial and community versions. InfiniDB offers the ability to distribute queries
across a cluster of machines. We haven’t seen anyone use it in production, though.

By the way, if you’re in the market for a column-oriented database that isn’t MySQL,
we’ve also evaluated LucidDB and MonetDB. You can find benchmarks and opinions
on the MySQL Performance Blog, although they will probably become somewhat out-
dated as time passes.

**Community storage engines**

A full list of community storage engines would run into the scores, and perhaps even
to triple digits if we researched them exhaustively. However, it’s safe to say that most
of them serve very limited niches, and many aren’t known or used by more than a few
people. We’ll just mention a few of them. We haven’t seen most of these in production
use. _Caveat emptor_!

_Aria_
Aria, formerly named Maria, is the original MySQL creator’s planned successor to
MyISAM. It’s available in MariaDB. Many of the features that were planned for it
seem to have been deferred in favor of improvements elsewhere in the MariaDB
server. At the time of writing it is probably best to describe it as a crash-safe version
of MyISAM, with several other improvements such as the ability to cache data (not
just indexes) in its own memory.

_Groonga_
This is a full-text search storage engine that claims to offer accuracy and high speed.

_OQGraph_
This engine from Open Query supports graph operations (think “find the shortest
path between nodes”) that are impractical or impossible to perform in SQL.

_Q4M_
This engine implements a queue inside MySQL, with support for operations that
SQL itself makes quite difficult or impossible to do in a single statement.

_SphinxSE_
This engine provides a SQL interface to the Sphinx full-text search server, which
we discuss more in Appendix F.

```
MySQL’s Storage Engines | 23
```

_Spider_
This engine partitions data into several partitions, effectively implementing trans-
parent sharding, and executes your queries in parallel across shards, which can be
located on different servers.

_VPForMySQL_
This engine supports vertical partitioning of tables through a sort of proxy storage
engine. That is, you can chop a table into several sets of columns and store those
independently, but query them as a single table. It’s by the same author as the
Spider engine.

#### Selecting the Right Engine

Which engine should you use? InnoDB is usually the right choice, which is why we’re
glad that Oracle made it the default engine in MySQL 5.5. The decision of which engine
to use can be summed up by saying, “Use InnoDB unless you need a feature it doesn’t
provide, and for which there is no good alternative approach.” For example, when we
need full-text search, we usually prefer to use InnoDB in combination with Sphinx,
rather than choosing MyISAM for its full-text indexing capabilities. Sometimes we
choose something other than InnoDB when we don’t need InnoDB’s features and an-
other engine provides a compelling benefit without downsides. For instance, we might
use MyISAM when its limited scalability, poor support for concurrency, and lack of
crash resilience aren’t an issue, but InnoDB’s increased space consumption is a
problem.

We prefer not to mix and match different storage engines unless absolutely needed. It
makes things much more complicated and exposes you to a whole new set of potential
bugs and edge-case behaviors. The interactions between the storage engines and the
server are complex enough without adding multiple storage engines into the mix. For
example, multiple storage engines make it difficult to perform consistent backups or
to configure the server properly.

If you believe that you do need a different engine, here are some factors you should
consider:

_Transactions_
If your application requires transactions, InnoDB (or XtraDB) is the most stable,
well-integrated, proven choice. MyISAM is a good choice if a task doesn’t require
transactions and issues primarily either SELECT or INSERT queries. Sometimes spe-
cific components of an application (such as logging) fall into this category.

_Backups_
The need to perform regular backups might also influence your choice. If your
server can be shut down at regular intervals for backups, the storage engines are
equally easy to deal with. However, if you need to perform online backups, you
basically need InnoDB.

**24 | Chapter 1: MySQL Architecture and History**


_Crash recovery_
If you have a lot of data, you should seriously consider how long it will take to
recover from a crash. MyISAM tables become corrupt more easily and take much
longer to recover than InnoDB tables. In fact, this is one of the most important
reasons why a lot of people use InnoDB when they don’t need transactions.

_Special features_
Finally, you sometimes find that an application relies on particular features or op-
timizations that only some of MySQL’s storage engines provide. For example, a
lot of applications rely on clustered index optimizations. On the other hand, only
MyISAM supports geospatial search inside MySQL. If a storage engine meets one
or more critical requirements, but not others, you need to either compromise or
find a clever design solution. You can often get what you need from a storage engine
that seemingly doesn’t support your requirements.

You don’t need to decide right now. There’s a lot of material on each storage engine’s
strengths and weaknesses in the rest of the book, and lots of architecture and design
tips as well. In general, there are probably more options than you realize yet, and it
might help to come back to this question after reading more. If you’re not sure, just
stick with InnoDB. It’s a safe default and there’s no reason to choose anything else if
you don’t know yet what you need.

These topics might seem rather abstract without some sort of real-world context, so
let’s consider some common database applications. We’ll look at a variety of tables and
determine which engine best matches with each table’s needs. We give a summary of
the options in the next section.

**Logging**

Suppose you want to use MySQL to log a record of every telephone call from a central
telephone switch in real time. Or maybe you’ve installed _mod_log_sql_ for Apache, so
you can log all visits to your website directly in a table. In such an application, speed
is probably the most important goal; you don’t want the database to be the bottleneck.
The MyISAM and Archive storage engines would work very well because they have
very low overhead and can insert thousands of records per second.

Things will get interesting, however, if you decide it’s time to start running reports to
summarize the data you’ve logged. Depending on the queries you use, there’s a good
chance that gathering data for the report will significantly slow the process of inserting
records. What can you do?

One solution is to use MySQL’s built-in replication feature to clone the data onto a
second server, and then run your time- and CPU-intensive queries against the data on
the replica. This leaves the master free to insert records and lets you run any query you
want on the replica without worrying about how it might affect the real-time logging.

```
MySQL’s Storage Engines | 25
```

You can also run queries at times of low load, but don’t rely on this strategy continuing
to work as your application grows.

Another option is to log to a table that contains the year and name or number of the
month in its name, such as web_logs_2012_01 or web_logs_2012_jan. While you’re busy
running queries against tables that are no longer being written to, your application can
log records to its current table uninterrupted.

**Read-only or read-mostly tables**

Tables that contain data used to construct a catalog or listing of some sort (jobs, auc-
tions, real estate, etc.) are usually read from far more often than they are written to.
This seemingly makes them good candidates for MyISAM— _if you don’t mind what
happens when MyISAM crashes_. Don’t underestimate how important this is; a lot of
users don’t really understand how risky it is to use a storage engine that doesn’t even
try to get their data written to disk. (MyISAM just writes the data to memory and
assumes the operating system will flush it to disk sometime later.)

```
It’s an excellent idea to run a realistic load simulation on a test server
and then literally pull the power plug. The firsthand experience of re-
covering from a crash is priceless. It saves nasty surprises later.
```
Don’t just believe the common “MyISAM is faster than InnoDB” folk wisdom. It is
_not_ categorically true. We can name dozens of situations where InnoDB leaves
MyISAM in the dust, especially for applications where clustered indexes are useful or
where the data fits in memory. As you read the rest of this book, you’ll get a sense of
which factors influence a storage engine’s performance (data size, number of I/O op-
erations required, primary keys versus secondary indexes, etc.), and which of them
matter to your application.

When we design systems such as these, we use InnoDB. MyISAM might seem to work
okay in the beginning, but it will absolutely fall on its face when the application gets
busy. Everything will lock up, and you’ll lose data when you have a server crash.

**Order processing**

When you deal with any sort of order processing, transactions are all but required.
Half-completed orders aren’t going to endear customers to your service. Another im-
portant consideration is whether the engine needs to support foreign key constraints.
InnoDB is your best bet for order-processing applications.

**26 | Chapter 1: MySQL Architecture and History**


**Bulletin boards and threaded discussion forums**

Threaded discussions are an interesting problem for MySQL users. There are hundreds
of freely available PHP and Perl-based systems that provide threaded discussions. Many
of them aren’t written with database efficiency in mind, so they tend to run a lot of
queries for each request they serve. Some were written to be database-independent, so
their queries do not take advantage of the features of any one database system. They
also tend to update counters and compile usage statistics about the various discussions.
Many of the systems also use a few monolithic tables to store all their data. As a result,
a few central tables become the focus of heavy read and write activity, and the locks
required to enforce consistency become a substantial source of contention.

Despite their design shortcomings, most of these systems work well for small and me-
dium loads. However, if a website grows large enough and generates significant traffic,
it will become very slow. The obvious solution is to switch to a different storage engine
that can handle the heavy read/write volume, but users who attempt this are sometimes
surprised to find that the system runs even more slowly than it did before!

What these users don’t realize is that the system is using a particular query, normally
something like this:

```
mysql> SELECT COUNT(*) FROM table;
```
The problem is that not all engines can run that query quickly: MyISAM can, but other
engines might not. There are similar examples for every engine. Later chapters will help
you keep such a situation from catching you by surprise and show you how to find and
fix the problems if it does.

**CD-ROM applications**

If you ever need to distribute a CD-ROM- or DVD-ROM-based application that uses
MySQL data files, consider using MyISAM or compressed MyISAM tables, which can
easily be isolated and copied to other media. Compressed MyISAM tables use far less
space than uncompressed ones, but they are read-only. This can be problematic in
certain applications, but because the data is going to be on read-only media anyway,
there’s little reason not to use compressed tables for this particular task.

**Large data volumes**

How big is too big? We’ve built and managed—or helped build and manage—many
InnoDB databases in the 3 TB to 5 TB range, or even larger. That’s on a single server,
not sharded. It’s perfectly feasible, although you have to choose your hardware wisely,
practice smart physical design, and plan for your server to be I/O-bound. At these sizes,
MyISAM is just a nightmare when it crashes.

If you’re going really big, such as tens of terabytes, you’re probably building a data
warehouse. In this case, Infobright is where we’ve seen the most success. Some very

```
MySQL’s Storage Engines | 27
```

large databases that aren’t suitable for Infobright might be candidates for TokuDB
instead.

#### Table Conversions

There are several ways to convert a table from one storage engine to another, each with
advantages and disadvantages. In the following sections, we cover three of the most
common ways.

**ALTER TABLE**

The easiest way to move a table from one engine to another is with an ALTER TABLE
statement. The following command converts mytable to InnoDB:

```
mysql> ALTER TABLE mytable ENGINE = InnoDB;
```
This syntax works for all storage engines, but there’s a catch: it can take a lot of time.
MySQL will perform a row-by-row copy of your old table into a new table. During that
time, you’ll probably be using all of the server’s disk I/O capacity, and the original table
will be read-locked while the conversion runs. So, take care before trying this technique
on a busy table. Instead, you can use one of the methods discussed next, which involve
making a copy of the table first.

When you convert from one storage engine to another, any storage engine–specific
features are lost. For example, if you convert an InnoDB table to MyISAM and back
again, you will lose any foreign keys originally defined on the InnoDB table.

**Dump and import**

To gain more control over the conversion process, you might choose to first dump the
table to a text file using the _mysqldump_ utility. Once you’ve dumped the table, you can
simply edit the dump file to adjust the CREATE TABLE statement it contains. Be sure to
change the table name as well as its type, because you can’t have two tables with the
same name in the same database even if they are of different types—and _mysqldump_
defaults to writing a DROP TABLE command before the CREATE TABLE, so you might lose
your data if you are not careful!

**CREATE and SELECT**

The third conversion technique is a compromise between the first mechanism’s speed
and the safety of the second. Rather than dumping the entire table or converting it all
at once, create the new table and use MySQL’s INSERT ... SELECT syntax to populate
it, as follows:

```
mysql> CREATE TABLE innodb_table LIKE myisam_table;
mysql> ALTER TABLE innodb_table ENGINE=InnoDB;
mysql> INSERT INTO innodb_table SELECT * FROM myisam_table;
```
**28 | Chapter 1: MySQL Architecture and History**


That works well if you don’t have much data, but if you do, it’s often more efficient to
populate the table incrementally, committing the transaction between each chunk so
the undo logs don’t grow huge. Assuming that id is the primary key, run this query
repeatedly (using larger values of x and y each time) until you’ve copied all the data to
the new table:

```
mysql> START TRANSACTION;
mysql> INSERT INTO innodb_table SELECT * FROM myisam_table
-> WHERE id BETWEEN x AND y;
mysql> COMMIT;
```
After doing so, you’ll be left with the original table, which you can drop when you’re
done with it, and the new table, which is now fully populated. Be careful to lock the
original table if needed to prevent getting an inconsistent copy of the data!

Tools such as Percona Toolkit’s _pt-online-schema-change_ (based on Facebook’s online
schema change technique) can remove the error-prone and tedious manual work from
schema changes.

### A MySQL Timeline

It is helpful to understand MySQL’s version history as a frame of reference when
choosing which version of the server you want to run. Plus, it’s kind of fun for old-
timers to remember what it used to be like in the good old days!

_Version 3.23 (2001)_
This release of MySQL is generally regarded as the moment MySQL “arrived” and
became a viable option for widespread use. MySQL was still not much more than
a query language over flat files, but MyISAM was introduced to replace ISAM, an
older and much more limited storage engine. InnoDB was available, but was not
shipped in the standard binary distribution because it was so new. If you wanted
to use InnoDB, you had to compile the server yourself with support for it. Version
3.23 introduced full-text indexing and replication. Replication was to become the
killer feature that propelled MySQL to fame as the database that powered much
of the Internet.

_Version 4.0 (2003)_
New syntax features appeared, such as support for UNION and multi-table DELETE
statements. Replication was rewritten to use two threads on the replica, instead of
one thread that did all the work and suffered from task switching. InnoDB was
shipped as a standard part of the server, with its full feature set: row-level locking,
foreign keys, and so on. The query cache was introduced in version 4.0 (and hasn’t
changed much since then). Support for SSL connections was also introduced.

_Version 4.1 (2005)_
More query syntax features were introduced, including subqueries and INSERT ON
DUPLICATE KEY UPDATE. The UTF-8 character set was supported. There was a new
binary protocol and prepared statement support.

```
A MySQL Timeline | 29
```

_Version 5.0 (2006)_
A number of “enterprise” features appeared in this release: views, triggers, stored
procedures, and stored functions. The ISAM engine was removed completely, but
new storage engines such as Federated were introduced.

_Version 5.1 (2008)_
This release was the first under Sun Microsystems’s ownership after its acquisition
of MySQL AB, and was over five years in the making. Version 5.1 introduced par-
titioning, row-based replication, and a variety of plugin APIs, including the
pluggable storage engine API. The BerkeleyDB storage engine—MySQL’s first
transactional storage engine—was removed and some others, such as Federated,
were deprecated. Also, Oracle, now the owner of Innobase Oy,^5 released the
InnoDB plugin storage engine.

_Version 5.5 (2010)_
MySQL 5.5 was the first release following Oracle’s acquisition of Sun (and there-
fore MySQL). It focused on improvements to performance, scalability, replication,
partitioning, and support for Microsoft Windows, but included many other im-
provements as well. InnoDB became the default storage engine, and many legacy
features and deprecated options and behaviors were scrubbed. The PERFORMANCE
_SCHEMA database was added, along with a first batch of enhanced instrumentation.
New plugin APIs for replication, authentication, and auditing were added. A plugin
for semisynchronous replication was available, and Oracle released commercial
plugins for authentication and thread pooling in 2011. There were also major ar-
chitectural changes to InnoDB, such as a partitioned buffer pool.

_Version 5.6 (Unreleased)_
MySQL 5.6 will have a raft of new features, including the first major improvements
to the query optimizer in many years, more plugin APIs (e.g., one for full-text
search), replication improvements, and greatly expanded instrumentation in the
PERFORMANCE_SCHEMA database. The InnoDB team is also hard at work, with a huge
variety of changes and improvements having been released in development mile-
stones and lab previews. Whereas MySQL 5.5 seemed to be about firming up and
fixing the fundamentals, with a limited number of new introductions, MySQL 5.6
appears to be focused on advancing server development and performance, using
5.5’s success as a springboard.

_Version 6.0 (Canceled)_
Version 6.0 is confusing because of the overlapping chronology. It was announced
during the 5.1 development years. There were rumors or promises of many new
features, such as online backups and server-level foreign keys for all storage en-
gines, subquery improvements, and thread pooling. This release was canceled, and
Sun resumed development with version 5.4, which was eventually released as

5. Oracle also now owns BerkeleyDB.

**30 | Chapter 1: MySQL Architecture and History**


```
version 5.5. Many of the features of the 6.0 codebase have been (or will be) released
in versions 5.5 and 5.6.
```
We’d summarize MySQL’s history this way: it was clearly a disruptive innovation^6 early
in its lifecycle, with limited and sometimes second-class functionality, but its features
and low price made it a killer application to power the explosion of the Internet. In the
early 5.x releases, it tried to move into enterprise territory with features such as views
and stored procedures, but these were buggy and brittle, so it wasn’t always smooth
sailing. In hindsight, MySQL 5.0’s flood of bug fixes didn’t settle down until around
the 5.0.50 releases, and MySQL 5.1 didn’t fare much better. The 5.0 and 5.1 releases
were delayed, and the Sun and Oracle acquisitions made many observers fearful. But
in our opinion, things are on track: MySQL 5.5 was the highest-quality release in
MySQL’s history, Oracle’s ownership is making MySQL much more palatable to en-
terprise customers, and version 5.6 promises great improvements in functionality and
performance.

Speaking of performance, we thought it would be interesting to show a basic bench-
mark of the server’s performance over time. We decided not to benchmark versions
older than 4.1, because it’s very rare to see 4.0 and older in production these days. In
addition, an apples-to-apples benchmark is very hard to produce across so many dif-
ferent versions, for reasons you’ll read more about in the next chapter. We had lots of
fun crafting a benchmark method that would work uniformly across the server versions
that we did use, and it took many tries to get it right. Table 1-2 shows the results in
transactions per second for several levels of concurrency.

_Table 1-2. Readonly benchmarks of several MySQL versions_

**Threads MySQL 4.1 MySQL 5.0 MySQL 5.1 MySQL 5.1 with InnoDB plugin MySQL 5.5 MySQL 5.6a**
1 686 640 596 594 531 526
2 1307 1221 1140 1139 1077 1019
4 2275 2168 2032 2043 1938 1831
8 3879 3746 3606 3681 3523 3320
16 4374 4527 4393 6131 5881 5573
32 4591 4864 4698 7762 7549 7139
64 4688 5078 4910 7536 7269 6994
aAt the time of our benchmark, MySQL 5.6 was not yet released as GA.

This is a little easier to see in graphical form, which we’ve shown in Figure 1-2.

Before we interpret the results, we need to tell you a little bit about the benchmark
itself. We ran it on our Cisco UCS C250 machine, which has two six-core CPUs, each
with two hardware threads. The server contains 384 GB of RAM, but we ran the

6. The term “disruptive innovation” originated in Clayton M. Christensen’s book _The Innovator’s_
    _Dilemma_ (Harper).

```
A MySQL Timeline | 31
```

benchmark with a 2.5 GB dataset, so we configured MySQL with a 4 GB buffer pool.
The benchmark was the standard SysBench read-only workload, with all data in
InnoDB, fully in-memory and CPU-bound. We ran the benchmark for 60 minutes for
each measurement point, measuring throughput every 10 seconds and using 900 sec-
onds of measurements after the server warmed up and stabilized to generate the final
results.

Now, looking at the results, two broad trends are clear. First, MySQL versions that
include the InnoDB plugin perform much better at higher concurrency, which is to say
that they are more scalable. This is to be expected, because we know older versions are
seriously limited at high concurrency. Second, newer MySQL versions are slower than
older versions in single-threaded workloads, which you might not have expected but
is easily explained by noting that this is a very simple read-only workload. Newer server
versions have a more complex SQL grammar, and lots of other features and improve-
ments that enable more complex queries but are simply additional overhead for the
simple queries we’re benchmarking here. Older versions of the server are simpler and
thus have an advantage for simple queries.

We wanted to show you a more complex read/write benchmark (such as TPC-C) over
a broader range of concurrencies, but we found it ultimately impossible to do across
such a diversity of server versions. We can say that in general, newer versions of the
server have better and more consistent performance on more complex workloads, es-
pecially at higher concurrency, and with a larger dataset.

_Figure 1-2. Readonly benchmarks of several MySQL versions_

**32 | Chapter 1: MySQL Architecture and History**


Which version should you use? This depends on your business more than on your
technical needs. You should ideally build on the newest version that’s available, but of
course you might choose to wait until the first bugs have been worked out of a brand-
new release. If you’re building an application that’s not in production yet, you might
even consider building it on the upcoming release so that you delay your upgrade life-
cycle as much as possible.

### MySQL’s Development Model

MySQL’s development process and release model have changed greatly over the years,
but now appear to have settled down into a steady rhythm. Oracle releases new
development milestones periodically, with previews of features that will eventually be
included in the next GA^7 release. These are for testing and feedback, not for production
use, but Oracle’s statement is that they’re high quality and essentially ready to release
at any time—and we see no reason to disagree with that. Oracle also periodically re-
leases lab previews, which are special builds that include only a selected feature for
interested parties to evaluate. These features are not guaranteed to be included in the
next release of the server. And finally, once in a while Oracle will bundle up the features
it deems to be ready and ship a new GA release of the server.

MySQL remains GPL-licensed and open source, with the full source code (except for
commercially licensed plugins, of course) available to the community. Oracle seems to
understand that it would be unwise to ship different versions of the server to the com-
munity and its paying customers. MySQL AB tried that, which resulted in its paying
customers becoming the bleeding-edge guinea pigs, robbing them of the benefit of
community testing and feedback. That policy was the reverse of what enterprise cus-
tomers need, and was discontinued in the Sun days.

Now that Oracle is releasing some server plugins for paying customers only, MySQL
is for all intents and purposes following the so-called open-core model. Although
there’s been some murmuring over the release of proprietary plugins for the server, it
comes from a minority and has sometimes been exaggerated. Most MySQL users we
know (and we know a lot of them) don’t seem to mind. The commercially licensed,
pay-only plugins are acceptable to those users who actually need them.

In any case, the proprietary extensions are just that: extensions. They do not represent
a crippleware development model, and the server is more than adequate without them.
Frankly, we appreciate the way that Oracle is building more features as plugins. If the
features were built right into the server with no API, there would be no choice: you’d
get exactly one implementation, with limited opportunity to build something that
suited you better. For example, if Oracle eventually releases InnoDB’s full-text search
functionality as a plugin, it will be an opportunity to use the same API to develop a
similar plugin for Sphinx or Lucene, which many people might find more useful. We

7. GA stands for generally available, which means “production quality” to pointy-haired bosses.

```
MySQL’s Development Model | 33
```

also appreciate clean APIs inside the server. They help to promote higher-quality code,
and who doesn’t want that?

### Summary

MySQL has a layered architecture, with server-wide services and query execution on
top and storage engines underneath. Although there are many different plugin APIs,
the storage engine API is the most important. If you understand that MySQL executes
queries by handing rows back and forth across the storage engine API, you’ve grasped
one of the core fundamentals of the server’s architecture.

MySQL was built around ISAM (and later MyISAM), and multiple storage engines and
transactions were added later. Many of the server’s quirks reflect this legacy. For ex-
ample, the way that MySQL commits transactions when you execute an ALTER TABLE
is a direct result of the storage engine architecture, as well as the fact that the data
dictionary is stored in _.frm_ files. (There’s nothing in InnoDB that forces an ALTER to be
nontransactional, by the way; absolutely everything InnoDB does is transactional.)

The storage engine API has its downsides. Sometimes choice isn’t a good thing, and
the explosion of storage engines in the heady days of the 5.0 and 5.1 versions of MySQL
might have introduced too much choice. In the end, InnoDB turns out to be a very good
storage engine for something like 95% or more of users (that’s just a rough guess). All
those other engines usually just make things more complicated and brittle, although
there are special cases where an alternative is definitely called for.

Oracle’s acquisition of first InnoDB and then MySQL brought both products under
one roof, where they can be codeveloped. This appears to be working out well for
everyone: InnoDB and the server itself are getting better by leaps and bounds in many
ways, MySQL remains GPL’ed and fully open source, the community and customers
alike are getting a solid and stable database, and the server is becoming ever more
extensible and useful.

**34 | Chapter 1: MySQL Architecture and History**
