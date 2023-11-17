# Optimizing SQL Queries

I often see engineers instinctively adding an index whenever they encounter a slow query. However, this isn't always the right solution. Let's dive into a more nuanced approach to optimizing complex queries in postgres.

## Table of Contents

- [Optimizing SQL Queries](#optimizing-sql-queries)
  - [Table of Contents](#table-of-contents)
  - [Start with the Query Plan](#start-with-the-query-plan)
  - [Understanding Basic Query Plans](#understanding-basic-query-plans)
  - [Advanced Query Plans: Inside Out](#advanced-query-plans-inside-out)
  - [Mapping SQL Queries to Query Plans](#mapping-sql-queries-to-query-plans)
  - [Debugging Complex Query Plans](#debugging-complex-query-plans)
  - [Understanding IO in SQL Query Execution](#understanding-io-in-sql-query-execution)
  - [When to Consider Indexes](#when-to-consider-indexes)
  - [Join Strategies and Performance](#join-strategies-and-performance)
  - [JSONB and TOAST](#jsonb-and-toast)



## Start with the Query Plan

The first step in query optimization is understanding the query plan. [**\`EXPLAIN ANALYZE\`**](https://www.postgresql.org/docs/current/sql-explain.html) is an invaluable tool for this. While **\`EXPLAIN\`** provides estimates, **\`ANALYZE\`** actually runs the queries. For read queries, always use **\`EXPLAIN ANALYZE\`**. For write queries, do the same within a transaction, like this: [**\`BEGIN; [queries]; ROLLBACK;\`**](https://www.postgresql.org/docs/current/sql-begin.html).


## Understanding Basic Query Plans

Here’s an example of a basic query, its corresponding explain query plan, and its explain analyze query plan:

```sql
SELECT * FROM users WHERE age > 30;
```

```sql
Seq Scan on users  (cost=0.00..35.50 rows=3000 width=204)
  Filter: (age > 30)
```

```sql
Seq Scan on users  (cost=0.00..100.00 rows=3000 width=204) (actual time=0.012..0.015 rows=500 loops=1)
  Filter: (age > 30)
  Rows Removed by Filter: 2500
Planning Time: 0.050 ms
Execution Time: 0.030 ms
```

The thing that probably jumps out to you is the execution time and the costs.

[Costs in a query plan are relative measures, not absolute.](https://www.postgresql.org/docs/current/runtime-config-query.html#RUNTIME-CONFIG-QUERY-CONSTANTS) They help you understand the efficiency of different parts of your query. Also from the docs: a sequential page read is 1.0, a random page read is 4.0, processing a row is 0.1, etc.

postgres calculates these costs based on the table's [STATISTICS](https://www.postgresql.org/docs/current/planner-stats.html), which are estimates of data distribution and are refreshed through [manual](https://www.postgresql.org/docs/current/routine-vacuuming.html#VACUUM-FOR-STATISTICS) or [auto-vacuum processes](https://www.postgresql.org/docs/current/routine-vacuuming.html#AUTOVACUUM). There is a formula that auto-vacuum follows (`vacuum threshold = vacuum base threshold + vacuum scale factor * number of tuples`) and you can tune it with these two parameters:

- [autovacuum_vacuum_scale_factor](https://postgresqlco.nf/doc/en/param/autovacuum_vacuum_scale_factor/)
- [autovacuum_vacuum_threshold](https://postgresqlco.nf/doc/en/param/autovacuum_vacuum_threshold/)

Peronsally I like to use [pg_cron](https://github.com/citusdata/pg_cron) and on a weekly basis run `VACUUM ANALYZE` to clean up any dead rows and update the statistics tables in order to help the query planner out.

This query plan is pretty simple to read and I want to look at something more complicated but before we get there let's dive in-depth a bit more. One thing that goes a long way is simply copying and pasting the plan into an LLM. This way you can essentially talk to the plan and poke and prod it to test your understanding. This is what chatGPT gave me for the above plan:

1. **Seq Scan on users**: The database performs a sequential scan on the users table. This means it reads every row in the table to check if it meets the filter criteria.

2. **Cost and Rows Estimate:**

* **cost=0.00..100.00**: Estimated cost of the operation. Here, 0.00 is the start-up cost, and 100.00 is the total cost after completing the scan.
* **rows=3000**: Estimated number of rows in the table.
* **width=204**: Estimated average width of each row in bytes.

3. **Filter:**

* **Filter: (age > 30)**: The filter condition applied to each row. Only rows where the age column is greater than 30 are selected.


4. **Rows and Filtering:**

* **Rows Removed by Filter: 2500**: Indicates that out of the estimated 3000 rows, 2500 were removed because they did not meet the filter condition.
* **actual time=0.012..0.015**: The actual time taken to perform the scan and filter operation.
* **rows=500**: Actual number of rows that met the filter condition.

5. **Planning and Execution Time:**

* **Planning Time: 0.050 ms**: Time taken to plan the query execution.
* **Execution Time: 0.030 ms**: Actual time taken to execute the query.


## Advanced Query Plans: Inside Out

Consider this complex query and its plan:

```sql
SELECT departments.name, COUNT(employees.id) AS total_employees, AVG(employees.salary) AS average_salary
  FROM departments
  JOIN employees ON departments.id = employees.department_id
  WHERE employees.hire_date > '2020-01-01'
  GROUP BY departments.name
  HAVING COUNT(employees.id) > 10
  ORDER BY average_salary DESC
  LIMIT 5;
```

```sql
Limit  (cost=100.30..100.32 rows=5 width=40) (actual time=0.123..0.125 rows=5 loops=1)
  ->  Sort  (cost=100.30..100.35 rows=20 width=40) (actual time=0.122..0.123 rows=5 loops=1)
        Sort Key: (AVG(employees.salary)) DESC
        Sort Method: top-N heapsort  Memory: 25kB
        ->  HashAggregate  (cost=99.80..99.95 rows=20 width=40) (actual time=0.108..0.110 rows=20 loops=1)
              Group Key: departments.name
              Filter: (COUNT(employees.id) > 10)
              Rows Removed by Filter: 2
              ->  Hash Join  (cost=12.50..99.00 rows=200 width=12) (actual time=0.035..0.065 rows=200 loops=1)
                    Hash Cond: (employees.department_id = departments.id)
                    ->  Seq Scan on employees  (cost=0.00..80.00 rows=1000 width=8) (actual time=0.010..0.020 rows=1000 loops=1)
                          Filter: (hire_date > '2020-01-01'::date)
                          Rows Removed by Filter: 800
                    ->  Hash  (cost=8.20..8.20 rows=20 width=4) (actual time=0.015..0.015 rows=20 loops=1)
                          Buckets: 1024  Batches: 1  Memory Usage: 9kB
                          ->  Seq Scan on departments  (cost=0.00..8.20 rows=20 width=4) (actual time=0.005..0.007 rows=20 loops=1)
Planning Time: 0.200 ms
Execution Time: 0.170 ms
```

To understand this plan, start in the middle and work your way outwards, similar to reading Lisp code. This method reveals how postgres transforms your SQL query into an executable plan.
The limit is actually the last thing that happens. The first thing that happens is a sequential scan on **\`departments\`**. Let's just go through this step by step.

1. **Seq Scan on departments**: The database starts by scanning the departments table.

2. **Hash**: A hash table is created for the departments table to facilitate the upcoming hash join.

3. **Seq Scan on employees**: Sequential scan on the employees table. Filter: hire_date > '2020-01-01', filtering employees hired after this date.

4. **Hash Join**: Performs a hash join between the employees and departments tables. Hash Cond: employees.department_id = departments.id, the condition used for joining.

5. **HashAggregate**: Aggregates the results from the join. Group Key: departments.name, groups results by the department name. Filter: COUNT(employees.id) > 10, applying the HAVING condition.

6. **Sort**: Sorts the aggregated results. Sort Key: (AVG(employees.salary)) DESC, sorting by average salary in descending order. Sort Method: top-N heapsort, an efficient sorting algorithm for top N results.

7. **Limit**: Applies a limit to the sorted result.

## Mapping SQL Queries to Query Plans

So how do we conceptualize a sql query to a query plan. I think [this article](https://dev.to/kanani_nirav/secret-to-optimizing-sql-queries-understand-the-sql-execution-order-28m1) does a great job.

<img src="https://github.com/ekeric13/sql-query-optimization-guide/assets/6489651/7d80c226-b1de-4d49-8617-f2b6ce31c608" alt="diagram" height="500"  />

I will say a query plan may apply WHERE clauses before joins, and you want to reduce the amount of data you are joining with by as much as possible. So this isn't completely accurate and that is something to look at for in your query plan.

## Debugging Complex Query Plans

Debugging a large query plan can be a lot, and if you use CTEs or views you do not get the full picture. Let's take a look at that now

```sql
WITH RecentOrders AS (
    SELECT * FROM orders WHERE order_date > '2023-01-01'
)
SELECT customer_id, COUNT(*) AS total_orders
FROM RecentOrders
GROUP BY customer_id;
```

```sql
CTE Scan on RecentOrders  (cost=100.00..200.00 rows=5000 width=50) (actual time=0.050..0.100 rows=500 loops=1)
  ->  GroupAggregate  (cost=100.00..150.00 rows=1000 width=100) (actual time=0.050..0.080 rows=100 loops=1)
        Group Key: RecentOrders.customer_id
        ->  Sort  (cost=100.00..125.00 rows=5000 width=50) (actual time=0.030..0.040 rows=500 loops=1)
              Sort Key: RecentOrders.customer_id
              Sort Method: quicksort  Memory: 25kB
              ->  Seq Scan on orders  (cost=0.00..50.00 rows=5000 width=50) (actual time=0.010..0.020 rows=500 loops=1)
                    Filter: (order_date > '2023-01-01'::date)
                    Rows Removed by Filter: 4500
Planning Time: 0.100 ms
Execution Time: 0.150 ms
```

The above plan shows operations at a high level but does not dive deeply into how the CTE's internal query (**\`SELECT * FROM orders WHERE order_date > '2023-01-01'\`**) is executed. It appears as a single step (**\`CTE Scan on RecentOrders\`**). CTEs are definitely useful for complex queries and views are definitely useful for complex DB design but they do make it harder to debug queries unfortunately.

## Understanding IO in SQL Query Execution

I have spent this time hyping up **\`EXPLAIN ANALYZE\`** but there is actually one better: **\`EXPLAIN ANAYLZE BUFFERS\`**. This gives insight into the IO of the query.

```sql
SELECT * FROM orders WHERE customer_id = 123;
```

```sql
Index Scan using customer_id_index on orders  (cost=0.29..11.95 rows=10 width=50) (actual time=0.023..0.027 rows=15 loops=1)
  Index Cond: (customer_id = 123)
  Buffers: shared hit=4
Planning Time: 0.056 ms
Execution Time: 0.049 ms
```

So the key part of this plan is **\`Buffers: shared hit=4\`**.

* **Shared**: Refers to shared buffers, which is postgres's cache for table and index data.
* **Hit**: Indicates that the required data was found in the shared buffer (cache) and did not require a disk read.
* **4**: The number of blocks in the shared buffer that were hit. This gives an idea of how much data was read from the cache.

So there are a few things going on here. Let's break them down:

1. **Cache Efficiency**: The fact that the data was found in the shared buffer (cache hit) and no read from disk was required (hit=4) indicates good cache efficiency. This reduces IO demand and speeds up query execution.
2. **Index Usage**: Utilization of an index, as seen in the plan, typically reduces the amount of IO needed as the database can quickly locate the desired rows without scanning the entire table.
3. **Buffer Hits vs Reads**: If there were disk reads, the plan would show read alongside hit. A higher number of reads might suggest the need for more memory allocation to postgres or adjustments in query/index design to improve cache usage.

So this leads us back to where we started. Indexes. But first some heuristics on when you might run into a large amount of IO in a query

1. **Large Table Scans**: Queries that scan large tables, especially without the use of indexes, can incur high IO due to the need to read large amounts of data from disk.

2. **Complex Sorting**: Sorting large datasets, or sorting based on columns that are not indexed, can lead to significant IO, especially if the sort operation spills over to disk.

3. **Large or Complex Joins**: Joins that involve large tables or that cannot efficiently use indexes might require substantial disk reads.

4. **Aggregations on Large Datasets**: Aggregating data over large datasets, particularly without the aid of indexes, can be IO-intensive.

In my experience it is the sorts and group bys that get engineers in the most trouble.

Some easy ways to fix this issue?

1. **Add indexes**: Query over your data more efficiently and bring less of it into memory
2. **Select only columns you need**: Bring less data into memory and potentially let the query planner use an index that it otherwise wouldn't because now it knows it can only needs to gather certain amount of data
3. **Increase work_mem**: **[work_mem](https://postgresqlco.nf/doc/en/param/work_mem/)** is a parameter that determines amount of memory a query can use.
4. **Increase shared_buffers**: **[shared_buffers](https://postgresqlco.nf/doc/en/param/shared_buffers/)** is a parameter that determines the size of your buffers (cache).


## When to Consider Indexes

Before adding indexes, it's essential to determine if they are necessary. postgres often knows the best index type, with B-tree being the most common. There are a [whole host of them though](https://www.postgresql.org/docs/current/indexes-types.html). B-Tree is very versatile since it can be used for `=`, `>`, `<`, etc. Hash index can only be used for `=`. The other indexes are more niche, like the GIN index that you might use for text search.

Let's say you have a B-tree index and postgres uses it. What does that mean? Well a B-tree is essentially the opposite of a binary tree where instead of extremely slender it is very bushy. And it is a denormalization of your data in a data structure that you can traverse without much IO. The nodes are pointers that tell you where the rowId you are looking for is... and the leafnode is a pointer to the table within the database that actually has your row data.

<img src="https://github.com/ekeric13/sql-query-optimization-guide/assets/6489651/8c2ebe7c-9d7b-48b9-aa25-50d8d2a098de" alt="diagram" height="350" style="background-color: white;" />


As mentioned, indexes are a form of denormalization. So the tradeoff here is that for writes you are making multiple writes (to your table but also to all effected indexes).

I do want to say outside of FKs I generally wait to see if I really need an index by running this query:

```sql
SELECT
  relname, seq_scan-idx_scan AS too_much_seq,
  case when seq_scan-idx_scan>0 THEN 'Missing Index?' ELSE 'OK' END,
  pg_relation_size(relid::regclass) AS rel_size, seq_scan, idx_scan
    FROM pg_stat_all_tables
    WHERE schemaname='public' AND pg_relation_size(relid::regclass)>80000
      ORDER BY too_much_seq DESC;
```

Here are some general heursitics to use when deciding what to index. Emphasis on compound indexes since those are more tricky:

1. **Large sequential Scans**

* If the query plan shows a sequential scan over a large amount of rows than an index will speed it up

2. **Column Usage in Queries**
* **Frequent Filters**: Prioritize columns that are frequently used in the WHERE clause of your queries.
* **Join Conditions**: Include columns commonly used in JOIN conditions.
* **Order By and Group By**: Consider columns used in ORDER BY and GROUP BY clauses.
  
3. **Selectivity**
* **High Selectivity First**: Place columns with high selectivity (i.e., columns with a wide range of unique values) at the beginning of the index. High selectivity columns help narrow down the result set more effectively.

4. **Query Patterns**
* **Common Column Combinations**: Analyze your query patterns. If certain columns often appear together in queries, they are good candidates for a compound index.

5. **Index Column Order**
* **Order Matters**: The order of columns in a compound index is crucial. The index can only be used effectively if the query's conditions match the prefix of the index. For example, in an index on (col1, col2, col3), the index is most effective if col1, or col1 and col2, or all three columns are used in the query.

6. **Balancing Performance and Maintenance**
* **Write Performance**: More indexes can slow down write operations (INSERT, UPDATE, DELETE) as each index must be updated. Balance the need for read optimization with the potential impact on write performance.
* **Index Size**: Compound indexes are larger than single-column indexes. Ensure that the increased disk space usage and memory footprint are justified by the performance gains.

7. **Covering Indexes**
* **Include Non-Filtered Columns**: If a query frequently selects specific columns, consider including these in the index even if they are not used in filtering. This creates a covering index, allowing the query to be satisfied entirely from the index without accessing the table.

8. **Avoid Redundant Indexes**
* If you create a compound index on (col1, col2), it can serve queries filtering on col1 alone but not col2 alone. Be mindful of existing indexes to avoid redundancy.

9. **Partial Indexes for Specific Cases**
* If the queries frequently involve a specific subset of rows (e.g., only rows where col3 IS NOT NULL), consider a partial index that only indexes these rows.


One last thing I wanted to touch on was a common thing in query plans that you see is **Bitmap Index Scan**. Like look at this plan:

```sql
Bitmap Heap Scan on users  (cost=4.20..145.23 rows=1000 width=204) (actual time=0.025..0.100 rows=800 loops=1)
  Recheck Cond: (age BETWEEN 25 AND 35)
  Heap Blocks: exact=103
  ->  Bitmap Index Scan on idx_users_age  (cost=0.00..4.15 rows=1000 width=0) (actual time=0.015..0.015 rows=800 loops=1)
        Index Cond: (age BETWEEN 25 AND 35)
Planning Time: 0.050 ms
Execution Time: 0.150 ms
```

What is going on with this bitmap stuff?

Well to begin with think of a bitmap as an array of yes and nos.

<img src="https://github.com/ekeric13/sql-query-optimization-guide/assets/6489651/ef52b52a-a496-4e9e-a7c0-ea8ac461ee0e" alt="diagram" height="350" />

A bitmap index scan typically occurs when there's an efficient index to use, but the index doesn't necessarily narrow down to a very small number of rows. It's a way for postgres to efficiently handle situations where multiple rows need to be fetched based on an index.


## Join Strategies and Performance

So when postgres does a join it uses one of three strategies:

1. **Nested Loop Join**: 

* **Usage**: Typically used when joining small tables or when there's a highly selective filter condition.
* **Performance**: Can be slow for large tables as it involves looping through rows of one table and comparing them with rows of the other table (O(n²) complexity). For small tables is usually the best because of the low overhead in setting everything up.
* **Conditions**: Can be used with any operator. If one of the tables is indexed, usually that is used as the inner loop as this really reduces the amount of IO operations.

2. **Merge Join**:

* **Usage**: Effective for larger datasets where both join columns are indexed and sorted (or can be efficiently sorted).
* **Performance**: It works by simultaneously iterating through both tables, which are sorted on the join columns (O(nlogn) complexity).
* **Conditions**: Can be used with most operators, but best with the equality operator.

3. **Hash Join**:

* **Usage**: Often chosen for larger tables where one table can fit into memory (as a hash table).
* **Performance**: Creates a hash table for the smaller table in memory and then scans the larger table to find matching rows (O(n) complexity).
* **Conditions**: Only works for equality operator.

So overall the main factors that go into choosing what join algo the query planner uses are:

* **Size of the Data**: Small tables tend to lead to nested loop joins, while larger tables usually use hash join, but if they are too large than it will be merge join.
* **Memory Availability (work_mem)**: The amount of memory available can affect whether a hash join is feasible, particularly the size of the smaller table relative to [work_mem](https://postgresqlco.nf/doc/en/param/work_mem/).
* **Indexes**: Availability and suitability of indexes significantly affect the choice. Indexed columns often lead to nested loop or merge joins.
* **Data Distribution and Selectivity**: The distribution of data and the selectivity of join conditions can influence the optimizer's choice.
* **Join Conditions**: The type of join condition (e.g., equality vs. non-equality) influences the join strategy.


## JSONB and TOAST

Postgres **[TOAST](https://www.postgresql.org/docs/current/storage-toast.html)** (The Oversized-Attribute Storage Technique) is how postgres deals with data that is too large. By default a page in postgres is 8kb. So if you have a row of data that is large, specifically larger than 2kb, postgres will compress it. If it cannot compress it uses TOAST which stores your data elsewhere and just makes a pointer pointing to it. And if you are querying over a table with a lot of TOAST rows the performance is much worse. TOAST is particularly bad at dealing with updating values, as it was made with atomic data types in mind. So when you update a TOAST value it duplicates the whole thing.

That leads me to **[JSONB](https://www.postgresql.org/docs/current/datatype-json.html)**. When people use jsonb they are generally storing a large amount of data, and even though they might be updating just one part of the json in the eyes of postgres they are updating the whole thing if it is stored using TOAST.

A solution to this problem is just better database design. It might make sense to separate your jsonb data into a different reference table with a foreign key. And then you can write a query that filters down a large amount of data into a small amount of data, and afterwards joins with the data that contains the jsonb column.
