

```
CHAPTER 4
```
```
Optimizing Schema and Data Types
```
Good logical and physical design is the cornerstone of high performance, and you must
design your schema for the specific queries you will run. This often involves trade-offs.
For example, a denormalized schema can speed up some types of queries but slow down
others. Adding counter and summary tables is a great way to optimize queries, but they
can be expensive to maintain. MySQL’s particular features and implementation details
influence this quite a bit.

This chapter and the following one, which focuses on indexing, cover the MySQL-
specific bits of schema design. We assume that you know how to design databases, so
this is not an introductory chapter, or even an advanced chapter, on database design.
It’s a chapter on MySQL database design—it’s about what is different when designing
databases with MySQL rather than other relational database management systems. If
you need to study the basics of database design, we suggest Clare Churcher’s book
_Beginning Database Design_ (Apress).

This chapter is preparation for the two that follow. In these three chapters, we will
explore the interaction of logical design, physical design, and query execution. This
requires a big-picture approach as well as attention to details. You need to understand
the whole system to understand how each piece will affect others. You might find it
useful to review this chapter after reading the chapters on indexing and query optimi-
zation. Many of the topics discussed can’t be considered in isolation.

### Choosing Optimal Data Types

MySQL supports a large variety of data types, and choosing the correct type to store
your data is crucial to getting good performance. The following simple guidelines can
help you make better choices, no matter what type of data you are storing:

_Smaller is usually better._
In general, try to use the smallest data type that can correctly store and represent
your data. Smaller data types are usually faster, because they use less space on the

```
115
```

```
disk, in memory, and in the CPU cache. They also generally require fewer CPU
cycles to process.
Make sure you don’t underestimate the range of values you need to store, though,
because increasing the data type range in multiple places in your schema can be a
painful and time-consuming operation. If you’re in doubt as to which is the best
data type to use, choose the smallest one that you don’t think you’ll exceed. (If the
system is not very busy or doesn’t store much data, or if you’re at an early phase
in the design process, you can change it easily later.)
```
_Simple is good._
Fewer CPU cycles are typically required to process operations on simpler data
types. For example, integers are cheaper to compare than characters, because
character sets and collations (sorting rules) make character comparisons compli-
cated. Here are two examples: you should store dates and times in MySQL’s built-
in types instead of as strings, and you should use integers for IP addresses. We
discuss these topics further later.

_Avoid_ NULL _if possible._
A lot of tables include nullable columns even when the application does not need
to store NULL (the absence of a value), merely because it’s the default. It’s usually
best to specify columns as NOT NULL unless you intend to store NULL in them.
It’s harder for MySQL to optimize queries that refer to nullable columns, because
they make indexes, index statistics, and value comparisons more complicated. A
nullable column uses more storage space and requires special processing inside
MySQL. When a nullable column is indexed, it requires an extra byte per entry
and can even cause a fixed-size index (such as an index on a single integer column)
to be converted to a variable-sized one in MyISAM.
The performance improvement from changing NULL columns to NOT NULL is usually
small, so don’t make it a priority to find and change them on an existing schema
unless you know they are causing problems. However, if you’re planning to index
columns, avoid making them nullable if possible.
There are exceptions, of course. For example, it’s worth mentioning that InnoDB
stores NULL with a single bit, so it can be pretty space-efficient for sparsely populated
data. This doesn’t apply to MyISAM, though.

The first step in deciding what data type to use for a given column is to determine what
general class of types is appropriate: numeric, string, temporal, and so on. This is usu-
ally pretty straightforward, but we mention some special cases where the choice is
unintuitive.

The next step is to choose the specific type. Many of MySQL’s data types can store the
same kind of data but vary in the range of values they can store, the precision they
permit, or the physical space (on disk and in memory) they require. Some data types
also have special behaviors or properties.

**116 | Chapter 4: Optimizing Schema and Data Types**


For example, a DATETIME and a TIMESTAMP column can store the same kind of data: date
and time, to a precision of one second. However, TIMESTAMP uses only half as much
storage space, is time zone–aware, and has special autoupdating capabilities. On the
other hand, it has a much smaller range of allowable values, and sometimes its special
capabilities can be a handicap.

We discuss base data types here. MySQL supports many aliases for compatibility, such
as INTEGER, BOOL, and NUMERIC. These are only aliases. They can be confusing, but they
don’t affect performance. If you create a table with an aliased data type and then ex-
amine SHOW CREATE TABLE, you’ll see that MySQL reports the base type, not the alias
you used.

#### Whole Numbers

There are two kinds of numbers: whole numbers and real numbers (numbers with a
fractional part). If you’re storing whole numbers, use one of the integer types: TINYINT,
SMALLINT, MEDIUMINT, INT, or BIGINT. These require 8, 16, 24, 32, and 64 bits of storage
space, respectively. They can store values from −2( _N_ –1) to 2( _N_ –1)–1, where _N_ is the number
of bits of storage space they use.

Integer types can optionally have the UNSIGNED attribute, which disallows negative val-
ues and approximately doubles the upper limit of positive values you can store. For
example, a TINYINT UNSIGNED can store values ranging from 0 to 255 instead of from
−128 to 127.

Signed and unsigned types use the same amount of storage space and have the same
performance, so use whatever’s best for your data range.

Your choice determines how MySQL _stores_ the data, in memory and on disk. However,
integer _computations_ generally use 64-bit BIGINT integers, even on 32-bit architectures.
(The exceptions are some aggregate functions, which use DECIMAL or DOUBLE to perform
computations.)

MySQL lets you specify a “width” for integer types, such as INT(11). This is meaningless
for most applications: it does not restrict the legal range of values, but simply specifies
the number of characters MySQL’s interactive tools (such as the command-line client)
will reserve for display purposes. For storage and computational purposes, INT(1) is
identical to INT(20).

```
Third-party storage engines, such as Infobright, sometimes have their
own storage formats and compression schemes, and don’t necessarily
use those that are common to MySQL’s built-in storage engines.
```
```
Choosing Optimal Data Types | 117
```

#### Real Numbers

Real numbers are numbers that have a fractional part. However, they aren’t just for
fractional numbers; you can also use DECIMAL to store integers that are so large they
don’t fit in BIGINT. MySQL supports both exact and inexact types.

The FLOAT and DOUBLE types support approximate calculations with standard floating-
point math. If you need to know exactly how floating-point results are calculated, you
will need to research your platform’s floating-point implementation.

The DECIMAL type is for storing exact fractional numbers. In MySQL 5.0 and newer, the
DECIMAL type supports exact math. MySQL 4.1 and earlier used floating-point math to
perform computations on DECIMAL values, which could give strange results because of
loss of precision. In these versions of MySQL, DECIMAL was only a “storage type.”

The server itself performs DECIMAL math in MySQL 5.0 and newer, because CPUs don’t
support the computations directly. Floating-point math is significantly faster, because
the CPU performs the computations natively.

Both floating-point and DECIMAL types let you specify a precision. For a DECIMAL column,
you can specify the maximum allowed digits before and after the decimal point. This
influences the column’s space consumption. MySQL 5.0 and newer pack the digits into
a binary string (nine digits per four bytes). For example, DECIMAL(18, 9) will store nine
digits from each side of the decimal point, using nine bytes in total: four for the digits
before the decimal point, one for the decimal point itself, and four for the digits after
the decimal point.

A DECIMAL number in MySQL 5.0 and newer can have up to 65 digits. Earlier MySQL
versions had a limit of 254 digits and stored the values as unpacked strings (one byte
per digit). However, these versions of MySQL couldn’t actually use such large numbers
in computations, because DECIMAL was just a storage format; DECIMAL numbers were
converted to DOUBLEs for computational purposes,

You can specify a floating-point column’s desired precision in a couple of ways, which
can cause MySQL to silently choose a different data type or to round values when you
store them. These precision specifiers are nonstandard, so we suggest that you specify
the type you want but not the precision.

Floating-point types typically use less space than DECIMAL to store the same range of
values. A FLOAT column uses four bytes of storage. DOUBLE consumes eight bytes and has
greater precision and a larger range of values than FLOAT. As with integers, you’re
choosing only the storage type; MySQL uses DOUBLE for its internal calculations on
floating-point types.

