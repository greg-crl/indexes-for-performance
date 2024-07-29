## Index columns with high cardinality

It’s considered a best practice to reduce your search span by indexing columns with high cardinality.

“Cardinality” in this context means the number of different values a variable can have.  Here, we mean how many different values are found in a given column of a table.  

When we index on a column, we’re dividing the table into segments whose size and number are determined by the different values of the indexed column.  Since we want to use indexes to narrow the search span when we have a known value (or values) for a column, we want those segments to be numerous and small.  So it makes sense to index on columns  with high cardinality.

### Exercise

The average paperback in our database costs $50.49.  Let’s ask a different question about price.  How many of paperbacks cost more than $75?

```
workshop> select count(*) from book where format='paperback' and price > 75;
  count
----------
  631068
(1 row)

Time: 1.215s total (execution 1.214s / network 0.000s)
```

Our index on format, storing price, cut the search span to 2.5 million rows, but the query still had to read each one to find out if the stored value was over our limit.  But in this case, there are only 630K books that match our criteria, about ¼ of the index entries we had to read.  Could we reduce the search span even further?

We can, if there were an index our query could use that defined an even smaller region that contains all the data we’re looking for.

We’re searching on a combination of format and price.  We have an index for format, but there are only 4 values for format.  (That is, the format column has a cardinality of 4.)  There are many more values for price (it has a much higher cardinality), and therefore a search on price could define a much smaller search space.


Create an index that’s the inverse of the storing index we’re using now.  Create an index on `price` STORING `format`.  Then exercise the same query and compare the performance.  What do you expect to happen?  What do you actually see?

### Explanation

```
workshop> create index on book(price) storing (format);

workshop> select count(*) from book where format='paperback' and price > 95;

  count
----------
  126263

Time: 333ms total (execution 333ms / network 0ms)
```

Much faster, 1/3 of a second instead of a second and a half.  Let’s look at the plan:

```
  • group (scalar)
  │ estimated row count: 1
  │
  └── • filter
      │ estimated row count: 126,841
      │ filter: format = 'paperback'
      │
      └── • scan
            estimated row count: 498,979 
            table: book@book_price_idx
            spans: [/95.00000000000001 - ]
```

The index on `price` produced a search span of only half a million rows, much smaller than the 2.5 million row span implied by `format = ‘paperback’`.

### Conclusion

We create indexes to serve queries.  Often, we create a particular index to improve the performance of a specific query.  It makes sense to define indexes that offer the smallest search span for the columns your query filters on.  Knowing the cardinality of the columns in your query makes it easier to define indexes that most improve your query's performance.


[Next Section](compound.md)
