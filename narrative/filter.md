## Index columns you filter on

Secondary indexes exist for the sole purpose of making queries faster.  It is a fundamental best practice to index the columns your queries filter on.  We'll explore that in this section.

You're starting out with a database containing a single table of books.  Each book entry consists of a title, the author's name, the format of the book (hardcover, paperback, ebook, or audio), and the publication date.
Have a look at the contents.  (Hint: There are 10 million records, so it wouldn't do to select all of them.  Recomment `limit 10`.)

```
workshop> select * from book limit 10;
```

### Exercise

You see that each book lists its author by name.  How many books were written by Genevieve Zemlak?

[In the query below, note that we're asking for the `count(*)` instead of the data.  That's because the data set is so large that returning the results to a CLI would be impractical.]


```
workshop> select count(*) from book where author_name = 'Genevieve Zemlak';
```

How many were written by anybody named Ann (or whose name starts with “Ann”)?
```
workshop> select count(*) from book where author_name like 'Ann%';
```


How long did it take to get the results?  Why?  

### Explanation

In our test cluster, the query took 3.3s to complete.  The `explain` plan for the query will show us how the query executed, and why it took so long.

Reminder: we read query plans from the bottom up.  The first operation in the execution of this query was the scan.  The `table: book@book_pkey` line tells us the query used the `book` table's primary key to scan the table, and that it would have to read 10M rows.

```
workshop> explain select count(*) from book where author_name like 'Ann%';

  • group (scalar)
  │ estimated row count: 1
  │
  └── • filter
      │ estimated row count: 6
      │ filter: author_name LIKE 'Ann%'
      │
      └── • scan
            estimated row count: 10,000,000
            table: book@book_pkey
            spans: FULL SCAN
```

The query is having to perform a `FULL SCAN` of the table, reading every one of 10 million records, to find the ones that match the query parameters.  We address this by creating an index on `author_name`.  That creates a key value that contains the author's name, allowing the query to quickly scan keys - without reading each row - and finding the rows it's looking for.  

### Exercise

Add an index on `author_name`.  Note how long that operation takes.
```
workshop> create index on book(author_name);
CREATE INDEX

Time: 33.684s total (execution 33.683s / network 0.000s)
```

Creating the index wrote 10M new entries into the database keyspace.  In the example above, it took 33 seconds, which doesn't seem all that long.  But remember that writing and updating keys takes time; creating and maintaining indexes carries a cost.  We'll get into that later in this workshop.

Re-run the query and note how long it takes to complete.  Look at the `explain` plan to see what index the query used for the scan and how many rows it has to read.

```
workshop> select count(*) from book where author_name like 'Ann%';
  count
---------
  36700
(1 row)

Time: 22ms total (execution 22ms / network 0ms)
```

22ms is a small fraction of the time it took to execute the same query without the index.

### Conclusion

A filter (`WHERE` clause) in your query provides a value for a given column that the query will need to compare for all the rows in the database.  If the query can find that value in a key, rather than having to read each row of the table to get it, then the query can execute much more quickly.  How much more quickly, of course, is a function of several things: how many rows there are in the table, and what fraction of them have contain the value you're looking for.

[Next Section](cover.md)
