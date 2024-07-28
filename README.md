## CockroachDB Workshop: Secondary indexes for optimizing query performance

This workshop consists of a series of self-paced exercises that allow the student to demonstrate and explore the concepts we present in the CEA enablement on Schema Design and Query Best Practices for using secondary indexes to optimize query performance.

### Setup: Creating a test cluster

Exercises can be run on any cluster, though the performance costs are easier to see when executed against a geographically dispersed cluster with small nodes, such as a GCE machine type `n1-highcpu-2`.  

A cluster for exercising this workshop can be set up with roachprod:
```
export CLUSTER=my_cluster
roachprod create $CLUSTER -n 3 --geo
roachprod stage $CLUSTER release latest
roachprod start $CLUSTER
```

Bring up the DB Console with the command

```
roachprod adminui $CLUSTER:1 --open
```

The exercises can be performed from a SQL client that can connect to the cluster.  The student can use a local copy of `cockroach` if they have one, or they can remote login to one of the nodes of the cluster and communicate with the local process.

The student will need to load the accompanying CSV into the database.  We’ll be loading it from `nodelocal` on one of the cluster nodes, so we’ll need the CSV placed wherever the SQL client is running.  If the student will be remoting into one of the cluster nodes, execute the following 2 commands:

```
roachprod put library.csv $CLUSTER:1
roachprod ssh $CLUSTER:1
```

(For a more realistic setup, we could set up another server with haproxy and a copy of the cockroach SQL shell binary, but that’s unnecessary for the exercises we’re going to do.)

### Setup: Creating the test database 

Put the `library.csv` where the cluster can find it:

```
$ cockroach nodelocal upload library.csv workshop/library.csv –insecure
```

Connect to the cluster with the SQL client, and create a `workshop` database.  Within it, use the following command to create the table `book`:

```
$ cockroach sql –insecure

> create database workshop;
 
> use workshop;

workshop> create table book (
	id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
	title STRING NOT NULL,
	author_name STRING,
	price float not null,
	format string not null,
	publish_date DATE NOT NULL
);
```

This is a very simple table, and not a realistic example of a library database schema, but it serves as the basis for the set of exercises in this workshop, and will be expanded in subsequent workshops.

Import the data from the uploaded CSV, replacing the string node with the node number of the cluster node you loaded the file to.  (Note that if you’re following the instructions using `roachprod`, the node is `1`.  If you’re running a local copy of the cockroach SQL client and connecting to a remote cluster, then the `nodelocal upload` command reported which node it connected to.)

```
workshop> IMPORT INTO book (title, author_name, price, format, publish_date) CSV DATA ('nodelocal://node/workshop/library.csv');
```

Check your work.  You should have a table book with 10M entries:

```
workshop> show tables;
  schema_name |  table_name  | type  | owner | estimated_row_count | locality
--------------+--------------+-------+-------+---------------------+-----------
  public      | book         | table | root  |            10000000 | NULL
```

[Next Section](narrative/filter.md)
