
---

copyright:
  years: 2018, 2021
lastupdated: "2021-03-25"

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}

# SQL reference

## Introduction
{: #chapterIntroduction}

With {{site.data.keyword.sqlquery_full}}, you can analyze and transform open data with SQL. It supports the various types of SELECT statements from the ANSI SQL standard.

The SELECT statement (or query statement) is used to read object data from {{site.data.keyword.cos_full}} (COS),
process the data, and store it back on Cloud {{site.data.keyword.cos_short}} eventually.

You can use {{site.data.keyword.sqlquery_short}} as a data transformation service, as it always writes the results of a query to a specified location in either {{site.data.keyword.cos_short}} or Db2 tables.
{{site.data.keyword.sqlquery_short}} provides extended SQL syntax inside a special INTO clause to control how the result data is stored physically. This includes control over data location, format, layout, and partitioning.

A query statement can be submitted through {{site.data.keyword.sqlquery_short}}'s web UI or programmatically,
either by using the service's REST API, or by using the Python or Node.JS SDK. You can also use {{site.data.keyword.DSX_full}}
and the Python SDK to use {{site.data.keyword.sqlquery_short}} interactively with Jupyter Notebooks. In addition, you can submit SQL queries that use {{site.data.keyword.openwhisk}}.

In addition to the ad hoc usage of data in {{site.data.keyword.cos_full}}, you can also register and manage your data in a catalog as tables, consisting of columns and partitions.

Several benefits to cataloging your data exist:

- It simplifies SQL SELECT statements because the SQL author does not need not know and specify exactly where and how the data is stored.
- The SQL execution can skip the inference of schema and partitioning because this information is available in the metastore. This can improve you query performance, especially for text-based data formats, such as CSV and JSON, where the schema inference requires a full scan of the data before the actual query execution.
<!-- Hide - With the *ANALYZE TABLE* command, you can gather statistics about your data, which is then used by the SQL compiler to do a cost-based optimization of the query plan, which can result in significantly improved query performance for queries on larger data volumes. -->

## Select
{: #chapterSQLQueryStatement}

See the following examples for an outline of the general syntax of an SQL query statement that uses the `query` clause and the `namedQuery` clause.

<h3 id="query">query</h3>

<!--include-svg src="./svgfiles/query.svg" target="./diagrams/query.svg" alt="syntax diagram for a query" layout="@break@" -->

<h3 id="namedQuery">namedQuery</h3>

<!--include-svg src="./svgfiles/namedQuery.svg" target="./diagrams/namedQuery.svg" alt="syntax diagram for a named query" layout="" -->

<h3 id="intoClause">intoClause</h3>

<!--include-svg src="./svgfiles/intoClause.svg" target="./diagrams/intoClause.svg" alt="syntax diagram for an INTO clause" layout="" -->

The query statement supports *common table expressions*. A common table expression permits defining a result table with a table name
that can be specified as a table name in any FROM clause of the fullselect that follows.

Common table expressions are defined by using the reserved keyword `WITH` followed by one or more *named queries*.
Each common table expression that is specified can also be referenced by name in the FROM clause of subsequent common table expressions.

Creating a common table expression avoids the overhead of creating and dropping an intermediate result object on
Cloud {{site.data.keyword.cos_short}} that is needed only for a certain query.

Moreover, a common table expression is beneficial when the same result table must be shared in a fullselect.

<h3>Examples</h3>

The common table expression examples use values clauses to define tables inline. For more information about the values clause, see [valuesClause](#valuesClause).

```sql
-- find the department with the highest total pay
WITH dtotal AS (
    SELECT
        col1 AS deptno,
        SUM(col3+col4) AS totalpay
    FROM  VALUES -- deptno, empid, salary, bonus
        (1, 1, 1000, 0), (1, 2, 2000, 500), (1, 3, 3000, 0),
        (2, 4, 5000, 200), (2, 5, 6000, 0), (2, 6, 4000, 0),
        (3, 7, 2000, 500), (3, 8, 2000, 500), (3, 9, 8000, 0)
    GROUP BY col1
)
SELECT deptno
FROM dtotal
WHERE totalpay = (SELECT MAX(totalpay) FROM  dtotal)
```
{: codeblock}

| DEPTNO |
| ------ |
| 2 |
<!--table-caption title="Query result for example 'find the department with the highest total pay'"-->

```sql
-- list products with a price above the average price
WITH  products AS (
    SELECT
        col1 AS productid,
        col2 AS price
    FROM VALUES -- productid, price
        (1, 10), (2, 20), (3, 30), (4, 40),
        (5, 5), (6, 10), (7, 15), (8, 40),
        (9, 100), (10, 200), (11, 300), (12, 400)
), avg_product_price AS (
    SELECT
        AVG(price) as avg_price
    FROM products
)
SELECT
    productid,
	price,
	(SELECT * FROM avg_product_price) AS avg_price,
	price-(SELECT * FROM avg_product_price) AS price_dist
FROM products WHERE price > (SELECT * FROM avg_product_price)
```
{: codeblock}

| PRODUCTID | PRICE | AVG_PRICE | PRICE_DIST |
| --------- | ----- | --------- | ---------- |
| 9         | 100   |  97.5     | 2.5        |
| 10        | 200   |  97.5     | 102.5      |
| 11        | 300   |  97.5     | 202.5      |
| 12        | 400   |  97.5     | 302.5      |
<!--table-caption title="Query result for example 'list products with a price above the average price'"-->

<h3 id="cosResultClause">cosResultClause</h3>

You can use the Cloud {{site.data.keyword.cos_short}} result clause to apply detailed control over the location, format, and layout of the SQL query result set being stored on Cloud {{site.data.keyword.cos_short}}.

The default is `JOBPREFIX JOBID`, which means that `jobid=` is always appended to the target prefix.
You can optionally specify `JOBID NONE`, which skips the appending of `jobid=`. This means that the results are written
exactly to the requested path. However, if that path contains existing objects (for example, from a previous query execution with the same target path),
all existing objects get removed when the new result objects are written.

A query cannot run if `JOBPREFIX NONE` is specified and the target overlaps with at least one of the input URIs.
The reason is that data cannot be overwritten by a query that is reading that same data.
For example, `SELECT * FROM cos://us-geo/mybucket/myprefix/mysubprefix INTO cos://us-geo/mybucket/myprefix JOBPREFIX NONE`
 returns an error when you try to submit it.

In the *COS result clause*, you can explicitly specify the storage location and type of a query result on Cloud {{site.data.keyword.cos_short}}.
The storage location is specified by a `COSURI`.

Valid query result object formats are: `AVRO`, `CSV`, `JSON`, `ORC`, and `PARQUET`. Object type names are not case-sensitive.

Being able to explicitly specify the location and the type of the result object enables a user to take the following actions:
* Read *raw* data stored in one format, for example, CSV.
* Cleanse the data by using SQL constructs.
* Store the cleansed data in a format that offers benefits regarding query performance and storage consumption on Cloud {{site.data.keyword.cos_short}}, for example, a column-based object format, such as Parquet.

Moreover, a user can explicitly define the way a query result is stored physically on Cloud {{site.data.keyword.cos_short}} by using the *result partitioned clause*.

As shown in the syntax diagrams, three main use cases exist to define the physical layout of a query's result on Cloud {{site.data.keyword.cos_short}}:

- Partition by columns, that is so-called Hive-style partitioning.
- Partition into buckets or objects (both terms can be used synonymously), that is, generate the query result into objects, with or without specifying columns.
- Partition by number of rows.

A partition is an object on Cloud {{site.data.keyword.cos_short}} that is potentially a part of an aggregated object.
The presence of multiple partitions allows for parallel input/output (I/O) during query execution. If no *result partitioned clause* is specified,
the query result is stored in a single partition on Cloud {{site.data.keyword.cos_short}}.

<!--include-svg src="./svgfiles/cosResultClause.svg" target="./diagrams/cosResultClause.svg" alt="syntax diagram for a COS result clause" layout="@break@" -->

<h3 id="partitionedClause">partitionedClause</h3>

You can use the result partitioned clause to control the layout of the SQL query result set being stored. The default behavior is to store the result into one single partition, that is a single object in Cloud {{site.data.keyword.cos_short}}.

<!--include-svg src="./svgfiles/partitionedClause.svg" target="./diagrams/partitionedClause.svg" alt="syntax diagram for a result partitioned clause" layout="@break@" -->

<h3 id="sortClause">sortClause</h3>

This clause can be used to sort in many ways. When specified in combination with PARTITIONED BY, it sorts the rows within each partition
by the sort order that is specified in the SORT BY clause. When specified in combination with PARTITIONED INTO, the same is done,
which is often referred to as clustering the rows by the specified columns into the fixed number of partitions specified by PARTITIONED INTO.
When specified without the PARTITIONED clause, it is equivalent to an ORDER BY clause specified at the top level of the SQL SELECT statement.
The ORDER BY clause is ignored, as soon PARTITIONED INTO is specified.

<!--include-svg src="./svgfiles/sortClause.svg" target="./diagrams/sortClause.svg" alt="syntax diagram for a result partitioned column clause" layout="@break@" -->


<h4>Partition by columns</h4>

When you use the `PARTITIONED BY (column-list)` clause without specifying `INTO x BUCKETS/OBJECTS`, you can store the query result
by using Hive-style partitioning, that is, to create partitions that contain only rows that have certain values for one or more columns.
Chose this physical layout if the stored object is further analyzed by using SQL queries that specify predicates on the partition columns.

For example, a result object that contains worldwide order data has a column `country` to represent the country that the order is initiated from.
Partitioning the result object by the column `PARTITIONED BY (country)`, would create a result object with a partition for each country present in the query result.

When the result object is stored this way on Cloud {{site.data.keyword.cos_short}}, each SQL query that contains a predicate,
such as `country = 'USA'` or `country in ('MALTA', 'ITALY', 'VATICAN CITY')`, benefits from this physical layout. The reason is that during
SQL query execution partitions must only be read if they contain data for the countries of interest. This tremendously cuts down the I/O traffic of the SQL query.

Some additional remarks on Hive-style partitioning:

- Hive-style partitions have an eye-catching naming scheme because the column names that are used for partitioning are part of the partition object prefix, for example, `/order/COUNTRY=USA/part-m-00000.snappy.parquet`.
- Hive-style partitions do not contain any values for partition columns since their values are *stored* in the object prefix of the partition.
Thus, if you copy a HIVE-style partition and rename the object prefix by removing the partition column values, you lose data.
- Hive-style partitions have a tendency for data skewing. For example, the partition that represents order data from Malta is likely much smaller
than the partition that represents order data from the US. You can partition the query result into separate objects if you want to have *equally sized* partitions.


<h4>Partition by columns into objects</h4>

By partitioning a query result into objects you can specify the exact number of *equally sized* result partitions. This lets you experimentally fine-tune the number of objects to meet certain criteria for partition size. To specify the number of partitions, use the `PARTITIONED INTO x BUCKETS/OBJECTS` clause.

For example, knowing the size of the query result, it is possible to calculate the number of objects to end up with partitions that have a certain size. For example, 128 MB, that is the Hadoop default file size, or any other size that meets application requirements.

The `INTO x BUCKETS/OBJECTS` clause can be combined with the `BY (column-list)` clause to create a certain number of partitions that support data affinity regarding specified partition columns.

Continue with the preceding example that specifies `PARTITION BY (customerid) INTO 10 OBJECTS` stores the query result into 10 objects ensuring that all data for a customer
is stored in the same partition. Although it is ensured that all data for a certain customer is stored in the same partition,
it is not ensured that the data is physically sorted according to the specified column.

<h4>Partition by number of rows</h4>

By partitioning by number of rows you can specify the number of rows that go into a single partition. To specify the number of rows stored in each partition,
use the `EVERY x ROWS` clause. In case the row length of the result object is not varying heavily, with the *partition every rows* clause you can also create *almost equally sized* result partitions.

The use of the `PARTITIONED EVERY x ROWS` clause on a sorted query result ensures that partitions are created to have some rows that are sorted according to the query's `SORT BY` clause. This physical layout can be useful to create partitions that are processed by an application in a pagination manner, for example, browsing order data sorted by *order date* and *customer ID*.

The use of `PARTITIONED EVERY x ROWS` clause causes data to be written single-threaded to Cloud {{site.data.keyword.cos_short}}, which means that no parallel I/O is performed to write query results to Cloud {{site.data.keyword.cos_short}}.

<h3 id="dbResultClause">dbResultClause</h3>

You can use the Db result clause to specify that you want query results to be stored as a relational database table in {{site.data.keyword.Bluemix_notm}}.
Currently, {{site.data.keyword.Db2_on_Cloud_long}} is the only supported target database.

Storing query results in a database creates a new table with the columns that are determined by the query result. When you write to Db2,
the following type-mapping rules apply:
 * String types are mapped to VARCHAR(32000) columns.
 * Struct types are not mapped and must be flattened first. See the `FLATTEN` [table transformation function](#tableTransformer).
 * Arrays, time series, and spatial data types are not mapped and must be converted with appropriate SQL functions.

The table name and optional schema are specified as part of the target URI.
**Important**: If a table with the name that is indicated exists in the target database, that table is dropped before the query executes
and all existing data is deleted.

Use the `PARALLELISM x` clause to specify that multiple parallel database connections are to be opened to write out
the result. Depending on the size of your result and the network connectivity of your target database service,
this can reduce the query processing time significantly.

<!--include-svg src="./svgfiles/dbResultClause.svg" target="./diagrams/dbResultClause.svg" alt="syntax diagram for a Db2 result clause" layout="@break@" -->

<h3 id="accessSecrets">accessSecrets</h3>

By default, either the credentials that are needed to access the target database are taken from the credentials object of a `CRN_URI`, or the IAM user
submitting the statement is used to connect to the `DB2_TABLE_URI`.
You can override this default by specifying either  a combination of `USER` and `PASSWORD` or an `APIKEY`. However, the password or API key is **not** included in the SQL statement as plain text. Instead, you must store it as a custom key in a {{site.data.keyword.keymanagementservicefull}} instance to which you have access.
For a description how to store and manage the secrets in {{site.data.keyword.keymanagementserviceshort}}, see [Setting up custom secrets in Key Protect](/docs/sql-query?topic=sql-query-kpsetup).

<!--include-svg src="./svgfiles/accessSecrets.svg" target="./diagrams/accessSecrets.svg" alt="syntax diagram for a Db2 result clause" layout="@break@" -->

<h3>More topics</h3>

For more information about clauses that are used in a *query*, see the following topics:
* [COSURI](#COSURI)
* [CRN_URI](#CRN_URI)
* [DB2_TABLE_URI](#DB2_TABLE_URI)
* [fullselect](#fullselect)
* [identifier](#identifier)
* [namedQuery](#namedQuery)
* [query](#query)

<h3>Related references</h3>

A *query* is referenced by the following clauses:
* [booleanExpression](#booleanExpression)
* [namedQuery](#namedQuery)
* [predicate](#predicate)
* [primaryExpression](#primaryExpression)


### Fullselect Clause
{: #chapterFullSelectClause}

<h3 id="fullselect">fullselect</h3>

A *fullselect* is the core component of a *query*. It is the only mandatory general component for a valid query statement. The other components outside of *fullselect* are optional. Its syntax is defined by the following syntax diagram.

<!--include-svg src="./svgfiles/fullselect.svg" target="./diagrams/fullselect.svg" alt="syntax diagram for a fullselect" layout="@break@" -->

The result set defined by a single fullselect can be combined with the result set of one or more other fullselects by using set operators.

The following set operators are supported and each set operator derives a result set from two other result sets R1 and R2:
* `INTERSECT`:  The result consists of all rows in **both** R1 and R2.
* `UNION`: The result consists of all rows in R1 and all rows in R2.
* `EXCEPT`: The result consists of all rows that do not have a corresponding row in R2.
* `MINUS`: The minus operator is a synonym for the except operator and is supported for compatibility with other SQL implementations.

These set operators can be further refined by using the following modifiers:
* `DISTINCT`: This modifier ensures that the overall result set does not contain any duplicates. This is the default modifier that applies if no modifier is present.
* `ALL`: All rows of a fullselect's result set are combined by using a set operator. Thus, the overall result set can contain duplicates.

The following combinations of set operators and set modifiers are not supported:
* `INTERSECT ALL`
* `EXCEPT ALL`
* `MINUS ALL`

The characteristics of a result set defined by a fullselect can be further defined by using the following clauses:
* `ORDER BY`: Define an overall ordering of the result set based on the criteria that are defined by the list of `sortItem` clauses.
The default order direction is ascending if not explicitly specified by a `sortItem` clause. The *order by* clause
cannot be used with *cluster by*, *distribute by* or *sort by* clause.
When you use partitioned output, the `ORDER BY` clause gets ignored. Use the `sortClause` instead.
* `DISTRIBUTE BY`: Distribute result set rows into new partitions based on the criteria that are defined by the list of `expression` clauses.
Result set rows that have the same expression values are moved to the same partition. The *distribute by* clause cannot be
used with *order by* or *cluster by* clause.
* `SORT BY`: You can define an overall ordering on a partition base as defined by the list of `expression` clauses.
The default order direction is ascending if not explicitly specified by an `expression` clause.  The *sort by* clause is used in conjunction
with the *distribute by* clause.
* `CLUSTER BY`: Distribute result set rows into new partitions based on the criteria that are defined by the list of `expression` clauses. Moreover, each
partition is sorted in ascending order based on the criteria that are defined by the set of `expression` clauses. Thus, this clause represents a shortcut
for *distribute by* clause combined with *sort by* in ascending order. You cannot use the *cluster by* attribute with the *order by*,
*distribute by*, *sort by* clause.
* `LIMIT`: Restrict the number of rows that are returned from the result set of the fullselect. The number of rows can be defined by an `expression` or by using
the keyword `ALL` that causes all rows to be returned.

`DISTRIBUTE BY`, `SORT BY` and `CLUSTER BY` have an effect only during your SQL query execution and do not influence the query result
written back to Cloud {{site.data.keyword.cos_short}}. Use these clauses only in execution of subqueries to optimize the outer
query execution that works on the intermediate result sets produced by the sub queries. To define the persistent target of the overall query
that is written back to Cloud {{site.data.keyword.cos_short}}, you need to use the dedicated [resultClause](#resultClause) instead.

<h3>Examples</h3>

The set operator examples use values clauses to define result sets for the set operations. For more information about the values clause, see [valuesClause](#valuesClause).

```sql
-- set union eliminating duplicate rows
SELECT * FROM VALUES 1, 2 ,3 UNION VALUES 1, 2, 3
```
{: codeblock}

```sql
-- set union eliminating duplicate rows
SELECT * FROM VALUES 1, 2 ,3 UNION DISTINCT VALUES 1, 2, 3
```
{: codeblock}

The result of the example queries is shown in the following table.

| COL1 |
| ---- |
|1     |
|3     |
|2     |
<!--table-caption title="Query result for example 'set union eliminating duplicate rows'"-->

```sql
-- set union with duplicate rows
SELECT * FROM VALUES 1, 2 ,3 UNION ALL VALUES 1, 2, 3
```
{: codeblock}

The result of the example query is shown in the following table.

|COL1|
|----|
|1   |
|2   |
|3   |
|1   |
|2   |
|3   |
<!--table-caption title="Query result for example 'set union with duplicate rows'"-->

```sql
-- intersecting set eliminating duplicate rows
SELECT * FROM VALUES 1, 2 ,3 INTERSECT VALUES 2, 2, 3, 3, 3
```
{: codeblock}

```sql
-- intersecting set eliminating duplicate rows
SELECT * FROM VALUES 1, 2 ,3 INTERSECT DISTINCT VALUES 2, 2, 3, 3, 3
```
{: codeblock}

The result of the example queries is shown in the following table.

|COL1|
|----|
|3   |
|2   |
<!--table-caption title="Query result for example 'intersecting set eliminating duplicate rows'"-->

```sql
-- Difference quantity eliminating duplicate rows
SELECT * FROM VALUES 1, 2 ,3 EXCEPT VALUES 2, 2, 3, 3, 3
```
{: codeblock}

```sql
-- difference quantity eliminating duplicate rows
SELECT * FROM VALUES 1, 2 ,3 EXCEPT DISTINCT VALUES 2, 2, 3, 3, 3
```
{: codeblock}

```sql
-- difference quantity eliminating duplicate rows
SELECT * FROM VALUES 1, 2 ,3 MINUS VALUES 2, 2, 3, 3, 3
```
{: codeblock}

```sql
-- difference quantity eliminating duplicate rows
SELECT * FROM VALUES 1, 2 ,3 MINUS DISTINCT VALUES 2, 2, 3, 3, 3
```
{: codeblock}

The result of the example queries is shown in the following table.

|COL1|
|----|
|1   |
<!--table-caption title="Query result for example 'difference quantity eliminating duplicate rows'"-->

<h3>More topics</h3>

For more information about the clauses that are used in a *fullselect*, see the following topics:
* [expression](#expression)
* [fullselect](#fullselect)
* [namedWindows](#namedWindows)
* [simpleselect](#simpleselect)
* [sortItem](#sortItem)
* [valuesClause](#valuesClause)

<h3>Related references</h3>

A *fullselect* is referenced by the following clauses:
* [fullselect](#fullselect)
* [query](#query)
* [relationPrimary](#relationPrimary)

### Simpleselect Clause
{: #chapterSimpleSelectClause}

A *simpleselect* is a component of a *fullselect*. Its syntax is defined by the following syntax diagram.

<h3 id="simpleselect">simpleselect</h3>

<!--include-svg src="./svgfiles/simpleselect.svg" target="./diagrams/simpleselect.svg" alt="syntax diagram for a simpleselect" layout="@break@"-->

With a *simpleselect*, you can specify the following characteristics of a result set:
* The list of *result columns* from *relations* or *lateral views* that are part of the final result set. The result column list can be further redefined by using the following modifier keywords:
    * `DISTINCT`: Eliminates all but one of each set of duplicate rows of the final result set.
    * `ALL`: Retains all rows of the final result set, and does not eliminate redundant duplicates. This is the default.
* The `FROM` clause defines the list of *relations* or *lateral views* that are combined to derive a result set.
* The `WHERE` clause defines the way how *relations* and *lateral views* are filtered and **joined** to derive a result set.
* In its simplest form, the `GROUP BY` clause defines how rows that qualify for the final result set are grouped based on *grouping expressions*. Each group is represented by a single row in the final result set.
* The `HAVING` clause is used with the *group by clause* to filter out groups from the final result set.

<h3 id="resultColumn">resultColumn</h3>

<!--include-svg src="./svgfiles/resultColumn.svg" target="./diagrams/resultColumn.svg" alt="syntax diagram for a result column" layout="" -->

A *result column* can be any expression that can optionally be associated with an identifier, that is, a *new name*. By providing custom identifiers, you can control the column names that are used in the result data set written to Cloud {{site.data.keyword.cos_short}}.
See the following examples for such expressions:
* A column name from one of the relations.
* An expression that uses column names from one or multiple relations.
* Arithmetic expressions that perform calculations.

<h3>Group By Clause</h3>

<h4 id="groupByClause">groupByClause</h4>

<!--include-svg src="./svgfiles/groupByClause.svg" target="./diagrams/groupByClause.svg" alt="syntax diagram for a group by clause" layout="@break@" -->

<h4 id="groupingSet">groupingSet</h4>

<!--include-svg src="./svgfiles/groupingSet.svg" target="./diagrams/groupingSet.svg" alt="syntax diagram for a grouping set" layout="" -->

More complex *group by* clauses use so called *grouping sets* to provide more insights into the set of rows grouped by a grouping expression.

Grouping sets are best explained by using examples. The following set examples use *values clauses* to define result sets for group by operations. For more information about the values clause, see [valuesClause](#valuesClause).

In the following example SQL query sales data is grouped per year and quarter.

```sql
-- grouping sales data per year and quarter
SELECT
    sales.col1 AS year,
    sales.col2 AS quarter,
    SUM(sales.col3) AS amount
FROM VALUES
    (2017, 1 ,100),
    (2017, 1 ,50),
    (2017, 2 ,200),
    (2017, 2 ,300),
    (2018, 1 ,300),
    (2018, 1 ,100),
    (2018, 2 ,400) AS sales
GROUP BY col1, col2
ORDER BY year, quarter
```
{: codeblock}


The result of the example query is shown in the following table.

|YEAR|QUARTER|AMOUNT|
|----|-------|------|
|2017|1      |150   |
|2017|2      |500   |
|2018|1      |400   |
|2018|2      |400   |
<!--table-caption title="Query result for example 'grouping sales data per year and quarter'"-->

A ROLLUP grouping is an extension to the *group by clause* that produces a result set containing subtotal rows in addition to the *regular* grouped rows.
A `GROUP BY COL1, COL2 WITH ROLLUP` generates the following grouping sets: **(COL1, COL2)**, **(COL1)**, **()**. The **N** grouping expressions of the ROLLUP convert to **N+1** grouping sets.

Referring to the preceding example, adding a `WITH ROLLUP` modifier to the *group by* clause computes *rollup* sales data on quarter by year basis,
a yearly basis, and a grand total as shown by the following example.

```sql
-- rollup sales data on quarter by year basis, a yearly basis and a grand total
SELECT
    sales.col1 AS year,
    sales.col2 AS quarter,
    SUM(sales.col3) AS amount
FROM VALUES
    (2017, 1 ,100),
    (2017, 1 ,50),
    (2017, 2 ,200),
    (2017, 2 ,300),
    (2018, 1 ,300),
    (2018, 1 ,100),
    (2018, 2 ,400) AS sales
GROUP BY col1, col2
WITH ROLLUP
ORDER BY year, quarter
```
{: codeblock}


The result of the example query is shown in the following table.

|YEAR|QUARTER|AMOUNT|
|----|-------|------|
|null|null   |1450  |
|2017|null   |650   |
|2017|1      |150   |
|2017|2      |500   |
|2018|null   |800   |
|2018|1      |400   |
|2018|2      |400   |
<!--table-caption title="Query result for example 'rollup sales data on quarter by year basis, a yearly basis and a grand total'"-->

A CUBE grouping is an extension to the *group by* clause that produces a result set that contains all the rows of a ROLLUP aggregation and
in addition, grouping sets that do not represent a subtotal or grand total. A `GROUPY BY COL1, COL2 WITH CUBE` generates the following grouping sets:
**(COL1, COL2)**, **(COL1)**, **(COL2)**, **()**. The **N** elements of a CUBE convert to **2\*\*N** (2 to the power N) grouping sets.

Referring to the preceding example, adding a `WITH CUBE` modifier to the *group by* clause computes *rollup* sales data on a quarter by year basis, a yearly basis,
a quarterly year-independent basis and a grand total as shown by the following example.

```sql
-- rollup sales data on a quarter by year basis, a yearly basis,
-- a quarterly year-independent basis and a grand total
SELECT
    sales.col1 AS year,
    sales.col2 AS quarter,
    SUM(sales.col3) AS amount
FROM VALUES
    (2017, 1 ,100),
    (2017, 1 ,50),
    (2017, 2 ,200),
    (2017, 2 ,300),
    (2018, 1 ,300),
    (2018, 1 ,100),
    (2018, 2 ,400) AS sales
GROUP BY col1, col2
WITH CUBE
ORDER BY year, quarter
```
{: codeblock}


The result of the example query is shown in the following table.

|YEAR|QUARTER|AMOUNT|
|----|-------|------|
|null|null   |1450  |
|null|1      |550   |
|null|2      |900   |
|2017|null   |650   |
|2017|1      |150   |
|2017|2      |500   |
|2018|null   |800   |
|2018|1      |400   |
|2018|2      |400   |
<!--table-caption title="Query result for example 'rollup sales data on a quarter by year basis, a yearly basis, a quarterly year-independent basis and a grand total'"-->

With a GROUPING SETS grouping, an extension to the *group by* clause, you can explicitly specify the grouping sets of interest. In other words, the ROLLUP and the CUBE groupings are shortcuts for common grouping set use cases.

Referring to the preceding example, by adding a `GROUPING SETS` modifier to the *group by* clause you can compute *rollup* sales data on a quarter-by-year basis and a yearly basis only as shown in the following example.

```sql
-- rollup sales data on a quarter by year basis and a yearly basis only
SELECT
    sales.col1 AS year,
    sales.col2 AS quarter,
    SUM(sales.col3) AS amount
FROM VALUES
    (2017, 1 ,100),
    (2017, 1 ,50),
    (2017, 2 ,200),
    (2017, 2 ,300),
    (2018, 1 ,300),
    (2018, 1 ,100),
    (2018, 2 ,400) AS sales
GROUP BY col1, col2
GROUPING SETS ( (COL1), (COL1, COL2) )
ORDER BY year, quarter
```
{: codeblock}


The result of the example query is shown in the following table.

|YEAR|QUARTER|AMOUNT|
|----|-------|------|
|2017|null   |650   |
|2017|1      |150   |
|2017|2      |500   |
|2018|null   |800   |
|2018|1      |400   |
|2018|2      |400   |
<!--table-caption title="Query result for example 'rollup sales data on a quarter by year basis and a yearly basis only'"-->

<h3>More topics</h3>

For more information about the clauses that are used in a *simpleselect*, see the following topics:
* [booleanExpression](#booleanExpression)
* [expression](#expression)
* [identifier](#identifier)
* [lateralView](#lateralView)
* [namedWindows](#namedWindows)
* [relation](#relation)
* [resultColumn](#resultColumn)

<h3>Related references</h3>

A *simpleselect* is referenced by the following clause:

* [fullselect](#fullselect)

### Sort Item Clause
{: #chapterSortItemClause}

*Sort items* are a component of a *fullselect* or a *window specification*.

<h3 id="sortItem">sortItem</h3>

<!--include-svg src="./svgfiles/sortItem.svg" target="./diagrams/sortItem.svg" alt="syntax diagram for a sort item"  layout="" -->

The semantics of the *sort item* components are as follows:
* `expression`: The expression represents a *sort key*. The value of the sort key is used to order the rows of the result.
* `ASC`: Uses the values of the sort key in ascending order. ASC is the default.
* `DESC`: Uses the values of the sort key in descending order.
* `NULLS`:
    * `FIRST`: Specifies that `NULL` values appear first in the order.
    * `LAST`: Specifies that `NULL` values appear last in the order.

<h3>More topics</h3>

For more information about the clauses that are used in a *expression* clause, see the following topic:
* [expression](#expression)

<h3>Related references</h3>

A *sort item clause* is referenced by the following clauses:
* [fullselect](#fullselect)
* [windowSpec](#windowSpec)

## Relations
{: #chapterRelations}

A *relation* is a component of a *simpleselect* or a *relation primary* (which itself is a component of a *relation*). It is usually referred to as a *table*.

A relation is an entity that represents input data that is either stored physically on Cloud {{site.data.keyword.cos_short}} or taken from virtual intermediate results. Such an intermediate result can be, for example, the result of a *subselect*, a *values clause*, a *common table expression*, or a *table-valued function*. Relations can be queried or joined with other relations.

Specifying the physical data stored on Cloud {{site.data.keyword.cos_short}} as a relation is done by using the [externalTableSpec](#externalTableSpec) syntax. An example of a valid table URI is `cos://us-geo/sql/orders.parquet`, which references one of the sample tables that are provided by {{site.data.keyword.sqlquery_short}} out of the box.

Multiple relations can be composed by using join operators. The syntax for joining relations is defined by the following syntax diagrams.

<h3 id="relation">relation</h3>

<!--include-svg src="./svgfiles/relation.svg" target="./diagrams/relation.svg" alt="syntax diagram for a relation" layout="" -->

<h3 id="joinClause">joinClause</h3>

<!--include-svg src="./svgfiles/joinClause.svg" target="./diagrams/joinClause.svg" alt="syntax diagram for a join clause" layout="@break@" -->

<h3 id="naturalJoinClause">naturalJoinClause</h3>

<!--include-svg src="./svgfiles/naturalJoinClause.svg" target="./diagrams/naturalJoinClause.svg" alt="syntax diagram for a natural join clause" layout="@break@" -->

Relations can be joined by using several types of joins that are described in detail in section [joinType](#joinType).

Apart from the join type, the following two different types of joins exist:
* Joins that specify a *join condition* by using a `booleanExpression` or a `USING clause`.
* `NATURAL` joins that make an implicit guess on which columns to use for joining relations. Use natural joins carefully.

<h3 id="relationPrimary">relationPrimary</h3>

<!--include-svg src="./svgfiles/relationPrimary.svg" target="./diagrams/relationPrimary.svg" alt="syntax diagram for a relation primary" layout="" -->

<h3 id="externalTableSpec">externalTableSpec</h3>

An external table specification represents an URI for an object that is stored on Cloud {{site.data.keyword.cos_short}} combined with a specification of the object type. Valid values for object type identifier are `AVRO`, `CSV`, `JSON`, `ORC`, or `PARQUET`.

If the file format is CSV, the optional `FIELDS TERMINATED BY` clause allows you to specify a field separator other than the default `,` (comma). For example, a query for parsing a CSV with `|` (vertical bar) as the delimiter looks like the following:

```sql
SELECT *
   FROM cos://us-geo/sql/BlackFriday.csv
       STORED AS CSV FIELDS TERMINATED BY '|'
   LIMIT 3`
```
All single Unicode characters are allowed as delimiters.

By default, it is assumed that CSV input objects have a header line that specifies the names of the input columns.  If the objects don't have a header line, you must specify the option `NOHEADER` in the `STORED AS CSV` clause. In this case, the names _C0, _C1, ... are used for the input columns.
For more information, see [COS URI](#COSURI).

By default, if the format of the input data is JSON, each line must contain a separate, self-contained, and valid JSON object, also called newline-delimited JSON.
However, if you specify the option `MULTILINE`, {{site.data.keyword.sqlquery_short}} can process JSON input data even if individual data records span multiple lines,
such as when the data was formatted to make it easier to read. Specify this option only if you really need it because it limits input parallelization and can
significantly reduce performance when you process large volumes of JSON data. If you need to frequently query large amounts of multiline JSON data, use {{site.data.keyword.sqlquery_short}} to transform the data into single -line JSON, or into a more performance optimized format, such as Parquet, before querying the transformed data.

If the file format is Parquet, the optional `MERGE SCHEMA` clause allows you to handle Parquet schema evolution by specifying to scan all qualifying Parquet objects for their schema, and to merge the final schema across all objects. By default, for Parquet input only the first Parquet object found is used to infer the schema, which guarantees minimal overhead for compiling the SQL. Thus, use this option if your Parquet input data does not have a homogeneous schema.

<!--include-svg src="./svgfiles/externalTableSpec.svg" target="./diagrams/externalTableSpec.svg" alt="syntax diagram for an external table specification" layout="" -->

<h3 id="timeSeriesProperties">timeSeriesProperties</h3>

The TIME_SERIES_FORMAT option triggers a read transformation mechanism that uses a set of timeSeriesProperties to dynamically generate one or more native time series columns (defined by the IN clause) from the specified value and key columns of the input data.

<!--include-svg src="./svgfiles/timeSeriesProperties.svg" target="./diagrams/timeSeriesProperties.svg" alt="syntax diagram for time series properties" layout="" -->

The parameters `timetick` and `value` are the only parameters that are required to be specified.

Following you see the descriptions of each parameter and how they affect the time series:

* `timetick`: The column that contains the timestamp or `timetick`. Ultimately, the resulting time series is sorted by this column.
If two rows contain the same `timetick`, it is uncertain which `timetick` comes first in the time series.

* `value`: The column that contains the value.

* `key`: Optionally specify a `key` column that you can use to group each time series by. If a `key` is indicated, you can assume that *n* time series are created, where *n* is the set of all keys in the `key` column. If no `key` column is specified, a single time series is created from the indicated data set.

* `starttime`: Optionally specify a `starttime` string (any properly formatted [`DateTime`](https://docs.oracle.com/javase/8/docs/api/java/time/format/DateTimeFormatter.html))
for which to set the time series [TRS](/docs/sql-query?topic=sql-query-TRS). If `starttime` is not indicated, and `granularity` is indicated, the `starttime` defaults to 1 January 1970 12am (midnight) GMT. However, if no `granularity` is indicated, a [TRS](/docs/sql-query?topic=sql-query-TRS) is not associated with the created time series.

* `granularity`: Optionally specify a `granularity` string (a properly formatted ISO-8601 duration format) for which to set the time series reference system [TRS](/docs/sql-query?topic=sql-query-TRS). If `granularity` is not indicated, and `starttime` is indicated, the default `granularity` is 1 millisecond. However, if no `starttime` is indicated, a [TRS](/docs/sql-query?topic=sql-query-TRS) is not associated with the created time series.

The following examples show you how to use  TIME_SERIES_FORMAT parameters for dynamic time series generation during the read process.

Create a time series per location, set the time series TRS with default start time and 1 ms granularity, and store with it with the name "ts".

```
SELECT
	location,
	ts
FROM cos://us-geo/sql/temperature_humidity.csv
USING TIME_SERIES_FORMAT(key="location", timetick="timestamp", value="humidity", granularity="PT0.001S") in ts
```

Create a time series per location, set the time series TRS with start time "2011-12-03T10:15:30" and default granularity (1 ms), and store it
with the name "ts".

```
SELECT
	location,
	ts
FROM cos://us-geo/sql/temperature_humidity.csv
USING TIME_SERIES_FORMAT(key="location", timetick="timestamp", value="humidity", starttime="2011-12-03T10:15:30") in ts
```

Create a time series per location with no TRS, store it with the name "ts".
If no granularity or start time are provided, a TRS will not be associated with the time series and therefore with_trs will run into exception.

```
SELECT
	location,
	ts
FROM cos://us-geo/sql/temperature_humidity.csv
USING TIME_SERIES_FORMAT(key="location", timetick="timestamp", value="humidity") in ts
```

Create a single time series, store it with the default name "time_series".
Without specifying a key, it is not possible to create multiple time series.

```
SELECT
	location,
	time_series
FROM cos://us-geo/sql/temperature_humidity.csv
USING TIME_SERIES_FORMAT(timetick="timestamp", value="humidity")
```

<!--include-svg src="./svgfiles/timeSeriesOptions.svg" target="./diagrams/timeSeriesOptions.svg" alt="syntax diagram for time series options" layout="" -->

<h3 id="tableTransformer">tableTransformer</h3>

A table transformer is a function that is applied to the input data set before it is sent to the actual SQL query compilation and execution.

You can wrap your external table definition optionally with the `FLATTEN` table transformation function.
It preprocesses your input table before query compilation to a fully flat column schema.
This can be useful when you have hierarchical input data as it is often found in JSON documents.
By using `FLATTEN`, you do not need to dereference all nested columns explicitly in your SQL statement.

For example, you can run a simple `SELECT * FROM FLATTEN(cos://us-geo/sql/iotmessages STORED AS JSON)` on a flattened JSON
input and use CSV output to easily browse a sample of your JSON input data.

The `FLATTEN` table transformation function creates a flat list of columns by concatenating all nested column names with _.
You can optionally also combine `FLATTEN` with `CLEANCOLS`.

You can wrap your external table definition optionally with the `CLEANCOLS` table transformation function.
It preprocesses your input table before query compilation by renaming all columns that have characters that are NOT supported by certain target formats, such as Parquet.
These characters are `,`, `;`, `,,,`, `=`, `(`, `)`, `{`, and `}`. They are replaced by the corresponding URL-encoded representation, for example, %20 for space (` `).
This allows you to write results, for example, into Parquet, without the need to provide column by column alias names in your SQL
when your input data has columns with these characters. A typical situation is the existence of space (` `) in input columns.

For example, you can use `SELECT * FROM CLEANCOLS(cos://us-geo/sql/iotmessages STORED AS JSON) INTO cos://us-geo/mybucket/myprefix STORED AS PARQUET` to produce a result set that can be stored as is into Parquet target format.

If you wrap your external table definition with the `DESCRIBE` table transformer,
the table does not show its actual content but the schema that is inferred from the objects in {{site.data.keyword.cos_full}} instead.
This allows you to explore the schema before authoring your actual SQL statements against it.

When you use the `DESCRIBE` table transformer in your SQL statement, the default output format is JSON instead of CSV.

You can also wrap `DESCRIBE` around the other table transformers to explore the transformed table schema.
However, you cannot wrap other table transformers around the `DESCRIBE` transformer.

<!--include-svg src="./svgfiles/tableTransformer.svg" target="./diagrams/tableTransformer.svg" alt="syntax diagram for an table transformer" layout="" -->

<h3 id="tableValuedFunction">tableValuedFunction</h3>

A table-valued function returns a relation, that is, a set of rows. An example of a table-valued function is `range()`. For more information, see [SQL functions](/docs/sql-query?topic=sql-query-sqlfunctions#sqlfunctions).

<!--include-svg src="./svgfiles/tableValuedFunction.svg" target="./diagrams/tableValuedFunction.svg" alt="syntax diagram for a table valued function" layout="" -->

<h3>More topics</h3>

For more information about the clauses that are used in *relation* clauses, see the following topics:
* [booleanExpression](#booleanExpression)
* [COSURI](#COSURI)
* [expression](#expression)
* [fullselect](#fullselect)
* [identifier](#identifier)
* [joinType](#joinType)
* [sample](#sample)
* [valuesClause](#valuesClause)

<h3>Related references</h3>

A *relation* is referenced by the following clause:

* [simpleselect](#simpleselect)

### Values Clause
{: #chapterValuesClause}

A *values clause* is a component of a *fullselect* or represents a *primary relation*. Its syntax is defined by the following syntax diagram.

<h3 id="valuesClause">valuesClause</h3>

<!--include-svg src="./svgfiles/valuesClause.svg" target="./diagrams/valuesClause.svg" alt="syntax diagram for a values clause" layout="" -->

With a values clause, you can define a result set by specifying actual values for each column of a row by using expressions.

Each `expression` in the list of expressions represents a row of the result set that is defined.

In a single-column result set, each expression represents the value of this column in a row.

In a multi-column result set, each expression represents a list of *n* expressions that are enclosed by parentheses, where *n* is the number of columns in the result set.

To join a *values clause* with other types of result sets, specify an `identifier` that acts as an **alias** for the values clause.

<h3>Examples</h3>

```sql
-- single column result set with 3 rows
SELECT * FROM VALUES 1, 2 , 3
```
{: codeblock}


The result of the example query is shown in the following table.

|COL1|
|----|
|1   |
|2   |
|3   |
<!--table-caption title="Query result for example 'single column result set with 3 rows'"-->

```sql
-- single column result set with 3 rows specifying parentheses for row expressions
SELECT * FROM VALUES (1), (2) , (3)
```
{: codeblock}


The result of the example query is shown in the following table.

|COL1|
|----|
|1   |
|2   |
|3   |
<!--table-caption title="Query result for example 'single column result set with 3 rows specifying parentheses for row expressions'"-->

```sql
--- joining two multi column result sets by using their identifier
SELECT
    emp.col1 AS id,
    emp.col2 AS name,
    mission.col2 AS missions
FROM
    VALUES
        (1, 'Spock'),
        (2,'Kirk') ,
        (3, 'McCoy'),
        (4,'Scotty') AS emp,
    VALUES
        (1,3000),
        (2,2000),
        (3,3000),
        (4,4000) AS mission
WHERE emp.col1 = mission.col1
```
{: codeblock}


The result of the example query is shown in the following table.

|ID |NAME  |MISSIONS|
|---|------|--------|
|1  |Spock |3000    |
|2  |Kirk  |2000    |
|3  |McCoy |3000    |
|4  |Scotty|4000    |
<!--table-caption title="Query result for example 'joining two multi column result sets by using their identifier'"-->

### Values Statement
{: #chapterValuesStatement}

A *values statement* is a statement on its own. It can be used instead of a *fullselect* if your statement
references only a single value and does not contain any join with other relations or values clauses.

<h3>Examples</h3>

```sql
-- values statement with single column result set with 3 rows
VALUES 1, 2, 3
```
{: codeblock}


The result of the example query is shown in the following table.

|COL1|
|----|
|1   |
|2   |
|3   |
<!--table-caption title="Query result for example 'values statement with single column result set with 3 rows'"-->

```sql
--- values statement with multi column result set
VALUES
        (1, 'Spock'),
        (2,'Kirk') ,
        (3, 'McCoy'),
        (4,'Scotty')
```
{: codeblock}

The result of the example query is shown in the following table.

|COL1|COL2  |
|----|------|
|1   |Spock |
|2   |Kirk  |
|3   |McCoy |
|4   |Scotty|
<!--table-caption title="Query result for example 'values statement with multi column result set'"-->

<h3>More topics</h3>
For more information about the clauses that are used in a *values clause*, see the following topics:
* [expression](#expression)
* [identifier](#identifier)

<h3>Related references</h3>

A *values clause* is referenced by the following clauses:
* [fullselect](#fullselect)
* [relationPrimary](#relationPrimary)

### Lateral Views
{: #chapterLateralViews}

A lateral view is a component of a *simpleselect*. Lateral views allow to build  *virtual tables* at query execution time
by using *table-generating functions*. Examples of table-generating functions are `explode()`, `posexplode()`, and `posexplode_outer()`.
The explode()-style functions take an array or map as input and return a row for each element in the array. For more information, see [SQL functions](/docs/sql-query?topic=sql-query-sqlfunctions#sqlfunctions).

<h3 id="lateralView">lateralView</h3>

The syntax of a lateral view clause is described by the following syntax diagram.

<!--include-svg src="./svgfiles/lateralView.svg" target="./diagrams/lateralView.svg" alt="syntax diagram for a lateral view" layout="@break@" -->

The semantics of the entities in order of appearance in the syntax diagrams is as follows:
* `OUTER`: Specifying this keyword ensures that the lateral view contains at least one row with *null* values in case the table-generating function does not return any rows.
* `qualifiedName`: Name of a table-generating function.
* `expression`: An expression that resolves to an array.
* `identifier`: Lateral view name, that is, the name of the new virtual table.
* `identifier`: Lateral view column names.

<h3>Examples</h3>

Lateral views are useful when you deal with repeating groups within a table, that is, not normalized tables. The examples show how to deal with tables that represent a 1-n relation and an n-m relation.

```sql
-- deal with a 1-n relation

SELECT
    master_child.col1 AS master_id,
    child_table.child_id
FROM VALUES
    ( 1 , ARRAY(1,2,3)),
    (2, ARRAY(4,5,6)) AS master_child
LATERAL VIEW EXPLODE(master_child.col2) child_table AS child_id
```
{: codeblock}


The result of the example query is shown in the following table.

|MASTER_ID|CHILD_ID|
|---------|--------|
|1        |1       |
|1        |2       |
|1        |3       |
|2        |4       |
|2        |5       |
|2        |6       |
<!--table-caption title="Query result for example 'deal with a 1-n relation'"-->

```sql
-- deal with an n-m relation

SELECT
    master_table.master_id,
    child_table.child_id
FROM VALUES
    ( ARRAY(10,20) , ARRAY(1,2,3)),
    ( ARRAY(30,40), ARRAY(4,5,6)) AS master_child
LATERAL VIEW EXPLODE(master_child.col1) master_table AS master_id
LATERAL VIEW EXPLODE(master_child.col2) child_table AS child_id
```
{: codeblock}

The result of the example query is shown in the following table.

|MASTER_ID|CHILD_ID|
|---------|--------|
|10       |1       |
|10       |2       |
|10       |3       |
|20       |1       |
|20       |2       |
|20       |3       |
|30       |4       |
|30       |5       |
|30       |6       |
|40       |4       |
|40       |5       |
|40       |6       |
<!--table-caption title="Query result for example 'deal with an n-m relation'"-->

<h3>More topics</h3>

For more information about the clauses that are used in a *lateral view* clause, see the following topics:
* [expression](#expression)
* [identifier](#identifier)
* [qualifiedName](#qualifiedName)

Note:
* A *simpleselect* can contain multiple lateral view clauses that are evaluated in order of appearance.
* Within a lateral view clause, you can refer to columns defined in any (virtual) table on the left of the current lateral view clause.

<h3>Related references</h3>

A *lateral view* clause is referenced by a [simpleselect](#simpleselect).

### Join Types
{: #chapterJoinTypes}

SELECT statements can retrieve and join column values from two or more tables into a single row. The retrieval is based on a specified condition, typically of matching column values.

The main characteristic of a join is, typically, matching column values in rows of each table that participates in the join. The result of a join associates rows from one table with rows from another table. Depending on the type of join operation, some rows might be formed that contain column values in one table that do not match column values in another table.

A joined table specifies an intermediate result table that is the result of either an `INNER` join, an `OUTER` join, a `CROSS` join, or an `ANTI` join.
The table is derived by applying one of the join operators to its operands.

<h3 id="joinType">joinType</h3>

<!--include-svg src="./svgfiles/joinType.svg" target="./diagrams/joinType.svg" alt="syntax diagram for join types"  layout="" -->

<h4>Inner join</h4>

An `INNER` join combines each row of the left table with each row of the right table, keeping only the rows in which the join condition is true.

```sql
-- inner join query
SELECT
    left_table.col1 AS l_col1,
    left_table.col2 AS l_col2,
    right_table.col1 AS r_col1,
    right_table.col2 AS r_col2
FROM
    VALUES (0, 10), (1, 11), (2, 12), (3,13), (4, 14), (5, 14) AS left_table
    INNER JOIN
    VALUES (0, 10), (2, 12), (4, 14), (6, 16) AS right_table
    ON left_table.col1 = right_table.col1
```
{: codeblock}


The result of the example query is shown in the following table.

|L_COL1|L_COL2|R_COL1|R_COL2|
|------|------|------|------|
|0     |10    |0     |10    |
|2     |12    |2     |12    |
|4     |14    |4     |14    |
<!--table-caption title="Query result for example 'inner join query'"-->

<h4>Outer join</h4>

An `OUTER` join includes the rows that are produced by the inner join, plus the missing rows, depending on the type of outer join.

A `LEFT OUTER` or `LEFT` join includes the rows from the left table that were missing from the inner join.

```sql
-- left outer join query
SELECT
    left_table.col1 AS l_col1,
    left_table.col2 AS l_col2,
    right_table.col1 AS r_col1,
    right_table.col2 AS r_col2
FROM VALUES (0, 10), (1, 11), (2, 12), (3,13), (4, 14), (5, 14) AS left_table
LEFT OUTER JOIN
VALUES (0, 10), (2, 12), (4, 14), (6, 16) AS right_table
ON left_table.col1 = right_table.col1
```
{: codeblock}


The result of the example query is shown in the following table.

|L_COL1|L_COL2|R_COL1|R_COL2|
|------|------|------|------|
|0     |10    |0     |10    |
|1     |11    |null  |null  |
|2     |12    |2     |12    |
|3     |13    |null  |null  |
|4     |14    |4     |14    |
|5     |14    |null  |null  |
<!--table-caption title="Query result for example 'left outer join query'"-->

A `RIGHT OUTER` or `RIGHT` join includes the rows from the right table that were missing from the inner join.

```sql
-- right outer join query
SELECT
    left_table.col1 AS l_col1,
    left_table.col2 AS l_col2,
    right_table.col1 AS r_col1,
    right_table.col2 AS r_col2
FROM
    VALUES (0, 10), (1, 11), (2, 12), (3,13), (4, 14), (5, 14) AS left_table
    RIGHT OUTER JOIN
    VALUES (0, 10), (2, 12), (4, 14), (6, 16) AS right_table
    ON left_table.col1 = right_table.col1
```
{: codeblock}


The result of the example query is shown in the following table.

|L_COL1|L_COL2|R_COL1|R_COL2|
|------|------|------|------|
|0     |10    |0     |10    |
|2     |12    |2     |12    |
|4     |14    |4     |14    |
|null  |null  |6     |16    |
<!--table-caption title="Query result for example 'right outer join query'"-->

A `FULL OUTER` or `FULL` join includes the rows from both tables that were missing from the inner join.

```sql
-- full outer join query
SELECT
    left_table.col1 AS l_col1,
    left_table.col2 AS l_col2,
    right_table.col1 AS r_col1,
    right_table.col2 AS r_col2
FROM
    VALUES (0, 10), (1, 11), (2, 12), (3, 13), (4, 14), (5, 14) AS left_table
    FULL OUTER JOIN
    VALUES (0, 10), (2, 12), (4, 14), (6, 16) AS right_table
    ON left_table.col1 = right_table.col1
```
{: codeblock}


The result of the example query is shown in the following table.

|L_COL1|L_COL2|R_COL1|R_COL2|
|------|------|------|------|
|1     |11    |null  |null  |
|null  |null  |6     |16    |
|3     |13    |null  |null  |
|5     |14    |null  |null  |
|4     |14    |4     |14    |
|2     |12    |2     |12    |
|0     |10    |0     |10    |
<!--table-caption title="Query result for example 'full outer join query'"-->

<h4>Cross join</h4>

A `CROSS` join creates a Cartesian product of the tables that are involved in the join operation, if a CROSS join that specifies a join condition behaves like an inner join.

```sql
-- cross join that specifies a join condition

SELECT
    left_table.col1 AS l_col1,
    left_table.col2 AS l_col2,
    right_table.col1 AS r_col1,
    right_table.col2 AS r_col2
FROM
    VALUES (0, 10), (1, 11), (2, 12), (3,13), (4, 14), (5, 14) AS left_table
    CROSS JOIN VALUES (0, 10), (2, 12), (4, 14), (6, 16) AS right_table
    ON left_table.col1 = right_table.col1
```
{: codeblock}


The result of the example query is shown in the following table.

|L_COL1|L_COL2|R_COL1|R_COL2|
|------|------|------|------|
|0     |10    |0     |10    |
|2     |12    |2     |12    |
|4     |14    |4     |14    |
<!--table-caption title="Query result for example 'cross join that specifies a join condition'"-->

```sql
-- cross join that specifies no join condition
SELECT
    left_table.col1 AS l_col1,
    left_table.col2 AS l_col2,
    right_table.col1 AS r_col1,
    right_table.col2 AS r_col2
FROM
    VALUES (0, 10), (1, 11), (2, 12), (3,13), (4, 14), (5, 14) AS left_table
    CROSS JOIN VALUES (0, 10), (2, 12), (4, 14), (6, 16) AS right_table
```
{: codeblock}


The result of the example query is shown in the following table.

|L_COL1|L_COL2|R_COL1|R_COL2|
|------|------|------|------|
|0     |10    |0     |10    |
|0     |10    |2     |12    |
|0     |10    |4     |14    |
|0     |10    |6     |16    |
|1     |11    |0     |10    |
|1     |11    |2     |12    |
|1     |11    |4     |14    |
|1     |11    |6     |16    |
|2     |12    |0     |10    |
|2     |12    |2     |12    |
|2     |12    |4     |14    |
|2     |12    |6     |16    |
|3     |13    |0     |10    |
|3     |13    |2     |12    |
|3     |13    |4     |14    |
|3     |13    |6     |16    |
|4     |14    |0     |10    |
|4     |14    |2     |12    |
|4     |14    |4     |14    |
|4     |14    |6     |16    |
|5     |14    |0     |10    |
|5     |14    |2     |12    |
|5     |14    |4     |14    |
|5     |14    |6     |16    |
<!--table-caption title="Query result for example 'cross join that specifies no join condition'"-->

<h4>Anti join</h4>

A `LEFT ANTI` or `ANTI` join returns only rows from the left table that do not have a matching row in the right table.
Columns from the right table cannot be included in the column list of the select statement.

```sql
-- left anti join query
SELECT
    left_table.col1 AS l_col1,
    left_table.col2 AS l_col2
FROM
    VALUES (0, 10), (1, 11), (2, 12), (3,13), (4, 14), (5, 14) AS left_table
    ANTI JOIN
    VALUES (0, 10), (2, 12), (4, 14), (6, 16) AS right_table
    ON left_table.col1 = right_table.col1
```
{: codeblock}


The result of the example query is shown in the following table.

|L_COL1|L_COL2|
|------|------|
|1     |11    |
|3     |13    |
|5     |14    |
<!--table-caption title="Query result for example 'left anti join query'"-->

<h4>Left semi join</h4>

A `LEFT SEMI` join acts like an inner join but does not include the columns of the right table.

```sql
-- left semi join query
SELECT
    left_table.col1 AS l_col1,
    left_table.col2 AS l_col2
FROM
    VALUES (0, 10), (1, 11), (2, 12), (3,13), (4, 14), (5, 14) AS left_table
    LEFT SEMI JOIN
    VALUES (0, 10), (2, 12), (4, 14), (6, 16) AS right_table
    ON left_table.col1 = right_table.col1
```
{: codeblock}

The result of the example query is shown in the following table.

|L_COL1|L_COL2|
|------|------|
|0     |10    |
|2     |12    |
|4     |14    |
<!--table-caption title="Query result for example 'left semi join query'"-->

<h3>Related references</h3>

The *join types* are specified in a [relation](#relation).

### Sampling table data
{: #chapterSamplingTableData}

Any table that is object stored on Cloud {{site.data.keyword.cos_short}}, used in a *from clause*, can be associated with a *table sample clause*.
The table sample clause defines how to retrieve a subset of rows from the underlying table (object stored on Cloud {{site.data.keyword.cos_short}}).
Thus, you can write queries for samples of the data, for example, for interactive data exploration and data mining.

The general syntax of a table sample clause is described by the following syntax diagram.

<h3 id="sample">sample</h3>

<!--include-svg src="./svgfiles/sample.svg" target="./diagrams/sample.svg" alt="syntax diagram for a sample" layout="" -->

<h3 id="bucketSampleClause">bucketSampleClause</h3>

<!--include-svg src="./svgfiles/bucketSampleClause.svg" target="./diagrams/bucketSampleClause.svg" alt="syntax diagram for a bucket sample clause" layout="@break@" -->

Three sampling types are supported:

* `TABLESAMPLE <number> PERCENT` lets you sample a certain percentage of rows from the underlying table.
* `TABLESAMPLE <expression> ROWS` lets you sample some rows from the underlying table.
* `TABLESAMPLE BUCKET x OUT OF y` lets you bucketize the underlying data into `y` buckets and returns rows from bucket `x`. Buckets are numbered from `1`to `y`.

<h3>Examples</h3>

The following examples demonstrate how to sample a subset of data from a Parquet object on Cloud {{site.data.keyword.cos_short}}.
The object referenced is accessible from the web UI as part of the provided sample queries.

```sql
-- retrieve 10 percent of employee data
SELECT * FROM cos://us-geo/sql/employees.parquet STORED AS PARQUET TABLESAMPLE (10 PERCENT)
```
{: codeblock}

```sql
-- retrieve 10 rows from the employee data object
SELECT * FROM cos://us-geo/sql/employees.parquet STORED AS PARQUET TABLESAMPLE (10 ROWS)
```
{: codeblock}

```sql
-- bucketize the employee data in 10 buckets and retrieve data from 2 buckets
SELECT * FROM cos://us-geo/sql/employees.parquet STORED AS PARQUET TABLESAMPLE (BUCKET 2 OUT OF 10)
```
{: codeblock}

<h3>More topics</h3>

For more information about the clauses that are used in a *table sample clause*, see the following topics:
 * [expression](#expression)
 * [identifier](#identifier)
 * [qualifiedName](#qualifiedName)
 * [unsignedInteger](#unsignedInteger)
 * [unsignedNumber](#unsignedNumber)

<h3>Related references</h3>

A *table sample clause* is referenced by the following clause:
* [relationPrimary](#relationPrimary)

## SQL functions
{: #chapterSqlFunctions}

The syntax for SQL function invocation is described by the following syntax diagram.

<h3 id="functionOrAggregate">functionOrAggregate</h3>

<!--include-svg src="./svgfiles/functionOrAggregate.svg" target="./diagrams/functionOrAggregate.svg" alt="syntax diagram for a function or aggregate" layout="" -->

Most function invocations look like `function(argument1, ..., argumentN)` but functions like `TRIM()`, `POSITION()`, `FIRST()`, `LAST()`, `STRUCT()`, `EXTRACT()`, and `SUBSTRING()` support a different invocation style.

Refer to section [SQL functions](/docs/sql-query?topic=sql-query-sqlfunctions#sqlfunctions) for details about supported functions.

<h3>More topics</h3>

For more information about the clauses that are used in a *function or aggregate* clause, see the following topics:
* [expression](#expression)
* [qualifiedName](#qualifiedName)
* [resultColumn](#resultColumn)
* [valueExpression](#valueExpression)
* [windowSpec](#windowSpec)

<h3>Related references</h3>

A *function or aggregate clause* is referenced by the following clause:
* [primaryExpression](#primaryExpression)

### Window functions
{: #chapterWindowFunctions}

Classic SQL **aggregation functions** like `SUM()`, `MAX()`, or `MIN()` process a group of rows to derive a single value. **Window functions** take this one step further by allowing to process a group of rows and derive a single value for each row in the group.
Note the difference to **scalar functions** that return a single value for each row. Scalar functions derive a single value from a single row and not a group of rows.

With window functions, it is possible to calculate things like *moving averages* or *cumulative sums*.

Working with window functions involves two steps:
1. Choose a *window function* to answer the question of interest.
2. Define a *window* the chosen window function is applied to.

There are three types of window functions:

- **Ranking functions**, for example, `rank()`, `ntile()`, or `rowNumber()`
- **Analytic functions**, for example, `cume_dist()`, `first_value()`, or `last_value()`
- **Aggregation functions**, for example, `sum()`, `max()`, or `min()`

For more information, see [SQL functions](/docs/sql-query?topic=sql-query-sqlfunctions#sqlfunctions).

A window can be defined in two ways:
* The `WINDOW` keyword lets you define an identifier for a window specification in a *fullselect* or *simpleselect*. This named window specification can then be referenced by the `OVER` keyword.
* Unnamed window specifications can be defined inline following the keyword `OVER` in a *fullselect* or *simpleselect*.

The syntax of a window specification is defined by the following syntax diagrams.

<h3 id="namedWindows">namedWindows</h3>

<!--include-svg src="./svgfiles/namedWindows.svg" target="./diagrams/namedWindows.svg" alt="syntax diagram for named windows" layout="" -->

<h3 id="namedWindow">namedWindow</h3>

<!--include-svg src="./svgfiles/namedWindow.svg" target="./diagrams/namedWindow.svg" alt="syntax diagram for a named window" layout="" -->

<h3 id="windowSpec">windowSpec</h3>

<!--include-svg src="./svgfiles/windowSpec.svg" target="./diagrams/windowSpec.svg" alt="syntax diagram for a window specification" layout="" -->

<h3 id="windowClusterBy">windowClusterBy</h3>

<!--include-svg src="./svgfiles/windowClusterBy.svg" target="./diagrams/windowClusterBy.svg" alt="syntax diagram for a window cluster by clause" layout="" -->

<h3 id="windowPartitionBy">windowPartitionBy</h3>

<!--include-svg src="./svgfiles/windowPartitionBy.svg" target="./diagrams/windowPartitionBy.svg" alt="syntax diagram for a window partition by clause" layout="" -->

<h3 id="windowOrderBy">windowOrderBy</h3>

<!--include-svg src="./svgfiles/windowOrderBy.svg" target="./diagrams/windowOrderBy.svg" alt="syntax diagram for a window order by clause" layout="" -->

The window specification consists of the following clauses:
* The `PARTITION BY` clause defines which rows belong to the same *window partition*. `DISTRIBUTE BY` can be used as a synonym for `PARTITION BY`.
* The `ORDER BY` clause defines the ordering of rows within a window partition. `SORT BY` can be used as a synonym for `ORDER BY`.
* There are two ways to define a *window size*:
    * `RANGE`: The window size is defined as a value range.
    * `ROW`: The window size is defined as the number of rows before and/or after the current row.
* The following keywords can be used to define range boundaries:
    * `CURRENT ROW`: Specifies to use the current row as a bound.
    * `UNBOUNDED`: Specifies to use negative or positive infinity as a lower or upper bound.
    * Depending on the context `PRECEDING` specifies:
        * `RANGE`: The lower bound of the value range.
        * `ROWS`: The number of rows before the current row.
    * Depending on the context `FOLLOWING` specifies:
        * `RANGE`: The upper bound of the value range.
        * `ROWS`: The number of rows after the current row.

<h3 id="windowFrame">windowFrame</h3>

<!--include-svg src="./svgfiles/windowFrame.svg" target="./diagrams/windowFrame.svg" alt="syntax diagram for a window frame" layout="" -->

<h3 id="frameBound">frameBound</h3>

<!--include-svg src="./svgfiles/frameBound.svg" target="./diagrams/frameBound.svg" alt="syntax diagram for a frame boundary" layout="" -->

<h3>Examples</h3>

The following window function examples use *values clauses* to define result sets for group by operations. For more information about the values clause, see [valuesClause](#valuesClause).

<h4>Ranking Function Example</h4>

This example uses a table that contains information about employee social media activity (posts):
* Column 1: employee ID
* Column 2: department ID
* Column 3: number of social media posts

The example shows how to retrieve the ID of the two most active employees in each department.

```sql
-- derive posts ranking by using a named window specification
SELECT *
FROM (
    SELECT
        posts.col1 AS emp_id,
        posts.col2 AS dept_id,
        posts.col3 AS posts,
        DENSE_RANK() OVER post_ranking AS rank
    FROM VALUES
        (1, 1 ,100),
        (2, 1 ,50),
        (8, 1 ,250),
        (3, 2 ,200),
        (4, 2 ,300),
        (9, 2 ,1000),
        (5, 3 ,300),
        (6, 3 ,100),
        (7, 3 ,400) AS posts
    WINDOW post_ranking AS (
        PARTITION BY posts.col2
        ORDER BY posts.col3 DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
)
WHERE rank <= 2
```
{: codeblock}

The result of the example query is shown in the following table.

|EMP_ID|DEPT_ID|POSTS|RANK|
|------|-------|-----|----|
|8     |1      |250  |1   |
|1     |1      |100  |2   |
|7     |3      |400  |1   |
|5     |3      |300  |2   |
|9     |2      |1000 |1   |
|4     |2      |300  |2   |
<!--table-caption title="Query result for example 'derive posts ranking by using a named window specification'"-->

```sql
-- derive posts ranking by using an inline window specification
SELECT * FROM (
    SELECT
        posts.col1 AS emp_id,
        posts.col2 AS dept_id,
        posts.col3 AS posts,
        DENSE_RANK() OVER (
                        PARTITION BY posts.col2
                        ORDER BY posts.col3 DESC
                        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
                        ) AS rank
        FROM VALUES
            (1, 1 ,100),
            (2, 1 ,50),
            (8, 1 ,250),
            (3, 2 ,200),
            (4, 2 ,300),
            (9, 2 ,1000),
            (5, 3 ,300),
            (6, 3 ,100),
            (7, 3 ,400) AS posts
) WHERE rank <= 2
```
{: codeblock}

The result of the example query is shown in the following table.

|EMP_ID|DEPT_ID|POSTS|RANK|
|------|-------|-----|----|
|8     |1      |250  |1   |
|1     |1      |100  |2   |
|7     |3      |400  |1   |
|5     |3      |300  |2   |
|9     |2      |1000 |1   |
|4     |2      |300  |2   |
<!--table-caption title="Query result for example 'derive posts ranking by using an inline window specification'"-->

<h4>Analytic Function Example</h4>

This example uses a table that contains transaction information. The layout is as follows:
* Column 1: transaction ID
* Column 2: account number
* Column 3: transaction amount

The example shows how to create a cumulative distribution of transaction amounts by using the analytic function `CUME_DIST()`.
The `CUME_DIST()` function returns the percentage of rows that have a value less than or equal to the current row’s value.

```sql
-- cumulative distribution of transaction amounts
SELECT
    txn_amount,
    MAX(balance_dist)
FROM (
        SELECT
            txn.col2 AS account,
            txn.col3 AS txn_amount,
            CUME_DIST() OVER current_balance AS balance_dist
        FROM VALUES
            (1, 42, 1000),
            (2, 4711, 2000),
            (3, 42, -200),
            (4, 42, -200),
            (5, 4711, 1000),
            (6, 4711, -300),
            (7, 4711, -300),
            (8, 42, 1000),
            (9, 4711, -400) AS txn
        WINDOW current_balance AS (ORDER BY txn.col3)
        ORDER BY txn.col3
)
GROUP BY txn_amount
```
{: codeblock}

The result of the example query is shown in the following table.

|TXN_AMOUNT|MAX(BALANCE_DIST) |
|----------|------------------|
|-400      |0.1111111111111111|
|-300      |0.3333333333333333|
|-200      |0.5555555555555556|
|1000      |0.8888888888888888|
|2000      |1.0               |
<!--table-caption title="Query result for example 'cumulative distribution of transaction amounts'"-->

<h4>Aggregation Function Example</h4>

This example uses a table containing transaction information. The layout is as follows:
* Column 1: transaction ID
* Column 2: account number
* Column 3: transaction amount

The example shows how to retrieve the *total balance* of each account at the time of each transaction.

```sql
--- total balance of each account at the time of each transaction
SELECT
    txn.col1 AS txn_id,
    txn.col2 AS account,
    txn.col3 AS txn_amount,
    SUM(txn.col3) OVER current_balance AS balance
FROM VALUES
    (1, 42, 100),
    (2, 4711, 300),
    (3, 42, 50),
    (4, 42, -250),
    (5, 4711, 1000),
    (6, 4711, -300),
    (7, 4711, 100),
    (8, 42, 200),
    (9, 4711, -400) AS txn
WINDOW current_balance AS (
        PARTITION BY txn.col2
        ORDER BY txn.col1
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
) ORDER BY account, txn_id
```
{: codeblock}


The result of the example query is shown in the following table.

|TXN_ID|ACCOUNT|TXN_AMOUNT|BALANCE|
|------|-------|----------|-------|
|1     |42     |100       |100    |
|3     |42     |50        |150    |
|4     |42     |-250      |-100   |
|8     |42     |200       |100    |
|2     |4711   |300       |300    |
|5     |4711   |1000      |1300   |
|6     |4711   |-300      |1000   |
|7     |4711   |100       |1100   |
|9     |4711   |-400      |700    |
<!--table-caption title="Query result for example 'total balance of each account at the time of each transaction'"-->

<h3>More topics</h3>
For more information about the clauses that are used by a *window function*, see the following topics:
* [expression](#expression)
* [identifier](#identifier)
* [sortItem](#sortItem)

<h3>Related references</h3>

A *named window clause* is referenced by the following clauses:
* [fullselect](#fullselect)
* [simpleselect](#simpleselect)

The keyword `OVER` lets you define an unnamed window specification in a [functionOrAggregate](#functionOrAggregate).

## SQL expressions
{: #chapterSqlExpressions}

<h3>Expressions</h3>

In the context of an SQL query statement, an *expression* is always a *Boolean expression*.

<h4 id="expression">expression</h4>

<!--include-svg src="./svgfiles/expression.svg" target="./diagrams/expression.svg" alt="syntax diagram for an expression" layout="" -->

<h4>More topics</h4>

For more information, see [booleanExpression](#booleanExpression).

<h4>Related references</h4>

An *expression* is referenced by the following clauses:

* [caseExpression](#caseExpression)
* [castExpression](#castExpression)
* [frameBound](#frameBound)
* [fullselect](#fullselect)
* [functionOrAggregate](#functionOrAggregate)
* [groupByClause](#groupByClause)
* [groupingSet](#groupingSet)
* [lateralView](#lateralView)
* [predicate](#predicate)
* [primaryExpression](#primaryExpression)
* [resultColumn](#resultColumn)
* [sample](#sample)
* [sortItem](#sortItem)
* [tableValuedFunction](#tableValuedFunction)
* [valuesClause](#valuesClause)
* [whenClause](#whenClause)
* [windowSpec](#windowSpec)

### Boolean expressions
{: #chapterBooleanExpressions}

The syntax of a *Boolean expression* is defined by the following syntax diagrams.

<h4 id="booleanExpression">booleanExpression</h4>

<!--include-svg src="./svgfiles/booleanExpression.svg" target="./diagrams/booleanExpression.svg" alt="syntax diagram for a Boolean expression" layout="" -->

A Boolean expression is one of the following:
* A negation of a Boolean expression by using Boolean operator `NOT`.
* A conjunction of Boolean expressions by using Boolean operator `AND`.
* A disjunction of Boolean expressions by using Boolean operator `OR`.
* An `EXIST` predicate.
* A *value expression* optionally followed by a *predicate*.

Refer to [booleanOperator](#booleanOperator) for details about Boolean operators `NOT`, `AND`, and `OR`.

The `EXISTS` predicate tests for the existence of certain rows. The `query` can specify any number of columns, and the following applies:
* The result is true only if the number of rows that is specified by the fullselect is not zero.
* The result is false only if the number of rows that is specified is zero.
* The result cannot be unknown.

<h4>Related references</h4>

A *Boolean expression* is referenced by the following clauses:

* [booleanExpression](#booleanExpression)
* [expression](#expression)
* [relation](#relation)
* [simpleselect](#simpleselect)


### Value expressions
{: #chapterValueExpressions}


<h4 id="valueExpression">valueExpression</h4>

<!--include-svg src="./svgfiles/valueExpression.svg" target="./diagrams/valueExpression.svg" alt="syntax diagram for a value expression" layout="" -->

A *value expression* is one of the following:
* A *primary expression*.
* The result of applying an unary operator to a value expression.
* The result of performing an arithmetic operation with two value expressions.
* The result of performing a comparison operation between two value expressions.

<h4>More topics</h4>

For more information about the clauses that are used by a *value expression*, see the following topics:
* [arithmeticOperator](#arithmeticOperator)
* [comparisonOperator](#comparisonOperator)
* [primaryExpression](#primaryExpression)
* [unaryOperator](#unaryOperator)

<h4>Related references</h4>

A *value expression* is referenced by the following clauses:
* [booleanExpression](#booleanExpression)
* [functionOrAggregate](#functionOrAggregate)
* [predicate](#predicate)


### Primary expressions
{: #chapterPrimaryExpressions}

<h4 id="primaryExpression">primaryExpression</h4>

<!--include-svg src="./svgfiles/primaryExpression.svg" target="./diagrams/primaryExpression.svg" alt="syntax diagram for a primary expression" layout=""  -->

<h4 id="constant">constant</h4>

<!--include-svg src="./svgfiles/constant.svg" target="./diagrams/constant.svg" alt="syntax diagram for a constant" layout="" -->

<h4 id="interval">interval</h4>

An *interval clause* lets you define time duration constants that can be used in expressions to add or subtract time ranges from a timestamp value.

<!--include-svg src="./svgfiles/interval.svg" target="./diagrams/interval.svg" alt="syntax diagram for an interval"  layout="" -->

<h4 id="timeUnitSpec">timeUnitSpec</h4>

<!--include-svg src="./svgfiles/timeUnitSpec.svg" target="./diagrams/timeUnitSpec.svg" alt="syntax diagram for a time unit specification"  layout="" -->

The following time units are valid:
* Singular form: `SECOND`, `MINUTE`, `DAY`, `MONTH`, `YEAR`
* Plural form: `SECONDS`, `MINUTES`, `DAYS`, `MONTHS`, `YEARS`

Both singular and plural forms can be used interchangeably.

The following example demonstrates how to add and subtract several time units from the current timestamp.

```sql
-- add and subtract several time units from the current timestamp
SELECT
    CURRENT_TIMESTAMP
    - INTERVAL 2 YEARS
    + INTERVAL 1 MONTH
    - INTERVAL 3 DAYS
    + INTERVAL 10 HOURS
    + interval 30 MINUTES
    - INTERVAL 20 SECOND AS past_timestamp
FROM VALUES ("dummy")
```
{: codeblock}


The result of the example query is shown in the following table.

|PAST_TIMESTAMP         |
|-----------------------|
|2016-11-13 20:23:43.621|
<!--table-caption title="Query result for example 'add and subtract several time units from the current timestamp'"-->

Since interval clauses can get quite long, especially if days, hours, minutes, and seconds are involved, it is possible to use an abbreviated syntax by specifying a format `STRING` and by using the `TO` keyword.

The format `STRING` can be used to specify the following time intervals:
* `YEAR TO MONTH` interval by using a format string that complies with `signYEAR-MONTH` with the following:
    * `sign`: optional `+` or `-` sign
    * `YEAR`: number of years
    * `MONTH`: number of months
* `DAY TO SECOND` interval by using a format string that complies with `signDAY HOUR:MINUTE:SECOND` with the following:
    * `sign`: optional `+` or `-` sign
    * `DAY`: number of days
    * `HOUR`: number of hours
    * `MINUTE`: number of minutes
    * `SECOND`: number of seconds

When you specify a time interval by using the keyword `TO`, only the singular form of a time unit `identifier` is supported.

The following table shows equivalent interval clauses:

| INTERVAL expression | INTERVAL TO expression |
| ------------------- | ---------------------- |
| INTERVAL 1 DAY - INTERVAL 2 HOURS - INTERVAL 3 MINUTES - INTERVAL 4 SECONDS | INTERVAL '1 2:3:4.100' DAY TO SECOND |
| INTERVAL 1 YEAR - INTERVAL 2 MONTH | INTERVAL '1-2' YEAR TO MONTH |
<!--table-caption title="Equivalent INTERVAL Clauses"-->

The following example demonstrates equivalent interval expressions when you deal with `YEAR` and `MONTH` time units.

```sql
-- equivalent interval expressions with YEAR and MONTH time units
WITH ts AS (
    SELECT CURRENT_TIMESTAMP AS now FROM VALUES ('dummy')
    )
SELECT
  now - INTERVAL 1 YEAR - INTERVAL 2 MONTH AS LONG_VERSION,
    now - INTERVAL '1-2' YEAR TO MONTH  AS SHORT_VERSION
FROM ts
```
{: codeblock}

The result of the example query is shown in the following table.

| LONG_VERSION            | SHORT_VERSION           |
|-------------------------|-------------------------|
| 2017-08-16 10:38:07.304 | 2017-08-16 10:38:07.304 |
<!--table-caption title="Query result for example 'equivalent interval expressions with YEAR and MONTH time units'"-->

The following example demonstrates equivalent interval expressions, when you deal with `DAY`, `HOUR`, `MINUTE`, and `SECOND` time units.

```sql
-- equivalent interval expressions when you deal with DAY, HOUR, MINUTE and SECOND time units
WITH ts AS (
    SELECT CURRENT_TIMESTAMP AS now FROM VALUES ('dummy')
    )
SELECT
    now - INTERVAL 1 DAY - INTERVAL 2 HOURS - INTERVAL 3 MINUTES - INTERVAL 4 SECONDS AS LONG_VERSION,
    now - INTERVAL '1 2:3:4.100' DAY TO SECOND AS SHORT_VERSION
FROM ts
```
{: codeblock}

The result of the example query is shown in the following table.

| LONG_VERSION           | SHORT_VERSION           |
|------------------------|-------------------------|
| 2018-10-15 08:33:00.84 | 2018-10-15 08:33:00.84  |
<!--table-caption title="Query result for example 'equivalent interval expressions when you deal with DAY, HOUR, MINUTE and SECOND time units'"-->

<h4 id="columnReference">columnReference</h4>

<!--include-svg src="./svgfiles/columnReference.svg" target="./diagrams/columnReference.svg" alt="syntax diagram for a column reference" layout="" -->

<h4 id="qualifiedName">qualifiedName</h4>

<!--include-svg src="./svgfiles/qualifiedName.svg" target="./diagrams/qualifiedName.svg" alt="syntax diagram for a qualified name" layout="" -->

A *qualified name* is a sequence of identifiers that are separated by `.`.
For example, a column name can be qualified by the name of the *relation* the column is defined in.

*Qualified names* are available in the following context:
* [functionOrAggregate](#functionOrAggregate)
* [lateralView](#lateralView)
* [sample](#sample)

<h4>Related references</h4>

For more information about the clauses that are used by a *primary expression*, see the following topics:
* [caseExpression](#caseExpression)
* [castExpression](#castExpression)
* [constant](#constant)
* [columnReference](#columnReference)
* [expression](#expression)
* [functionOrAggregate](#functionOrAggregate)
* [identifier](#identifier)
* [number](#number)
* [query](#query)
* [STRING](#STRING)
* [valueExpression](#valueExpression)
* [timeSeriesExpression](#timeSeriesExpression)

### Predicates
{: #chapterPredicates}

<h4 id="predicate">predicate</h4>

<!--include-svg src="./svgfiles/predicate.svg" target="./diagrams/predicate.svg" alt="syntax diagram for a predicate" layout="" -->

The `BETWEEN ... AND` predicate compares a value with a range of values. If `NOT` is specified, the result is reversed.

The `IN` predicate compares a value or values with a collection of values. The range of values is either defined by a query or a list of expressions that are enclosed in parentheses.
The query must identify a number of columns that is the same as the number of expressions that are specified to the left of the IN keyword.
In addition, the number of elements in the list of expressions must be the same as the number of expressions that are specified to the left of the IN keyword.
If `NOT` is specified, the result is reversed.

The `LIKE` predicate searches for strings that have a certain pattern. The pattern is specified by a string in which certain characters have a special meaning.
* The underscore character `_` represents any single character.
* The percent sign `%` represents a string of zero or more characters.
* Any other character represents itself. Thus trailing blanks in a pattern are part of the pattern.
If `NOT` is specified, the result is reversed.

The `RLIKE` predicate searches for a regular expression pattern in a string. If the pattern expression is found, the result is true.
If the pattern expression is not found, the result is false. If the value of any of the arguments is null, the result of the RLIKE predicate is unknown.
If `NOT` is specified, the result is reversed.

The regular expression pattern must be a Java™ regular expression as defined by Java class `java.util.regex.Pattern`.
Meta characters that start with a `\` must be escaped for the regular expression to work, for example, use `\\d` instead of `\d`
in a pattern string to represent a digit. For more information, such as supported meta characters and predefined character classes, see the latest Java documentation.

The `IS NULL` predicate tests for null values. The result of a NULL predicate cannot be unknown. If the value of the expression is null, the result is true. If the value is not null, the result is false. If `NOT` is specified, the result is reversed.

The `IS DISTINCT FROM` predicate compares two expressions and evaluates to TRUE if their values are not identical. The result of a DISTINCT predicate cannot be null. If `NOT` is specified, the result is reversed. The result of a DISTINCT predicate depends on whether either or both of its input expressions are null:

| Input expressions | IS DISTINCT FROM | IS NOT DISTINCT FROM |
| :----| :---- | :---- |
| Both inputs are non-null. | Evaluates to TRUE if the inputs are not identical and FALSE if they are. Equivalent to the <> operator. | Evaluates to FALSE if the inputs are not identical and TRUE if they are. Equivalent to the = operator. |
| One input is null. | Evaluates to TRUE. | Evaluates to FALSE. |
| Both inputs are null. | Evaluates to FALSE. | Evaluates to TRUE. |
<!--table-caption title="NULL Values and DISTINCT Predicate Value"-->

The following DISTINCT predicates are logically equivalent to the corresponding search conditions:

| DISTINCT predicate | Search condition |
| ------------------ | ---------------- |
| value 1 IS NOT DISTINCT FROM value2 | (value1 IS NOT NULL AND value2 IS NOT NULL AND value1 = value 2) OR (value1 IS NULL AND value2 IS NULL) |
| value 1 IS DISTINCT FROM value2 | NOT (value1 IS NOT DISTINCT FROM value2)|
<!--table-caption title="DISTINCT Predicate and Search Condition"-->

<h4>Examples</h4>

<h5><code>IS DISTINCT FROM</code> Examples</h5>

```sql
-- select all rows with distinct values in column A and B
SELECT * FROM (
    SELECT
        col1 AS a,
        col2  AS b
    FROM VALUES
            (1 , 2),
            (2, 2),
            (null, 2),
            (1, null),
            (2, null),
            (null, null)
) WHERE a IS DISTINCT FROM b
```
{: codeblock}


The result of the example query is shown in the following table.

|A   |B   |
|----|----|
|1   |2   |
|null|2   |
|1   |null|
|2   |null|
<!--table-caption title="Query result for example 'select all rows with distinct values in column A and B'"-->

```sql
-- all rows that have no distinct values in column A and B
SELECT * FROM (
    SELECT
        col1 AS a,
        col2 AS b
    FROM VALUES
            (1, 2),
            (2, 2),
            (null, 2),
            (1, null),
            (2, null),
            (null, null)
) WHERE a IS NOT DISTINCT FROM b
```
{: codeblock}


The result of the example query is shown in the following table.

|A   |B   |
|----|----|
|2   |2   |
|null|null|
<!--table-caption title="Query result for example 'all rows that have no distinct values in column A and B'"-->

<h5><code>BETWEEN ... AND ...</code> Examples</h5>

```sql
-- all employees with a salary between 4000 and 8000
SELECT
    emp.col1 AS emp_id,
    emp.col2 AS salary
FROM VALUES
    (0, 1000),
    (2, 2000),
    (3, 3000),
    (4, 4000),
    (5, 5000),
    (6, 6000),
    (7, null),
    (8, 8000),
    (9,9000) AS emp
WHERE emp.col2 BETWEEN 4000 AND 8000
```
{: codeblock}

The result of the example query is shown in the following table.

|EMP_ID|SALARY|
|------|------|
|4     |4000  |
|5     |5000  |
|6     |6000  |
|8     |8000  |
<!--table-caption title="Query result for example 'all employees with a salary between 4000 and 8000'"-->

```sql
-- all employees with a salary not between 4000 and 8000
SELECT
    emp.col1 AS emp_id,
    emp.col2 AS salary
FROM VALUES
    (0, 1000),
    (2, 2000),
    (3, 3000),
    (4, 4000),
    (5, 5000),
    (6, 6000),
    (7, null),
    (8, 8000),
    (9,9000) AS emp
WHERE emp.col2 NOT BETWEEN 4000 AND 8000
```
{: codeblock}


The result of the example query is shown in the following table.

|EMP_ID|SALARY|
|------|------|
|0     |1000  |
|2     |2000  |
|3     |3000  |
|9     |9000  |
<!--table-caption title="Query result for example 'all employees with a salary between 4000 and 8000'"-->

<h5><code>IN</code> Examples</h5>

```sql
-- all employees working in department D01 or D02
SELECT
    emp.col1 AS emp_id,
    emp.col2 AS emp_dept
FROM VALUES
    (0, 'D01'),
    (2, 'C01'),
    (3, 'C02'),
    (4, 'D01'),
    (5, 'D02'),
    (6, 'C01'),
    (7, 'D01'),
    (8, 'C03'),
    (9,'D01') AS emp
WHERE emp.col2 IN ('D01','D02')
```
{: codeblock}

The result of the example query is shown in the following table.

|EMP_ID|EMP_DEPT|
|------|--------|
|0     |D01     |
|4     |D01     |
|5     |D02     |
|7     |D01     |
|9     |D01     |
<!--table-caption title="Query result for example 'all employees working in department D01 or D02'"-->

```sql
-- all employees that are managing a department
SELECT
    emp.col1 AS emp_id,
    emp.col2 AS emp_dept
FROM VALUES
    (0, 'D01'),
    (2, 'C01'),
    (3, 'C02'),
    (4, 'D01'),
    (5, 'D02'),
    (6, 'C01'),
    (7, 'D01'),
    (8, 'C03'),
    (9,'D01') AS emp
WHERE (emp.col1,emp.col2) IN (
    SELECT
        mgr.col1,
        mgr.col2
    FROM VALUES
        (2, 'C01'),
        (4, 'D01'),
        (5, 'D02') AS mgr
    )
```
{: codeblock}

The result of the example query is shown in the following table.

|EMP_ID|EMP_DEPT|
|------|--------|
|2     |C01     |
|4     |D01     |
|5     |D02     |
<!--table-caption title="Query result for example 'all employees that are managing a department'"-->

<h5><code>LIKE</code> Examples</h5>

```sql
-- all employees that work in a department that starts with letter C
SELECT
    emp.col1 AS emp_id,
    emp.col2 AS emp_dept
FROM VALUES
        (0, 'D01'),
        (2, 'C01'),
        (3, 'C02'),
        (4, 'D01'),
        (5, 'D02'),
        (6, 'C01'),
        (7, 'D01'),
        (8, 'C03'),
        (9, 'D01') AS emp
WHERE emp.col2 LIKE 'C%'
```
{: codeblock}

The result of the example query is shown in the following table.

|EMP_ID|EMP_DEPT|
|------|--------|
|2     |C01     |
|3     |C02     |
|6     |C01     |
|8     |C03     |
<!--table-caption title="Query result for example 'all employees that work in a department that starts with letter C'"-->

```sql
-- all department names that do not start with letter C
SELECT
    DISTINCT emp.col2 AS emp_dept
FROM VALUES
    (0, 'D01'),
    (2, 'C01'),
    (3, 'C02'),
    (4, 'D01'),
    (5, 'D02'),
    (6, 'C01'),
    (7, 'D01'),
    (8, 'C03'),
    (9,'D01') AS emp
WHERE emp.col2 NOT LIKE 'C%'
```
{: codeblock}

The result of the example query is shown in the following table.

|EMP_DEPT|
|--------|
|D01     |
|D02     |
<!--table-caption title="Query result for example 'all department names that do not start with letter C'"-->

<h5><code>RLIKE</code> Examples</h5>

```sql
-- all rows that contain in col2 a value ending with 'bc'
SELECT *
FROM VALUES
    (0, 'Abc'),
    (1, 'xyz abc'),
    (2, 'abcabcabc'),
    (3, 'abc xyzxyz abc'),
    (4, '123 456 789') AS data
WHERE data.col2 RLIKE 'bc$'
```
{: codeblock}

The result of the example query is shown in the following table.

|COL1|COL2          |
|----|--------------|
|0   |Abc           |
|1   |xyz abc       |
|2   |abcabcabc     |
|3   |abc xyzxyz abc|
<!--table-caption title="Query result for example 'all rows that contain in col2 a value ending with bc'"-->


```sql
-- all rows that contain in col2 a sequence of 3 'abc' string occurrences
SELECT *
FROM VALUES
    (0, 'Abc'),
    (1, 'xyz abc'),
    (2, 'abcabcabc'),
    (3, 'abc xyzxyz abc'),
    (4, '123 456 789') AS data
WHERE data.col2 RLIKE '(abc){3}'
```
{: codeblock}


The result of the example query is shown in the following table.

|COL1|COL2     |
|----|---------|
|2   |abcabcabc|
<!--table-caption title="Query result for example 'all rows that contain in col2 a sequence of 3 abc string occurrences'"-->

```sql
-- all rows that contain in col2 a sequence of integer values (3 digits) separated by blank or tab
SELECT *
FROM VALUES
        (0, 'Abc'),
        (1, 'xyz abc'),
        (2, 'abcabcabc'),
        (3, 'abc xyzxyz abc'),
        (4, '123\t456 789') AS data
WHERE data.col2 RLIKE '\\d{3}[ \\t]\\d{3}[ \\t]\\d{3}'
```
{: codeblock}

The result of the example query is shown in the following table.

|COL1|COL2       |
|----|-----------|
|4   |123 456 789|
<!--table-caption title="Query result for example 'all rows that contain in col2 a sequence of integer values (3 digits) separated by blank or tab'"-->

<h5><code>IS NULL</code> Examples</h5>

```sql
--- all employees with missing salary information
SELECT
    emp.col1 AS emp_id,
    emp.col2 AS salary
FROM VALUES
    (0, 1000),
    (2, 2000),
    (3, 3000),
    (4, 4000),
    (5, 5000),
    (6, 6000),
    (7, NULL),
    (8, 8000),
    (9,9000) AS emp
WHERE emp.col2 IS NULL
```
{: codeblock}


The result of the example query is shown in the following table.

|EMP_ID|SALARY|
|------|------|
|7     |null  |
<!--table-caption title="Query result for example 'all employees with missing salary information'"-->

### CAST Expression
{: #chapterCastExpression}

The syntax of a *cast expression* is described by the following syntax diagrams.

The cast specification returns the cast operand (the first operand) cast to the type specified by the data type.

If the specified data type is not supported, you receive an error.

<h4 id="castExpression">castExpression</h4>

<!--include-svg src="./svgfiles/castExpression.svg" target="./diagrams/castExpression.svg" alt="syntax diagram for a cast expression" layout="" -->

In case an expression cannot be cast to the data type specified in the cast expression, the expression result is `null`.

<h4>More topics</h4>

For more information about the clauses that are used by a *cast expression*, see the following topics:
* [dataType](#dataType)
* [expression](#expression)
* [identifier](#identifier)

<h4>Related references</h4>

A *cast expression* is referenced by the following clause:
* [primaryExpression](#primaryExpression)

### Case expressions
{: #chapterCaseExpressions}

A case expression allows an expression to be selected based on the evaluation of one or more conditions.

The syntax of a case expression is described by the following syntax diagrams.

<h4 id="caseExpression">caseExpression</h4>

<!--include-svg src="./svgfiles/caseExpression.svg" target="./diagrams/caseExpression.svg" alt="syntax diagram for a case expression" layout="" -->

The upper path in the syntax diagram represents a *searched when clause*, that is, the `WHEN` keyword follows directly after the `CASE` keyword.
The lower path is a *simple when clause*, that is, an expression follows the `CASE` keyword.

In general, the value of the case expression is the value of the *result expression*, following the first (leftmost) case that evaluates to true.
If no case evaluates to true and the ELSE keyword is present, the result is the value of the ELSE case result expression.
If no case evaluates to true and the ELSE keyword is not present, the result is NULL. When a case evaluates to unknown (because of NULLs), the case is not true and hence is treated the same way as a case that evaluates to false.

When you use the *simple when clause*, the value of the expression before the first WHEN keyword is tested for equality with the value of the expression that follows the WHEN keyword.
Therefore, the data type of the expression before the first WHEN keyword must be comparable to the data types of each expression that follows the WHEN keywords.

A *result expression* is an expression that follows the `THEN` or `ELSE` keywords.

<h4 id="whenClause">whenClause</h4>

<!--include-svg src="./svgfiles/whenClause.svg" target="./diagrams/whenClause.svg" alt="syntax diagram for a when clause" layout="" -->

<h4>Examples</h4>

```sql
-- simple case expression with no ELSE clause
SELECT
    dep.col1 AS dep_id,
    CASE dep.col2
        WHEN 'A' THEN 'Administration'
        WHEN 'B' THEN 'Human Resource'
        WHEN 'C' THEN 'Development'
    END AS dep_name
FROM VALUES (0, 'A'), (1, 'B'), (2, 'C'), (3, 'D'), (4, 'E') AS dep
```
{: codeblock}

The result of the example query is shown in the following table.

|DEP_ID|DEP_NAME      |
|------|--------------|
|0     |Administration|
|1     |Human Resource|
|2     |Development   |
|3     |null          |
|4     |null          |
<!--table-caption title="Query result for example 'simple case expression with no ELSE clause'"-->

```sql
-- simple case expression with ELSE clause
SELECT
    dep.col1 AS dep_id,
    CASE dep.col2
        WHEN 'A' THEN 'Administration'
        WHEN 'B' THEN 'Human Resource'
        WHEN 'C' THEN 'Development'
        ELSE 'UNKOWN'
    END AS dep_name
FROM VALUES (0, 'A'), (1, 'B'), (2, 'C'), (3, 'D'), (4, 'E') AS dep
```
{: codeblock}


The result of the example query is shown in the following table.

|DEP_ID|DEP_NAME      |
|------|--------------|
|0     |Administration|
|1     |Human Resource|
|2     |Development   |
|3     |UNKOWN        |
|4     |UNKOWN        |
<!--table-caption title="Query result for example 'simple case expression with ELSE clause'"-->

There are two scalar functions, `NULLIF()` and `COALESCE()`, that are specialized to handle a subset of the functionality provided by `CASE`.

| Expression | Equivalent Expression |
| ---------- | --------------------- |
| CASE WHEN e1=e2 THEN NULL ELSE e1 END | NULLIF(e1,e2) |
| CASE WHEN e1 IS NOT NULL THEN e1 ELSE e2 END| COALESCE(e1,e2) |
| CASE WHEN e1 IS NOT NULL THEN e1 ELSE COALESCE(e2,...,eN) END| COALESCE(e1,e2,...,eN)|
<!--table-caption title="CASE, NULLIF(), and COALESCE()"-->

For more information, see [SQL functions](/docs/sql-query?topic=sql-query-sqlfunctions#sqlfunctions).

For more information about the clauses that are used by a *case expression*, see the following topic:
* [expression](#expression)

<h4>Related references</h4>

A *case expression* is referenced by the following clause:
* [primaryExpression](#primaryExpression)


### Time series expressions
{: #chapterTimeSeriesExpressions}

The syntax of a *time series expression* is described by the following syntax diagrams.

<h4 id="timeSeriesExpression">timeSeriesExpression</h4>

<!--include-svg src="./svgfiles/timeSeriesExpression.svg" target="./diagrams/timeSeriesExpression.svg" alt="syntax diagram for time series expression" layout="" -->

The syntax shows time series functions that require expressions, such as  `TS_MAP()`,  `TS_FILTER()`, `TS_SEGMENT_BY_ANCHOR()`, `TS_SEGMENT_BY_MARKER()`, `TS_SEGMENT_BY_DUAL_MARKER()`,
`TS_FIND()`, and `TS_COUNT_ANCHOR()`.

For more information on each function, see [Data processing functions](/docs/sql-query?topic=sql-query-data_processing_functions).

<h4>Example</h4>

```sql
WITH timeseries_input AS (SELECT location, TIME_SERIES_WITH_TRS(TS_TIMESTAMP(timestamp), humidity, TS_TRS_DEFAULT()) AS ts
                          FROM cos://us-geo/sql/temperature_humidity.csv STORED AS CSV
                          GROUP BY location),
    only_40_or_above_ts AS (
	    SELECT location,
	   		TS_FILTER(ts, TS_EXP_GT(TS_EXP_ID(), 40.0)) AS above_40_ts
	    FROM timeseries_input
	)
SELECT location, TS_EXPLODE(above_40_ts) AS (timestamp, humidity) FROM only_40_or_above_ts
```
{: codeblock}

A *time series expression* is referenced by the following clause:
* [primaryExpression](#primaryExpression)

<h4 id="booleanTimeSeriesExpression">booleanTimeSeriesExpression</h4>

<!--include-svg src="./svgfiles/booleanTimeSeriesExpression.svg" target="./diagrams/booleanTimeSeriesExpression.svg" alt="syntax diagram for boolean time series expression" layout="" -->

The Boolean time series expression syntax shows the available Boolean expresssions, such as `TS_EXP_GT()`, which is also used in the previous example.

For more information on each function, see [Artifact creation functions](/docs/sql-query?topic=sql-query-artifact).

<h4 id="valueTimeSeriesExpression">valueTimeSeriesExpression</h4>

<!--include-svg src="./svgfiles/valueTimeSeriesExpression.svg" target="./diagrams/valueTimeSeriesExpression.svg" alt="syntax diagram for value time series expression" layout="" -->

Time series values for expressions can either be a `string` or a `double` datatype.

<h4 id="doubleTimeSeriesExpression">doubleTimeSeriesExpression</h4>

<!--include-svg src="./svgfiles/doubleTimeSeriesExpression.svg" target="./diagrams/doubleTimeSeriesExpression.svg" alt="syntax diagram for double time series expression" layout="" -->

The functions shown in the double time series expressions, such as `TS_EXP_ABS()` and `TS_EXP_LENGTH()`, are able to consume again double time series expressions,
`number`, or an identity time series expression.

For more information on each function, see [Artifact creation functions](/docs/sql-query?topic=sql-query-artifact).

<h4 id="stringTimeSeriesExpression">stringTimeSeriesExpression</h4>

<!--include-svg src="./svgfiles/stringTimeSeriesExpression.svg" target="./diagrams/stringTimeSeriesExpression.svg" alt="syntax diagram for string time series expression" layout="" -->

The string function `TS_EXP_ID_TO_STRING()` converts an ID to a string and the `TS_EXP_CONCAT()` function concatenates the result of two string expressions.

For more information on each function see [Artifact creation functions](/docs/sql-query?topic=sql-query-artifact).

<h4 id="stringConditionalExpression">stringConditionalExpression</h4>

<!--include-svg src="./svgfiles/stringConditionalExpression.svg" target="./diagrams/stringConditionalExpression.svg" alt="syntax diagram for string conditional time series expression" layout="" -->

There are three conditional expression functions for string values `TS_EXP_IF_THEN_ELSE()`, `TS_EXP_IF_THEN()`, and `TS_EXP_MATCH_CASE()`.

For more information on each function, see [Artifact creation functions](/docs/sql-query?topic=sql-query-artifact).

<h4 id="identityTimeSeriesExpression">identityTimeSeriesExpression</h4>

<!--include-svg src="./svgfiles/identityTimeSeriesExpression.svg" target="./diagrams/identityTimeSeriesExpression.svg" alt="syntax diagram for identity time series expression" layout="" -->

The identity expression denotes current observation values in time series.

### Operator
{: #chapterOperator}

The following types of operators can be used:
* [Unary](#unaryOperator)
* [Arithmetic](#arithmeticOperator)
* [String](#stringOperator)
* [Comparison](#comparisonOperator)
* [Boolean](#booleanOperator)

<h3 id="unaryOperator">Unary operators</h3>

| Operator | Operand types | Description |
| :----: | ---- | ---- |
| `-A` | All number types | Unary negative operator. The type of the result is the same as the type of A. |
| `+A` | All number types | Unary positive operator. The type of the result is the same as the type of A. |
| `~A` | All number types | Bitwise NOT operator. The type of the result is the same as the type of A. |
<!--table-caption title="Unary operators"-->

<h3 id="arithmeticOperator">Arithmetic operators</h3>

| Operator | Operand types | Description |
| :----: | ---- | ---- |
| `A + B`   | All number types | Returns the result of adding A and B. The type of the result is the same as the type of the operand that is highest in the type hierarchy. For example, if A is of type FLOAT and B is of type INT, the result is of type FLOAT. |
| `A - B`   | All number types | Returns the result of subtracting B from A. The type of the result is the same as the type of the operand that is highest in the type hierarchy. For example, if A is of type FLOAT and B is of type INT, the result is of type FLOAT. |
| `A * B`   | All number types | Returns the result of multiplying A and B. The type of the result is the same as the type of the operand that is highest in the type hierarchy. For example, if A is of type FLOAT and B is of type INT, the result is of type FLOAT. If the operation causes an overflow, cast at least one of the operators to a type that is higher in the type hierarchy. |
| `A / B`   | All number types | Returns the result of dividing A by B. The type of the result is DOUBLE. |
| `A % B`   | All number types | Returns the reminder after dividing A by B.  For example, 13.7 % 3 returns 1.7. The type of the result is the same as the type of the operand that is highest in the type hierarchy. For example, if A is of type FLOAT and B is of type INT, the result is of type FLOAT. |
| `A DIV B` | Integer types    | Returns the integer part of the result of dividing A by B. For example, 13.7 DIV 3 returns the integer 4. |
| `A & B`   | All number types | Returns the result of bitwise AND of A and B. The type of the result is the same as the type of the operand that is highest in the type hierarchy. |
| `A | B` | All number types | Returns the result of bitwise OR of A and B. The type of the result is the same as the type of the operand that is highest in the type hierarchy. |
| `A ^ B`  | All number types | Returns the result of bitwise XOR of A and B. The type of the result is the same as the type of the operand that is highest in the type hierarchy. |
<!--table-caption title="Arithmetic operators"-->

<h3 id="stringOperator">String operator</h3>

| Operator | Operand types | Description |
| :----: | ---- | ---- |
| `A || B` |  All types | Returns the concatenation of A and B. If A or B is not a string, it is first converted into a string type. The result is a string. |
<!--table-caption title="String operator"-->

<h3 id="comparisonOperator">Comparison operators</h3>

| Operator | Operand types | Description |
| :----: | ---- | ---- |
| `A = B` | All primitive types | Returns TRUE if A is equal to B, FALSE otherwise. |
| `A == B`| All primitive types| Synonym for the equal (=) operator. |
| `A <> B` | All primitive types| Returns NULL if A or B is NULL, TRUE if A is not equal to B, FALSE otherwise. |
| `A != B` | All primitive types| Synonym for the not equal (<>) operator.|
| `A < B` | All primitive types| Returns NULL if A or B is NULL, TRUE if A is less than B, FALSE otherwise. |
| `A <= B` | All primitive types| Returns NULL if A or B is NULL, TRUE if A is less than or equal to B, FALSE otherwise. |
| `A !> B` | All primitive types| Returns NULL if A or B is NULL, TRUE if A is not greater than B, FALSE otherwise. |
| `A > B` | All primitive types| Returns NULL if A or B is NULL, TRUE if A is greater than B, FALSE otherwise. |
| `A >= B` |All primitive types | Returns NULL if A or B is NULL, TRUE if A is greater than or equal to B, FALSE otherwise. |
| `A !< B` | All primitive types| Returns NULL if A or B is NULL, TRUE if A is not less than B, FALSE otherwise.|
| `A <=> B` | All primitive types| Returns the same result as the equal (=) operator if both A and B are not NULL, TRUE if both A and B are NULL, FALSE if A or B (but not both) is NULL.|
<!--table-caption title="Comparison operators"-->

<h3 id="booleanOperator">Boolean operators</h3>

| Operator | Operand types | Description |
| :----: | ---- | ---- |
| `NOT A` | Boolean expressions | TRUE if A is FALSE, FALSE otherwise. |
| `A AND B` | Boolean expressions | TRUE if A and B are both TRUE, FALSE otherwise. |
| `A OR B` | Boolean expressions | FALSE if A and B are both FALSE, TRUE otherwise. |
<!--table-caption title="Boolean operators"-->

<h3>Related references</h3>

An *operator* is referenced by [valueExpression](#valueExpression).

## Data Types
{: #dataType}

<!--include-svg src="./svgfiles/dataType.svg" target="./diagrams/dataType.svg" alt="syntax diagram for a data type" layout="" -->

Data types can be either primitive types like numeric or string types, or they can be composite types that are built from other
primitive or composite types. Composite types can have the following structure:

* *Struct* types describe types that are built from a fixed number of named fields. Each field can have its own type.
* *Array* types describe a sequence of elements that can have an arbitrary length, but must all have the same type.
* *Map* types describe a mapping from keys to values.

Composite types can be nested, as in the following example:
```sql
STRUCT<
    firstName: STRING,
    lastName: STRING,
    age: INTEGER,
    addresses: ARRAY<
        STRUCT<
            streetAddress: STRING,
            city: STRING,
            postalCode: STRING,
            country: STRING
        >
    >
>
```
{: codeblock}

Some data formats, particularly CSV, do not support composite types. When your query result contains data with a composite type,
use an [INTO clause](#intoClause) to specify an appropriate target format, like JSON.

<h3 id="primitiveType">primitiveType</h3>

The following primitive types are supported in {{site.data.keyword.sqlquery_short}}:

<h4>Numeric Types</h4>

Numeric data types are summarized in the following table.

| Identifier | Type | Bytes | Minimum Value | Maximum Value |
| :---- | :----: | :----: | :----: | :----: |
| `TINYINT` | signed integer | 1 | -128 | 127 |
| `SMALLINT` | signed integer | 2 | -32768 | 32767 |
| `INT` or `INTEGER` | signed integer | 4 | -2147483648 | 2147483647 |
| `INTEGER` | signed integer | 4 | -2147483648 | 2147483647 |
| `BIGINT` or `LONG` | signed integer | 8 | -9223372036854775808 | 9223372036854775807 |
| `FLOAT` | single precision floating point number | 4 | n/a | n/a |
| `DOUBLE` | double precision floating point number | 8 | n/a | n/a |
| `DECIMAL` | precision of 38 digits | n/a | -10e37+1 | 10e37-1 |
<!--table-caption title="Numeric Data Types"-->

<h4>String Types</h4>

Strings are represented as `STRING` data type. The type definitions `VARCHAR(n)` and `CHAR(n)` can be used as aliases for `STRING`.
The syntax requires that you specify a maximum length for these, but no length restriction is enforced.

<h4>Date and Time Types</h4>

String values with appropriate formats can be converted to a timestamp or date, by using data types `TIMESTAMP` or `DATE`.

```sql
-- cast string values to timestamp and date types
SELECT
    CAST('2018-10-31 23:55:00' AS TIMESTAMP),
    CAST('2018-10-31 23:55:00' AS DATE),
    CAST('HELLO' AS TIMESTAMP)
FROM VALUES ('dummy')
```
{: codeblock}


The result of the example query is shown in the following table.

|CAST(2018-10-31 23:55:00 AS TIMESTAMP)|CAST(2018-2-28 23:55:00 AS DATE)|CAST(HELLO AS TIMESTAMP)|
|--------------------------------------|--------------------------------|------------------------|
|2018-10-31 23:55:00.0                 |2018-02-28                      |null                    |
<!--table-caption title="Query result for example 'cast string values to TIMESTAMP and DATE types'"-->

<h4>Boolean Type</h4>

The `BOOLEAN` type represents a domain with two values, `true` or `false`.

Any numeric value that represents zero, for example, `0`, `0.0`, or `0.0E10`, can be cast to `false`.

Numeric values representing a nonzero value, for example, 1, 1.0, 1.0E10, or 21474.83648 can be cast to `true`.

The string value `'0'` can be cast to `false` and `'1'` can be cast to `true`. Any other string value is cast to `false`.

<h4>Binary Type</h4>

A `BINARY` type represents an array of byte values. Thus, string values can be cast to type `BINARY`.

<h4>Related references</h4>

A *dataType* is referenced by the following clauses:
* [castExpression](#castExpression)
* [createTable](#createTable)


## Catalog management
{: #chapterHiveCatalog}

The following commands allow users to store table metadata catalog in the {{site.data.keyword.sqlquery_short}} catalog.
By defining the tables, columns, and partitions in the catalog, you can use short table names in the SQL SELECT statements. Each instance of {{site.data.keyword.sqlquery_short}} has its own catalog, and table definitions are not visible from other instances.
For more information, see [catalog management](/docs/sql-query?topic=sql-query-hivemetastore).

### Create table
{: #chapterCreateTable}

<h4 id="createTable">createTable</h4>

<!--include-svg src="./svgfiles/createTable.svg" target="./diagrams/createTable.svg" alt="syntax diagram for a create table command" layout="@break@" -->

<h4 id="columnDefinition">columnDefinition</h4>

<!--include-svg src="./svgfiles/columnDefinition.svg" target="./diagrams/columnDefinition.svg" alt="syntax diagram for column definition" layout="@break@" -->

Create a table definition in the catalog based on the objects in the specified {{site.data.keyword.cos_short}} location. The `LOCATION` option is mandatory.
If a table or view with the same name exists in the same {{site.data.keyword.sqlquery_short}} instance, you receive an error, unless the `IF NOT EXISTS` clause is specified.

The column and partition definitions are optional. If they are not provided, the table schema and partitioning is detected from the structure of the data at the indicated location.
If you explicitly provide these definitions, ensure that they match the objects that are stored in {{site.data.keyword.cos_short}}.
See [data types](#dataType) for details on the supported column types.

```sql
-- create a definition for the table customer
CREATE TABLE customers (
  customerID string,
  companyName string,
  contactName string,
  contactTitle string,
  address string,
  city string,
  region string,
  postalCode string,
  country string,
  phone string,
  fax string
)
USING CSV
location  cos://us-geo/sql/customers.csv
```
{: codeblock}

Before you can use a newly created partitioned table, you must call `ALTER TABLE tablename RECOVER PARTITIONS`. Otherwise, querying the table returns an empty result.

```sql
-- create a definition for the table customers_partitioned
CREATE TABLE customers_partitioned (
  customerID string,
  companyName string,
  contactName string,
  contactTitle string,
  address string,
  city string,
  region string,
  postalCode string,
  country string,
  phone string,
  fax string
)
USING CSV
PARTITIONED BY (COUNTRY)
location  cos://us-geo/sql/customers_partitioned.csv

-- attach table partitions by scanning the location of the table
ALTER TABLE customers_partitioned RECOVER PARTITIONS
```
{: codeblock}

An alternative way to create a table definition is to use the automatic schema detection where you do not need to specify any columns.

```sql
-- create a definition for the table shippers with automatic schema detection
CREATE TABLE shippers
USING parquet
location  cos://us-geo/sql/shippers.parquet
```
{: codeblock}

<h4 id="tableProperty">tableProperty</h4>

The tableProperty option consists of one or more key and value pairs.

Key | Value | Default | Description
--- | --- | --- | ---
HEADER | true or false | true | Use the HEADER option to specify whether your CSV object has a header included.
DELIMITER | single (possibly escaped) character | `,` (comma) | Use the DELIMITER option to specify the used delimiter in your CSV objects. All single Unicode characters are allowed as delimiters.
MULTILINE | true or false | false | Use the MULITLINE option to specify if the JSON object is single or multiline.

```sql
-- Example of creating a table definition in the catalog for a CSV data without header line:
CREATE TABLE LAX_CARGO
(Extract_Date timestamp, Report_Date timestamp, Arrival_Departure string,
Domestic_International string, Cargo_Type string, Air_Cargo_Tons int)
USING csv
location cos://us-geo/sql/LAX_Cargo.csv
options(HEADER=false)
```
{: codeblock}

### Drop Table
{: #chapterDropTable}

<h4 id="dropTable">dropTable</h4>

<!--include-svg src="./svgfiles/dropTable.svg" target="./diagrams/dropTable.svg" alt="syntax diagram for a drop table command" layout="@break@" -->

Drop a table definition from the catalog. If the table does not exist, you receive an error, unless the `IF EXISTS` option is specified.

Note: This command does not delete any data in {{site.data.keyword.cos_short}}. It removes only the table definition from the catalog.

```sql
-- drop a definition for the table customer
DROP TABLE customers
```
{: codeblock}

### Create View
{: #chapterCreateView}

<h4 id="createView">createView</h4>

<!--include-svg src="./svgfiles/createView.svg" target="./diagrams/createView.svg" alt="syntax diagram for a create view command" layout="@break@" -->

<h4 id="identifierComment">identifierComment</h4>

<!--include-svg src="./svgfiles/identifierComment.svg" target="./diagrams/identifierComment.svg" alt="syntax diagram for identifier comment definition" layout="@break@" -->

Create a view definition in the catalog, based on existing table and view definitions.
If a table or view with the same name exists in the same {{site.data.keyword.sqlquery_short}} instance, you receive an error, unless the `IF NOT EXISTS` clause is specified.

The query definition is mandatory. It automatically specifies the SQL query that is used, whenever you use the view in a FROM clause of a query.
You can hide some complexity of your data model by creating views on top of your tables. It is also possible to define views on top of other views.

```sql
-- create a view on top of table customer
CREATE VIEW CUSTOMER_STATISTICS AS
    SELECT country, region, count(*) customers
        FROM CUSTOMERS
        WHERE region is NOT NULL
        GROUP BY country, region
```
{: codeblock}

### Drop View
{: #chapterDropView}

<h4 id="dropView">dropView</h4>

<!--include-svg src="./svgfiles/dropView.svg" target="./diagrams/dropView.svg" alt="syntax diagram for a drop view command" layout="@break@" -->

Drop a view definition from the catalog. If the view does not exist, you receive an error, unless the `IF EXISTS` option is specified.

Note: This command does not delete any data in {{site.data.keyword.cos_short}}. It removes only the view definition from the catalog.

```sql
-- drop a view definition for the view customer_statistics
DROP VIEW customer_statistics
```
{: codeblock}

### Alter Table Partitions
{: #chapterAlterTable}

<h4 id="alterTablePartitions">alterTablePartitions</h4>

<!--include-svg src="./svgfiles/alterTablePartitions.svg" target="./diagrams/alterTablePartitions.svg" alt="syntax diagram for a alter table partitions command" layout="@break@" -->

Use alter table to modify the definition of the partitions or to automatically discover the available partitions.

Use the `RECOVER PARTITIONS` option to automatically replace the table partition metadata with the structure that is detected from {{site.data.keyword.cos_short}} data by using the location prefix specified for the table.

```sql
-- replace the table partitions by scanning the location of the table
ALTER TABLE customers_partitioned RECOVER PARTITIONS
```
{: codeblock}

<h4 id="partitionSpec">partitionSpecification</h4>

<!--include-svg src="./svgfiles/partitionSpec.svg" target="./diagrams/partitionSpec.svg" alt="syntax diagram for a partition's specification" layout="@break@" -->

To add or remove partitions individually, use the `ADD PARTITION` or `DROP PARTITION` options.

The `ADD PARTITION` option allows you to specify an explicit location for the new partition. This way, you can construct a table from object locations that do not share a common {{site.data.keyword.cos_short}} prefix, or are even located in separate buckets.
If the partition location is not specified, it is inferred from the location of the table and the value(s) of the partitioning column(s). `ADD PARTITION` does not validate the specified or inferred location.


```sql
-- alter the table partitions by adding a partition
ALTER TABLE customers_partitioned ADD IF NOT EXISTS PARTITION ( COUNTRY = 'Spain') LOCATION cos://us-geo/sql/customers_partitioned.csv/COUNTRY=Spain
-- alter the table partitions by dropping a partition
ALTER TABLE customers_partitioned DROP IF EXISTS PARTITION ( COUNTRY = 'Nowhere')
```
{: codeblock}

The `SET LOCATION` option can be used to change the location of an existing partition.


```sql
-- modify the location of an existing partition
ALTER TABLE customers_partitioned PARTITION (country = 'Spain') SET LOCATION cos://eu-de/sql/customers_partitioned.csv/COUNTRY=Spain
```
{: codeblock}

Use the `EXISTS` option to avoid getting errors during `ADD` or `DROP`.

<!-- HIDE START ### Analyze Table

<h4 id="analyzeTable">Analyze Table</h4>

*!<add comment lines here> include-svg src="./svgfiles/analyzeTable.svg" target="./diagrams/analyzeTable.svg" alt="syntax diagram for a analyze table command" layout="@break@" <add comment lines here>*

The `ANALYZE TABLE` statement collects statistics about the specified table and for the specified columns. This information can be used by the query optimizer to improve the query plan.
For example, to decide which table is smaller when you use a broadcast hash join, add those columns that are used in the SELECT statements.

```sql
<add comment lines here> analyze statistics for the table customer without scanning each object
analyze table customer compute STATISTICS NOSCAN
```
{: codeblock}

The option `NOSCAN` only collects the sizes of the objects. HIDE END -->

### Describe Table
{: #chapterDescribeTable}

<h4 id="describeTable">describeTable</h4>

<!--include-svg src="./svgfiles/describeTable.svg" target="./diagrams/describeTable.svg" alt="syntax diagram for describe tables command" layout="@break@" -->

Return the schema (column names and data types) of a table or view definition. If the table or view does not exist, you receive an error.

```sql
-- returns detailed information about the customer table
DESCRIBE TABLE  customers_partitioned
```
{: codeblock}

### Show Tables
{: #chapterShowTables}

<h4 id="showTables">showTables</h4>

<!--include-svg src="./svgfiles/showTables.svg" target="./diagrams/showTables.svg" alt="syntax diagram for show tables command" layout="@break@" -->

Returns the list of the defined tables and views in the catalog. The `LIKE` option allows to filter for a indicated pattern. Use `*` as wildcard character.

```sql
-- returns all defined tables in the catalog for this instance
SHOW TABLES
```
{: codeblock}

<!-- HIDE START ### Show Table Properties

<h4 id="showTblProperties">Show Table Properties</h4>

*!<add comment lines here>  include-svg src="./svgfiles/showTblProperties.svg" target="./diagrams/showTblProperties.svg" alt="syntax diagram for show table properties" layout="@break@" <add comment lines here>*

<h4 id="tableProperty">Table Property</h4>

*!<add comment lines here>  include-svg src="./svgfiles/tableProperty.svg" target="./diagrams/tableProperty.svg" alt="syntax diagram for table properties" layout="@break@" <add comment lines here>*

*!<add comment lines here>  include-svg src="./svgfiles/tablePropertyKey.svg" target="./diagrams/tablePropertyKey.svg" alt="syntax diagram for table properties" layout="@break@" <add comment lines here>*


Return either all properties of a table definition or a specific property. You receive an error if the table does not exist.

```sql
<add comment lines here> returns all specified table options for the table customer
SHOW TBLPROPERTIES customer
```
{: codeblock} HIDE END -->

### Show Partitions
{: #chapterShowPartitions}

<h4 id="showPartitions">showPartitions</h4>

<!--include-svg src="./svgfiles/showPartitions.svg" target="./diagrams/showPartitions.svg" alt="syntax diagram for show partitions command" layout="@break@" -->

List the defined partitions of a table when a table was created as partitioned. You can filter the returned partitions by using the *partitionSpec* option.


```sql
-- returns all partitions for the table customers_partitioned
SHOW PARTITIONS customers_partitioned
```
{: codeblock}

## Index Management
{: #chapterIndexManagement}

The following commands allow you to create indexes for data skipping during SQL execution to improve performance and lower the costs of your SQL queries.
The indexes store summary metadata for each partition of your table to avoid scanning data that is not needed for the query execution.
For more information, see [index management](/docs/sql-query?topic=sql-query-index_management).

### Create Index
{: #chapterCreateIndex}

<h4 id="createIndex">createIndex</h4>

<!--include-svg src="./svgfiles/metaindexCreateCommand.svg" target="./diagrams/metaindexCreateCommand.svg" alt="syntax diagram for create index command" layout="@break@" -->

Create an index on the objects in the specified {{site.data.keyword.cos_short}} location or on the specified table. Define the required index type for each column that you want to calculate the summary metadata for. Create the index on columns that are used for predicates in the SQL statements.

<h4 id="metaindexIndextype">metaindexIndextype</h4>

<!--include-svg src="./svgfiles/metaindexIndextype.svg" target="./diagrams/metaindexIndextype.svg" alt="syntax diagram for the different index types" layout="@break@" -->

* MINMAX: Stores minimum or maximum values for a column for all types, except for complex types.
* VALUELIST: Stores the list of unique values for the column for all types if the distinct values in that column are low.
* BLOOMFILTER: Uses bloom filter technique for byte, string, long, integer, or short types if the distinct values in that column are high.
* GEOSPATIAL: Stores a geospatial bounding box for geometry types.

```sql
-- create an index on the columns temp, lat, lng, vid and city of the metergen sample table
CREATE METAINDEX
MINMAX FOR temp,
MINMAX FOR lat,
MINMAX FOR lng,
BLOOMFILTER FOR vid,
VALUELIST FOR city
ON cos://us-geo/sql/metergen STORED AS parquet
```
{: codeblock}

```sql
-- create an index on the columns  customerID and city of the sample table CUSTOMERS_PARTITIONED
CREATE METAINDEX
VALUELIST for city,
BLOOMFILTER for customerID
ON TABLE CUSTOMERS_PARTITIONED
```
{: codeblock}

```sql
-- create a geospatial index on the column location of the hospitals sample table
CREATE METAINDEX
GEOSPATIAL FOR location
ON cos://us-geo/sql/hospitals_geometry.parquet STORED AS parquet
```
{: codeblock}

Before you start to use data skipping index management commands, ensure that you set the base location in {{site.data.keyword.cos_short}}, where you want the metadata to be stored. Use the following command:
```sql
-- set the default location for all indexes
ALTER METAINDEX SET LOCATION cos://us-south/<mybucket>/<mypath>
```
{: codeblock}

### Drop Index
{: #chapterDropIndex}

<h4 id="dropIndex">dropIndex</h4>

<!--include-svg src="./svgfiles/metaindexDropCommand.svg" target="./diagrams/metaindexDropCommand.svg" alt="syntax diagram for drop index command" layout="@break@" -->

Drop an existing index based on the objects in the specified {{site.data.keyword.cos_short}} location or on the specified table. Use the following command when the index is no longer needed:

```sql
-- drop the index based on the metergen sample data set
DROP METAINDEX ON cos://us-geo/sql/metergen STORED AS parquet
```
{: codeblock}

### Refresh Index
{: #chapterRefreshIndex}

<h4 id="refreshIndex">refreshIndex</h4>

<!--include-svg src="./svgfiles/metaindexRefreshCommand.svg" target="./diagrams/metaindexRefreshCommand.svg" alt="syntax diagram for refresh index command" layout="@break@" -->

Refresh an existing index based on the objects in the specified {{site.data.keyword.cos_short}} location or on the specified table.
Use the following command if the data changed and you need to update the index:

```sql
-- refresh the index based on metergen sample data set
REFRESH METAINDEX ON cos://us-geo/sql/metergen STORED AS parquet
```
{: codeblock}

### Describe Index
{: #chapterDescribeIndex}

<h4 id="describeIndex">describeIndex</h4>

<!--include-svg src="./svgfiles/metaindexDescribeCommand.svg" target="./diagrams/metaindexDescribeCommand.svg" alt="syntax diagram for describe index command" layout="@break@" -->

Describe an existing index based on the objects in the specified {{site.data.keyword.cos_short}} location or on the specified table.
Use the following command to receive information of the index, such as index status, types that are used, location where it is stored, or number of objects processed.

```sql
-- describe the index based on the metergen sample data set
DESCRIBE METAINDEX ON cos://us-geo/sql/metergen STORED AS parquet
```
{: codeblock}

### Show Indexes
{: #chapterShowIndexes}

<h4 id="showIndexes">showIndexes</h4>

<!--include-svg src="./svgfiles/metaindexShowCommand.svg" target="./diagrams/mmetaindexShowCommand.svg" alt="syntax diagram for show indexes command" layout="@break@" -->

List all stored indexes in the base location. Tables with a different index location are not displayed in the list.

```sql
-- list all Metaindexes in the base location
SHOW METAINDEXES
```
{: codeblock}

### Alter Index
{: #chapterAlterIndex}

<h4 id="alterIndex">alterIndex</h4>

<!--include-svg src="./svgfiles/metaindexLocationCommand.svg" target="./diagrams/metaindexLocationCommand.svg" alt="syntax diagram for alter index command" layout="@break@" -->

You must alter the {{site.data.keyword.cos_short}} location for all indexes only once to define the base location.
If you change it later, {{site.data.keyword.sqlquery_short}} cannot find the index metadata anymore.
Existing index metadata on previous location is not dropped, therefore you can always switch back to the old location when needed.

```sql
-- set the default location for all indexes
ALTER METAINDEX SET LOCATION cos://us-south/<mybucket>/<mypath>/
```
{: codeblock}


### Alter Table Set Location
{: #chapterAlterTableSetLocation}

<h4 id="alterTableSetLocation">alterTableSetLocation</h4>

<!--include-svg src="./svgfiles/hiveMetaindexLocationCommand.svg" target="./diagrams/hiveMetaindexLocationCommand.svg" alt="syntax diagram for alter table set location command" layout="@break@" -->

This command lets you define a location for this specified Hive table. If you change it later, {{site.data.keyword.sqlquery_short}} does not find the index metadata anymore. Existing index metadata on previous location is not dropped, therefore you can always switch back to the old location when needed.

```sql
-- set the index location for the table CUSTOMERS_PARTITIONED
ALTER TABLE CUSTOMERS_PARTITIONED SET METAINDEX LOCATION cos://us-south/<mybucket>/<mypath>
```
{: codeblock}


### Alter Table Drop Location
{: #chapterAlterTableDropLocation}

<h4 id="alterTableDropLocation">alterTableDropLocation</h4>

<!--include-svg src="./svgfiles/hiveMetaindexDropLocationCommand.svg" target="./diagrams/hiveMetaindexDropLocationCommand.svg" alt="syntax diagram for alter table drop location command" layout="@break@" -->

This command allows you to drop a location for the specified table. Use this command if the index metadata is to be fetched from the base location.
The metadata for the index that is stored in {{site.data.keyword.cos_short}} is not dropped and must be cleaned up manually.

```sql
-- set the index location for the table CUSTOMERS_PARTITIONED
ALTER TABLE CUSTOMERS_PARTITIONED DROP METAINDEX LOCATION
```
{: codeblock}

### IndexAsset
{: #chapterIndexAsset}

<h4 id="metaindexAsset">metaindexAsset</h4>

The indexAsset is either based on a table or Cloud {{site.data.keyword.cos_short}} location.

<!--include-svg src="./svgfiles/metaindexAsset.svg" target="./diagrams/metaindexAsset.svg" alt="syntax diagram for index asset" layout="@break@" -->

The metaindexAssetLocation is a subset of the [externalTableSpec](#externalTableSpec).

<!--include-svg src="./svgfiles/metaindexAssetLocation.svg" target="./diagrams/metaindexAssetLocation.svg" alt="syntax diagram for index asset location" layout="@break@" -->

The metaindexAssetHiveTable refers to a Hive table.

<!--include-svg src="./svgfiles/metaindexAssetHiveTable.svg" target="./diagrams/metaindexAssetHiveTable.svg" alt="syntax diagram for index asset Hive table" layout="@break@" -->

## Miscellaneous Definitions
{: #chapterMiscDefinitions}

<h3 id ="COSURI">COSURI</h3>

A Cloud {{site.data.keyword.cos_short}} Uniform Resource Identifier (COS URI) is a string of characters that uniquely identifies an object on Cloud {{site.data.keyword.cos_short}}. By definition URIs are case-insensitive.

The syntax of a COS URI is thoroughly described in section [Table unique resource identifier](https://cloud.ibm.com/docs/sql-query?topic=sql-query-overview#unique).

<h3 id ="CRN_URI">CRN_URI</h3>

A database table CRN is a unique identifier that consists of the CRN of a database service instance and a specific table name that instance.
The user must have access to this service instance and its credentials.

The syntax of a table CRN is thoroughly described in section [Table unique resource identifier](/docs/sql-query?topic=sql-query-overview#unique).

<h3 id ="DB2_TABLE_URI">DB2_TABLE_URI</h3>

A Db2 table URI is a string of characters that uniquely identifies a table in an {{site.data.keyword.Db2_on_Cloud_long}} and {{site.data.keyword.dashdblong}} instance.
The instance must be enabled for IAM and the IBMid if the user must be added as a database user.

The syntax of a Db2 Table URI is thoroughly described in section [Table unique resource identifier](/docs/sql-query?topic=sql-query-overview#unique).

<h3 id="identifier">Identifier</h3>

An *identifier* is a name that uniquely identifies an entity. There are two types of identifiers, unquoted identifiers and back quoted identifiers.

<h4>Unquoted identifier</h4>

An unquoted identifier is at least one character long. The following valid characters can be used:
* Digits `0-9`
* Letters `a-z`, `A-Z`
* Underscore `_`

<h4>Back quoted identifier</h4>

This is an identifier that is embraced by grave accent <code>\`</code> characters. A back quoted identifier can
contain any character. That includes the grave accent character that must be escaped like this <code>\`\`</code>.

The following example shows how to add a column name thata contains a special character:

```sql
SELECT col1 as `Lösung` FROM VALUES 1, 2 ,3
```

<h3 id="tableIdentifier">Table Identifier</h3>

A *table identifier* uniquely identifies a table or view in the catalog of the {{site.data.keyword.sqlquery_short}} instance. The following valid characters can be used:
* Digits `0-9`
* Letters `a-z`, `A-Z`
* Underscore `_`

<h3 id="number">Number</h3>

A number is either a signed or unsigned *integer* or a *decimal* number.

<h3 id="unsignedNumber">Unsigned Number</h3>

An unsigned number is an *integer* or *decimal* number without sign or type suffix.

<h3 id="unsignedInteger">Unsigned Integer Number</h3>

An unsigned integer is an *integer* number without sign or type suffix.

<h4 id="INTEGER_VALUE">Integer Number</h4>

An integer number is represented by a sequence of at least one digit, that is, `0` to `9`.
The integer number can have a suffix denoting the type of integer number. There are three types of integer numbers:
* `Y`: tiny integer number
* `S`: small integer number
* `L`: large integer number

For more information about data types, see [dataType](#dataType).

<h4 id="DECIMAL_VALUE">Decimal Number</h4>

The following is a decimal number:
* A sequence of at least one digit followed by a positive or negative exponent, for example, `10E2` represents integer `1000` and `1E-1` represents 0.1.
* A decimal number, for example, `3.14`.
* A decimal number followed by a positive or negative exponent, for example, `3.14E+3` represents `3140.00` or `3.14E-3` represents `0.00314`.

The decimal number can have a suffix denoting the type of decimal number. There are two types of decimal numbers:
* `D`: double decimal number
* `BD`: large decimal number

For more information about data types, see [dataType](#dataType).

<h3 id="STRING">String</h3>

A *string* is a sequence of arbitrary characters that includes escaped characters, for example,
`\t`, either enclosed in single quotation marks `'`, or double quotation marks, `"`.
To include any quotation mark characters in the string, they must be escaped as <code>\\\\\`</code> or `\\"`.
