Transaction isolation level test suite
======================================

[This git repository](https://github.com/ept/isolation) is an attempt to nail down precisely what
different database systems actually mean with their isolation levels. It's a suite of tests that
simulates various concurrency issues — some common, some more obscure — and documents how different
databases handle those situations.

This project was started by [Martin Kleppmann](http://martin.kleppmann.com/) as background research
for his book, [Designing Data-Intensive Applications](http://dataintensive.net/). In this repository
you'll find a lot of nitty-gritty detail. For a gentle, friendly introduction to the topic, please
read the book.


Summary of test results
-----------------------

The cryptic abbreviations (G1c, PMP etc) are different kinds of concurrency *anomalies* — issues
which can occur when multiple clients are executing transactions at the same time, and which can
cause application bugs. The precise definitions of these anomalies are given in the literature
(see below for details).

| DBMS          | So-called isolation level    | Actual isolation level | G0 | G1a | G1b | G1c | OTV | PMP | P4 | G-single | G2-item | G2   |
|:--------------|:-----------------------------|:-----------------------|:--:|:---:|:---:|:---:|:---:|:---:|:--:|:--------:|:-------:|:----:|
| PostgreSQL    | "read committed" ★           | monotonic atomic views | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
|               | "repeatable read"            | snapshot isolation     | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | —       | —    |
|               | "serializable"               | serializable           | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | ✓       | ✓    |
|               |                              |                        |    |     |     |     |     |     |    |          |         |      |
| MySQL/InnoDB  | "read uncommitted"           | read uncommitted       | ✓  | —   | —   | —   | —   | —   | —  | —        | —       | —    |
|               | "read committed"             | monotonic atomic views | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
|               | "repeatable read" ★          | monotonic atomic views | ✓  | ✓   | ✓   | ✓   | ✓   | R/O | —  | R/O      | —       | —    |
|               | "serializable"               | serializable           | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | ✓       | ✓    |
|               |                              |                        |    |     |     |     |     |     |    |          |         |      |
| Oracle DB     | "read committed" ★           | monotonic atomic views | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
|               | "serializable"               | snapshot isolation     | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | —       | some |
|               |                              |                        |    |     |     |     |     |     |    |          |         |      |
| MS SQL Server | "read uncommitted"           | read uncommitted       | ✓  | —   | —   | —   | —   | —   | —  | —        | —       | —    |
|               | "read committed" (locking) ★ | monotonic atomic views | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
|               | "read committed" (snapshot)  | monotonic atomic views | ✓  | ✓   | ✓   | ✓   | ✓   | —   | —  | —        | —       | —    |
|               | "repeatable read"            | repeatable read        | ✓  | ✓   | ✓   | ✓   | ✓   | —   | ✓  | some     | ✓       | —    |
|               | "snapshot"                   | snapshot isolation     | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | —       | —    |
|               | "serializable"               | serializable           | ✓  | ✓   | ✓   | ✓   | ✓   | ✓   | ✓  | ✓        | ✓       | ✓    |

Legend:

* ★ = default configuration
* ✓ = isolation level prevents this anomaly from occurring
* — = isolation level does not prevent this anomaly, so it can occur
* R/O = isolation level prevents this anomaly in a read-only context, but when you perform writes,
  the anomaly can occur (see test cases for details)
* some = isolation level prevents this anomaly in some cases, but not in others (see test cases for details)


Background
----------

*Isolation* is the I in ACID, and it describes how a database protects an application from
concurrency problems (race conditions). If you read a traditional 
[database theory textbook](http://research.microsoft.com/en-us/people/philbe/ccontrol.aspx),
it will tell you that isolation is supposed to mean *serializability*, i.e. you can pretend
that transactions are executed one after another, and concurrency problems do not happen.
However, if you look at the implementations of
[isolation in practice](http://www.bailis.org/blog/when-is-acid-acid-rarely/), you see that
serializability is rarely used, and some popular databases (such as Oracle) don't even implement it.

So what does isolation actually mean? Well, in practice, many database systems allow you to choose your
isolation level, as a trade-off between performance and safety (weaker isolation is faster but exposes
you to more potential race conditions). Unfortunately, those weaker isolation levels are quite
[poorly understood](http://www.bailis.org/blog/understanding-weak-isolation-is-a-serious-problem/).
Even though our industry has been working with this stuff for 20 years or more, there are not many
people who can explain off-the-cuff the difference between, say, *read committed* and *repeatable read*.
This is a problem, because if you don't know what guarantees you can expect from your database, you
cannot know whether your code has concurrency bugs and race conditions.

The [SQL standard](http://synthesis.ipi.ac.ru/synthesis/student/oodb/essayRef/sqlFoundation) tried
to define four isolation levels (read uncommitted, read committed, repeatable read and serializable),
but its definition is [flawed](http://research.microsoft.com/pubs/69541/tr-95-51.pdf). Several academic
researchers have tried to nail down more precise definitions of weak (i.e. non-serializable) isolation
levels. In particular:

* Peter Bailis, Aaron Davidson, Alan Fekete, Ali Ghodsi, Joseph M Hellerstein and Ion Stoica:
  “[Highly Available Transactions: Virtues and Limitations (Extended Version)](http://arxiv.org/pdf/1302.0309.pdf),”
  at *40th International Conference on Very Large Data Bases* (VLDB), September 2014.
* Alan Fekete, Dimitrios Liarokapis, Elizabeth O'Neil, Patrick O'Neil, and Dennis Shasha:
  “[Making Snapshot Isolation Serializable](http://www.researchgate.net/publication/220225203_Making_snapshot_isolation_serializable/file/e0b49520567eace81f.pdf),”
  *ACM Transactions on Database Systems* (TODS), volume 30, number 2, pages 492–528, June 2005.
  [doi:10.1145/1071610.1071615](http://dx.doi.org/10.1145/1071610.1071615)
* Atul Adya: “[Weak Consistency: A Generalized Theory and Optimistic Implementations for Distributed
  Transactions](http://pmg.csail.mit.edu/papers/adya-phd.pdf),” PhD Thesis, Massachusetts Institute of
  Technology, Cambridge, MA, USA, March 1999.
* Hal Berenson, Phil Bernstein, Jim Gray, Jim Melton, Elizabeth O'Neil and Patrick O'Neil:
  “[A Critique of ANSI SQL Isolation Levels](http://research.microsoft.com/pubs/69541/tr-95-51.pdf),”
  at *ACM International Conference on Management of Data* (SIGMOD), volume 24, number 2, May 1995.
  [doi:10.1145/568271.223785](http://dx.doi.org/10.1145/568271.223785)
* Jim N Gray, Raymond A Lorie, Gianfranco R Putzolu, and Irving L Traiger:
  “[Granularity of Locks and Degrees of Consistency in a Shared Data
  Base](http://citeseer.ist.psu.edu/viewdoc/download?doi=10.1.1.92.8248&rep=rep1&type=pdf),”
  in *Modelling in Data Base Management Systems: Proceedings of the IFIP Working Conference on
  Modelling in Data Base Management Systems*, edited by G.M. Nijssen, Elsevier/North Holland
  Publishing, pages 364–394, 1976.
  Also in [*Readings in Database Systems*](http://redbook.cs.berkeley.edu/), edited by Joseph M.
  Hellerstein and Michael Stonebraker, 4th edition, MIT Press, 2005. ISBN: 978-0-262-69314-1

This project is based on the formal definition of weak isolation introduced by Adya, as extended by
Bailis et al. They mathematically define certain *anomalies* (or *phenomena*) which can occur in an
unrestricted concurrency model, and define isolation levels as *prohibiting* or *preventing* certain
anomalies from occurring.

The formal definitions are not easy to understand, but at least they are precise. By comparison, the
database vendors' documentation of isolation levels is also hard to understand, but on top of that
it's also frustratingly vague:

* [PostgreSQL](http://www.postgresql.org/docs/9.3/static/transaction-iso.html)
* [MySQL/InnoDB](http://dev.mysql.com/doc/refman/5.7/en/set-transaction.html)
* [Oracle](https://docs.oracle.com/cd/B28359_01/server.111/b28318/consist.htm)
* [SQL Server](http://msdn.microsoft.com/en-us/library/ms173763.aspx)


Goals of this project
---------------------

This repository contains a series of tests which probe for a range of concurrency anomalies.
They are based on the definitions in the literature above. This is useful for several reasons:

* It allows us to compare isolation levels easily: the more check marks in the table above,
  the stronger its guarantees.
* For anyone who needs help choosing the right isolation level for their application, the test
  suites provide concrete examples of the differences between isolation levels.
* Hopefully, this effort can be part of a journey towards a better understanding of weak
  isolation. It looks like weak isolation isn't going away, so we need to learn to be more
  precise about what it means, otherwise we'll just continue creating buggy applications.


Using this project
------------------

The tests are currently executed by hand: you simply open two or three connections to the
same database in different terminal windows, and run the queries in the order they appear
in the test script. A comment indicates which transaction executes a particular query, and
what the expected result is.

This could probably be automated, but it's actually quite interesting to go through the
exercise of stepping through transactions one line at a time, and watching how the
database responds.

At the moment, this project only compares four databases, but many more databases offer
transactions. It would be especially interesting to add the new generation of distributed
transactional databases ("NewSQL" if you like marketing-speak) to this comparison:
FoundationDB, Aerospike, NuoDB, MemSQL, etc. If you want to port the test suite to one
of those databases, or add new tests, your contribution would be most welcome!
 

License
-------

Copyright Martin Kleppmann, 2014. This work is licensed under a
[Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).

[![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png)](http://creativecommons.org/licenses/by/4.0/)