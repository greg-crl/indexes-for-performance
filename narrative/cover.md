## Use Covering Indexes to avoid index joins (or worse)

This exercise introduces covering indexes.  A covering index stores additional data from each row with the index.  If the data a query needs to process and/or return is stored with the index the query is using to reduce the search span, then the query won't need to access the main table to fetch any of the data it needs.  

We’re going to explore an example of a circumstance where these issues come up.

### Exercise

Let’s answer some questions about book prices.  We were looking at the Anns earlier.  How many of them are there?

```
workshop> select count(*) from book where author_name like 'Ann%';
  count
---------
  36700

Time: 18ms total (execution 18ms / network 0ms)
```

Tens of thousands of them.  And because we have an `author_name` index, it’s fast to find them.

What’s the average price of their books?

```
workshop> select round(avg(price),2) from book where author_name like 'Ann%';
  round
---------
  50.38

Time: 181ms total (execution 181ms / network 0ms)
```

That query took 10 times as long to run as the previous query.  They should both be using the `author_name` index, so why so much longer?  

### Explanation

Here’s the explain plan.

```
workshop> explain select round(avg(price),2) from book where author_name like 'Ann%';

  • render
  │
  └── • group (scalar)
      │ estimated row count: 1
      │
      └── • index join
          │ estimated row count: 8,683
          │ table: book@book_pkey
          │
          └── • scan
                estimated row count: 8,683 
                table: book@book_author_name_idx
                spans: [/'Ann' - /'Ano')
```

The query is using the index to find the authors named Ann.  But once it’s found the right rows, it needs to read them to get the price we’ve asked for.  That’s the *index join* .  The query read the index, collected the primary key value for each of the Ann records it found, then read those records from the main table to read price.

### Another example of even worse performance

Let’s ask the same question about paperbacks that we asked about books by “Ann”:

If we’re interested in finding out things about books by format, it makes sense that we’ll need an index on format.  Before we can ask questions specifying format, we’ll need to know what our data looks like.  What are the values for format in our table, and how many of the books are of each one?

```
workshop> select distinct format, count(format) from book group by format;
   format   |  count
------------+----------
  audio     | 2502901
  paperback | 2497407
  ebook     | 2500177
  hardcover | 2499515

Time: 4.978s total (execution 4.978s / network 0.000s)
```

There are only 4 types, with about 2.5 million books of each type.   Reducing the search span to ¼ of a full scan is better than nothing.  Let’s index it.

```
workshop> create index on book(format);
```

And verify it works.

```
workshop> select count(*) from book where format = 'ebook';
   count
-----------
  2500177

Time: 1.496s total (execution 1.496s / network 0.000s)
```

As expected, that was much faster than before.  Presumably the query is using the new index.  Let’s check.

```
workshop> explain select count(*) from book where format = 'ebook';

  • group (scalar)
  │ estimated row count: 1
  │
  └── • scan
        estimated row count: 2,523,000
        table: book@book_format_idx
        spans: [/'ebook' - /'ebook']
```

Now let's ask our question about the price of paperbacks.

```
workshop> select round(avg(price),2) from book where format = 'paperback';
  round
---------
  50.49

Time: 3.457s total (execution 3.457s / network 0.000s)
```

What’s going on?  That took as long as the query before we created the index.  Let’s look at the explain plan.

```
  • render
  │
  └── • group (scalar)
      │ estimated row count: 1
      │
      └── • filter
          │ estimated row count: 2,531,000
          │ filter: format = 'paperback'
          │
          └── • scan
                estimated row count: 10,000,000 
                table: book@book_pkey
                spans: FULL SCAN
```

 It’s doing a full scan.  Why’s that?

### Explanation

When we counted the Anns, it took 18ms.  When we index-joined the Anns to get price, it took 10 times as long.  The result set for paperbacks is much larger.  Counting them took almost 1.5 seconds; if an index join takes 10 times as long, it'd be 15 seconds.  And that’s longer than the 4 or 5 seconds it takes to do a FULL SCAN of the table.  The optimizer recognized that using the index would be even more expensive than a not using it.

### Exercise

Index joins aren’t bad.  Sometimes they can’t be avoided.  But if an index join makes your query less performant that you like, it’s worth investigating whether we can avoid it.  And a full table scan - especially of very large tables - is to be avoided if at all possible. 

The obvious way to avoid an index join is if everything you want from the query is in the index.  When we were counting Anns, we only needed access to the author’s name, which was in the index; no need to fetch it from the main table.

There’s a way to specify data you want to store with the index, so that it’s available during the index scan.  That’s an index with a `STORING` clause, also called a  “covering index”.  Let’s store price with the author_name index and see the effect it has on our query about the Anns.

```
workshop> create index on book(author_name) storing (price);

workshop> select round(avg(price),2) from book where author_name like 'Ann%';
  round
---------
  50.38

Time: 22ms total (execution 21ms / network 0ms)
```
Much better.  Let’s see how that worked.

### Explanation

```
workshop> explain select round(avg(price),2) from book where author_name like 'Ann%';

  • render
  │
  └── • group (scalar)
      │ estimated row count: 1
      │
      └── • scan
            estimated row count: 8,683 
            table: book@book_author_name_idx1
            spans: [/'Ann' - /'Ano')
```

Note that this is using our new index, not the original `book_author_name_idx` that isn’t `STORING` price.  Let’s have a look at the indexes on book:

```
workshop> show index from book;
```

The output lists all the indexs we’ve defined so far, including the primary key:
* `book_author_name_idx`
* `book_author_name_idx1`
* `book_format_idx`
* `book_format_idx1`
* `book_pkey`

The multiple entries for each index show what columns they’re made up of, and which are stored columns.  Notice that the primary key is basically a covering index that stores ALL the columns of the table. 

### Exercise

Let’s apply the same technique to our format index and see how much better performing our average price calculation will be.

```
workshop> select round(avg(price),2) from book where format = 'paperback';
  round
---------
  50.49

Time: 1.320s total (execution 1.320s / network 0.000s)
```

Much better, and in line with our expectations; 1.3 seconds is about how long we’ve seen it take to scan the format index looking for paperbacks.  

### Conclusion

Using covering indexes is a best practice, but it should be used judiciously.  The data from the columns specified in the STORING clause is getting duplicated on disk.  There's a storage cost and an upkeep cost associated with the extra data.  If many columns must be covered by an index to satisfy a query, or if a column must be covered by a number of indexes to satisfy a variety of queries, then the cost may be prohibitive.

[Next Section](cardinality.md)
