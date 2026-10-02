---
layout: post
title: "Defining the problem space"
subtitle: "Part 1: getting a beat marketplace out of a text column"
date: 2026-09-22
series: "jsonb migration"
part: 1
tags: [postgres, jsonb, go, migration]
---

During the last month or so, I was in charge of a migration at work, that dealt
with deciding how to store, parse, and handle quite big JSON objects (200MB+) through our
service.

I figured it would be fun to illustrate the problem, some findings along the way,
and what I've learned by doing this.

For the sake of simplifying the scope of the problem, we will use a beat marketplace.
_Wink_ at CMU Database examples. I just need to throw Wu-Tang somewhere in there.

Say we have a legacy schema:

```sql
CREATE TABLE beats_text (
    id      bigint PRIMARY KEY,
    payload text NOT NULL
);
```

Life happens, features start to be requested: filter by tag, search by BPM, which
RZA beat uses this sample, etc. The legacy system fetches the whole document, unmarshals
it, and walks it with nested loops. I have a hunch we can do better than this...

---

## Just use tables

First objection that anyone might have would be: "Why not just use a relational
schema?".

Let's consider the following:
- We didn't design the schema of the payload and we can't version it.
- This is emitted for us.
- Different device models have different data shapes
- The file has to come back out byte identical (keeping track of the original)

If we would RTFM. we would see that:

> Ideally, JSON documents should each represent an atomic datum that business
> rules dictate cannot reasonably be further subdivided into smaller datums that
> could be modified independently.

## Just use a NOSQL DB

If tables are not ideal, use Mongo (or whatever document store you prefer)

Let's consider the following:
- You've read [Postgres for everything](https://www.manning.com/books/just-use-postgres)
- All our services are using exclusively PostgreSQL
- All the emitter services are using exclusively PostgreSQL
- The team is not experienced (i.e. have not used a document store before)

It's a bit tongue-in-cheek what I am doing right now. We did do a comparison between
MongoDB and PostgreSQL. I will touch on it in the later parts.

## Just use JSONB

Couple of caveats worth considering...again:

`jsonb` will throw away:
- whitespace
- key order
- duplicate keys

Thus:

```sql
SELECT '{"bpm":93,"bpm":90}'::json  AS as_json,
       '{"bpm":93,"bpm":90}'::jsonb AS as_jsonb;
```
```
      as_json        |  as_jsonb
---------------------+-------------
 {"bpm":93,"bpm":90} | {"bpm": 90}
```

When considering to use it, I think it's quite important to check if it even models
to the type of data you want to use. A neat trick we can do is to:
`IS JSON OBJECT WITH UNIQUE KEYS`

_or_, if you are using PostgreSQL 15 or older:

```sql
CREATE FUNCTION toplevel_keys_lost(t text) RETURNS integer
LANGUAGE plpgsql IMMUTABLE PARALLEL SAFE AS $$
DECLARE raw int; parsed int;
BEGIN
    SELECT count(*) INTO raw    FROM json_object_keys(t::json);
    SELECT count(*) INTO parsed FROM jsonb_object_keys(t::jsonb);
    RETURN raw - parsed;
EXCEPTION WHEN others THEN RETURN -1;   -- unparseable
END; $$;
```

We ran this on our beats marketplace and it looked like our data has no problems.

## Converting to JSONB

Once set on this, we could rewrite the table, and convert the existing `TEXT` type
to `jsonb`:

```sql
ALTER TABLE beats_text ALTER COLUMN payload TYPE jsonb USING payload::jsonb;
```

However, this would have an `ACCESS EXCLUSIVE` lock on it when doing so. If we
don't want to lock everything, we can do it in batches:

