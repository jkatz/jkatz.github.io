+++
author = "Jonathan Katz"
title = "Scalar and binary quantization for pgvector vector search and storage"
date = "2024-04-09"
description = "Quantization can reduce vector sizes, but how does it impact query performance and quality?"
tags = [
    "postgres",
    "postgresql",
    "pgvector"
]
+++

While many AI/ML embedding models generate vectors that provide large amounts of information by using high dimensionality, this can come at the cost of using more memory for searches and more overall storage. Both of these can have an impact on the cost and performance of a system that's storing vectors, including when using [PostgreSQL](https://www.postgresql.org/) with the [pgvector](https://github.com/pgvector/pgvector/) for these use cases.

When I [talk](https://jkatz05.com/talks/) about [vector search in PostgreSQL](https://www.youtube.com/watch?v=PhIC4JlYg7A&ab_channel=AWSEvents), I have a slide that I like to call "no shortcuts without tradeoffs" that calls out the different challenges around searching vectors in a database. I first like to highlight that a 1,536 dimensional vector with 4-byte floating point (`fp32`) dimensions requires 6KiB of storage. That may not seem like a lot, but storing 1,000,000 of these vectors in a PostgreSQL database requires 5.7GB of storage without any indexing. And again, that may not seem like a lot, but a single database row is closer to 600B of data (and typically even less than that) than 6KB.

The next point I talk about is compression: unfortunately, you just can't compress a bunch of seemingly random floating point numbers. However, there are techniques to reduce the amount of information that you can store, from [principal component analysis](https://en.wikipedia.org/wiki/Principal_component_analysis) (aka [PCA](https://en.wikipedia.org/wiki/Principal_component_analysis)), which can reduce the overall dimensionality, to quantization techniques that can reduce the size or overall number of dimensions of a vector. These can reduce your overall storage and memory requirements -- and perhaps even improve performance in certain areas, but recall that I mentioned there are no shortcuts without tradeoffs. To use quantization for vector searches, we'll have to give something up, whether it's in the overall relevancy of our searches ([recall](https://en.wikipedia.org/wiki/Precision_and_recall)) or in some area of performance in our system.

There are three common quantization techniques around vector databases:

* **Scalar quantization**, which reduces the overall size of the dimensions to a smaller data type (e.g. a 4-byte float to a 2-byte float or 1-byte integer).
* **Binary quantization**, which is a subset of scalar quantization that reduces the dimensions to a single bit (e.g. `> 0` to `1`, `<=0` to `0`).
* **Product quantization**, which uses a clustering technique to effectively remaps the original vector to a vector with smaller dimensionality and indexes that (e.g. reduce a vector from 128-dim to 8-dim).

There is already a lot of content available that does better exploration and explanation of these techniques than I will; instead I'll focus on how to use 2 of these 3 techniques in the context of pgvector.

The [upcoming pgvector 0.7.0](https://github.com/pgvector/pgvector/issues/508) release is planning to include functionality that allows for you to use both scalar and binary quantization as part of your indexing strategy. Specifically, pgvector 0.7.0 adds support for indexing 2-byte floats (`halfvec`) and bit/binary vectors (using the PostgreSQL [`bit`](https://www.postgresql.org/docs/current/datatype-bit.html) data type). The 0.7.0 will also add support for sparse vectors via `sparsevec`, but that will be for a future blog post.

**ASIDE**: These quantization techniques also let you index vectors that are **larger than 2,000 dimensions**, which has been a major request for pgvector. `halfvec` can store up to 4,000 dimensions, `bit` can store up to 64,000 dimensions, and `sparsevec` can store up to 1,000 nonzero elements (which means it can store vectors with very high dimensionality).

Using existing pgvector functionality, such as [HNSW indexing](/post/postgres/pgvector-hnsw-performance/), along with PostgreSQL features like [expression indexes](https://www.postgresql.org/docs/current/indexes-expressional.html), we can explore how scalar and binary quantization can help reduce both space and memory consumption -- letting us scale vector workloads even further on PostgreSQL, understand what performance gains and tradeoffs we must make, how embedding models can impact results, and determine when it makes sense to use quantization.

## Test setup and system configuration

Before diving in, let's do a quick review of how we want to test these techniques. I [previously provided guidance on testing approximate nearest neighbor (ANN)](/post/postgres/pgvector-hnsw-performance/) algorithms over vector indexes; below is a quick summary of what we will look at over the course of this testing:

* `Index size`: This is a key metric with a quantization test, as we should be building indexes that are smaller than a full (or "flat") vector index. The interesting data is "how much smaller," and the impacts to the other metrics we'll review.
* `Index build time`: How does quantization impact overall build time? If the build time is longer, do we gain enough in index size / recall / query performance that it's OK to make that tradeoff? If the build time is shorter, do we lose anything in recall / query performance such that we should avoid quantization?
* `Queries per second` (QPS): How does quantization impact how many queries we can execute per second, or our overall throughput?
* `Latency` (p99): The time it takes to return a single result, but using a measurement that represents the 99th percentile ("very slow") timing. This serves as an "upper bound" on our latency times.
* `Recall`: A measurement of the relevancy of our results - what percentage of the expected results do we return?

For the test itself, I used a `r7gd.16xlarge` and stored the data on the local NVMe to rule out the impact of network latency. For these tests, I only ran on the ARM-based architecture of the Graviton3; as pgvector does leverage SIMD, exact timings may vary on different architectures.

I used PostgreSQL 16.2, with the following (relevant to this test) non-default configuration (in alphabetical order - thanks `\dconfig`!):

* `checkpoint_timeout`: 2h
* `effective_cache_size`: 256GB
* `jit`: off
* `maintenance_work_mem`: 64GB
* `max_parallel_maintenance_workers`: 63
* `max_parallel_workers`: 64
* `max_parallel_workers_per_gather`: 64
* `max_wal_size`: 20GB
* `max_worker_processes`: 128
* `shared_buffers`: 128GB
* `wal_compression`: zstd
* `work_mem`: 64MB

For testing, I used [ANN Benchmarks](https://github.com/erikbern/ann-benchmarks) with some modifications to the pgvector module to work with scalar and binary quantization. The modifications were purely to support scalar/binary quantization and would not make a material difference in performance values. I used the following datasets, though I won't necessarily show the data from all of them:

* mnist-784-euclidean (60K, 784-dim)
* sift-128-euclidean (1M, 128-dim)
* gist-960-euclidean (1M, 960-dim)
* dbpedia-openai-1000k-angular (1M, 1536-dim)
* glove-25-angular (1.1M, 25-dim)
* glove-100-angular (1.1M, 100-dim)

For this blog post, I'll focus on results from `sift-128-euclidean`, `gist-960-euclidean`, and `dbpedia-openai-1000k-angular`.

For the tests, I used HNSW indexing and used the following values:

* I fixed `m` at `16`, but ran tests with `ef_construction` at 32, 64, 128, 256, 512
* For each test, I varied `ef_search` with 10, 20, 40, 80, 120, 200, 400, 800

Finally, for each test, I maintained the original table structure (i.e. storing the original / flat vector in the table) while quantizing the indexed vector.

We'll first look individually at scalar and binary quantization with pgvector, followed by a summarization of findings and some recommendations.

## Scalar quantization with 2-byte (fp16) floats (`halfvec`)

Scalar quantization is often the simplest technique to use to shrink vector index storage, as it's just a matter of converting dimensions to a smaller data type (e.g. a 4-byte float to a 2-byte float). In some ways, quantize to a 2-byte float makes sense: when performing a distance operation, the biggest "differentiator" between two dimensions is in the more significant bits of data. If we marginally reduce the information to only care about those bits, we shouldn't see much difference in recall.

The first thing needed for this test is to add support for 2-byte float scalar quantization when building the HNSW index. As mentioned, we can do this in PostgreSQL using an expression index. Below is an example of a table and HNSW index using cosine distance that uses scalar quantization to store a 3,072 dimensional vector, such as the ones found in the `text-embedding-3-large` model:

```sql
CREATE TABLE documents (
    id bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    content text,
    embedding vector(3072)
);

CREATE INDEX ON documents
    USING hnsw ((embedding::halfvec(3072)) halfvec_cosine_ops);
```

Note that `hnsw ((embedding::halfvec(3072)) halfvec_cosine_ops)` contains the expression (`(embedding::halfvec(3072))`) that casts the full, 4-byte vector to become a 2-byte vector. Also note that we need to use a different operator class for the `halfvec` data type.

For completeness, if you want to use scalar quantization with Euclidean / L2 distance, you'd use the following query:

```sql
CREATE INDEX ON documents
    USING hnsw ((embedding::halfvec(3072)) halfvec_l2_ops);
```

A query that would use scalar quantization would look similar to this:

```sql
SELECT id
FROM documents
ORDER BY embedding::halfvec(3072) <=> $1::halfvec(3072)
LIMIT 10;
```

where `$1` represented a parameter that's a full vector being quantized to a 2-byte float.

Now, let's see how scalar quantization impacts vector search in pgvector. We'll first see the impact on index build times. In the table below, `vector` represents the index with no quantization, whereas `halfvec` contains the quantized vector. We'll only show the `ef_construction` value, as `m` is fixed to `16`:

### `sift-128-euclidean`

| `ef_construction` | `vector` size (MB) | `halfvec` size (MB) | Space reduction | `vector` build-time (s) | `halfvec` build-time (s) | Speedup  |
| ----------------- | ------------------ | ------------------- | ----- | ----------------------- | ------------------------ | ----- |
| 32                | 793                | 544                 | 1.46x | 33                      | 31                       | 1.06x |
| 64                | 782                | 538                 | 1.45x | 37                      | 34                       | 1.09x |
| 128               | 782                | 538                 | 1.45x | 44                      | 40                       | 1.10x |
| 256               | 782                | 538                 | 1.45x | 58                      | 51                       | 1.14x |
| 512               | 782                | 536                 | 1.46x | 88                      | 74                       | 1.19x |

### `gist-960-euclidean`

| `ef_construction` | `vector` size (MB) | `halfvec` size (MB) | Space reduction | `vector` build-time (s) | `halfvec` build-time (s) | Speedup  |
| ----------------- | ------------------ | ------------------- | ----- | ----------------------- | ------------------------ | ----- |
| 32                | 7,811              | 2,603               | 3.00x | 145                     | 68                       | 2.13x |
| 64                | 7,684              | 2,561               | 3.00x | 161                     | 77                       | 2.09x |
| 128               | 7,680              | 2,560               | 3.00x | 190                     | 101                      | 1.88x |
| 256               | 7,678              | 2,559               | 3.00x | 247                     | 147                      | 1.68x |
| 512               | 7,678              | 2,559               | 3.00x | 349                     | 229                      | 1.52x |

### `dbpedia-openai-1000k-angular`

| `ef_construction` | `vector` size (MB) | `halfvec` size (MB) | Space reduction | `vector` build-time (s) | `halfvec` build-time (s) | Speedup  |
| ----------------- | ------------------ | ------------------- | ----- | ----------------------- | ------------------------ | ----- |
| 32                | 7,734              | 3,867               | 2.00x | 244                     | 77                       | 3.17x |
| 64                | 7,734              | 3,867               | 2.00x | 264                     | 90                       | 2.93x |
| 128               | 7,734              | 3,867               | 2.00x | 301                     | 115                      | 2.62x |
| 256               | 7,734              | 3,867               | 2.00x | 377                     | 163                      | 2.31x |
| 512               | 7,734              | 3,867               | 2.00x | 508                     | 254                      | 2.00x |

The above shows that using `halfvec` to index a vector has both space savings and index build time benefits for vectors with larger dimensions. Without quantization, `gist-960-euclidean` (960-dim) and `dbpedia-openai-1000k-angular` (1536-dim) vectors can respectively only fit 2 and 1 vectors on an index page. Quantizing to a `halfvec` allows to store up to respectively store 4 and 2 on a page, which lines up with th space saving numbers we see. With less pages required, it also makes sense that we'd see a build speedup, as pgvector doesn't need to look up as many pages when placing a new vector into the index. The `sift-128-euclidean` (128-dim) tests did showed space savings, the overall changes in build times were minimal, though importantly they did not show regression.

It seems like this means we should start using 2-byte float scalar quantization, right? We can't make that determination just yet -- we need to understand the impact of these changes on query performance and recall. for the purposes of this blog, we'll look at the query performance values at `hnsw.ef_search` set to `10`, `40`, `200`, and `800`. 10 is the minimum value we can set to for this test (given the `LIMIT 10`) and will give us a lower bound. 40 is the `hnsw.ef_search` default for pgvector. At 200, we'd expect to see high recall, and at 800 we should be close to converging towards perfect recall (if we haven't done so already).

Visualizing these results is a bit challenging, as there are several variables to consider (dataset, quantization, `ef_construction`, `ef_search`) along with several outputs (QPS, p99 latency, recall). For simplicity, we'll consider the data at `ef_construction=256`, as that is the build value I recommend for folks to use for index builds when using the parallel build functionality. You can see the results below:

### `sift-128-euclidean` @ `ef_construction=256`

| `hnsw.ef_search` | `vector` recall | `halfvec` recall | `vector` QPS | `halfvec` QPS | `vector` p99 latency (ms) | `halfvec` p99 latency (ms) |
| ---------------- | --------------- | ---------------- |  ----------- | ------------- | ------------------------- | ------------- |
| 10               | 77.7%           | 77.5%            | 2,100        | 2,161         | 0.71                      | 0.68          |
| 40               | 95.4%           | 95.4%            | 1,020        | 1,097         | 1.19                      | 1.20          |
| 200              | 99.8%           | 99.8%            | 268          | 293           | 4.60                      | 4.38          |
| 800              | 100.0%          | 100.0%           | 84           | 91            | 15.07                     | 14.80         |

### `gist-960-euclidean` @ `ef_construction=256`

| `hnsw.ef_search` | `vector` recall | `halfvec` recall | `vector` QPS | `halfvec` QPS | `vector` p99 latency (ms) | `halfvec` p99 latency (ms) |
| ---------------- | --------------- | ---------------- |  ----------- | ------------- | ------------------------- | ------------- |
| 10               | 50.4%           | 50.1%            | 1,114        | 1,207         | 1.36                      | 1.21          |
| 40               | 78.0%           | 78.1%            | 513          | 579           | 2.64                      | 2.30          |
| 200              | 96.0%           | 96.0%            | 135          | 156           | 8.70                      | 7.49          |
| 800              | 99.6%           | 99.6%            | 41           | 47            | 29.20                     | 25.44         |


### `dbpedia-openai-1000k-angular` @ `ef_construction=256`

| `hnsw.ef_search` | `vector` recall | `halfvec` recall | `vector` QPS | `halfvec` QPS | `vector` p99 latency (ms) | `halfvec` p99 latency (ms) |
| ---------------- | --------------- | ---------------- |  ----------- | ------------- | ------------------------- | ------------- |
| 10               | 85.1%           | 85.2%            | 1,162        | 1,163         | 1.40                      | 1.21          |
| 40               | 96.8%           | 96.8%            | 567          | 578           | 2.70                      | 2.61          |
| 200              | 99.6%           | 99.6%            | 156          | 163           | 9.01                      | 8.59          |
| 800              | 99.9%           | 99.9%            | 48           | 51            | 30.50                     | 28.49         |

These are very good results for 2-byte float quantization with `halfvec`. In addition to the improved build performance, we see that (at least at `ef_construction=256`) the recall values between `vector` and `halfvec` are nearly idenical, and query performance is identical or slightly better (particularly with latency) with `halfvec`.

Across all the tests I ran with the ANN Benchmark datasets listed above, I saw similar results. This makes a strong case that moving forward, you should first attempt to use HNSW with 2-byte float quantization, as you'll save space and improve index build times while maintaining comparable performance to storing the full vector! If you're not seeing the recall that you want, you can increase `m`/`ef_construction`, or fall back to using the full 4-byte float vector in your index.

Let's now explore binary quantization.

## Binary quantization

Binary quantization is a more extreme quantization technique: it can reduce the full value of the dimension of a vector to a single bit of information. Specifically, binary quantization will reduce any positive value to `1`, and any zero or negative value to `0`. This can certainly lead to a huge reduction in memory/storage utilization, and likely query performance, as you're storing much more data on a single page. However, this can have a notable impact on recall, as we'll see in the experiments below.

I'm going to spoil one of the experiments: using binary quantization on its own can lead to poor recall if there's not enough bit-diversity in your vectors. Typically, you'll see more diversity amongst vectors with higher dimensionality, as these vectors are more likely to have varied bits. However, you can improve recall by "reranking," i.e. you use binary quantization to narrow down your set of vectors, and then you reorder the reduced set of vectors by using the original (flat) vectors stored in the table.

First, let's see how we can set up a table to support binary quantization. We'll first look at the results of building a HNSW index that stores binary quantized vectors, and then compare results between using binary quantization with and without re-ranking.

Recall that pgvector ultimately gets support for binary quantization through supporting vector operations through the PostgreSQL `bit` type. The pgvector 0.7.0 release provides two different bitwise distance functions: [Jaccard](https://en.wikipedia.org/wiki/Jaccard_index) (`<%>`) and [Hamming distance](https://en.wikipedia.org/wiki/Hamming_distance) (`<~>`). For these tests, I chose the Hamming distance function due to its performance characteristics, though I plan/hope to go into a deeper dive on these two options in a future blog post. 

Below is an example of how you can create a table and index that supports binary quantization and Hamming distance:

```sql
CREATE TABLE documents (
    id bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    content text,
    embedding vector(3072)
);

CREATE INDEX ON documents
    USING hnsw ((quantize_binary(embedding)::bit(3072)) bit_hamming_ops);
```

Again, notice we use a PostgreSQL expression index to quantize the embedding, and explicitly cast the bit dimensionality. That final cast is important: `quantize_binary` just returns a bit string, and pgvector index building needs to detect and explicit dimension.

The following query is how to search the index using binary quantization:

```sql
SELECT id
FROM documents
ORDER BY quantize_binary(embedding)::bit(3072) <~> quantize_binary($1)
LIMIT 10;
```

where $1 represents a parameterized value that is a  `vector` type.

As mentioned earlier, we'll also run an experiment that involves re-ranking the results before returning them. The following query does exactly that; note that the subquery has a higher limit, but it will ultimately be governed by the value of `hnsw.ef_search` (you can always set the `LIMIT` to be the value of `hnsw.ef_search`).

```sql
SELECT i.id
FROM (
    SELECT id, embedding <=> $1 AS distance
    FROM items
    ORDER BY quantize_binary(embedding)::bit(3072) <~> quantize_binary($1)
    LIMIT 800 -- bound by hnsw.ef_search
) i
ORDER BY i.distance
LIMIT 10;
```

where $1 represents a parameterized value that is a  `vector` type. Note that in the SELECT list of the subqery, there is the `embedding <=> $1 AS distance` expression: this performs the distance calculation that is ultimately used in the rerank. This example uses cosine distance, but you can choose your preferred distance metric.

Now, let's see how binary quantization with and without reanking impacts vector search in pgvector. As before, we'll first see the impact on index build times. In the table below, `vector` represents the index with no quantization, whereas `bit` contains the binary quantized vector. Note that reranking only impacts the query itself, so we don't have to show multiple sets of build times. We'll only show the `ef_construction` value, as `m` is fixed to `16`:

### `sift-128-euclidean`

| `ef_construction` | `vector` size (MB) | `bit` size (MB) | Space reduction | `vector` build-time (s) | `bit` build-time (s) | Speedup  |
| ----------------- | ------------------ | ------------------- | ----- | ----------------------- | ------------------------ | ----- |
| 32                | 793                | 304                 | 2.61x | 33                      | 28                       | 1.18x |
| 64                | 782                | 298                 | 2.62x | 37                      | 31                       | 1.19x |
| 128               | 782                | 298                 | 2.62x | 44                      | 37                       | 1.19x |
| 256               | 782                | 298                 | 2.62x | 58                      | 46                       | 1.26x |
| 512               | 782                | 298                 | 2.62x | 88                      | 67                       | 1.31x |

### `gist-960-euclidean`

| `ef_construction` | `vector` size (MB) | `bit` size (MB) | Space reduction | `vector` build-time (s) | `bit` build-time (s) | Speedup  |
| ----------------- | ------------------ | ------------------- | ----- | ----------------------- | ------------------------ | ----- |
| 32                | 7,811              | 405                 | 19.29x | 145                     | 77                       | 1.88x |
| 64                | 7,684              | 405                 | 18.97x | 161                     | 76                       | 2.12x |
| 128               | 7,680              | 405                 | 18.96x | 190                     | 76                       | 2.50x |
| 256               | 7,678              | 405                 | 18.96x | 247                     | 77                       | 3.20x |
| 512               | 7,678              | 405                 | 18.96x | 349                     | 76                       | 4.59x |

### `dbpedia-openai-1000k-angular`

| `ef_construction` | `vector` size (MB) | `bit` size (MB) | Space reduction | `vector` build-time (s) | `bit` build-time (s) | Speedup  |
| ----------------- | ------------------ | ------------------- | ----- | ----------------------- | ------------------------ | ----- |
| 32                | 7,734              | 473               | 16.35x | 244                     | 151                       | 1.62x |
| 64                | 7,734              | 473               | 16.35x | 264                     | 155                       | 1.70x |
| 128               | 7,734              | 473               | 16.35x | 301                     | 162                      | 1.86x |
| 256               | 7,734              | 473               | 16.35x | 377                     | 187                      | 2.02x |
| 512               | 7,734              | 473               | 16.35x | 508                     | 227                      | 2.24x |

From these tests, we see that the space savings becomes significantly more pronounced as the dimensions of the vectors becomes larger, but we do see the reduction trailing off for the 1536-dim vector. This comes back earlier to the fact that a PostgreSQL page is 8KB: we can fit about 68 `bit(960)` on an index page, whereas we can only fit about 42 `bit(1536)` on the same page. However, notice that we don't have much space savings for the 128-dim vector: this makes sense, as we're only transforming 128 bytes (1024 bits) into 128 bits, and while it's an 8x reduction in information, we're already efficiently clustering the flat 128-dim vectors in the index pages. The other interesting element is that the speedups, while at least faster for the larger vectors, don't correlate with the space reduction. There is likely still some optimization work we can do in pgvector to speed up the Hamming distance function.

Now let's at the impact on query performance and recall. First, let's evaluate performance and recall without any reranking:

### `sift-128-euclidean` @ `ef_construction=256` (no rerank)

| `hnsw.ef_search` | `vector` recall | `bit` recall | `vector` QPS | `bit` QPS | `vector` p99 latency (ms) | `bit` p99 latency (ms) |
| ---------------- | --------------- | ---------------- |  ----------- | ------------- | ------------------------- | ------------- |
| 10               | 77.7%           | 2.18%            | 2,100        | 2,412         | 0.71                      | 0.61          |
| 40               | 95.4%           | 2.42%            | 1,020        | 1,095         | 1.19                      | 1.25          |
| 200              | 99.8%           | 2.52%            | 268          | 280           | 4.60                      | 4.91          |
| 800              | 100.0%          | 2.52%            | 84           | 86            | 15.07                     | 16.05         |

### `gist-960-euclidean` @ `ef_construction=256` (no rerank)

| `hnsw.ef_search` | `vector` recall | `bit` recall | `vector` QPS | `bit` QPS | `vector` p99 latency (ms) | `bit` p99 latency (ms) |
| ---------------- | --------------- | ---------------- |  ----------- | ------------- | ------------------------- | ------------- |
| 10               | 50.4%           | 0.00%            | 1,114        | 6,050         | 1.36                      | 0.18          |
| 40               | 78.0%           | 0.00%            | 513          | 4,847         | 2.64                      | 0.24          |
| 200              | 96.0%           | 0.00%            | 135          | 4,057         | 8.70                      | 0.27          |
| 800              | 99.6%           | 0.00%            | 41           | 3,871         | 29.20                     | 0.28          |


### `dbpedia-openai-1000k-angular` @ `ef_construction=256` (no rerank)

| `hnsw.ef_search` | `vector` recall | `bit` recall | `vector` QPS | `bit` QPS | `vector` p99 latency (ms) | `bit` p99 latency (ms) |
| ---------------- | --------------- | ---------------- |  ----------- | ------------- | ------------------------- | ------------- |
| 10               | 85.1%           | 60.1%            | 1,162        | 1,556         | 1.40                      | 1.02          |
| 40               | 96.8%           | 66.8%            | 567          | 848           | 2.70                      | 1.79          |
| 200              | 99.6%           | 68.3%            | 156          | 251           | 9.01                      | 5.61          |
| 800              | 99.9%           | 68.6%            | 48           | 71            | 30.50                     | 19.57         |

The above shows why with approximate nearest neighbor search, you **must** measure performance and recall. Without doing so, you'd walk away thinking that using binary quantization for the `gist-960-euclidean` set is a clear, high performant winner, but you'd be returning garbage results. Similarly, `sift-128-euclidean` has terrible recall with binary quantization, but does not show much difference in query performance between the flat value.

However, `dbpedia-openai-1000k-angular` looks promising: while the recall numbers are not great, they're significantly higher than the others, likely due to the bit-diversity between the values. We may be able to improve our results if we re-rank. Remember that our rerank query looks something like this (using a 3072-dim vector):

```sql
SELECT i.id
FROM (
    SELECT id, embedding <=> $1 AS distance
    FROM items
    ORDER BY quantize_binary(embedding)::bit(3072) <~> quantize_binary($1)
    LIMIT 800 -- bound by hnsw.ef_search
) i
ORDER BY i.distance
LIMIT 10;
```

Let's look at the same results with reranking:

### `sift-128-euclidean` @ `ef_construction=256` (with rerank)

| `hnsw.ef_search` | `vector` recall | `bit` recall | `vector` QPS | `bit` QPS | `vector` p99 latency (ms) | `bit` p99 latency (ms) |
| ---------------- | --------------- | ---------------- |  ----------- | ------------- | ------------------------- | ------------- |
| 10               | 77.7%           | 2.31%            | 2,100        | 2,194         | 0.71                      | 0.65          |
| 40               | 95.4%           | 4.19%            | 1,020        | 971           | 1.19                      | 1.34          |
| 200              | 99.8%           | 8.88%            | 268          | 246           | 4.60                      | 5.32          |
| 800              | 100.0%          | 15.69%           | 84           | 76            | 15.07                     | 17.28         |

### `gist-960-euclidean` @ `ef_construction=256` (with rerank)

| `hnsw.ef_search` | `vector` recall | `bit` recall | `vector` QPS | `bit` QPS | `vector` p99 latency (ms) | `bit` p99 latency (ms) |
| ---------------- | --------------- | ---------------- |  ----------- | ------------- | ------------------------- | ------------- |
| 10               | 50.4%           | 0.00%            | 1,114        | 2,744         | 1.36                      | 0.40          |
| 40               | 78.0%           | 0.00%            | 513          | 1,030         | 2.64                      | 1.00          |
| 200              | 96.0%           | 0.00%            | 135          | 569           | 8.70                      | 1.79          |
| 800              | 99.6%           | 0.00%            | 41           | 562           | 29.20                     | 1.82          |


### `dbpedia-openai-1000k-angular` @ `ef_construction=256` (with rerank)

| `hnsw.ef_search` | `vector` recall | `bit` recall | `vector` QPS | `bit` QPS | `vector` p99 latency (ms) | `bit` p99 latency (ms) |
| ---------------- | --------------- | ---------------- |  ----------- | ------------- | ------------------------- | ------------- |
| 10               | 85.1%           | 60.1%            | 1,162        | 1,503         | 1.40                      | 1.03          |
| 40               | 96.8%           | 91.6%            | 567          | 760           | 2.70                      | 1.91          |
| 200              | 99.6%           | 99.0%            | 156          | 222           | 9.01                      | 6.01          |
| 800              | 99.9%           | 99.8%            | 48           | 62            | 30.50                     | 21.34         |

So, what do we learn with binary quantization and reranking? With `sift-128-euclidean` and re-ranking, we're able to improve on recall as we increase `hnsw.ef_search`, but the recall is still very poor. The `gist-960-euclidean` test is still returning garbage, but not as quickly as before.

But the `dbpedia-openai-1000k-angular` results are very interesting. Once we expand our search radius vis-a-vis `hnsw.ef_search=40`, we see that we get a 1.34x boost in QPS and a 29% reduction in p99 latency with only sacrificing 5% in recall. This is using an index that's 16x smaller and builds twice as fast! Moving up to `hnsw.ef_search=200`, we get 1.42x faster QPS with a 33% decrease in p99 latency with comparable recall!

There are a few takeaways from this. First, given how binary quantization works, the experiments show that to ensure your results have meaning, you most certainly will need to perform a reranking. Additionally, bit-diversity matters: the most distinct values we have in the index, the more likely we'll be able to get distinguishable results that lead to higher recall. This leads to a decision though: how much recall matters? If you're doing k=10 / top-10 queries, 90% should be good enough, which means in the above example, you can use `hnsw.ef_search=40` on the `dbpedia-openai-1000k-angular` dataset to achieve low latency queries that meet your recall target.

One indirect observation I have from these tests is around how the overall size of a dataset impacts recall for binary quantization. We see good results on the `dbpedia-openai-1000k-angular` dataset because the overall dataset size is small (1,000,000) and has good diversity amongst the bit vectors. However, looking at the results of the other two data sets, I would be willing to bet that recall degrades as the 1536-dim data set gets larger (e.g. 1,000,000,000) and requires a larger `hnsw.ef_search` + reranking to achieve the same results.

Finally, for all of these experiments, note that the workload was able to fit entirely into memory (the benefit of using a r7gd.16xlarge) to ensure we're getting a fair comparison between methods. However, in the "real-world," binary quantization lets you keep your index entirely in memory on much smaller systems - for example, with `dbpedia-openai-1000k-angular` the binary quantized index takes up 473MB vs. 7,734MB (7.5GB) for the full index. If you're getting the recall results you want from an index using binary quantization, this is an effective technique to reduce your overall storage and memory footprint in your system.

## Future experiments

While the results are really interesting, there is still more to test (and develop!) around quantization. A few more experiments I'd like to run in the future:

* With `dbpedia-openai-1000k-angular`, quantize the `vector` to `halfvec` in the table, and then use binary quantization on the `halfvec`. I suspect that we reduce the storage/memory footprint even further, get slightly higher QPS/better p99 latency, and don't impact recall.
* Similarly, test storing everything as `halfvec` in the table and then building the `halfvec` index. I also suspect comparable recall, but possibly better query performance numbers. This does lose precision in the vector stored in the table, so I think the results will be model dependent.
* Test the explicit impact of CPU, i.e. SIMD, acceleration. The unreleased PostgreSQL 17 has planned support for AVX-512 some functions used for computing binary distances; we may be able to further speed up those distance functions.
* Test the hypothesis around recall with binary quantization degrading on much larger datasets as there's less bit diversity amongst the values.

## Conclusion: What (if any) quantization technique should I use?

When I give a [talk](/talks/) on pgvector or vector search in general, my concluding slide always ends with "plan for today and tomorrow." I think these quantization techniques are a great example of that.

First and foremost, pgvector maintains its changes as additive, meaning that as you upgrade, you will not need to make costly on-disk changes. Adopting a new feature may mean reindexing, though fortunately PostgreSQL and pgvector support the nonblocking `REINDEX CONCURRENTLY` operation.

That said, I think there are a few clear takeaways and tradeoffs to consider when determining what quantization technique to use:

* **Scalar quantization from 4-byte to 2-byte floats looks like a clear winner**, at least amongst these datasets. I feel comfortable recommending storing the `vector` in the table and quantizing to `halfvec` in the index. I'd like to see the results over larger datasets, but I still think this is a really good starting point.
* **Binary quantization works better for datasets that produce larger vectors that can be distinguished by their bits**. The more distinguished bits, the better recall we'll achieve.
  * I also suspect that as these datasets get very large, we start to see degraded recall results.
* **Effective quantization allows you to reduce your storage/memory footprint, which on one hand means you can use smaller instances, but it also means you can scale your workload even further**. This is great news for pgvector, particularly as we see more ["billion-scale" and beyond datasets](/post/postgres/distributed-pgvector/).

While ANN Benchmarks and other ANN testing tools for databases do provide a good set of datasets to evaluate quantization techniques, this may not reflect the results you see with your actual workload. You still need to test to see if scalar or binary quantization make sense. However, one of my goals is to give you a guide on how to make that evaluation, and hopefully save some time in the process.
