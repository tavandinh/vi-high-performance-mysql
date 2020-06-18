

```
APPENDIX A
```
```
Forks and Variants of MySQL
```
In Chapter 1, we discussed the history of MySQL’s acquisition by Sun Microsystems,
then Sun’s acquisition by Oracle Corporation, and how the server has fared through
these stewardship changes. But there is much more to the story. MySQL isn’t available
solely from Oracle anymore. In the process of two acquisitions, several variants of
MySQL appeared. Although most users are unlikely to want anything but the “official”
version of MySQL from Oracle, the variants are genuinely important and have made a
big difference to all MySQL users—even those who would never consider using them.

There have been a handful of MySQL variants over the years, but three major variants
have stood the test of time so far. These three are Percona Server, MariaDB, and Drizzle.
All of them have active user communities and some degree of commercial backing. All
are supported by independent service providers.

As the creators of Percona Server, we’re biased to some extent, but we think this ap-
pendix is fairly objective because we provide services, support, consulting, training,
and engineering for all of the variants of MySQL. We also invited Brian Aker and Monty
Widenius, who created the Drizzle and MariaDB projects, respectively, to contribute
to this appendix, so that it wouldn’t just be our version of the story.

### Percona Server

Percona Server ( _[http://www.percona.com/software/](http://www.percona.com/software/)_ ) grew out of our efforts to solve
customer problems. In the second edition of this book, we mentioned some patches
that we had created to enhance the MySQL server’s logging and instrumentation. That
was really the genesis of Percona Server. We modified the server’s source code when
we encountered problems that could not be solved any other way.

```
679
```

Percona Server has three primary goals:

_Transparency_
Added instrumentation permits users to inspect the server internals and behavior
more closely. This includes features such as counters in SHOW STATUS, tables in the
INFORMATION_SCHEMA, and especially added verbosity in the slow query log.

_Performance_
Percona Server includes many improvements to performance and scalability. Raw
performance is important, but Percona Server also enhances predictability and the
stability of performance. Most of the focus is on InnoDB.

_Operational flexibility_
Percona Server contains many features that remove limitations. Although some of
the limitations seem small, they can make it hard for operations staff and system
administrators to run MySQL as a reliable and stable component of their infra-
structure.

Percona Server is a backwards-compatible drop-in replacement for MySQL, with min-
imal changes that do not alter SQL syntax, the client/server protocol, or file formats
on disk.^1 Anything that runs on MySQL will run without modification on Percona
Server. Switching to Percona Server requires only shutting down MySQL and starting
Percona Server, with no need to export and reimport data. Switching back is similarly
painless, and this is actually very important: many problems have been solved by
switching temporarily, using the improved instrumentation to diagnose the problem,
and then reverting to standard MySQL.

We choose enhancements that deviate from standard MySQL only where needed and
that provide significant benefit. We believe that most users are best served by sticking
to the official version of MySQL as distributed by Oracle, and strive to remain as close
to this as possible.

Percona Server includes the Percona XtraDB storage engine, Percona’s enhanced ver-
sion of InnoDB. This is also a backward-compatible replacement. For example, if you
create a table with the InnoDB storage engine, Percona Server recognizes it automati-
cally and uses Percona XtraDB instead. Percona XtraDB is also included in MariaDB.

Some of the enhancements in Percona Server have been included into Oracle’s version
of MySQL, and many others have been reimplemented in slightly different ways. As a
result, Percona Server has become a sort of early-access preview to features that some-
times appear later in standard MySQL. Many of the enhancements in Percona Server
5.1 and 5.5 are probably going to be reimplemented in MySQL 5.6.

1. Historically, there have been a few changes to file formats, but these are disabled by default and can be
    enabled if desired.

**680 | Appendix A: Forks and Variants of MySQL**


### MariaDB

After Sun’s MySQL acquisition Monty Widenius, the cofounder of MySQL, left Sun
Microsystems over disagreements about the MySQL development process. He founded
Monty Program AB and created MariaDB to foster an “open development environment
that would encourage outside participation.” MariaDB’s goals are community devel-
opment, along with bug fixes and lots of new features—especially integration of com-
munity-developed features. To quote Monty again,^2 “the vision for MariaDB is for it
to be user and customer driven, as well as more inclusive of community patches and
plugins.”

What’s different about MariaDB? As compared to Percona Server, it includes much
more extensive changes to the server. (Most of Percona Server’s big changes are in the
Percona XtraDB storage engine, not the server level.) There are many changes to the
query optimizer and replication, for example. And it uses the Aria storage engine for
internal temporary tables (those used for complex queries, such as DISTINCT or subqu-
eries) instead of MyISAM. Aria was originally named Maria, and was intended as an
InnoDB replacement during the uncertain Sun times. It is essentially a crash-safe version
of MyISAM.

In addition to Percona XtraDB and Aria, MariaDB also includes a number of commu-
nity storage engines, such as SphinxSE and PBXT.

MariaDB is a superset of stock MySQL, so existing applications should keep working
with no changes, just as with Percona Server. However, MariaDB will work much better
for some scenarios, such as complex subqueries or many-table joins. It also features a
segmented MyISAM key cache, which makes MyISAM much more scalable on modern
hardware.

Perhaps the finest work in MariaDB, however, is in MariaDB 5.3, which is in Release
Candidate status at the time of writing. This version includes an enormous amount of
work on the query optimizer—probably the biggest optimizer improvements MySQL
has seen in a decade. It adds new query execution plans such as hash joins, and fixes
many of the things we’ve pointed out as weaknesses in MySQL throughout this book,
such as outside-in subquery execution. It also includes significant extensions to the
server, such as dynamic columns, role-based access control, and microsecond time-
stamp support.

For a more complete list of improvements to MariaDB, you can read the documentation
on _[http://www.askmonty.org](http://www.askmonty.org)_ or these web pages that summarize the changes: _[http://](http://)
askmonty.org/blog/the-2-year-old-mariadb/_ and _[http://kb.askmonty.org/en/what-is-ma](http://kb.askmonty.org/en/what-is-ma)
riadb-53_.

2. The quotes are from _[http://monty-says.blogspot.com/2009/02/time-to-move-on.html](http://monty-says.blogspot.com/2009/02/time-to-move-on.html)_ and _[http://monty-says](http://monty-says)_
    _.blogspot.com/2010/03/time-flies-one-year-of-mariadb.html_.

```
MariaDB | 681
```

### Drizzle

Drizzle is a true fork of MySQL, not just a variant or enhancement. It is not compatible
with MySQL, although it’s not so different that it’s unrecognizable. In most cases, you
won’t simply be able to switch out your MySQL backend and replace it with Drizzle,
due to changes such as different SQL syntax.

Drizzle was created in 2008 to better serve the needs of MySQL users. It is built to
satisfy the core functionality needed by web applications. It is greatly streamlined and
simplified compared to MySQL, with many fewer choices; for example, it uses only
utf8 for character storage, and there is only one type of BLOB. It is built primarily for
64-bit hardware, and it supports IPv6 networking.

One of the key goals of the Drizzle database server is to eliminate surprises and legacy
behaviors that exist in MySQL, such as declaring a column NOT NULL and then finding
that the database somehow stored a NULL into it. The poorly implemented or unwieldy
features you can find in MySQL, such as triggers, the query cache, and INSERT ON
DUPLICATE KEY UPDATE, are simply removed.

At the code level, Drizzle is built on a microkernel architecture with a lean core and
plugins. The core of the server has been stripped down to a much smaller codebase
than MySQL. Nearly everything is a plugin—even functions such as SLEEP(). This
makes Drizzle very easy and productive to work with on the source code level.

Drizzle uses standard open-source libraries such as Boost, and complies to standards
in code, build infrastructure, and APIs. It uses the Google Protocol Buffers open mes-
saging format for purposes such as replication, and it uses a modified version of
InnoDB as its default storage engine.

The Drizzle team began benchmarking the server very early, using industry-standard
benchmarks with up to 1,024 threads to measure performance at high concurrencies.
Performance gains at high concurrency are prioritized over low-end gains, and there
has been much progress on improving performance.

Drizzle is a community-developed project, and it has attracted more open source con-
tributions than MySQL was ever able to. The server’s license is pure GPL, with no dual
licensing. However—and this is one of the most important aspects for developing a
commercial ecosystem—there is a new client library that speaks the MySQL client-
server protocol but is BSD-licensed. This means that you can build a proprietary ap-
plication that connects to MySQL through the Drizzle client library, and you do not
need to purchase a commercial license to the MySQL client library or make your soft-
ware available under the GPL license. The _libmysql_ client library for MySQL was one
of the primary reasons that companies purchased commercial licenses for MySQL—
without the commercial license that permitted them to link to _libmysql_ , they would
have been forced to release their software under the GPL. This is no longer necessary,
because now they can use the Drizzle library instead.

**682 | Appendix A: Forks and Variants of MySQL**


Drizzle is deployed in some production environments, but not very widely from what
we’ve seen. The Drizzle project’s philosophy of casting off the chains of backward-
compatibility means that it’s probably a better candidate for new applications than for
migrating an existing application.

### Other MySQL Variants

There are, or have been, many other variants of the MySQL server. Many large com-
panies, such as Google, Facebook, and eBay, maintain modified versions of the server
that suit their precise needs and deployment scenarios. Much of this source code has
been made available publicly; perhaps the best-known examples are the Facebook and
Google patches for MySQL.

In addition, there have been several forks or redistributions, such as OurDelta,
DorsalSource, and, for a brief time, a distribution from Henrik Ingo.

Finally, many people don’t realize that when they install MySQL from their GNU/
Linux distribution’s package repositories, they are actually getting a modified version
of the server—in some cases, quite heavily modified. Red Hat and Debian (and there-
fore Fedora and Ubuntu) ship a nonstandard version of MySQL, as does Gentoo and
practically every other GNU/Linux distribution. In contrast to the other variants we’ve
mentioned, these distributions don’t advertise how much they’ve changed the server’s
source code, because they keep the MySQL name.

We’ve had a lot of problems in the past with such modified versions of MySQL. This
is one reason that we tend to advocate using Oracle’s version of MySQL unless there
is a compelling reason to do otherwise.

### Summary

The forks and variants of MySQL have rarely resulted in significant amounts of code
being adopted back into the main MySQL source code tree, but they have nevertheless
influenced the direction and pace of MySQL development greatly. In some cases they
provide a superior alternative.

Should you use a fork instead of Oracle’s official MySQL? We don’t think this is usually
necessary. The choice is usually based on perceptions (which are never completely
accurate) or business reasons, such as having an enterprise-wide relationship with
Oracle. There are two general categories of people who tend to turn away from the
official version of the server:

```
Summary | 683
```

- Those who are facing a specific problem that can’t be solved without a source code
    modification
- Those who distrust Oracle’s stewardship of MySQL^3 and feel happier with a variant
    that they regard as being truly open-source

Why would you choose any specific fork? We’d summarize it as follows. If you want
to stay as close as possible to official MySQL, but get better performance, instrumen-
tation, and helpful features, choose Percona Server. Choose MariaDB if you are more
comfortable with big changes to the server, or if you want a broader range of community
contributions such as additional storage engines. Choose Drizzle if you want a lean,
stripped-down database server and you don’t mind that it’s not compatible with
MySQL, or if you want to be able to make your own enhancements to the database
much more easily.

How popular are the forks and variants? Nobody really knows, but one thing we all
agree on is that if you add together all the deployments of unofficial MySQL versions,
they constitute only a tiny fraction of the number of official MySQL deployments in
the world. In terms of relative popularity, we’re biased because many of our customers
choose to use Percona Server, but from what we’ve see deployed “in the wild,” Percona
Server appears to be the most popular, followed by MariaDB.

Speaking of Percona, in general all of the service providers have a lot of experience with
the official MySQL, but naturally Percona has the most experts in working with Percona
Server, and Monty Program is correspondingly the most familiar with MariaDB. This
matters a lot when you’re looking for bug-fix support contracts. Only Oracle can guar-
antee that a bug will be fixed in the official MySQL releases; other vendors can provide
fixes but have no power to get them included in the official releases. This is one answer
to the question of why to choose a fork: some people choose one of the forks simply
because it is the version of MySQL that their service provider controls fully and can
conveniently fix and enhance.

3. As we explained in Chapter 1, we are actually quite happy with Oracle as MySQL’s owner

**684 | Appendix A: Forks and Variants of MySQL**

