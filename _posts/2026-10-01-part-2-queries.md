---
layout: post
title: "Moving away from nested loops"
subtitle: "Part 2: the Go traversal, rewritten as SQL"
date: 2026-10-01
series: "jsonb migration"
part: 2
tags: [postgres, jsonb, go, sql]
---


In [Part 1]({% post_url 2026-09-22-part-1-migration %}) we moved the column to
`jsonb`. Nothing got faster, because nothing in the application changed.

Let's see how the application would go through the data to find a predicate:

```go
for _, m := range beat.Mixes {           // L1
    for _, p := range m.Patterns {       // L2
        for _, h := range p.Hits {       // L3
            if h.Sound.SampleID == id {  // L4
                found = true
            }
        }
    }
}
```

To run that, we fetched 199 MB and allocated 706 MB of Go heap, to answer a
yes/no question.

---

Let's go a bit through our a,b,c's of handling `jsonb` before we go more in-depth.
Feel free to roll your eyes and skip this.

## Selecting data

`->` returns `jsonb`. `->>` returns `text`.
`->` chains because its output is still `jsonb`.
`#>` and `#>>` take the path as an array, which is easier to build from
application code than a chain of arrows.


```sql
SELECT payload->>'title'                      AS title,
       payload->'mixes'->0->>'label'          AS first_mix,
       payload#>>'{mixes,0,patterns,0,name}'  AS first_pattern
FROM beats LIMIT 1;
```
```
         title         |  first_mix   | first_pattern
-----------------------+--------------+---------------
 36 Chambers Type #001 | instrumental | intro
```


While we are here, two gotchas:
a) JSON arrays are zero-based while native Postgres arrays are
one-based, and both live in the same query.
b) Negative indices work:
`->'mixes'->-1` is the last mix.


## Missing paths return NULL

```sql
SELECT payload->'nope'->'still_nope'->>'nothing_here' FROM beats LIMIT 1;
--  (null)
```

Ta-da: no error. This matters for us specifically: our documents come
from several different versions, so optional blocks appear and vanish.
In Go, that means nil checks and type switches.
In SQL it's a value that might be `NULL`, which everyone already handles.

## Contains vs exists

Let's say you want to pattern match on some part of the json.

We could do:

```sql
SELECT id, title FROM beats
WHERE payload @> '{"mixes":[{"patterns":[{"hits":[{"sound":
                  {"sample_id":"SMP-RARE-0001"}}]}]}]}';
```
```
 id  |         title
-----+-----------------------
  25 | Four Track Soul #025
  50 | Crate Digger #050
  75 | Sample Flip #075
 100 | Late Night Study #100
```

The search went deep into the array levels and matched on the value. In this way,
we just match on the keys we care about and the array order does not really matter.

Contrast that with the existence operator `?`, which looks similar but only ever
looks at the top level:

```sql
SELECT '{"mixes":[{"patterns":[{"hits":[{"instrument":"kick"}]}]}]}'::jsonb
         ? 'instrument' AS nested,      -- false
       '{"mixes":[{"patterns":[{"hits":[{"instrument":"kick"}]}]}]}'::jsonb
         ? 'mixes'      AS top_level;   -- true
```

`?` only ever checks top-level keys! Same for `?|` and `?&`.
If you want to ask about a nested key, use containment or apply `?` to an
already extracted subtree.

## Flattening

__Before Postgres 17__

`jsonb_array_elements` turns an array into rows; `LATERAL` lets each level see
the one above it:

```sql
SELECT b.title,
       count(*)                                                 AS hits,
       count(DISTINCT h->>'instrument')                         AS instruments,
       count(*) FILTER (WHERE (h#>>'{sound,reverse}')::boolean) AS reversed
FROM beats b
CROSS JOIN LATERAL jsonb_array_elements(b.payload->'mixes')  m
CROSS JOIN LATERAL jsonb_array_elements(m->'patterns')       p
CROSS JOIN LATERAL jsonb_array_elements(p->'hits')           h
WHERE b.id <= 3
GROUP BY b.title;
```
```
         title         | hits | instruments | reversed
-----------------------+------+-------------+----------
 36 Chambers Type #001 | 2484 |           8 |      127
 36 Chambers Type #002 | 2484 |           8 |       96
 Sample Flip #003      | 2484 |           8 |      129
```

What this does is replace three `LATERAL`s for three `for` loops.
The `FILTER` clause was fifteen lines of accumulator variables before.

## JSON_TABLE

__Postgres 17 and newer__

If you are using Postgres **17 or newer**, there's a better way to write everything
above. Better as in more succinct and faster.


