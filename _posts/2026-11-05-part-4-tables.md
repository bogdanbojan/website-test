---
layout: post
title: "Try it with tables"
subtitle: "Part 4: eleven tables, one missing ORDER BY, and a different beat"
date: 2026-11-05
series: "jsonb migration"
part: 4
tags: [postgres, jsonb, schema, normalization]
---

I think anyone who reads this will think why go into so much depth with JSONB?
Wouldn't just splitting the data into tables be easier, and probably more efficient?

Let's see!

## Splitting the data into tables

```
beat_mix              (+ ord)
beat_mix_tag          (+ ord)   []string
beat_mix_mood         (+ ord)   []string
beat_pattern          (+ ord)
beat_pattern_preset   (+ ord)   []string, only [0] is read
beat_mixer            (1:1)
beat_mixer_send       (+ ord)   [0]=reverb [1]=delay [2]=master
beat_hit              (+ ord, + 15 inlined sound columns)
beat_layer            (+ ord)   first match wins
beat_fx               (+ ord)   the chain
sample_lookup         (flat)
```

## Storage


| | size |
|---|---|
| `jsonb` blob (compressed) | 412 kB |
| flat derived table | 317 kB |
| **normalized (3 of the 11 tables)** | **3,032 kB** |

Three tables cost 7.4× the whole compressed document. Decomposing destroys
compression, and Postgres charges ~24 bytes of tuple header per row, which on
322,920 rows adds up.

Most of the time I see normalization as the space efficient option. It's intersting
here because it's not the case.

## Row order

An interesting thing happens if you don't persist the order (`ORDER BY ord`).
Sure, this is just a mock example, but it can bite you if you don't know about it.

**From the document, JSONB:**
```
  kick     |. . . . . . # x X X . . . . # .|
  snare    |. . . . . x x X . . - . . . . -|
  hihat    |. . . . . . . - . . - . - x # .|
  bass     |X . . . X . . . - . . - x . # X|
```

**From the tables, with `ORDER BY ord`:**
```
  kick     |. . . . . . # x X X . . . . # .|
  snare    |. . . . . x x X . . - . . . . -|
  hihat    |. . . . . . . - . . - . - x # .|
  bass     |X . . . X . . . - . . - x . # X|
```

**From the tables, without `ORDER BY ord`:**
```
  kick     |. . . . . . X x X X . . . . X .|
  snare    |. . . . . x x X . . - . . . . -|
  hihat    |. . . . . . . - . . - . - x X .|
  bass     |X . . . X . . . - . . - x . X X|
```

Why did `#` become `X`? Because our velocity layers are first-match-wins:

```json
"layers": [
  { "min_velocity": 120, "symbol": "#" },
  { "min_velocity": 100, "symbol": "X" },
  { "min_velocity":  64, "symbol": "x" },
  { "min_velocity":   0, "symbol": "-" }
]
```

Then, a hit at velocity 124 checks 120 first and renders `#`. However, if you
put 100 first, the same hit renders `X`.

---

I'll just push on this a bit more, to see where this might happen IRL.

```sql
UPDATE r_layer SET min_velocity = min_velocity WHERE ...;
```

Maybe you think it's a bit convoluted, since I'm assigning it's own current value
to every row, therefore nothing changes. But the important thing to note, is that
`UPDATE` in Postgres is kind of delete and then insert. Ergo, each row got rewritten:

```
-- WITH ORDER BY ord          -- WITHOUT
 ord | min_velocity            ord | min_velocity
-----+--------------          -----+--------------
   1 |          120             2 |          100
   2 |          100             3 |           64
   3 |           64             4 |            0
   4 |            0             1 |          120
```

Whether it's a no-op update like this, or a real edit, it would reorder the array
nevertheless.

In the document, order is preserved. In the table split, now we have to carry an `ord`
column.

## Other scenarios

Let's say we want to see *Which projects use this sample?*. This would need us
to go four array levels deep. No index would get chosen (remember TOAST), and we
would end up scanning all ~hundred entries.

What we could do is have one flat, denormalized table:

```sql
CREATE TABLE hits_flat AS
SELECT b.id AS beat_id, b.title, m->>'label' AS mix,
       h#>>'{sound,sample_id}' AS sample_id
FROM beats b
CROSS JOIN LATERAL jsonb_array_elements(b.payload->'mixes') m
CROSS JOIN LATERAL jsonb_array_elements(m->'patterns')      p
CROSS JOIN LATERAL jsonb_array_elements(p->'hits')          h;

CREATE INDEX ON hits_flat (sample_id);
```

| | time |
|---|---|
| containment over the blob, planner's choice | ~112 ms |
| full-document GIN, forced | ~3.4 ms |
| targeted GIN on the projected `sample_id` path, forced | ~4.3 ms |
| **flat table** | **~0.013 ms** |

Using this flat table is also convenient. We don't have to have: `ord` columns,
foreign keys, joins, we can truncate it, rebuild it from documents,etc.

## Comparison

| | document + projections | 11 normalised tables |
|---|---|---|
| tables | 2 (+3 columns) | 11 |
| round trips | 1 | 5 (or one 32× product) |
| storage per project | 729 kB | 3,032 kB (3 of 11) |
| repository code | `json.Unmarshal` + one `INSERT … SELECT` | ~280 lines of reassembly |
| `ORDER BY` invariants | 0 | 8 |
| "which projects use this sample?" | 0.013 ms | needs the joins |
| firmware adds a field | nothing | migration + DDL + ingest + backfill |
| byte-exact re-export | free | **impossible** |


So? Document or table?

For our usecase, it the document is faster and easier to use.
We do have to keep in mind a couple of restraints that pop up normally when
thinking about a similar situation:

1. **Can't design the format and can't version it**
2. **Hand it back to other actors intact**
3. **Order matters**

---

*PostgreSQL 16.15 in Docker, 100 projects averaging 2042 kB, Go 1.26.6. Schema,
generator, render demo and raw `EXPLAIN` output in the companion repo.*
