

```
APPENDIX F
```
```
Using Sphinx with MySQL
```
Sphinx ( _[http://www.sphinxsearch.com](http://www.sphinxsearch.com)_ ) is a free, open source, full-text search engine,
designed from the ground up to integrate well with databases. It has DBMS-like fea-
tures, is very fast, supports distributed searching, and scales well. It is also designed for
efficient memory and disk I/O, which is important because they’re often the limiting
factors for large operations.

Sphinx works well with MySQL. It can be used to accelerate a variety of queries, in-
cluding full-text searches; you can also use it to perform fast grouping and sorting
operations, among other applications. It speaks MySQL’s wire protocol and a mostly
MySQL-compatible SQL-like dialect, so you can actually query it just like a MySQL
database. Additionally, there is a pluggable storage engine that lets a programmer or
administrator access Sphinx directly through MySQL. Sphinx is especially useful for
certain queries that MySQL’s general-purpose architecture doesn’t optimize very well
for large datasets in real-world settings. In short, Sphinx can enhance MySQL’s func-
tionality and performance.

The source of data for a Sphinx index is usually the result of a MySQL SELECT query,
but you can build an index from an unlimited number of sources of varying types, and
each instance of Sphinx can search an unlimited number of indexes. For example, you
can pull some of the documents in an index from a MySQL instance running on one
remote server, some from a PostgreSQL instance running on another server, and some
from the output of a local script through an XML pipe mechanism.

In this appendix, we explore some use cases where Sphinx’s capabilities enable en-
hanced performance, show a summary of the steps needed to install and configure
it, explain its features in detail, and we discuss several examples of real-world
implementations.

```
745
```

### A Typical Sphinx Search

We start with a simple but complete Sphinx usage example to provide a starting point
for further discussion. We use PHP because of its popularity, although APIs are avail-
able for a number of other languages, too.

Assume that we’re implementing full-text searching for a comparison-shopping engine.
Our requirements are to:

- Maintain a searchable full-text index on a product table stored in MySQL
- Allow full-text searches over product titles and descriptions
- Be able to narrow down searches to a given category if needed
- Be able to sort the result not only by relevance, but by item price or submission date

We begin by setting up a data source and an index in the Sphinx configuration file:

```
source products
{
type = mysql
sql_host = localhost
sql_user = shopping
sql_pass = mysecretpassword
sql_db = shopping
sql_query = SELECT id, title, description, \
cat_id, price, UNIX_TIMESTAMP(added_date) AS added_ts \
FROM products
sql_attr_uint = cat_id
sql_attr_float = price
sql_attr_timestamp = added_ts
}
```
```
index products
{
source = products
path = /usr/local/sphinx/var/data/products
docinfo = extern
}
```
This example assumes that the MySQL shopping database contains a products table
with the columns we request in our SELECT query to populate our Sphinx index. The
Sphinx index is also named products. After creating a new source and index, we run
the _indexer_ program to create the initial full-text index data files and then (re)start the
_searchd_ daemon to pick up the changes:

```
$ cd /usr/local/sphinx/bin
$ ./indexer products
$ ./searchd --stop
$ ./searchd
```
The index is now ready to answer queries. We can test it with Sphinx’s bundled
_test.php_ sample script:

**746 | Appendix F: Using Sphinx with MySQL**


```
$ php -q test.php -i products ipod
```
```
Query 'ipod ' retrieved 3 of 3 matches in 0.010 sec.
Query stats:
'ipod' found 3 times in 3 documents
Matches:
```
1. doc_id=123, weight=100, cat_id=100, price=159.99, added_ts=2008-01-03 22:38:26
2. doc_id=124, weight=100, cat_id=100, price=199.99, added_ts=2008-01-03 22:38:26
3. doc_id=125, weight=100, cat_id=100, price=249.99, added_ts=2008-01-03 22:38:26

The final step is to add searching to our web application. We need to set sorting and
filtering options based on user input and format the output nicely. Also, because Sphinx
returns only document IDs and configured attributes to the client—it doesn’t store any
of the original text data—we need to pull additional row data from MySQL ourselves:

```
1 <?php
2 include ( "sphinxapi.php" );
3 // ... other includes, MySQL connection code,
4 // displaying page header and search form, etc. all go here
5
6 // set query options based on end-user input
7 $cl = new SphinxClient ();
8 $sortby = $_REQUEST["sortby"];
9 if ( !in_array ( $sortby, array ( "price", "added_ts" ) ) )
10 $sortby = "price";
11 if ( $_REQUEST["sortorder"]=="asc" )
12 $cl->SetSortMode ( SPH_SORT_ATTR_ASC, $sortby );
13 else
14 $cl->SetSortMode ( SPH_SORT_ATTR_DESC, $sortby );
15 $offset = ($_REQUEST["page"]-1)*$rows_per_page;
16 $cl->SetLimits ( $offset, $rows_per_page );
17
18 // issue the query, get the results
19 $res = $cl->Query ( $_REQUEST["query"], "products" );
20
21 // handle search errors
22 if ( !$res )
23 {
24 print "<b>Search error:</b>". $cl->GetLastError ();
25 die;
26 }
27
28 // fetch additional columns from MySQL
29 $ids = join ( ",", array_keys ( $res["matches"] );
30 $r = mysql_query ( "SELECT id, title FROM products WHERE id IN ($ids)" )
31 or die ( "MySQL error: ". mysql_error() );
32 while ( $row = mysql_fetch_assoc($r) )
33 {
34 $id = $row["id"];
35 $res["matches"][$id]["sql"] = $row;
36 }
37
38 // display the results in the order returned from Sphinx
39 $n = 1 + $offset;
40 foreach ( $res["matches"] as $id=>$match )
```
```
A Typical Sphinx Search | 747
```

```
41 {
42 printf ( "%d. <a href=details.php?id=%d>%s</a>, USD %.2f<br>\n",
43 $n++, $id, $match["sql"]["title"], $match["attrs"]["price"] );
44 }
45
46 ?>
```
Even though the snippet just shown is pretty simple, there are a few things worth
highlighting:

- The SetLimits() call tells Sphinx to fetch only the number of rows that the client
    wants to display on a page. It’s cheap to impose this limit in Sphinx (unlike in
    MySQL’s built-in search facility), and the number of results that would have been
    returned without the limit are available in $result['total_found'] at no extra cost.
- Because Sphinx only _indexes_ the title column and doesn’t _store_ it, we must fetch
    that data from MySQL.
- We retrieve data from MySQL with a single combined query for the whole docu-
    ment batch using the clause WHERE id IN (...), instead of running one query for
    each match (which would be inefficient).
- We inject the rows pulled from MySQL into our full-text search result set, to keep
    the original sorting order. We explain this more in a moment.
- We display the rows using data pulled from both Sphinx and MySQL.

The row injection code, which is PHP-specific, deserves a more detailed explanation.
We couldn’t simply iterate over the result set from the MySQL query, because the row
order can (and in most cases actually will) be different from that specified in the WHERE
id IN (...) clause. PHP hashes (associative arrays), however, keep the order in which
the matches were inserted into them, so iterating over $result["matches"] will produce
rows in the proper sorting order as returned by Sphinx. To keep the matches in the
proper order returned from Sphinx (rather than the semirandom order returned from
MySQL), therefore, we inject the MySQL query results one by one into the hash that
PHP stores from the Sphinx result set of matches.

There are also a few major implementation and performance differences between
MySQL and Sphinx when it comes to counting matches and applying a LIMIT clause.
First, LIMIT is cheap in Sphinx. Consider a LIMIT 500,10 clause. MySQL will retrieve
510 semirandom rows (which is slow) and throw away 500, whereas Sphinx will return
the IDs that you will use to retrieve the 10 rows you actually need from MySQL. Second,
Sphinx will always return the exact number of matches it actually found in the result
set, no matter what’s in the LIMIT clause. MySQL can’t do this efficiently, although in
MySQL 5.6 it will have partial improvements for this limitation.

**748 | Appendix F: Using Sphinx with MySQL**


### Why Use Sphinx?

Sphinx can complement a MySQL-based application in many ways, bolstering perfor-
mance where MySQL is not a good solution and adding functionality MySQL can’t
provide. Typical usage scenarios include:

- Fast, efficient, scalable, relevant full-text searches
- Optimizing WHERE conditions on low-selectivity indexes or columns without in-
    dexes
- Optimizing ORDER BY ... LIMIT _N_ queries and GROUP BY queries
- Generating result sets in parallel
- Scaling up and scaling out
- Aggregating partitioned data

We explore each of these scenarios in the following sections. This list is not exhaustive,
though, and Sphinx users find new applications regularly. For example, one of Sphinx’s
most important uses—scanning and filtering records quickly—was a user innovation,
not one of Sphinx’s original design goals.

#### Efficient and Scalable Full-Text Searching

MyISAM’s full-text search capability is fast for smaller datasets but performs badly
when the data size grows. With millions of records and gigabytes of indexed text, query
times can vary from a second to more than 10 minutes, which is unacceptable for a
high-performance web application. Although it’s possible to scale MyISAM’s full-text
searches by distributing the data in many locations, this requires you to perform
searches in parallel and merge the results in your application.

Sphinx works significantly faster than MyISAM’s built-in full-text indexes. For in-
stance, it can search over 1 GB of text within 10 to 100 milliseconds—and that scales
well up to 10–100 GB per CPU. Sphinx also has the following advantages:

- It can index data stored with InnoDB and other engines, not just MyISAM.
- It can create indexes on data combined from many source tables, instead of being
    limited to columns in a single table.
- It can dynamically combine search results from multiple indexes.
- In addition to indexing textual columns, its indexes can contain an unlimited
    number of numeric _attributes_ , which are analogous to “extra columns.” Sphinx
    attributes can be integers, floating-point numbers, and Unix timestamps.
- It can optimize full-text searches with additional conditions on attributes.
- Its phrase-based ranking algorithm helps it return more relevant results. For in-
    stance, if you search a table of song lyrics for “I love you, dear,” a song that contains

```
Why Use Sphinx? | 749
```

```
that exact phrase will turn up at the top, before songs that just contain “love” or
“dear” many times.
```
- It makes scaling out much easier.

#### Applying WHERE Clauses Efficiently

Sometimes you’ll need to run SELECT queries against very large tables (containing mil-
lions of records), with several WHERE conditions on columns that have poor index se-
lectivity (i.e., return too many rows for a given WHERE condition) or could not be indexed
at all. Common examples include searching for users in a social network and searching
for items on an auction site. Typical search interfaces let the user apply WHERE conditions
to 10 or more columns, while requiring the results to be sorted by other columns. See
the indexing case study in Chapter 5 for an example of such an application and the
required indexing strategies.

With the proper schema and query optimizations, MySQL can work acceptably for
such queries, as long as the WHERE clauses don’t contain too many columns. But as the
number of columns grows, the number of indexes required to support all possible
searches grows exponentially. Covering all the possible combinations for just four col-
umns strains MySQL’s limits. It becomes very slow and expensive to maintain the
indexes, too. This means it’s practically impossible to have all the required indexes for
many WHERE conditions, and you have to run the queries without indexes.

More importantly, even if you can add indexes, they won’t give much benefit unless
they’re selective. The classic example is a gender column, which isn’t much help because
it typically selects around half of all rows. MySQL will generally revert to a full table
scan when the index isn’t selective enough to help it.

Sphinx can perform such queries much faster than MySQL. You can build a Sphinx
index with only the required columns from the data. Sphinx then allows two types of
access to the data: an indexed search on a keyword or a full scan. In both cases, Sphinx
applies _filters_ , which are its equivalent of a WHERE clause. Unlike MySQL, which decides
internally whether to use an index or a full scan, Sphinx lets you choose which access
method to use.

To use a full scan with filters, specify an empty string as the search query. To use an
indexed search, add pseudokeywords to your full-text fields while building the index
and then search for those keywords. For example, if you wanted to search for items in
category 123, you’d add a “category123” keyword to the document during indexing
and then perform a full-text search for “category123.” You can either add keywords to
one of the existing fields using the CONCAT() function, or create a special full-text field
for the pseudokeywords for more flexibility. Normally, you should use filters for non-
selective values that cover over 30% of the rows, and fake keywords for selective ones
that select 10% or less. If the values are in the 10–30% gray zone, your mileage may
vary, and you should use benchmarks to find the best solution.

**750 | Appendix F: Using Sphinx with MySQL**


Sphinx will perform both indexed searches and scans faster than MySQL. Sometimes
Sphinx actually performs a full scan faster than MySQL can perform an index read.

#### Finding the Top Results in Order

Web applications frequently need the top _N_ results in order. As we discussed previously,
this is hard to optimize in MySQL 5.5 and older versions.

The worst case is when the WHERE condition finds many rows (let’s say 1 million) and
the ORDER BY columns aren’t indexed. MySQL uses the index to identify all the matching
rows, reads the records one by one into the sort buffer with semirandom disk reads,
sorts them all with a filesort, and then discards most of them. It will temporarily store
and process the entire result, ignoring the LIMIT clause and churning RAM. And if the
result set doesn’t fit in the sort buffer, it will need to go to disk, causing even more
disk I/O.

This is an extreme case, and you might think it happens rarely in the real world, but in
fact the problems it illustrates happen often. MySQL’s limitations on indexes for
sorting—using only the leftmost part of the index, not supporting loose index scans,
and allowing only a single range condition—mean many real-world queries can’t ben-
efit from indexes. And even when they can, using semirandom disk I/O to retrieve rows
is a performance killer.

Paginated result sets, which usually require queries of the form SELECT ... LIMIT _N,
M_ , are another performance problem in MySQL. They read _N + M_ rows from disk, causing
a large amount of random I/O and wasting memory resources. Sphinx can accelerate
such queries significantly by eliminating the two biggest problems:

_Memory usage_
Sphinx’s RAM usage is always strictly limited, and the limit is configurable. Sphinx
supports a result set offset and size similar to the MySQL LIMIT _N, M_ syntax, but it
also has a max_matches option. This controls the equivalent of the “sort buffer” size,
on both a per-server and a per-query basis. Sphinx’s RAM footprint is guaranteed
to be within the specified limits.

_I/O_
If attributes are stored in RAM, Sphinx does not do any I/O at all. And even if
attributes are stored on disk, Sphinx will perform sequential I/O to read them,
which is much faster than MySQL’s semirandom retrieval of rows from disks.

You can sort search results by a combination of relevance (weight), attribute values,
and (when using GROUP BY) aggregate function values. The sorting clause syntax is sim-
ilar to a SQL ORDER BY clause:

```
<?php
$cl = new SphinxClient ();
$cl->SetSortMode ( SPH_SORT_EXTENDED, 'price DESC, @weight ASC' );
// more code and Query() call here...
?>
```
```
Why Use Sphinx? | 751
```

In this example, price is a user-specified attribute stored in the index, and @weight is a
special attribute, created at runtime, that contains each result’s computed relevance.
You can also sort by an arithmetic expression involving attribute values, common math
operators, and functions:

```
<?php
$cl = new SphinxClient ();
$cl->SetSortMode ( SPH_SORT_EXPR, '@weight + log(pageviews)*1.5' );
// more code and Query() call here...
?>
```
#### Optimizing GROUP BY Queries

Support for everyday SQL-like clauses would be incomplete without GROUP BY func-
tionality, so Sphinx has that, too. But unlike MySQL’s general-purpose implementa-
tion, Sphinx specializes in solving a practical subset of GROUP BY tasks efficiently. This
subset covers the generation of reports from big (1–100 million row) datasets when one
of the following cases holds:

- The result is only a “small” number of grouped rows (where “small” is on the order
    of 100,000 to 1 million rows).
- Very fast execution speed is required and approximate COUNT(*) results are accept-
    able, when many groups are retrieved from data distributed over a cluster of
    machines.

This is not as restrictive as it might sound. The first scenario covers practically all
imaginable time-based reports. For example, a detailed per-hour report for a period of
10 years will return fewer than 90,000 records. The second scenario could be expressed
in plain English as something like “as quickly and accurately as possible, find the 20
most important records in a 100-million-row sharded table.”

These two types of queries can accelerate general-purpose queries, but you can also use
them for full-text search applications. Many applications need to display not only full-
text matches, but some aggregate results as well. For example, many search result pages
show how many matches were found in each product category, or display a graph of
matching document counts over time. Another common requirement is to group the
results and show the most relevant match from each category. Sphinx’s group-by
support lets you combine grouping and full-text searching, eliminating the overhead
of doing the grouping in your application or in MySQL.

As with sorting, grouping in Sphinx uses fixed memory. It is slightly (10% to 50%)
more efficient than similar MySQL queries on datasets that fit in RAM. In this case,
most of Sphinx’s power comes from its ability to distribute the load and greatly reduce
the latency. For huge datasets that could never fit in RAM, you can build a special disk-
based index for reporting, using inline attributes (defined later). Queries against such
indexes execute about as fast as the disk can read the data—about 30–100 MB/sec on

**752 | Appendix F: Using Sphinx with MySQL**


modern hardware. In this case, the performance can be many times better than
MySQL’s, though the results will be approximate.

The most important difference from MySQL’s GROUP BY is that Sphinx may, under
certain circumstances, yield approximate results. There are two reasons for this:

- Grouping uses a fixed amount of memory. If there are too many groups to hold in
    RAM and the matches are in a certain “unfortunate” order, per-group counts might
    be smaller than the actual values.
- A distributed search sends only the aggregate results, not the matches themselves,
    from node to node. If there are duplicate records in different nodes, per-group
    distinct counts might be greater than the actual values, because the information
    that can remove the duplicates is not transmitted between nodes.

In practice, it is often acceptable to have fast approximate group-by counts. If this isn’t
acceptable, it’s often possible to get exact results by configuring the daemon and client
application carefully.

You can generate the equivalent of COUNT(DISTINCT _<attribute>_ ), too. For example, you
can use this to compute the number of distinct sellers per category in an auction site.

Finally, Sphinx lets you choose criteria to select the single “best” document within each
group. For example, you can select the most relevant document from each domain,
while grouping by domain and sorting the result set by per-domain match counts. This
is not possible in MySQL without a complex query.

#### Generating Parallel Result Sets

Sphinx lets you generate several results from the same data simultaneously, again using
a fixed amount of memory. Compared to the traditional SQL approach of either run-
ning two queries (and hoping that some data stays in the cache between runs) or cre-
ating a temporary table for each search result set, this yields a noticeable improvement.

For example, assume you need per-day, per-week, and per-month reports over a period
of time. To generate these with MySQL you’d have to run three queries with different
GROUP BY clauses, processing the source data three times. Sphinx, however, can process
the underlying data once and accumulate all three reports in parallel.

Sphinx does this with a _multi-query_ mechanism. Instead of issuing queries one by one,
you batch several queries and submit them in one request:

```
<?php
$cl = new SphinxClient ();
$cl->SetSortMode ( SPH_SORT_EXTENDED, "price desc" );
$cl->AddQuery ( "ipod" );
$cl->SetGroupBy ( "category_id", SPH_GROUPBY_ATTR, "@count desc" );
$cl->AddQuery ( "ipod" );
$cl->RunQueries ();
?>
```
```
Why Use Sphinx? | 753
```

Sphinx will analyze the request, identify query parts it can combine, and parallelize the
queries where possible.

For example, Sphinx might notice that only the sorting and grouping modes differ, and
that the queries are otherwise the same. This is the case in the sample code just shown,
where the sorting is by price but the grouping is by category_id. Sphinx will create
several sorting queues to process these queries. When it runs the queries, it will retrieve
the rows once and submit them to all queues. Compared to running the queries one by
one, this eliminates several redundant full-text search or full scan operations.

Note that generating parallel result sets, although it’s a common and important opti-
mization, is only a particular case of the more generalized multi-query mechanism. It
is not the _only_ possible optimization. The rule of thumb is to combine queries in one
request where possible, which generally allows Sphinx to apply internal optimizations.
Even if Sphinx can’t parallelize the queries, it still saves network round-trips. And if
Sphinx adds more optimizations in the future, your queries will use them automatically
with no further changes.

#### Scaling

Sphinx scales well both horizontally (scaling out) and vertically (scaling up).

Sphinx is fully distributable across many machines. All the use cases we’ve mentioned
can benefit from distributing the work across several CPUs.

The Sphinx search daemon ( _searchd_ ) supports special _distributed indexes_ , which know
which local and remote indexes should be queried and aggregated. This means scaling
out is a trivial configuration change. You simply partition the data across the nodes,
configure the master node to issue several remote queries in parallel with local ones,
and that’s it.

You can also scale up, as in using more cores or CPUs on a single machine to improve
latency. To accomplish this, you can just run several instances of _searchd_ on a single
machine and query them all from another machine via a distributed index. Alterna-
tively, you can configure a single instance to communicate with itself so that the parallel
“remote” queries actually run on a single machine, but on different CPUs or cores.

In other words, with Sphinx a single query can be made to use more than one CPU
(multiple concurrent queries will use multiple CPUs automatically). This is a major
difference from MySQL, where one query always gets one CPU, no matter how many
are available. Also, Sphinx does not need any synchronization between concurrently
running queries. That lets it avoid mutexes (a synchronization mechanism), which are
a notorious MySQL performance bottleneck on multi-CPU systems.

Another important aspect of scaling up is scaling disk I/O. Different indexes (including
parts of a larger distributed index) can easily be put on different physical disks or RAID
volumes to improve latency and throughput. This approach has some of the same

**754 | Appendix F: Using Sphinx with MySQL**


benefits as MySQL 5.1’s partitioned tables, which can also partition data into multiple
locations. However, distributed indexes have some advantages over partitioned tables.
Sphinx uses distributed indexes both to distribute the load and to process all parts of
a query in parallel. In contrast, MySQL’s partitioning can optimize some queries (but
not all) by pruning partitions, but the query processing will not be parallelized. And
even though both Sphinx and MySQL partitioning will improve query throughput, if
your queries are I/O-bound, you can expect linear latency improvement from Sphinx
on all queries, whereas MySQL’s partitioning will improve latency only on those queries
where the optimizer can prune entire partitions.

The distributed searching workflow is straightforward:

1. Issue remote queries on all remote servers.
2. Perform sequential local index searches.
3. Read the partial search results from the remote servers.
4. Merge all the partial results into the final result set, and return it to the client.

If your hardware resources permit it, you can search through several indexes on the
same machine in parallel, too. If there are several physical disk drives and several CPU
cores, the concurrent searches can run without interfering with each other. You can
pretend that some of the indexes are remote and configure _searchd_ to contact itself to
launch a parallel query on the same machine:

```
index distributed_sample
{
type = distributed
local = chunk1 # resides on HDD1
agent = localhost:3312:chunk2 # resides on HDD2, searchd contacts itself
}
```
From the client’s point of view, distributed indexes are absolutely no different from
local indexes. This lets you create “trees” of distributed indexes by using nodes as
proxies for sets of other nodes. For example, the first-level node could proxy the queries
to a number of the second-level nodes, which could in turn either search locally them-
selves or pass the queries to other nodes, to an arbitrary depth.

#### Aggregating Sharded Data

Building a scalable system often involves _sharding_ (partitioning) the data across differ-
ent physical MySQL servers. We discussed this in depth in Chapter 11.

When the data is sharded at a fine level of granularity, simply fetching a few rows with
a selective WHERE (which should be fast) means contacting many servers, checking for
errors, and merging the results together in the application. Sphinx alleviates this prob-
lem, because all the necessary functionality is already implemented inside the search
daemon.

```
Why Use Sphinx? | 755
```

Consider an example where a 1 TB table with a billion blog posts is sharded by user ID
over 10 physical MySQL servers, so a given user’s posts always go to the same server.
As long as queries are restricted to a single user, everything is fine: we choose the server
based on user ID and work with it as usual.

Now assume that we need to implement an archive page that shows the user’s friends’
posts. How are we going to display “Other sysbench features,” with entries 981 to 1000,
sorted by post date? Most likely, the various friends’ data will be on different servers.
With only 10 friends, there’s about a 90% chance that more than 8 servers will be used,
and that probability increases to 99% if there are 20 friends. So, for most queries, we
will need to contact all the servers. Worse, we’ll need to pull 1,000 posts from _each_
server and sort them all in the application. Following the suggestions we’ve made pre-
viously in this book, we’d trim down the required data to the post ID and timestamp
only, but that’s still 10,000 records to sort in the application. Most modern scripting
languages consume a lot of CPU time for that sorting step alone. In addition, we’ll
either have to fetch the records from each server sequentially (which will be slow) or
write some code to juggle the parallel querying threads (which will be difficult to im-
plement and maintain).

In such situations, it makes sense to use Sphinx instead of reinventing the wheel. All
we’ll have to do in this case is set up several Sphinx instances, mirror the frequently
accessed post attributes from each table—in this example, the post ID, user ID, and
timestamp—and query the master Sphinx instance for entries 981 to 1000, sorted by
post date, in approximately three lines of code. This is a much smarter way to scale.

### Architectural Overview

Sphinx is a standalone set of programs. The two main programs are:

_indexer_
A program that fetches documents from specified sources (e.g., from MySQL query
results) and creates a full-text index over them. This is a background batch job,
which sites usually run regularly.

_searchd_
A daemon that serves search queries from the indexes _indexer_ builds. This provides
the runtime support for applications.

The Sphinx distribution also includes native _searchd_ client APIs in a number of pro-
gramming languages (PHP, Python, Perl, Ruby, and Java, at the time of this writing),
and SphinxSE, which is a client implemented as a pluggable storage engine for
MySQL 5.0 and newer. The APIs and SphinxSE allow a client application to connect
to _searchd_ , pass it the search query, and fetch back the search results.

Each Sphinx full-text index can be compared to a table in a database; in place of rows
in a table, the Sphinx index consists of _documents_. (Sphinx also has a separate data

**756 | Appendix F: Using Sphinx with MySQL**


structure called a _multivalued attribute_ , discussed later.) Each document has a unique
32-bit or 64-bit integer identifier that should be drawn from the database table being
indexed (for instance, from a primary key column). In addition, each document has
one or more full-text fields (each corresponding to a text column from the database)
and numerical attributes. Like a database table, the Sphinx index has the same fields
and attributes for all of its documents. Table F-1 shows the analogy between a database
table and a Sphinx index.

_Table F-1. Database structure and corresponding Sphinx structure_

```
Database structure Sphinx structure
CREATE TABLE documents (
id int(11) NOT NULL auto_increment,
title varchar(255),
content text,
group_id int(11),
added datetime,
PRIMARY KEY (id)
);
```
```
index documents
document ID
title field, full-text indexed
content field, full-text indexed
group_id attribute, sql_attr_uint
added attribute, sql_attr_timestamp
```
Sphinx does not store the text fields from the database but just uses their contents to
build a search index.

#### Installation Overview

Sphinx installation is straightforward and typically includes the following steps:

```
1.Building the programs from sources:
$ configure && make && make install
```
2. Creating a configuration file with definitions for data sources and full-text indexes
3. Initial indexing
4. Launching _searchd_

After that, the search functionality is immediately available for client programs:

```
<?php
include ( 'sphinxapi.php' );
$cl = new SphinxClient ();
$res = $cl->Query ( 'test query', 'myindex' );
// use $res search result here
?>
```
The only thing left to do is run _indexer_ regularly to update the full-text index data.
Indexes that _searchd_ is currently serving will stay fully functional during reindexing:
_indexer_ will detect that they are in use, create a “shadow” index copy instead, and notify
_searchd_ to pick up that copy on completion.

Full-text indexes are stored in the filesystem (at the location specified in the configu-
ration file) and are in a special “monolithic” format, which is not well suited for

```
Architectural Overview | 757
```

incremental updates. The normal way to update the index data is to rebuild it from
scratch. This is not as big a problem as it might seem, though, for the following reasons:

- Indexing is fast. Sphinx can index plain text (without HTML markup) at a rate of
    4–8 MB/sec on modern hardware.
- You can partition the data in several indexes, as shown in the next section, and
    reindex only the updated part from scratch on each run of _indexer_.
- There is no need to “defragment” the indexes—they are built for optimal I/O,
    which improves search speed.
- Numeric attributes can be updated without a complete rebuild.

A future version will offer an additional index backend, which will support real-time
index updates.

#### Typical Partition Use

Let’s discuss partitioning in a bit more detail. The simplest partitioning scheme is the
_main + delta_ approach, in which two indexes are created to index one document col-
lection. _main_ indexes the whole document set, while _delta_ indexes only documents that
have changed since the last time the main index was built.

This scheme matches many data modification patterns perfectly. Forums, blogs, email
and news archives, and vertical search engines are all good examples. Most of the data
in those repositories never changes once it is entered, and only a tiny fraction of docu-
ments are changed or added on a regular basis. This means the delta index is small and
can be rebuilt as frequently as required (e.g., once every 1–15 minutes). This is equiv-
alent to indexing just the newly inserted rows.

You don’t need to rebuild the indexes to change attributes associated with
documents—you can do this online via _searchd_. You can mark rows as deleted by
simply setting a “deleted” attribute in the main index. Thus, you can handle updates
by marking this attribute on documents in the main index, then rebuilding the delta
index. Searching for all documents that are not marked as “deleted” will return the
correct results.

Note that the indexed data can come from the results of any SELECT statement; it doesn’t
have to come from just a single SQL table. There are no restrictions on the SELECT
statements. That means you can preprocess the results in the database before they’re
indexed. Common preprocessing examples include joins with other tables, creating
additional fields on the fly, excluding some fields from indexing, and manipulating
values.

**758 | Appendix F: Using Sphinx with MySQL**


### Special Features

Besides “just” indexing and searching through database content, Sphinx offers several
other special features. Here’s a partial list of the most important ones:

- The searching and ranking algorithms take word positions and the query phrase’s
    proximity to the document content into account.
- You can bind numeric attributes to documents, including multivalued attributes.
- You can sort, filter, and group by attribute values.
- You can create document snippets with search query keyword highlighting.
- You can distribute searching across several machines.
- You can optimize queries that generate several result sets from the same data.
- You can access the search results from within MySQL using SphinxSE.
- You can fine-tune the load Sphinx imposes on the server.

We covered some of these features earlier. This section covers a few of the remaining
features.

#### Phrase Proximity Ranking

Sphinx remembers word positions within each document, as do other open source
full-text search systems. But unlike most other ones, it uses the positions to rank
matches and return more relevant results.

A number of factors might contribute to a document’s final rank. To compute the rank,
most other systems use only keyword frequency: the number of times each keyword
occurs. The classic BM25 weighting function^1 that virtually all full-text search systems
use is built around giving more weight to words that either occur frequently in the
particular document being searched or occur rarely in the whole collection. The BM25
result is usually returned as the final rank value.

In contrast, Sphinx also computes query phrase proximity, which is simply the length
of the longest verbatim query subphrase contained in the document, counted in words.
For instance, the phrase “John Doe Jr” queried against a document with the text “John
Black, John White Jr, and Jane Dunne” will produce a phrase proximity of 1, because
no two words in the query appear together in the query order. The same query against
“Mr. John Doe Jr and friends” will yield a proximity of 3, because three query words
occur in the document in the query order. The document “John Gray, Jane Doe Jr” will
produce a proximity of 2, thanks to its “Doe Jr” query subphrase.

1. See _[http://en.wikipedia.org/wiki/Okapi_BM25](http://en.wikipedia.org/wiki/Okapi_BM25)_ for details.

```
Special Features | 759
```

By default, Sphinx ranks matches using phrase proximity first and the classic BM25
weight second. This means that verbatim query quotes are guaranteed to be at the very
top, quotes that are off by a single word will be right below those, and so on.

When and how does phrase proximity affect results? Consider searching 1,000,000
pages of text for the phrase “To be or not to be.” Sphinx will put the pages with verbatim
quotes at the very top of the search results, whereas BM25-based systems will first
return the pages with the most mentions of “to,” “be,” “or,” and “not”—pages with
an exact quote match but only a few instances of “to” will be buried deep in the results.

Most major web search engines today rank results with keyword positions as well.
Searching for a phrase on Google will likely result in pages with perfect or near-perfect
phrase matches appearing at the very top of the search results, followed by the “bag of
words” documents.

However, analyzing keyword positions requires additional CPU time, and sometimes
you might need to skip it for performance reasons. There are also cases when phrase
ranking produces undesired, unexpected results. For example, searching for tags in a
cloud is better without keyword positions: it makes no difference whether the tags from
the query are next to each other in the document.

To allow for flexibility, Sphinx offers a choice of ranking modes. Besides the default
mode of proximity plus BM25, you can choose from a number of others that include
BM25-only weighting, fully disabled weighting (which provides a nice optimization if
you’re not sorting by rank), and more.

#### Support for Attributes

Each document might contain an unlimited number of numeric attributes. Attributes
are user-specified and can contain any additional information required for a specific
task. Examples include a blog post’s author ID, an inventory item’s price, a category
ID, and so on.

Attributes enable efficient full-text searches with additional filtering, sorting, and
grouping of the search results. In theory, they could be stored in MySQL and pulled
from there every time a search is performed. But in practice, if a full-text search locates
even hundreds or thousands of rows (which is not many), retrieving them from MySQL
is unacceptably slow.

Sphinx supports two ways to store attributes: inline in the document lists or externally
in a separate file. Inlining requires all attribute values to be stored in the index many
times, once for each time a document ID is stored. This inflates the index size and
increases I/O, but reduces use of RAM. Storing the attributes externally requires pre-
loading them into RAM upon _searchd_ startup.

Attributes normally fit in RAM, so the usual practice is to store them externally. This
makes filtering, sorting, and grouping very fast, because accessing data is a matter of

**760 | Appendix F: Using Sphinx with MySQL**


quick in-memory lookup. Also, only the externally stored attributes can be updated at
runtime. Inline storage should be used only when there is not enough free RAM to hold
the attribute data.

Sphinx also supports _multivalued attributes_ (MVAs). MVA content consists of an ar-
bitrarily long list of integer values associated with each document. Examples of good
uses for MVAs are lists of tag IDs, product categories, and access control lists.

#### Filtering

Having access to attribute values in the full-text engine allows Sphinx to filter and reject
candidate matches as early as possible while searching. Technically, the filter check
occurs after verification that the document contains all the required keywords, but
before certain computationally intensive calculations (such as ranking) are done. Be-
cause of these optimizations, using Sphinx to combine full-text searching with filtering
and sorting can be 10 to 100 times faster than using Sphinx for searching and then
filtering results in MySQL.

Sphinx supports two types of filters, which are analogous to simple WHERE conditions
in SQL:

- An attribute value matches a specified range of values (analogous to a BETWEEN
    clause, or numeric comparisons).
- An attribute value matches a specified set of values (analogous to an IN() list).

If the filters will have a fixed number of values (“set” filters instead of “range” filters),
and if such values are selective, it makes sense to replace the integer values with “fake
keywords” and index them as full-text content instead of attributes. This applies to
both normal numeric attributes and MVAs. We’ll see some examples of how to do this
later.

Sphinx can also use filters to optimize full scans. Sphinx remembers minimum and
maximum attribute values for short continuous row blocks (128 rows, by default) and
can quickly throw away whole blocks based on filtering conditions. Rows are stored
in the order of ascending document IDs, so this optimization works best for columns
that are correlated with the ID. For instance, if you have a row-insertion timestamp
that grows along with the ID, a full scan with filtering on that timestamp will be very
fast.

#### The SphinxSE Pluggable Storage Engine

Full-text search results received from Sphinx almost always require additional work
involving MySQL—at the very least, to pull out the text column values that the Sphinx
index does not store. As a result, you’ll frequently need to JOIN search results from
Sphinx with other MySQL tables.

```
Special Features | 761
```

Although you can achieve this by sending the result’s document IDs to MySQL in a
query, that strategy leads to neither the cleanest nor the fastest code. For high-volume
situations, you should consider using SphinxSE, a pluggable storage engine that you
can compile into MySQL 5.0 or newer, or load into MySQL 5.1 or newer as a plugin.

SphinxSE lets programmers query _searchd_ and access search results from within
MySQL. The usage is as simple as creating a special table with an ENGINE=SPHINX clause
(and an optional CONNECTION clause to locate the Sphinx server if it’s at a nondefault
location), and then running queries against that table:

```
mysql> CREATE TABLE search_table (
-> id INTEGER NOT NULL,
-> weight INTEGER NOT NULL,
-> query VARCHAR(3072) NOT NULL,
-> group_id INTEGER,
-> INDEX(query)
-> ) ENGINE=SPHINX CONNECTION="sphinx://localhost:3312/test";
Query OK, 0 rows affected (0.12 sec)
```
```
mysql> SELECT * FROM search_table WHERE query='test;mode=all' \G
*************************** 1. row ***************************
id: 123
weight: 1
query: test;mode=all
group_id: 45
1 row in set (0.00 sec)
```
Each SELECT passes a Sphinx query as the query column in the WHERE clause. The Sphinx
_searchd_ server returns the results. The SphinxSE storage engine then translates these
into MySQL results and returns them to the SELECT statement.

Queries might include JOINs with any other tables stored using any other storage
engines.

The SphinxSE engine supports most searching options available via the API, too. You
can specify options such as filtering and limits by plugging additional clauses into the
query string:

```
mysql> SELECT * FROM search_table WHERE query='test;mode=all;
-> filter=group_id,5,7,11;maxmatches=3000';
```
Per-query and per-word statistics that are returned by the API are also accessible
through SHOW STATUS:

```
mysql> SHOW ENGINE SPHINX STATUS \G
*************************** 1. row ***************************
Type: SPH INX
Name: stats
Status: total: 3, total found: 3, time: 8, words: 1
*************************** 2. row ***************************
Type: SPHINX
Name: words
Status: test:3:5
2 rows in set (0.00 sec)
```
**762 | Appendix F: Using Sphinx with MySQL**


Even when you’re using SphinxSE, the rule of thumb still is to allow _searchd_ to perform
sorting, filtering, and grouping—i.e., to add all the required clauses to the query string
rather than use WHERE, ORDER BY, or GROUP BY. This is especially important for WHERE
conditions. The reason is that SphinxSE is only a client to _searchd_ , not a full-blown
built-in search library. Thus, you need to pass everything that you can to the Sphinx
engine to get the best performance.

#### Advanced Performance Control

Both indexing and searching operations could impose a significant additional load on
either the search server or the database server. Fortunately, a number of settings let you
limit the load coming from Sphinx.

An undesired database-side load can be caused by _indexer_ queries that either stall
MySQL completely with their locks or just occur too quickly and hog resources from
other concurrent queries.

The first case is a notorious problem with MyISAM, where long-running reads lock the
tables and stall other pending reads and writes—you can’t simply do SELECT * FROM
big_table on a production server, because you risk disrupting all other operations. To
work around that, Sphinx offers _ranged queries_. Instead of configuring a single huge
query, you can specify one query that quickly computes the indexable row ranges and
another query that pulls out the data step by step, in small chunks:

```
sql_query_range = SELECT MIN(id),MAX(id) FROM documents
sql_range_step = 1000
sql_query = SELECT id, title, body FROM documents \
WHERE id>=$start AND id<=$end
```
This feature is extremely helpful for indexing MyISAM tables, but it should also be
considered when using InnoDB tables. Although InnoDB won’t just lock the table and
stall other queries when running a big SELECT *, it will still use significant machine
resources because of its MVCC architecture. Multiversioning for a thousand transac-
tions that cover a thousand rows each can be less expensive than a single long-running
million-row transaction.

The second cause of excessive load happens when _indexer_ is able to process the data
more quickly than MySQL provides it. You should also use ranged queries in this case.
The sql_ranged_throttle option forces _indexer_ to sleep for a given time period (in
milliseconds) between subsequent ranged query steps, increasing indexing time but
easing the load on MySQL.

Interestingly enough, there’s a special case where you can configure Sphinx to achieve
exactly the opposite effect: that is, you can improve indexing time by placing _more_ load
on MySQL. When the connection between the _indexer_ box and the database box is 100
Mbps, and the rows compress well (which is typical for text data), the MySQL com-
pression protocol can improve overall indexing time. The improvement comes at a
cost of more CPU time spent on both the MySQL and _indexer_ sides to compress and

```
Special Features | 763
```

uncompress the rows transmitted over the network, respectively but the overall index-
ing time could be up to 20–30% less because of greatly reduced network traffic.

Search clusters can suffer from occasional overload, too, so Sphinx provides a few ways
to help avoid _searchd_ going off on a spin.

First, a max_children option simply limits the total number of concurrently running
queries and tells clients to retry when that limit is reached.

Then there are query-level limits. You can specify that query processing stop either at
a given threshold of matches found or a given threshold of elapsed time, using the
SetLimits() and SetMaxQueryTime() API calls, respectively. This is done on a per-query
basis, so you can ensure that more important queries always complete fully.

Finally, periodic _indexer_ runs can cause bursts of additional I/O that will in turn cause
intermittent _searchd_ slowdowns. To prevent that, options that limit _indexer_ disk I/O
exist. max_iops enforces a minimal delay between I/O operations that ensures that no
more than max_iops disk operations per second will be performed. But even a single
operation could be too much; consider a 100 MB read() call as an example. The
max_iosize option takes cares of that, guaranteeing that the length of every disk read
or write will be under a given boundary. Larger operations are automatically split into
smaller ones, and these smaller ones are then controlled by max_iops settings.

### Practical Implementation Examples

Each of the features we’ve described can be found successfully deployed in production.
The following sections review several of these real-world Sphinx deployments, briefly
describing the sites and some implementation details.

#### Full-Text Searching on Mininova.org

A popular torrent search engine, Mininova ( _[http://www.mininova.org](http://www.mininova.org)_ ) provides a clear
example of how to optimize “just” full-text searching. Sphinx replaced several MySQL
replicas using MySQL built-in full-text indexes, which were unable to handle the load.
After the replacement, the search servers were underloaded; the current load average
is now in the 0.3–0.4 range.

Here are the database size and load numbers:

- The site has a small database, with about 300,000–500,000 records and about
    300–500 MB of index.
- The site load is quite high: about 8–10 million searches per day at the time of this
    writing.

The data mostly consists of user-supplied filenames, frequently without proper punc-
tuation. For this reason, prefix indexing is used instead of whole-word indexing. The

**764 | Appendix F: Using Sphinx with MySQL**


resulting index is several times larger than it would otherwise be, but it is still small
enough that it can be built quickly and its data can be cached effectively.

Search results for the 1,000 most frequent queries are cached on the application side.
About 20–30% of all queries are served from the cache. Because of the “long tail” query
distribution, a larger cache would not help much more.

For high availability, the site uses two servers with complete full-text index replicas.
The indexes are rebuilt from scratch every few minutes. Indexing takes less than one
minute, so there’s no point in implementing more complex schemes.

The following are lessons learned from this example:

- Caching search results in the application helps a lot.
- There might be no need to have a huge cache, even for busy applications. A mere
    1,000 to 10,000 entries can be enough.
- For databases on the order of 1 GB in size, simple periodic reindexing instead of
    more complicated schemes is OK, even for busy sites.

#### Full-Text Searching on BoardReader.com

Mininova is an extreme high-load project case—there’s not that much data, but there
are a lot of queries against that data. BoardReader ( _[http://www.boardreader.com](http://www.boardreader.com)_ ) is just
the opposite: a forum search engine that performs many fewer searches on a much
larger dataset. Sphinx replaced a commercial full-text search engine that took up to 10
seconds per query to search through a 1 GB collection. Sphinx allowed BoardReader
to scale greatly, both in terms of data size and query throughput.

Here’s some general information:

- There are more than 1 billion documents and 1.5 TB of text in the database.
- There are about 500,000 page views and between 700,000 and 1 million searches
    per day.

At the time of this writing, the search cluster consists of six servers, each with four
logical CPUs (two dual-core Xeons), 16 GB of RAM, and 0.5 TB of disk space. The
database itself is stored on a separate cluster. The search cluster is used only for
indexing and searching.

Each of the six servers runs four _searchd_ instances, so all four cores are used. One of
the four instances aggregates the results from the other three. That makes a total of 24
_searchd_ instances. The data is distributed evenly across all of them. Every _searchd_ copy
carries several indexes over approximately 1/24 of the total data (about 60 GB).

The search results from the six “first-tier” _searchd_ nodes are in turn aggregated by
another _searchd_ instance running on the frontend web server. This instance carries
several purely distributed indexes, which reference the six search cluster servers but
have no local data at all.

```
Practical Implementation Examples | 765
```

Why have four _searchd_ instances per node? Why not have only one _searchd_ instance
per server, configure it to carry four index chunks, and make it contact itself as though
it’s a remote server to utilize multiple CPUs, as we suggested earlier? Having four in-
stances instead of just one has its benefits. First, it reduces startup time. There are
several gigabytes of attribute data that need to be preloaded in RAM; starting several
daemons at a time lets us parallelize that. Second, it improves availability. In the event
of _searchd_ failures or updates, only 1/24 of the whole index is inaccessible, instead
of 1/6.

Within each of the 24 instances on the search cluster, we used time-based partitioning
to reduce the load even further. Many queries need to be run only on the most recent
data, so the data is divided into three disjoint index sets: data from the last week, from
the last three months, and from all time. These indexes are distributed over several
different physical disks on a per-instance basis. This way, each instance has its own
CPU and physical disk drive and won’t interfere with the others.

Local _cron_ jobs update the indexes periodically. They pull the data from MySQL over
the network but create the index files locally.

Using several explicitly separated “raw” disks proved to be faster than a single RAID
volume. Raw disks give control over which files go on which physical disk. That is not
the case with RAID, where the controller decides which block goes on which physical
disk. Raw disks also guarantee fully parallel I/O on different index chunks, but con-
current searches on RAID are subject to I/O stepping. We chose RAID 0, which has no
redundancy, because we don’t care about disk failures; we can easily rebuild the indexes
on the search nodes. We could also have used several RAID 1 (mirror) volumes to give
the same throughput as raw disks while improving reliability.

Another interesting thing to learn from BoardReader is how Sphinx version updates
are performed. Obviously, the whole cluster cannot be taken down. Therefore, back-
ward compatibility is critical. Fortunately, Sphinx provides it—newer _searchd_ versions
usually can read older index files, and they are always able to communicate to older
clients over the network. Note that the first-tier nodes that aggregate the search results
look just like clients to the second-tier nodes, which do most of the actual searching.
Thus, the second-tier nodes are updated first, then the first-tier ones, and finally the
web frontend.

Lessons learned from this example are:

- The Very Large Database Motto: partition, partition, partition, parallelize.
- On big search farms, organize _searchd_ in trees with several tiers.
- Build optimized indexes with a fraction of the total data where possible.
- Map files to disks explicitly rather than relying on the RAID controller.

**766 | Appendix F: Using Sphinx with MySQL**


#### Optimizing Selects on Sahibinden.com

Sahibinden ( _[http://www.sahibinden.com](http://www.sahibinden.com)_ ), a leading Turkish online auction site, had a
number of performance problems, including full-text search performance. After de-
ploying Sphinx and profiling some queries, we found that Sphinx could perform many
of the frequent application-specific queries with filters faster than MySQL—even when
there was an index on one of the participating columns in MySQL. Besides, using
Sphinx for non-full-text searches resulted in unified application code that was simpler
to write and support.

MySQL was underperforming because the selectivity on each individual column was
not enough to reduce the search space significantly. In fact, it was almost impossible
to create and maintain all the required indexes, because too many columns required
them. The product information tables had about 100 columns, each of which the web
application could technically use for filtering or sorting.

Active insertion and updates to the “hot” products table slowed to a crawl, because of
too many index updates.

For that reason, Sphinx was a natural choice for _all_ the SELECT queries on the product
information tables, not just the full-text search queries.

Here are the database size and load numbers for the site:

- The database contains about 400,000 records and 500 MB of data.
- The load is about 3 million queries per day.

To emulate normal SELECT queries with WHERE conditions, the Sphinx indexing process
included special keywords in the full-text index. The keywords were of the
form _ _CAT _N_ _ _ _, where _N_ was replaced with the corresponding category ID. This
replacement happened during indexing with the CONCAT() function in the MySQL
query, so the source data was not altered.

The indexes needed to be rebuilt as frequently as possible. We settled on rebuilding
them every minute. A full reindexing took 9–15 seconds on one of many CPUs, so the
_main + delta_ scheme discussed earlier was not necessary.

The PHP API turned out to spend a noticeable amount of time (7–9 milliseconds per
query) parsing the result set when it had many attributes. Normally, this overhead
would not be an issue because the full-text search costs, especially over big collections,
would be higher than the parsing cost. But in this specific case, we also needed non-
full-text queries against a small collection. To alleviate the issue, the indexes were sep-
arated into pairs: a “lightweight” one with the 34 most frequently used attributes, and
a “complete” one with all 99 attributes.

Other possible solutions would have been to use SphinxSE or to implement a feature
to pull only the specified columns into Sphinx. However, the workaround with two
indexes was by far the fastest to implement, and time was a concern.

```
Practical Implementation Examples | 767
```

The following are the lessons learned from this example:

- Sometimes, a full scan in Sphinx performs better than an index read in MySQL.
- For selective conditions, use a “fake keyword” instead of filtering on an attribute,
    so the full-text search engine can do more of the work.
- APIs in scripting languages can be a bottleneck in certain extreme but real-world
    cases.

#### Optimizing GROUP BY on BoardReader.com

An improvement to the BoardReader service required counting hyperlinks and building
various reports from the linking data. For instance, one of the reports needed to show
the top _N_ second-level domains linked to during the last week. Another counted the top
_N_ second- and third-level domains that linked to a given site, such as YouTube. The
queries to build these reports had the following common characteristics:

- They always group by domain.
- They sort by count per group or by the count of distinct values per group.
- They process a lot of data (up to millions of records), but the result set with the
    best groups is always small.
- Approximate results are acceptable.

During the prototype-testing phase, MySQL took up to 300 seconds to execute these
queries. In theory, by partitioning the data, splitting it across servers, and manually
aggregating the results in the application, it would have been possible to optimize the
queries to around 10 seconds. But this is a complicated architecture to build; even the
partitioning implementation is far from straightforward.

Because we had successfully distributed the search load with Sphinx, we decided to
implement an approximate distributed GROUP BY with Sphinx, too. This required pre-
processing the data before indexing to convert all the interesting substrings into stand-
alone “words.” Here’s a sample URL before and after preprocessing:

```
source_url = http://my.blogger.com/my/best-post.php
processed_url = my$blogger$com, blogger$com, my$blogger$com$my,
my$blogger$com$my$best, my$blogger$com$my$best$post.php
```
Dollar signs ($) are merely a unified replacement for URL separator characters so that
searches can be conducted on any URL part, be it domain or path. This type of pre-
processing extracts all “interesting” substrings into single keywords that are the fastest
to search. Technically, we could have used phrase queries or prefix indexing, but that
would have resulted in bigger indexes and slower performance.

Links are preprocessed at indexing time using a specially crafted MySQL UDF. We also
enhanced Sphinx with the ability to count distinct values for this task. After that, we
were able to move the queries completely to the search cluster, distribute them easily,
and reduce query latency greatly.

**768 | Appendix F: Using Sphinx with MySQL**


Here are the database size and load numbers:

- There are about 150–200 million records, which becomes about 50–100 GB of data
    after preprocessing.
- The load is approximately 60,000–100,000 GROUP BY queries per day.

The indexes for the distributed GROUP BY were deployed on the same search cluster of
6 machines and 24 logical CPUs described previously. This is a minor complementary
load to the main search load over the 1.5 TB text database.

Sphinx replaced MySQL’s exact, slow, single-CPU computations with approximate,
fast, distributed computations. All of the factors that introduce approximation errors
are present here: the incoming data frequently contains too many rows to fit in the “sort
buffer” (we use a fixed RAM limit of 100K rows), we use COUNT(DISTINCT), and the
result sets are aggregated over the network. Despite that, the results for the first 10 to
1000 groups—which are actually required for the reports—are from 99% to 100%
correct.

The indexed data is very different from the data that would be used for an ordinary
full-text search. There are a huge number of documents and keywords, even though
the documents are very small. The document numbering is nonsequential, because a
special numbering convention (source server, source table, and primary key) that does
not fit in 32 bits is used. The huge amount of search “keywords” was also causing
frequent CRC32 collisions (Sphinx uses CRC32 to map keywords to internal word IDs).
For these reasons, we were forced to use 64-bit identifiers everywhere internally.

The current performance is satisfactory. For the most complex domains, queries nor-
mally complete in 0.1 to 1.0 seconds.

The following are the lessons learned from this example:

- For GROUP BY queries, some precision can be traded for speed.
- With huge textual collections or moderately sized special collections, 64-bit
    identifiers might be required.

#### Optimizing Sharded JOIN Queries on Grouply.com

Grouply ( _[http://www.grouply.com](http://www.grouply.com)_ ) built a Sphinx-based solution to search its multi-
million-record database of tagged messages, using Sphinx’s MVA support. The data-
base is split across many physical servers for massive scalability, so it might be necessary
to query tables that are located on different servers. Arbitrary large-scale joins are im-
possible because there are too many participating servers, databases, and tables.

Grouply uses Sphinx’s MVA attributes to store message tags. The tag list is retrieved
from a Sphinx cluster via the PHP API. This replaces multiple sequential SELECTs
from several MySQL servers. To reduce the number of SQL queries as well, certain

```
Practical Implementation Examples | 769
```

presentation-only data (for example, a small list of users who last read the message) is
also stored in a separate MVA attribute and accessed through Sphinx.

Two key innovations here are using Sphinx to prebuild JOIN results and using its dis-
tributed capabilities to merge data scattered over many shards. This would be next to
impossible with MySQL alone. Efficient merging would require partitioning the data
over as few physical servers and tables as possible, but that would hurt both scalability
and extensibility.

Lessons learned from this example are:

- Sphinx can be used to aggregate highly partitioned data efficiently.
- MVAs can be used to store and optimize prebuilt JOIN results.

### Summary

We’ve discussed the Sphinx full-text search system only briefly in this appendix. To
keep it short, we intentionally omitted discussions of many other Sphinx features, such
as HTML indexing support, ranged queries for better MyISAM support, morphology
and synonym support, prefix and infix indexing, and CJK indexing. Nevertheless, this
appendix should give you some idea of how Sphinx can solve many different real-world
problems efficiently. It is not limited to full-text searching; it can solve a number of
difficult problems that would traditionally be done in SQL.

Sphinx is neither a silver bullet nor a replacement for MySQL. However, in many cases
(which are becoming common in modern web applications), it can be used as a very
useful complement to MySQL. You can use it to simply offload some work, or even to
create new possibilities for your application.

Download it at _[http://www.sphinxsearch.com—and](http://www.sphinxsearch.com—and)_ don’t forget to share your own
usage ideas!

**770 | Appendix F: Using Sphinx with MySQL**


```
Index
```
**Symbols**
32-bit architecture, 390
404 errors, 614, 617
451 Group, 549
64-bit architecture, 390
:= assign operator, 249
@ user variable, 253
@@ system variable, 334

**A**
ab tool, Apache, 51
Aborted_clients variable, 688
Aborted_connects variable, 688
access time, 398
access types, 205, 727
ACID transactions, 6, 551
active caches, 611
active data, keeping separate, 554
active-active access, 574
Adaptec controllers, 405
adaptive hash indexes, 154, 703
Adaptive Query Localization, 550, 577
Address Resolution Protocol (ARP), 560, 584
Adminer, 666
admission control features, 373
advanced performance control, 763
after-action reviews, 571
aggregating sharded data, 755
Ajax, 607
Aker, Brian, 296, 679
Akiban, 549, 552
algebraic equivalence rules, 217
algorithms, load-balancing, 562
ALL_O_DIRECT variable, 363

```
ALTER TABLE command, 11, 28, 141–144,
266, 472, 538
Amazon EBS (Elastic Block Store), 589, 595
Amazon EC2 (Elastic Compute Cloud), 589,
595–598
Amazon RDS (Relational Database Service),
589, 600
Amazon Web Services (AWS), 589
Amdahl scaling, 525
Amdahl’s Law, 74, 525
ANALYZE TABLE command, 195
ANSI SQL isolation levels, 8
Apache ab, 51
application-level optimization
alternatives to MySQL, 619
caching, 611–618
common problems, 605–607
extending MySQL, 618
finding the optimal concurrency, 609
web server issues, 608
approximations, 243
Archive storage engine, 19, 220
Aria storage engine, 23, 681
ARP (Address Resolution Protocol), 560, 584
Aslett, Matt, 549
Aspersa (see Percona Toolkit)
asynchronous I/O, 702
asynchronous replication, 447
async_unbuffered, 364
atomicity, 6
attributes, 749, 760
audit plugins, 297
auditing, 622
authentication plugins, 298
auto-increment keys, 578
```
We’d like to hear your suggestions for improving our indexes. Send email to _index@oreilly.com_.

```
771
```

AUTOCOMMIT mode, 10
autogenerated schemas, 131
AUTO_INCREMENT, 142, 275, 505, 545
availability zone, 572
AVG() function, 139
AWS (Amazon Web Services), 589

**B**
B-Tree indexes, 148, 171, 197, 217, 269
Background Patrol Read, 418
Backup & Recovery (Preston), 621
backup load, 626
backup time, 626
backup tools
Enterprise Backup, MySQL, 658
mydumper, 659
mylvmbackup, 659
mysqldump, 660
Percona XtraBackup, 658
Zmanda Recovery Manager, 659
backups, 425, 621
binary logs, 634–636
data, 637–648
designing a MySQL solution, 624–634
online or offline, 625
reasons for, 622
and replication, 449
scripting, 661–663
snapshots not, 646
and storage engines, 24
tools for, 658–661
balanced trees, 223
Barth, Wolfgang, 668
batteries in SSDs, 405
BBU (battery backup unit), 422
BEFORE INSERT trigger, 287
Beginning Database Design (Churcher), 115
Bell, Charles, 519
Benchmark Suite, 52, 55
BENCHMARK() function, 53
benchmarks, 35–37
analyzing results, 47
capturing system performance and status,
44
common mistakes, 40, 340
design and planning, 41
examples, 54–66
file copy, 718
flash memory, 403

```
getting accurate results, 45
good uses for, 340
how long to run, 42
iterative optimization by, 338
MySQL versions read-only, 31
plotting, 49
SAN, 423
strategies, 37–40
tactics, 40–50
tools, 51–53
what to measure, 38
BerkeleyDB, 30
BIGINT type, 117
binary logs
backing up, 634–636
format, 635
master record changes (events), 449, 496
purging old logs safely, 636
status, 688
binlog dump command, 450, 474
binlog_do_db variable, 466
binlog_ignore_db variable, 466
Birthday Paradox, 156
BIT type, 127
bit-packed data types, 127
bitwise operations, 128
Blackhole storage engine, 20, 475, 480
blktrace, 442
BLOB type, 21, 121, 375
blog, MySQL Performance, 23
BoardReader.com, 765, 768
Boolean full-text searches, 308
Boost library, 682
Bouman, Roland, 281, 287, 667
buffer pool
InnoDB, 704
size of, 344
buffer threads, 702
built-in MySQL engines, 19–21
bulletin boards, 27
burstable capacity, 42
bzip2, 716
```
```
C
cache hits, 53, 316, 340, 395
CACHE INDEX command, 351
cache tables, 136
cache units, 396
cachegrind, 78
```
**772 | Index**


caches
allocating memory for, 349
control policies, 614
hierarchy, 393, 616
invalidations, 322
misses, 321, 352, 397
RAID, 419
read-ahead data, 421
tuning by ratio, 340
writes, 421
Cacti, 430, 669
Calpont InfiniDB, 23
capacitors in SSDs, 405
capacity planning, 425, 482
cardinality, 160, 215
case studies
building a queue table, 256
computing the distance between points,
258
diagnostics, 102–110
indexing, 189–194
using user-defined functions, 262
CD-ROM applications, 27
Change Data Capture (CDC) utilities, 138
CHANGE MASTER TO command, 453, 457,
489, 491, 501
CHAR type, 120
character sets, 298, 301–305, 330
character_set_database, 300
CHARSET() function, 300
CHAR_LENGTH() function, 304
CHECK OPTION variable, 278
CHECK TABLES command, 371
CHECKSUM TABLE command, 488
chunk size, 419
Churcher, Clare, 115
Circonus, 671
circular replication, 473
Cisco server, 598
client, returning results to, 228
client-side emulated prepared statements, 295
client/server communication settings, 299
cloud, MySQL in the, 589–602
benchmarks, 598
benefits and drawbacks, 590–592
DBaaS, 600
economics, 592
four fundamental resources, 594
performance, 595–598

```
scaling and HA, 591
Cluster Control, SeveralNines, 577
Cluster, MySQL, 576
clustered indexes, 17, 168–176, 397, 657
clustering, scaling by, 548
Clustrix, 549, 565
COALESCE() function, 254
code
backing up, 630
stored, 282–284, 289
Codership Oy, 577
COERCIBILITY() function, 300
cold or warm copy, 456
Cole, Jeremy, 85
collate clauses, 300
COLLATION() function, 300
collations, 119, 298, 301–305
collisions, hash, 156
column-oriented storage engines, 22
command counters, 689
command-line monitoring with innotop, 672–
676
command-line utilities, 666
comments
stripping before compare, 316
version-specific, 289
commercial monitoring systems, 670
common_schema, 187, 667
community storage engines, 23
complete result sets, 315
COMPRESS() function, 377
compressed files, 715
compressed MyISAM tables, 19
computations
distance between points, 258–262
integer, 117
temporal, 125
Com_admin_commands variable, 689
CONCAT() function, 750, 767
concurrency
control, 3–6
inserts, 18
measuring, 39
multiversion concurrency control (MVCC),
12
need for high, 596
configuration
by cache hit ratio, 340
completing basic, 378–380
```
```
Index | 773
```

creating configuration files, 342–347
InnoDB flushing algorithm, 412
memory usage, 347–356
MySQL concurrency, 371–374
workload-based, 375–377
connection management, 2
connection pooling, 561, 607
connection refused error, 429
connection statistics, 688
CONNECTION_ID() function, 257, 289, 316,
502, 635
consistency, 7
consolidation
scaling by, 547
storage, 407, 425
constant expressions, 217
Continuent Tungsten Replicator, 481, 516
CONVERT() function, 300
Cook, Richard, 571
correlated subqueries, 229–233
corrupt system structures, 657
corruption, finding and repairing, 194, 495–
498
COUNT() function optimizations, 206, 217,
241–243, 292
counter tables, 139
counters, 686, 689
covering indexes, 177–182, 218
CPU-bound machines, 442
CPUs, 56, 70, 388–393, 594, 598
crash recovery, 25
crash testing, 422
CRC32() function, 156, 541
CREATE and SELECT conversions, 28
CREATE INDEX command, 353
CREATE TABLE command, 184, 266, 353,
476, 481
CREATE TEMPORARY TABLE command,
689
cron jobs, 288, 585, 630
crontab, 504
cross-data center replication, 475
cross-shard queries, 535, 538
CSV format, 638
CSV logging table, 601
CSV storage engine, 20
CURRENT_DATE() function, 316
CURRENT_USER() function, 316, 460
cursors, 290

```
custom benchmark suite, 339
custom replication solutions, 477–482
```
```
D
daemon plugins, 297
dangling pointer records, 553
data
archiving, 478, 509
backing up nonobvious, 629
changes on the replica, 500
consistency, 632
deduplication, 631
dictionary, 356
distribution, 448
fragmentation, 197
loss, avoiding, 553
optimizing access to, 202–207
scanning, 269
sharding, 533–547, 565, 755
types, 115
volume of and search engine choice, 27
Data Definition Language (DDL), 11
Data Recovery Toolkit, 195
data types
BIGINT, 117
BIT, 127
BLOB, 21, 121, 375
CHAR, 120
DATETIME, 117, 126
DECIMAL, 118
DOUBLE, 118
ENUM, 123, 130, 132, 282
FLOAT, 118
GEOMETRY, 157
INT, 117
LONGBLOB, 122
LONGTEXT, 122
MEDIUMBLOB, 122
MEDIUMINT, 117
MEDIUMTEXT, 122
RANGE COLUMNS, 268
SET, 128, 130
SMALLBLOB, 122
SMALLINT, 117
SMALLTEXT, 122
TEXT, 21, 121, 375
TIMESTAMP, 117, 126, 631
TINYBLOB, 122
TINYINT, 117
```
**774 | Index**


TINYTEXT, 122
VARCHAR, 119, 124, 131, 513
data=journal option, 433
data=ordered option, 433
data=writeback option, 433
Database as a Service (DBaaS), 589, 600
database servers, 393
Database Test Suite, 52
Date, C. J., 255
DATETIME type, 117, 126
DBaaS (Database as a Service), 589, 600
dbShards, 547, 549
dbt2 tool, 52, 61
DDL (Data Definition Language), 11
deadlocks, 9
Debian, 683
debug symbols, 99
debugging locks, 735–744
DECIMAL type, 118
deduplication, data, 631
“degraded” mode, 485
DELAYED hint, 239
delayed key writes, 19
delayed replication, 654
DELETE command, 267, 278
delimited file backups, 638, 651
DeNA, 618
denormalization, 133–136
dependencies on nonreplicated data, 501
derived tables, 238, 277, 725
DETERMINISTIC variable, 284
diagnostics, 92
capturing diagnostic data, 97–102
case study, 102–110
single-query versus server-wide problems,
93–96
differential backups, 630
directio() function, 362
directory servers, 542
dirty reads, 8
DISABLE KEYS command, 143, 313
disaster recovery, 622
disk queue scheduler, 434
disk space, 511
disruptive innovations, 31
DISTINCT queries, 135, 219, 244
distributed (XA) transactions, 313
distributed indexes, 754
distributed memory caches, 613

```
distributed replicated block device (DRBD),
494, 568, 574, 581
distribution master and replicas, 474
DNS (Domain Name System), 556, 559, 572,
584
document pointers, 306
Domain Name System (DNS), 556, 559, 572,
584
DorsalSource, 683
DOUBLE type, 118
doublewrite buffer, 368, 412
downtime, causes of, 568
DRBD (distributed replicated block device),
494, 568, 574, 581
drinking from the fire hose, 211
Drizzle, 298, 682
DROP DATABASE command, 624
DROP TABLE command, 28, 366, 573, 652
DTrace, 431
dump and import conversions, 28
duplicate indexes, 185–187
durability, 7
DVD-ROM applications, 27
dynamic allocation, 541–543
dynamic optimizations, 216
dynamic SQL, 293, 335–337
```
```
E
early termination, 218
EBS (Elastic Block Store), Amazon, 589, 595
EC2 (Elastic Compute Cloud), 589, 595–598
edge side (ESI), 608
Elastic Block Store (EBS), Amazon, 589, 595
Elastic Compute Cloud (EC2), Amazon, 589,
595–598
embedded escape sequences, 301
eMLC (enterprise MLC), 402
ENABLE KEYS command, 313
encryption overhead, avoiding, 716
end_log_pos, 635
Enterprise Backup, MySQL, 457, 624, 627, 631,
658
enterprise MLC (eMLC), 402
Enterprise Monitor, MySQL, 80, 670
ENUM type, 123, 130, 132, 282
equality propagation, 219, 234
errors
404 error, 614, 617
from data corruption or loss, 495–498
```
```
Index | 775
```

ERROR 1005, 129
ERROR 1168, 275
ERROR 1267, 300
escape sequences, 301
evaluation order, 253
Even Faster Websites (Souders), 608
events, 282, 288
exclusive locks, 4
exec_time, 636
EXISTS operator, 230, 232
expire_logs_days variable, 381, 464, 624, 636
EXPLAIN command, 89, 165, 182, 222, 272,
277, 719–733
explicit allocation, 543
explicit invalidation, 614
explicit locking, 11
external XA transactions, 315
extra column, 732

**F**
Facebook, 77, 408, 592
fadvise() function, 626
failback, 582
failover, 449, 582, 585
failures, mean time between, 570
Falcon storage engine, 22
fallback, 582
fast warmup feature, 351
FathomDB, 602
FCP (Fibre Channel Protocol), 422
fdatasync() function, 362
Federated storage engine, 20
Fedora, 683
fencing, 584
fetching mistakes, 203
Fibre Channel Protocol (FCP), 422
FIELD() function, 124, 128
FILE () function, 600
FILE I/O, 702
files
consistency of, 633
copying, 715
descriptors, 690
transferring large, 715–718
filesort, 226, 377
filesystems, 432–434, 573, 640–648
filtered column, 732
filtering, 190, 466, 564, 750, 761
fincore tool, 353

```
FIND_IN_SET() function, 128
fire hose, drinking from the, 211
FIRST() function, 255
first-write penalty, 595
Five Whys, 571
fixed allocation, 541–543
flapping, 583
flash storage, 400–414
Flashcache, 408–410
Flexviews tools, 138, 280
FLOAT type, 118
FLOOR() function, 260
FLUSH LOGS command, 492, 630
FLUSH QUERY CACHE command, 325
FLUSH TABLES WITH READ LOCK
command, 355, 370, 490, 494, 626,
644
flushing algorithm, InnoDB, 412
flushing binary logs, 663
flushing log buffer, 360, 703
flushing tables, 663
FOR UPDATE hint, 240
FORCE INDEX hint, 240
foreign keys, 129, 281, 329
Forge, MySQL, 667, 710
FOUND_ROWS() function, 240
fractal trees, 22, 158
fragmentation, 197, 320, 322, 324
free space fragmentation, 198
FreeBSD, 431, 640
“freezes”, 69
frequency scaling, 392
.frm file, 14, 142, 354, 711
FROM_UNIXTIME() function, 126
fsync() function, 314, 362, 368, 656, 693
full-stack benchmarking, 37, 51
full-text searching, 157, 305–313, 479
on BoardReader.com, 765
Boolean full-text searches, 308
collection, 306
on Mininova.org, 764
parser plugins, 297
Sphinx storage engine, 749
functional partitioning, 531, 564
furious flushing, 49, 704, 706
Fusion-io, 407
```
```
G
Galbraith, Patrick, 296
```
**776 | Index**


Galera, 549, 577, 579
Ganglia, 670
garbage collection, 401
GDB stack traces, 99
gdb tool, 99–100
general log, 81
GenieDB, 549, 551
Gentoo, 683
GEOMETRY type, 157
geospatial searches, 25, 157, 262
GET_LOCK() function, 256, 288
get_name_from_id() function, 613
Gladwell, Malcom, 571
glibc libraries, 348
global locks, 736, 738
global scope, 333
global version/session splits, 558
globally unique IDs (GUIDs), 545
gnuplot, 49, 96
Goal (Goldratt), 526, 565
Goal-Driven Performance Optimization white
paper, 70
GoldenGate, Oracle, 516
Goldratt, Eliyahu M., 526, 565
Golubchik, Sergei, 298
Graphite, 670
great-circle formula, 259
GREATEST() function, 254
grep, 638
Grimmer, Lenz, 659
Groonga storage engine, 23
Groundwork Open Source, 669
GROUP BY queries, 135, 137, 163, 244, 312,
752
group commit, 314
Grouply.com, 769
GROUP_CONCAT() function, 230
Guerrilla Capacity Planning (Gunther), 525,
565
GUID values, 545
Gunther, Neil J., 525, 565
gunzip tool, 716
gzip compression, 609, 716, 718

**H**
Hadoop, 620
handler API, 228
handler operations, 228, 265, 690
HandlerSocket, 618

```
HAProxy, 556
hard disks, choosing, 398
hardware and software RAID, 418
hardware threads, 388
hash codes, 152
hash indexes, 21, 152
hash joins, 234
Haversine formula, 259
header, 693
headroom, 573
HEAP tables, 20
heartbeat record, 487
HEX() function, 130
Hibernate Core interfaces, 547
Hibernate Shards, 547
high availability
achieving, 569–572
avoiding single points of failure, 572–581
defined, 567
failover and failback, 581–585
High Availability Linux project, 582
high bits, 506
High Performance Web Sites (Souders), 608
high throughput, 389
HIGH_PRIORITY hint, 238
hit rate, 322
HiveDB, 547
hot data, segregating, 269
“hot” online backups, 17
How Complex Systems Fail (Cook), 571
HTTP proxy, 585
http_load tool, 51, 54
Hutchings, Andrew, 298
Hyperic HQ, 669
hyperthreading, 389
```
```
I
I/O
benchmark, 57
InnoDB, 357–363
MyISAM, 369–371
performance, 595
slave thread, 450
I/O-bound machines, 443
IaaS (Infrastructure as a Service), 589
.ibd files, 356, 366, 648
Icinga, 668
id column, 723
identifiers, choosing, 129–131
```
```
Index | 777
```

idle machine’s vmstat output, 444
IF() function, 254
IfP (instrumentation-for-php), 78
IGNORE INDEX hint, 165, 240
implicit locking, 11
IN() function, 190–193, 219, 260
incr() function, 546
incremental backups, 630
.index files, 464
index-covered queries, 178–181
indexer, Sphinx, 756
indexes
benefits of, 158
case study, 189–194
clustered, 168–176
covering, 177–182
and locking, 188
maintaining, 194–198
merge optimizations, 234
and mismatched PARTITION BY, 270
MyISAM storage engine, 143
order of columns, 165–168
packed (prefix-compressed), 184
reducing fragmentation, 197
redundant and duplicate, 185–187
and scans, 182–184, 269
statistics, 195, 220
strategies for high performance, 159–168
types of, 148–158
unused, 187
INET_ATON() function, 131
INET_NTOA() function, 131
InfiniDB, Calpont, 23
info() function, 195
Infobright, 22, 28, 117, 269
INFORMATION_SCHEMA tables, 14, 110,
297, 499, 742–744
infrastructure, 617
Infrastructure as a Service (IaaS), 589
Ingo, Henrik, 515, 683
inner joins, 216
Innobase Oy, 30
InnoDB, 13, 15
advanced settings, 383–385
buffer pool, 349, 711
concurrency configuration, 372
crash recovery, 655–658
data dictionary, 356, 711
data layout, 172–176

```
Data Recovery Toolkit, 195
and deadlocks, 9
and filesystem snapshots, 644–646
flushing algorithm, 412
Hot Backup, 457, 658
I/O configuration, 357–363, 411
lock waits in, 740–744
log files, 411
and query cache, 326
release history, 16
row locks, 188
tables, 710, 742
tablespace, 364
transaction log, 357, 496
InnoDB locking selects, 503
innodb variable, 383
InnoDB-specific variables, 692
innodb_adaptive_checkpoint variable, 412
innodb_analyze_is_persistent variable, 197,
356
innodb_autoinc_lock_mode variable, 177,
384
innodb_buffer_pool_instances variable, 384
innodb_buffer_pool_size variable, 348
innodb_commit_concurrency variable, 373
innodb_concurrency_tickets variable, 373
innodb_data_file_path variable, 364
innodb_data_home_dir variable, 364
innodb_doublewrite variable, 368
innodb_file_io_threads variable, 702
innodb_file_per_table variable, 344, 362, 365,
414, 419, 648, 658
innodb_flush_log_at_trx_commit variable,
360, 364, 369, 418, 491, 508
innodb_flush_method variable, 344, 361, 419,
437
innodb_flush_neighbor_pages variable, 412
innodb_force_recovery variable, 195, 657
innodb_io_capacity variable, 384, 411
innodb_lazy_drop_table variable, 366
innodb_locks_unsafe_for_binlog variable,
505, 508
innodb_log_buffer_size variable, 359
innodb_log_files_in_group variable, 358
innodb_log_file_size variable, 358
innodb_max_dirty_pages_pct variable, 350
innodb_max_purge_lag variable, 367
innodb_old_blocks_time variable, 385
innodb_open_files variable, 356
```
**778 | Index**


innodb_overwrite_relay_log_info variable,
383
innodb_read_io_threads variable, 385, 702
innodb_recovery_stats variable, 359
innodb_stats_auto_update variable, 197
innodb_stats_on_metadata variable, 197, 356
innodb_stats_sample_pages variable, 196
innodb_strict_mode variable, 385
innodb_support_xa variable, 314, 330
innodb_sync_spin_loops variable, 695
innodb_thread_concurrency variable, 101,
372
innodb_thread_sleep_delay variable, 372
innodb_use_sys_stats_table variable, 197, 356
innodb_version variable, 742
innodb_write_io_threads variable, 385, 702
innotop tool, 500, 672, 693
INSERT ... SELECT statements, 28, 240, 488,
503
insert buffer, 413, 703
INSERT command, 267, 278
INSERT ON DUPLICATE KEY UPDATE
command, 252, 682
insert-to-select rate, 323
inspecting server status variables, 346
INSTEAD OF trigger, 278
instrumentation, 73
instrumentation-for-php (IfP), 78
INT type, 117
integer computations, 117
integer types, 117, 130
Intel X-25E drives, 404
Intel Xeon X5670 Nehalem CPU, 598
interface tools, 665
intermittent problems, diagnosing, 92
capturing diagnostic data, 97–102
case study, 102–110
single-query versus server-wide problems,
93–96
internal concurrency issues, 391
internal XA transactions, 314
intra-row fragmentation, 198
introducers, 300
invalidation on read, 615
ionice, 626
iostat, 438–442, 591, 646
IP addresses, 560, 584
IP takeover, 583
ISNULL() function, 254

```
isolating columns, 159
isolation, 7
iterative optimization by benchmarking, 338
```
```
J
JMeter, 51
joins, 132, 234
decomposition, 209
execution strategy, 220
JOIN queries, 244
optimizers for, 223–226
journaling filesystems, 433
Joyent, 589
```
```
K
Karlsson, Anders, 510
Karwin, Bill, 256
Keep-Alive, 608
key block size, 353
key buffers, 351
key column, 729
key_buffer_size variable, 335, 351
key_len column, 729
Köhntopp, Kristian, 252
Kyte, Tom, 76
```
```
L
L-values, 250
lag, 484, 486, 507–511
Lahdenmaki, Tapio, 158, 204
LAST() function, 255
LAST_INSERT_ID() function, 239
latency, 38, 398, 576
LATEST DETECTED DEADLOCK, 697
LATEST FOREIGN KEY ERROR, 695
Launchpad, 64
lazy UNIONs, 254
LDAP authentication, 298
Leach, Mike, 158, 204
LEAST() function, 254
LEFT JOIN queries, 219
LEFT OUTER JOIN queries, 231
left-deep trees, 223
Leith, Mark, 712
LENGTH() function, 254, 304
lighttpd, 608
lightweight profiling, 76
LIMIT query, 218, 227, 246
```
```
Index | 779
```

limited replication bandwidth, 511
linear scalability, 524
“lint checking”, 249
Linux Virtual Server (LVS), 449, 556, 560
Linux-HA stack, 582
linuxthreads, 435
Little’s Law, 441
load balancers, 561
load balancing, 449, 555–565
LOAD DATA FROM MASTER command,
457
LOAD DATA INFILE command, 79, 301, 504,
508, 511, 600, 651
LOAD INDEX command, 272, 352
LOAD TABLE FROM MASTER command,
457
LOAD_FILE() function, 281
local caches, 612
local shared-memory caches, 613
locality of reference, 393
lock contention, 503
LOCK IN SHARE MODE command, 240
LOCK TABLES command, 11, 632
lock time, 626
lock waits, 735, 740–744
lock-all-tables variable, 457
lock-free InnoDB backups, 644
locks
debugging, 735–744
granularities, 4
implicit and explicit, 11
read/write, 4
row, 5
table, 5
log buffer, 358–361
log file coordinates, 456
log file size, 344, 358–361, 411
log positions, locating, 492
log servers, 481, 654
log threads, 702
log, InnoDB transaction, 703
logging, 10, 25
logical backups, 627, 637–639, 649–651
logical concurrency issues, 391
logical reads, 395
logical replication, 460
logical unit numbers (LUNs), 423
log_bin variable, 458

```
log_slave_updates variable, 453, 465, 468, 511,
635
LONGBLOB type, 122
LONGTEXT type, 122
lookup tables, 20
loose index scans, 235
lost time, 74
low latency, 389
LOW_PRIORITY hint, 238
Lua language, 53
Lucene, 313
LucidDB, 23
LUNs (logical unit numbers), 423
LVM snapshots, 434, 633, 640–648
lvremove command, 643
LVS (Linux Virtual Server), 449, 556, 560
lzo, 626
```
```
M
Maatkit (see Percona Toolkit)
maintenance operations, 271
malloc() function, 319
manual joins, 606
mapping tables, 20
MariaDB, 19, 484, 681
master and replicas, 468, 474, 564
master shutdown, unexpected, 495
master-data variable, 457
master-master in active-active mode, 469
master-master in active-passive mode, 471
master-master replication, 473, 505
master.info file, 459, 464, 489, 496
Master_Log_File, 491
MASTER_POS_WAIT() function, 495, 564
MATCH() function, 216, 306, 307, 311
materialized views, 138, 280
Matsunobu, Yoshinori, 581
MAX() function, 217, 237, 292
Maxia, Giuseppe, 282, 456, 512, 515, 518,
667
maximum system capacity, 521, 609
max_allowed_packet variable, 381
max_connections setting variable, 378
max_connect_errors variable, 381
max_heap_table_size setting variable, 378
mbox mailbox messages, 3
MBRCONTAINS() function, 157
McCullagh, Paul, 22
MD5() function, 53, 130, 156, 507
```
**780 | Index**


md5sum, 718
mean time between failures (MTBF), 569
mean time to recover (MTTR), 569–572, 576,
582, 586
measurement uncertainty, 72
MEDIUMBLOB type, 122
MEDIUMINT type, 117
MEDIUMTEXT type, 122
memcached, 533, 546, 613, 616
Memcached Access, 618
memory
allocating for caches, 349
configuring, 347–356
consumption formula for, 341
InnoDB buffer pool, 349
InnoDB data dictionary, 356
limits on, 347
memory-to-disk ratio, 397
MyISAM key cache, 351–353
per-connection needs, 348
pool, 704
reserving for operating system, 349
size, 595
Sphinx RAM, 751
table cache, 354
thread cache, 353
Memory storage engine, 20
Merge storage engine, 21
merge tables, 273–276
merged read and write requests, 440
mget() call, 616
MHA toolkit, 581
middleman solutions, 560–563, 584
migration, benchmarking after, 46
Millsap, Cary, 70, 74, 341
MIN() function, 217, 237, 292
Mininova.org, 764
mk-parallel-dump tool, 638
mk-parallel-restore tool, 638
mk-query-digest tool, 72
mk-slave-prefetch tool, 510
MLC (multi-level cell), 402, 407
MMM replication manager, 572, 580
mod_log_config variable, 79
MonetDB, 23
Monitis, 671
monitoring tools, 667–676
MONyog, 671
mpstat tool, 438

```
MRTG (Multi Router Traffic Grapher), 430,
669
MTBF (mean time between failures), 569
mtop tool, 672
MTTR (mean time to recovery), 569–572, 576,
582, 586
Mulcahy, Lachlan, 659
Multi Router Traffic Grapher (MRTG), 430,
669
multi-level cell (MLC), 402, 407
multi-query mechanism, 753
multicolumn indexes, 163
multiple disk volumes, 427
multiple partitioning keys, 537
multisource replication, 470, 480
multivalued attributes, 757, 761
Munin, 670
MVCC (multiversion concurrency control), 12,
551
my.cnf file, 452, 490, 501
.MYD file, 371, 633, 648
mydumper, 638, 659
.MYI file, 633, 648
MyISAM storage engine, 17
and backups, 631
concurrency configuration, 18, 373
and COUNT() queries, 242
data layout, 171
delayed key writes, 19
indexes, 18, 143
key block size, 353
key buffer/cache, 351–353, 690
performance, 19
tables, 19, 498
myisamchk, 629
myisampack, 276
mylvmbackup, 658, 659
MySQL
concurrency, 371–374
configuration mechanisms, 332–337
development model, 33
GPL-licensing, 33
logical architecture, 1
proprietary plugins, 33
Sandbox script, 456, 481
version history, 29–33, 182, 188
MySQL 5.1 Plugin Development (Golubchik &
Hutchings), 298
MySQL Benchmark Suite, 52, 55
```
```
Index | 781
```

MySQL Cluster, 577
MySQL Enterprise Backup, 457, 624, 627, 631,
658
MySQL Enterprise Monitor, 80, 670
MySQL Forge, 667, 710
MySQL High Availability (Bell et al.), 519
MySQL Stored Procedure Programming
(Harrison & Feuerstein), 282
MySQL Workbench Utilities, 665
mysql-bin.index file, 464
mysql-relay-bin.index file, 464
mysqladmin, 666, 686
mysqlbinlog tool, 460, 481, 492, 654
mysqlcheck tool, 629, 666
mysqld tool, 99, 344
mysqldump tool, 456, 488, 623, 627, 637, 660
mysqlhotcopy tool, 658
mysqlimport tool, 627, 651
mysqlslap tool, 51
mysql_query() function, 212, 292
mysql_unbuffered_query() function, 212
mytop tool, 672

**N**
Nagios, 668
Nagios System and Network Monitoring
(Barth), 643, 668
name locks, 736, 739
NAS (network-attached storage), 422–427
NAT (network address translation), 584
Native POSIX Threads Library (NPTL), 435
natural identifiers, 134
natural-language full-text searches, 306
NDB API, 619
NDB Cluster storage engine, 21, 535, 549, 550,
576
nesting cursors, 290
netcat, 717
network address translation (NAT), 584
network configuration, 429–431
network overhead, 202
network performance, 595
network provider, reliance on single, 572
network-attached storage (NAS), 422–427
New Relic, 77, 671
next-key locking, 17
NFS, SAN over, 424
Nginx, 608, 612
nice, 626

```
nines rule of availability, 567
Noach, Shlomi, 187, 666, 687, 710
nodes, 531, 538
non-SELECT queries, 721
nondeterministic statements, 499
nonrepeatable reads, 8
nonreplicated data, 501
nonsharded data, 538
nontransactional tables, 498
nonunique server IDs, 500
nonvolatile random access memory (NVRAM),
400
normalization, 133–136
NOT EXISTS() queries, 219, 232
NOT NULL, 116, 682
NOW() function, 316
NOW_USEC() function, 296, 513
NPTL (Native POSIX Threads Library), 435
NULL, 116, 133, 270
null hypothesis, 47
NULLIF() function, 254
NuoDB, 22
NVRAM (nonvolatile random access memory),
400
```
```
O
object versioning, 615
object-relational mapping (ORM) tool, 131,
148, 606
OCZ, 407
OFFSET variable, 246
OLTP (online transaction processing), 22, 38,
59, 478, 509, 596
on-controller cache (see RAID)
on-disk caches, 614
on-disk temporary tables, 122
online transaction processing (OLTP), 22, 38,
59, 478, 509, 596
open() function, 363
openark kit, 666
opened tables, 355
opening and locking partitions, 271
OpenNMS, 669
operating system
choosing an, 431
how to select CPUs for MySQL, 388
optimization, 387
status of, 438–444
what limits performance, 387
```
**782 | Index**


oprofile tool, 99–102, 111
Opsview, 668
optimistic concurrency control, 12
optimization, 3
(see also application-level optimization)
(see also query optimization)
BLOB workload, 375
DISTINCT queries, 244
filesort, 377
full-text indexes, 312
GROUP BY queries, 244, 752, 768
JOIN queries, 244
LIMIT and OFFSET, 246
OPTIMIZE TABLE command, 170, 310,
501
optimizer traces, 734
optimizer_prune_level, 240
optimizer_search_depth, 240
optimizer_switch, 241
prepared statements, 292
queries, 272
query cache, 327
query optimizer, 215–220
RAID performance, 415–417
ranking queries, 250
selects on Sahibinden.com, 767
server setting optimization, 331
sharded JOIN queries on Grouply.com,
769
for solid-state storage, 410–414
sorts, 193
SQL_CALC_FOUND_ROWS variable,
248
subqueries, 244
TEXT workload, 375
through profiling, 72–75, 91
UNION variable, 248
Optimizer
hints
DELAYED, 239
FOR UPDATE, 240
FORCE INDEX, 240
HIGH_PRIORITY, 238
IGNORE INDEX, 240
LOCK IN SHARE MODE, 240
LOW_PRIORITY, 238
SQL_BIG_RESULT, 239
SQL_BUFFER_RESULT, 239
SQL_CACHE, 239

```
SQL_CALC_FOUND_ROWS, 239
SQL_NO_CACHE, 239
SQL_SMALL_RESULT, 239
STRAIGHT_JOIN, 239
USE INDEX, 240
limitations of
correlated subqueries, 229–233
equality propogation, 234
hash joins, 234
index merge optimizations, 234
loose index scans, 235
MIN() and MAX(), 237
parallel execution, 234
SELECT and UPDATE on the Same
Table, 237
UNION limitations, 233
query, 214–227
complex queries versus many queries,
207
COUNT() aggregate function, 241
join decomposition, 209
limitations of MySQL, 229–238
optimizing data access, 202–207
reasons for slow queries, 201
restructuring queries, 207–209
Optimizing Oracle Performance (Millsap), 70,
341
options, 332
OQGraph storage engine, 23
Oracle Database, 408
Oracle development milestones, 33
Oracle Enterprise Linux, 432
Oracle GoldenGate, 516
ORDER BY queries, 163, 182, 226, 253
order processing, 26
ORM (object-relational mapping), 148, 606
OurDelta, 683
out-of-sync replicas, 488
OUTER JOIN queries, 221
outer joins, 216
outliers, 74
oversized packets, 511
O_DIRECT variable, 362
O_DSYNC variable, 363
```
```
P
Pacemaker, 560, 582
packed indexes, 184
packed tables, 19
```
```
Index | 783
```

PACK_KEYS variable, 184
page splits, 170
paging, 436
PAM authentication, 298
parallel execution, 234
parallel result sets, 753
parse tree, 3
parser, 214
PARTITION BY variable, 265, 270
partitioning, 415
across multiple nodes, 531
how to use, 268
keys, 535
with replication filters, 564
sharding, 533–547, 565, 755
tables, 265–276, 329
types of, 267
passive caches, 611
Patricia tries, 158
PBXT, 22
PCIe cards, 400, 406
Pen, 556
per-connection memory needs, 348
per-connection needs, 348
percent() function, 676
percentile response times, 38
Percona InnoDB Recovery Toolkit, 657
Percona Server, 598, 679, 711
BLOB and TEXT types, 122
buffer pool, 711
bypassing operating system caches, 344
corrupted tables, 657
doublewrite buffer, 411
enhanced slow query log, 89
expand_fast_index_creation, 198
extended slow query log, 323, 330
fast warmup features, 351, 563, 598
FNV64() function, 157
HandlerSocket plugin, 297
idle transaction timeout parameter, 744
INFORMATION_SCHEMA.INDEX_STA
TISTICS table, 187
innobd_use_sys_stats_table option, 197
InnoDB online text creation, 144
innodb_overwrite_relay_log_info option,
383
innodb_read_io_threads option, 702
innodb_recovery_stats option, 359
innodb_use_sys_stats_table option, 356

```
innodb_write_io_threads option, 702
larger log files, 411
lazy page invalidation, 366
limit data dictionary size, 356, 711
mutex issues, 384
mysqldump, 628
object-level usage statistics, 110
query-level instrumentation, 73
read-ahead, 412
replication, 484, 496, 508, 516
slow query log, 74, 80, 84, 89, 95
stripping query comments, 316
temporary tables, 689, 711
user statistics tables, 711
Percona Toolkit, 666
Aspersa, 666
Maatkit, 658, 666
mk-parallel-dump tool, 638
mk-parallel-restore tool, 638
mk-query-digest tool, 72
mk-slave-prefetch tool, 510
pt-archiver, 208, 479, 504, 545, 553
pt-collect, 99, 442
pt-deadlock-logger, 697
pt-diskstats, 45, 442
pt-duplicate-key-checker, 187
pt-fifo-split, 651
pt-find, 502
pt-heartbeat, 476, 487, 492, 559
pt-index-usage, 187
pt-kill, 744
pt-log-player, 340
pt-mext, 347, 687
pt-mysql-summary, 100, 103, 347, 677
pt-online-schema-change, 29
pt-pmp, 99, 101, 390
pt-query-advisor, 249
pt-query-digest, 375, 507, 563
extracting from comments, 79
profiling, 72–75
query log, 82–84
slow query logging, 90, 95, 340
pt-sift, 100, 442
pt-slave-delay, 516, 634
pt-slave-restart, 496
pt-stalk, 98, 99, 442
pt-summary, 100, 103, 677
pt-table-checksum, 488, 495, 519, 634
pt-table-sync, 489
```
**784 | Index**


pt-tcp-model, 611
pt-upgrade, 187, 241, 570, 734
pt-visual-explain, 733
Percona tools, 52, 64–66, 195
Percona XtraBackup, 457, 624, 627, 631, 648,
658
Percona XtraDB Cluster, 516, 549, 577–580,
680
performance optimization, 69–72, 107
plotting metrics, 49
profiling, 72–75
SAN, 424
views and, 279
Performance Schema, 90
Perl scripts, 572
Perldoc, 662
perror utility, 355
persistent connections, 561, 607
persistent memory, 597
pessimistic concurrency control, 12
phantom reads, 8
PHP profiling tools, 77
phpMyAdmin tool, 666
phrase proximity ranking, 759
phrase searches, 309
physical reads, 395
physical size of disk, 399
pigz tool, 626
“pileups”, 69
Pingdom, 671
pinging, 606, 689
Planet MySQL blog aggregator, 667
planned promotions, 490
plugin-specific variables, 692
plugins, 297
point-in-time recovery, 625, 652
poor man’s profiler, 101
port forwarding, 584
possible_keys column, 729
post-mortems, 571
PostgreSQL, 258
potential cache size, 323
power grid, 572
preferring a join, 244
prefix indexes, 160–163
prefix-compressed indexes, 184
preforking, 608
pregenerating content, 617
prepared statements, 291–295, 329

```
preprocessor, 214
Preston, W. Curtis, 621
primary key, 17, 173–176
PRIMARY KEY constraint, 185
priming the cache, 509
PROCEDURE ANALYSE command, 297
procedure plugins, 297
processor speed, 392
profiling
and application speed, 76
applications, 75–80
diagnosing intermittent problems, 92–110
interpretation, 74
MySQL queries, 80–84
optimization through, 72–75, 91
single queries, 84–91
tools, 72, 110–112
promotions of replicas, 491, 583
propagation of changes, 584
proprietary plugins, 33
proxies, 556, 584, 609
pruning, 270
pt-archiver tool, 208, 479, 504, 545, 553
pt-collect tool, 99, 442
pt-deadlock-logger tool, 697
pt-diskstats tool, 45, 442
pt-duplicate-key-checker tool, 187
pt-fifo-split tool, 651
pt-find tool, 502
pt-heartbeat tool, 476, 487, 492, 559
pt-index-usage tool, 187
pt-kill tool, 744
pt-log-player tool, 340
pt-mext tool, 347, 687
pt-mysql-summary tool, 100, 103, 347, 677
pt-online-schema-change tool, 29
pt-pmp tool, 99, 101, 390
pt-query-advisor tool, 249
pt-query-digest (see Percona Toolkit)
pt-sift tool, 100, 442
pt-slave-delay tool, 516, 634
pt-slave-restart tool, 496
pt-stalk tool, 98, 99, 442
pt-summary tool, 100, 103, 677
pt-table-checksum tool, 488, 495, 519, 634
pt-table-sync tool, 489
pt-tcp-model tool, 611
pt-upgrade tool, 187, 241, 570, 734
pt-visual-explain tool, 733
```
```
Index | 785
```

PURGE MASTER LOGS command, 369, 464,
486
purging old binary logs, 636
pushdown joins, 550, 577

**Q**
Q mode, 673
Q4M storage engine, 23
Qcache_lowmem_prunes variable, 325
query cache, 214, 315, 330, 690
alternatives to, 328
configuring and maintaining, 323–325
InnoDB and the, 326
memory use, 318
optimizations, 327
when to use, 320–323
query execution
MySQL client/server protocol, 210–213
optimization process, 214
query cache, 214, 315–328
query execution engine, 228
query logging, 95
query optimization, 214–227
complex queries versus many queries, 207
COUNT() aggregate function, 241
join decomposition, 209
limitations of MySQL, 229–238
optimizing data access, 202–207
reasons for slow queries, 201
restructuring queries, 207–209
query states, 213
query-based splits, 557
querying across shards, 537
query_cache_limit variable, 324
query_cache_min_res_unit value variable, 324
query_cache_size variable, 324, 336
query_cache_type variable, 323
query_cache_wlock_invalidate variable, 324
queue scheduler, 434
queue tables, 256
queue time, 204
quicksort, 226

**R**
R-Tree indexes, 157
Rackspace Cloud, 589
RAID
balancing hardware and software, 418

```
configuration and caching, 419–422
failure, recovery, and monitoring, 417
moving files from flash to, 411
not for backup, 624
performance optimization, 415–417
splits, 647
with SSDs, 405
RAND() function, 160, 724
random read-ahead, 412
random versus sequential I/O, 394
RANGE COLUMNS type, 268
range conditions, 192
raw file
backup, 627
restoration, 648
RDBMS technology, 400
RDS (Relational Database Service), 589, 600
read buffer size, 343
READ COMMITTED isolation level, 8, 13
read locks, 4, 189
read threads, 703
READ UNCOMMITTED isolation level, 8, 13
read-ahead, 412
read-around writes, 353
read-mostly tables, 26
read-only variable, 26, 382, 459, 479
read-write splitting, 557
read_buffer_size variable, 336
Read_Master_Log_Pos, 491
read_rnd_buffer_size variable, 336
real number data types, 118
rebalancing shards, 544
records_in_range() function, 195
recovery
from a backup, 647–658
defined, 622
defining requirements, 623
more advanced techniques, 653
recovery point objective (RPO), 623, 625
recovery time objective (RTO), 623, 625
Red Hat, 432, 683
Redis, 620
redundancy, replication-based, 580
Redundant Array of Inexpensive Disks (see
RAID)
redundant indexes, 185–187
ref column, 730
```
**786 | Index**


Relational Database Index Design and the
Optimizers (Lahdenmaki & Leach),
158, 204
Relational Database Service (RDS), Amazon,
589, 600
relay log, 450, 496
relay-log.info file, 464
relay_log variable, 453, 459
relay_log_purge variable, 459
relay_log_space_limit variable, 459, 511
RELEASE_LOCK() function, 256
reordering joins, 216
REORGANIZE PARTITION command, 271
REPAIR TABLE command, 144, 371
repairing MyISAM tables, 18
REPEATABLE READ isolation level, 8, 13,
632
replica hardware, 414
replica shutdown, unexpected, 496
replicate_ignore_db variable, 478
replication, 447, 634
administration and maintenance, 485
advanced features in MySQL, 514
backing up configuration, 630
and capacity planning, 482–485
changing masters, 489–494
checking consistency of, 487
checking for up-to-dateness, 565
configuring master and replica, 452
creating accounts for, 451
custom solutions, 477–482
filtering, 466, 564
how it works, 449
initializing replica from another server, 456
limitations, 512
master and multiple replicas, 468
master, distribution master, and replicas,
474
master-master in active-active mode, 469
master-master in active-passive mode, 471
master-master with replicas, 473
measuring lag, 486
monitoring, 485
other technologies, 516
problems and solutions, 495–512
problems solved by, 448
promotions of replicas, 491, 583
recommended configuration, 458
replica consistency with master, 487

```
replication files, 463
resyncing replica from master, 488
ring, 473
row-based, 447, 460–463
sending events to other replicas, 465
setting up, 451
speed of, 512–514
splitting reads and writes in, 557
starting the replica, 453–456
statement-based, 447, 460–463
status, 708
switching master-master configuration
roles, 494
topologies, 468, 490
tree or pyramid, 476
REPLICATION CLIENT privilege, 452
REPLICATION SLAVE privilege, 452
replication-based redundancy, 580
RESET QUERY CACHE command, 325
RESET SLAVE command, 490
resource consumption, 70
response time, 38, 69, 204
restoring
defined, 622
logical backups, 649–651
RethinkDB, 22
ring replication, 473
ROLLBACK command, 499
round-robin database (RRD) files, 669
row fragmentation, 198
row locks, 5, 12
ROW OPERATIONS, 705
row-based logging, 636
row-based replication, 447, 460–463
rows column, 731
rows examined, number of, 205
rows returned, number of, 205
ROW_COUNT command, 287
RPO (recovery point objective), 623, 625
RRDTool, 669
rsync, 195, 456, 717, 718
RTO (recovery time objective), 623, 625
running totals and averages, 255
```
```
S
safety and sanity settings, 380–383
Sahibinden.com, 767
SandForce, 407
SANs (storage area networks), 422–427
```
```
Index | 787
```

sar, 438
sargs, 166
SATA SSDs, 405
scalability, 521
by clustering, 548
by consolidation, 547
frequency, 392
and load balancing, 555
mathematical definition, 523
multiple CPUs/cores, 391
planning for, 527
preparing for, 528
“scale-out” architecture, 447
scaling back, 552
scaling out, 531–547
scaling pattern, 391
scaling up, 529
scaling writes, 483
Sphinx, 754
universal law of, 525–527
scalability measurements, 39
ScaleArc, 547, 549
ScaleBase, 547, 549, 551, 594
ScaleDB, 407, 574
scanning data, 269
scheduled tasks, 504
schemas, 13
changes, 29
design, 131
normalized and denormalized, 135
Schooner Active Cluster, 549
scope, 333
scp, 716
search engine, selecting the right, 24–28
search space, 226
searchd, Sphinx, 746, 754, 756–766
secondary indexes, 17, 656
security, connection management, 2
sed, 638
segmented key cache, 19
segregating hot data, 269
SELECT command, 237, 267, 721
SELECT FOR UPDATE command, 256, 287
SELECT INTO OUTFILE command, 301, 504,
508, 600, 638, 651, 657
SELECT types, 690
selective replication, 477
selectivity, index, 160
select_type column, 724

```
SEMAPHORES, 693
sequential versus random I/O, 394
sequential writes, 576
SERIALIZABLE isolation level, 8, 13
serialized writes, 509
server, 685
adding/removing, 563
configuration, backing up, 630
consolidation, 425
INFORMATION_SCHEMA database, 711
MySQL configuration, 332
PERFORMANCE_SCHEMA database,
712
profiling and speed of, 76, 80
server-wide problems, 93–96
setting optimization, 331
SHOW ENGINE INNODB MUTEX
command, 707–709
SHOW ENGINE INNODB STATUS
command, 692–706
SHOW PROCESSLIST command, 706
SHOW STATUS command, 686–692
status variables, 346
workload profiling, 80
server-side prepared statements, 295
service time, 204
session scope, 333
session-based splits, 558
SET CHARACTER SET command, 300
SET GLOBAL command, 494
SET GLOBAL SQL_SLAVE_SKIP_COUNTER
command, 654
SET NAMES command, 300
SET NAMES utf8 command, 300, 606
SET SQL_LOG_BIN command, 503
SET TIMESTAMP command, 635
SET TRANSACTION ISOLATION LEVEL
command, 11
SET type, 128, 130
SetLimits() function, 748, 764
SetMaxQueryTime() function, 764
SeveralNines, 550, 577
SHA1() function, 53, 130, 156
Shard-Query system, 547
sharding, 533–547, 565, 755
shared locks, 4
shared storage, 573–576
SHOW BINLOG EVENTS command, 486,
708
```
**788 | Index**


SHOW commands, 255
SHOW CREATE TABLE command, 117, 163
SHOW CREATE VIEW command, 280
SHOW ENGINE INNODB MUTEX
command, 695, 707–709
SHOW ENGINE INNODB STATUS
command, 97, 359, 366, 384, 633,
692–706, 740
SHOW FULL PROCESSLIST command, 81,
700
SHOW GLOBAL STATUS command, 88, 93,
346, 686
SHOW INDEX command, 197
SHOW INDEX FROM command, 196
SHOW INNODB STATUS command (see
SHOW ENGINE INNODB STATUS
command)
SHOW MASTER STATUS command, 452,
457, 486, 490, 558, 630, 643
SHOW PROCESSLIST command, 94–96, 256,
289, 606, 706
SHOW PROFILE command, 85–89
SHOW RELAYLOG EVENTS command, 708
SHOW SLAVE STATUS command, 453, 457,
486, 491, 558, 630, 708
SHOW STATUS command, 88, 352
SHOW TABLE STATUS command, 14, 197,
365, 672
SHOW VARIABLES command, 352, 685
SHOW WARNINGS command, 222, 277
signed types, 117
single-component benchmarking, 37, 51
single-level cell (SLC), 402, 407
single-shard queries, 535
single-transaction variable, 457, 632
skip_innodb variable, 476
skip_name_resolve variable, 381, 429, 570
skip_slave_start variable, 382, 459
slavereadahead tool, 510
slave_compressed_protocol variable, 475, 511
slave_master_info variable, 383
slave_net_timeout variable, 382
Slave_open_temp_tables variable, 503
SLC (single-level cell), 402, 407
Sleep state, 607
SLEEP() function, 256, 682, 737
sleeping before entering queue, 373
slots, 694
slow queries, 71, 74, 80, 89, 109, 321

```
SMALLBLOB type, 122
SMALLINT type, 117
SMALLTEXT type, 122
Smokeping tool, 430
snapshots, 457, 624, 640–648
Solaris SPARC hardware, 431
Solaris ZFS filesystem, 431
solid-state drives (SSD), 147, 268, 361, 404
solid-state storage, 400–414
sort buffer size, 343
sort optimizations, 226, 691
sorting, 193
sort_buffer_size variable, 336
Souders, Steve, 608
SourceForge, 52
SPARC hardware, 431
spatial indexes, 157
Sphinx, 313, 619, 745, 770
advanced performance control, 763
applying WHERE clauses, 750
architectural overview, 756–758
efficient and scalable full-text searching,
749
filtering, 761
finding top results in order, 751
geospatial search functions, 262
installation overview, 757
optimizing GROUP BY queries, 752, 768
optimizing selects on Sahibinden.com, 767
optimizing sharded JOIN queries on
Grouply.com, 769
phrase proximity ranking, 759
searching, 746–748
special features, 759–764
SphinxSE, 756, 759, 761, 767
support for attributes, 760
typical partition use, 758
Spider storage engine, 24
spin-wait, 695
spindle rotation speed, 399
splintering, 533–547
split-brain syndrome, 575, 578
splitting reads and write in replication, 557
Splunk, 671
spoon-feeding, 608
SQL and Relational Theory (Date), 255
SQL Antipatterns (Karwin), 256
SQL dumps, 637
SQL interface prepared statements, 295
```
```
Index | 789
```

SQL slave thread, 450
SQL statements, 638
SQL utilities, 667
sql-bench, 52
SQLyog tool, 665
SQL_BIG_RESULT hint, 239, 245
SQL_BUFFER_RESULT hint, 239
SQL_CACHE hint, 239
SQL_CACHE variable, 321, 328
SQL_CALC_FOUND_ROWS hint, 239
SQL_CALC_FOUND_ROWS variable, 248
sql_mode, 382
SQL_MODE configuration variable, 245
SQL_NO_CACHE hint, 239
SQL_NO_CACHE variable, 328
SQL_SMALL_RESULT hint, 239, 245
Squid, 608
SSD (solid-state drives), 147, 268, 361, 404
SSH, 716
staggering numbers, 505
stale-data splits, 557
“stalls”, 69
Starkey, Jim, 22
START SLAVE command, 654
START SLAVE UNTIL command, 654
start-position variable, 498
statement handles, 291
statement-based replication, 447, 460–463
static optimizations, 216
static query analysis, 249
STEC, 407
STONITH, 584
STOP SLAVE command, 487, 490, 498
stopwords, 306, 312
storage area networks (SANs), 422–427
storage capacity, 399
storage consolidation, 425
storage engine API, 2
storage engines, 13, 23–28
Archive, 19
Blackhole, 20
column-oriented, 22
community, 23
and consistency, 633
CSV, 20
Falcon, 22
Federated, 20
InnoDB, 15
Memory, 20

```
Merge, 21
mixing, 11, 500
MyISAM, 18
NDB Cluster, 21
OLTP, 22
ScaleDB, 574
XtraDB, 680
stored code, 282–284, 289
Stored Procedure Library, 667
stored procedures and functions, 284
stored routines, 282, 329
strace tool, 99, 111
STRAIGHT_JOIN hint, 224, 239
string data types, 119–125, 130
string locks, 736
stripe chunk size, 420
subqueries, 218, 244
SUBSTRING() function, 122, 304, 375
sudo rules, 630
SUM() function, 139
summary tables, 136
Super Smack, 52
surrogate keys, 173
Swanhart, Justin, 138, 280, 547
swapping, 436, 444
switchover, 582
synchronization, two-way, 287
synchronous MySQL replication, 576–580
sync_relay_log variable, 383
sync_relay_log_info variable, 383
sysbench, 39, 53, 56–61, 419, 426, 598
SYSDATE() function, 382
sysdate_is_now variable, 382
system of record approach, 517
system performance, benchmarking, 44
system under test (SUT), 44
system variables, 685
```
```
T
table definition cache, 356
tables
building a queue, 256
cache memory, 354
column, 724–727
conversions, 28
derived, 238, 277, 725
finding and repairing corruption, 194
INFORMATION_SCHEMA in Percona
Server, 711
```
**790 | Index**


locks, 5, 692, 735–738
maintenance, 194–198
merge, 273–276
partitioned, 265–276, 329
reducing to an MD5 hash value, 255
SELECT and UPDATE on, 237
SHOW TABLE STATUS output, 14
splitting, 554
statistics, 220
tablespaces, 16, 364
views, 276–280
table_cache_size variable, 335, 379
tagged cache, 615
TCP, 556, 583
tcpdump tool, 81, 95, 99
tcp_max_syn_backlog variable, 430
temporal computations, 125
temporary files and tables, 21, 502, 689, 711
TEMPTABLE algorithm, 277
Texas Memory Systems, 407
TEXT type, 21, 121, 122
TEXT workload, optimizing for, 375
Theory of Constraints, 526
third-party storage engines, 21
thread and connection statistics, 688
thread cache memory, 353
threaded discussion forums, 27
threading, 213, 435
Threads_connected variable, 354, 596
Threads_created variable, 354
Threads_running variable, 596
thread_cache_size variable, 335, 354, 379
throttling variables, 627
throughput, 38, 70, 398, 576
tickets, 373
time to live (TTL), 614
time-based data partitioning, 554
TIMESTAMP type, 117, 126, 631
TIMESTAMPDIFF() function, 513
TINYBLOB type, 122
TINYINT type, 117
TINYTEXT type, 122
Tkachenko, Vadim, 405
tmp_table_size setting, 378
TokuDB, 22, 158
TO_DAYS() function, 268
TPC Benchmarks
dbt2, 61
TPC-C, 52

```
TPC-H, 41
TPCC-MySQL tool, 52, 64–66
transactional tables, 499
transactions, 24
ACID test, 6
deadlocks, 9
InnoDB, 366, 699
isolation levels, 7
logging, 10
in MySQL, 10
and storage engines, 24
transfer speed, 398
transferring large files, 715–718
transparency, 556, 578, 611
tree or pyramid replication, 476
tree-formatted output, 733
trial-and-error troubleshooting, 92
triggers, 97, 282, 286
TRIM command, 404
Trudeau, Yves, 262
tsql2mysql tool, 282
TTL (time to live), 614
tunefs, 433
Tungsten Replicator, Continuent, 481, 516
“tuning”, 340
turbo boost technology, 392
type column, 727
```
```
U
Ubuntu, 683
UDF Library, 667
UDFs, 262, 295
unarchiving, 553
uncommitted data, 8
uncompressed files, 715
undefined server IDs, 501
underutilization, 485
UNHEX() function, 130
UNION ALL query, 248
UNION limitations, 233
UNION query, 220, 248, 254, 724–727
UNION syntax, 274
UNIQUE constraint, 185
unit of sharding, 535
Universal Scalability Law (USL), 525–527
Unix, 332, 432, 504, 582, 630
UNIX_TIMESTAMP() function, 126
UNLOCK TABLES command, 12, 142, 643
UNSIGNED attribute, 117
```
```
Index | 791
```

“unsinkable” systems, 573
unused indexes, 187
unwrapping, 255
updatable views, 278
UPDATE command, 237, 267, 278
UPDATE RETURNING command, 252
upgrades
replication before, 449
validating MySQL, 241
USE INDEX hint, 240
user logs, 740
user optimization issues, 39, 166
user statistics tables, 711
user-defined functions (UDFs), 262, 295
user-defined variables, 249–255
USER_STATISTICS tables, 110
“Using filesort” value, 733
“Using index” value, 733
USING query, 218
“Using temporary” value, 733
“Using where” value, 733
USL (Universal Scalability Law), 525–527
UTF-8, 298, 303
utilities, SQL, 667
UUID() function, 130, 507, 546
UUID_SHORT() function, 546

**V**
Valgrind, 78
validating MySQL upgrades, 241
VARCHAR type, 119, 124, 131, 513
variables, 332
assignments in statements, 255
setting dynamically, 335–337
user-defined, 249–255
version-based splits, 558
versions
and full-text searching, 310
history of MySQL, 29–33
improvements in MySQL 5.6, 734
old row, 366
replication before upgrading, 449
version-specific comments, 289
vgdisplay command, 642
views, 276–280, 329
Violin Memory, 407
Virident, 403, 409
virtual IP addresses, 560, 583
virtualization, 548

```
vmstat tool, 436, 438, 442, 591, 646
volatile memory, 597
VoltDB, 549
volume groups, 641
VPForMySQL storage engine, 24
```
```
W
Wackamole, 556
waiters flag, 694
warmup, 351, 573
wear leveling, 401
What the Dog Saw (Gladwell), 571
WHERE clauses, 255, 750
whole number data types, 117
Widenius, Monty, 679, 681
Windows, 504
WITH ROLLUP variable, 246
Workbench Utilities, MySQL, 665, 666
working concurrency, 39
working sets of data, 395, 597
workload-based configuration, 375–377
worst-case selectivity, 162
write amplification, 401
write cache and power failure, 405
write locks, 4, 189
write synchronization, 565
write threads, 703
write-ahead logging, 10, 395
write-invalidate policy, 614
write-set replication, 577
write-update, 614
writes, scaling, 483
WriteThrough vs. WriteBack, 418
```
```
X
X-25E drives, 404
X.509 certificates, 2
x86 architecture, 390, 431
XA transactions, 314, 330
xdebug, 78
Xeround, 549, 602
xhprof tool, 77
XtraBackup, Percona, 457, 624, 627, 631, 648,
658
XtraDB Cluster, Percona, 516, 549, 577–580,
680
```
**792 | Index**


**Y**
YEAR() function, 268, 270

**Z**
Zabbix, 668
Zenoss, 669
ZFS filer, 631, 640
ZFS filesystem, 408, 431
zlib, 19, 511
Zmanda Recovery Manager (ZRM), 659

```
Index | 793
```


**About the Authors**

**Baron Schwartz** is a software engineer who lives in Charlottesville, Virginia, and goes
by the online handle of “Xaprb,” which is his first name typed in QWERTY on a Dvorak
keyboard. When he’s not busy solving a fun programming challenge, he relaxes with
his wife, Lynn, and dog, Carbon. He blogs about software engineering at _[http://www](http://www)
.xaprb.com/blog/_.

A former manager of the High Performance Group at MySQL AB, **Peter Zaitsev** now
runs the mysqlperformanceblog.com site. He specializes in helping administrators fix
issues with websites handling millions of visitors a day, dealing with terabytes of data
using hundreds of servers. He is used to making changes and upgrades both to hardware
and to software (such as query optimization) in order to find solutions. He also speaks
frequently at conferences.

**Vadim Tkachenko** was a Performance Engineer in at MySQL AB. As an expert in
multithreaded programming and synchronization, his primary tasks were benchmarks,
profiling, and finding bottlenecks. He also worked on a number of features for perfor-
mance monitoring and tuning, and getting MySQL to scale well on multiple CPUs.

**Colophon**

The animal on the cover of _High Performance MySQL_ is a sparrow hawk ( _Accipiter
nisus_ ), a small woodland member of the falcon family found in Eurasia and North
Africa. Sparrow hawks have a long tail and short wings; males are bluish-gray with a
light brown breast, and females are more brown-gray and have an almost fully white
breast. Males are normally somewhat smaller (11 inches) than females (15 inches).

Sparrow hawks live in coniferous woods and feed on small mammals, insects, and birds.
They nest in trees and sometimes on cliff ledges. At the beginning of the summer, the
female lays four to six white eggs, blotched red and brown, in a nest made in the boughs
of the tallest tree available. The male feeds the female and their young.

Like all hawks, the sparrow hawk is capable of bursts of high speed in flight. Whether
soaring or gliding, the sparrow hawk has a characteristic flap-flap-glide action; its large
tail enables the hawk to twist and turn effortlessly in and out of cover.

The cover image is a nineteenth-century engraving from the Dover Pictorial Archive.
The cover font is Adobe ITC Garamond. The text font is Linotype Birka; the heading
font is Adobe Myriad Condensed; and the code font is LucasFont’s TheSansMono-
Condensed.
