---
layout: post
title: "UUIDv4 vs bigint primary keys in Postgres (buffers + locality demo)"
description: "A tiny psql script + real output to see how random PKs amplify page touches"
---

I previously wrote about UUIDv7 in Postgres. After reading Andy Atkinson’s deep dive on UUIDv4 primary keys, the “why” clicked even harder: **random primary keys turn into extra work in B-tree indexes**.

# The script
Gist: <https://gist.github.com/LxYuan0420/1cfe75637dbb811c6b88fef605ac88e4>

# Why UUIDv4 primary keys hurt (in Postgres)
Postgres primary keys are backed by B-tree indexes. With UUIDv4:
- inserts don’t “append” to the right-most leaf page; they land on random pages
- random inserts cause more page splits + fragmentation + larger indexes
- UUID is 16 bytes (vs 8 bytes for `bigint`), so you fit fewer entries per 8KB page
- low correlation between key order and physical row order => range scans / pagination touch more pages

Also: UUIDs are not security “capabilities” (RFC 4122 explicitly warns against treating them as unguessable secrets).

# Run the script (step-by-step)
0) Confirm Postgres is reachable:

```bash
$ psql -d postgres -c 'select version();'
```

1) Download the script from the gist:

```bash
$ curl -L 'https://gist.github.com/LxYuan0420/1cfe75637dbb811c6b88fef605ac88e4/raw' -o uuid_pk_bench.sql
```

2) Create a scratch database (safe; the script drops a schema named `uuid_pk_bench` inside this DB):

```bash
$ dropdb uuid_pk_bench 2>/dev/null || true
$ createdb uuid_pk_bench
```

3) Run it (and save output):

```bash
$ psql -d uuid_pk_bench -f uuid_pk_bench.sql -v nrows=200000 -v limit_rows=20000 | tee bench.out
```

## Alternative: run without saving the file locally:

```bash
$ curl -L 'https://gist.github.com/LxYuan0420/1cfe75637dbb811c6b88fef605ac88e4/raw' \
  | psql -d uuid_pk_bench -v nrows=200000 -v limit_rows=20000 \
  | tee bench.out
```

Optional: VACUUM first (often reduces `Heap Fetches` for index-only scans):

```bash
$ psql -d uuid_pk_bench -f uuid_pk_bench.sql -v nrows=200000 -v limit_rows=20000 -v vacuum=1 | tee bench_vacuum.out
```

# Read the output
If you saved output to `bench.out`, I usually scan it like this:

```bash
$ rg '^== ' bench.out
$ rg 'Buffers:|Heap Fetches:|Execution Time:' bench.out
```

# What it measures (and why)
- `Sizes`: table + PK index sizes (UUID PK indexes are often bigger).
- `Correlation`: `pg_stats.correlation` for the `id` column (1.0 = strongly ordered, ~0 = random).
- `ORDER BY id LIMIT ...`: a rough proxy for keyset pagination / range scans (locality matters).
- `UPDATE` that same LIMIT set: same idea, but with heap writes.

Note: this is intentionally *not* a point-lookup benchmark (`WHERE id = ...`). It’s meant to show what happens when you traverse lots of rows in primary-key order (common in pagination, maintenance jobs, or ORMs that implicitly `ORDER BY id`).

# Example output (nrows=200000, limit_rows=20000)
This is trimmed to the lines I actually look at.

```text
== Sizes (table + pkey index) ==
    relname     |  size
----------------+---------
 uuidv4_pk      | 28 MB
 bigint_pk      | 27 MB
 uuidv4_pk_pkey | 8536 kB
 bigint_pk_pkey | 4408 kB

== Correlation (how well physical order matches key order) ==
 tablename | attname | correlation
-----------+---------+-------------
 bigint_pk | id      |           1
 uuidv4_pk | id      | 0.010380032

== Index-only scan (id only) ==
-- bigint_pk
Buffers: shared hit=402
Heap Fetches: 20000
Execution Time: 2.526 ms

-- uuidv4_pk
Buffers: shared hit=20112
Heap Fetches: 20000
Execution Time: 8.450 ms

== Index scan + heap fetch (payload) ==
-- bigint_pk
Buffers: shared hit=402
Execution Time: 2.407 ms

-- uuidv4_pk
Buffers: shared hit=20112
Execution Time: 4.678 ms

== Update the same LIMIT set (forces heap writes) ==
-- bigint_pk
Buffers: shared hit=144555 dirtied=242 written=242
Execution Time: 73.993 ms

-- uuidv4_pk
Buffers: shared hit=164486 dirtied=233 written=233
Execution Time: 88.780 ms
```

## How to interpret:
- **Index size**: the UUID PK index is ~1.9× bigger here (8.5MB vs 4.4MB).
- **Correlation**: `bigint` is perfectly correlated (1.0) because it’s inserted sequentially; UUIDv4 is basically random (~0.01).
- **Buffers**: each buffer is an 8KB page. So `402` buffers is ~3.1MB of page touches, while `20112` buffers is ~157MB — for the same `LIMIT 20000`. (`hit=...` means it was already in Postgres shared buffers; if you see `read=...` that’s disk I/O.)
- **Heap Fetches in an “index-only” plan**: Postgres still has to visit the heap to check tuple visibility unless the visibility map says the page is “all-visible”. That’s why the script has an optional `-v vacuum=1` mode.
- **Updates**: UUIDv4 tends to spread writes/reads across more pages, so you’ll often see higher buffer counts and slightly higher latency for equivalent updates.

# Links
- Andy Atkinson: <https://andyatkinson.com/avoid-uuid-version-4-primary-keys>
- My UUIDv7 note: <https://lxyuan0420.github.io/posts/uuidv7-postgresql>
