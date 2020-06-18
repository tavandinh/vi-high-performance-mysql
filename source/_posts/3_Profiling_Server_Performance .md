

```
CHAPTER 3
```
```
Profiling Server Performance
```
The three most common performance-related requests we receive in our consulting
practice are to find out whether a server is doing all of its work optimally, to find out
why a specific query is not executing quickly enough, and to troubleshoot mysterious
intermittent incidents, which users commonly call “stalls,” “pileups,” or “freezes.” This
chapter is a direct response to those three types of requests. We’ll show you tools and
techniques to help you speed up a server’s overall workload, speed up a single query,
or troubleshoot and solve a problem when it’s hard to observe, and you don’t know
what causes it or even how it manifests.

This might seem like a tall order, but it turns out that a simple method can show you
the signal within the noise. That method is to focus on measuring what the server spends
its time doing, and the technique that supports this is called _profiling_. In this chapter,
we’ll show you how to measure systems and generate profiles, and we’ll show you how
to profile your whole stack, from the application to the database server to individual
queries.

But you must empty your cup before you can fill it, so let’s dispel a few common mis-
conceptions about performance first. This gets a bit dense, so stay with us and we’ll
explain it all with examples later.

### Introduction to Performance Optimization

Ask 10 people to define performance and you’ll probably get 10 different answers,
filled with terms such as “queries per second,” “CPU utilization,” “scalability,” and so
on. This is fine for most purposes, because people understand performance differently
in different contexts, but we will use a formal definition in this chapter. Our definition
is that performance is measured by the time required to complete a task. In other words,
_performance is response time_. This is a very important principle. We measure
performance by tasks and time, not by resources. A database server’s purpose is to
execute SQL statements, so the tasks we care about are queries or statements—the

```
69
```

bread-and-butter SELECT, UPDATE, INSERT, and so on.^1 A database server’s performance
is measured by query response time, and the unit of measurement is time per query.

Now for another rhetorical question: what is optimization? We’ll return to this later,
but for now let’s agree that _performance_ optimization is the practice of reducing re-
sponse time as much as possible^2 for a given workload.

We find that many people are very confused about this. If you think performance op-
timization requires you to reduce CPU utilization, for example, you’re thinking about
reducing resource consumption. But this is a trap. Resources are there to be consumed.
Sometimes making things faster requires that you _increase_ resource consumption.
We’ve upgraded many times from an old version of MySQL with an ancient version of
InnoDB, and witnessed a dramatic increase in CPU utilization as a result. This is usually
nothing to be concerned about. It usually means that the newer version of InnoDB is
spending more time doing useful work and less time fighting with itself. Looking at
query response time is the best way to know whether the upgrade was an improvement.
Sometimes an upgrade introduces a bug such as not using an index, which can also
manifest as increased CPU utilization. CPU utilization is a symptom, not a goal, and
it’s best to measure the goal, or you could get derailed.

Similarly, if you thought that performance optimization was about improving queries
per second, then you were thinking about throughput optimization. Increased through-
put can be considered as a side effect of performance optimization.^3 Optimizing queries
makes it possible for the server to execute more queries per second, because each one
requires less time to execute when the server is optimized. (The unit of throughput is
queries per time, which is the inverse of our definition of performance.)

So if the goal is to reduce response time, we need to understand why the server requires
a certain amount of time to respond to a query, and reduce or eliminate whatever
unnecessary work it’s doing to achieve the result. In other words, we need to measure
where the time goes. This leads to our second important principle of optimization: _you
cannot reliably optimize what you cannot measure_. Your first job is therefore to measure
where time is spent.

1. We don’t distinguish between queries and statements, DDL and DML, and so on. If you send a command
    to the server, no matter what it is, you just care about how quickly it executes. We tend to use “query”
    as a catch-all phrase for any command you send.