```sql
SELECT b.id, b.title, jt.label, jt.sample_id
FROM beats b,
JSON_TABLE(b.payload, '$.mixes[*]' COLUMNS (
    label text PATH '$.label',
    NESTED PATH '$.patterns[*]' COLUMNS (
        NESTED PATH '$.hits[*]' COLUMNS (
            sample_id text PATH '$.sound.sample_id')))) jt
WHERE b.id = 7;
```

 `JSON_TABLE` is the SQL/JSON standard construct for turning a document into
 rows, and `NESTED PATH` handles what we were doing with the `LATERAL` chain .
All in all, we have one construct instead of three joins and our document shape
is visible in the query.


Twice as fast here:

| | PG17 | PG18 |
|---|---|---|
| three `CROSS JOIN LATERAL` | 3.145 ms | 3.111 ms |
| `JSON_TABLE` with `NESTED PATH` | **1.638 ms** | **1.626 ms** |


---

What it doesn't do is save us from Part 3. If we put a scalar from the document in
the `SELECT` list of a `JSON_TABLE` query, we get the same time (well, almost):

| | time |
|---|---|
| `JSON_TABLE` + `b.payload->>'title'` in the SELECT list | 1,873 ms |
| `JSON_TABLE` + a generated column | **1.60 ms** |

## `JSON_VALUE` and `JSON_EXISTS`

PG17 also brought standard-SQL spellings for extraction and existence:

```sql
SELECT JSON_VALUE(payload, '$.title')                    -- vs payload->>'title'
FROM beats WHERE JSON_EXISTS(payload, '$.mixes[*] ? (@.label == "clean")');
```

This is more syntactic preference, since they return the same answers, and on our
data they run at the same speed: `->>` 88.2 ms against `JSON_VALUE` 88.3 ms across all documents, `@?` 88.3 ms
against `JSON_EXISTS` 88.4 ms.

The only reason I see to use this is that  `JSON_VALUE` lets you say `RETURNING
integer` and set `ON ERROR` behaviour, which is a bit nicer than casting
`->>` and hoping to be ok.


## Read only endpoint response

For read-only endpoints you can skip the model layer:

```sql
SELECT jsonb_build_object(
         'id', b.id, 'title', b.title, 'bpm', b.bpm,
         'mixes', (SELECT jsonb_agg(jsonb_build_object(
                            'label',    m->>'label',
                            'patterns', jsonb_array_length(m->'patterns')))
                   FROM jsonb_array_elements(b.payload->'mixes') m)
       ) FROM beats b WHERE b.id = 25;
```

```go
var resp json.RawMessage
pool.QueryRow(ctx, q, id).Scan(&resp)
w.Write(resp)
```

This way we have: no structs, no unmarshal, and no re-marshal.

If it's going straight over the wire use `json_*` rather than `jsonb_*`;
there's no point building a binary representation you're about to serialise
back to text.

---

## References

- [PostgreSQL 17 manual, §9.16 JSON Functions and Operators][pg17-json] —
  `JSON_TABLE`, `JSON_VALUE`, `JSON_EXISTS`, `JSON_QUERY`, and the SQL/JSON
  path language.
- [PostgreSQL 16 manual, §8.14.3 jsonb Containment and Existence][pg-containment] — why `@>` nests
  and `?` doesn't, which is the thing that cost me an afternoon.
- [PostgreSQL 17 manual, §9.16.2 The SQL/JSON Path Language][pg-jsonpath] — the `@?` and `jsonpath`
  syntax behind `JSON_EXISTS` and the path queries above.
- [Dimitri Fontaine, *The Art of PostgreSQL*][artofpg] — the case for replacing
  application loops with SQL, which is the whole thesis of this post.
- [Martin Kleppmann, *Designing Data-Intensive Applications*, Ch. 2][ddia] —
  declarative versus imperative queries, i.e. why deleting the nested loops is
  the point.

[pg17-json]: https://www.postgresql.org/docs/17/functions-json.html
[pg-containment]: https://www.postgresql.org/docs/16/datatype-json.html#JSON-CONTAINMENT
[pg-jsonpath]: https://www.postgresql.org/docs/17/functions-json.html#FUNCTIONS-SQLJSON-PATH
[artofpg]: https://theartofpostgresql.com/
[ddia]: https://dataintensive.net/

---

*PostgreSQL 16.15 in Docker, 100 projects averaging 2042 kB, Go 1.26.6. The
`JSON_TABLE` and `JSON_VALUE` figures are from PostgreSQL 17.11 and 18.6, since
those constructs need 17+; `sql/12-modern.sql` runs on 16, 17 and 18 and skips
what doesn't apply.*
