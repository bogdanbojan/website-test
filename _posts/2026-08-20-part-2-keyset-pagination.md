---
layout: post
title: "A Cursor Is Just a Row You Already Saw"
subtitle: "Part 2: keyset pagination done correctly, including ties"
date: 2026-08-20
series: "offset to keyset"
part: 2
tags: [postgres, pagination, go, sql]
---

In [Part 1]({% post_url 2026-08-20-part-1-keyset-pagination %}) `OFFSET` turned
out to cost `O(offset)` per page, and — independently of speed — to serve
duplicate or missing rows once the table is being written to while a client is
paging through it.

This part replaces it. The fix has a name — keyset pagination, also called
cursor-based or seek pagination — and the idea fits in one sentence: **stop
asking for a page number, and start asking for the rows after the last row you
saw.**

---

## The query that replaces OFFSET

```sql
SELECT id, customer, total, created_at
FROM orders
WHERE created_at < $1
ORDER BY created_at DESC
LIMIT 20;
```

`$1` is the `created_at` of the last row from the previous page. No offset,
no counting rows to skip — the `WHERE` clause seeks straight to where the last
page ended, and the index does the rest:

```
EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM orders WHERE created_at < '2026-06-01 00:00:00+00'
ORDER BY created_at DESC LIMIT 20;
```

```
 Limit (actual time=0.03..0.05 rows=20 loops=1)
   ->  Index Scan using orders_created_at_idx on orders
         (actual time=0.03..0.05 rows=20 loops=1)
```

`rows=20`. Not 20,020, not 400,020 — twenty, regardless of where `$1` falls in
the table. Page 1 and the 20,000th page cost the same, because the executor
seeks to a value in the index instead of counting from the start:

| page depth (equivalent offset) | OFFSET latency (Part 1) | keyset latency |
|---|---|---|
| 0 | 0.4 ms | 0.4 ms |
| 1,000 | 1.1 ms | 0.4 ms |
| 10,000 | 8.7 ms | 0.4 ms |
| 100,000 | 49.0 ms | 0.4 ms |
| 400,000 | 187 ms | 0.4 ms |

Flat. That's the whole pitch for performance. The more interesting part is
that it also happens to fix the correctness bug from Part 1 as a side effect,
for free — a cursor is a value, not a position, so it doesn't care that rows
shifted around underneath it. There is nothing to shift relative to.

---

## Why one column is not enough

The query above has a bug, and it's the kind that passes every manual test
because manual testing rarely produces two rows with the same timestamp.

```sql
SELECT id, created_at FROM orders
WHERE created_at = '2026-06-01 09:14:22.118+00';
```

```
 id   | created_at
------+------------------------------
 8841 | 2026-06-01 09:14:22.118+00
 8842 | 2026-06-01 09:14:22.118+00
 8843 | 2026-06-01 09:14:22.118+00
```

