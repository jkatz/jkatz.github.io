+++
author = "Jonathan Katz"
title = "\"Read-Only\" Mode for PostgreSQL"
date = "2022-01-23"
description = "Learn how to put your Postgres database into \"read-only\" mode."
tags = [
    "postgres",
    "postgreql",
]
+++

Typically when discussing having "read-only" connections to a [PostgreSQL](https://www.postgresql.org) database, it is in the context of connecting to a [replica](https://www.postgresql.org/docs/current/high-availability.html).

There are a variety of methods available to route connections with known read-only queries (i.e. queries with `SELECT` statements...that are not calling [`VOLATILE` functions](https://www.postgresql.org/docs/current/xfunc-volatility.html) that modify data). This includes connection proxy software like [Pgpool-II](https://www.pgpool.net/) or framework mechanisms such as [Django's database router](https://docs.djangoproject.com/en/4.0/topics/db/multi-db/#automatic-database-routing).

However, there are situations where you might need to force read-only connections to your primary (read-write) Postgres instance. Some examples include putting your application into a degraded state to perform a database move or upgrade, or allowing an administrator to inspect a system that may be accumulating [write-ahead logs](https://www.postgresql.org/docs/current/wal-intro.html) that track all changes to the system.

PostgreSQL has a configuration parameter call [`default_transaction_read_only`](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-DEFAULT-TRANSACTION-READ-ONLY). Setting `default_transaction_read_only` globally to `on` forces all connections to disallow writes to the database. `default_transaction_read_only` is a reloadable parameter, so you do not need to restart your Postgres instance to use it.

Here is a quick example of how `default_transaction_read_only` works. First, ensure your system does not have `default_transaction_read_only` set:

```sql
postgres=# SHOW default_transaction_read_only ;

 default_transaction_read_only
-------------------------------
 off
(1 row)

postgres=# CREATE TABLE IF NOT EXISTS abc (id int); INSERT INTO abc VALUES (1) RETURNING id;
CREATE TABLE

 id
----
  1
(1 row)

INSERT 0 1
```

This works as expected: we're able to create a table and insert data. Now let's put the system into `default_transaction_read_only` mode (note that I am running this on [PostgreSQL 14](https://www.postgresql.org/about/news/postgresql-14-released-2318/))

```sql
ALTER SYSTEM SET default_transaction_read_only TO on;
SELECT pg_reload_conf();
SHOW default_transaction_read_only;
```

Ensure that `default_transaction_read_only` is enabled:

```sql
postgres=# SHOW default_transaction_read_only;

 default_transaction_read_only
-------------------------------
 on
(1 row)
```

Now verify that writes are disallowed:

```sql
postgres=# INSERT INTO abc VALUES (2) RETURNING id;
ERROR:  cannot execute INSERT in a read-only transaction
```

Excellent!

Note that `default_transaction_read_only` is not a panacea: there are some [caveats](https://en.wikipedia.org/wiki/Caveat_emptor) that you should be aware of.

First, `default_transaction_read_only` can be overriden in a session, even if the value is set database-wide. For example:

```sql
postgres=# SHOW default_transaction_read_only ;
 default_transaction_read_only
-------------------------------
 on
(1 row)

postgres=# SET default_transaction_read_only TO off;
SET

postgres=# INSERT INTO abc VALUES (2) RETURNING id;

 id
----
  2
(1 row)

INSERT 0 1
```

Second, when utilizing `default_transaction_read_only` with an application, you must also ensure your app can be configured to send only read queries to the database, ensuring a smooth user experience.

That said, if you have a situation where you need to put a PostgreSQL primary instance into a "read-only" mode temporarily, you can use `default_transaction_read_only` to prevent write transactions.
