+++
author = "Jonathan Katz"
title = "A tribute to PostgreSQL 10"
date = "2022-11-10"
description = "A look back on the impact of PostgreSQL 10, which went EOL in Nov 2022."
tags = [
    "postgres",
    "postgresql",
]
+++

The [last release of PostgreSQL 10](https://www.postgresql.org/about/news/postgresql-151-146-139-1213-1118-and-1023-released-2543/) took place on Nov 10, 2022. While there are many articles on the new and exciting features in PostgreSQL, I thought this would be a good time to reflect on the impact of PostgreSQL 10.

PostgreSQL 10 had a lot of firsts. It was the first PostgreSQL release to have the two digit numbering scheme (PostgreSQL 10 vs. PostgreSQL 9.6). It was the first release with logical replication, declarative partitioning, and SCRAM.

The PostgreSQL 10 release also had personal significance. The [release announcement](https://www.postgresql.org/about/news/postgresql-10-released-1786/) was the first major version release where I helped to assemble the different pieces. This involved reading through the release notes, asking developers what exactly they built, iterating through the content of the release with the community, and working with our international community to localize and translate the content (and if you are interested in helping with any of this process, reach out to me!).

As we are now at the final release of PostgreSQL 10, I wanted to take a moment to memorialize this release and review the lasting impact it has had on the project and how people run their database workloads on PostgreSQL.

## What's in a number? Moving from X.Y.Z => X.Y

One of the hardest problems in computer science is naming things. There is a corollary that [versioning is a close second](https://www.postgresql.org/message-id/flat/CA%2BTgmoa2HVxpTUOjTvEMDrZfmdoFuJ3BMaBDE1Y5xAnA%3DXiRLw%40mail.gmail.com). There are a variety of reasons why PostgreSQL moved from a `X.Y.Z` versioning scheme to a `X.Y`, but one that stood out to me was around the clarity of upgrading. It is easier to explain the upgrade jump between a "10" and "11" than a "9.5" and "9.6". It also help remove the terms like "PostgreSQL 9" from the lexicon, especially given the huge feature difference between 9.0 and 9.6.

It did take some time to propagate information for how this worked -- for years the release announcements included guidance on the new numbering scheme. Of everything in the PostgreSQL 10 release, this is arguably the most impactful change simply based on how it made people talk about PostgreSQL.

## Parallelize "all the things!"

PostgreSQL 9.6 was the first release to introduce parallel query, or the ability to have multiple workers simultaneously retrieve data. PostgreSQL 10 [greatly built on this functionality](https://www.postgresql.org/docs/10/release-10.html#id-1.11.6.28.5.3.2) and added many more parallel operations.

What I found amazing about this functionality was its immediate impact upon upgrading. Where I was working at the time, I had our production OLTP workload running PostgreSQL 9.4. I began profiling our workload against a PostgreSQL 10 instance, and just with upgrading, I saw a 2.5x speedup over our **entire workload**. When I drilled into it further, I observed that the parallel query work was a big driver of this.

There have been a lot of enhancements since PostgreSQL 10 that have made similar "just upgrade to get big performance improvements", whether it is around indexing, concurrency management, or query performance optimizations. But I distinctly remember how impressed I was that I could update to PostgreSQL 10 and observe a site-wide performance improvement with minimal tuning.

## An easier (and better) way to partition

Longtime PostgreSQL users note that PostgreSQL had partitioning before PostgreSQL 10. You could create partitioned data using PostgreSQL "rules" system to help place it into different tables and query it. For a college internship, I actually used this system as part of segmenting and storing data for US TV channel schedules, though it did take a bit of work to set up and maintain the rules system.

PostgreSQL 10 lowered the complexity of partition management in PostgreSQL by letting users create partitions with simple SQL commands. While subsequent releases expanded the partitioning functionality and improved its performance, PostgreSQL 10 laid the groundwork for this essential feature. You can see the impact of PostgreSQL's partitioning support in the extension ecosystem too, as many extensions either provided more management functions or query mechanisms over data with lots of partitions.

## Publishers, subscribers: logical replication comes to PostgreSQL

When physical replication, or copying the binary data stored on disk between PostgreSQL servers, was introduced in PostgreSQL 9.0, one of the next asks was for logical replication. Logical replication has many uses, including moving data between two different writable PostgreSQL systems, or different major versions, or even into other databases! PostgreSQL 9.4 introduced the fundamental layer for logical replication in PostgreSQL: the ability to "decode" statements into a different format. This spawned various logical decoding output plugins, including [wal2json](https://github.com/eulerto/wal2json) that turned PostgreSQL writes into JSON that could be used in applications (e.g. [change data capture](https://en.wikipedia.org/wiki/Change_data_capture)).

PostgreSQL 10 added direct support for [logical replication](https://www.postgresql.org/docs/current/logical-replication.html), introducing the ability to create publications and subscribers between databases. Instead of relying on extensions or custom-built solutions, PostgreSQL users could run `CREATE PUBLICATION` and `CREATE SUBSCRIPTION` to set up logical replication.

Fast forward to today (PostgreSQL 15) and you see a whole array of products using PostgreSQL logical replication as part of their change data capture functionality, major version upgrades, or part of their replication strategy. 
While at Crunchy Data, I wrote up a blog post on how you can combine two features originally introduced in PostgreSQL 10 -- declarative partitioning and logical replication -- to create an [active-active federated PostgreSQL clusters](https://www.crunchydata.com/blog/active-active-postgres-federation-on-kubernetes).

As of this writing, there is still a bunch of ongoing work on logical replication in PostgreSQL, including making it possible to replicate more objects like sequences and DDL commands. But I remember the release of PostgreSQL 10 being a release that significantly helped users in how they could move data between systems.

## Reaching a quorum

During the PostgreSQL 10 release, I was less familiar with how high availability systems work. While I knew adding "quorum commit" for synchronous replication was a big feature, I did not fully understand why.

Here's how to think of quorum commit. Let's say you have 5 PostgreSQL instances: 1 primary and 4 standbys. You can set a rule that you consider a transaction to be committed if it is written to at least 3 instances. Prior to PostgreSQL 10, if you were using synchronous replication, a transaction would only be considered committed if it was written to **all** instances, not a subset of them.

Adding quorum commit advanced PostgreSQL's ability to be a part of high availability systems. It also made it easier to use synchronous replicas with PostgreSQL, reducing performance and availability burdens. If you're interested in how many high availability systems work, read up on the [Raft algorithm](https://raft.github.io/). The paper itself is not too difficult a read (though I did read through sections a few times to understand them), but it does show the significance of a quorum commit.

## Making password more secure

I'm an unabashed proponent of using the [SCRAM password authentication method](https://www.rfc-editor.org/rfc/rfc5802). Seriously, [I will talk to you for an hour (or longer if you let me) on why you should use SCRAM](https://www.slideshare.net/jkatz05/get-your-insecure-postgresql-passwords-to-scram). But why?

Before PostgreSQL 10, the primary way to use PostgreSQL authentication was with its own `md5` authentication method. The `md5` method has [some known flaws](https://www.postgresql.org/docs/current/auth-password.html), including that someone who has the stored `md5` and its corresponding username can log in as that username.

SCRAM changes that. The SCRAM method, in a nutshell, lets two parties verify that the other knows a password without exchanging the password. Even if an attacker is able to capture the "SCRAM secret" stored on a server, their options for learning the credential are either through a 1/ MITM attack (which can be mitigated through good TLS practices, i.e. `sslmode=verify-full`) or 2/ brute-forcing the password.

Using SCRAM required changes to [PostgreSQL drivers](https://wiki.postgresql.org/wiki/List_of_drivers). The community responded: you can see from that list that all the libpq and non-libpq based drivers added support for SCRAM. This allowed PostgreSQL to make SCRAM the default authentication method in PostgreSQL 14, and hopefully in the future we can remove `md5` altogether.

## Lost at the time, found later on

The above features were major highlights in the [PostgreSQL 10 release announcement](https://www.postgresql.org/about/news/postgresql-10-released-1786/). They withstood the "test of time" and continue to have impact on users of newer PostgreSQL releases.

But there were other PostgreSQL 10 features that also had significant impact though lacked fanfare. I'm sure I am going to miss some, but here are the ones that I noticed while re-reading the PostgreSQL 10 release notes.

PostgreSQL 10 introduced the ability to "push" joins and aggregates to PostgreSQL servers being accessed through the [`postgres_fdw`](https://www.postgresql.org/docs/current/postgres-fdw.html). Instead of having to pull all the data using joins and aggregates from remote PostgreSQL servers into a single-PostgreSQL server for processing, which is a potentially exhaustive operation, that work could be handled on remote servers and reduce the amount of data that needed to be transferred. This has had significant ramifications for federated or sharded workloads, as it decreased the burden on the primary instance.

PostgreSQL 10 also introduced "identity columns", a SQL standard way for specifying `serial` or an autoincrement. This means that:

```sql
CREATE TABLE xyz (
    id serial PRIMARY KEY
);
```

can be written as:

```sql
CREATE TABLE xyz (
    id int GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY
);
```

and be SQL compliant. My SQL code and examples now use the identity column syntax!

PostgreSQL 10 also added the [`amcheck`](https://www.postgresql.org/docs/current/amcheck.html) extension, which has become an essential DBA tool for finding data corruption. PostgreSQL 14 later introduced [`pg_amcheck`](https://www.postgresql.org/docs/15/app-pgamcheck.html) to provide a CLI for detecting data corruption.

Finally, PostgreSQL 10 added the ability to change TLS configuration using a "reload" instead of a "restart". I became a huge fan of this feature when helping to build a system that had automatic rotation of TLS certificates in a PostgreSQL server without forcing any user downtime.

## Conclusion: PostgreSQL 10 lives on

PostgreSQL has come a long way since the PostgreSQL 10 release five years ago. Given how transformative this release was, and how impactful it has remained over these past five years, I wanted to ensure we could give it an appropriate tribute for its final update.

It's also important to reflect how far PostgreSQL has come, even in the past five years, from both a functionality and adoption standpoint. I'm still amazed at all the conversations I have with people over what kinds of workloads they're running with PostgreSQL, what features they are using and enjoying, and all the different ways they are deploying PostgreSQL!

While there is always more work to do (e.g. see my comments on ongoing logical replication work), it's also good to step back at how far PostgreSQL has come, and celebrate the releases that helped move the project significantly forward.

...and while PostgreSQL 10 was an impactful release, if you are still using it in production, you should seriously consider upgrading 😉