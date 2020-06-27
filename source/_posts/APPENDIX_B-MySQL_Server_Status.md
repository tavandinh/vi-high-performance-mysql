
```
APPENDIX B
```
```
MySQL Server Status
```
You can answer many questions about a MySQL server by inspecting its status. MySQL
exposes information about server internals in several ways. The newest is the PERFOR
MANCE_SCHEMA database in MySQL 5.5, but the standard INFORMATION_SCHEMA database
has existed since MySQL 5.0, and there are a series of SHOW commands that have existed
practically forever. Some information you can get via SHOW commands isn’t found in the
INFORMATION_SCHEMA tables.

The challenges for you are determining what is relevant to your problem, how to get
the information you need, and how to interpret it. Although MySQL lets you see a lot
of information about what’s going on inside the server, it’s not always easy to use that
information. Understanding it requires patience, experience, and ready access to the
MySQL manual. Good tools are helpful, too.

This appendix is mostly reference material, but you will also find some information on
the functioning of server internals, especially in the sections on InnoDB.

### System Variables

MySQL exposes many system variables through the SHOW VARIABLES SQL command, as
variables you can use in expressions, or with _mysqladmin variables_ at the command
line. From MySQL 5.1, you can also access them through tables in the INFORMATION
_SCHEMA database.

These variables represent a variety of configuration information, such as the server’s
default storage engine (storage_engine), the available time zones, the connection’s col-
lation, and startup parameters. We explained how to set and use system variables in
Chapter 8.

```
685
```

### SHOW STATUS

The SHOW STATUS command shows server status variables in a two-column name-value
table. Unlike the server variables we mentioned in the previous section, these are read-
only. You can view the variables by either executing SHOW STATUS as a SQL command
or executing _mysqladmin extended-status_ as a shell command. If you use the SQL com-
mand, you can use LIKE and WHERE to limit the results; the LIKE does a standard pattern
match on the variable name. The commands return a table of results, but you can’t sort
it, join it to other tables, or do other standard things you can do with MySQL tables.
In MySQL 5.1 and newer, you can select values directly from the INFORMATION_
SCHEMA.GLOBAL_STATUS and INFORMATION_SCHEMA.SESSION_STATUS tables.

```
We use the term “status variable” to refer to a value from SHOW STATUS
and the term “system variable” to refer to a server configuration variable.
```
The behavior of SHOW STATUS changed greatly in MySQL 5.0, but you might not notice
unless you’re paying close attention. Instead of just maintaining one set of global vari-
ables, MySQL now maintains some variables globally and some on a per-connection
basis. Thus, SHOW STATUS contains a mixture of global and session variables. Many of
them have dual scope: there’s both a global and a session variable, and they have the
same name. SHOW STATUS also now shows session variables by default, so if you were
accustomed to running SHOW STATUS and seeing global variables, you won’t see them
anymore; now you have to run SHOW GLOBAL STATUS instead.^1

There are hundreds of status variables. Most either are counters or contain the current
value of some status metric. Counters increment every time MySQL does something,
such as initiating a full table scan (Select_scan). Metrics, such as the number of open
connections to the server (Threads_connected), may increase and decrease. Sometimes
there are several variables that seem to refer to the same thing, such as Connections (the
number of connection attempts to the server) and Threads_connected; in this case, the
variables are related, but similar names don’t always imply a relationship.

Counters are stored as unsigned integers. They use 4 bytes on 32-bit builds and 8 bytes
on 64-bit builds, and they wrap back to 0 after reaching their maximum values. If you’re
monitoring the variables incrementally, you might need to watch for and correct the
wrap; you should also be aware that if your server has been up for a long time, you
might see lower values than you expect simply because the variable’s values have wrap-
ped around to zero. (This is very rarely a problem on 64-bit builds.)

1. There’s a gotcha waiting here: if you use an old version of _mysqladmin_ on a new server, it won’t use SHOW
    GLOBAL STATUS, so it won’t display the “right” information.

**686 | Appendix B: MySQL Server Status**


A good way to get a feel for your overall workload is to compare values within a group
of related status variables—for example, look at all the Select_* variables together, or
all the Handler_* variables. If you’re using _innotop_ , this is easy to do in Command
Summary mode, but you can also do it manually with a command like _mysqladmin
extended -r -i60 | grep Handler__. Here’s what _innotop_ shows for the Select_* variables
on one server we checked:

```
____________________ Command Summary _____________________
Name Value Pct Last Incr Pct
Select_scan 756582 59.89% 2 100.00%
Select_range 497675 39.40% 0 0.00%
Select_full_join 7847 0.62% 0 0.00%
Select_full_range_join 1159 0.09% 0 0.00%
Select_range_check 1 0.00% 0 0.00%
```
The first two columns of values are since the server was booted, and the last two are
since the last refresh (10 seconds ago, in this case). The percentages are over the total
of the values shown in the display, not over the total of all queries.

For a side-by-side view of current and previous snapshots and the differences between
them, you can also use the _pt-mext_ tool from Percona Toolkit, or this clever query from
Shlomi Noach:^2

