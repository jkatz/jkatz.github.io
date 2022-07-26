+++
author = "Jonathan Katz"
title = "BEGIN ATOMIC: a better way to create functions in PostgreSQL 14"
date = "2022-07-26"
description = "The SQL-standard BEGIN ATOMIC syntax for functions parses them at creation and tracks dependencies!"
tags = [
    "postgres",
    "postgresql",
]
+++

Around this time of year, I am reading through the upcoming PostgreSQL release notes (hello [PostgreSQL 15](https://www.postgresql.org/docs/15/release-15.html)), reading [mailing lists](https://www.postgresql.org/list/), and talking to [PostgreSQL](https://www.postgresql.org) users to understand what are the impactful features. Many of these features will be highlighted in the [feature matrix](https://www.postgresql.org/about/featurematrix/) and release announcement (hello [PostgreSQL 14](https://www.postgresql.org/about/press/presskit14/)!).

However, sometimes I miss an impactful feature. It could be that we need to see how the feature is actually used post-release, or it could be that it was missed. I believe one such change is the introduction of the SQL-standard `BEGIN ATOMIC` syntax in PostgreSQL 14 that is used to create function and stored procedure bodies.

Let's see how functions we could create functions before PostgreSQL 14, the drawbacks to this method, and how going forward, `BEGIN ATOMIC` makes it easier and safer to manage functions!

## Before PostgreSQL 14: Creating Functions as Strings

PostgreSQL has supported the ability to create stored functions (or "user-defined functions") since POSTGRES 4.2 in 1994 (thanks [Bruce Momjian[(https://momjian.us/)] for the answer on this). When declaring the function body, you would write the code as a string (which is why you see the `$$` marks in functions). For example, here is a simple function to add two numbers:

```sql
CREATE FUNCTION add(int, int)
RETURNS int
LANGUAGE SQL
IMMUTABLE PARALLEL SAFE
AS $$
  SELECT $1 + $2;
$$;
```

If I want to have functions that calls another user-defined function, I can do so similarly to the above. For example, here is a function that uses the `add` function to add up a whole bunch of numbers:

```sql
CREATE FUNCTION test1_add_stuff(int, int, int, int)
RETURNS int
LANGUAGE SQL
IMMUTABLE PARALLEL SAFE
AS $$
  SELECT add(add($1, $2), add($3, $4));
$$;
```

After I create the two functions, if I try to run `test1_add_stuff`, it works:

```
SELECT test1_add_stuff(1,2,3,4);

 test1_add_stuff
-----------------
              10
```

What happens if I drop the `add` function?

```sql
DROP FUNCTION add(int, int);
```

It drops successfully:

```
DROP FUNCTION add(int, int);
DROP FUNCTION
```

And if I try calling `test1_add_stuff` again?

```
SELECT test1_add_stuff(1,2,3,4);
ERROR:  function add(integer, integer) does not exist
LINE 2:   SELECT add(add($1, $2), add($3, $4));
                     ^
HINT:  No function matches the given name and argument types. You might need to add explicit type casts.
QUERY:  
  SELECT add(add($1, $2), add($3, $4));

CONTEXT:  SQL function "test1_add_stuff" during startup
```

Well, this stinks. Using the pre-v14 style of creating custom functions in PostgreSQL lacks dependency tracking. So dropping one function could end up breaking other functions.

PostgreSQL does support [dependency tracking](https://www.postgresql.org/docs/current/ddl-depend.html). Prior to PostgreSQL 14, PostgreSQL could track dependencies in functions that involve attributes such as arguments or returns types, but it could not track dependencies within a function body.

This is where `BEGIN ATOMIC` function bodies changes things.

## BEGIN ATOMIC: a better way to create and manage PostgreSQL user-defined functions

I started looking at the `BEGIN ATOMIC` method of creating functions [thanks to a note from Morris de Oryx](https://www.postgresql.org/message-id/CAKqncciYKMZSNM0LZuCYoKsGgDBxE%3D%3DLEAAH0svqmajYhO1oxw%40mail.gmail.com) on the usefulness of this feature (and the fact it was [previously missing from the feature matrix](https://www.postgresql.org/about/featurematrix/detail/393/)). Morris succinctly pointed out two of the most impactful attributes of creating a PostgreSQL function with `BEGIN ATOMIC`:

* Because PostgreSQL parses the functions during creation, it can catch additional errors.
* PostgreSQL has improved dependency tracking of functions.

Let's use the above example again to see how `BEGIN ATOMIC` changes things.

### Dependency Tracking

Let's create the `add` function using the `BEGIN ATOMIC` syntax:

```sql
CREATE FUNCTION add(int, int)
RETURNS int
LANGUAGE SQL
IMMUTABLE PARALLEL SAFE
BEGIN ATOMIC;
  SELECT $1 + $2;
END;
```

Notice the difference: the function body is no longer represented as a string, but actual code statements. Let's now create a new function called `test2_add_stuff` that will use the `add` function:

```sql
CREATE FUNCTION test2_add_stuff(int, int, int, int)
RETURNS int
LANGUAGE SQL
IMMUTABLE PARALLEL SAFE
BEGIN ATOMIC;
  SELECT add(add($1, $2), add($3, $4));
END;
```

As expected, `test2_add_stuff` should work correctly:

```
SELECT test2_add_stuff(1,2,3,4);
 test2_add_stuff
-----------------
              10
```

What happens if we try to drop the `add` function now?

```
DROP FUNCTION add(int,int);
ERROR:  cannot drop function add(integer,integer) because other objects depend on it
DETAIL:  function test2_add_stuff(integer,integer,integer,integer) depends on function add(integer,integer)
HINT:  Use DROP ... CASCADE to drop the dependent objects too.
```

This is awesome: using `BEGIN ATOMIC` when for functions guards against dropping a function that is called by one or more other functions!

### Creation Time Parsing Checks

Let's look at one more example at how `BEGIN ATOMIC` helps us avoid errors.

Prior to PostgreSQL 14, there are some checks that do occur at function creation time. For example, let's create a function that calls a function that calls a function.

```sql
CREATE FUNCTION a(int)
RETURNS int
LANGUAGE SQL
IMMUTABLE PARALLEL SAFE
AS $$
  SELECT $1 * 3;
$$;

CREATE FUNCTION b(int)
RETURNS int
LANGUAGE SQL
IMMUTABLE PARALLEL SAFE
AS $$
  SELECT a($1) * 2;
$$;

CREATE FUNCTION c(int)
RETURNS int
LANGUAGE SQL
IMMUTABLE PARALLEL SAFE
AS $$
  SELECT b($1);
$$;
```

Calling `c` should work as expected:

```
SELECT c(1);
 c
---
 6
```

Let's drop both `a` and `c`:

```sql
DROP FUNCTION a(int);
DROP FUNCTION c(int);
```

What happens when we recreate `c`?

```
CREATE FUNCTION c(int)
RETURNS int
LANGUAGE SQL
IMMUTABLE PARALLEL SAFE
AS $$
  SELECT b($1);
$$;

CREATE FUNCTION
```

The function creation works, even though `a` is still missing. While functions created without `BEGIN ATOMIC` can check for dependencies within the body of the function itself, they do not check throughout the entire parse tree. We can see our call to `c` fail due to `a` still missing:

```
SELECT c(3);
ERROR:  function a(integer) does not exist
LINE 2:   SELECT a($1) * 2;
                 ^
HINT:  No function matches the given name and argument types. You might need to add explicit type casts.
QUERY:  
  SELECT a($1) * 2;

CONTEXT:  SQL function "b" during inlining
SQL function "c" during startup
```

Using `BEGIN ATOMIC` for creating all of these functions will prevent us from getting into this situation, as PostgreSQL will scan the parse tree to ensure all of these functions exist. Let's now recreate the functions to use `BEGIN ATOMIC` for their function bodies:

```sql
CREATE OR REPLACE FUNCTION a(int)
RETURNS int
LANGUAGE SQL
IMMUTABLE PARALLEL SAFE
BEGIN ATOMIC;
  SELECT $1 * 3;
END;

CREATE OR REPLACE FUNCTION b(int)
RETURNS int
LANGUAGE SQL
IMMUTABLE PARALLEL SAFE
BEGIN ATOMIC;
  SELECT a($1) * 2;
END;

CREATE OR REPLACE FUNCTION c(int)
RETURNS int
LANGUAGE SQL
IMMUTABLE PARALLEL SAFE
BEGIN ATOMIC;
  SELECT b($1);
END;
```

Now, try dropping `a` - PostgreSQL will prevent this from happening.

```
DROP FUNCTION a(int);
ERROR:  cannot drop function a(integer) because other objects depend on it
DETAIL:  function b(integer) depends on function a(integer)
function c(integer) depends on function b(integer)
HINT:  Use DROP ... CASCADE to drop the dependent objects too.
```

Great. But what happens if I do a `DROP FUNCTION ... CASCADE` on function `a`?

```
DROP FUNCTION a(int) CASCADE;
NOTICE:  drop cascades to 2 other objects
DETAIL:  drop cascades to function b(integer)
drop cascades to function c(integer)
DROP FUNCTION
```

PostgreSQL drops **all** functions that depend on `a`, including both `b` and `c`. While this is handy for cleaning up a test example, be careful when cascading drops on our production systems so you do not accidentally remove an important object!

## Conclusion

The `BEGIN ATOMIC` syntax for creating PostgreSQL functions should make managing user-defined functions less error prone, particularly the "accidental dropped function" case. Going forward I will definitely use this feature to manage my PostgreSQL stored functions.

PostgreSQL releases are packed with new features. In fact, PostgreSQL 14 has over 190 features listed in the release notes! This makes it possible to miss a feature that may make it much easier for you to build applications with PostgreSQL. I encourage you to read the [release notes](https://www.postgresql.org/docs/release/) to see if there is a feature in them that will help your PostgreSQL experience!