Three orders inserted in the same batch, same timestamp to the millisecond —
common under any kind of bulk insert or high write throughput. `created_at`
alone does not identify a row's position in the ordering. If a page happens to
end in the middle of this trio, `WHERE created_at < $1` either re-serves the
rows that share `$1`'s timestamp (if you use `<=` and forget to exclude the
one you already returned) or silently skips whichever of them sort before the
cursor within that same instant (if you use `<`, and one of the tied rows
hadn't been returned yet).

The fix is a tiebreaker: a second column, unique, that's part of the sort
order and the cursor.

```sql
SELECT id, customer, total, created_at
FROM orders
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

`(created_at, id) < ($1, $2)` is row-wise comparison — standard SQL, and
Postgres implements it directly rather than you having to spell out the
`OR` by hand. It reads left to right, exactly like comparing version numbers:
compare `created_at` first, and only fall through to comparing `id` when the
first column ties. That matches the sort order exactly, so it's the one WHERE
clause that correctly resumes from *any* row, tied timestamp or not.

The equivalent unrolled by hand, in case your driver doesn't support row
comparison syntax cleanly:

```sql
WHERE created_at < $1
   OR (created_at = $1 AND id < $2)
```

Same result, worse for the planner in older Postgres versions — row
comparison lets a single index on `(created_at, id)` satisfy the whole
predicate with one scan; the `OR` form sometimes gets planned as two scans
bitmap-OR'd together. Prefer the tuple form when your client library allows
binding it.

**The rule generalizes:** if you sort by N columns, your cursor and your
comparison need all N, plus a tiebreaker if the last one isn't already unique.
Sorting by `(status, created_at)`? Cursor is `(status, created_at, id)`. There
is no shortcut around this — it's the same reason a phone book needs a first
*and* last name before it can tell you whose entry comes next.

---

## The index has to match, in the same order

```sql
CREATE INDEX orders_created_at_id_idx ON orders (created_at DESC, id DESC);
```

Column order and sort direction in the index need to mirror the `ORDER BY`
exactly. Get the direction wrong — index it ascending while sorting
descending — and Postgres can usually still use it by scanning backward, but
it's worth confirming with `EXPLAIN` rather than assuming:

```
->  Index Scan Backward using orders_created_at_id_idx on orders
```

`Backward` there is fine — it's still an index scan, still `O(log n)` to
seek, still cheap. What you don't want to see is `Seq Scan`, which means the
index doesn't match the query shape at all and you're back to reading the
whole table.

---

## Building the cursor in Go

The cursor is just the sort-key values of the last row returned, opaque to
the client:

```go
type Cursor struct {
	CreatedAt time.Time `json:"created_at"`
	ID        int64     `json:"id"`
}

func encodeCursor(c Cursor) string {
	b, _ := json.Marshal(c)
	return base64.URLEncoding.EncodeToString(b)
}

func decodeCursor(s string) (Cursor, error) {
	var c Cursor
	b, err := base64.URLEncoding.DecodeString(s)
	if err != nil {
		return c, err
	}
	return c, json.Unmarshal(b, &c)
}
```

```go
func ListOrders(ctx context.Context, cursor string, pageSize int) ([]Order, string, error) {
	var after Cursor
	if cursor != "" {
		var err error
		if after, err = decodeCursor(cursor); err != nil {
			return nil, "", err
		}
	} else {
		after = Cursor{CreatedAt: farFuture, ID: math.MaxInt64} // no filter, effectively
	}

	rows, err := pool.Query(ctx,
		`SELECT id, customer, total, created_at FROM orders
		 WHERE (created_at, id) < ($1, $2)
		 ORDER BY created_at DESC, id DESC LIMIT $3`,
		after.CreatedAt, after.ID, pageSize)
	...

	var next string
	if len(orders) == pageSize {
		last := orders[len(orders)-1]
		next = encodeCursor(Cursor{CreatedAt: last.CreatedAt, ID: last.ID})
	}
	return orders, next, err
}
```

The response carries `next_cursor` instead of a page number. The client
doesn't decode it, doesn't construct one, just echoes it back on the next
request. That opacity is doing real work: it means the sort key backing
pagination can change later — add a tiebreaker, change a column — without
every client that's bookmarked a URL breaking, because they never depended on
its internal shape.

---

## What you give up

Keyset pagination isn't a strict upgrade — it trades away a few things
`OFFSET` gave you for free:

- **No jumping to page N.** There's no query that means "skip to page 40"
  without walking there one cursor at a time, because a cursor only knows
  the row before it, not an absolute position. If your UI has numbered page
  links people click directly, this is the one that hurts.
- **"Total pages" needs a separate count.** `OFFSET`'s result set doesn't
  tell you the total either, but people expect page-number UIs to show one,
  and that's still a `count(*)` you have to run and pay for on its own.
- **Going backward is the same query, mirrored.** Flip the comparison
  (`>` instead of `<`), flip `ORDER BY` to ascending, run it, then reverse the
  returned rows client-side before rendering. It's a real feature, just not
  the same query as forward — worth wrapping in a small helper so nobody
  reimplements it slightly wrong per endpoint.

None of these are reasons to keep `OFFSET` for a feed, an infinite-scroll list,
an export job, or an API meant to be paged through start to finish — which
covers most list endpoints. They're reasons to keep `OFFSET`, deliberately,
for the specific screens where a user actually clicks "page 7" directly, and
to be honest that you're paying its cost there in exchange for that.

---

Part 3 is the part every team actually asks about once the theory makes
sense: what do you do about the existing API that already has clients out in
the world constructing `?page=7`, and can't all be updated in the same
release.

---

*Numbers above: same setup as Part 1 — PostgreSQL 16 in Docker, 500,000 rows,
warm cache, M-series Mac. Keyset latency is flat across all five depths
because the query does the same amount of work regardless of how far into
the table the cursor points.*