Because of the additional space requirements and computational cost, you should use
DECIMAL only when you need exact results for fractional numbers—for example, when
storing financial data. But in some high-volume cases it actually makes sense to use a
BIGINT instead, and store the data as some multiple of the smallest fraction of currency

**118 | Chapter 4: Optimizing Schema and Data Types**


you need to handle. Suppose you are required to store financial data to the ten-
thousandth of a cent. You can multiply all dollar amounts by a million and store the
result in a BIGINT, avoiding both the imprecision of floating-point storage and the cost
of the precise DECIMAL math.

#### String Types

MySQL supports quite a few string data types, with many variations on each. These
data types changed greatly in versions 4.1 and 5.0, which makes them even more com-
plicated. Since MySQL 4.1, each string column can have its own character set and set
of sorting rules for that character set, or _collation_ (see Chapter 7 for more on these
topics). This can impact performance greatly.

**VARCHAR and CHAR types**

The two major string types are VARCHAR and CHAR, which store character values. Un-
fortunately, it’s hard to explain exactly how these values are stored on disk and in
memory, because the implementations are storage engine–dependent. We assume you
are using InnoDB and/or MyISAM. If not, you should read the documentation for your
storage engine.

Let’s take a look at how VARCHAR and CHAR values are typically stored on disk. Be aware
that a storage engine may store a CHAR or VARCHAR value differently in memory from how
it stores that value on disk, and that the server may translate the value into yet another
storage format when it retrieves it from the storage engine. Here’s a general comparison
of the two types:

VARCHAR
VARCHAR stores variable-length character strings and is the most common string
data type. It can require less storage space than fixed-length types, because it uses
only as much space as it needs (i.e., less space is used to store shorter values). The
exception is a MyISAM table created with ROW_FORMAT=FIXED, which uses a fixed
amount of space on disk for each row and can thus waste space.
VARCHAR uses 1 or 2 extra bytes to record the value’s length: 1 byte if the column’s
maximum length is 255 bytes or less, and 2 bytes if it’s more. Assuming the
latin1 character set, a VARCHAR(10) will use up to 11 bytes of storage space. A
VARCHAR(1000) can use up to 1002 bytes, because it needs 2 bytes to store length
information.
VARCHAR helps performance because it saves space. However, because the rows are
variable-length, they can grow when you update them, which can cause extra work.
If a row grows and no longer fits in its original location, the behavior is storage
engine–dependent. For example, MyISAM may fragment the row, and InnoDB
may need to split the page to fit the row into it. Other storage engines may never
update data in-place at all.

```
Choosing Optimal Data Types | 119
```

```
It’s usually worth using VARCHAR when the maximum column length is much larger
than the average length; when updates to the field are rare, so fragmentation is not
a problem; and when you’re using a complex character set such as UTF-8, where
each character uses a variable number of bytes of storage.
In version 5.0 and newer, MySQL preserves trailing spaces when you store and
retrieve values. In versions 4.1 and older, MySQL strips trailing spaces.
It’s trickier with InnoDB, which can store long VARCHAR values as BLOBs. We discuss
this later.
```
CHAR
CHAR is fixed-length: MySQL always allocates enough space for the specified num-
ber of characters. When storing a CHAR value, MySQL removes any trailing spaces.
(This was also true of VARCHAR in MySQL 4.1 and older versions—CHAR and VAR
CHAR were logically identical and differed only in storage format.) Values are padded
with spaces as needed for comparisons.
CHAR is useful if you want to store very short strings, or if all the values are nearly
the same length. For example, CHAR is a good choice for MD5 values for user pass-
words, which are always the same length. CHAR is also better than VARCHAR for data
that’s changed frequently, because a fixed-length row is not prone to fragmenta-
tion. For very short columns, CHAR is also more efficient than VARCHAR; a CHAR(1)
designed to hold only Y and N values will use only one byte in a single-byte character
set,^1 but a VARCHAR(1) would use two bytes because of the length byte.

This behavior can be a little confusing, so we’ll illustrate with an example. First, we
create a table with a single CHAR(10) column and store some values in it:

```
mysql> CREATE TABLE char_test( char_col CHAR(10));
mysql> INSERT INTO char_test(char_col) VALUES
-> ('string1'), (' string2'), ('string3 ');
```
When we retrieve the values, the trailing spaces have been stripped away:

```
mysql> SELECT CONCAT("'", char_col, "'") FROM char_test;
+----------------------------+
| CONCAT("'", char_col, "'") |
+----------------------------+
| 'string1' |
| ' string2' |
| 'string3' |
+----------------------------+
```
If we store the same values into a VARCHAR(10) column, we get the following result upon
retrieval:

1. Remember that the length is specified in characters, not bytes. A multibyte character set can require more
    than one byte to store each character.

**120 | Chapter 4: Optimizing Schema and Data Types**


```
mysql> SELECT CONCAT("'", varchar_col, "'") FROM varchar_test;
+-------------------------------+
| CONCAT("'", varchar_col, "'") |
+-------------------------------+
| 'string1' |
| ' string2' |
| 'string3 ' |
+-------------------------------+
```
How data is stored is up to the storage engines, and not all storage engines handle fixed-
length and variable-length data the same way. The Memory storage engine uses fixed-
size rows, so it has to allocate the maximum possible space for each value even when
it’s a variable-length field.^2 However, the padding and trimming behavior is consistent
across storage engines, because the MySQL server itself handles that.

The sibling types for CHAR and VARCHAR are BINARY and VARBINARY, which store binary
strings. Binary strings are very similar to conventional strings, but they store bytes
instead of characters. Padding is also different: MySQL pads BINARY values with \0 (the
zero byte) instead of spaces and doesn’t strip the pad value on retrieval.^3

These types are useful when you need to store binary data and want MySQL to compare
the values as bytes instead of characters. The advantage of byte-wise comparisons is
more than just a matter of case insensitivity. MySQL literally compares BINARY strings
one byte at a time, according to the numeric value of each byte. As a result, binary
comparisons can be much simpler than character comparisons, so they are faster.

```
Generosity Can Be Unwise
Storing the value 'hello' requires the same amount of space in a VARCHAR(5) and a
VARCHAR(200) column. Is there any advantage to using the shorter column?
As it turns out, there is a big advantage. The larger column can use much more memory,
because MySQL often allocates fixed-size chunks of memory to hold values internally.
This is especially bad for sorting or operations that use in-memory temporary tables.
The same thing happens with filesorts that use on-disk temporary tables.
The best strategy is to allocate only as much space as you really need.
```
**BLOB and TEXT types**

BLOB and TEXT are string data types designed to store large amounts of data as either
binary or character strings, respectively.

2. The Memory engine in Percona Server supports variable-length rows.
3.Be careful with the BINARY type if the value must remain unchanged after retrieval. MySQL will pad it to
    the required length with \0s.

```
Choosing Optimal Data Types | 121
```

In fact, they are each families of data types: the character types are TINYTEXT, SMALL
TEXT, TEXT, MEDIUMTEXT, and LONGTEXT, and the binary types are TINYBLOB, SMALLBLOB,
BLOB, MEDIUMBLOB, and LONGBLOB. BLOB is a synonym for SMALLBLOB, and TEXT is a synonym
for SMALLTEXT.

Unlike with all other data types, MySQL handles each BLOB and TEXT value as an object
with its own identity. Storage engines often store them specially; InnoDB may use a
separate “external” storage area for them when they’re large. Each value requires from
one to four bytes of storage space in the row and enough space in external storage to
actually hold the value.

The only difference between the BLOB and TEXT families is that BLOB types store binary
data with no collation or character set, but TEXT types have a character set and collation.

MySQL sorts BLOB and TEXT columns differently from other types: instead of sorting the
full length of the string, it sorts only the first max_sort_length bytes of such columns.
If you need to sort by only the first few characters, you can either decrease the
max_sort_length server variable or use ORDER BY SUBSTRING( _column, length_ ).

MySQL can’t index the full length of these data types and can’t use the indexes for
sorting. (You’ll find more on these topics in the next chapter.)