```
SELECT STRAIGHT_JOIN
LOWER(gs0.VARIABLE_NAME) AS variable_name,
gs0.VARIABLE_VALUE AS value_0,
gs1.VARIABLE_VALUE AS value_1,
(gs1.VARIABLE_VALUE - gs0.VARIABLE_VALUE) AS diff,
(gs1.VARIABLE_VALUE - gs0.VARIABLE_VALUE) / 10 AS per_sec,
(gs1.VARIABLE_VALUE - gs0.VARIABLE_VALUE) * 60 / 10 AS per_min
FROM (
SELECT VARIABLE_NAME, VARIABLE_VALUE
FROM INFORMATION_SCHEMA.GLOBAL_STATUS
UNION ALL
SELECT '', SLEEP(10) FROM DUAL
) AS gs0
JOIN INFORMATION_SCHEMA.GLOBAL_STATUS gs1 USING (VARIABLE_NAME)
WHERE gs1.VARIABLE_VALUE <> gs0.VARIABLE_VALUE;
+-----------------------+---------+---------+------+---------+---------+
| variable_name | value_0 | value_1 | diff | per_sec | per_min |
+-----------------------+---------+---------+------+---------+---------+
| handler_read_rnd_next | 2366 | 2953 | 587 | 58.7 | 3522 |
| handler_write | 2340 | 3218 | 878 | 87.8 | 5268 |
| open_files | 22 | 20 | −2 | −0.2 | −12 |
| select_full_join | 2 | 3 | 1 | 0.1 | 6 |
| select_scan | 7 | 9 | 2 | 0.2 | 12 |
+-----------------------+---------+---------+------+---------+---------+
```
2. First published at _[http://code.openark.org/blog/mysql/mysql-global-status-difference-using-single-query](http://code.openark.org/blog/mysql/mysql-global-status-difference-using-single-query)_.

```
SHOW STATUS | 687
```

It’s most useful to look at the values of all these variables and metrics over the course
of the last several minutes, as well as over the entire uptime of the server.

The following is an overview—not an exhaustive list—of the different categories of
variables you’ll see in SHOW STATUS. For full details on a given variable, you should
consult the MySQL manual, which helpfully documents them at _[http://dev.mysql.com/](http://dev.mysql.com/)
doc/en/mysqld-option-tables.html_. When we discuss a set of related variables whose
name begins with a common prefix, we refer to the group collectively as “the _<pre
fix>_*_ variables.”

#### Thread and Connection Statistics

These variables track connection attempts, aborted connections, network traffic, and
thread statistics:

- Connections, Max_used_connections, Threads_connected
- Aborted_clients, Aborted_connects
- Bytes_received, Bytes_sent
- Slow_launch_threads, Threads_cached, Threads_created, Threads_running

If Aborted_connects isn’t zero, it might mean that you have network problems or that
someone is trying to connect and failing (perhaps a user is specifying the wrong pass-
word or an invalid database, or maybe a monitoring system is opening TCP port 3306
to check if the server is alive). If this value gets too high, it can have serious side effects:
it can cause MySQL to block a host.

Aborted_clients has a similar name but a completely different meaning. If this value
increments, it usually means there’s been an application error, such as the programmer
forgetting to close MySQL connections properly before terminating the program. This
is not usually indicative of a big problem.

#### Binary Logging Status

The Binlog_cache_use and Binlog_cache_disk_use status variables show how many
transactions have been stored in the binary log cache, and how many transactions were
too large for the binary log cache and so had their statements stored in a temporary file.
MySQL 5.5 also includes Binlog_stmt_cache_use and Binlog_stmt_cache_disk_use,
which show corresponding metrics for nontransactional statements. The so-called
“binary log cache hit ratio” is not usually useful for configuring the binary log cache
size. See Chapter 8 for more on this topic.

**688 | Appendix B: MySQL Server Status**


#### Command Counters

The Com_* variables count the number of times each type of SQL or C API command
has been issued. For example, Com_select counts the number of SELECT statements, and
Com_change_db counts the number of times a connection’s default database has been
changed, either with the USE statement or via a C API call. The Questions variable^3
counts the total number of queries and commands the server has received. However,
it doesn’t quite equal the sum of all the Com_* variables, because of query cache hits,
closed and aborted connections, and possibly other factors.

The Com_admin_commands status variable might be very large. It counts not only admin-
istrative commands, but ping requests to the MySQL instance as well. These requests
are issued through the C API and typically come from client code, such as the following
Perl code:

```
my $dbh = DBI->connect(...);
while ( $dbh && $dbh->ping ) {
# Do something
}
```
These ping requests are “garbage” queries. They usually don’t load the server very
much, but they’re still a waste and contribute a lot to application response time because
of the network round trip time. We’ve seen ORM systems (Ruby on Rails comes to
mind) that ping the server before each query, which is pointless; pinging the server and
then querying it is a classic example of the “look before you leap” design pattern, which
creates a race condition. We’ve also seen database abstraction libraries that change the
default database before every query, which will show up as a very large number of
Com_change_db commands. It’s best to eliminate both practices.

#### Temporary Files and Tables

You can view the variables that count how many times MySQL has created temporary
tables and files with:

```
mysql> SHOW GLOBAL STATUS LIKE 'Created_tmp%';
```
This shows statistics about implicit temporary tables and files—those created internally
to execute queries. In Percona Server, there is also a command that can show explicit
temporary tables, which are created by users with CREATE TEMPORARY TABLE:

```
mysql> SHOW GLOBAL TEMPORARY TABLES ;
```
3. In MySQL 5.1, this variable was split into Questions and Queries, with slightly different meanings.

```
SHOW STATUS | 689
```

#### Handler Operations

The handler API is the interface between MySQL and its storage engines. The Hand
ler_* variables count handler operations, such as the number of times MySQL asks a
storage engine to read the next row from an index. You can view these variables with:

```
mysql> SHOW GLOBAL STATUS LIKE 'Handler_%';
```
#### MyISAM Key Buffer

The Key_* variables contain metrics and counters about the MyISAM key buffer. You
can view these variables with:

```
mysql> SHOW GLOBAL STATUS LIKE 'Key_%';
```
#### File Descriptors

If you mainly use the MyISAM storage engine the Open_* variables reveal how often
MySQL opens each table’s _.frm, .MYI_ , and _.MYD_ files. InnoDB keeps all data in its
tablespace files, so if you mainly use InnoDB, these variables aren’t accurate. You can
view the Open_* variables with:

```
mysql> SHOW GLOBAL STATUS LIKE 'Open_%';
```
#### Query Cache

You can inspect the query cache by looking at the Qcache_* status variables, with:

```
mysql> SHOW GLOBAL STATUS LIKE 'Qcache_%';
```
#### SELECT Types

The Select_* variables are counters for certain types of SELECT queries. They can help
you see the ratio of SELECT queries that use various query plans. Unfortunately, there
are no such status variables for other kinds of queries, such as UPDATE and REPLACE;
however, you can look at the Handler_* status variables (discussed earlier) for insight
into the relative numbers of non-SELECT queries. To see all the Select_* variables, use:

```
mysql> SHOW GLOBAL STATUS LIKE 'Select_%';
```
In our judgment, the Select_* status variables can be ranked as follows, in order of
ascending cost:

Select_range
The number of joins that scanned an index range on the first table.

Select_scan
The number of joins that scanned the entire first table. There is nothing wrong
with this if every row in the first table should participate in the join; it’s only a bad
thing if you don’t want all the rows and there is no index to find the ones you want.

**690 | Appendix B: MySQL Server Status**


Select_full_range_join
The number of joins that used a value from table _n_ to retrieve rows from a range
of the reference index in table _n_ + 1. Depending on the query, this can be more or
less costly than Select_scan.

Select_range_check
The number of joins that reevaluate indexes in table _n_ + 1 for every row in table
_n_ to see which is least expensive. This generally means no indexes in table _n_ + 1 are
useful for the join. This query plan has very high overhead.

Select_full_join
The number of cross joins, or joins without any criteria to match rows in the tables.
The number of rows examined is the product of the number of rows in each table.
This is usually a very bad thing.

The last two variables usually should not increase rapidly, and if they do, it might be
an indication that a “bad” query has been introduced into the system. See Chapter 3
for details on how to find such queries.

#### Sorts

We covered a lot of MySQL’s sorting optimizations in several previous chapters, so you
should have a good idea of how sorting works. When MySQL can’t use an index to
retrieve rows presorted, it has to do a filesort, and it increments the Sort_* status vari-
ables. Aside from Sort_merge_passes, you can influence these values only by adding
indexes that MySQL can use for sorting. Sort_merge_passes depends on the sort_
buffer_size server variable (not to be confused with the myisam_sort
_buffer_size server variable). MySQL uses the sort buffer to hold a chunk of rows for
sorting. When it’s finished sorting them, it merges these sorted rows into the result,
increments Sort_merge_passes, and fills the buffer with the next chunk of rows to sort.
However, it’s not a great idea to use this variable as a guide to sort buffer sizing, as
shown in Chapter 3.

You can see all the Sort_* variables with:

```
mysql> SHOW GLOBAL STATUS LIKE 'Sort_%';
```
MySQL increments the Sort_scan and Sort_range variables when it reads sorted rows
from the results of a filesort and returns them to the client. The difference is merely
that the first is incremented when the query plan causes Select_scan to increment (see
the preceding section), and the second is incremented when Select_range increments.
There is no implementation or cost difference between the two; they merely indicate
the type of query plan that caused the sort.

```
SHOW STATUS | 691
```

#### Table Locking

The Table_locks_immediate and Table_locks_waited variables tell you how many locks
were granted immediately and how many had to be waited for. Be aware, however, that
they show only server-level locking statistics, not storage engine locking statistics.

#### InnoDB-Specific

The Innodb_* variables show some of the data included in SHOW ENGINE INNODB STA
TUS, discussed later in this appendix. The variables can be grouped together by name:
Innodb_buffer_pool_*, Innodb_log_*, and so on. We discuss InnoDB’s internals more
in a moment, when we examine SHOW ENGINE INNODB STATUS.

These variables are available in MySQL 5.0 and newer, and they have an important side
effect: they create a global lock and traverse the entire InnoDB buffer pool before re-
leasing the lock. In the meantime, other threads will run into the lock and block until
it is released. This skews some status values, such as Threads_running, so they will
appear higher than normal (possibly much higher, depending on how busy your server
is). The same effect happens when you run SHOW ENGINE INNODB STATUS or access these
statistics via the INFORMATION_SCHEMA tables (in MySQL 5.0 and newer, SHOW STATUS and
SHOW VARIABLES are mapped to queries against the INFORMATION_SCHEMA tables behind
the scenes).

These operations can, therefore, be expensive in these versions of MySQL—checking
the server status too frequently (e.g., once a second) can cause significant overhead.
Using SHOW STATUS LIKE doesn’t help, because it retrieves the full status and then post-
filters it.

There are many more variables in MySQL 5.5 than in 5.1, and even more in Percona
Server.

#### Plugin-Specific

MySQL 5.1 and newer support pluggable storage engines and provide a mechanism
for storage engines to register their own status and configuration variables with the
MySQL server. You might see some plugin-specific variables if you’re using a pluggable
storage engine. Such variables always begin with the name of the plugin.

### SHOW ENGINE INNODB STATUS

The InnoDB storage engine exposes a lot of information about its internals in the output
of SHOW ENGINE INNODB STATUS, or its older synonym, SHOW INNODB STATUS.

Unlike most of the SHOW commands, its output consists of a single string, not rows and
columns. It is divided into sections, each of which shows information about a different
part of the InnoDB storage engine. Some of the output is most useful for InnoDB

**692 | Appendix B: MySQL Server Status**


developers, but much of it is interesting—or even essential—if you’re trying to under-
stand and configure InnoDB for high performance.

```
Older versions of InnoDB often print out 64-bit numbers in two pieces:
the high 32 bits and the low 32 bits. An example is a transaction ID,
such as TRANSACTION 0 3793469. You can calculate the 64-bit number’s
value by shifting the first number left 32 bits and adding it to the second
one. We show some examples later.
```
The output includes some average statistics, such as fsync() calls per second. These
show average activity since the last time the output was generated, so if you’re exam-
ining these values, make sure you wait 30 seconds or so between samples to give the
statistics time to accumulate, and sample multiple times and examine the changes to
the counters to understand their behaviors. The output is not all generated at a single
point in time, so not all averages that appear in the output are calculated over the same
time interval. Also, InnoDB has an internal reset interval that is unpredictable and varies
between versions; you should examine the output to see the time over which the aver-
ages were generated, because it will not necessarily be the same as the time between
samples.

There’s enough information in the output to calculate averages for most of the statistics
manually if you want. However, a tool such as _innotop_ —which does incremental dif-
ferences and averages for you—is very helpful here.

#### Header

The first section is the header, which simply announces the beginning of the output,
the current date and time, and how long it has been since the last printout. Line 2 shows
the current date and time. Line 4 shows the time frame over which the averages were
calculated, which is either the time since the last printout or the time since the last
internal reset:

```
1 =====================================
2 070913 10:31:48 INNODB MONITOR OUTPUT
3 =====================================
4 Per second averages calculated from the last 49 seconds
```
#### SEMAPHORES

If you have a high-concurrency workload, you might want to pay attention to the next
section, SEMAPHORES. It contains two kinds of data: event counters and, optionally, a list
of current waits. If you’re having trouble with bottlenecks, you can use this information
to help you find the bottlenecks. Unfortunately, knowing what to do about them is a
little more complex, but we give some advice later in this appendix. Here is some sample
output for this section:

```
SHOW ENGINE INNODB STATUS | 693
```

```
1 ----------
2 SEMAPHORES
3 ----------
4 OS WAIT ARRAY INFO: reservation count 13569, signal count 11421
5 --Thread 1152170336 has waited at ./../include/buf0buf.ic line 630 for 0.00 seconds
the semaphore:
6 Mutex at 0x2a957858b8 created file buf0buf.c line 517, lock var 0
7 waiters flag 0
8 wait is ending
9 --Thread 1147709792 has waited at ./../include/buf0buf.ic line 630 for 0.00 seconds
the semaphore:
10 Mutex at 0x2a957858b8 created file buf0buf.c line 517, lock var 0
11 waiters flag 0
12 wait is ending
13 Mutex spin waits 5672442, rounds 3899888, OS waits 4719
14 RW-shared spins 5920, OS waits 2918; RW-excl spins 3463, OS waits 3163
```
Line 4 gives information about the operating system wait array, which is an array of
“slots.” InnoDB reserves slots in the array for semaphores, which the operating system
uses to signal threads that they can go ahead and do the work they’re waiting to do.
This line shows how many times InnoDB has needed to use operating system waits.
The reservation count indicates how often InnoDB has allocated slots, and the signal
count measures how often threads have been signaled via the array. Operating system
waits are costly relative to spin waits, as we’ll see momentarily.

Lines 5 through 12 show the InnoDB threads that are currently waiting for a mutex.
The example shows two waits, each beginning with the text “-- Thread <num> has
waited....” This section should be empty unless your server has a high-concurrency
workload that causes InnoDB to resort to operating system waits. The most useful thing
to look at, unless you’re familiar with InnoDB source code, is the filename at which the
thread is waiting. This gives you a hint where the hot spots are inside InnoDB. For
example, if you see many threads waiting at a file called _buf0buf.ic_ , you have buffer
pool contention. The output indicates how long the thread has been waiting, and the
“waiters flag” shows how many waiters are waiting for the mutex.

The text “wait is ending” means the mutex is actually free already, but the operating
system hasn’t scheduled the thread to run yet.

You might wonder what exactly InnoDB is waiting for. InnoDB uses mutexes and
semaphores to protect critical sections of code by restricting them to only one thread
at a time, or to restrict writers when there are active readers, and so on. There are many
critical sections in InnoDB’s code, and under the right conditions any of them could
appear here. Gaining access to a buffer pool page is one you might see commonly.

After the list of waiting threads, lines 13 and 14 show more event counters. Line 13
shows several counters related to mutexes, and line 14 is for read/write shared and
exclusive locks. In each case, you can see how often InnoDB has resorted to an operating
system wait.

**694 | Appendix B: MySQL Server Status**


InnoDB has a multiphase wait policy. First, it tries to spin-wait for the lock. If this
doesn’t succeed after a preconfigured number of spin rounds (specified by the innodb
_sync_spin_loops configuration variable), it falls back to the more expensive and com-
plex wait array.^4

Spin waits are relatively low-cost, but they burn CPU cycles by checking repeatedly if
a resource can be locked. This isn’t as bad as it sounds, because there are typically free
CPU cycles while the processor is waiting for I/O. And even if there aren’t any free CPU
cycles, spin waits are often much less expensive than the alternative. However, spinning
monopolizes the processor when another thread might be able to do some work.

The alternative to a spin wait is for the operating system to do a context switch, so
another thread can run while the thread waits, then wake the sleeping thread when it
is signaled via the semaphore in the wait array. Signaling via a semaphore is efficient,
but the context switch is expensive. These can add up quickly: thousands of them per
second can cause a lot of overhead.

You can try to strike a balance between spin waits and operating system waits by
changing the innodb_sync_spin_loops system variable. Don’t worry about spin waits
unless you see many (perhaps in the range of hundreds of thousands) spin rounds per
second. This is something you usually need to resolve by understanding the source code
involved, or by consulting with experts. You can also consider using the Performance
Schema, or look at SHOW ENGINE INNODB MUTEX.

#### LATEST FOREIGN KEY ERROR

The next section, LATEST FOREIGN KEY ERROR, doesn’t appear unless your server has had
a foreign key error. There are many places in the source code that can generate this
output, depending on the kind of error. Sometimes the problem is to do with a trans-
action and the parent or child rows it was looking for while trying to insert, update, or
delete a record. At other times it’s a type mismatch between tables while InnoDB was
trying to add or delete a foreign key, or alter a table that already had a foreign key.

This section’s output is very helpful for debugging the exact causes of InnoDB’s often
vague foreign key errors. Let’s look at some examples. First, we’ll create two tables with
a foreign key between them, and insert a little data:

```
CREATE TABLE parent (
parent_id int NOT NULL,
PRIMARY KEY(parent_id)
) ENGINE=InnoDB;
CREATE TABLE child (
parent_id int NOT NULL,
KEY parent_id (parent_id),
CONSTRAINT child_ibfk_1 FOREIGN KEY (parent_id) REFERENCES parent (parent_id)
) ENGINE=InnoDB;
```
4. The wait array was changed to be much more efficient in MySQL 5.1.

```
SHOW ENGINE INNODB STATUS | 695
```

```
INSERT INTO parent(parent_id) VALUES(1);
INSERT INTO child(parent_id) VALUES(1);
```
There are two basic classes of foreign key errors. Adding, updating, or deleting data in
a way that would violate the foreign key causes the first class of errors. For example,
here’s what happens when we delete the row from the parent table:

```
DELETE FROM parent;
ERROR 1451 (23000): Cannot delete or update a parent row: a foreign key constraint
fails (`test/child`, CONSTRAINT `child_ibfk_1` FOREIGN KEY (`parent_id`) REFERENCES
`parent` (`parent_id`))
```
The error message is fairly straightforward, and you’ll get similar messages for all errors
caused by adding, updating, or deleting nonmatching rows. Here’s the output from
SHOW ENGINE INNODB STATUS:

```
1 ------------------------
2 LATEST FOREIGN KEY ERROR
3 ------------------------
4 070913 10:57:34 Transaction:
5 TRANSACTION 0 3793469, ACTIVE 0 sec, process no 5488, OS thread id 1141152064
updating or deleting, thread declared inside InnoDB 499
6 mysql tables in use 1, locked 1
7 4 lock struct(s), heap size 1216, undo log entries 1
8 MySQL thread id 9, query id 305 localhost baron updating
9 DELETE FROM parent
10 Foreign key constraint fails for table `test/child`:
11 '
12 CONSTRAINT `child_ibfk_1` FOREIGN KEY (`parent_id`) REFERENCES `parent` (`parent_
id`)
13 Trying to delete or update in parent table, in index `PRIMARY` tuple:
14 DATA TUPLE: 3 fields;
15 0: len 4; hex 80000001; asc ;; 1: len 6; hex 00000039e23d; asc 9 =;; 2: len
7; hex 000000002d0e24; asc - $;;
16
17 But in child table `test/child`, in index `parent_id`, there is a record:
18 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
19 0: len 4; hex 80000001; asc ;; 1: len 6; hex 000000000500; asc ;;
```
Line 4 shows the date and time of the last foreign key error. Lines 5 through 9 show
details about the transaction that violated the foreign key constraint; we explain more
about these lines later. Lines 10 through 19 show the exact data InnoDB was trying to
change when it found the error. A lot of this output is the row data converted to print-
able formats; we say more about this later, too.

So far, so good, but there’s another class of foreign key error that can be much harder
to debug. Here’s what happens when we try to alter the parent table:

```
ALTER TABLE parent MODIFY parent_id INT UNSIGNED NOT NULL;
ERROR 1025 (HY000): Error on rename of './test/#sql-1570_9' to './test/parent'
(errno: 150)
```
This is less than clear, but the SHOW ENGINE INNODB STATUS text sheds some light on it:

**696 | Appendix B: MySQL Server Status**


```
1 ------------------------
2 LATEST FOREIGN KEY ERROR
3 ------------------------
4 070913 11:06:03 Error in foreign key constraint of table test/child:
5 there is no index in referenced table which would contain
6 the columns as the first columns, or the data types in the
7 referenced table do not match to the ones in table. Constraint:
8 ,
9 CONSTRAINT child_ibfk_1 FOREIGN KEY (parent_id) REFERENCES parent (parent_id)
10 The index in the foreign key in table is parent_id
11 See http://dev.mysql.com/doc/refman/5.0/en/innodb-foreign-key-constraints.html
12 for correct foreign key definition.
```
The error in this case is a different data type. Foreign-keyed columns must have _ex-
actly_ the same data type, including any modifiers (such as UNSIGNED, which was the
problem in this case). Whenever you see error 1025 and don’t understand why, the
best place to look is in SHOW ENGINE INNODB STATUS.

The foreign key error messages are overwritten every time there’s a new error. The
_pt-fk-error-logger_ tool from Percona Toolkit can help you save these for later analysis.

#### LATEST DETECTED DEADLOCK

Like the foreign key section, the LATEST DETECTED DEADLOCK section appears only if your
server has had a deadlock. The deadlock error messages are also overwritten every time
there’s a new error, and the _pt-deadlock -logger_ tool from Percona Toolkit can help you
save these for later analysis.

A deadlock is a cycle in the waits-for graph, which is a data structure of row locks held
and waited for. The cycle can be arbitrarily large. InnoDB detects deadlocks instantly,
because it checks for a cycle in the graph every time a transaction has to wait for a row
lock. Deadlocks can be quite complex, but this section shows only the last two trans-
actions involved, the last statement executed in each of the transactions, and the locks
that created the cycle in the graph. You don’t see other transactions that might also be
included in the cycle, nor do you see the statement that might have really acquired the
locks earlier in a transaction. Nevertheless, you can often find out what caused the
deadlock by looking at this output.

There are actually two types of InnoDB deadlocks. The first, which is what most people
are accustomed to, is a true cycle in the waits-for graph. The other type is a waits-for
graph that is too expensive to check for cycles. If InnoDB has to check more than a
million locks in the graph, or if it recurses through more than 200 transactions while
checking, it gives up and says there’s a deadlock. These numbers are hardcoded con-
stants in the InnoDB source, and you can’t configure them (though you can change
them and recompile InnoDB if you wish). You’ll know when exceeding these limits
causes a deadlock, because you’ll see “TOO DEEP OR LONG SEARCH IN THE
LOCK TABLE WAITS-FOR GRAPH” in the output.

```
SHOW ENGINE INNODB STATUS | 697
```

InnoDB prints not only the transactions and the locks they held and waited for, but
also the records themselves. This information is mostly useful to the InnoDB develop-
ers, but there’s currently no way to disable it. Unfortunately, it can be so large that it
runs over the length allocated for output and prevents you from seeing the sections that
follow. The only way to remedy this is to cause a small deadlock to replace the large
one, or to use Percona Server, which adds configuration variables to suppress the overly
verbose text.

Here’s a sample deadlock:

```
1 ------------------------
2 LATEST DETECTED DEADLOCK
3 ------------------------
4 070913 11:14:21
5 *** (1) TRANSACTION:
6 TRANSACTION 0 3793488, ACTIVE 2 sec, process no 5488, OS thread id 1141287232
starting index read
7 mysql tables in use 1, locked 1
8 LOCK WAIT 4 lock struct(s), heap size 1216
9 MySQL thread id 11, query id 350 localhost baron Updating
10 UPDATE test.tiny_dl SET a = 0 WHERE a <> 0
11 *** (1) WAITING FOR THIS LOCK TO BE GRANTED:
12 RECORD LOCKS space id 0 page no 3662 n bits 72 index `GEN_CLUST_INDEX` of table
`test/tiny_dl` trx id 0 3793488 lock_mode X waiting
13 Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
14 0: len 6; hex 000000000501 ...[ omitted ] ...
15
16 *** (2) TRANSACTION:
17 TRANSACTION 0 3793489, ACTIVE 2 sec, process no 5488, OS thread id 1141422400
starting index read, thread declared inside InnoDB 500
18 mysql tables in use 1, locked 1
19 4 lock struct(s), heap size 1216
20 MySQL thread id 12, query id 351 localhost baron Updating
21 UPDATE test.tiny_dl SET a = 1 WHERE a <> 1
22 *** (2) HOLDS THE LOCK(S):
23 RECORD LOCKS space id 0 page no 3662 n bits 72 index `GEN_CLUST_INDEX` of table
`test/tiny_dl` trx id 0 3793489 lock mode S
24 Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
25 0: ... [ omitted ] ...
26
27 *** (2) WAITING FOR THIS LOCK TO BE GRANTED:
28 RECORD LOCKS space id 0 page no 3662 n bits 72 index `GEN_CLUST_INDEX` of table
`test/tiny_dl` trx id 0 3793489 lock_mode X waiting
29 Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
30 0: len 6; hex 000000000501 ...[ omitted ] ...
31
32 *** WE ROLL BACK TRANSACTION (2)
```
Line 4 shows when the deadlock occurred, and lines 5 through 10 show information
about the first transaction involved in the deadlock. We explain the meaning of this
output in detail in the next section.

**698 | Appendix B: MySQL Server Status**


Lines 11 through 15 show the locks transaction 1 was waiting for when the deadlock
happened. We’ve omitted some of the information that’s useful only for debugging
InnoDB on line 14. The important thing to notice is line 12, which says this transaction
wanted an exclusive (X) lock on GEN_CLUST_INDEX^5 on the test.tiny_dl table.

Lines 16 through 21 show the second transaction’s status, and lines 22 through 26 show
the locks it held. There are several records listed on line 25, which we’ve removed for
brevity. One of these was the record for which the first transaction was waiting. Finally,
lines 27 through 31 show the locks for which it was waiting.

A cycle in the waits-for graph occurs when each transaction holds a lock the other wants
and wants a lock the other holds. InnoDB doesn’t show all the locks held and waited
for, but it often shows enough to help you determine what indexes the queries were
using, which is valuable in determining whether you can avoid deadlocks.

If you can get both queries to scan the same index in the same direction, you can often
reduce the number of deadlocks, because queries can’t create a cycle when they request
locks in the same order. This is sometimes easy to do. For example, if you need to
update a number of records within a transaction, sort them by their primary key in the
application’s memory, then update them in that order—then they can’t deadlock. At
other times, however, it can be infeasible (such as when you have two processes that
need to work on the same table but are using different indexes).

Line 32 shows which transaction was chosen as the deadlock victim. InnoDB tries to
choose the transaction it thinks will be easiest to roll back, which is the one with the
fewest updates.

It’s very helpful to examine the general log, find all the queries from the threads in-
volved, and see what really caused the deadlock. Read the next section to see where to
find the thread ID in the deadlock output.

#### TRANSACTIONS

This section contains a little summary information about InnoDB transactions, fol-
lowed by a list of the currently active transactions. Here are the first few lines (the
header):

```
1 ------------
2 TRANSACTIONS
3 ------------
4 Trx id counter 0 80157601
5 Purge done for trx's n:o <0 80154573 undo n:o <0 0
6 History list length 6
7 Total number of lock structs in row lock hash table 0
```
The output varies depending on the MySQL version, but it includes at least the
following:

5. This is the index InnoDB creates internally when you don’t specify a primary key.

```
SHOW ENGINE INNODB STATUS | 699
```

- Line 4: the current transaction identifier, which is a system variable that increments
    for each new transaction.
- Line 5: the transaction ID to which InnoDB has purged old MVCC row versions.
    You can see how many old versions haven’t yet been purged by looking at the
    difference between this value and the current transaction ID. There’s no hard and
    fast rule as to how large this number can safely get. If nothing is updating any data,
    a large number doesn’t mean there’s unpurged data, because all the transactions
    are actually looking at the same version of the database. On the other hand, if many
    rows are being updated, one or more versions of each row is staying in memory.
    The best policy for reducing overhead is to ensure that transactions commit when
    they’re done instead of staying open a long time, because even an open transaction
    that doesn’t do any work keeps InnoDB from purging old row versions.
    Also in line 5: the undo log record number InnoDB’s purge process is currently
    working on, if any. If it’s “0 0”, as in our example, the purge process is idle.
- Line 6: the history list length, which is the number of pages in the undo space in
    InnoDB’s data files. When a transaction performs updates and commits, this num-
    ber increases; when the purge process removes the old versions, it decreases. The
    purge process also updates the value in line 5.
- Line 7: the number of lock structs. Each lock struct usually holds many row locks,
    so this is not the same as the number of rows locked.

The header is followed by a list of transactions. Current versions of MySQL don’t
support nested transactions, so there’s a maximum of one transaction per client con-
nection at a time, and each transaction belongs to only a single connection. Each trans-
action has at least two lines in the output. Here’s a sample of the minimal information
you’ll see about a transaction:

```
1 ---TRANSACTION 0 3793494, not started, process no 5488, OS thread id 1141152064
2 MySQL thread id 15, query id 479 localhost baron
```
The first line begins with the transaction’s ID and status. This transaction is “not
started,” which means it has committed and not issued any more statements that affect
transactions; it’s probably just idle. Then there’s some process and thread information.
The second line shows the MySQL process ID, which is also the same as the Id column
in SHOW FULL PROCESSLIST. This is followed by an internal query number and some
connection information (also the same as what you can find in SHOW FULL PROCESSLIST).

Each transaction can print much more information than that, though. Here’s a more
complex example:

```
1 ---TRANSACTION 0 80157600, ACTIVE 4 sec, process no 3396, OS thread id 1148250464,
thread declared inside InnoDB 442
2 mysql tables in use 1, locked 0
3 MySQL thread id 8079, query id 728899 localhost baron Sending data
4 select sql_calc_found_rows * from b limit 5
5 Trx read view will not see trx with id>= 0 80157601, sees <0 80157597
```
**700 | Appendix B: MySQL Server Status**


Line 1 in this sample shows the transaction has been active for four seconds. The
possible states are “not started,” “active,” “prepared,” and “committed in memory”
(once it commits to disk, the state will change to “not started”). You might also see
information about what the transaction is currently doing, though this example doesn’t
show that. There are over 30 string constants in the source that can be printed here,
such as “fetching rows,” “adding foreign keys,” and so on.

The “thread declared inside InnoDB 442” text on line 1 means the thread is doing some
operation inside the InnoDB kernel and has 442 “tickets” left to use. In other words,
the same SQL query is allowed to reenter the InnoDB kernel 442 more times. The ticket
system limits thread concurrency inside the kernel to prevent thread thrashing on some
platforms. Even if the thread’s state is “inside InnoDB,” the thread might not necessarily
be doing all its work inside InnoDB; the query might be processing some operations at
the server level and just interacting with the InnoDB kernel in some way. You might
also see that the transaction’s status is “sleeping before joining InnoDB queue” or
“waiting in InnoDB queue.”

The next line you might see shows how many tables the current statement has used
and locked. InnoDB doesn’t normally lock tables, but it does for some statements.
Locked tables can also show up if the MySQL server has locked them at a higher level
than InnoDB. If the transaction has locked any rows, there will be a line showing the
number of lock structs (again, not the same thing as row locks) and the heap size; you
can see examples of this in the earlier deadlock output. In MySQL 5.1 and newer, this
line also shows the actual number of row locks the transaction holds.

The heap size is the amount of memory used to hold row locks. InnoDB implements
row locks with a special table of bitmaps, which can theoretically use as little as one
bit per row it locks. Our tests have shown that it generally uses no more than four bits
per lock.

The third line in this example has a little more information than the second line in the
previous sample: at the end of the line is the thread status, “Sending data.” This is the
same as what you’ll see in the Command column in SHOW FULL PROCESSLIST.

If the transaction is actively running a query, the query’s text (or, in some MySQL
versions, just an excerpt of it) will come next, in this case in line 4.

Line 5 shows the transaction’s read view, which indicates the range of transaction
identifiers that are definitely visible and definitely invisible to the transaction because
of versioning. In this case, there’s a gap of four transactions between the two numbers.
These four transactions might not be visible. When InnoDB executes a query, it must
check the visibility of any rows whose transaction identifiers fall into this gap.

If the transaction is waiting for a lock, you’ll also see the lock information just after the
query. There are examples of this in the earlier deadlock sample as well. Unfortunately,
the output doesn’t say which other transaction _holds_ the lock for which this transaction

```
SHOW ENGINE INNODB STATUS | 701
```

is waiting. You can find that in the INFORMATION_SCHEMA tables in MySQL 5.1 and newer,
if you’re using the InnoDB plugin.

If there are many transactions, InnoDB might limit the number it prints to try to keep
the output from growing too large. You’ll see “ ...truncated... ” if this happens.

#### FILE I/O

The FILE I/O section shows the state of the I/O helper threads, along with performance
counters:

```
1 --------
2 FILE I/O
3 --------
4 I/O thread 0 state: waiting for i/o request (insert buffer thread)
5 I/O thread 1 state: waiting for i/o request (log thread)
6 I/O thread 2 state: waiting for i/o request (read thread)
7 I/O thread 3 state: waiting for i/o request (write thread)
8 Pending normal aio reads: 0, aio writes: 0,
9 ibuf aio reads: 0, log i/o's: 0, sync i/o's: 0
10 Pending flushes (fsync) log: 0; buffer pool: 0
11 17909940 OS file reads, 22088963 OS file writes, 1743764 OS fsyncs
12 0.20 reads/s, 16384 avg bytes/read, 5.00 writes/s, 0.80 fsyncs/s
```
Lines 4 through 7 show the I/O helper thread states. Lines 8 through 10 show the
number of pending operations for each helper thread, and the number of pending
fsync() operations for the log and buffer pool threads. The abbreviation “aio” means
“asynchronous I/O.” Line 11 shows the number of reads, writes, and fsync() calls
performed. Absolute values will vary with your workload, so it’s more important to
monitor how they change over time. Line 12 shows per-second averages over the time
interval shown in the header section.

The pending values on lines 8 and 9 are good ways to detect an I/O-bound application.
If most of these types of I/O have some pending operations, the workload is probably
I/O-bound.

On Windows, you can adjust the number of I/O helper threads with the innodb
_file_io_threads configuration variable, so you might see more than one read and
write thread. And in MySQL 5.1 and newer with the InnoDB plugin, or with Percona
Server, you can use innodb_read_io_threads and innodb_write_io_threads to configure
multiple threads for reading and writing. However, you’ll always see at least these four
threads on all platforms:

_Insert buffer thread_
Responsible for insert buffer merges (i.e., records being merged from the insert
buffer into the tablespace)

_Log thread_
Responsible for asynchronous log flushes

**702 | Appendix B: MySQL Server Status**


_Read thread_
Performs read-ahead operations to try to prefetch data InnoDB predicts it will need

_Write thread_
Flushes dirty buffers

#### INSERT BUFFER AND ADAPTIVE HASH INDEX

This section shows the status of these two structures inside InnoDB:

```
1 -------------------------------------
2 INSERT BUFFER AND ADAPTIVE HASH INDEX
3 -------------------------------------
4 Ibuf for space 0: size 1, free list len 887, seg size 889, is not empty
5 Ibuf for space 0: size 1, free list len 887, seg size 889,
6 2431891 inserts, 2672643 merged recs, 1059730 merges
7 Hash table size 8850487, used cells 2381348, node heap has 4091 buffer(s)
8 2208.17 hash searches/s, 175.05 non-hash searches/s
```
Line 4 shows information about the insert buffer’s size, the length of its “free list,” and
its segment size. The text “for space 0” seems to indicate the possibility of multiple
insert buffers—one per tablespace—but that was never implemented, and this text has
been removed in more recent MySQL versions. There’s only one insert buffer, so line
5 is really redundant. Line 6 shows statistics about how many buffer operations
InnoDB has done. The ratio of merges to inserts gives a good idea of how efficient the
buffer is.

Line 7 shows the adaptive hash index’s status. Line 8 shows how many hash index
operations InnoDB has done over the time frame mentioned in the header section. The
ratio of hash index lookups to non-hash index lookups is advisory information; you
can’t configure the adaptive hash index.

#### LOG

This section shows statistics about InnoDB’s transaction log (redo log) subsystem:

```
1 ---
2 LOG
3 ---
4 Log sequence number 84 3000620880
5 Log flushed up to 84 3000611265
6 Last checkpoint at 84 2939889199
7 0 pending log writes, 0 pending chkp writes
8 14073669 log i/o's done, 10.90 log i/o's/second
```
Line 4 shows the current log sequence number, and line 5 shows the point up to which
the logs have been flushed. The log sequence number is just the number of bytes written
to the log files, so you can use it to calculate how much data in the log buffer has not
yet been flushed to the log files. In this case, it is 9,615 bytes (13000620880–
13000611265). Line 6 shows the last checkpoint (a checkpoint identifies an instant at

```
SHOW ENGINE INNODB STATUS | 703
```

which the data and log files were in a known state, and can be used for recovery). If the
last checkpoint falls too far behind the log sequence number, and the difference be-
comes close to the size of the log files, InnoDB will trigger “furious flushing,” which is
very bad for performance. Lines 7 and 8 show pending log operations and statistics,
which you can compare to values in the FILE I/O section to see how much of your I/O
is caused by your log subsystem relative to other causes of I/O.

#### BUFFER POOL AND MEMORY

This section shows statistics about InnoDB’s buffer pool and how it uses memory:

```
1 ----------------------
2 BUFFER POOL AND MEMORY
3 ----------------------
4 Total memory allocated 4648979546; in additional pool allocated 16773888
5 Buffer pool size 262144
6 Free buffers 0
7 Database pages 258053
8 Modified db pages 37491
9 Pending reads 0
10 Pending writes: LRU 0, flush list 0, single page 0
11 Pages read 57973114, created 251137, written 10761167
12 9.79 reads/s, 0.31 creates/s, 6.00 writes/s
13 Buffer pool hit rate 999 / 1000
```
Line 4 shows the total memory allocated by InnoDB, and how much of that amount is
allocated in the additional memory pool. The additional memory pool is just a (typically
small) amount of memory it allocates when it wants to use its own internal memory
allocator. Modern versions of InnoDB typically use the operating system’s memory
allocator, but older versions had their own allocator because some operating systems
didn’t provide a very good implementation.

Lines 5 through 8 show buffer pool metrics, in units of pages. The metrics are the total
buffer pool size, the number of free pages, the number of pages allocated to store da-
tabase pages, and the number of “dirty” database pages. InnoDB uses some pages in
the buffer pool for lock indexes, the adaptive hash index, and other system structures,
so the number of database pages in the pool will never equal the total pool size.

Lines 9 and 10 show the number of pending reads and writes (i.e., the number of logical
reads and writes InnoDB needs to do for the buffer pool). These values will not match
values in the FILE I/O section, because InnoDB might merge many logical operations
into a single physical I/O operation. LRU stands for “least recently used”; it’s a method
of freeing space for frequently used pages by flushing infrequently used ones from the
buffer pool. The flush list holds old pages that need to be flushed by the checkpoint
process, and single page writes are independent page writes that won’t be merged.

Line 8 in this output shows that the buffer pool contains 37,491 dirty pages, which
need to be flushed to disk at some point (they have been modified in memory but not
on disk). However, line 10 shows that no flushes are scheduled at the moment. This is

**704 | Appendix B: MySQL Server Status**


not a problem; InnoDB will flush them when it needs to. If you see a high number of
pending I/O operations anywhere in InnoDB’s status output, it’s typically indicative of
a pretty severe problem.

Line 11 shows how many pages InnoDB has read, created, and written. The pages read
and written values refer to data that’s read into the buffer pool from disk, or vice versa.
The pages created value refers to pages that InnoDB allocates in the buffer pool without
reading their contents from the data file, because it doesn’t care what the contents are
(for example, they might have belonged to a table that has since been dropped).

Line 13 reports the buffer pool hit rate, which measures the rate at which InnoDB finds
the pages it needs in the buffer pool. It measures hits since the last InnoDB status
printout, so if the server has been quiet since then, you’ll see “No buffer pool page gets
since the last printout.” It’s not useful as a metric for buffer pool sizing.

In MySQL 5.5, there might be several buffer pools, and each one will print out a section
in the output. Percona XtraDB will also print more detailed output—for example,
showing exactly where memory is allocated.

#### ROW OPERATIONS

This section shows miscellaneous InnoDB statistics:

```
1 --------------
2 ROW OPERATIONS
3 --------------
4 0 queries inside InnoDB, 0 queries in queue
5 1 read views open inside InnoDB
6 Main thread process no. 10099, id 88021936, state: waiting for server activity
7 Number of rows inserted 143, updated 3000041, deleted 0, read 24865563
8 0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
9 ----------------------------
10 END OF INNODB MONITOR OUTPUT
11 ============================
```
Line 4 shows how many threads are inside the InnoDB kernel (we referred to this in
our discussion of the TRANSACTIONS section). Queries in the queue are threads InnoDB
is not admitting into the kernel yet to restrict the number of threads concurrently ex-
ecuting. Queries can also be sleeping before they go into the queue to wait, as discussed
earlier.

Line 5 shows how many read views InnoDB has open. A read view is a consistent MVCC
“snapshot” of the database’s contents as of the point the transaction started. You can
see whether a specific transaction has a read view in the TRANSACTIONS section.

Line 6 shows the kernel’s main thread status. Possible status values are as follows:

- doing background drop tables
- doing insert buffer merge
- flushing buffer pool pages

```
SHOW ENGINE INNODB STATUS | 705
```

- flushing log
- making checkpoint
- purging
- reserving kernel mutex
- sleeping
- suspending
- waiting for buffer pool flush to end
- waiting for server activity

You should usually see “sleeping” in most servers, and if you take several snapshots
and repeatedly see a different status, such as “flushing buffer pool pages,” you should
suspect a problem with the related activity—for example, a “furious flushing” problem
caused by a version of InnoDB with a poor flushing algorithm, or poor configuration
such as too-small transaction log files.

Lines 7 and 8 show statistics on the number of rows inserted, updated, deleted, and
read, and per-second averages of these values. These are good numbers to monitor if
you want to watch how much work InnoDB is doing.

The SHOW ENGINE INNODB STATUS output ends with lines 9 through 13. If you don’t see
this text, you probably have a very large deadlock that’s truncating the output.

### SHOW PROCESSLIST

The process list is the list of connections, or threads, that are currently connected to
MySQL. SHOW PROCESSLIST lists the threads, with information about each thread’s
status. For example:

```
mysql> SHOW FULL PROCESSLIST\G
*************************** 1. row ***************************
Id: 61539
User: sphinx
Host: se02:58392
db: art136
Command: Query
Time: 0
State: Sending data
Info: SELECT a.id id, a.site_id site_id, unix_timestamp(inserted) AS
inserted,forum_id, unix_timestamp(p
*************************** 2. row ***************************
Id: 65094
User: mailboxer
Host: db01:59659
db: link84
Command: Killed
Time: 12931
State: end
```
**706 | Appendix B: MySQL Server Status**


```
Info: update link84.link_in84 set url_to =
replace(replace(url_to,'&amp;','&'),'%20','+'), url_prefix=repl
```
There are several tools (such as _innotop_ ) that can show you an updating view of the
process list.

You can also retrieve this information from a table in the INFORMATION_SCHEMA. Percona
Server and MariaDB add more useful information to this table, such as a high-resolution
time column or a column that indicates how much work the query has done, which
you can use as a progress indicator.

The Command and State columns are where the thread’s “status” is really indicated.
Notice that the first of our processes is running a query and sending data while the
second has been killed, probably because it took a very long time to complete and
someone deliberately terminated it with the KILL command. A thread can remain in
this state for some time, because a kill might not complete instantly. For example, it
might take a while to roll back the thread’s transaction.

SHOW FULL PROCESSLIST (with the added FULL keyword) shows the full text of each query,
which is otherwise truncated after 100 characters.

### SHOW ENGINE INNODB MUTEX

SHOW ENGINE INNODB MUTEX returns detailed InnoDB mutex information and is mostly
useful for gaining insight into scalability and concurrency problems. Each mutex pro-
tects a critical section in the code, as explained previously.

The output varies depending on the MySQL version and compile options. Here’s a
sample from a MySQL 5.5 server:

```
mysql> SHOW ENGINE INNODB MUTEX;
+--------+------------------------------+-------------+
| Type | Name | Status |
+--------+------------------------------+-------------+
| InnoDB | &table->autoinc_mutex | os_waits=1 |
| InnoDB | &table->autoinc_mutex | os_waits=1 |
| InnoDB | &table->autoinc_mutex | os_waits=4 |
| InnoDB | &table->autoinc_mutex | os_waits=1 |
| InnoDB | &table->autoinc_mutex | os_waits=12 |
| InnoDB | &dict_sys->mutex | os_waits=1 |
| InnoDB | &log_sys->mutex | os_waits=12 |
| InnoDB | &fil_system->mutex | os_waits=11 |
| InnoDB | &kernel_mutex | os_waits=1 |
| InnoDB | &dict_table_stats_latches[i] | os_waits=2 |
| InnoDB | &dict_table_stats_latches[i] | os_waits=54 |
| InnoDB | &dict_table_stats_latches[i] | os_waits=1 |
| InnoDB | &dict_table_stats_latches[i] | os_waits=31 |
| InnoDB | &dict_table_stats_latches[i] | os_waits=41 |
| InnoDB | &dict_table_stats_latches[i] | os_waits=12 |
| InnoDB | &dict_table_stats_latches[i] | os_waits=1 |
| InnoDB | &dict_table_stats_latches[i] | os_waits=90 |
| InnoDB | &dict_table_stats_latches[i] | os_waits=1 |
```
```
SHOW ENGINE INNODB MUTEX | 707
```

```
| InnoDB | &dict_operation_lock | os_waits=13 |
| InnoDB | &log_sys->checkpoint_lock | os_waits=66 |
| InnoDB | combined &block->lock | os_waits=2 |
+--------+------------------------------+-------------+
```
You can use the output to help determine which parts of InnoDB are bottlenecks, based
on the number of waits. Anywhere there’s a mutex, there’s a potential for contention.
You might need to write a script to aggregate the output, which can be very large.

There are three main strategies for easing mutex-related bottlenecks: try to avoid
InnoDB’s weak points, try to limit concurrency, or try to balance between CPU-inten-
sive spin waits and resource-intensive operating system waits. We discussed this earlier
in this appendix, and in Chapter 8.

### Replication Status

MySQL has several commands for monitoring replication. On a master server, SHOW
MASTER STATUS shows the master’s replication status and configuration:

```
mysql> SHOW MASTER STATUS\G
*************************** 1. row ***************************
File: mysql-bin.000079
Position: 13847
Binlog_Do_DB:
Binlog_Ignore_DB:
```
The output includes the master’s current binary log position. You can get a list of binary
logs with SHOW BINARY LOGS:

```
mysql> SHOW BINARY LOGS
+------------------+-----------+
| Log_name | File_size |
+------------------+-----------+
| mysql-bin.000044 | 13677 |
...
| mysql-bin.000079 | 13847 |
+------------------+-----------+
36 rows in set (0.18 sec)
```
To view the events in the binary logs, use SHOW BINLOG EVENTS. In MySQL 5.5, you can
also use SHOW RELAYLOG EVENTS.

On a replica server, you can view the replica’s status and configuration with SHOW SLAVE
STATUS. We won’t include the output here, because it’s a bit verbose, but we will note
a few things about it. First, you can see the status of both the replica I/O and replica
SQL threads, including any errors. You can also see how far behind the replica is in
replication. Finally, for the purposes of backups and cloning replicas, there are three
sets of binary log coordinates in the output:

Master_Log_File/Read_Master_Log_Pos
The position at which the I/O thread is reading in the master’s binary logs.

**708 | Appendix B: MySQL Server Status**


Relay_Log_File/Relay_Log_Pos
The position at which the SQL thread is executing in the replica’s relay logs.

Relay_Master_Log_File/Exec_Master_Log_Pos
The position at which the SQL thread is executing in the master’s binary logs. This
is the same logical position as Relay_Log_File/Relay_Log_Pos, but it’s in the rep-
lica’s relay logs instead of the master’s binary logs. In other words, if you look at
these two positions in the logs, you will find the same log events.

### The INFORMATION_SCHEMA

The INFORMATION_SCHEMA database is a set of system views defined in the SQL standard.
MySQL implements many of the standard views and adds some others. In MySQL 5.1,
many of the views correspond to MySQL’s SHOW commands, such as SHOW FULL
PROCESSLIST and SHOW STATUS. However, there are also some views that have no corre-
sponding SHOW command.

The beauty of the INFORMATION_SCHEMA views is that you can query them with standard
SQL. This gives you much more flexibility than the SHOW commands, which produce
result sets that you can’t aggregate, join, or otherwise manipulate with standard SQL.
Having all this data available in system views makes it possible to write interesting and
useful queries.

For example, what tables have a reference to the actor table in the Sakila sample
database? The consistent naming convention makes this relatively easy to determine:

```
mysql> SELECT TABLE_NAME FROM INFORMATION_SCHEMA.COLUMNS
-> WHERE TABLE_SCHEMA='sakila' AND COLUMN_NAME='actor_id'
-> AND TABLE_NAME <> 'actor';
+------------+
| TABLE_NAME |
+------------+
| actor_info |
| film_actor |
+------------+
```
We needed to find tables with multiple-column indexes for several of the examples in
this book. Here’s a query for that:

```
mysql> SELECT TABLE_NAME, GROUP_CONCAT(COLUMN_NAME)
-> FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
-> WHERE TABLE_SCHEMA='sakila'
-> GROUP BY TABLE_NAME, CONSTRAINT_NAME
-> HAVING COUNT(*) > 1;
+---------------+--------------------------------------+
| TABLE_NAME | GROUP_CONCAT(COLUMN_NAME) |
+---------------+--------------------------------------+
| film_actor | actor_id,film_id |
| film_category | film_id,category_id |
| rental | customer_id,rental_date,inventory_id |
+---------------+--------------------------------------+
```
```
The INFORMATION_SCHEMA | 709
```

You can also write more complex queries, just as you would against any ordinary tables.
The MySQL Forge ( _[http://forge.mysql.com](http://forge.mysql.com)_ ) is a great place to find and share queries
against these views. There are samples to find duplicate or redundant indexes, find
indexes with very low cardinality, and much more. There is also a set of useful views
written on top of the INFORMATION_SCHEMA views in Shlomi Noach’s _common_schema_
project ( _[http://code.openark.org/forge/common_schema](http://code.openark.org/forge/common_schema)_ ).

The biggest drawback is that the views are sometimes very slow compared to the cor-
responding SHOW commands. They typically fetch all the data, store it in a temporary
table, then make the temporary table available to the query. Querying the INFORMA
TION_SCHEMA tables on a server with a lot of data or many tables can cause a great deal
of load on the server, and can cause the server to stall or become unresponsive to other
users, so do be cautious about using it on a heavily loaded, large server in production.
The main tables that can be dangerous to query are the ones that contain table meta-
data: TABLES, COLUMNS, REFERENTIAL_CONSTRAINTS, KEY_COLUMN_USAGE, and so forth.
Queries against these tables can cause MySQL to ask the storage engine for data such
as index statistics on the tables in the server, which is especially burdensome in InnoDB.

The views aren’t updatable. Although you can retrieve server settings from them, you
can’t update them to influence the server’s configuration, so you’ll still need to use the
SHOW and SET commands for configuration, even though the INFORMATION_SCHEMA views
are very useful for other tasks.

#### InnoDB Tables

In MySQL 5.1 and newer, the InnoDB plugin creates a number of INFORMATION
_SCHEMA tables. These are very helpful. There are more in MySQL 5.5, and even more
in the unreleased MySQL 5.6.

In MySQL 5.1, the following tables exist:

INNODB_CMP _and_ INNODB_CMP_RESET
These tables show information about data that’s compressed in InnoDB’s new file
format, Barracuda. The second table shows the same information as the first but
has the side effect of resetting the data it contains, sort of like using a FLUSH
command.

INNODB_CMPMEM _and_ INNODB_CMPMEM_RESET
These tables show information about buffer pool pages used for InnoDB com-
pressed data. The second table is again a reset table.

INNODB_TRX _and_ INNODB_LOCKS
These tables show transactions, and transactions that hold and wait for locks.
They are very important for diagnosing lock wait problems and long-running
transactions. The MySQL manual contains sample queries you can copy and paste
to show which transactions are blocking which others, the queries they’re running,
and so forth.

**710 | Appendix B: MySQL Server Status**


In addition to these tables, MySQL 5.5 adds INNODB_LOCK_WAITS, which can help diag-
nose more types of lock-waiting problems more easily. MySQL 5.6 will add tables that
show more information about InnoDB’s internals, including the buffer pool and data
dictionary, and a new table called INNODB_METRICS, which will be an alternative to using
the Performance Schema.

#### Tables in Percona Server

Percona Server adds a large number of tables to the INFORMATION_SCHEMA database. The
stock MySQL 5.5 server has 39 tables, and Percona Server 5.5 has 61 tables. Here’s an
overview of the additional tables:

_The “user statistics” tables_
These tables originated in Google’s patches for MySQL. They show activity sta-
tistics for clients, indexes, tables, threads, and users. We’ve mentioned uses for
these throughout this book, such as determining when replication is beginning to
approach the limit of its ability to keep up with the master.

_The InnoDB data dictionary_
A series of tables that expose InnoDB’s internal data dictionary as read-only tables:
columns, foreign keys, indexes, statistics, and so on. These are very helpful for
examining and understanding InnoDB’s view of the database, which can differ
from MySQL’s due to MySQL’s reliance on _.frm_ files to store the data dictionary.
Similar tables will be included with MySQL 5.6 when it is released.

_The InnoDB buffer pool_
These tables let you query the buffer pool as though it’s a table where each page is
a row, so you can see what pages are resident in the buffer pool, what types of pages
they are, and so on. These tables have proven useful for diagnosing problems such
as a bloated insert buffer.

_Temporary tables_
These tables show the same type of information available in the INFORMATION
_SCHEMA.TABLES table, but for temporary tables instead. There is one for your own
session’s temporary tables, and one for all temporary tables in the whole server.
Both are helpful for gaining visibility into which temporary tables exist, for which
sessions, and how much space they’re using.

_Miscellaneous tables_
There are a handful of other tables that add visibility into query execution times,
files, tablespaces, and more InnoDB internals.

The documentation for Percona Server’s additional tables is available at _[http://www](http://www)
.percona.com/doc/_.

```
The INFORMATION_SCHEMA | 711
```

### The Performance Schema

The Performance Schema (which resides in the PERFORMANCE_SCHEMA database) is
MySQL’s new home for enhanced instrumentation, as of MySQL 5.5. We discussed it
a bit in Chapter 3.

By default, the Performance Schema is disabled, and you have to turn it on and enable
specific instrumentation points (“consumers”) that you wish to collect. We bench-
marked the server in a few different configurations and found that the Performance
Schema caused around an 8% to 11% overhead even when it was collecting no data,
and 19% to 25% with all consumers enabled, depending on whether it was a read-only
or read/write workload. Whether this is a little or a lot is up to you to decide.

This is slated to improve in MySQL 5.6, especially when the feature itself is enabled
but all of the instrumentation points are disabled. This will make it more practical for
some users to enable the Performance Schema, but leave it inactive until they want to
gather some information.

In MySQL 5.5, the Performance Schema contains tables that instrument instances of
condition variables, mutexes, read/write locks, and file I/O. There are also tables that
instrument the waits on the instances, and these are what you’ll usually be interested
in querying first, with joins to the instance tables. These event wait tables come in a
few variations that hold current and historical information about server performance
and behavior. Finally, there are is a group of “setup tables,” which you use to enable
or disable the desired consumers.

In MySQL 5.6.3 development milestone release 6, the number of tables in the Perfor-
mance Schema increases from 17 to 49. That means that there is a lot more instru-
mentation in MySQL 5.6! Added instrumentation includes SQL statements, stages of
statements (basically the same thing as the thread status you can see in SHOW PROCESS
LIST), tables, indexes, hosts, threads, users, accounts, and a larger variety of summary
and history tables, among other things.

How can you use these tables? With 49 of them, the time has come for someone to
write tools to help with this. However, for some good examples of old-fashioned SQL
against the Performance Schema tables, you can read some of the articles on Oracle
engineer Mark Leith’s blog, such as _[http://www.markleith.co.uk/?p=471](http://www.markleith.co.uk/?p=471)_.

**712 | Appendix B: MySQL Server Status**


### Summary

MySQL’s primary means of exposing server internals is the SHOW commands, but that’s
changing. The introduction in MySQL 5.1 of pluggable INFORMATION_SCHEMA tables per-
mitted the InnoDB plugin to add some very valuable instrumentation, and Percona
Server adds many more. However, the ability to read SHOW ENGINE INNODB STATUS output
and interpret it remains essential for managing InnoDB. In MySQL 5.5 and newer server
versions, the Performance Schema is available, and it will probably become the most
powerful and complete means of inspecting the server’s internals. The great thing about
the Performance Schema is that it’s time-based, meaning that MySQL is finally getting
instrumented for elapsed time, not just operation counts.

```
Summary | 713
```

