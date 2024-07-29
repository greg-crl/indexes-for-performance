## Compound indexes

We create an an index in order to reduce the search span required to find the rows a query is interested in.  If our search criteria include 2 or more filter columns, it may be worth creating an index that combines the values of those columns to further narrow the search.

### Exercise

Our last index reduced the search for paperback books that cost more than $75 to just the half million paperbacks.  Our covering index kept us from having to index join back to the table, but we still had to look at half a million prices to find the 126K that we were looking for.

Our query would have performed even better if we’d only had to look at a quarter as many records.  We can do that by including the price in the key value of the index, instead of in a stored column where we have to read the index record (instead of just its key) to see it.

Define an index that includes both of our search terms.  Rerun the query, and evaluate the results.  Does performance improve?  Why?

### Explanation

```
workshop> create index on book (format, price);

workshop> select count(*) from book where format='paperback' and price > 95;
  count
----------
  126263

Time: 92ms total (execution 92ms / network 0ms)
```

Much much better.  Let’s look at why:

```
workshop> explain select count(*) from book where format='paperback' and price > 95;

  • group (scalar)
  │ estimated row count: 1
  │
  └── • scan
        estimated row count: 118,845 
        table: book@book_format_price_idx
        spans: [/'paperback'/95.00000000000001 - /'paperback']
```

You can see here that our index keyspace is segmented first into books grouped by `format`.  Within that `format` group, the keys are sorted by `price`.  That means that the key values for all the rows we’re looking for in this query are consecutive in that index.

### Another best practice

In any engineering task, database design included, it’s important to understand the tradeoffs, and pay attention to costs.  As we’ve noted before, indexes are terrifically useful, but they’re not free.  They cost time (CPU cycles and query latency to keep up to date) and money (disk space to store them).  It’s a best practice to monitor the indexes you’ve created and delete any that are no longer useful.

### Exercise

We’ve created some indexes and significantly improved the performance of our queries.  But at what cost?  Let’s see what indexes we’ve got.

```
workshop> show create table book;

INDEX book_author_name_idx (author_name ASC),
             |     id UUID NOT NULL DEFAULT gen_random_uuid(),
             |     title STRING NOT NULL,
             |     author_name STRING NULL,
             |     price FLOAT8 NOT NULL,
             |     format STRING NOT NULL,
             |     INDEX book_author_name_idx (author_name ASC),
             |     INDEX book_author_name_idx1 (author_name ASC) STORING (price),
             |     INDEX book_format_idx (format ASC),
             |     INDEX book_format_idx1 (format ASC) STORING (price),
             |     INDEX book_format_price_idx (format ASC, price ASC)
             | )
```

Each of our secondary indexes has an entry for each of the 10M rows in our table, so we’ve added 50M entries to our keyspace.  And two of those indexes have STORING clauses, so they take up even more space.  Let’s have a look at how much.  CockroachDB keeps track of the size of each range, and we can use that to calculate the size of tables and indexes.  Use this handy little query to look at the ranges for the `book` table and its indexes

```
workshop> show ranges from table book with indexes, details
```

Ugh, the output’s unreadable.  Way too much.  Let’s just look at the columns we’re interested in.  Also, each index has multiple ranges, and we want to know how much room each index takes up, so let’s try:

```
workshop> select index_name, sum(range_size_mb) from 
    [show ranges from table book with indexes, details] 
    group by index_name;

       index_name       |  mb
------------------------+-------
  book_pkey             | 1401
  book_author_name_idx  |  585
  book_author_name_idx1 |  675
  book_format_idx      |  520
  book_format_idx1       |  610
  book_format_price_idx |  610
```

It looks like we’re spending more than twice the storage on our collection of secondary indexes than we are on the table itself (that is, the primary index, `book_pkey`).  And not all of these indexes are useful:

* `book_author_name_idx1` duplicates the grouping and sort order of `book_author_name_idx`.  They’re both useful for searching on `author_name`, but the covering index is more useful.
* The same thing applies to `book_format_idx1` and `book_format_idx`.
* the compound index `book_format_price_idx` is the same size as `book_format_idx1`, and carries the same information, but sorted in a way that makes it more useful.

We can drop the less useful duplicate indexes:

```
workshop> drop index book_author_name_idx;

workshop> drop index book_format_idx;

workshop> drop index book_format_idx1;
```

### Exercise

All our secondary indexes include price information, so let’s query for books that cost more than $50.

```
workshop> select count(*) from book where price > 50;
   count
-----------
  5052596

Time: 6.046s total (execution 6.046s / network 0.000s)
```

That didn’t go as well as we’d like.  What happened?  

### Explanation

The explain plan says:

```
  • group (scalar)
  │ estimated row count: 1
  │
  └── • filter
      │ estimated row count: 5,018,306
      │ filter: price > 50.0
      │
      └── • scan
            estimated row count: 10,000,000 
            table: book@book_format_price_idx
            spans: FULL SCAN
```

We’re using one of our secondary indexes, but we’re scanning the whole thing, looking at every book in the table.  What’s going on?

Let’s review the structure of our index.  From the `show create table book;` command above:

```
INDEX book_format_price_idx (format ASC, price ASC)
```

We’re using our compound index instead of the primary key, but only because the value of price is encoded in the key value.  That's allowing the query look at the price for each book without having to reaad a record, because it's part of the key value. But the keys are kept in sorted order, and the keys of this index start with `format`.  That means that paperbacks with prices above $50 are all together in one place, but hardcovers with prices above $50 are grouped together somewhere else, and so on.  The optimizer determined that it would be cheaper to read through the whole index than try to find the chunks of the index with the data we’re looking for.

The lesson here is that **column order matters in compound indexes**.  A compound index is only useful when your query specifies a value for the leading columns.  You don’t have to specify **ALL** the columns, but the first column you **don’t** specify is the one where the sort order of the keys doesn’t help you anymore.

The existing `book_format_price_idx index` is useful for searches that filter on `format` and `price`, or just `format`, but not for searches that only filter on `price`.

### Conclusion

You can define compound indexes to serve more than one kind of query, but you may need the same index columns combined in different order to serve others.  Everything depends on the queries.  This isn't a surprise, since secondary indexes are defined in the context of the queries they're intended to optimize.

[Next Section](partial.md)