```
On-Disk Temporary Tables and Sort Files
Because the Memory storage engine doesn’t support the BLOB and TEXT types, queries
that use BLOB or TEXT columns and need an implicit temporary table will have to use on-
disk MyISAM temporary tables, even for only a few rows. (Percona Server’s Memory
storage engine supports the BLOB and TEXT types, but at the time of writing, it doesn’t
yet prevent on-disk tables from being used.)
This can result in a serious performance overhead. Even if you configure MySQL to
store temporary tables on a RAM disk, many expensive operating system calls will be
required.
The best solution is to avoid using the BLOB and TEXT types unless you really need them.
If you can’t avoid them, you may be able to use the SUBSTRING( column, length ) trick
everywhere a BLOB column is mentioned (including in the ORDER BY clause) to convert
the values to character strings, which will permit in-memory temporary tables. Just be
sure that you’re using a short enough substring that the temporary table doesn’t grow
larger than max_heap_table_size or tmp_table_size, or MySQL will convert the table
to an on-disk MyISAM table.
The worst-case length allocation also applies to sorting of values, so this trick can help
with both kinds of problems: creating large temporary tables and sort files, and creating
them on disk.
```
**122 | Chapter 4: Optimizing Schema and Data Types**


```
Here’s an example. Suppose you have a table with 10 million rows, which uses a couple
of gigabytes on disk. It has a VARCHAR(1000) column with the utf8 character set. This
can use up to 3 bytes per character, for a worst-case size of 3,000 bytes. If you mention
this column in your ORDER BY clause, a query against the whole table can require over
30 GB of temporary space just for the sort files!
If the Extra column of EXPLAIN contains “Using temporary,” the query uses an implicit
temporary table.
```
**Using ENUM instead of a string type**

Sometimes you can use an ENUM column instead of conventional string types. An ENUM
column can store a predefined set of distinct string values. MySQL stores them very
compactly, packed into one or two bytes depending on the number of values in the list.
It stores each value internally as an integer representing its position in the field defini-
tion list, and it keeps the “lookup table” that defines the number-to-string correspond-
ence in the table’s _.frm_ file. Here’s an example:

```
mysql> CREATE TABLE enum_test(
-> e ENUM('fish', 'apple', 'dog') NOT NULL
-> );
mysql> INSERT INTO enum_test(e) VALUES('fish'), ('dog'), ('apple');
```
The three rows actually store integers, not strings. You can see the dual nature of the
values by retrieving them in a numeric context:

```
mysql> SELECT e + 0 FROM enum_test;
+-------+
| e + 0 |
+-------+
| 1 |
| 3 |
| 2 |
+-------+
```
This duality can be terribly confusing if you specify numbers for your ENUM constants,
as in ENUM('1', '2', '3'). We suggest you don’t do this.

Another surprise is that an ENUM field sorts by the internal integer values, not by the
strings themselves:

```
mysql> SELECT e FROM enum_test ORDER BY e;
+-------+
| e |
+-------+
| fish |
| apple |
| dog |
+-------+
```
```
Choosing Optimal Data Types | 123
```

You can work around this by specifying ENUM members in the order in which you want
them to sort. You can also use FIELD() to specify a sort order explicitly in your queries,
but this prevents MySQL from using the index for sorting:

```
mysql> SELECT e FROM enum_test ORDER BY FIELD(e, 'apple', 'dog', 'fish');
+-------+
| e |
+-------+
| apple |
| dog |
| fish |
+-------+
```
If we’d defined the values in alphabetical order, we wouldn’t have needed to do that.

The biggest downside of ENUM is that the list of strings is fixed, and adding or removing
strings requires the use of ALTER TABLE. Thus, it might not be a good idea to use ENUM
as a string data type when the list of allowed string values is likely to change arbitrarily
in the future, unless it’s acceptable to add them at the end of the list, which can be done
without a full rebuild of the table in MySQL 5.1.

Because MySQL stores each value as an integer and has to do a lookup to convert it to
its string representation, ENUM columns have some overhead. This is usually offset by
their smaller size, but not always. In particular, it can be slower to join a CHAR or
VARCHAR column to an ENUM column than to another CHAR or VARCHAR column.

To illustrate, we benchmarked how quickly MySQL performs such a join on a table in
one of our applications. The table has a fairly wide primary key:

```
CREATE TABLE webservicecalls (
day date NOT NULL,
account smallint NOT NULL,
service varchar(10) NOT NULL,
method varchar(50) NOT NULL,
calls int NOT NULL,
items int NOT NULL,
time float NOT NULL,
cost decimal(9,5) NOT NULL,
updated datetime,
PRIMARY KEY (day, account, service, method)
) ENGINE=InnoDB;
```
The table contains about 110,000 rows and is only about 10 MB, so it fits entirely in
memory. The service column contains 5 distinct values with an average length of 4
characters, and the method column contains 71 values with an average length of 20
characters.

We made a copy of this table and converted the service and method columns to ENUM,
as follows:

```
CREATE TABLE webservicecalls_enum (
... omitted ...
service ENUM(...values omitted...) NOT NULL,
method ENUM(...values omitted...) NOT NULL,
```
**124 | Chapter 4: Optimizing Schema and Data Types**


```
... omitted ...
) ENGINE=InnoDB;
```
We then measured the performance of joining the tables by the primary key columns.
Here is the query we used:

```
mysql> SELECT SQL_NO_CACHE COUNT(*)
-> FROM webservicecalls
-> JOIN webservicecalls USING(day, account, service, method);
```
We varied this query to join the VARCHAR and ENUM columns in different combinations.
Table 4-1 shows the results.

_Table 4-1. Speed of joining VARCHAR and ENUM columns_

```
Test Queries per second
VARCHAR joined to VARCHAR 2.6
VARCHAR joined to ENUM 1.7
ENUM joined to VARCHAR 1.8
ENUM joined to ENUM 3.5
```
The join is faster after converting the columns to ENUM, but joining the ENUM columns to
VARCHAR columns is slower. In this case, it looks like a good idea to convert these col-
umns, as long as they don’t have to be joined to VARCHAR columns. It’s a common design
practice to use “lookup tables” with integer primary keys to avoid using character-based
values in joins.

However, there’s another benefit to converting the columns: according to the Data_
length column from SHOW TABLE STATUS, converting these two columns to ENUM made
the table about 1/3 smaller. In some cases, this might be beneficial even if the ENUM
columns have to be joined to VARCHAR columns. Also, the primary key itself is only about
half the size after the conversion. Because this is an InnoDB table, if there are any other
indexes on this table, reducing the primary key size will make them much smaller, too.
We explain this in the next chapter.

#### Date and Time Types

MySQL has many types for various kinds of date and time values, such as YEAR and
DATE. The finest granularity of time MySQL can store is one second. (MariaDB has
microsecond-granularity temporal types.) However, it can do temporal _computations_
with microsecond granularity, and we’ll show you how to work around the storage
limitations.

Most of the temporal types have no alternatives, so there is no question of which one
is the best choice. The only question is what to do when you need to store both the
date and the time. MySQL offers two very similar data types for this purpose: DATE

```
Choosing Optimal Data Types | 125
```

TIME and TIMESTAMP. For many applications, either will work, but in some cases, one
works better than the other. Let’s take a look:

DATETIME
This type can hold a large range of values, from the year 1001 to the year 9999,
with a precision of one second. It stores the date and time packed into an integer
in YYYYMMDDHHMMSS format, independent of time zone. This uses eight bytes
of storage space.
By default, MySQL displays DATETIME values in a sortable, unambiguous format,
such as 2008-01-16 22:37:08. This is the ANSI standard way to represent dates
and times.

TIMESTAMP
As its name implies, the TIMESTAMP type stores the number of seconds elapsed since
midnight, January 1, 1970, Greenwich Mean Time (GMT)—the same as a Unix
timestamp. TIMESTAMP uses only four bytes of storage, so it has a much smaller
range than DATETIME: from the year 1970 to partway through the year 2038. MySQL
provides the FROM_UNIXTIME() and UNIX_TIMESTAMP() functions to convert a Unix
timestamp to a date, and vice versa.
MySQL 4.1 and newer versions format TIMESTAMP values just like DATETIME values,
but MySQL 4.0 and older versions display them without any punctuation between
the parts. This is only a display formatting difference; the TIMESTAMP storage format
is the same in all MySQL versions.
The value a TIMESTAMP displays also depends on the time zone. The MySQL server,
operating system, and client connections all have time zone settings.
Thus, a TIMESTAMP that stores the value 0 actually displays it as 1969-12-31 19:00:00
in Eastern Standard Time (EST), which has a five-hour offset from GMT. It’s worth
emphasizing this difference: if you store or access data from multiple time zones,
the behavior of TIMESTAMP and DATETIME will be very different. The former preserves
values relative to the time zone in use, while the latter preserves the textual repre-
sentation of the date and time.
TIMESTAMP also has special properties that DATETIME doesn’t have. By default,
MySQL will set the first TIMESTAMP column to the current time when you insert a
row without specifying a value for the column.^4 MySQL also updates the first
TIMESTAMP column’s value by default when you update the row, unless you assign
a value explicitly in the UPDATE statement. You can configure the insertion and
update behaviors for any TIMESTAMP column. Finally, TIMESTAMP columns are NOT
NULL by default, which is different from every other data type.

