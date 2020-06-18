

**THIRD EDITION**

# High Performance MySQL

#### Baron Schwartz, Peter Zaitsev, and Vadim Tkachenko

```
Beijing • Cambridge • Farnham • Köln • Sebastopol • Tokyo
```

**High Performance MySQL, Third Edition**
by Baron Schwartz, Peter Zaitsev, and Vadim Tkachenko

Copyright © 2012 Baron Schwartz, Peter Zaitsev, and Vadim Tkachenko. All rights reserved.
Printed in the United States of America.

Published by O’Reilly Media, Inc., 1005 Gravenstein Highway North, Sebastopol, CA 95472.

O’Reilly books may be purchased for educational, business, or sales promotional use. Online editions
are also available for most titles ( _[http://my.safaribooksonline.com](http://my.safaribooksonline.com)_ ). For more information, contact our
corporate/institutional sales department: (800) 998-9938 or _corporate@oreilly.com_.

**Editor:** Andy Oram
**Production Editor:** Holly Bauer
**Proofreader:** Rachel Head

```
Indexer: Jay Marchand
Cover Designer: Karen Montgomery
Interior Designer: David Futato
Illustrator: Rebecca Demarest
```
March 2004: First Edition.
June 2008: Second Edition.
March 2012: Third Edition.

**Revision History for the Third Edition:**
2012-03-01 First release
See _[http://oreilly.com/catalog/errata.csp?isbn=9781449314286](http://oreilly.com/catalog/errata.csp?isbn=9781449314286)_ for release details.

Nutshell Handbook, the Nutshell Handbook logo, and the O’Reilly logo are registered trademarks of
O’Reilly Media, Inc. _High Performance MySQL_ , the image of a sparrow hawk, and related trade dress
are trademarks of O’Reilly Media, Inc.

Many of the designations used by manufacturers and sellers to distinguish their products are claimed as
trademarks. Where those designations appear in this book, and O’Reilly Media, Inc., was aware of a
trademark claim, the designations have been printed in caps or initial caps.

While every precaution has been taken in the preparation of this book, the publisher and authors assume
no responsibility for errors or omissions, or for damages resulting from the use of the information con-
tained herein.

ISBN: 978-1-449-31428-

[LSI]

1330630256


## Table of Contents

**Foreword................................................................... xv**











- 1. MySQL Architecture and History Preface xvii
   - MySQL’s Logical Architecture
      - Connection Management and Security
      - Optimization and Execution
   - Concurrency Control
      - Read/Write Locks
      - Lock Granularity
   - Transactions
      - Isolation Levels
      - Deadlocks
      - Transaction Logging
      - Transactions in MySQL
   - Multiversion Concurrency Control
   - MySQL’s Storage Engines
      - The InnoDB Engine
      - The MyISAM Engine
      - Other Built-in MySQL Engines
      - Third-Party Storage Engines
      - Selecting the Right Engine
      - Table Conversions
   - A MySQL Timeline
   - MySQL’s Development Model
   - Summary
- 2. Benchmarking MySQL
   - Why Benchmark?
   - Benchmarking Strategies
      - What to Measure
   - Benchmarking Tactics
      - Designing and Planning a Benchmark
      - How Long Should the Benchmark Last?
      - Capturing System Performance and Status
      - Getting Accurate Results
      - Running the Benchmark and Analyzing Results
      - The Importance of Plotting
   - Benchmarking Tools
      - Full-Stack Tools
      - Single-Component Tools
   - Benchmarking Examples
      - http_load
      - MySQL Benchmark Suite
      - sysbench
      - dbt2 TPC-C on the Database Test Suite
      - Percona’s TPCC-MySQL Tool
   - Summary
- 3. Profiling Server Performance
   - Introduction to Performance Optimization
      - Optimization Through Profiling
      - Interpreting the Profile
   - Profiling Your Application
      - Instrumenting PHP Applications
   - Profiling MySQL Queries
      - Profiling a Server’s Workload
      - Profiling a Single Query
      - Using the Profile for Optimization
   - Diagnosing Intermittent Problems
      - Single-Query Versus Server-Wide Problems
      - Capturing Diagnostic Data
      - A Case Study in Diagnostics
   - Other Profiling Tools
      - Using the USER_STATISTICS Tables
      - Using strace
   - Summary
- 4. Optimizing Schema and Data Types
   - Choosing Optimal Data Types
      - Whole Numbers
      - Real Numbers
      - String Types
      - Date and Time Types
      - Bit-Packed Data Types
      - Choosing Identifiers
      - Special Types of Data
   - Schema Design Gotchas in MySQL
   - Normalization and Denormalization
      - Pros and Cons of a Normalized Schema
      - Pros and Cons of a Denormalized Schema
      - A Mixture of Normalized and Denormalized
   - Cache and Summary Tables
      - Materialized Views
      - Counter Tables
   - Speeding Up ALTER TABLE
      - Modifying Only the .frm File
      - Building MyISAM Indexes Quickly
   - Summary
- 5. Indexing for High Performance
   - Indexing Basics
      - Types of Indexes
   - Benefits of Indexes
   - Indexing Strategies for High Performance
      - Isolating the Column
      - Prefix Indexes and Index Selectivity
      - Multicolumn Indexes
      - Choosing a Good Column Order
      - Clustered Indexes
      - Covering Indexes
      - Using Index Scans for Sorts
      - Packed (Prefix-Compressed) Indexes
      - Redundant and Duplicate Indexes
      - Unused Indexes
      - Indexes and Locking
   - An Indexing Case Study
      - Supporting Many Kinds of Filtering
      - Avoiding Multiple Range Conditions
      - Optimizing Sorts
   - Index and Table Maintenance
      - Finding and Repairing Table Corruption
      - Updating Index Statistics
      - Reducing Index and Data Fragmentation
   - Summary
- 6. Query Performance Optimization
   - Why Are Queries Slow?
   - Slow Query Basics: Optimize Data Access
      - Are You Asking the Database for Data You Don’t Need?
      - Is MySQL Examining Too Much Data?
   - Ways to Restructure Queries
      - Complex Queries Versus Many Queries
      - Chopping Up a Query
      - Join Decomposition
   - Query Execution Basics
      - The MySQL Client/Server Protocol
      - The Query Cache
      - The Query Optimization Process
      - The Query Execution Engine
      - Returning Results to the Client
   - Limitations of the MySQL Query Optimizer
      - Correlated Subqueries
      - UNION Limitations
      - Index Merge Optimizations
      - Equality Propagation
      - Parallel Execution
      - Hash Joins
      - Loose Index Scans
      - MIN() and MAX()
      - SELECT and UPDATE on the Same Table
   - Query Optimizer Hints
   - Optimizing Specific Types of Queries
      - Optimizing COUNT() Queries
      - Optimizing JOIN Queries
      - Optimizing Subqueries
      - Optimizing GROUP BY and DISTINCT
      - Optimizing LIMIT and OFFSET
      - Optimizing SQL_CALC_FOUND_ROWS
      - Optimizing UNION
      - Static Query Analysis
      - Using User-Defined Variables
   - Case Studies
      - Building a Queue Table in MySQL
      - Computing the Distance Between Points
      - Using User-Defined Functions
   - Summary
- 7. Advanced MySQL Features
   - Partitioned Tables
      - How Partitioning Works
      - Types of Partitioning
      - How to Use Partitioning
      - What Can Go Wrong
      - Optimizing Queries
      - Merge Tables
   - Views
      - Updatable Views
      - Performance Implications of Views
      - Limitations of Views
   - Foreign Key Constraints
   - Storing Code Inside MySQL
      - Stored Procedures and Functions
      - Triggers
      - Events
      - Preserving Comments in Stored Code
   - Cursors
   - Prepared Statements
      - Prepared Statement Optimization
      - The SQL Interface to Prepared Statements
      - Limitations of Prepared Statements
   - User-Defined Functions
   - Plugins
   - Character Sets and Collations
      - How MySQL Uses Character Sets
      - Choosing a Character Set and Collation
      - How Character Sets and Collations Affect Queries
   - Full-Text Searching
      - Natural-Language Full-Text Searches
      - Boolean Full-Text Searches
      - Full-Text Changes in MySQL 5.1
      - Full-Text Tradeoffs and Workarounds
      - Full-Text Configuration and Optimization
   - Distributed (XA) Transactions
      - Internal XA Transactions
      - External XA Transactions
   - The MySQL Query Cache
      - How MySQL Checks for a Cache Hit
      - How the Cache Uses Memory
      - When the Query Cache Is Helpful
      - How to Configure and Maintain the Query Cache
      - InnoDB and the Query Cache
      - General Query Cache Optimizations
      - Alternatives to the Query Cache
   - Summary
- 8. Optimizing Server Settings
   - How MySQL’s Configuration Works
      - Syntax, Scope, and Dynamism
      - Side Effects of Setting Variables
      - Getting Started
      - Iterative Optimization by Benchmarking
   - What Not to Do
   - Creating a MySQL Configuration File
      - Inspecting MySQL Server Status Variables
   - Configuring Memory Usage
      - How Much Memory Can MySQL Use?
      - Per-Connection Memory Needs
      - Reserving Memory for the Operating System
      - Allocating Memory for Caches
      - The InnoDB Buffer Pool
      - The MyISAM Key Caches
      - The Thread Cache
      - The Table Cache
      - The InnoDB Data Dictionary
   - Configuring MySQL’s I/O Behavior
      - InnoDB I/O Configuration
      - MyISAM I/O Configuration
   - Configuring MySQL Concurrency
      - InnoDB Concurrency Configuration
      - MyISAM Concurrency Configuration
   - Workload-Based Configuration
      - Optimizing for BLOB and TEXT Workloads
      - Optimizing for Filesorts
   - Completing the Basic Configuration
   - Safety and Sanity Settings
   - Advanced InnoDB Settings
   - Summary
- 9. Operating System and Hardware Optimization
   - What Limits MySQL’s Performance?
   - How to Select CPUs for MySQL
      - Which Is Better: Fast CPUs or Many CPUs?
      - CPU Architecture
   - Scaling to Many CPUs and Cores
- Balancing Memory and Disk Resources
   - Random Versus Sequential I/O
   - Caching, Reads, and Writes
   - What’s Your Working Set?
   - Finding an Effective Memory-to-Disk Ratio
   - Choosing Hard Disks
- Solid-State Storage
   - An Overview of Flash Memory
   - Flash Technologies
   - Benchmarking Flash Storage
   - Solid-State Drives (SSDs)
   - PCIe Storage Devices
   - Other Types of Solid-State Storage
   - When Should You Use Flash?
   - Using Flashcache
   - Optimizing MySQL for Solid-State Storage
- Choosing Hardware for a Replica
- RAID Performance Optimization
   - RAID Failure, Recovery, and Monitoring
   - Balancing Hardware RAID and Software RAID
   - RAID Configuration and Caching
- Storage Area Networks and Network-Attached Storage
   - SAN Benchmarks
   - Using a SAN over NFS or SMB
   - MySQL Performance on a SAN
   - Should You Use a SAN?
- Using Multiple Disk Volumes
- Network Configuration
- Choosing an Operating System
- Choosing a Filesystem
- Choosing a Disk Queue Scheduler
- Threading
- Swapping
- Operating System Status
   - How to Read vmstat Output
   - How to Read iostat Output
   - Other Helpful Tools
   - A CPU-Bound Machine
   - An I/O-Bound Machine
   - A Swapping Machine
   - An Idle Machine
- Summary
- 10. Replication
   - Replication Overview
      - Problems Solved by Replication
      - How Replication Works
   - Setting Up Replication
      - Creating Replication Accounts
      - Configuring the Master and Replica
      - Starting the Replica
      - Initializing a Replica from Another Server
      - Recommended Replication Configuration
   - Replication Under the Hood
      - Statement-Based Replication
      - Row-Based Replication
      - Statement-Based or Row-Based: Which Is Better?
      - Replication Files
      - Sending Replication Events to Other Replicas
      - Replication Filters
   - Replication Topologies
      - Master and Multiple Replicas
      - Master-Master in Active-Active Mode
      - Master-Master in Active-Passive Mode
      - Master-Master with Replicas
      - Ring Replication
      - Master, Distribution Master, and Replicas
      - Tree or Pyramid
      - Custom Replication Solutions
   - Replication and Capacity Planning
      - Why Replication Doesn’t Help Scale Writes
      - When Will Replicas Begin to Lag?
      - Plan to Underutilize
   - Replication Administration and Maintenance
      - Monitoring Replication
      - Measuring Replication Lag
      - Determining Whether Replicas Are Consistent with the Master
      - Resyncing a Replica from the Master
      - Changing Masters
      - Switching Roles in a Master-Master Configuration
   - Replication Problems and Solutions
      - Errors Caused by Data Corruption or Loss
      - Using Nontransactional Tables
      - Mixing Transactional and Nontransactional Tables
      - Nondeterministic Statements
      - Different Storage Engines on the Master and Replica
      - Data Changes on the Replica
      - Nonunique Server IDs
      - Undefined Server IDs
      - Dependencies on Nonreplicated Data
      - Missing Temporary Tables
      - Not Replicating All Updates
      - Lock Contention Caused by InnoDB Locking Selects
      - Writing to Both Masters in Master-Master Replication
      - Excessive Replication Lag
      - Oversized Packets from the Master
      - Limited Replication Bandwidth
      - No Disk Space
      - Replication Limitations
   - How Fast Is Replication?
   - Advanced Features in MySQL Replication
   - Other Replication Technologies
   - Summary
- 11. Scaling MySQL
   - What Is Scalability?
      - A Formal Definition
   - Scaling MySQL
      - Planning for Scalability
      - Buying Time Before Scaling
      - Scaling Up
      - Scaling Out
      - Scaling by Consolidation
      - Scaling by Clustering
      - Scaling Back
   - Load Balancing
      - Connecting Directly
      - Introducing a Middleman
      - Load Balancing with a Master and Multiple Replicas
   - Summary
- 12. High Availability
   - What Is High Availability?
   - What Causes Downtime?
   - Achieving High Availability
      - Improving Mean Time Between Failures
      - Improving Mean Time to Recovery
   - Avoiding Single Points of Failure
      - Shared Storage or Replicated Disk
      - Synchronous MySQL Replication
      - Replication-Based Redundancy
   - Failover and Failback
      - Promoting a Replica or Switching Roles
      - Virtual IP Addresses or IP Takeover
      - Middleman Solutions
      - Handling Failover in the Application
   - Summary
- 13. MySQL in the Cloud
   - Benefits, Drawbacks, and Myths of the Cloud
   - The Economics of MySQL in the Cloud
   - MySQL Scaling and HA in the Cloud
   - The Four Fundamental Resources
   - MySQL Performance in Cloud Hosting
      - Benchmarks for MySQL in the Cloud
   - MySQL Database as a Service (DBaaS)
      - Amazon RDS
      - Other DBaaS Solutions
   - Summary
- 14. Application-Level Optimization
   - Common Problems
   - Web Server Issues
      - Finding the Optimal Concurrency
   - Caching
      - Caching Below the Application
      - Application-Level Caching
      - Cache Control Policies
      - Cache Object Hierarchies
      - Pregenerating Content
      - The Cache as an Infrastructure Component
      - Using HandlerSocket and memcached Access
   - Extending MySQL
   - Alternatives to MySQL
   - Summary
- 15. Backup and Recovery
   - Why Backups?
   - Defining Recovery Requirements
   - Designing a MySQL Backup Solution
      - Online or Offline Backups?
      - Logical or Raw Backups?
            - What to Back Up
            - Storage Engines and Consistency
            - Replication
         - Managing and Backing Up Binary Logs
            - The Binary Log Format
            - Purging Old Binary Logs Safely
         - Backing Up Data
            - Making a Logical Backup
            - Filesystem Snapshots
         - Recovering from a Backup
            - Restoring Raw Files
            - Restoring Logical Backups
            - Point-in-Time Recovery
            - More Advanced Recovery Techniques
            - InnoDB Crash Recovery
         - Backup and Recovery Tools
            - MySQL Enterprise Backup
            - Percona XtraBackup
            - mylvmbackup
            - Zmanda Recovery Manager
            - mydumper
            - mysqldump
         - Scripting Backups
         - Summary
- 16. Tools for MySQL Users
         - Interface Tools
         - Command-Line Utilities
         - SQL Utilities
         - Monitoring Tools
            - Open Source Monitoring Tools
            - Commercial Monitoring Systems
            - Command-Line Monitoring with Innotop
         - Summary
   - A. Forks and Variants of MySQL
   - B. MySQL Server Status
      - C. Transferring Large Files
   - D. Using EXPLAIN


**E. Debugging Locks ..................................................... 735**

**F. Using Sphinx with MySQL .............................................. 745**

**Index..................................................................... 771**

**xiv |Table of Contents**