```sql
ALTER TABLE beats_text ADD COLUMN payload_jsonb jsonb;   -- instant, no rewrite

-- repeat until it reports 0
WITH batch AS (
    SELECT id FROM beats_text WHERE payload_jsonb IS NULL
    ORDER BY id LIMIT 25 FOR UPDATE SKIP LOCKED
)
UPDATE beats_text t SET payload_jsonb = t.payload::jsonb
FROM batch WHERE t.id = batch.id;

-- both must return zero
SELECT count(*) FROM beats_text WHERE payload_jsonb IS NULL;
SELECT count(*) FROM beats_text WHERE payload::jsonb IS DISTINCT FROM payload_jsonb;

BEGIN;
  ALTER TABLE beats_text DROP COLUMN payload;
  ALTER TABLE beats_text RENAME COLUMN payload_jsonb TO payload;
COMMIT;
```

## Compression and size

__Size__

I want to touch briefly on the canonical example when comparing `json` to `jsonb`.

The usual demo goes like this:

```sql
SELECT pg_column_size('5'::json),   -- 5 bytes
       pg_column_size('5'::jsonb);  -- 20 bytes
```

Moreover, at document scale, on our data:

| | logical | on disk | table |
|---|---|---|---|
| `text` | 2042 kB | 330 kB | 34 MB |
| `jsonb` | 2283 kB | **396 kB** | **40 MB** |


_However_, the same experiment can have different results on a different dataset.
Here, it's 20% bigger, but I have ran other experiments where it was 11% _smaller_.

Our beat files, for example, are full of floats, and `jsonb` stores numbers as
`numeric`, which costs more than the four characters `0.42`. Therefore, string-heavy
documents with repeated keys will go the other way.

The best way, as usual, is to run a big batch of real documents in both ways and measure.
This way, we know for sure.

__Compression__


This is more like a caveat. You may look into optimizing size by using a compression
codec. I had a surprise when I compared the native codec (`pglz`) with `lz4`: they
were identical.

Take a look at this:

```sql
SELECT DISTINCT pg_column_compression(payload) FROM beats_lz4;
--  pglz
```

That's because of how I did the `INSERT` and `SELECT`. Postgres won't decompress and recompress
your values just because the destination has different settings. The
`COMPRESSION` and `SET STORAGE` clauses are silently ignored for anything
arriving that way.


```sql
INSERT INTO dst SELECT id, payload FROM src_jsonb;        -- keeps the old codec
INSERT INTO dst SELECT id, payload::jsonb FROM src_text;  -- applies the new one
```

If your migration plan copies into a new table to change storage settings, check
what you got:

```sql
SELECT DISTINCT pg_column_compression(payload) FROM dst;
```

## Next

The column is `jsonb`. No application code has changed, so nothing is faster yet.
Every query still fetches the whole project and we still walk through it.

That's Part 2. Then Part 3, where the queries get faster but nowhere near as fast
as they should, and every index I build, GIN and B-tree alike, gets ignored by
the planner. Part 4 tries the whole thing with tables instead, and Part 5 covers
the write path and what changed in Postgres 17 and 18.

---

## References

- [GitLab — Avoiding downtime in migrations][gitlab-downtime] — the practical
  checklist for schema changes that don't take the table down.
- [ankane/strong_migrations][strong-migrations] — which migrations lock, and the
  safer rewrite for each.
- [Braintree/PayPal — PostgreSQL at Scale: schema changes without downtime][braintree]
  — the canonical expand / backfill / contract pattern behind the batched
  conversion above.
- [Nikolay Samokhvalov — Common DB schema change mistakes][postgresai] — the
  failure modes, with Postgres-specific detail.
- [Haki Benita — The surprising impact of medium-size texts on PostgreSQL][haki-toast]
  — TOAST and medium-size values, which backs the size section.

[gitlab-downtime]: https://docs.gitlab.com/ee/development/database/avoiding_downtime_in_migrations.html
[strong-migrations]: https://github.com/ankane/strong_migrations
[braintree]: https://medium.com/paypal-tech/postgresql-at-scale-database-schema-changes-without-downtime-20d3749ed680
[postgresai]: https://postgres.ai/blog/20220525-common-db-schema-change-mistakes
[haki-toast]: https://hakibenita.com/sql-medium-text-performance

---

*PostgreSQL 16.15 in Docker, 100 projects averaging 2042 kB, M-series Mac.
Generator and scripts in the companion repo.*