4. The rules for TIMESTAMP behavior are complex and have changed in various MySQL versions, so you should
    verify that you are getting the behavior you want. It’s usually a good idea to examine the output of SHOW
    CREATE TABLE after making changes to TIMESTAMP columns.

**126 | Chapter 4: Optimizing Schema and Data Types**


Special behavior aside, in general if you can use TIMESTAMP you should, because it is
more space-efficient than DATETIME. Sometimes people store Unix timestamps as integer
values, but this usually doesn’t gain you anything. The integer format is often less
convenient to deal with, so we do not recommend doing this.

What if you need to store a date and time value with subsecond resolution? MySQL
currently does not have an appropriate data type for this, but you can use your own
storage format: you can use the BIGINT data type and store the value as a timestamp in
microseconds, or you can use a DOUBLE and store the fractional part of the second after
the decimal point. Both approaches will work well. Or you can use MariaDB instead
of MySQL.

#### Bit-Packed Data Types

MySQL has a few storage types that use individual bits within a value to store data
compactly. All of these types are technically string types, regardless of the underlying
storage format and manipulations:

BIT
Before MySQL 5.0, BIT is just a synonym for TINYINT. But in MySQL 5.0 and newer,
it’s a completely different data type with special characteristics. We discuss the
new behavior here.
You can use a BIT column to store one or many true/false values in a single column.
BIT(1) defines a field that contains a single bit, BIT(2) stores 2 bits, and so on; the
maximum length of a BIT column is 64 bits.
BIT behavior varies between storage engines. MyISAM packs the columns together
for storage purposes, so 17 individual BIT columns require only 17 bits to store
(assuming none of the columns permits NULL). MyISAM rounds that to three bytes
for storage. Other storage engines, such as Memory and InnoDB, store each column
as the smallest integer type large enough to contain the bits, so you don’t save any
storage space.
MySQL treats BIT as a string type, not a numeric type. When you retrieve a BIT
(1) value, the result is a string but the contents are the binary value 0 or 1 , not the
ASCII value “0” or “1”. However, if you retrieve the value in a numeric context,
the result is the number to which the bit string converts. Keep this in mind if you
need to compare the result to another value. For example, if you store the value
b'00111001' (which is the binary equivalent of 57) into a BIT(8) column and retrieve
it, you will get the string containing the character code 57. This happens to be the
ASCII character code for “9”. But in a numeric context, you’ll get the value 57 :
mysql> **CREATE TABLE bittest(a bit(8));**
mysql> **INSERT INTO bittest VALUES(b'00111001');**
mysql> **SELECT a, a + 0 FROM bittest;**

```
Choosing Optimal Data Types | 127
```

```
+------+-------+
| a | a + 0 |
+------+-------+
| 9 | 57 |
+------+-------+
This can be very confusing, so we recommend that you use BIT with caution. For
most applications, we think it is a better idea to avoid this type.
If you want to store a true/false value in a single bit of storage space, another option
is to create a nullable CHAR(0) column. This column is capable of storing either the
absence of a value (NULL) or a zero-length value (the empty string).
```
SET
If you need to store many true/false values, consider combining many columns into
one with MySQL’s native SET data type, which MySQL represents internally as a
packed set of bits. It uses storage efficiently, and MySQL has functions such as
FIND_IN_SET() and FIELD() that make it easy to use in queries. The major drawback
is the cost of changing the column’s definition: this requires an ALTER TABLE, which
is very expensive on large tables (but see the workaround later in this chapter). In
general, you also can’t use indexes for lookups on SET columns.

_Bitwise operations on integer columns_
An alternative to SET is to use an integer as a packed set of bits. For example, you
can pack eight bits in a TINYINT and manipulate them with bitwise operators. You
can make this easier by defining named constants for each bit in your application
code.
The major advantage of this approach over SET is that you can change the “enu-
meration” the field represents without an ALTER TABLE. The drawback is that your
queries are harder to write and understand (what does it mean when bit 5 is set?).
Some people are comfortable with bitwise manipulations and some aren’t, so
whether you’ll want to try this technique is largely a matter of taste.

An example application for packed bits is an access control list (ACL) that stores per-
missions. Each bit or SET element represents a value such as CAN_READ, CAN_WRITE, or
CAN_DELETE. If you use a SET column, you’ll let MySQL store the bit-to-value mapping
in the column definition; if you use an integer column, you’ll store the mapping in your
application code. Here’s what the queries would look like with a SET column:

```
mysql> CREATE TABLE acl (
-> perms SET('CAN_READ', 'CAN_WRITE', 'CAN_DELETE') NOT NULL
-> );
mysql> INSERT INTO acl(perms) VALUES ('CAN_READ,CAN_DELETE');
mysql> SELECT perms FROM acl WHERE FIND_IN_SET('AN_READ', perms);
+---------------------+
| perms |
+---------------------+
| CAN_READ,CAN_DELETE |
+---------------------+
```
If you used an integer, you could write that example as follows:

**128 | Chapter 4: Optimizing Schema and Data Types**


```
mysql> SET @CAN_READ := 1 << 0,
-> @CAN_WRITE := 1 << 1,
-> @CAN_DELETE := 1 << 2;
mysql> CREATE TABLE acl (
-> perms TINYINT UNSIGNED NOT NULL DEFAULT 0
-> );
mysql> INSERT INTO acl(perms) VALUES(@CAN_READ + @CAN_DELETE);
mysql> SELECT perms FROM acl WHERE perms & @CAN_READ;
+-------+
| perms |
+-------+
| 5 |
+-------+
```
We’ve used variables to define the values, but you can use constants in your code
instead.

#### Choosing Identifiers

Choosing a good data type for an identifier column is very important. You’re more
likely to compare these columns to other values (for example, in joins) and to use them
for lookups than other columns. You’re also likely to use them in other tables as foreign
keys, so when you choose a data type for an identifier column, you’re probably choosing
the type in related tables as well. (As we demonstrated earlier in this chapter, it’s a good
idea to use the same data types in related tables, because you’re likely to use them for
joins.)

When choosing a type for an identifier column, you need to consider not only the
storage type, but also how MySQL performs computations and comparisons on that
type. For example, MySQL stores ENUM and SET types internally as integers but converts
them to strings when doing comparisons in a string context.

Once you choose a type, make sure you use the same type in all related tables. The
types should match exactly, including properties such as UNSIGNED.^5 Mixing different
data types can cause performance problems, and even if it doesn’t, implicit type con-
versions during comparisons can create hard-to-find errors. These may even crop up
much later, after you’ve forgotten that you’re comparing different data types.

Choose the smallest size that can hold your required range of values, and leave room
for future growth if necessary. For example, if you have a state_id column that stores
US state names, you don’t need thousands or millions of values, so don’t use an INT.
A TINYINT should be sufficient and is three bytes smaller. If you use this value as a foreign
key in other tables, three bytes can make a big difference. Here are a few tips:

5. If you’re using the InnoDB storage engine, you may not be able to create foreign keys unless the data
    types match exactly. The resulting error message, “ERROR 1005 (HY000): Can’t create table,” can be
    confusing depending on the context, and questions about it come up often on MySQL mailing lists.
    (Oddly, you can create foreign keys between VARCHAR columns of different lengths.)

```
Choosing Optimal Data Types | 129
```

_Integer types_
Integers are usually the best choice for identifiers, because they’re fast and they
work with AUTO_INCREMENT.

