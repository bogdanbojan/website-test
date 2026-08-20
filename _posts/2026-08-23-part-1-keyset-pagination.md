---
layout: post
title: "The Twentieth Page Costs More Than the First"
subtitle: "Part 1: why OFFSET pagination gets slower, and sometimes wrong"
date: 2026-08-23
series: "offset to keyset"
part: 1
tags: [postgres, pagination, go, performance]
---

Every list endpoint starts the same way.

```go
func ListOrders(ctx context.Context, page, pageSize int) ([]Order, error) {
	rows, err := pool.Query(ctx,
		`SELECT id, customer, total, created_at
		 FROM orders
		 ORDER BY created_at DESC
		 LIMIT $1 OFFSET $2`,
		pageSize, page*pageSize)
	...
}
```

It works. It's obvious. Page 1 comes back in a couple of milliseconds, and
nobody thinks about it again until a support ticket says the "export all
orders" feature times out somewhere around page 400.

This is a three-part series about replacing `OFFSET` pagination with keyset
(cursor) pagination. This part covers the diagnosis: why the cost of a page
grows with its page number, and a correctness bug that has nothing to do with
performance and everything to do with rows quietly appearing twice or not at
all.

- **Part 1 — the diagnosis** (you are here)
- Part 2 — keyset pagination done correctly, including ties and composite sort keys
- Part 3 — migrating an existing OFFSET-based API without breaking every client that's bookmarked a page

---

## Where it starts

```sql
CREATE TABLE orders (
    id         bigint PRIMARY KEY,
    customer   text NOT NULL,
    total      numeric NOT NULL,
    created_at timestamptz NOT NULL
);
CREATE INDEX orders_created_at_idx ON orders (created_at DESC);
```

500,000 rows, an index on the column we sort by. By every instinct this
should be fast at any page number — we have an index, and we're only ever
asking for 20 rows.

## OFFSET doesn't skip. It reads and throws away.

That's the part the query hides from you. `OFFSET 10000` does not seek to the
10,000th row. Postgres walks the index in order, materializes every row up to
`OFFSET + LIMIT`, and discards the first `OFFSET` of them. The work is
proportional to how deep you page, not to how many rows you asked for.

```
EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 100000;
```

```
 Limit (actual time=48.9..49.0 rows=20 loops=1)
   ->  Index Scan Backward using orders_created_at_idx on orders
         (actual time=0.02..44.1 rows=100020 loops=1)
```

`rows=100020` is the tell. To hand back 20 rows, the executor produced and
discarded 100,000 of them first. The `LIMIT` only trims the output; it does
nothing to reduce the scan.

Page timings on the same table, same query shape, `LIMIT 20`:

| offset | rows scanned | latency |
|---|---|---|
| 0 | 20 | 0.4 ms |
| 1,000 | 1,020 | 1.1 ms |
| 10,000 | 10,020 | 8.7 ms |
| 100,000 | 100,020 | 49.0 ms |
| 400,000 | 400,020 | 187 ms |

It's a straight line. Every page costs more than the one before it, and by
the time an export loop reaches the end of the table it is paying almost as
much for one page as it took to read the entire table once.

Nobody notices this in development, because development doesn't have
400,000 rows. It shows up in production, on the page number that corresponds
to "all of it," which is exactly where a support ticket or a cron job ends
up asking.

## The trap: pages don't just get slow, they get wrong

This is the one that doesn't show up in `EXPLAIN` at all, because it isn't a
performance problem. `OFFSET` defines a page purely by *position* in a
result set that is being reordered underneath you by every insert and delete.

Say a client is paging through `orders`, newest first, 20 rows a page. Between
fetching page 1 and page 2, one new order arrives.

```
page 1 (offset 0)  fetched before insert : rows 1-20  (newest → oldest)
                    new row inserted at the very front
page 2 (offset 20) fetched after insert  : rows 21-40 of the *new* ordering
```

Row 20 from the old ordering is now row 21 — it gets served again, on page 2.
Meanwhile whatever used to be row 40 has been pushed to row 41 and never gets
served at all. No error, no warning, just a duplicate the client renders and
a row that silently never appeared in the export.

Deletes do the same thing in the other direction: a row ahead of the cursor's
position gets removed, everything shifts up by one, and the next page skips a
row it should have shown.

The failure mode scales with how busy the table is and how long the user
takes to click "next." An admin dashboard that's paged through once a day
might never notice. An export job walking 400,000 rows while orders are
actively being inserted will drop rows on almost every run, and because the
symptom is "one row missing out of thousands," it looks like anything except
what it actually is.

## Why the index doesn't save you

It's tempting to assume a covering index fixes this — after all, the plan
above is already using one. It doesn't, because the problem was never seeking
to a value. `OFFSET N` has no value to seek to. It's a request for "the Nth
row of this ordering," and the only way to know which row is Nth is to count
that many rows from the start. An index makes the counting faster per row; it
does not make the count anything other than `O(offset)`.

The fix is to stop asking "give me row N" and start asking "give me the rows
after the last one I saw" — a question that has an actual value to seek on,
and one an index can answer in `O(log n)` regardless of how deep the client
has paged. That's keyset pagination, and it's also naturally immune to the
duplicate/skip problem above, because a page is defined by a cursor value
instead of a position that drifts under concurrent writes.

That rewrite — including the part that's easy to get wrong, which is what
happens when two rows share the same `created_at` — is Part 2.

Part 3 is the less satisfying half of this: most `OFFSET` APIs already have
clients out in the world that construct page numbers directly, and you can't
always demand they switch to an opaque cursor overnight. That part is about
running both schemes side by side without the two disagreeing about what
"page 2" means.

---

*Numbers above: PostgreSQL 16 in Docker, 500,000 rows, single unpartitioned
table, warm cache, M-series Mac. The point is the shape of the curve — linear
in offset — which holds regardless of exact hardware.*