2. We’ll mostly avoid philosophical discussions about performance optimization, but we have two
    suggestions for further reading. There is a white paper called _Goal-Driven Performance Optimization_ on
    Percona’s website (http://www.percona.com), which is a compact quick-reference sheet. It is also very
    worthwhile to read Cary Millsap’s book _Optimizing Oracle Performance_ (O’Reilly). Cary’s performance
    optimization method, Method R, is the gold standard in the Oracle world.
3. Some people define performance in terms of throughput, which is okay, but it’s not the definition we use
    here. We think response time is more useful, although throughput is often easier to measure in
    benchmarks.

**70 | Chapter 3: Profiling Server Performance**


We’ve observed that many people, when trying to optimize something, spend the bulk
of their time changing things and very little time measuring. In contrast, we aim to
spend most of our time—perhaps upwards of 90%—measuring where the response
time is spent. If we don’t find the answer, we might not have measured correctly or
completely. When we gather complete and properly scoped measurements about server
activity, performance problems usually can’t hide, and the solution often becomes
trivially obvious. Measuring can be a challenge, however, and it can also be hard to
know what to do with the results once you have them—measuring where the time is
spent is not the same thing as understanding why the time is spent.

We mentioned proper scoping, but what does that mean? A properly scoped measure-
ment is one that measures only the activity you want to optimize. There are two com-
mon ways that you can capture something irrelevant:

- You can begin and end your measurements at the wrong time.
- You can measure things in aggregate instead of specifically targeting the activity
    itself.

For example, a common mistake is to observe a slow query, and then look at the whole
server’s behavior to try to find what’s wrong. If the query is slow, then it’s best to
measure the query, not the whole server. And it’s best to measure from the beginning
of the query to the end, not before or after.

The time required to execute a task is spent either executing, or waiting. The best way
to reduce the time required to execute is to identify and measure the subtasks, and then
do one or more of the following: eliminate subtasks completely, make them happen
less often, or make them happen more efficiently. Reducing waiting is a more complex
exercise, because waiting can be caused by “collateral damage” from other activities
on the system, and thus there can be interaction between the task and other tasks that
might be contending for access to resources such as the disk or CPU. And you might
need to use different techniques or tools, depending on whether the time is spent ex-
ecuting or waiting.

In the preceding paragraph we said that you need to identify and optimize subtasks.
But that’s an oversimplification. Infrequent or short subtasks might contribute so little
to overall response time that it’s not worth your time to optimize them. How do you
determine which tasks to target for optimization? This is why profiling was invented.

```
Introduction to Performance Optimization | 71
```

```
How Do You Know If Measurements Are Right?
If measurements are so important, then what if the measurements are wrong? In fact,
measurements are always wrong. The measurement of a quantity is not the same as the
quantity itself. The measurements might not be wrong enough to make a big difference,
but they’re wrong. So the question really should be, “How uncertain is the measure-
ment?” This is a topic that’s addressed in great detail in other books, so we won’t tackle
it here. Just be conscious that you’re working with measurements, not the actual
quantities they represent. As usual, the measurements can be presented in confusing
or ambiguous ways, which can lead to wrong conclusions, too.
```
#### Optimization Through Profiling

Once you have learned and practiced the response time–oriented method of perfor-
mance optimization, you’ll find yourself profiling systems over and over.

Profiling is the primary means of measuring and analyzing where time is consumed.
Profiling entails two steps: measuring tasks and the time elapsed, and aggregating and
sorting the results so that the important tasks bubble to the top.

Profiling tools all work in pretty much the same way. When a task begins, they start a
timer, and when it ends, they stop the timer and subtract the start time from the end
time to derive the response time. Most tools also record the task’s parent. The resulting
data can be used to construct call graphs, but more importantly for our purpose, similar
tasks can be grouped together and summed up. It can be helpful to do sophisticated
statistical analysis on the tasks that were grouped into one, but at a minimum, you need
to know how many tasks were grouped together, and the sum of their response times.
The _profile report_ accomplishes this. A profile report consists of a table of tasks, one
line per task. Each line shows a name, the number of times the task executed, the total
time consumed, the average time per execution, and what portion of the whole this
task consumed. The profile report should be sorted in order of total time consumed,
descending.

To make this clearer, let’s look at a real profile of an entire server’s workload, which
shows the types of queries that the server spends its time executing. This is a top-level
view of where the response time goes; we’ll show others later. The following is from
Percona Toolkit’s _pt-query-digest_ tool, which is the successor to Maatkit’s _mk-query-
digest_. We’ve simplified it slightly and included only the first few types of queries, to
remove distractions:

```
Rank Response time Calls R/Call Item
==== ================ ===== ====== =======
1 11256.3618 68.1% 78069 0.1442 SELECT InvitesNew
2 2029.4730 12.3% 14415 0.1408 SELECT StatusUpdate
3 1345.3445 8.1% 3520 0.3822 SHOW STATUS
```
**72 | Chapter 3: Profiling Server Performance**


We’ve shown only the first few lines in the profile, ranked in order of total response
time consumption, with the minimal set of columns that a profile ought to have. Each
row shows the response time as a total and as a percent of the overall total, the number
of times the query executed, the average response time per query, and an abstraction
of the query. This profile makes it clear how expensive each of these types of queries
is, relative to each other as well as to the whole. In this case, tasks are queries, which
is probably the most common way that you’ll profile MySQL.

We will actually discuss two kinds of profiling: _execution-time profiling_ and _wait
analysis_. Execution-time profiling shows which tasks consume the most time, whereas
wait analysis shows where tasks get stuck or blocked the most.

When tasks are slow because they’re consuming too many resources and are spending
most of their time executing, they won’t spend much time waiting, and wait analysis
will not be useful. The reverse is true, too: when tasks are waiting all the time and not
consuming any resources, measuring where they spend time executing won’t be very
helpful. If you’re not sure which kind of time consumption is the problem, you might
need to do both. We’ll show some examples of that later.

In practice, when execution-time profiling shows that a task is responsible for a lot of
elapsed time, you might be able to drill into it and find that some of the “execution
time” is spent waiting, at some lower level. For example, our simplified profile above
shows that a lot of time is consumed by a SELECT against the InvitesNew table, but at a
lower level, that time might be spent waiting for I/O to complete.

Before you can profile a system, you need to be able to measure it, and that often
requires _instrumentation_. An instrumented system has measurement points where data
is captured, and some way to make the data available for collection. Systems that are
well-instrumented are rather uncommon. Most systems are not built with a lot of in-
strumentation points, and those that are often provide only counts of activities, and no
way to measure how much time those activities took. MySQL is an example of this, at
least until version 5.5 when the first version of the Performance Schema introduced a
few time-based measurement points.^4 Versions 5.1 and earlier of MySQL had practi-
cally no time-based measurement points; most of the data you could get about the
server’s operation was in the form of SHOW STATUS counters, which simply count how
many times activities occur. That’s the main reason we ended up creating Percona
Server, which has offered detailed query-level instrumentation since version 5.0.

Fortunately, even though our ideal performance optimization technique works best
with great instrumentation, you can still make progress even with imperfectly instru-
mented systems. It’s often possible to measure the systems externally, or, failing that,
to make educated guesses based on knowledge of the system and the best information
available to you. However, when you do so, just be conscious that you’re operating on

4. The Performance Schema in MySQL 5.5 doesn’t provide query-level details; that is added in MySQL 5.6.

```
Introduction to Performance Optimization | 73
```

potentially flawed data, and your guesses are not guaranteed to be correct. This is a
risk that you usually take when you observe systems that aren’t perfectly transparent.

For example, in Percona Server 5.0, the slow query log can reveal a few of the most
important causes of poor performance, such as waiting for disk I/O or row-level locks.
If the log shows 9.6 seconds of disk I/O wait for a 10-second query, it’s not important
to find out where the remaining 4% of the response time went. The disk I/O is clearly
the most important problem.

#### Interpreting the Profile

The profile shows you the most important tasks first, but what it doesn’t show you can
be just as important. Refer to the example profile we showed earlier. Unfortunately,
there’s a lot that it conceals, because all it shows is ranks, sums, and averages. Here’s
what’s missing:

_Worthwhile queries_
The profile doesn’t automatically show you which queries are worth your time to
optimize. This brings us back to the meaning of optimization. If you read Cary
Millsap’s book, you’ll get a lot more on this topic, but we’ll repeat two salient
points. First, some tasks aren’t worth optimizing because they contribute such a
small portion of response time overall. Because of Amdahl’s Law, a query that
consumes only 5% of total response time can contribute only 5% to overall
speedup, no matter how much faster you make it. Second, if it costs you a thousand
dollars to optimize a task and the business ends up making no additional money
as a result, you just deoptimized the business by a thousand dollars. Thus, opti-
mization should halt when the cost of improvement outweighs the benefit.

_Outliers_
Tasks might need to be optimized even if they don’t sort to the top of the profile.
If an occasional task is very slow, it might be unacceptable to users, even though
it doesn’t happen often enough to constitute a significant portion of overall re-
sponse time.

_Unknown unknowns_^5
A good profiling tool will show you the “lost time,” if possible. Lost time is the
amount of wall-clock time not accounted in the tasks measured. For example, if
you measure the process’s overall CPU time as 10 seconds, but your profile of
subtasks adds up to 9.7 seconds, there are 300 milliseconds of lost time. This can
be an indication that you’re not measuring everything, or it could just be unavoid-
able due to rounding errors and the cost of measurement itself. You should pay
attention to this, if the tool shows it. You might be missing something important.
If the profile doesn’t show this, you should try to be conscious of its absence and

5. With apologies to Donald Rumsfeld. His comments were actually very insightful, even if they sounded
    funny.

**74 | Chapter 3: Profiling Server Performance**


```
include it in your mental (or real) notes about what information you’re missing.
Our example profile doesn’t show lost time; that’s just a limitation of the tool we
used.
```
_Buried details_
The profile doesn’t show anything about the distribution of the response times.
Averages are dangerous because they hide information from you, and the average
isn’t a good indication of the whole. Peter often likes to say that the average tem-
perature of patients in the hospital isn’t important.^6 What if item #1 in the profile
we showed earlier were really composed of two queries with one-second response
times, and 12,771 queries with response times in the tens of microseconds? There’s
no way to know from what we’re given. In order to make the best decisions about
where to concentrate your efforts, you need more information about the 12,773
queries that got packed into that single line in the profile. It’s especially helpful to
have more information on the response times, such as histograms, percentiles, the
standard deviation, and the index of dispersion.

Good tools can help you by automatically showing you some of these things. In fact,
_pt-query-digest_ includes many of these details in its profile, and in the detailed report
that follows the profile. We simplified so that we could focus the example on the im-
portant basics: sorting the most expensive tasks to the top. We’ll show examples of a
richer and more useful profile report later in this chapter.

Another very important thing that’s missing from our example profile is the ability to
analyze interactions at a higher layer in the stack. When we’re looking solely at queries
in the server, we don’t really have the ability to link together related queries and un-
derstand whether they were all part of the same user interaction. We have tunnel vision,
so to speak, and we can’t zoom out and profile at the level of transactions or page views.
There are some ways to solve this problem, such as tagging queries with special com-
ments indicating where they originated and then aggregating at that level. Or you can
add instrumentation and profiling capabilities at the application layer, which is the
subject of our next section.

### Profiling Your Application

You can profile pretty much anything that consumes time, and this includes your ap-
plication. In fact, profiling your application is generally easier than profiling your
database server, and much more rewarding. Although we’ve started by showing a pro-
file of a MySQL server’s queries for the purposes of illustration, it’s better to try to
measure and profile from the top down.^7 This lets you trace tasks as they flow through

6. Blimey! (It’s an inside joke. We can’t resist.)
7. We’ll show examples later where we have _a priori_ knowledge that the problem originates at a lower level,
    so we skip the top-down approach.

```
Profiling Your Application | 75
```

the system from the user to the servers and back. It’s often true that the database server
is to blame for performance problems, but it’s the application’s fault at least as often.
Bottlenecks can also be caused by any of the following:

- External resources, such as calls to web services or search engines
- Operations that require processing large amounts of data in the application, such
    as parsing big XML files
- Expensive operations in tight loops, such as abusing regular expressions
- Badly optimized algorithms, such as naïve search algorithms to find items in lists

Fortunately, it’s easy to figure out whether MySQL is the problem. You just need an
application profiling tool. (As a bonus, once you have it in place, it can help developers
write efficient code from the start.)

We recommend that you include profiling code in _every_ new project you start. It might
be hard to inject profiling code into an existing application, but it’s easy to include it
in new applications.

```
Will Profiling Slow Your Servers?
Yes, it will make your application slower. No, it will make your application much faster.
Wait, we can explain.
Profiling and routine monitoring add overhead. The important questions are how much
overhead they add and whether the extra work is worth the benefit.
Many people who design and build high-performance applications believe that you
should measure everything you can and just accept the cost of measurement as a part
of your application’s work. Oracle performance guru Tom Kyte was famously asked
how costly Oracle’s instrumentation is, and he replied that the instrumentation makes
it possible to improve performance by at least 10%. We agree with this philosophy,
and for most applications that wouldn’t otherwise receive detailed performance eval-
uations every day, we think the improvement is likely to be much more than 10%. Even
if you don’t agree, it’s a great idea to build in at least some lightweight profiling that
you can enable permanently. It’s no fun to hit a performance bottleneck you never saw
coming, just because you didn’t build your systems to capture day-to-day changes in
their performance. Likewise, when you find a problem, historical data is invaluable.
You can also use the profiling data to help you plan hardware purchases, allocate re-
sources, and predict load for peak times or seasons.
What do we mean by “lightweight” profiling? Timing all SQL queries, plus the total
script execution time, is certainly cheap. And you don’t have to do it for every page
view. If you have a decent amount of traffic, you can just profile a random sample by
enabling profiling in your application’s setup file:
```
**76 | Chapter 3: Profiling Server Performance**


```
<?php
$profiling_enabled = rand(0, 100) > 99;
?>
Profiling just 1% of your sessions should help you find the worst problems. It’s ex-
tremely helpful to do this in production, because you’ll find things that you’ll never see
elsewhere.
```
A few years ago, when we wrote the second edition of this book, good prefabricated
tools for profiling applications in production weren’t all that readily available for the
popular web programming languages and frameworks, so we showed you a code ex-
ample of baking your own in a simple but effective way. Today we’re glad to say that
great tools are available and all you have to do is open the box and start improving
performance.

First and foremost, we want to tout the benefits of a software-as-a-service product called
New Relic. We aren’t paid to praise it, and we normally don’t endorse specific com-
panies or products, but this is a great tool. If you can possibly use it, you should. Our
customers who use New Relic are able to solve their problems without our help much
more often, and they can sometimes use it to identify problems correctly even when
they can’t find the solution. New Relic plugs into your application, profiles it, and sends
the data back to a web-based dashboard that makes it easy to take a response time–
oriented approach to application performance. You end up doing the right thing
without having to think about it. And New Relic instruments a lot of the user experi-
ence, from the web browser to the application code to the database and other external
calls.

What’s great about tools like New Relic is that they let you instrument your code in
production, all the time—not just in development, and not just sometimes. This is an
important point because many profiling tools, or the instrumentation they need to
function, can be so expensive that people are afraid to run them in production. You
need to instrument in production because you’ll discover things about your system’s
performance that you won’t find in development or staging environments. If your
chosen tools are really too expensive to run all the time, try to at least run them on one
application server in the cluster, or instrument just a fraction of executions, as men-
tioned in the sidebar “Will Profiling Slow Your Servers?”.

#### Instrumenting PHP Applications

If you can’t use New Relic, there are other good options. For PHP in particular, there
are several tools that can help you do profile your application. One of them is _xhprof_
( _[http://pecl.php.net/package/xhprof](http://pecl.php.net/package/xhprof)_ ), which Facebook developed for its own use and
open sourced in 2009. It has a lot of advanced features, but for our purposes, the pri-
mary things to mention are that it’s easy to install and use, it’s lightweight and built for
scale so it can run in production all the time even on a very large installation, and it
generates a sensible profile of function calls sorted by time consumption. In addition

```
Profiling Your Application | 77
```

to _xhprof_ , there are low-level profiling tools such as _xdebug, Valgrind_ , and _cachegrind_
to help you inspect your code in various ways.^8 Some of these tools are not suitable for
production use because of their verbosity and high overhead, but can be great to use
in your development environment.

The other PHP profiling tool we’ll discuss is one that we wrote ourselves, based
partially on the code and principles we introduced in the second edition of this book.
It is called _instrumentation-for-php_ (IfP), and it’s hosted on Google Code at _[http://code](http://code)
.google.com/p/instrumentation-for-php/_. It doesn’t instrument PHP itself as thoroughly
as _xhprof_ does, but it instruments database calls more thoroughly, and thus it’s an
extremely valuable way to profile your application’s database usage when you don’t
have much access to or control over the database, which is often the case. IfP is a
singleton class that provides counters and timers, so it’s also easy to put into production
without requiring access to your PHP configuration, which again is the norm for a lot
of developers.

IfP doesn’t profile all of your PHP functions automatically—just the most important
ones. You have to start and stop custom counters manually when you identify things
that you want to profile, for example. But it times the whole page execution automat-
ically, and it makes it easy to instrument database and _memcached_ calls automatically,
so you don’t have to start and stop counters explicitly for those important items. This
means that you can profile three very valuable things in a jiffy: the application at the
level of requests (page views), database queries, and cache queries. It also exports
the counters and timers to the Apache environment, so you can get Apache to write
the results out to the log. This is an easy and very lightweight way to store the results
for later analysis. IfP doesn’t store any other data on your systems, so there’s no need
for additional system administrator involvement.

To use it, you simply call start_request() at the very start of the page execution. Ideally,
this should be the first thing your application does:

```
require_once('Instrumentation.php');
Instrumentation::get_instance()->start_request();
```
This registers a shutdown function, so you don’t have to do anything further at the end
of the execution.

IfP adds comments to your SQL queries automatically. This makes it possible to analyze
the application quite flexibly by looking at the database server’s query log, and it also
makes it easy to know what’s really going on when you look at SHOW PROCESSLIST and
see some abusive query running in MySQL. If you’re like most people, you’ll have a
hard time tracking down the source of a bad query, especially if it’s a query that was
cobbled together through string concatenation and so forth, so you can’t just search
for it in the source code. This solves that problem. It tells you which application host

8. Unlike PHP, many programming languages have some built-in support for profiling. For Ruby, use the
    - _r_ command-line option; for Perl you can use _perl -d:DProf_ , and so on.

**78 | Chapter 3: Profiling Server Performance**


sent the query, even if you’re using a proxy or a load balancer. It tells you which ap-
plication user is responsible, and you can find the page request, source code function,
and line number, as well as key-value pairs for all of the counters you’ve created. Here’s
an example:

```
-- File: index.php Line: 118 Function: fullCachePage request_id: ABC session_id: XYZ
SELECT * FROM ...
```
How you instrument the calls to MySQL depends on which interface you use to connect
to MySQL. If you’re using the object-oriented _mysqli_ interface, it’s a one-line change:
just replace the call to the _mysqli_ constructor with a call to the automatically instru-
mented _mysqli_x_ constructor instead. This constructor is a subclass provided by IfP,
with instrumentation and query rewrites baked in. If you’re not using the object-
oriented interface, or you’re using some other database access layer, you might need
to rewrite a little bit of code. Hopefully you don’t have database calls scattered
haphazardly throughout your code, but if you do, you can use an integrated develop-
ment environment (IDE) such as Eclipse to help you refactor it easily. Centralizing your
database access code is a very good practice, for many reasons.

Analyzing the results is easy. The _pt-query-digest_ tool from Percona Toolkit has func-
tionality to extract the embedded name-value pairs from the query comments, so you
can simply log the queries with the MySQL log file and process the log file. And you
can use _mod_log_config_ with Apache to set up custom logging with environment vari-
ables exported by IfP, along with the %D macro to capture request times in microseconds.

You can load the Apache log into a MySQL database with LOAD DATA INFILE and ex-
amine it with SQL queries easily. There is a PDF slideshow on the IfP website that gives
examples of how to do all of these things and more, with sample queries and command-
line arguments.

If you’re resisting adding instrumentation to your application, or if you feel too busy,
consider that it might be much easier than you think. The effort invested will pay you
back many times over in time savings and performance improvements. There’s no sub-
stitute for application instrumentation. Use New Relic, _xhprof_ , IfP, or any of a number
of other solutions for various application languages and environments; this is not a
wheel you need to reinvent.

```
Profiling Your Application | 79
```

```
The MySQL Enterprise Monitor’s Query Analyzer
One of the tools you should consider using is the MySQL Enterprise Monitor. It’s part
of a commercial MySQL support subscription from Oracle. It can capture queries to
your server, either from the application’s MySQL connection libraries or from a proxy
(although we’re not fans of using the proxy). It has a very nice graphical user interface
that shows a profile of queries on the server and makes it easy to zoom into a specific
time period, such as during a suspicious spike in a graph of status counters. You can
also see information such as the queries’ EXPLAIN plans, making it a very useful trou-
bleshooting and diagnosis tool.
```
### Profiling MySQL Queries

There are two broad approaches to profiling queries, which address two of the ques-
tions we mentioned in this chapter’s introduction. You can profile a whole server, in
terms of which queries contribute the most to its load. (If you’ve started at the top with
application-level profiling, you might already know which queries need attention.)
Then, once you’ve targeted specific queries for optimization, you can drill down to
profiling them individually, measuring which subtasks contribute the most to their
response times.

#### Profiling a Server’s Workload

The server-wide approach is worthwhile because it can help you to audit a server for
inefficient queries. Identifying and fixing these “bad” queries can help you improve the
application’s performance overall, as well as target specific trouble spots. You can re-
duce the overall load on the server, thus making all queries faster by reducing conten-
tion for shared resources (“collateral benefit”). Reducing load on the server can help
you delay or avoid upgrades or other more costly measures, and you can discover and
address poor user experiences, such as outliers.

MySQL is getting more instrumentation with each new release, and if the current trend
is a reliable indicator, it will soon have world-class support for measuring most impor-
tant aspects of its performance. But in terms of profiling queries and finding the most
expensive ones, we don’t really need all that sophistication. The tool we need the most
has been there for a long time. It’s the so-called _slow query log_.

**Capturing MySQL’s queries to a log**

In MySQL, the slow query log was originally meant to capture just “slow” queries, but
for profiling, we need it to log _all_ queries. And we need high-resolution response times,
not the one-second granularity that was available in MySQL 5.0 and earlier. Fortu-
nately, those old limitations are a thing of the past. In MySQL 5.1 and newer versions,
the slow query log is enhanced so that you can set the long_query_time server variable

**80 | Chapter 3: Profiling Server Performance**


to zero, capturing all queries, and the query response time is available with microsecond
resolution. If you are using Percona Server, this functionality is available in version 5.0,
and Percona Server adds a great deal more control over the log’s contents and capturing
queries.

The slow query log is the lowest-overhead, highest-fidelity way to measure query exe-
cution times in current versions of MySQL. If you’re worried about the additional
I/O it might cause, put your mind at ease. We benchmarked it, and on I/O-bound
workloads, the overhead is negligible. (It’s actually _more_ noticeable on CPU-bound
workloads.) A more valid concern is filling up your disk. Make sure that you have log
rotation set up for the slow query log, if you leave it on all the time. Or, just don’t enable
it all the time; leave it disabled, and turn it on only for a period of time to gather a
workload sample.

MySQL has another type of query log, called the “general log,” but it’s not much use
for analyzing and profiling a server. The queries are logged as they arrive at the server,
so the log has no information on response times or the query execution plan. MySQL
5.1 and later also support logging queries to tables, but that too is a nonstarter for
most purposes. The performance impact is huge, and although MySQL 5.1 prints query
times with microsecond precision in the slow query log, it reverts to one-second gran-
ularity for logging slow queries to a table. That’s not very helpful.

Percona Server logs significantly more details to the slow query log than MySQL does.
There is valuable information on the query execution plan, locking, I/O activity, and
much more. These additional bits of data were added slowly over time, as we faced
different optimization scenarios that demanded more details about how queries ac-
tually executed and where they spent their time. We also made it much easier to
administer. For example, we added the ability to control every connection’s long_
query_time threshold globally, so you can make them start or stop logging their quer-
ies when the application uses a connection pool or persistent connections and you can’t
reset their session-level variables. All in all, it is a lightweight and full-featured way to
profile a server and optimize its queries.

Sometimes you don’t want to log queries on the server, or you can’t for some reason,
such as not having access to the server. We encountered these same limitations, so we
developed two alternative techniques and programmed them both into Percona Tool-
kit’s _pt-query-digest_ tool. The first tactic is watching SHOW FULL PROCESSLIST repeatedly
with the _--processlist_ option, noting when queries first appear and when they disappear.
This is a sufficiently accurate method for some purposes, but it can’t capture all queries.
Very short-lived queries can sneak in and finish before the tool can observe them.

The second technique is capturing TCP network traffic and inspecting it, then decoding
the MySQL client/server protocol. You can use _tcpdump_ to save the traffic to disk, then
use _pt-query-digest_ with the _--type=tpcdump_ option to decode and analyze the queries.
This is a much higher-precision technique, and it can capture all queries. It even works
with advanced protocol features such as the binary protocol used to create and execute

```
Profiling MySQL Queries | 81
```

server-side prepared statements, and the compressed protocol. You can also use
MySQL Proxy with a logging script, but in practice we rarely do this.

**Analyzing the query log**

We suggest that at least every now and then you should use the slow query log to
capture all queries executing on your server, and analyze them. Log the queries for some
representative period of time, such as an hour during your peak traffic time. If your
workload is very homogeneous, a minute or less might even be enough to find bad
queries that need to be optimized.

Don’t just open up the log and start looking at it directly—it’s a waste of time and
money. Generate a profile first, and if you need to, then you can go look at specific
samples in the log. It’s best to work from a high-level view down to the low level, or
you could de-optimize the business, as mentioned earlier.

Generating a profile from the slow query log requires a good log analysis tool. We
suggest _pt-query-digest_ , which is arguably the most powerful tool available for MySQL
query log analysis. It supports a large variety of functionality, including the ability to
save query reports to a database and track changes in workload over time.

By default, you simply execute it and pass it the slow query log file as an argument, and
it just does the right thing. It prints out a profile of the queries in the log, and then
selects “important” classes of queries and prints out a detailed report on each one. The
report has dozens of little niceties to make your life easier. We continue to develop this
tool actively, so you should read the documentation for the most recent version to learn
about its current functionality.

We’ll give you a brief tour of the report _pt-query-digest_ prints out, beginning with the
profile. Here is an uncensored version of the profile we showed earlier in this chapter:

```
# Profile
# Rank Query ID Response time Calls R/Call V/M Item
# ==== ================== ================ ===== ====== ===== =======
# 1 0xBFCF8E3F293F6466 11256.3618 68.1% 78069 0.1442 0.21 SELECT InvitesNew?
# 2 0x620B8CAB2B1C76EC 2029.4730 12.3% 14415 0.1408 0.21 SELECT StatusUpdate?
# 3 0xB90978440CC11CC7 1345.3445 8.1% 3520 0.3822 0.00 SHOW STATUS
# 4 0xCB73D6B5B031B4CF 1341.6432 8.1% 3509 0.3823 0.00 SHOW STATUS
# MISC 0xMISC 560.7556 3.4% 23930 0.0234 0.0 <17 ITEMS>
```
There’s a little more detail here than we saw previously. First, each query has an ID,
which is a hash of its “fingerprint.” A fingerprint is the normalized, canonical version
of the query with literal values removed, whitespace collapsed, and everything lower-
cased (notice that queries 3 and 4 appear to be the same, but they have different fin-
gerprints). The tool also merges tables with similar names into a canonical form. The
question mark at the end of the InvitesNew table name signifies that there is a shard
identifier appended to the table name, and the tool has removed that so that queries
against tables with a similar purpose are aggregated together. This report is from a
heavily sharded Facebook application.

**82 | Chapter 3: Profiling Server Performance**


Another bit of extra detail here is the variance-to-mean ratio, in the V/M column. This
is also known as the _index of dispersion_. Queries with a higher index of dispersion have
a more variable execution-time profile, and highly variable queries are generally good
candidates for optimization. If you specify the _--explain_ option to _pt-query-digest_ , it will
also add a column with a short representation of the query’s EXPLAIN plan—sort of a
“geek code” for the query. This, in combination with the V/M column, makes it a snap
to see which queries are bad and potentially easy to optimize.

Finally, there’s an additional line at the bottom, showing the presence of 17 other types
of queries that the tool didn’t consider important enough to report individually,
and a summary of the statistics for all of them. You can use options such as _--limit_ and
_--outliers_ to make the tool show more details instead of collapsing unimportant queries
into this final line. By default, the tool prints out queries that are either in the top 10
time consumers overall, or whose execution time was over a one-second threshold too
many times. Both of these limits are configurable.

Following the profile, the tool prints out a detailed report on each type of query. You
can match the query reports to the profile by looking for the query ID or the rank.
Here’s the report for the #1 ranked query, the “worst” one:

```
# Query 1: 24.28 QPS, 3.50x concurrency, ID 0xBFCF8E3F293F6466 at byte 5590079
# This item is included in the report because it matches --limit.
# Scores: V/M = 0.21
# Query_time sparkline: | _^_.^_ |
# Time range: 2008-09-13 21:51:55 to 22:45:30
# Attribute pct total min max avg 95% stddev median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count 63 78069
# Exec time 68 11256s 37us 1s 144ms 501ms 175ms 68ms
# Lock time 85 134s 0 650ms 2ms 176us 20ms 57us
# Rows sent 8 70.18k 0 1 0.92 0.99 0.27 0.99
# Rows examine 8 70.84k 0 3 0.93 0.99 0.28 0.99
# Query size 84 10.43M 135 141 140.13 136.99 0.10 136.99
# String:
# Databases production
# Hosts
# Users fbappuser
# Query_time distribution
# 1us
# 10us #
# 100us ####################################################
# 1ms ###
# 10ms ################
# 100ms ################################################################
# 1s #
# 10s+
# Tables
# SHOW TABLE STATUS FROM `production ` LIKE'InvitesNew82'\G
# SHOW CREATE TABLE `production `.`InvitesNew82'\G
# EXPLAIN /*!50100 PARTITIONS*/
SELECT InviteId, InviterIdentifier FROM InvitesNew82 WHERE (InviteSetId = 87041469)
AND (InviteeIdentifier = 1138714082) LIMIT 1\G
```
```
Profiling MySQL Queries | 83
```

The report contains a variety of metadata at the top, including how often the query
executes, its average concurrency, and the byte offset where the worst-performing in-
stance of the query was found in the log file. There is a tabular printout of the numeric
metadata, including statistics such as the standard deviation.^9

This is followed by a histogram of the response times. Interestingly, you can see that
this query has a double-peak histogram, under Query_time distribution. It usually
executes in the hundreds of milliseconds, but there’s also a significant spike of queries
that execute about three orders of magnitude faster. If this log were from Percona
Server, we’d have a richer set of attributes in the query log, so we’d be able to slice and
dice the queries to determine why that happens. Perhaps those are queries against
specific values that are disproportionately common, so a different index is used, or
perhaps they’re query cache hits, for example. This sort of double-peak histogram
shape is not unusual in real systems, especially for simple queries, which will usually
have only a few alternative execution paths.

Finally, the report detail section ends with little helper snippets to make it easy for you
to copy and paste commands into a terminal and examine the schema and status of the
tables mentioned, and an EXPLAIN-ready sample query. The sample contains all of the
literals and isn’t “fingerprinted,” so it’s a real query. It’s actually the instance of this
query that had the worst execution time in our example.

After you choose the queries you want to optimize, you can use this report to examine
the query execution very quickly. We use this tool constantly, and we’ve spent a lot of
time tweaking it to be as efficient and helpful as possible. We definitely recommend
that you get comfortable with it. MySQL might gain more sophisticated built-in in-
strumentation and profiling in the future, but at the time of writing, logging queries
with the slow query log or _tcpdump_ and running the resulting log through _pt-query-
digest_ is about as good as you can get.

#### Profiling a Single Query

After you’ve identified a single query to optimize, you can drill into it and determine
why it takes as much time as it does, and how to optimize it. The actual techniques for
optimizing queries are covered in later chapters in this book, along with the background
necessary to support those techniques. Our purpose here is simply to show you how
to measure what the query does and how long each part of that takes. Knowing this
helps you decide which optimization techniques to use.

Unfortunately, most of the instrumentation in MySQL isn’t very helpful for profiling
queries. This is changing, but at the time of writing, most production servers don’t have
the newest profiling features. So for practical purposes, we’re pretty much limited to

9. We’re keeping it simple here for clarity, but Percona Server’s query log will produce a much more detailed
    report, which could help you understand why the query is apparently spending 144 ms to examine a
    single row—that’s a lot!

**84 | Chapter 3: Profiling Server Performance**


SHOW STATUS, SHOW PROFILE, and examining individual entries in the slow query log (if
you have Percona Server—standard MySQL doesn’t have any additional information
in the log). We’ll demonstrate all three techniques for a single query and show you
what you can learn about the query execution from each.

**Using SHOW PROFILE**

The SHOW PROFILE command is a community contribution from Jeremy Cole that’s in-
cluded in MySQL 5.1 and newer, and some versions of MySQL 5.0. It is the only real
query profiling tool available in a GA release of MySQL at the time of writing. It is
disabled by default, but can be enabled for the duration of a session (connection) simply
by setting a server variable:

```
mysql> SET profiling = 1;
```
After this, whenever you issue a statement to the server, it will measure the elapsed
time and a few other types of data whenever the query changes from one execution
state to another. The feature actually has quite a bit of functionality, and was designed
to have more, but it will probably be replaced or superseded by the Performance Schema
in a future release. Regardless, the most useful functionality of this feature is to generate
a profile of the work the server did during statement execution.

Every time you issue a query to the server, it records the profiling information in a
temporary table and assigns the statement an integer identifier, starting with 1. Here’s
an example of profiling a view included with the Sakila sample database:^10

```
mysql> SELECT * FROM sakila.nicer_but_slower_film_list;
[query results omitted]
997 rows in set (0.17 sec)
```
The query returned 997 rows in about a sixth of a second. Let’s see what SHOW PRO
FILES (note the plural) knows about this query:

```
mysql> SHOW PROFILES;
+----------+------------+-------------------------------------------------+
| Query_ID | Duration | Query |
+----------+------------+-------------------------------------------------+
| 1 | 0.16767900 | SELECT * FROM sakila.nicer_but_slower_film_list |
+----------+------------+-------------------------------------------------+
```
The first thing you’ll notice is that it shows the query’s response time with higher
precision, which is nice. Two decimal places of precision, as shown in the MySQL
client, often isn’t enough when you’re working on fast queries. Now let’s look at the
profile for this query:

10. The view is too lengthy to show here, but the Sakila database is available for download from MySQL’s
    website.

```
Profiling MySQL Queries | 85
```

```
mysql> SHOW PROFILE FOR QUERY 1;
+----------------------+----------+
| Status | Duration |
+----------------------+----------+
| starting | 0.000082 |
| Opening tables | 0.000459 |
| System lock | 0.000010 |
| Table lock | 0.000020 |
| checking permissions | 0.000005 |
| checking permissions | 0.000004 |
| checking permissions | 0.000003 |
| checking permissions | 0.000004 |
| checking permissions | 0.000560 |
| optimizing | 0.000054 |
| statistics | 0.000174 |
| preparing | 0.000059 |
| Creating tmp table | 0.000463 |
| executing | 0.000006 |
| Copying to tmp table | 0.090623 |
| Sorting result | 0.011555 |
| Sending data | 0.045931 |
| removing tmp table | 0.004782 |
| Sending data | 0.000011 |
| init | 0.000022 |
| optimizing | 0.000005 |
| statistics | 0.000013 |
| preparing | 0.000008 |
| executing | 0.000004 |
| Sending data | 0.010832 |
| end | 0.000008 |
| query end | 0.000003 |
| freeing items | 0.000017 |
| removing tmp table | 0.000010 |
| freeing items | 0.000042 |
| removing tmp table | 0.001098 |
| closing tables | 0.000013 |
| logging slow query | 0.000003 |
| logging slow query | 0.000789 |
| cleaning up | 0.000007 |
+----------------------+----------+
```
The profile allows you to follow through every step of the query’s execution and see
how long it took. You’ll notice that it’s a bit hard to scan this output and see where
most of the time was spent. It is sorted in chronological order, but we don’t really care
about the order in which the steps happened—we just care how much time they took,
so we know what was costly. Unfortunately, you can’t sort the output of the command
with an ORDER BY. Let’s switch from using the SHOW PROFILE command to querying the
corresponding INFORMATION_SCHEMA table, and format to look like the profiles we’re used
to seeing:

```
mysql> SET @query_id = 1;
Query OK, 0 rows affected (0.00 sec)
```
```
mysql> SELECT STATE, SUM(DURATION) AS Total_R,
```
**86 | Chapter 3: Profiling Server Performance**


```
-> ROUND(
-> 100 * SUM(DURATION) /
-> (SELECT SUM(DURATION)
-> FROM INFORMATION_SCHEMA.PROFILING
-> WHERE QUERY_ID = @query_id
-> ), 2) AS Pct_R,
-> COUNT(*) AS Calls,
-> SUM(DURATION) / COUNT(*) AS "R/Call"
-> FROM INFORMATION_SCHEMA.PROFILING
-> WHERE QUERY_ID = @query_id
-> GROUP BY STATE
-> ORDER BY Total_R DESC;
+----------------------+----------+-------+-------+--------------+
| STATE | Total_R | Pct_R | Calls | R/Call |
+----------------------+----------+-------+-------+--------------+
| Copying to tmp table | 0.090623 | 54.05 | 1 | 0.0906230000 |
| Sending data | 0.056774 | 33.86 | 3 | 0.0189246667 |
| Sorting result | 0.011555 | 6.89 | 1 | 0.0115550000 |
| removing tmp table | 0.005890 | 3.51 | 3 | 0.0019633333 |
| logging slow query | 0.000792 | 0.47 | 2 | 0.0003960000 |
| checking permissions | 0.000576 | 0.34 | 5 | 0.0001152000 |
| Creating tmp table | 0.000463 | 0.28 | 1 | 0.0004630000 |
| Opening tables | 0.000459 | 0.27 | 1 | 0.0004590000 |
| statistics | 0.000187 | 0.11 | 2 | 0.0000935000 |
| starting | 0.000082 | 0.05 | 1 | 0.0000820000 |
| preparing | 0.000067 | 0.04 | 2 | 0.0000335000 |
| freeing items | 0.000059 | 0.04 | 2 | 0.0000295000 |
| optimizing | 0.000059 | 0.04 | 2 | 0.0000295000 |
| init | 0.000022 | 0.01 | 1 | 0.0000220000 |
| Table lock | 0.000020 | 0.01 | 1 | 0.0000200000 |
| closing tables | 0.000013 | 0.01 | 1 | 0.0000130000 |
| System lock | 0.000010 | 0.01 | 1 | 0.0000100000 |
| executing | 0.000010 | 0.01 | 2 | 0.0000050000 |
| end | 0.000008 | 0.00 | 1 | 0.0000080000 |
| cleaning up | 0.000007 | 0.00 | 1 | 0.0000070000 |
| query end | 0.000003 | 0.00 | 1 | 0.0000030000 |
+----------------------+----------+-------+-------+--------------+
```
Much better! Now we can see that the reason this query took so long was that it spent
over half its time copying data into a temporary table. We might need to look into
rewriting this query so it doesn’t use a temporary table, or perhaps do it more efficiently.
The next biggest time consumer, “Sending data,” is really kind of a catch-all state that
could represent any number of different server activities, including searching for match-
ing rows in a join and so on. It’s hard to say whether we’ll be able to shave any time off
this. Notice that “Sorting result” takes up a very small portion of the time, not enough
to be worth optimizing. This is rather typical, which is why we encourage people not
to spend time on “tuning the sort buffers” and similar activities.

As usual, although the profile helps us identify what types of activity contribute the
most to the elapsed time, it doesn’t tell us _why_. To find out why it took so much time
to copy data into the temporary table, we’d have to drill down into that state and
produce a profile of the subtasks it executed.

```
Profiling MySQL Queries | 87
```

**Using SHOW STATUS**

MySQL’s SHOW STATUS command returns a variety of counters. There is a server-wide
global scope for the counters, as well as a session scope that is specific to your own
connection. The Queries counter, for example, starts at zero in your session and in-
creases every time you issue a query. If you execute SHOW GLOBAL STATUS (note the ad-
dition of the GLOBAL keyword), you’ll see a server-wide count of queries the server has
issued since it was started. The scope of each counter varies—counters that don’t have
a session-level scope still appear in SHOW STATUS, masquerading as session counters—
and this can be confusing. It’s something to keep in mind as you use this command.
As we discussed earlier, gathering properly scoped measurements is key. If you’re trying
to optimize something that you can observe occurring in your specific connection to
the server, measurements that are being polluted by server-wide activity are not helpful.
The MySQL manual has a great reference to all of the variables and whether they have
session or global scope.

SHOW STATUS can be a helpful tool, but it isn’t really profiling.^11 Most of the results from
SHOW STATUS are just counters. They tell you how often various activities took place,
such as reads from an index, but they tell you nothing about how much time was
consumed. There is only one counter in SHOW STATUS that shows time consumed by an
operation (Innodb_row_lock_time), and it has only global scope, so you can’t use it to
examine only the work you’ve done in your session.

Still, although SHOW STATUS doesn’t provide timings, it can be helpful to look at it after
you execute a query and examine the values for a few of the counters. You can form a
guess about which types of expensive operations took place and how likely they were
to contribute to the query time. The most important counters are the handler counters
and the temporary file and table counters. We explain these in more detail in Appen-
dix B. Here’s an example of resetting the session status counters to zero, selecting from
the same view we used in the previous section, and then looking at the counters:

```
mysql> FLUSH STATUS;
mysql> SELECT * FROM sakila.nicer_but_slower_film_list;
[query results omitted]
mysql> SHOW STATUS WHERE Variable_name LIKE 'Handler%'
OR Variable_name LIKE 'Created%';
+----------------------------+-------+
| Variable_name | Value |
+----------------------------+-------+
| Created_tmp_disk_tables | 2 |
| Created_tmp_files | 0 |
| Created_tmp_tables | 3 |
| Handler_commit | 1 |
| Handler_delete | 0 |
| Handler_discover | 0 |
| Handler_prepare | 0 |
| Handler_read_first | 1 |
```
11. If you own the second edition of this book, you’ll notice that we’re doing an about-face on this point.

**88 | Chapter 3: Profiling Server Performance**


```
| Handler_read_key | 7483 |
| Handler_read_next | 6462 |
| Handler_read_prev | 0 |
| Handler_read_rnd | 5462 |
| Handler_read_rnd_next | 6478 |
| Handler_rollback | 0 |
| Handler_savepoint | 0 |
| Handler_savepoint_rollback | 0 |
| Handler_update | 0 |
| Handler_write | 6459 |
+----------------------------+-------+
```
It looks like the query used three temporary tables—two of them on disk—and did a
lot of unindexed reads (Handler_read_rnd_next). If we didn’t know anything about the
view we just accessed, we might guess that the query is perhaps doing a join without
an index, possibly because of a subquery that created temporary tables and then made
it the right-hand input to a join. Temporary tables created to hold the results of sub-
queries don’t have indexes, so this seems plausible.

When you use this technique, be aware that SHOW STATUS itself creates a temporary table,
and accesses this table with handler operations, so the numbers you see in the output
are actually impacted by SHOW STATUS. This varies between server versions. Given what
we already know about the query’s execution from SHOW PROFILES, it looks like the
count of temporary tables might be overstated by 2.

It’s worth noting that you can probably discover most of the same information by
looking at an EXPLAIN plan for this query. But EXPLAIN is an estimate of what the server
thinks it will do, and looking at the status counters is a measurement of what it actually
did. EXPLAIN won’t tell you whether a temporary table was created on disk, for example,
which is slower than in memory. There’s more on EXPLAIN in Appendix D.

**Using the slow query log**

What does the enhanced slow query log in Percona Server reveal about this query?
Here’s what it captured from the very same execution of the query that we demon-
strated in the section on SHOW PROFILE:

```
# Time: 110905 17:03:18
# User@Host: root[root] @ localhost [127.0.0.1]
# Thread_id: 7 Schema: sakila Last_errno: 0 Killed: 0
# Query_time: 0.166872 Lock_time: 0.000552 Rows_sent: 997 Rows_examined: 24861
Rows_affected: 0 Rows_read: 997
# Bytes_sent: 216528 Tmp_tables: 3 Tmp_disk_tables: 2 Tmp_table_sizes: 11627188
# InnoDB_trx_id: 191E
# QC_Hit: No Full_scan: Yes Full_join: No Tmp_table: Yes Tmp_table_on_disk: Yes
# Filesort: Yes Filesort_on_disk: No Merge_passes: 0
# InnoDB_IO_r_ops: 0 InnoDB_IO_r_bytes: 0 InnoDB_IO_r_wait: 0.000000
# InnoDB_rec_lock_wait: 0.000000 InnoDB_queue_wait: 0.000000
# InnoDB_pages_distinct: 20
# PROFILE_VALUES ... Copying to tmp table: 0.090623... [omitted]
SET timestamp=1315256598;
SELECT * FROM sakila.nicer_but_slower_film_list;
```
```
Profiling MySQL Queries | 89
```

It looks like the query did create three temp tables after all, which was somewhat hidden
from view in SHOW PROFILE (perhaps due to a subtlety in the way the server executed
the query). Two of the temp tables were on disk. And we’re shortening the output here
for readability, but toward the end, the SHOW PROFILE data for this query is actually
written to the log, so you can even log that level of detail in Percona Server.

As you can see, this highly verbose slow query log entry contains just about everything
you can see in SHOW PROFILE and SHOW STATUS, and then some. This makes the log a very
useful place to look for more detail when you find a “bad” query with _pt-query-digest_.
When you’re looking at a report from _pt-query-digest_ , you’ll see a header line such as
the following:

```
# Query 1: 0 QPS, 0x concurrency, ID 0xEE758C5E0D7EADEE at byte 3214 _____
```
You can use the byte offset from the header to zoom right into that section of the log,
like this:

```
tail -c +3214 /path/to/query.log | head -n100
```
And presto, you can look at all the details. By the way, _pt-query-digest_ understands all
the added name-value pairs in the Percona Server slow query log format, and auto-
matically prints out a much more detailed report as a result.

**Using the Performance Schema**

At the time of writing, the Performance Schema tables introduced in MySQL 5.5 don’t
support query-level profiling. The Performance Schema is rather new and in rapid de-
velopment, with much more functionality in the works for future releases. However,
even MySQL 5.5’s initial functionality can reveal interesting information. For example,
here’s a query that shows the top causes of waiting in the system:

```
mysql> SELECT event_name, count_star, sum_timer_wait
-> FROM events_waits_summary_global_by_event_name
-> ORDER BY sum_timer_wait DESC LIMIT 5;
+----------------------------------------+------------+------------------+
| event_name | count_star | sum_timer_wait |
+----------------------------------------+------------+------------------+
| innodb_log_file | 205438 | 2552133070220355 |
| Query_cache::COND_cache_status_changed | 8405302 | 2259497326493034 |
| Query_cache::structure_guard_mutex | 55769435 | 361568224932147 |
| innodb_data_file | 62423 | 347302500600411 |
| dict_table_stats | 15330162 | 53005067680923 |
+----------------------------------------+------------+------------------+
```
There are a few of things that limit the Performance Schema’s use as a general-purpose
profiling tool at present. First, it doesn’t yet provide the level of detail on query exe-
cution stages and timing that we’ve been showing with existing tools. Second, it hasn’t
been “in the wild” for all that long, and the implementation has more overhead at
present than many conservative users are comfortable with. (There is reason to believe
this will be fixed soon.)

**90 | Chapter 3: Profiling Server Performance**


Finally, it’s sometimes too complex and low-level to be accessible to most users in its
raw form. The features implemented so far are mostly targeted toward the things we
need to measure when modifying MySQL source code to improve the server’s perfor-
mance. This includes things like waits and mutexes. Some of the features in MySQL
5.5 are valuable to power users as opposed to server developers, but those still need
some frontend tool development to make it convenient to use them and interpret the
results. Right now the state of the art is writing complex queries against a large variety
of metadata tables with lots and lots of columns. It’s a pretty intimidating amount of
instrumentation to navigate and understand.

When the Performance Schema gets more functionality in MySQL 5.6 and beyond, and
there are nice tools to use it, it’s going to be awesome. And it’s really nice that Oracle
is implementing it as tables accessible through SQL so that users can consume the data
in whatever manner is most useful to them. For the time being, though, it’s not quite
a workable replacement for the slow query log or other tools that can help us imme-
diately see how to improve server and query performance.

#### Using the Profile for Optimization

So you’ve got a profile of your server or your query—what do you do with it? A good
profile usually makes the problem obvious, but the _solution_ might not be (although it
often is). At this point, especially when optimizing queries, you need to rely on a lot of
knowledge about the server and how it executes queries. The profile, or as much of one
as you can gather, points you in the right direction and gives you a basis for using further
tools, such as EXPLAIN, to apply your knowledge and measure the results. That’s a topic
for future chapters, but at least you have the right starting point.

In general, although a profile with complete measurements ought to make determining
the problem trivial, we can’t always measure perfectly because the systems we’re trying
to measure don’t support it. In the example we’ve been looking at, we suspect that
temporary tables and unindexed reads are contributing most of the response time to
the query, but we can’t prove it. Sometimes problems are hard to solve because you
might not have measured everything you need, or your measurements might be badly
scoped. You might be measuring server-wide activity instead of looking specifically at
what you’re trying to optimize, for example, or you might be looking at measurements
that count from a point in time before your query started to execute, rather than the
instant the query began.

There’s another possibility. Suppose you analyze your slow query log and find a simple
query that took an unreasonably long time to execute a handful of times, although it
ran quickly thousands of other times. You run the query again, and it is lightning fast,
as it should be. You use EXPLAIN, and it is using an index correctly. You even try similar
queries with different values in the WHERE clause to ensure you aren’t just seeing cache
hits, and they run quickly. Nothing seems to be wrong with this query. What gives?

```
Profiling MySQL Queries | 91
```

If you have only the standard MySQL slow query log, with no execution plan or detailed
timing information, you are limited to the knowledge that the query ran badly at the
point that it was logged—you can’t see why that was. Perhaps something else was
consuming resources on the system, such as a backup, or perhaps some kind of locking
or contention blocked the query’s progress. Intermittent problems are a special case
that we’ll cover in the next section.

### Diagnosing Intermittent Problems

Intermittent problems such as occasional server stalls or slow queries can be frustrating
to diagnose, and the most egregious wastes of time we’ve seen have been results of
phantom problems that happen only when you’re not looking, or aren’t reliably re-
producible. We’ve seen people spend literally months fighting with such problems. In
the process, some of them reverted to a trial-and-error troubleshooting approach, and
sometimes made things dramatically worse by trying to change things such as server
settings at random, hoping to stumble upon something that would help.

Try to avoid trial and error if you can. Trial-and-error troubleshooting is risky, because
the results can be bad, and it can be frustrating and inefficient. If you can’t figure out
what the problem is, you might not be measuring correctly, you might be measuring
in the wrong place, or you might not know the necessary tools to use. (Or the tools
might not exist—we’ve developed a number of tools specifically to address the lack
of transparency in various system components, from the operating system to MySQL
itself.)

To illustrate the importance of trying to avoid trial and error, here are some sample
resolutions we’ve found to some of the intermittent database performance problems
we’ve been called in to solve:

- The application was executing _curl_ to fetch exchange rate quotes from an external
    service, which was running very slowly at times.
- Important cache entries were expiring from _memcached_ , causing the application
    to flood MySQL with requests to regenerate the cached items.
- DNS lookups were timing out randomly.
- The query cache was freezing MySQL periodically due to mutex contention or
    inefficient internal algorithms for deleting cached queries.
- InnoDB scalability limitations were causing query plan optimization to take too
    long when concurrency was over some threshold.

As you can see, some of these problems were in the database, and some of them weren’t.
Only by beginning at the place where the misbehavior could be observed and working
through the resources it used, measuring as completely as possible, can you avoid
hunting in the wrong place for problems that don’t exist there.

**92 | Chapter 3: Profiling Server Performance**


We’ll stop lecturing you now, and explain the approach and tools we use for solving
intermittent problems.

#### Single-Query Versus Server-Wide Problems

Do you have any evidence of the problem? If so, try to determine whether the problem
is with a single isolated query, or if it’s server-wide. This is important to point you in
the right direction. If everything on the server is suffering, and then everything is okay
again, then any given query that’s slow isn’t likely to be the problem. Most of the slow
queries are likely to be victims of some other problem instead. On the other hand, if
the server is running nicely as a whole and a single query is slow for some reason, you
have to look more closely at that query.

Server-wide problems are fairly common. As more powerful hardware has become
available in the last several years, with 16-core and bigger servers becoming the norm,
MySQL’s scalability limitations on SMP systems have become more noticeable. Most
of these problems are in older versions, which are unfortunately still widely used in
production. MySQL still has some scalability problems even in newer versions, but
they are much less serious, and much less frequently encountered, because they’re edge
cases. This is good news and bad news: good because you’re much less likely to hit
them, and bad because they require more knowledge of MySQL internals to diagnose.
It also means that a lot of problems can be solved by simply upgrading MySQL.^12

How do you determine whether the problem is server-wide or isolated to a single query?
If the problem occurs repeatedly enough that you can observe it in action, or run a
script overnight and look at the results the next day, there are three easy techniques
that can make it obvious in most cases. We’ll cover those next.

**Using SHOW GLOBAL STATUS**

The essence of this technique is to capture samples of SHOW GLOBAL STATUS at high
frequency, such as once per second, and when the problem manifests, look for “spikes”
or “notches” in counters such as Threads_running, Threads_connected, Questions, and
Queries. This is a simple method that anyone can use (no special privileges are required)
without impacting the server, so it’s a great way to learn more about the nature of the
problem without a big investment of time. Here’s a sample command and output:

```
$ mysqladmin ext -i1 | awk '
/Queries/{q=$4-qp;qp=$4}
/Threads_connected/{tc=$4}
/Threads_running/{printf "%5d %5d %5d\n", q, tc, $4}'
2147483647 136 7
798 136 7
767 134 9
828 134 7
```
12. Again, don’t do that without a good reason to believe that it’s the solution.

```
Diagnosing Intermittent Problems | 93
```

```
683 134 7
784 135 7
614 134 7
108 134 24
187 134 31
179 134 28
1179 134 7
1151 134 7
1240 135 7
1000 135 7
```
The command captures samples of SHOW GLOBAL STATUS every second and pipes those
into an _awk_ script that prints out queries per second, Threads_connected, and
Threads_running (number of queries currently executing). These three tend to be very
sensitive to server-wide stalls. What usually happens is that, depending on the nature
of the problem and how the application connects to MySQL, queries per second will
drop and at least one of the other two will spike. Here the application is probably using
a connection pool, so there’s no spike of connected threads, but there’s a clear bump
in in-progress queries at the same time that the queries per second value drops to a
fraction of its normal level.

What could explain this behavior? It’s risky to guess, but in practice we’ve seen two
common cases. One is some kind of internal bottleneck in the server, causing new
queries to begin executing but to pile up against some lock that the older queries are
waiting to acquire. This type of lock usually puts back-pressure on the application
servers and causes some queueing there, too. The other common case we’ve seen is a
spike of heavy queries such as those that can happen with a badly handled _memc-
ached_ expiration.

At one line per second, you can easily let this run for hours or days and make a quick
plot to see if there are any areas with aberrations. If a problem is truly intermittent, you
can let it run as long as needed and then refer back to the output when you notice the
problem. In most cases this output will show the problem clearly.

**Using SHOW PROCESSLIST**

With this method, you capture samples of SHOW PROCESSLIST and look for lots of threads
that are in unusual states or have some other unusual characteristic. For example, it’s
rather rare for queries to stay in the “statistics” state for very long, because this is the
phase of query optimization where the server determines the best join order—normally
very fast. Likewise, it’s rare to see a lot of threads reporting the user as “Unauthenticated
user,” because this is a state that happens in the middle of the connection handshake
when the client specifies the user it’s trying to use to log in.

Vertical output with the \G terminator is very helpful for working with SHOW PROCESS
LIST, because it puts each column of each row of the output onto its own line, making
it easy to do a little _sort|uniq|sort_ incantation that helps you view the count of unique
values in any desired column easily:

**94 | Chapter 3: Profiling Server Performance**


```
$ mysql -e 'SHOW PROCESSLIST\G' | grep State: | sort | uniq -c | sort -rn
744 State:
67 State: Sending data
36 State: freeing items
8 State: NULL
6 State: end
4 State: Updating
4 State: cleaning up
2 State: update
1 State: Sorting result
1 State: logging slow query
```
Just change the _grep_ pattern if you want to examine a different column. The State
column is a good one for a lot of cases. Here we can see that there are an awful lot of
threads in states that are part of the end of query execution: “freeing items,” “end,”
“cleaning up,” and “logging slow query.” In fact, in many samples on the server from
which this output came, this pattern or a similar one occurred. The most characteristic
and reliable indicator of a problem was a high number of queries in the “freeing items”
state.

You don’t have to use command-line techniques to find problems like this. You can
query the PROCESSLIST table in the INFORMATION_SCHEMA if your server is new enough, or
use _innotop_ with a fast refresh rate and watch the screen for an unusual buildup of
queries. The example we just showed was of a server with InnoDB internal contention
and flushing problems, but it can be far more mundane than that. The classic example
would be a lot of queries in the “Locked” state. That’s the unlovable trademark of
MyISAM with its table-level locking, which quickly escalates into server-wide pileups
when there’s enough write activity on the tables.

**Using query logging**

To find problems in the query log, turn on the slow query log and set long_query_time
to 0 globally, and make sure that all of the connections see the new setting. You might
have to recycle connections so they pick up the new global value, or use Percona Server’s
feature to force it to take effect instantly without disrupting existing connections.

If you can’t enable the slow query log to capture all queries for some reason, use
_tcpdump_ and _pt-query-digest_ to emulate it. Look for periods in the log where the
throughput drops suddenly. Queries are sent to the slow query log at completion time,
so pileups typically result in a sudden drop of completions, until the culprit finishes
and releases the resource that’s blocking the other queries. The other queries will then
complete. What’s helpful about this characteristic behavior is that it lets you blame the
first query that completes after a drop in throughput. (Sometimes it’s not quite the first
query; other queries might be running unaffected while some are blocked, so this isn’t
completely reliable.)

Again, good tools can help with this. You can’t be looking through hundreds of giga-
bytes of queries by hand. Here’s a one-liner that relies on MySQL’s pattern of writing
the current time to the log when the clock advances one second:

```
Diagnosing Intermittent Problems | 95
```

```
$ awk '/^# Time:/{print $3, $4, c;c=0}/^# User/{c++}' slow-query.log
080913 21:52:17 51
080913 21:52:18 29
080913 21:52:19 34
080913 21:52:20 33
080913 21:52:21 38
080913 21:52:22 15
080913 21:52:23 47
080913 21:52:24 96
080913 21:52:25 6
080913 21:52:26 66
080913 21:52:27 37
080913 21:52:28 59
```
There was a drop in throughput in that output, which was interestingly also preceded
by a rush of queries completing. Without looking into the log around these timestamps
it’s hard to say what happened, but it’s possible that the spike is related to the drop
immediately afterward. In any case, it’s clear that something odd happened in this
server, and digging into the log around the timestamps in question could be very fruit-
ful. (When we looked into this log, we found that the spike was due to connections
being disconnected. Perhaps an application server was being restarted. Not everything
is a MySQL problem.)

**Making sense of the findings**

Nothing beats visualization of the data. We’ve shown only small examples here, but in
reality many of these techniques can result in thousands of lines of output. Get com-
fortable with _gnuplot_ or _R_ or another graphing tool of your choice. You can use them
to plot things in a jiffy—much faster than a spreadsheet—and you can instantly zoom
in on aberrations in a plot that you’ll have a hard time seeing in a scrolling terminal,
even if you think you’re pretty good at Matrix-watching.^13

We suggest trying the first two approaches—SHOW STATUS and SHOW PROCESSLIST—
initially, because they’re cheap and can be done interactively with nothing more than
a little bit of shell scripting or running queries repeatedly. Analyzing the slow query log
is much more disruptive and harder to do, and often shows what looks like funny
patterns that disappear as you look closer. We’ve found that it’s easy to imagine pat-
terns where there are none.

When you find an aberration, what does it mean? It usually means that queries are
queueing somewhere, or there’s a flood or spike of a particular kind of query. Now the
task is to find out why.

13. We haven’t seen the woman in the red dress yet, but we’ll let you know if we do.

**96 | Chapter 3: Profiling Server Performance**


#### Capturing Diagnostic Data

When an intermittent problem strikes, it’s important to measure _everything_ you pos-
sibly can, preferably for only the duration of the problem. If you do this right, you will
gather a ton of diagnostic data. The data you don’t collect often seems to be the data
you really need to diagnose the problem.

To get started, you need two things:

1. A reliable and real-time “trigger”—a way to know when the problem happens
2. A tool to gather the diagnostic data

**The diagnostic trigger**

The trigger is very important to get right. It’s the foundation for capturing the data
when the problem happens. There are two common problems that cause this to go
sideways: false positives and false negatives. If you have a false positive, you’ll gather
diagnostic data when nothing’s wrong, and you’ll waste time and get frustrated. False
negatives will result in missed opportunities and more wasted time and frustration.
Spend a little extra time making sure that your trigger indicates for sure that the problem
is happening, if you need to. It’s worth it.

What is a good criterion for a trigger? As shown in our examples, Threads_running
tends to be very sensitive to problems, but pretty stable when nothing is wrong. A spike
of unusual thread states in SHOW PROCESSLIST is another good indicator. But there can
be many more ways to observe the problem, including specific output in SHOW INNODB
STATUS, a spike in the server’s load average, and so on. The key is to express this as
something that you can compare to a definite threshold. This usually means a count.
A count of threads running, a count of threads in “freeing items” state, and so on works
well. The _-c_ option to _grep_ is your friend when looking at thread states:

```
$ mysql -e 'SHOW PROCESSLIST\G' | grep -c "State: freeing items"
36
```
Pick a threshold that’s high enough that you won’t hit it during normal operation, but
not so high that you won’t capture the problem in action. Beware, too, of setting the
threshold too high to catch the problem when it begins. Problems that escalate tend to
cause cascades of other problems, and if you capture diagnostic information after things
have really gone down the toilet, you’ll likely have a harder time isolating the original
cause. You want to collect your data when things are clearly circling the drain, if
possible, but before the loud flushing sound deafens you. For example, spikes in
Threads_connected can go insanely high—we’ve seen it escalate from 100 to 5000 or
more in the space of a couple minutes. You could clearly use 4999 as the threshold, but
why wait for things to get that bad? If the application doesn’t open more than 150
connections when it’s healthy, start collecting at 200 or 300.

Referring back to our earlier example of Threads_running, it looks like the normal con-
currency is less than 10. But 10 is not a good threshold—it is way too likely to produce

```
Diagnosing Intermittent Problems | 97
```

false positives, and 15 isn’t far enough away to definitely be out of the normal range of
behavior either. There could be a mini-pileup at 15, but it’s quite possible that it could
not quite cross the tipping point, and the problem could clear right up before getting
bad enough to be clearly diagnosable. We’d suggest setting 20 as the threshold in that
example.

You probably also want to capture the problem as soon as it is clearly happening, but
only after waiting briefly to ensure that it’s not a false positive or short-term spike. So,
our final trigger would be this: watch the status variables once per second, and if
Threads_running exceeds 20 for more than 5 seconds, start gathering diagnostic data.
(By the way, our example showed the problem going away after three seconds. That’s
a bit contrived to keep the example brief. A three-second problem is not likely to be
easily diagnosable, and most problems we’ve seen last a bit longer.)

Now you need to set up some kind of tool to watch the server and take action when
the trigger condition occurs. You could script this yourself, but we’ve saved you the
trouble. There’s a tool called _pt-stalk_ in Percona Toolkit that is custom-built just for
this. It has a lot of nice features whose necessity we’ve learned of through the school
of hard knocks. For example, it looks at how much disk space is free, so it won’t fill up
your disk with the data it collects and crash your server. Not that we’ve ever done that,
you understand!

The _pt-stalk_ tool is really simple to use. You can configure the variable to watch, the
threshold, the frequency of checks, and so forth. It supports a lot more fanciness than
that if needed, but that’s all you need to do for our example. Read the user’s manual
that comes with it before you use it. It relies on another tool for actually collecting the
data, which we’ll discuss next.

**What kinds of data should you collect?**

Now that you have determined a diagnostic trigger, you can use it to fire some process
to collect data. But what kind of data should you collect? The answer, as mentioned
previously, is everything you possibly can—but for only a reasonable amount of time.
Gather operating system stats, CPU usage, disk utilization and free space, samples of
_ps_ output, memory usage, and everything you can from within MySQL, such as samples
of SHOW STATUS, SHOW PROCESSLIST, and SHOW INNODB STATUS. You’ll need all of these
things, and probably more, to diagnose problems.

Execution time is spent doing work or waiting, as you’ll recall. When an unknown
problem happens, there are two types of causes, broadly speaking. The server could be
doing a lot of work—consuming a lot of CPU cycles—or it could be stuck waiting for
resources to become free. You need two different approaches to gather the diagnostic
data to identify the causes of each of these types of problems: a profile when the system
is doing too much work, and wait analysis when the system is doing too much waiting.
But how do you know which of these to focus on when the problem is unknown? You
don’t, so it’s best to collect data for both.

**98 | Chapter 3: Profiling Server Performance**


The primary profiling tool we rely on for server internals on GNU/Linux (as opposed
to queries server-wide) is _oprofile_. We’ll show examples of this a bit later. You can also
profile the server’s system calls with _strace_ , but we have found this to be riskier on
production systems. More on that later, too. For capturing queries to profile, we like
to use _tcpdump_. It’s hard to turn the slow query log on and off reliably at a moment’s
notice on most versions of MySQL, but you can get a pretty good simulation of it from
TCP traffic. Besides, the traffic is useful for lots of other kinds of analysis.

For wait analysis, we often use GDB stack traces.^14 Threads that are stuck in a particular
spot inside of MySQL for a long time tend to have the same stack trace. The procedure
is to start _gdb_ , attach it to the _mysqld_ process, and dump stack traces for all threads.
You can then use some short scripts to aggregate common stack traces together and do
the _sort|uniq|sort_ magic to show which ones are most common. We’ll show how to use
the _pt-pmp_ tool for this a bit later.

You can also do wait analysis with data such as snapshots of SHOW PROCESSLIST and
SHOW INNODB STATUS by observing thread and transaction states. None of these
approaches is perfectly foolproof, but in practice they work often enough to be very
helpful.

Gathering all of this data sounds like a lot of work! You probably anticipated this
already, but we’ve built a tool to do this for you too. It’s called _pt-collect_ , and it’s also
part of Percona Toolkit. It’s intended to be executed from _pt-stalk_. It needs to be run
as _root_ in order to gather most of the important data. By default, it will collect data for
30 seconds and then exit. This is usually enough to diagnose most problems, but not
so much that it causes problems when there’s a false positive.

The tool is easy to download and doesn’t need any configuration—all of the configu-
ration goes into _pt-stalk_. You will want to ensure that _gdb_ and _oprofile_ are installed on
your server, and enable those in the _pt-stalk_ configuration. You also need to ensure that
_mysqld_ has debug symbols.^15 When the trigger condition occurs, the tool will gather a
pretty complete set of data. It will create timestamped files in a specified directory. At
the time of writing, it’s rather oriented toward GNU/Linux and will need tweaking on
other operating systems, but it’s still a good place to start.

**Interpreting the data**

If you’ve set up your trigger condition correctly and let _pt-stalk_ run long enough to
catch the problem in action a few times, you’ll end up with a lot of data to sift through.
What’s the most useful place to start? We suggest looking at just a few things, with two

14. A caveat: using GDB is intrusive. It will freeze the server momentarily, especially if you have a lot of
    threads (connections), and can sometimes even crash it. The benefit still sometimes outweighs the risk.
    If the server becomes unusable anyway during a stall, it’s not such a bad thing to double-freeze it.
15. Sometimes symbols are omitted as an “optimization,” which really is not an optimization; it just makes
    diagnosing problems harder. You can use the _nm_ tool to check if you have them, and install the
    _debuginfo_ packages for MySQL to supply symbols.

```
Diagnosing Intermittent Problems | 99
```

purposes in mind. First, check that the problem really did occur, because if you have
many samples to examine, you won’t want to spend your time on false positives. Sec-
ond, see if something obvious jumps out at you.

```
It’s very helpful to capture samples of how the server looks when it’s
behaving well, not just when it’s in trouble. This will help you determine
whether a particular sample, or even a portion of a sample, is abnormal
or not. For example, when you’re looking at the states of queries in the
process list, you can answer questions such as “Is it normal for a lot of
queries to be sorting their results?”
```
The most fruitful things to look at are usually query or transaction behavior, and server
internals behavior. Query or transaction behavior shows you whether the problem was
caused by the way the server is being used: badly written SQL, bad indexing, bad logical
database design, and so on. You can see what the users are doing to the server by looking
at places where queries and transactions appear: in the logged TCP traffic, in the SHOW
PROCESSLIST output, and so on. Server internals behavior tells you whether the server
is buggy or has built-in performance or scalability problems. You can see this in some
of the same places, but also in _oprofile_ and _gdb_ output. This takes more experience to
interpret.

If you don’t know how to interpret what’s wrong, you can tarball the directory full of
collected data and submit it to a support provider for analysis. Any competent MySQL
support professional will be able to interpret the data and tell you what it means. And
they’ll love you for sending such detailed data to peruse. You might also want to send
the output of two other tools in Percona Toolkit: _pt-mysql-summary_ and _pt-summary_.
These show status and configuration snapshots of your MySQL instance and the op-
erating system and hardware, respectively.

Percona Toolkit includes a tool designed to help you look through lots of samples of
collected data quickly. It’s called _pt-sift_ , and it helps you navigate between samples,
shows a summary of each sample, and lets you drill down into particular bits of the
data if desired. It can save a lot of keystrokes.

We showed some examples of status counters and thread states earlier. We’ll finish out
this chapter by showing some examples of output from _oprofile_ and _gdb_. Here’s an
_oprofile_ report from a server that was having trouble. Can you find the problem?

```
samples % image name app name symbol name
893793 31.1273 /no-vmlinux /no-vmlinux (no symbols)
325733 11.3440 mysqld mysqld Query_cache::free_memory_block()
117732 4.1001 libc libc (no symbols)
102349 3.5644 mysqld mysqld my_hash_sort_bin
76977 2.6808 mysqld mysqld MYSQLparse()
71599 2.4935 libpthread libpthread pthread_mutex_trylock
52203 1.8180 mysqld mysqld read_view_open_now
46516 1.6200 mysqld mysqld Query_cache::invalidate_query_block_list()
42153 1.4680 mysqld mysqld Query_cache::write_result_data()
```
**100 | Chapter 3: Profiling Server Performance**


```
37359 1.3011 mysqld mysqld MYSQLlex()
35917 1.2508 libpthread libpthread __pthread_mutex_unlock_usercnt
34248 1.1927 mysqld mysqld __intel_new_memcpy
```
If you said “the query cache,” you were right. This server’s query cache was causing far
too much work and slowing everything down. This had happened overnight, a factor
of 50 slowdown, with no other changes to the system. Disabling the query cache re-
turned the server to its normal performance. This is an example of when server internals
are relatively straightforward to interpret.

Another important tool for bottleneck analysis is wait analysis using stack traces from
_gdb_. A single thread’s stack trace normally looks like the following, which we’ve for-
matted a bit for printing:

```
Thread 992 (Thread 0x7f6ee0111910 (LWP 31510)):
#0 0x0000003be560b2f9 in pthread_cond_wait@@GLIBC_2.3.2 () from /libpthread.so.0
#1 0x00007f6ee14f0965 in os_event_wait_low () at os/os0sync.c:396
#2 0x00007f6ee1531507 in srv_conc_enter_innodb () at srv/srv0srv.c:1185
#3 0x00007f6ee14c906a in innodb_srv_conc_enter_innodb () at handler/ha_innodb.cc:609
#4 ha_innodb::index_read () at handler/ha_innodb.cc:5057
#5 0x00000000006538c5 in ?? ()
#6 0x0000000000658029 in sub_select() ()
#7 0x0000000000658e25 in ?? ()
#8 0x00000000006677c0 in JOIN::exec() ()
#9 0x000000000066944a in mysql_select() ()
#10 0x0000000000669ea4 in handle_select() ()
#11 0x00000000005ff89a in ?? ()
#12 0x0000000000601c5e in mysql_execute_command() ()
#13 0x000000000060701c in mysql_parse() ()
#14 0x000000000060829a in dispatch_command() ()
#15 0x0000000000608b8a in do_command(THD*) ()
#16 0x00000000005fbd1d in handle_one_connection ()
#17 0x0000003be560686a in start_thread () from /lib64/libpthread.so.0
#18 0x0000003be4ede3bd in clone () from /lib64/libc.so.6
#19 0x0000000000000000 in ?? ()
```
The stack reads from the bottom up; that is, the thread is currently executing inside of
the pthread_cond_wait function, which was called from os_event_wait_low. Reading
down the trace, it looks like this thread was trying to enter the InnoDB kernel
(srv_conc_enter_innodb), but got put on an internal queue (os_event_wait_low) because
more than innodb_thread_concurrency threads were already inside the kernel. The real
value of stack traces is aggregating lots of them together, however. This is a technique
that Domas Mituzas, a former MySQL support engineer, made popular with his “poor
man’s profiler” tool. He currently works at Facebook, and he and others there have
developed a wide variety of tools for gathering and analyzing stack traces. You can find
out more about what’s available at _[http://www.poormansprofiler.org](http://www.poormansprofiler.org)_.

We have an implementation of the poor man’s profiler in Percona Toolkit, called _pt-
pmp_. It’s a shell and _awk_ program that collapses similar stack traces together and does
the usual _sort|uniq|sort_ to show the most common ones first. Here’s what the full set
of stack traces looks like after crunching it down to its essence. We’re going to use

```
Diagnosing Intermittent Problems | 101
```

the _-l 5_ option to truncate the stack traces after five levels so that we don’t get so many
traces with common tops but different bottoms, which would prevent them from ag-
gregating together and showing where things are really waiting:

```
$ pt-pmp -l 5 stacktraces.txt
507 pthread_cond_wait,one_thread_per_connection_end,handle_one_connection,
start_thread,clone
398 pthread_cond_wait,os_event_wait_low,srv_conc_enter_innodb,
innodb_srv_conc_enter_innodb,ha_innodb::index_read
83 pthread_cond_wait,os_event_wait_low,sync_array_wait_event,mutex_spin_wait,
mutex_enter_func
10 pthread_cond_wait,os_event_wait_low,os_aio_simulated_handle,fil_aio_wait,
io_handler_thread
7 pthread_cond_wait,os_event_wait_low,srv_conc_enter_innodb,
innodb_srv_conc_enter_innodb,ha_innodb::general_fetch
5 pthread_cond_wait,os_event_wait_low,sync_array_wait_event,rw_lock_s_lock_spin,
rw_lock_s_lock_func
1 sigwait,signal_hand,start_thread,clone,??
1 select,os_thread_sleep,srv_lock_timeout_and_monitor_thread,start_thread,clone
1 select,os_thread_sleep,srv_error_monitor_thread,start_thread,clone
1 select,handle_connections_sockets,main
1 read,vio_read_buff,::??,my_net_read,cli_safe_read
```
```
1 pthread_cond_wait,os_event_wait_low,sync_array_wait_event,rw_lock_x_lock_low,
rw_lock_x_lock_func
1 pthread_cond_wait,MYSQL_BIN_LOG::wait_for_update,mysql_binlog_send,
dispatch_command,do_command
1 fsync,os_file_fsync,os_file_flush,fil_flush,log_write_up_to
```
The first line is the characteristic signature of an idle thread in MySQL, so you can
ignore that. The second line is the most interesting one: it shows that a lot of threads
are waiting to enter the InnoDB kernel but are blocked. The third line shows many
threads waiting on some mutex, but we can’t see which one because we have truncated
the deeper levels of the stack trace. If it is important to know which mutex that is, we
would need to re run the tool with a larger value for the _-l_ option. In general, the stack
traces show that lots of things are waiting for their turn inside InnoDB, but why? That
isn’t clear at all. To find out, we probably need to look elsewhere.

As the preceding stack trace and _oprofile_ reports show, these types of analysis are not
always useful to those who aren’t experts with MySQL and InnoDB source code, and
you should ask for help from someone else if you get stuck.

Now let’s move on to a server whose problems don’t show up on either a profile or
wait analysis, and need to be diagnosed differently.

#### A Case Study in Diagnostics

In this section we’ll step you through the process of diagnosing a real customer’s in-
termittent performance problem. This case study is likely to get into unfamiliar territory
unless you’re an expert with MySQL, InnoDB, and GNU/Linux. However, the specifics

**102 | Chapter 3: Profiling Server Performance**


we’ll discuss are not the point. Try to look for the method within the madness: read
this section with an eye toward the assumptions and guesses we make, the reasoning-
based and measurement-based approaches we take, and so on. We are delving into a
specific and detailed case simply to illustrate generalities.

Before beginning to solve a problem at someone else’s request, it’s good to try to clear
up two things, preferably taking notes to help avoid forgetting or omitting anything:

1. First, what’s the problem? Try to be clear on that. It’s surprisingly easy to go hunt-
    ing for the wrong problem. In this case, the customer complained that once every
    day or two, the server rejected connections with a max_connections error. It lasted
    from a few seconds to a few minutes, and was highly sporadic.
2. Next, what has been done to try to fix it? In this case, the customer did not attempt
    to resolve the issue at all. This was extremely helpful, because few things are as
    hard to understand as another person’s description of the exact sequence of events,
    changes they made, and effects thereof. This is especially true when they call in
    desperation after a couple of sleepless nights and caffeine-filled days. A server that
    has been subjected to unknown changes, with unknown effects, is much harder to
    troubleshoot, especially if you’re under time pressure.

With that behind us, let’s get started. It’s a good idea not only to try to understand
how the server behaves, but also to take an inventory of the server’s status, configura-
tion, software, and hardware. We did so with the _pt-summary_ and _pt-mysql-summary_
tools. Briefly, this server had 16 CPU cores, 12 GB of RAM, and a total of 900 MB of
data, all in InnoDB, on a solid-state drive. The server was running GNU/Linux with
MySQL 5.1.37 and the InnoDB plugin version 1.0.4. We’d worked with this customer
previously on other unexpected performance problems, and we knew the systems. The
database was never the problem in the past; it had always been bad application behav-
ior. We took a look at the server and found nothing obviously wrong at a glance. The
queries weren’t perfect, but they were still running in less than 10 ms most of the time.
So we confirmed that the server was fine under normal circumstances. (This is impor-
tant to do; many problems that are noticed only sporadically are actually symptoms of
chronic problems, such as failed hard drives in RAID arrays.)

```
This case study might be a little tedious. We’ll “play dumb” to show all
of the diagnostic data, explain everything we see in detail, and follow
several potential trains of thought to completion. In reality, we don’t
take such a frustratingly slow approach to every problem, and we’re not
trying to say that you should, either.
```
We installed our diagnostic toolkit and set it to trigger on Threads_connected, which
was normally less than 15 but increased to several hundred during these problems.
We’ll present a sample of the data we collected as a result, but we’ll hold our

```
Diagnosing Intermittent Problems | 103
```

commentary until later. See if you can drink from the fire hose and pick out items that
are likely to be important:

- The query activity ranged from 1,000 to 10,000 queries per second, with many of
    them being “garbage” commands such as pinging the server to see if it was alive.
    Most of the rest were SELECT commands—from 300 to 2,000 per second—and there
    were a very small number of UPDATE commands (about 5 per second).
- There were basically two distinct types of queries in SHOW PROCESSLIST, varying only
    in the values in the WHERE clauses. Here are the query states, summarized:
       $ **grep State: processlist.txt | sort | uniq -c | sort -rn**
       161 State: Copying to tmp table
       156 State: Sorting result
       136 State: statistics
       50 State: Sending data
       24 State: NULL
       13 State:
       7 State: freeing items
       7 State: cleaning up
       1 State: storing result in query cache
       1 State: end
- Most queries were doing index scans or range scans—no full-table scans or cross
    joins.
- There were between 20 and 100 sorts per second, with between 1,000 and 12,000
    rows sorted per second.
- There were between 12 and 90 temporary tables created per second, with about 3
    to 5 of them on disk.
- There was no problem with table locking or the query cache.
- In SHOW INNODB STATUS, we observed that the main thread state was “flushing buffer
    pool pages,” but there were only a few dozen dirty pages to flush (Innodb_buffer
    _pool_pages_dirty), there was practically no change in Innodb_buffer_pool
    _pages_flushed, and the difference between the log sequence number and the last
    checkpoint was very small. The InnoDB buffer pool wasn’t even close to being full;
    it was much bigger than the data size. Most threads were waiting in the InnoDB
    queue: “12 queries inside InnoDB, 495 queries in queue.”
- We captured _iostat_ output for 30 seconds, one sample per second. This showed
    that there was essentially no read activity at all on the disks, but writes went through
    the roof, and average I/O wait times and queue length were extremely high. Here
    is the first bit of the output, simplified to fit on the page without wrapping:
       r/s w/s rsec/s wsec/s avgqu-sz await svctm %util
       1.00 500.00 8.00 86216.00 5.05 11.95 0.59 29.40
       0.00 451.00 0.00 206248.00 123.25 238.00 1.90 85.90
       0.00 565.00 0.00 269792.00 143.80 245.43 1.77 100.00
       0.00 649.00 0.00 309248.00 143.01 231.30 1.54 100.10
       0.00 589.00 0.00 281784.00 142.58 232.15 1.70 100.00
       0.00 384.00 0.00 162008.00 71.80 238.39 1.73 66.60

**104 | Chapter 3: Profiling Server Performance**


```
0.00 14.00 0.00 400.00 0.01 0.93 0.36 0.50
0.00 13.00 0.00 248.00 0.01 0.92 0.23 0.30
0.00 13.00 0.00 408.00 0.01 0.92 0.23 0.30
```
- The output of _vmstat_ confirmed what we saw in _iostat_ and showed that the CPUs
    were basically idle except for some I/O wait during the spike of writes (ranging up
    to 9% wait).

Is your brain full yet? This can happen quickly when you dig into a system in detail and
you don’t have (or you try to ignore) any preconceived notions, so you end up looking
at _everything_. Most of what you’ll look at is either completely normal, or shows the
effects of the problem but doesn’t indicate the source of the problem. Although at this
point we have some good guesses about the cause of the problem, we’ll keep going by
looking at the _oprofile_ report, and we’ll begin to add commentary and interpretation
as we throw more data at you:

```
samples % image name app name symbol name
473653 63.5323 no-vmlinux no-vmlinux /no-vmlinux
95164 12.7646 mysqld mysqld /usr/libexec/mysqld
53107 7.1234 libc-2.10.1.so libc-2.10.1.so memcpy
13698 1.8373 ha_innodb.so ha_innodb.so build_template()
13059 1.7516 ha_innodb.so ha_innodb.so btr_search_guess_on_hash
11724 1.5726 ha_innodb.so ha_innodb.so row_sel_store_mysql_rec
8872 1.1900 ha_innodb.so ha_innodb.so rec_init_offsets_comp_ordinary
7577 1.0163 ha_innodb.so ha_innodb.so row_search_for_mysql
6030 0.8088 ha_innodb.so ha_innodb.so rec_get_offsets_func
5268 0.7066 ha_innodb.so ha_innodb.so cmp_dtuple_rec_with_match
```
It’s not at all obvious what most of these symbols represent, and most of the time is
lumped together in the kernel^16 and in a generic _mysqld_ symbol that doesn’t tell us
anything.^17 Don’t get distracted by all of the ha_innodb.so symbols. Look at the per-
centage of time they contributed: regardless of what they do, they’re burning so little
time that you can be sure they’re not the problem. This is an example of a problem that
isn’t going to yield results from this type of profile analysis. We are looking at the wrong
data. When you see something like the previous sample, move along and look at other
data to see if there’s a more obvious pointer to the cause.

At this point, if you’re interested in the wait analysis from _gdb_ stack traces, please refer
to the end of the preceding section. The sample we showed there is from the system
we’re currently diagnosing. If you recall, the bulk of stack traces were simply waiting
to enter the InnoDB kernel, which corresponds to “12 queries inside InnoDB, 495
queries in queue” in the output from SHOW INNODB STATUS.

Do you see anything that points conclusively to a specific problem? We didn’t; we saw
possible symptoms of many different problems, and at least two potential causes of the

16. In theory, we need kernel symbols to understand what’s going on inside the kernel. In practice, this can
    be a hassle to install, and we know from looking at _vmstat_ that the system CPU usage was low, so we’re
    unlikely to find much other than “sleeping” there anyway.
17. It looks like this was a bad build of MySQL.

```
Diagnosing Intermittent Problems | 105
```

problem based on intuition and experience. But we also saw something that didn’t
make sense. If you look at the _iostat_ output again, in the wsec/s column you can see
that for about six seconds, the server is writing hundreds of megabytes of data per
second to the disks. Each sector is 512 bytes, so those samples show up to 150 MB of
writes per second at times. Yet the entire database is only 900 MB, and the workload
is mostly SELECT queries. How can this happen?

When you examine a system, try to ask yourself whether there are any things like this
that simply don’t add up, and investigate them further. Try to follow each train of
thought to its conclusion, and try not to get sidetracked on too many tangents, or you
could forget about a promising possibility. Write down little notes and cross them off
to help ensure that you’ve dotted all the Ts.^18

At this point, we could jump right to a conclusion, and it would be wrong. We see from
the main thread state that InnoDB is trying to flush dirty pages, which generally doesn’t
appear in the status output unless flushing is delayed. We know that this version of
InnoDB is prone to the “furious flushing” problem, also called a _checkpoint stall_. This
is what happens when InnoDB doesn’t spread flushing out evenly over time, and it
suddenly decides to force a checkpoint (flush a lot of data) to make up for that. This
can cause serious blocking inside InnoDB, making everything queue and wait to enter
the kernel, and thus pile up at the layers above InnoDB in the server. We showed an
example in Chapter 2 of the periodic drops in performance that can happen when there
is furious flushing. Many of this server’s symptoms are similar to what happens during
a forced checkpoint, but it’s not the problem in this case. You can prove that in many
ways, perhaps most easily by looking at the SHOW STATUS counters and tracking the
change in the Innodb_buffer_pool_pages_flushed counter, which, as we mentioned
earlier, was not increasing much. In addition, we noted that the buffer pool doesn’t
have much dirty data to flush anyway—certainly not hundreds of megabytes. This is
not surprising, because the workload on this server is almost entirely SELECT queries.
We can therefore conclude that instead of blaming the problem on InnoDB
flushing, we should blame InnoDB’s flushing delay on the problem. It is a symptom—
an effect—not a cause. The underlying problem is causing the disks to become satu-
rated that InnoDB isn’t having any luck getting its I/O tasks done. So we can eliminate
this as a possible cause, and cross off one of our intuition-based ideas.

Distinguishing cause from effect can be hard sometimes, and it can be tempting to just
skip the investigation and jump to the diagnosis when a problem looks familiar. It is
good to avoid taking shortcuts, but it’s equally important to pay attention to your
intuition. If something looks familiar, it is prudent to spend a little time measuring the
necessary and sufficient conditions to prove whether that’s the problem. This can save
a lot of time you’d otherwise spend looking through other data about the system and
its performance. Just try not to jump to conclusions based on a gut feeling that “I’ve

18. Or whatever that phrase is. Put all your eggs in one haystack?

**106 | Chapter 3: Profiling Server Performance**


seen this before, and I am sure it’s the same thing.” Gather some evidence, if you can—
especially evidence that could disprove your gut feeling.

The next step was to try to figure out what was causing the server’s I/O usage to be so
strange. We call your attention to the reasoning we used earlier: “The server writes
hundreds of megabytes to disk for many seconds, but the database is only 900 MB.
How can this happen?” Notice the implicit assumption that the database is doing the
writing? What evidence did we have that it’s the database? Try to catch yourself when
you think unsubstantiated thoughts, and when something doesn’t make sense, ask if
you’re assuming something. If you can, measure and remove the doubt.

We saw two possibilities: either the database was causing the I/O—and if we could
find the source of that, we thought that it was likely that we’d find the cause of the
problem—or, the database wasn’t doing all that I/O, but rather something else was,
and the lack of I/O resources could have been impacting the database. We’re stating
that very carefully to avoid another implicit assumption: just because the disks are busy
doesn’t guarantee that MySQL will suffer. Remember, this server basically has a read-
only in-memory workload, so it is quite possible to imagine that the disks could stop
responding for a long time without causing serious problems.

If you’re following our reasoning, you might see that we need to go back and gut-check
another assumption. We can see that the disk device was behaving badly, as evidenced
by the high wait times. A solid-state drive shouldn’t take a quarter of a second per I/O
on average. And indeed, we can see that _iostat_ claimed the drive itself was responding
quickly, but things were taking a long time to get through the block device queue to
the drive. Remember that this is only what _iostat_ claims; it could be wrong.

```
What Causes Poor Performance?
When a resource is behaving badly, it’s good to try to understand why. There are a few
possibilities:
```
1. The resource is being overworked and doesn’t have the capacity to behave well.
2. The resource is not configured properly.
3. The resource is broken or malfunctioning.
In the case we’re examining, _iostat_ ’s output could point to either too much work, or
misconfiguration (why are I/O requests queueing so long before reaching the disk, if
it’s actually responding quickly?). However, a very important part of deciding what’s
wrong is to compare the demand on the system to its known capacity. We know from
extensive benchmarking that the particular SSD drive this customer was using can’t
sustain hundreds of megabytes of writes per second. Thus, although _iostat_ claims the
disk is responding just fine, it’s likely that this isn’t entirely true. In this case, we had
no way to prove that the disk was slower than _iostat_ claimed, but it looked rather likely
to be the case. Still, this doesn’t change our opinion: this could be disk abuse,^19 mis-
configuration, or both.

```
Diagnosing Intermittent Problems | 107
```

After working through the diagnostic data to reach this point, the next task was obvi-
ous: measure what was causing the I/O. Unfortunately, this was infeasible on the ver-
sion of GNU/Linux the customer was using. We could have made an educated guess
with some work, but we wanted to explore other options first. As a proxy, we could
have measured how much I/O was coming from MySQL, but again, in this version of
MySQL that wasn’t really feasible due to lack of instrumentation.

Instead, we opted to try to observe MySQL’s I/O, based on what we know about how
it uses the disk. In general, MySQL writes only data, logs, sort files, and temporary
tables to disk. We eliminated data and logs from consideration, based on the status
counters and other information we discussed earlier. Now, suppose MySQL were to
suddenly write a bunch of data to disk temporary tables or sort files. How could we
observe this? Two easy ways are to watch the amount of free space on the disk, or to
look at the server’s open filehandles with the _lsof_ command. We did both, and the
results were convincing enough to satisfy us. Here’s what _df -h_ showed every second
during the same incident we’ve been studying:

```
Filesystem Size Used Avail Use% Mounted on
/dev/sda3 58G 20G 36G 36% /
/dev/sda3 58G 20G 36G 36% /
/dev/sda3 58G 19G 36G 35% /
/dev/sda3 58G 19G 36G 35% /
/dev/sda3 58G 19G 36G 35% /
/dev/sda3 58G 19G 36G 35% /
/dev/sda3 58G 18G 37G 33% /
/dev/sda3 58G 18G 37G 33% /
/dev/sda3 58G 18G 37G 33% /
```
And here’s the data from _lsof_ , which for some reason we gathered only once per five
seconds. We’re simply summing the sizes of all of the files _mysqld_ has open in _/tmp_ ,
and printing out the total for each timestamped sample in the file:

```
$ awk '
/mysqld.*tmp/ {
total += $7;
}
/^Sun Mar 28/ && total {
printf "%s %7.2f MB\n", $4, total/1024/1024;
total = 0;
}' lsof.txt
18:34:38 1655.21 MB
18:34:43 1.88 MB
18:34:48 1.88 MB
18:34:53 1.88 MB
18:34:58 1.88 MB
```
Based on this data, it looks like MySQL is writing about 1.5 GB of data to temporary
tables in the beginning phases of the incident, and this matches what we found in the
SHOW PROCESSLIST states (“Copying to tmp table”). The evidence points to a storm of

19. Someone call the 1-800 hotline!

**108 | Chapter 3: Profiling Server Performance**


bad queries all at once saturating the disk. The most common cause of this we’ve seen
(this is our intuition at work) is a cache stampede, when cached items expire all at once
from _memcached_ and many instances of the application try to repopulate the cache
simultaneously. We showed samples of the queries to the developers and discussed
their purpose. Indeed, it turned out that simultaneous cache expiration was the cause
(confirming our intuition). In addition to the developers addressing the problem at the
application level, we were able to help them modify the queries so they didn’t use
temporary tables on disk. Either one of these fixes might have prevented the problem,
but it was much better to do both than just one.

Now, we’d like to apply a little hindsight to explain some questions you might have
had as we went along (we certainly critiqued our own approach as we reviewed it for
this chapter):

_Why didn’t we just optimize the slow queries to begin with?_
Because the problem wasn’t slow queries, it was “too many connections” errors.
Sure, it’s logical to see that long-running queries cause things to stack up and the
connection count to climb. But so can dozens of other things. In the absence of
finding a good reason for why things are going wrong, it’s all too tempting to fall
back to looking for slow queries or other general things that look like they could
be improved.^20 But this goes badly more often than it goes well. If you took your
car to the mechanic and complained about an unfamiliar noise, and then got slap-
ped with a bill for balancing the tires and changing the transmission fluid because
the mechanic couldn’t figure out the real problem and went looking for other things
to do, wouldn’t you be annoyed?

_But isn’t it a red flag that the queries were running slowly with a bad_ EXPLAIN_?_
They were indeed—during the incidents. Was that a cause or an effect? It wasn’t
obvious until we dug into things more deeply. And remember, the queries seemed
to be running well enough in normal circumstances. Just because a query does a
filesort with a temporary table doesn’t mean it is a problem. Getting rid of filesorts
and temporary tables is a catch-all, “best practice” type of tactic.
Generic “best practices” reviews have their place, but they are seldom the solution
to highly specific problems. The problem could easily have been misconfiguration,
for example. We’ve seen many cases where someone tried to fix a misconfigured
server with tactics such as optimizing queries, which was ultimately a waste of time
and just prolonged the damage caused by the real problem.

_If cached items were being regenerated many times, wouldn’t there be multiple identical
queries?_
Yes, and this is something we did not investigate at the time. Multiple threads
regenerating the same cached item would indeed cause many completely identical
queries. (This is different from having multiple queries of the same general type,

20. Also known as the “when all you have is a hammer, everything looks like a nail” approach.

```
Diagnosing Intermittent Problems | 109
```

```
which might differ in a parameter to the WHERE clause, for example.) Noticing this
could have stimulated our intuition and directed us to the solution more quickly.
```
_There were hundreds of_ SELECT _queries per second, but only five_ UPDATE _s. What’s to say
that these five weren’t really heavy queries?_
They could indeed have been responsible for a lot of load on the server. We didn’t
show the actual queries because it would clutter things too much, but it’s a valid
point that the absolute number of each type of query isn’t necessarily meaningful.

_Isn’t the “proof” about the origin of the I/O storms still pretty weak?_
Yes, it is. There could be many explanations for why a small database would write
a huge amount of data to disk, or why the disk’s free space decreased quickly. This
is something that’s ultimately pretty hard to measure (though not impossible) on
the versions of MySQL and GNU/Linux in question. Although it’s possible to play
devil’s advocate and come up with lots of scenarios, we chose to balance the cost
and potential benefit by pursuing what seemed like the most promising leads first.
The harder it is to measure and be certain, the higher the cost/benefit ratio climbs,
and the more willing we are to accept uncertainty.

_We said “the database was never the problem in the past.” Wasn’t that a bias?_
Yes, that was a bias. If you caught it, great—if not, well, then hopefully it serves
as a useful illustration that we all have biases.

We’d like to finish this troubleshooting case study by pointing out that this issue
probably could have been solved (or prevented) without our involvement by using an
application profiling tool such as New Relic.

### Other Profiling Tools

We’ve shown a variety of ways to profile MySQL, the operating system, and queries.
We’ve demonstrated those that we think you’ll find most useful, and of course, we’ll
show more tools and techniques for inspecting and measuring systems throughout this
book. But wait, there’s more!

#### Using the USER_STATISTICS Tables

Percona Server and MariaDB include additional INFORMATION_SCHEMA tables for object-
level usage statistics. These were originally created at Google. They are extremely useful
for finding out how much or little the various parts of your server are actually used. In
a large enterprise, where the DBAs are responsible for managing the databases and have
little control over the developers, they can be vital for measuring and auditing database
activity and enforcing usage policies. They’re similarly useful for multitenant applica-
tions such as shared hosting environments. When you’re hunting for performance
problems, on the other hand, they can be great for helping you figure out who’s spend-
ing the most time in the database or what tables and indexes are most or least used.
Here are the tables:

**110 | Chapter 3: Profiling Server Performance**


```
mysql> SHOW TABLES FROM INFORMATION_SCHEMA LIKE '%_STATISTICS';
+---------------------------------------------+
| Tables_in_information_schema (%_STATISTICS) |
+---------------------------------------------+
| CLIENT_STATISTICS |
| INDEX_STATISTICS |
| TABLE_STATISTICS |
| THREAD_STATISTICS |
| USER_STATISTICS |
+---------------------------------------------+
```
We don’t have space for examples of all the queries you can perform against these
tables, but a couple of bullet points won’t hurt:

- You can find the most-used and least-used tables and indexes, by reads, updates,
    or both.
- You can find unused indexes, which are candidates for removal.
- You can look at the CONNECTED_TIME versus the BUSY_TIME of the replication user to
    see whether replication will likely have a hard time keeping up soon.

In MySQL 5.6, the Performance Schema adds tables that serve purposes similar to the
aforementioned tables.

#### Using strace

The _strace_ tool intercepts system calls. There are several ways you can use it. One is to
time the system calls and print out a profile:

```
$ strace -cfp $(pidof mysqld)
Process 12648 attached with 17 threads - interrupt to quit
^CProcess 12648 detached
% time seconds usecs/call calls errors syscall
------ ----------- ----------- --------- --------- ----------------
73.51 0.608908 13839 44 select
24.38 0.201969 20197 10 futex
0.76 0.006313 1 11233 3 read
0.60 0.004999 625 8 unlink
0.48 0.003969 22 180 write
0.23 0.001870 11 178 pread64
0.04 0.000304 0 5538 _llseek
[some lines omitted for brevity]
------ ----------- ----------- --------- --------- ----------------
100.00 0.828375 17834 46 total
```
In this way, it’s a bit like _oprofile_. However, _oprofile_ will profile the internal symbols of
the program, not just the system calls. In addition, _strace_ uses a different technique for
intercepting the calls, which is a bit more unpredictable and adds a lot of overhead.
And _strace_ measures wall-clock time, whereas _oprofile_ measures where the CPU cycles
are spent. As an example, _strace_ will show when I/O waits are a problem because it
measures from the beginning of a system call such as read or pread64 to the end of the

```
Other Profiling Tools | 111
```

call, but _oprofile_ usually won’t because the I/O system call doesn’t actually do any CPU
work—it just waits for the I/O to complete.

We use _oprofile_ only when necessary, because it can have strange side effects on a big
multithreaded process like _mysqld_ , and while _strace_ is attached, _mysqld_ will run so
slowly that it’s pretty much unusable for a production workload. Still, it can be ex-
tremely useful in some circumstances, and there is a tool in Percona Toolkit called
_pt-ioprofile_ that uses _strace_ to generate a true profile of I/O activity. This has been
helpful in proving or disproving some hard-to-measure cases that we couldn’t bring to
a close otherwise. (If the server had been running MySQL 5.6, we’d have been able to
do this with the Performance Schema instead.)

### Summary

This chapter lays the foundation of thought processes and techniques you’ll need to
succeed in performance optimization. The right mental approach is the key to unlock-
ing the full potential of your systems and applying the knowledge you’ll gain in the rest
of this book. Here are some of the fundamentals we tried to illustrate:

- We think that the most useful way to define performance is in terms of response
    time.
- You cannot reliably improve what you cannot measure, so performance improve-
    ment works best with high-quality, well-scoped, complete measurements of re-
    sponse time.
- The best place to start measuring is the application, not the database. If there is a
    problem in the lower layers, such as the database, good measurements will make
    it obvious.
- Most systems are impossible to measure completely, and measurements are
    always wrong. But you can usually work around the limitations and get a good
    outcome anyway, if you acknowledge the imperfection and uncertainty of the pro-
    cess you use.
- Thorough measurements produce way too much data to analyze, so you need a
    profile. This is the best tool to help you bubble important things to the top so you
    can decide where to start.
- A profile is a summary, which obscures and discards details. It also usually doesn’t
    show you what’s missing. It’s unwise to take a profile at face value.
- There are two kinds of time consumption: working and waiting. Many profiling
    tools can only measure work done, so wait analysis is sometimes an important
    supplement, especially when CPU usage is low but things still aren’t getting done.
- Optimization is not the same thing as improvement. Stop working when further
    improvement is not worth the cost.

**112 | Chapter 3: Profiling Server Performance**


- Pay attention to your intuition, but try to use it to direct your analysis, not to decide
    on changes to the system. Base your decisions on data, not gut feeling, as much as
    possible.

The overall approach we showed to solving performance problems is to first clarify the
question, then choose the appropriate technique to answer it. If you’re trying to see if
you can improve the server’s performance overall, a good way to start is to log all the
queries and produce a system-wide profile with _pt-query-digest_. If you’re hunting for
bad queries you might not know about, logging and profiling will also help. Look for
top time consumers, queries that are causing a bad user experience, and queries that
are highly variable or have strange response-time histograms. When you find “bad”
queries, drill into them by looking at the detailed report from _pt-query-digest_ , using
SHOW PROFILE, and using other tools such as EXPLAIN.

If the queries are performing badly for no discernible reason, you might be experiencing
sporadic server-wide problems. To find out, you can measure and plot the server’s
status counters at a fine level of detail. If this reveals a problem, use the same data to
formulate a reliable trigger condition, so you can capture a burst of detailed diagnostic
data. Invest as much time and care as necessary to find a good trigger condition that
avoids false positives and false negatives. If you capture the problem in action but you
still don’t understand the cause, gather more data, or ask for help.

You’re working with systems you can’t measure fully, but they’re just state machines,
so if you are careful, logical, and persistent, you will usually get results. Try not to
confuse effects with causes, and try not to make changes until you have identified the
problem.

The theoretically pure approach of top-down profiling and exhaustive measurement is
the ideal toward which we aspire, but we often have to deal with real systems. Real
systems are complex and inadequately instrumented, so we do the best we can with
what we have. Tools such as _pt-query-digest_ and the MySQL Enterprise Monitor’s query
analyzer aren’t perfect, and often won’t show conclusive proof of a problem’s cause.
But they are good enough to get the job done much of the time.

```
Summary | 113
```