ENUM _and_ SET
The ENUM and SET types are generally a poor choice for identifiers, though they can
be okay for static “definition tables” that contain status or “type” values. ENUM and
SET columns are appropriate for holding information such as an order’s status, a
product’s type, or a person’s gender.
As an example, if you use an ENUM field to define a product’s type, you might want
a lookup table primary keyed on an identical ENUM field. (You could add columns
to the lookup table for descriptive text, to generate a glossary, or to provide mean-
ingful labels in a pull-down menu on a website.) In this case, you’ll want to use the
ENUM as an identifier, but for most purposes you should avoid doing so.

_String types_
Avoid string types for identifiers if possible, because they take up a lot of space and
are generally slower than integer types. Be especially cautious when using string
identifiers with MyISAM tables. MyISAM uses packed indexes for strings by de-
fault, which can make lookups much slower. In our tests, we’ve noted up to six
times slower performance with packed indexes on MyISAM.
You should also be very careful with completely “random” strings, such as those
produced by MD5(), SHA1(), or UUID(). Each new value you generate with them will
be distributed in arbitrary ways over a large space, which can slow INSERT and some
types of SELECT queries:^6

- They slow INSERT queries because the inserted value has to go in a random
    location in indexes. This causes page splits, random disk accesses, and clus-
    tered index fragmentation for clustered storage engines. More about this in
    the next chapter.
- They slow SELECT queries because logically adjacent rows will be widely dis-
    persed on disk and in memory.
- Random values cause caches to perform poorly for all types of queries because
    they defeat locality of reference, which is how caching works. If the entire
    dataset is equally “hot,” there is no advantage to having any particular part of
    the data cached in memory, and if the working set does not fit in memory, the
    cache will have a lot of flushes and misses.

If you do store UUID values, you should remove the dashes or, even better, convert the
UUID values to 16-byte numbers with UNHEX() and store them in a BINARY(16) column.
You can retrieve the values in hexadecimal format with the HEX() function.

6. On the other hand, for some very large tables with many writers, such pseudorandom values can actually
    help eliminate “hot spots.”

**130 | Chapter 4: Optimizing Schema and Data Types**


Values generated by UUID() have different characteristics from those generated by a
cryptographic hash function such as SHA1(): the UUID values are unevenly distributed
and are somewhat sequential. They’re still not as good as a monotonically increasing
integer, though.

```
Beware of Autogenerated Schemas
We’ve covered the most important data type considerations (some with serious and
others with more minor performance implications), but we haven’t yet told you about
the evils of autogenerated schemas.
Badly written schema migration programs and programs that autogenerate schemas
can cause severe performance problems. Some programs use large VARCHAR fields for
everything , or use different data types for columns that will be compared in joins. Be
sure to double-check a schema if it was created for you automatically.
Object-relational mapping (ORM) systems (and the “frameworks” that use them) are
another frequent performance nightmare. Some of these systems let you store any type
of data in any type of backend data store, which usually means they aren’t designed to
use the strengths of any of the data stores. Sometimes they store each property of each
object in a separate row, even using timestamp-based versioning, so there are multiple
versions of each property!
This design may appeal to developers, because it lets them work in an object-oriented
fashion without needing to think about how the data is stored. However, applications
that “hide complexity from developers” usually don’t scale well. We suggest you think
carefully before trading performance for developer productivity, and always test on a
realistically large dataset, so you don’t discover performance problems too late.
```
#### Special Types of Data

Some kinds of data don’t correspond directly to the available built-in types. A time-
stamp with subsecond resolution is one example; we showed you some options for
storing such data earlier in the chapter.

Another example is an IPv4 address. People often use VARCHAR(15) columns to store IP
addresses. However, they are really unsigned 32-bit integers, not strings. The dotted-
quad notation is just a way of writing it out so that humans can read it more easily. You
should store IP addresses as unsigned integers. MySQL provides the INET_ATON() and
INET_NTOA() functions to convert between the two representations.

### Schema Design Gotchas in MySQL

Although there are universally bad and good design principles, there are also issues that
arise from how MySQL is implemented, and that means you can make MySQL-specific
mistakes, too. This section discusses problems that we’ve observed in schema designs

```
Schema Design Gotchas in MySQL | 131
```

with MySQL. It might help you avoid those mistakes and choose alternatives that work
better with MySQL’s specific implementation.

_Too many columns_
MySQL’s storage engine API works by copying rows between the server and the
storage engine in a row buffer format; the server then decodes the buffer into col-
umns. But it can be costly to turn the row buffer into the row data structure with
the decoded columns. MyISAM’s fixed row format actually matches the server’s
row format exactly, so no conversion is needed. However, MyISAM’s variable row
format and InnoDB’s row format always require conversion. The cost of this con-
version depends on the number of columns. We discovered that this can become
expensive when we investigated an issue with high CPU consumption for a cus-
tomer with extremely wide tables (hundreds of columns), even though only a few
columns were actually used. If you’re planning for hundreds of columns, be aware
that the server’s performance characteristics will be a bit different.

_Too many joins_
The so-called entity-attribute-value (EAV) design pattern is a classic case of a uni-
versally bad design pattern that especially doesn’t work well in MySQL. MySQL
has a limitation of 61 tables per join, and EAV databases require many self-joins.
We’ve seen more than a few EAV databases eventually exceed this limit. Even at
many fewer joins than 61, however, the cost of planning and optimizing the query
can become problematic for MySQL. As a rough rule of thumb, it’s better to have
a dozen or fewer tables per query if you need queries to execute very fast with high
concurrency.

