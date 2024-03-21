+++
author = "Jonathan Katz"
title = "Will PostgreSQL ever change its license?"
date = "2024-03-20"
description = "Nope."
tags = [
    "postgres",
    "postgresql"
]
+++

(Disclosure: I'm on the [PostgreSQL Core Team](https://www.postgresql.org/developer/core/), but what's written in this post are my personal views and not official project statements...unless I link to something that's an official project statement ;)

I was very sad to learn today that the [Redis project will no longer be released under an open source license](https://redis.com/blog/redis-adopts-dual-source-available-licensing/). Sad for two reasons: as a longtime Redis user and pretty early adopter, and as an open source contributor. I'll preface that I'm empathetic to the challenges of building businesses around open source, having been on multiple sides of this equation. I'm also cognizant of the downstream effects of these changes that can completely flip how a user adopts and uses a piece of technology.

Whenever there's a shakeup in open source licensing, particularly amongst databases and related systems (MySQL => Sun => Oracle being the one that first springs to mind), I'll hear the question "Will PostgreSQL ever change its license?"

The PostgreSQL website [has an answer](https://www.postgresql.org/about/licence/):

> Will PostgreSQL ever be released under a different license?
> The PostgreSQL Global Development Group remains committed to making PostgreSQL available as free and open > source software in perpetuity. There are no plans to change the PostgreSQL License or release PostgreSQL under a different license.

(Disclosure: I did help write the above paragraph).

[The PostgreSQL Licence](https://www.postgresql.org/about/licence/) (aka "License" -- [Dave Page](https://pgsnake.blogspot.com/) and I have fun going back and forth on this) is an [Open Source Initiative (OSI) recognized license](https://opensource.org/license/postgresql), and has a very permissive model. In terms of which license it's most similar to, I defer to this email that [Tom Lane wrote in 2009](https://www.postgresql.org/message-id/1776.1256525282@sss.pgh.pa.us).

That said, there are a few reasons why PostgreSQL won't change it's license:

* It's "[The PostgreSQL Licence](https://www.postgresql.org/about/licence/)" -- why change license when you have it named after the project?
* The PostgreSQL Project began as a collaborative open source effort and is set up to prevent a single entity to take control. This carries through in the project's ethos almost 30 years later, and is even codified throughout the [project policies](https://www.postgresql.org/about/policies/).
* [Dave Page explicitly said so in this email](https://www.postgresql.org/message-id/937d27e10910260840s1d28aab2o799f2c58d14dfb1e%40mail.gmail.com) :)

The question then becomes - is there a reason that PostgreSQL would change its license? Typically these changes happen as part of a business decision - but it seems that business around PostgreSQL is as robust as its feature set. Ruohang Feng (Vonng) recently [wrote a blog post](https://medium.com/@fengruohang/postgres-is-eating-the-database-world-157c204dcfc4) that highlighted just a slice of the PostgreSQL software and business ecosystem that's been built around it, which is only possible through the PostgreSQL Licence. I say "just a slice" because there's even more, both historically and current, projects and business that are built up around some portion of the PostgreSQL codebase. While many of these projects may be released under different licenses or be closed source, they have helped drive, both directly and indirectly, PostgreSQL adoption, and have helped make the PostgreSQL protocol ubiquitous.

But the biggest reason why PostgreSQL would not change its license is the disservice it would do to all PostgreSQL users. It takes a long time to build trust in a technology that is often used for the most critical part of an application: storage and retrieval of data. [PostgreSQL has earned a strong reputation for its proven architecture, reliability, data integrity, robust feature set, extensibility, and the dedication of the open source community behind the software to consistently deliver performant and innovative solutions](https://www.postgresql.org/about/). Changing the license of PostgreSQL would shatter all of the goodwill the project has built up through the past (nearly) 30 years.

While there are definitely parts of the PostgreSQL project that are imperfect (and I certainly contribute to those imperfections), the PostgreSQL Licence is a true gift to the PostgreSQL community and open source in general that we'll continue to cherish and help keep PostgreSQL truly free and open source. After all, it says [so on the website](https://www.postgresql.org/about/licence/) ;)
