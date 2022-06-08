+++
author = "Jonathan Katz"
title = "Notes on updating to PostgreSQL 14.3, 13.7, 12.11, 11.16, and 10.21"
date = "2022-06-07"
description = "This PostgreSQL update release has some issues that could impact you. Learn what they are and if you should update."
tags = [
    "postgres",
    "postgresql",
]
+++

On May 12, 2022, the PostgreSQL Global Development Group
[released its regular quarterly update](https://www.postgresql.org/about/news/postgresql-143-137-1211-1116-and-1021-released-2449/) for all of its supported versions (10-14) containing
bug fixes and a security fix for [CVE-2022-1552](https://www.postgresql.org/support/security/CVE-2022-1552/). Per its [versioning policy](https://www.postgresql.org/support/versioning/),
the PostgreSQL community advises that users run the
"[latest available minor release available for a major version](https://www.postgresql.org/support/versioning/)."
This is generally the correct approach: update releases make each major release
more stable, and the community makes a concerted effort to avoid introducing
breaking changes. You should *always* test each update release before releasing
it into your production environment.

However, the May 12, 2022 release had some changes that could have an impact on
your environment. Specifically, there are two bugs that you should be aware of
and consider how they may impact your PostgreSQL installations if you choose to
upgrade. You do need to weigh the decision to upgrade against incorporating the
fix for CVE-2022-1552 and the
[other bug fixes available in this release](https://www.postgresql.org/about/news/postgresql-143-137-1211-1116-and-1021-released-2449/).

The below explains what each issue is, what versions of PostgreSQL it effects,
the tradeoffs around upgrading and any remediations.

## `CREATE INDEX CONCURRENTLY` / `REINDEX CONCURRENTLY` on PostgreSQL 14

Vacuuming is an
[essential part of PostgreSQL maintenance](https://www.postgresql.org/docs/current/routine-vacuuming.html)
that performs actions such as reclaiming disk space from updated and deleted
rows. The [GA release of PostgreSQL 14](https://www.postgresql.org/about/news/postgresql-14-released-2318/)
(14.0) introduced an
[optimization for `VACUUM` when `CREATE INDEX CONCURRENTLY` and `REINDEX CONCURRENTLY`](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d9d076222f5b) were
running at the same time.

There were a few
[bug reports of index corruption in PostgreSQL 14](https://www.postgresql.org/message-id/17485-396609c6925b982d%40postgresql.org) and shortly after the PostgreSQL 14.3
release, several members of the PostgreSQL community were able to consistently
reproduce the issue. The optimization described in the above paragraph could
lead to cases of silent index corruption when indexes are built with
[`CREATE INDEX CONCURRENTLY`](https://www.postgresql.org/docs/current/sql-createindex.html)
or [`REINDEX CONCURRENTLY`](https://www.postgresql.org/docs/current/sql-reindex.html).
The issue was present since PostgreSQL 14.0: it does not affect any of the other
supported versions of PostgreSQL (i.e.. PostgreSQL 10 - 13).

After some discussion, the PostgreSQL community decided to
[revert the `VACUUM` optimization](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) for
the time being until a solution that does not contain the risk of silent index
corruption can be implemented. The community has discussed how to best detect
this corruption issue using
[`pg_amcheck`](https://www.postgresql.org/docs/current/app-pgamcheck.html),
specifically with the `--heapallindexed` flag. This will take an
[`ACCESS SHARE`](https://www.postgresql.org/docs/current/explicit-locking.html)
lock on each table, but it will not block `VACUUM` and can be run on a standby.
Note that `pg_amcheck` can only detect the corruption issue on B-tree indexes,
and the community is unsure if it can detect all cases of corruption.

If you have run `CREATE INDEX CONCURRENTLY` or `REINDEX CONCURRENTLY` using
PostgreSQL 14 and need an immediate fix, you can fix your indexes by running
either running `REINDEX` or dropping and recreating the index **without** the
`CONCURRENTLY` option. Once PostgreSQL 14.4 is available, you can use
`CONCURRENTLY`. The
[`reindexdb`](https://www.postgresql.org/docs/current/app-reindexdb.html)
command-line utility can help with the process as the `--jobs` flag lets you
execute multiple `REINDEX` operations at the same time across the entire
database.

## A brief explanation of CVE-2022-1552

To understand the other issue, its first necessary to understand the impact of
CVE-2022-1552. As described,
[CVE-2022-1552](https://www.postgresql.org/support/security/CVE-2022-1552/)
closes a vulnerability where an unprivileged user can craft malicious SQL and
use certain commands (Autovacuum, `REINDEX`, `CREATE INDEX`,
`REFRESH MATERIALIZED VIEW`, `CLUSTER`, and `pg_amcheck`) to escalate to become
a PostgreSQL superuser. A malicious user still needs to have an account with the
PostgreSQL system to perform this exploit.

Systems that have unprivileged PostgreSQL users that have risk of SQL injection
(e.g. web applications) or multi-tenant systems may be particularly affected by
this CVE. While upgrading to 14.3 et al. fixes the issue, the community provides
guidance that if you cannot take this upgrade, you can still remediate the issue
by disabling autovacuum (with a warning on performance tradeoffs), not running
the above commands, and to not perform restores using the output from
`pg_dump`.

This issue affects all supported versions of PostgreSQL (10-14) but, as the CVE
notes, the issue is quite old and is not patched in unsupported versions (e.g.
9.6 and older).

## Creating an expression index using an operator class from a different schema

Shortly after the May 12, 2022 update release, there was a report on the
PostgreSQL bugs mailing list where a user could not create an
[expression index](https://www.postgresql.org/docs/current/indexes-expressional.html)
as an unprivileged user when
[using an operator class from a different schema that was created by a different user](https://www.postgresql.org/message-id/flat/20220531152855.GA2236210%40nathanxps13#cf62ad5183084f9da0f458c446bb995d).
Specifically, the case used the the
[`gist_trgm_ops`](https://www.postgresql.org/docs/current/pgtrgm.html#id-1.11.7.42.8)
operator class from the `pg_trgm` index to allow text similarity operators to be
indexable. While the issue was first reported based on the output of
[`pg_dump`](https://www.postgresql.org/docs/current/app-pgdump.html), this can
be reproduced in a straightforward way using a
[few commands](https://www.postgresql.org/message-id/20220526055047.GA3153526%40rfd.leadboat.com).
There may be a few other cases where this issue may occur with other expression
indexes, but the above situation has been consistently reproduced.

The fix for [CVE-2022-1552](https://www.postgresql.org/support/security/CVE-2022-1552/)
introduced this issue and only affects PostgreSQL 14.3, 13.7, 12.11, 11.16, and
10.21. As of the writing of this blog post, there is no fix available. For a
remediation, you can add the operator classes to the same schema where you are
creating the index. Ensure that any changes comply with the security posture
you are enforcing for your database.

## Should I upgrade to 14.3, 13.7, 12.11, 11.16, 10.21?

If your database has a single-user and is the PostgreSQL superuser, you should
be able to upgrade without issues. Note that if you are on PostgreSQL 14, you
are still affected by the `CREATE INDEX CONCURRENTLY` / `REINDEX CONCURRENTLY`
issue and you should not use those commands until the fix is in place.

For all other cases, you will need to weigh the tradeoffs of the above issues.
If you are on PostgreSQL 14, you will be affected by the
`CREATE INDEX CONCURRENTLY` / `REINDEX CONCURRENTLY` issue regardless if you
take this update. You should be aware of this issue and not run those commands.
If you have, you may need to reindex. The index corruption issue should not
prevent you from updating from PostgreSQL 14.3.

If you are running a system that contains an unprivileged PostgreSQL user, you
will need to weigh the tradeoff of incorporating the fix for CVE-2022-1552
versus potential breakage with your application. The bug most likely shows
itself when performing "schema migrations" or restoring from a `pg_dump`, but is
limited to if you are using any operator classes (e.g. for indexing) and how you
have structured your schemas. There may be some other unreported cases
that are affected by this issue, so be sure you test restoring your schema from
a `pg_dump` (e.g. `pg_dump --schema-only`).

As the CVE mentions, you can still remediate the vulnerability without
upgrading, but there are performance and potentially stability risks with these
steps. Vacuuming is
[an essential part of PostgreSQL maintenance](https://www.postgresql.org/docs/current/routine-vacuuming.html)
and if you do not use it, your system can end up slowing down. In more extreme
cases, a system can hit
[transaction ID wraparound](https://www.postgresql.org/docs/current/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND),
which will put a PostgreSQL database into an unusable state.

If you do not believe your application is affected by the issue with creating
indexes, you should consider upgrading. The fix for CVE-2022-1552 is much easier
to apply than the remediation steps. The remediation carries a risk of
performance degradation and instability for your system, so if you believe it is
safe to take the upgrade, you should do so.

The PostgreSQL community guidance to
[run the latest release of a major version](https://www.postgresql.org/support/versioning/)
is a good best practice to follow. You should read through the
[release announcement](https://www.postgresql.org/about/news/postgresql-143-137-1211-1116-and-1021-released-2449/) and [release notes](https://www.postgresql.org/docs/release/)
to understand what fixes are available, and test your applications against the
update releases before deploying them to production.