_The all-powerful_ ENUM
Beware of overusing ENUM. Here’s an example we saw:
CREATE TABLE ... (
country enum('','0','1','2',...,'31')
The schema was sprinkled liberally with this pattern. This would probably be a
questionable design decision in any database with an enumerated value type, be-
cause it really should be an integer that is foreign-keyed to a “dictionary” or
“lookup” table anyway. But in MySQL, you can’t add a new country to the list
without an ALTER TABLE, which is a blocking operation in MySQL 5.0 and earlier,
and even in 5.1 and newer if you add the value anywhere but at the end of the list.
(We’ll show some hacks to address this later, but they’re just hacks.)

_The_ ENUM _in disguise_
An ENUM permits the column to hold one value from a set of defined values. A SET
permits the column to hold one or more values from a set of defined values. Some-
times these can be easy to confuse. Here’s an example:
CREATE TABLE ...(
is_default set('Y','N') NOT NULL default 'N'

**132 | Chapter 4: Optimizing Schema and Data Types**


```
That almost surely ought to be an ENUM instead of a SET, assuming that it can’t be
both true and false at the same time.
```
NULL _not invented here_
We wrote earlier about the benefits of avoiding NULL, and indeed we suggest con-
sidering alternatives when possible. Even when you do need to store a “no value”
fact in a table, you might not need to use NULL. Perhaps you can use zero, a special
value, or an empty string instead.
However, you can take this to extremes. Don’t be too afraid of using NULL when
you need to represent an unknown value. In some cases, it’s better to use NULL than
a magical constant. Selecting one value from the domain of a constrained type,
such as using −1 to represent an unknown integer, can complicate your code a lot,
introduce bugs, and just generally make a total mess out of things. Handling
NULL isn’t always easy, but it’s often better than the alternative.
Here’s one example we’ve seen pretty frequently:
CREATE TABLE ... (
dt DATETIME NOT NULL DEFAULT '0000-00-00 00:00:00'
That bogus all-zeros value can cause lots of problems. (You can configure MySQL’s
SQL_MODE to disallow nonsense dates, which is an especially good practice for a new
application that hasn’t yet created a database full of bad data.)
On a related topic, MySQL does index NULLs, unlike Oracle, which doesn’t include
non-values in indexes.

### Normalization and Denormalization

There are usually many ways to represent any given data, ranging from fully normalized
to fully denormalized and anything in between. In a normalized database, each fact is
represented once and only once. Conversely, in a denormalized database, information
is duplicated, or stored in multiple places.

If you’re not familiar with normalization, you should study it. There are many good
books on the topic and resources online; here, we just give a brief introduction to the
aspects you need to know for this chapter. Let’s start with the classic example of em-
ployees, departments, and department heads:

```
EMPLOYEE DEPARTMENT HEAD
Jones Accounting Jones
Smith Engineering Smith
Brown Accounting Jones
Green Engineering Smith
```
```
Normalization and Denormalization | 133
```

The problem with this schema is that inconsistencies can occur while the data is being
modified. Say Brown takes over as the head of the Accounting department. We need
to update multiple rows to reflect this change, and that’s a pain and introduces oppor-
tunities for error. If the “Jones” row says the head of the department is something
different from the “Brown” row, there’s no way to know which is right. It’s like the old
saying, “A person with two watches never knows what time it is.” Furthermore, we
can’t represent a department without employees—if we delete all employees in the
Accounting department, we lose all records about the department itself. To avoid these
problems, we need to normalize the table by separating the employee and department
entities. This process results in the following two tables for employees:

```
EMPLOYEE_NAME DEPARTMENT
Jones Accounting
Smith Engineering
Brown Accounting
Green Engineering
```
and departments:

```
DEPARTMENT HEAD
Accounting Jones
Engineering Smith
```
These tables are now in second normal form, which is good enough for many purposes.
However, second normal form is only one of many possible normal forms.

```
We’re using the last name as the primary key here for purposes of illus-
tration, because it’s the “natural identifier” of the data. In practice,
however, we wouldn’t do that. It’s not guaranteed to be unique, and it’s
usually a bad idea to use a long string for a primary key.
```
#### Pros and Cons of a Normalized Schema

People who ask for help with performance issues are frequently advised to normalize
their schemas, especially if the workload is write-heavy. This is often good advice. It
works well for the following reasons:

- Normalized updates are usually faster than denormalized updates.
- When the data is well normalized, there’s little or no duplicated data, so there’s
    less data to change.
- Normalized tables are usually smaller, so they fit better in memory and perform
    better.

**134 | Chapter 4: Optimizing Schema and Data Types**


- The lack of redundant data means there’s less need for DISTINCT or GROUP BY queries
    when retrieving lists of values. Consider the preceding example: it’s impossible
    to get a distinct list of departments from the denormalized schema without DIS
    TINCT or GROUP BY, but if DEPARTMENT is a separate table, it’s a trivial query.

The drawbacks of a normalized schema usually have to do with retrieval. Any nontrivial
query on a well-normalized schema will probably require at least one join, and perhaps
several. This is not only expensive, but it can make some indexing strategies impossible.
For example, normalizing may place columns in different tables that would benefit
from belonging to the same index.

#### Pros and Cons of a Denormalized Schema

A denormalized schema works well because everything is in the same table, which
avoids joins.

If you don’t need to join tables, the worst case for most queries—even the ones that
don’t use indexes—is a full table scan. This can be much faster than a join when the
data doesn’t fit in memory, because it avoids random I/O.

A single table can also allow more efficient indexing strategies. Suppose you have a
website where users post their messages, and some users are premium users. Now say
you want to view the last 10 messages from premium users. If you’ve normalized the
schema and indexed the publishing dates of the messages, the query might look like
this:

```
mysql> SELECT message_text, user_name
-> FROM message
-> INNER JOIN user ON message.user_id=user.id
-> WHERE user.account_type='premiumv
-> ORDER BY message.published DESC LIMIT 10;
```
To execute this query efficiently, MySQL will need to scan the published index on
the message table. For each row it finds, it will need to probe into the user table and
check whether the user is a premium user. This is inefficient if only a small fraction of
users have premium accounts.

The other possible query plan is to start with the user table, select all premium users,
get all messages for them, and do a filesort. This will probably be even worse.

The problem is the join, which is keeping you from sorting and filtering simultaneously
with a single index. If you denormalize the data by combining the tables and add an
index on (account_type, published), you can write the query without a join. This will
be very efficient:

```
mysql> SELECT message_text,user_name
-> FROM user_messages
-> WHERE account_type='premium'
-> ORDER BY published DESC
-> LIMIT 10;
```
```
Normalization and Denormalization | 135
```

#### A Mixture of Normalized and Denormalized

Given that both normalized and denormalized schemas have benefits and drawbacks,
how can you choose the best design?

The truth is, fully normalized and fully denormalized schemas are like laboratory rats:
they usually have little to do with the real world. In the real world, you often need to
mix the approaches, possibly using a partially normalized schema, cache tables, and
other techniques.

The most common way to denormalize data is to duplicate, or cache, selected columns
from one table in another table. In MySQL 5.0 and newer, you can use triggers to update
the cached values, which makes the implementation easier.

In our website example, for instance, instead of denormalizing fully you can store
account_type in both the user and message tables. This avoids the insert and delete
problems that come with full denormalization, because you never lose information
about the user, even when there are no messages. It won’t make the user_message table
much larger, but it will let you select the data efficiently.

However, it’s now more expensive to update a user’s account type, because you have
to change it in both tables. To see whether that’s a problem, you must consider how
frequently you’ll have to make such changes and how long they will take, compared to
how often you’ll run the SELECT query.

Another good reason to move some data from the parent table to the child table is for
sorting. For example, it would be extremely expensive to sort messages by the author’s
name on a normalized schema, but you can perform such a sort very efficiently if you
cache the author_name in the message table and index it.

It can also be useful to cache derived values. If you need to display how many messages
each user has posted (as many forums do), either you can run an expensive subquery
to count the data every time you display it, or you can have a num_messages column in
the user table that you update whenever a user posts a new message.

### Cache and Summary Tables

Sometimes the best way to improve performance is to keep redundant data in the same
table as the data from which it was derived. However, sometimes you’ll need to build
completely separate summary or cache tables, specially tuned for your retrieval needs.
This approach works best if you can tolerate slightly stale data, but sometimes you
really don’t have a choice (for instance, when you need to avoid complex and expensive
real-time updates).

The terms “cache table” and “summary table” don’t have standardized meanings. We
use the term “cache tables” to refer to tables that contain data that can be easily, if more
slowly, retrieved from the schema (i.e., data that is logically redundant). When we say

**136 | Chapter 4: Optimizing Schema and Data Types**


“summary tables,” we mean tables that hold aggregated data from GROUP BY queries
(i.e., data that is not logically redundant). Some people also use the term “roll-up tables”
for these tables, because the data has been “rolled up.”

Staying with the website example, suppose you need to count the number of messages
posted during the previous 24 hours. It would be impossible to maintain an accurate
real-time counter on a busy site. Instead, you could generate a summary table every
hour. You can often do this with a single query, and it’s more efficient than maintaining
counters in real time. The drawback is that the counts are not 100% accurate.

If you need to get an accurate count of messages posted during the previous 24-hour
period (with no staleness), there is another option. Begin with a per-hour summary
table. You can then count the exact number of messages posted in a given 24-hour
period by adding the number of messages in the 23 whole hours contained in that
period, the partial hour at the beginning of the period, and the partial hour at the end
of the period. Suppose your summary table is called msg_per_hr and is defined
as follows:

```
CREATE TABLE msg_per_hr (
hr DATETIME NOT NULL,
cnt INT UNSIGNED NOT NULL,
PRIMARY KEY(hr)
);
```
You can find the number of messages posted in the previous 24 hours by adding the
results of the following three queries. We’re using LEFT(NOW(), 14) to round the current
date and time to the nearest hour:

```
mysql> SELECT SUM(cnt) FROM msg_per_hr
-> WHERE hr BETWEEN
-> CONCAT(LEFT(NOW(), 14), '00:00') - INTERVAL 23 HOUR
-> AND CONCAT(LEFT(NOW(), 14), '00:00') - INTERVAL 1 HOUR;
mysql> SELECT COUNT(*) FROM message
-> WHERE posted >= NOW() - INTERVAL 24 HOUR
-> AND posted < CONCAT(LEFT(NOW(), 14), '00:00') - INTERVAL 23 HOUR;
mysql> SELECT COUNT(*) FROM message
-> WHERE posted >= CONCAT(LEFT(NOW(), 14), '00:00');
```
Either approach—an inexact count or an exact count with small range queries to fill
in the gaps—is more efficient than counting all the rows in the message table. This is
the key reason for creating summary tables. These statistics are expensive to compute
in real time, because they require scanning a lot of data, or queries that will only run
efficiently with special indexes that you don’t want to add because of the impact they
will have on updates. Computing the most active users or the most frequent “tags” are
typical examples of such operations.

Cache tables, in turn, are useful for optimizing search and retrieval queries. These
queries often require a particular table and index structure that is different from the
one you would use for general online transaction processing (OLTP) operations.

```
Cache and Summary Tables | 137
```

For example, you might need many different index combinations to speed up various
types of queries. These conflicting requirements sometimes demand that you create a
cache table that contains only some of the columns from the main table. A useful tech-
nique is to use a different storage engine for the cache table. If the main table uses
InnoDB, for example, by using MyISAM for the cache table you’ll gain a smaller index
footprint and the ability to do full-text search queries. Sometimes you might even want
to take the table completely out of MySQL and into a specialized system that can search
more efficiently, such as the Lucene or Sphinx search engines.

When using cache and summary tables, you have to decide whether to maintain their
data in real time or with periodic rebuilds. Which is better will depend on your appli-
cation, but a periodic rebuild not only can save resources but also can result in a more
efficient table that’s not fragmented and has fully sorted indexes.

When you rebuild summary and cache tables, you’ll often need their data to remain
available during the operation. You can achieve this by using a “shadow table,” which
is a table you build “behind” the real table. When you’re done building it, you can swap
the tables with an atomic rename. For example, if you need to rebuild my_summary, you
can create my_summary_new, fill it with data, and swap it with the real table:

```
mysql> DROP TABLE IF EXISTS my_summary_new, my_summary_old;
mysql> CREATE TABLE my_summary_new LIKE my_summary;
-- populate my_summary_new as desired
mysql> RENAME TABLE my_summary TO my_summary_old, my_summary_new TO my_summary;
```
If you rename the original my_summary table my_summary_old before assigning the name
my_summary to the newly rebuilt table, as we’ve done here, you can keep the old version
until you’re ready to overwrite it at the next rebuild. It’s handy to have it for a quick
rollback if the new table has a problem.

#### Materialized Views

Many database management systems, such as Oracle or Microsoft SQL Server, offer a
feature called _materialized views_. These are views that are actually precomputed and
stored as tables on disk, and can be refreshed and updated through various strategies.
MySQL doesn’t support this natively (we’ll go into details about its support for views
in Chapter 7). However, you can implement materialized views yourself, using Justin
Swanhart’s open source Flexviews tools ( _[http://code.google.com/p/flexviews/](http://code.google.com/p/flexviews/)_ ). Flex-
views is more sophisticated than roll-your-own solutions and offers a lot of nice features
that make materialized views simpler to create and maintain. It consists of a few parts:

- A Change Data Capture (CDC) utility that reads the server’s binary logs and ex-
    tracts relevant changes to rows
- A set of stored procedures that help define and manage the view definitions
- Tools to apply the changes to the materialized data in the database

**138 | Chapter 4: Optimizing Schema and Data Types**


In contrast to typical methods of maintaining summary and cache tables, Flexviews
can recalculate the contents of the materialized view incrementally by extracting delta
changes to the source tables. This means it can update the view without needing to
query the source data. For example, if you create a summary table that counts groups
of rows, and you add a row to the source table, Flexviews simply increments the cor-
responding group by one. The same technique works for other aggregate functions,
such as SUM() and AVG(). It takes advantage of the fact that row-based binary logging
includes images of the rows before and after they are updated, so Flexviews can see not
only the new value of each row, but the delta from the previous version, without even
looking at the source table. Computing with deltas is much more efficient than reading
the data from the source table.

We don’t have space for a full exploration of how to use Flexviews, but we can give an
overview. You start by writing a SELECT statement that expresses the data you want to
derive from your existing database. This can include joins and aggregations (GROUP
BY). There’s a helper tool in Flexviews that transforms your SQL query into Flexviews
API calls. Then Flexviews does all the dirty work of watching changes to the database
and transforming them into updates to the tables that store your materialized view over
the original tables. Now your application can simply query the materialized view in-
stead of the tables from which it was derived.

Flexviews has good coverage of SQL, including tricky expressions that you might not
expect a tool to handle outside the server. That makes it useful for building views over
complex SQL expressions, so you can replace complex queries with simple, fast queries
against the materialized view.

#### Counter Tables

An application that keeps counts in a table can run into concurrency problems when
updating the counters. Such tables are very common in web applications. You can use
them to cache the number of friends a user has, the number of downloads of a file, and
so on. It’s often a good idea to build a separate table for the counters, to keep it small
and fast. Using a separate table can help you avoid query cache invalidations and lets
you use some of the more advanced techniques we show in this section.

To keep things as simple as possible, suppose you have a counter table with a single
row that just counts hits on your website:

```
mysql> CREATE TABLE hit_counter (
-> cnt int unsigned not null
-> ) ENGINE=InnoDB;
```
Each hit on the website updates the counter:

```
mysql> UPDATE hit_counter SET cnt = cnt + 1;
```
```
Cache and Summary Tables | 139
```

The problem is that this single row is effectively a global “mutex” for any transaction
that updates the counter. It will serialize those transactions. You can get higher con-
currency by keeping more than one row and updating a random row. This requires the
following change to the table:

```
mysql> CREATE TABLE hit_counter (
-> slot tinyint unsigned not null primary key,
-> cnt int unsigned not null
-> ) ENGINE=InnoDB;
```
Prepopulate the table by adding 100 rows to it. Now the query can just choose a random
slot and update it:

```
mysql> UPDATE hit_counter SET cnt = cnt + 1 WHERE slot = RAND() * 100;
```
To retrieve statistics, just use aggregate queries:

```
mysql> SELECT SUM(cnt) FROM hit_counter;
```
A common requirement is to start new counters every so often (for example, once a
day). If you need to do this, you can change the schema slightly:

```
mysql> CREATE TABLE daily_hit_counter (
-> day date not null,
-> slot tinyint unsigned not null,
-> cnt int unsigned not null,
-> primary key(day, slot)
-> ) ENGINE=InnoDB;
```
You don’t want to pregenerate rows for this scenario. Instead, you can use ON DUPLICATE
KEY UPDATE:

```
mysql> INSERT INTO daily_hit_counter(day, slot, cnt)
-> VALUES(CURRENT_DATE, RAND() * 100, 1)
-> ON DUPLICATE KEY UPDATE cnt = cnt + 1;
```
If you want to reduce the number of rows to keep the table smaller, you can write a
periodic job that merges all the results into slot 0 and deletes every other slot:

```
mysql> UPDATE daily_hit_counter as c
-> INNER JOIN (
-> SELECT day, SUM(cnt) AS cnt, MIN(slot) AS mslot
-> FROM daily_hit_counter
-> GROUP BY day
-> ) AS x USING(day)
-> SET c.cnt = IF(c.slot = x.mslot, x.cnt, 0),
-> c.slot = IF(c.slot = x.mslot, 0, c.slot);
mysql> DELETE FROM daily_hit_counter WHERE slot <> 0 AND cnt = 0;
```
**140 | Chapter 4: Optimizing Schema and Data Types**


```
Faster Reads, Slower Writes
You’ll often need extra indexes, redundant fields, or even cache and summary tables
to speed up read queries. These add work to write queries and maintenance jobs, but
this is still a technique you’ll see a lot when you design for high performance: you
amortize the cost of the slower writes by speeding up reads significantly.
However, this isn’t the only price you pay for faster read queries. You also increase
development complexity for both read and write operations.
```
### Speeding Up ALTER TABLE

MySQL’s ALTER TABLE performance can become a problem with very large tables.
MySQL performs most alterations by making an empty table with the desired new
structure, inserting all the data from the old table into the new one, and deleting the
old table. This can take a very long time, especially if you’re short on memory and the
table is large and has lots of indexes. Many people have experience with ALTER TABLE
operations that have taken hours or days to complete.

MySQL 5.1 and newer include support for some types of “online” operations that won’t
lock the table for the whole operation. Recent versions of InnoDB^7 also support build-
ing indexes by sorting, which makes building indexes much faster and results in a
compact index layout.

In general, most ALTER TABLE operations will cause interruption of service in MySQL.
We’ll show some techniques to work around this in a bit, but those are for special cases.
For the general case, you need to use either operational tricks such as swapping servers
around and performing the ALTER on servers that are not in production service, or a
“shadow copy” approach. The technique for a shadow copy is to build a new table with
the desired structure beside the existing one, and then perform a rename and drop to
swap the two. Tools can help with this: for example, the “online schema change” tools
from Facebook’s database operations team ( _https://launchpad.net/mysqlatfacebook_ ),
Shlomi Noach’s openark toolkit ( _[http://code.openark.org/](http://code.openark.org/)_ ), and Percona Toolkit ( _[http:](http:)
//www.percona.com/software/_ ). If you are using Flexviews (discussed in “Materialized
Views” on page 138), you can perform nonblocking schema changes with its CDC
utility too.

Not all ALTER TABLE operations cause table rebuilds. For example, you can change or
drop a column’s default value in two ways (one fast, and one slow). Say you want to
change a film’s default rental duration from three to five days. Here’s the expensive way:

```
mysql> ALTER TABLE sakila.film
-> MODIFY COLUMN rental_duration TINYINT(3) NOT NULL DEFAULT 5;
```
7. This applies to the so-called “InnoDB plugin,” which is the only version of InnoDB that exists anymore
    in MySQL 5.5 and newer versions. See Chapter 1 for the details on InnoDB’s release history.

```
Speeding Up ALTER TABLE | 141
```

SHOW STATUS shows that this statement does 1,000 handler reads and 1,000 inserts. In
other words, it copies the table to a new table, even though the column’s type, size,
and nullability haven’t changed.

In theory, MySQL could have skipped building a new table. The default value for the
column is actually stored in the table’s _.frm_ file, so you should be able to change it
without touching the table itself. MySQL doesn’t yet use this optimization, however;
any MODIFY COLUMN will cause a table rebuild.

You can change a column’s default with ALTER COLUMN,^8 though:

```
mysql> ALTER TABLE sakila.film
-> ALTER COLUMN rental_duration SET DEFAULT 5;
```
This statement modifies the _.frm_ file and leaves the table alone. As a result, it is very fast.

#### Modifying Only the .frm File

We’ve seen that modifying a table’s _.frm_ file is fast and that MySQL sometimes rebuilds
a table when it doesn’t have to. If you’re willing to take some risks, you can convince
MySQL to do several other types of modifications without rebuilding the table.

```
The technique we’re about to demonstrate is unsupported, undocu-
mented, and may not work. Use it at your own risk. We advise you to
back up your data first!
```
You can potentially do the following types of operations without a table rebuild:

- Remove (but not add) a column’s AUTO_INCREMENT attribute.
- Add, remove, or change ENUM and SET constants. If you remove a constant and
    some rows contain that value, queries will return the value as the empty string.

The basic technique is to create a _.frm_ file for the desired table structure and copy it
into the place of the existing table’s _.frm_ file, as follows:

1. Create an empty table with _exactly the same layout_ , except for the desired modifi-
    cation (such as added ENUM constants).
2. Execute FLUSH TABLES WITH READ LOCK. This will close all tables in use and prevent
    any tables from being opened.
3. Swap the _.frm_ files.
4. Execute UNLOCK TABLES to release the read lock.

As an example, let’s add a constant to the rating column in sakila.film. The current
column looks like this:

```
8.ALTER TABLE lets you modify columns with ALTER COLUMN, MODIFY COLUMN, and CHANGE COLUMN. All three
do different things.
```
**142 | Chapter 4: Optimizing Schema and Data Types**


```
mysql> SHOW COLUMNS FROM sakila.film LIKE 'rating';
+--------+------------------------------------+------+-----+---------+-------+
| Field | Type | Null | Key | Default | Extra |
+--------+------------------------------------+------+-----+---------+-------+
| rating | enum('G','PG','PG-13','R','NC-17') | YES | | G | |
+--------+------------------------------------+------+-----+---------+-------+
```
We’ll add a PG-14 rating for parents who are just a little bit more cautious about films:

```
mysql> CREATE TABLE sakila.film_new LIKE sakila.film;
mysql> ALTER TABLE sakila.film_new
-> MODIFY COLUMN rating ENUM('G','PG','PG-13','R','NC-17', 'PG-14')
-> DEFAULT 'G';
mysql> FLUSH TABLES WITH READ LOCK;
```
Notice that we’re adding the new value _at the end of the list of constants_. If we placed
it in the middle, after PG-13, we’d change the meaning of the existing data: existing R
values would become PG-14, NC-17 would become R, and so on.

Now we swap the _.frm_ files from the operating system’s command prompt:

```
/var/lib/mysql/sakila# mv film.frm film_tmp.frm
/var/lib/mysql/sakila# mv film_new.frm film.frm
/var/lib/mysql/sakila# mv film_tmp.frm film_new.frm
```
Back in the MySQL prompt, we can now unlock the table and see that the changes took
effect:

```
mysql> UNLOCK TABLES;
mysql> SHOW COLUMNS FROM sakila.film LIKE 'rating'\G
*************************** 1. row ***************************
Field: rating
Type: enum('G','PG','PG-13','R','NC-17','PG-14')
```
The only thing left to do is drop the table we created to help with the operation:

```
mysql> DROP TABLE sakila.film_new;
```
#### Building MyISAM Indexes Quickly

The usual trick for loading MyISAM tables efficiently is to disable keys, load the data,
and reenable the keys:

```
mysql> ALTER TABLE test.load_data DISABLE KEYS;
-- load the data
mysql> ALTER TABLE test.load_data ENABLE KEYS;
```
This works because it lets MyISAM delay building the keys until all the data is loaded,
at which point it can build the indexes by sorting. This is much faster and results in a
defragmented, compact index tree.^9

Unfortunately, it doesn’t work for unique indexes, because DISABLE KEYS applies only
to nonunique indexes. MyISAM builds unique indexes in memory and checks the

9. MyISAM will also build indexes by sorting when you use LOAD DATA INFILE and the table is empty.

```
Speeding Up ALTER TABLE | 143
```

uniqueness as it loads each row. Loading becomes extremely slow as soon as the
index’s size exceeds the available memory.

In modern versions of InnoDB, you can use an analogous technique that relies on
InnoDB’s fast online index creation capabilities. This calls for dropping all of the non-
unique indexes, adding the new column, and then adding back the indexes you drop-
ped. Percona Server supports doing this automatically.

As with the ALTER TABLE hacks in the previous section, you can speed up this process
if you’re willing to do a little more work and assume some risk. This can be useful for
loading data from backups, for example, when you already know all the data is valid
and there’s no need for uniqueness checks.

```
Again, this is an undocumented, unsupported technique. Use it at your
own risk, and back up your data first.
```
Here are the steps you’ll need to take:

```
1.Create a table of the desired structure, but without any indexes.
```
2. Load the data into the table to build the _.MYD_ file.
3. Create another empty table with the desired structure, this time including the in-
    dexes. This will create the _.frm_ and _.MYI_ files you need.
4. Flush the tables with a read lock.
5. Rename the second table’s _.frm_ and _.MYI_ files, so MySQL uses them for the first
    table.
6. Release the read lock.
7. Use REPAIR TABLE to build the table’s indexes. This will build all indexes by sorting,
    including the unique indexes.

This procedure can be much faster for very large tables.

**144 | Chapter 4: Optimizing Schema and Data Types**


### Summary

Good schema design is pretty universal, but of course MySQL has special implemen-
tation details to consider. In a nutshell, it’s a good idea to keep things as small and
simple as you can. MySQL likes simplicity, and so will the people who have to work
with your database:

- Try to avoid extremes in your design, such as a schema that will force enormously
    complex queries, or tables with oodles and oodles of columns. (An oodle is some-
    where between a scad and a gazillion.)
- Use small, simple, appropriate data types, and avoid NULL unless it’s actually the
    right way to model your data’s reality.
- Try to use the same data types to store similar or related values, especially if they’ll
    be used in a join condition.
- Watch out for variable-length strings, which might cause pessimistic full-length
    memory allocation for temporary tables and sorting.
- Try to use integers for identifiers if you can.
- Avoid the legacy MySQL-isms such as specifying precisions for floating-point
    numbers or display widths for integers.
- Be careful with ENUM and SET. They’re handy, but they can be abused, and they’re
    tricky sometimes. BIT is best avoided.

Normalization is good, but denormalization (duplication of data, in most cases) is
sometimes actually necessary and beneficial. We’ll see more examples of that in the
next chapter. And precomputing, caching, or generating summary tables can also be a
big win. Justin Swanhart’s Flexviews tool can help maintain summary tables.

Finally, ALTER TABLE can be painful because in most cases, it locks and rebuilds the
whole table. We showed a number of workarounds for specific cases; for the general
case, you’ll have to use other techniques, such as performing the ALTER on a replica and
then promoting it to master. There’s more about this later in the book.

```
Summary | 145
```
