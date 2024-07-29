
## Partial indexes

We’ve learned that we create secondary indexes to minimize the number of rows a query has to look at to find the ones it wants.  But there are cases where that’s difficult, or needlessly expensive.  And for some of those cases, where you’re looking for a comparatively small subset of the table’s rows, a partial index may be the answer.

Take, for example, a column with low cardinality but very lopsided distribution of values amond the rows.  Let’s suppose we had a column in our `book` table for `language`, and 99% of the books were in English, with the remainder in Latin.  An index on `language` would be useful for queries that specify Latin, because it would reduce the search of our 10M row database to just 100,000 rows.  But it would be useless for searches for English books because it doesn’t reduce the search span by much at all.  All thoe 900,000 index entries for English language books would be a waste of space.

Or consider the case where we want to search on a condition that isn’t reflected in the sort order of a column.  Imagine we frequently need to be able to find books whose prices are in whole dollars, or books by authors with names longer than 20 characters.  (Yes, these are abitrary examples, but they illustrate the point.)  Sorting prices doesn’t group the whole numbers together, independent of their value.  Sorting names doesn’t group longer names together, independent of their alphabetical order.  A simple index on those columns wouldn’t help us.

But if we could create an index that only contains entries for the small fraction of rows that meet those criteria, we could find them all easily.  That’s what partial indexes do.

A partial index, also called a sparse index, is an index defined with a predicate clause that determines whether or not that row should get an entry.  Any query with a search term that matches the predicate could use that index to narrow the span.

### Exercise

How many books in the table are written by somebody whose name is 20 characters or greater in length?

```
workshop> select count(*) from book where length(author_name) > 20;
  count
---------
  28134

Time: 6.895s total (execution 6.894s / network 0.000s)
```

That took as long as we expected, since the query had to scan the entire 10M rows.

```
  • group (scalar)
  │ estimated row count: 1
  │
  └── • filter
      │ estimated row count: 3,333,333
      │ filter: length(author_name) > 20
      │
      └── • scan
            estimated row count: 10,000,000
            table: book@book_author_name_idx
            spans: FULL SCAN
```

And, as expected, it used the index on `author_name` so that it could inspect the names by looking at the keys, rather than by reading the table rows.

Now define a partial index on just those books whose authors have names longer than 20 characters.  Then re-run the query and see how it performs.

### Explanation

```
workshop> create index on book(author_name) where length(author_name) > 20;

workshop> select count(*) from book where length(author_name) > 20;
  count
---------
  28134

Time: 19ms total (execution 19ms / network 0ms)
```

That’s a significant improvement, from 6.89 seconds down to 19ms.  (That’s on the order of a 360-fold increase in performance.)

And if we look at the size of the index, we see that it costs next to nothing in storage:

```
       index_name       |           sum
------------------------+---------------------------
  book_author_name_idx2 |    1.8670500000000000000
```

And the cost of calculating the string length of names is paid up front when we `INSERT` or `UPDATE` a book record, and perform the index predicate to decide whether or not to include the row in this index.  None of that work has to be done by the query itself.

### Conclusion

Partial indexes are a special-purpose type of index, useful for finding the proverbial needle in a haystack.  They’re especially useful when no sorting of the indexed column will narrow the search span appropriately, or when the condition on which you’re indexing is a function of multiple columns.  

[Next Section](end.md)
