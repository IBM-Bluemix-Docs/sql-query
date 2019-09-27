---

copyright:
  year: 2018, 2019
lastupdated: "2019-09-25"

keywords: what's new, support, release, beta

subcollection: sql-query

---

# What's new

## September 2019

- Support for ETL to {{site.data.keyword.Db2_on_Cloud_long}}: You can now specify [Db2 target tables](/docs/services/sql-query?topic=sql-query-overview#table-unique-resource-identifier) in your SQL queries in order to process data from {{site.data.keyword.cos_full}} and save the {{site.data.keyword.sqlquery_full}} result into {{site.data.keyword.Db2_on_Cloud_short}}.

## August 2019

- Support for the [`DESCRIBE`](/docs/services/sql-query?topic=sql-query-sql-reference#tableTransformer) table transformation function that enables easy exploration of the schema of data on  by returning the schema definition instead of data as the table content. Check out the new Starter Query sample in the UI.

## July 2019

- JSON preview:
  You can now directly preview query results in JSON format in the SQL console. Simply add `INTO <COS URI> STORED AS JSON` to your SQL     statement to produce JSON output and preview it in the web console.
- Support for Parquet schema evolution via the [`MERGE SCHEMA`](/docs/services/sql-query?topic=sql-query-sql-reference#externalTableSpec) sub clause for `STORED AS PARQUET` input data. Check out the new samples in the UI.
- Support for [`CLEANCOLS`](/docs/services/sql-query?topic=sql-query-sql-reference#tableTransformer) table transformation function that generically cleanses all input column names from characters that are not supported by the Parquet target format. Check out the new Samples.
- Support for [`FLATTEN`](/docs/services/sql-query?topic=sql-query-sql-reference#tableTransformer) table transformation function that generically flattens all nested input columns into a flat hierarchy, allowing to easily work with, for example, JSON input data and write the results out to flat CSV files. Check out the new samples in the UI.

## June 2019

- {{site.data.keyword.sqlquery_short}} is now available in Frankfurt, Germany. When provisioning new instances, you can now select whether it is being provisioned in Dallas or in Frankfurt.
- Support for [time series SQL functions](/docs/services/sql-query?topic=sql-query-ts_intro) to process time series data, for example,
to identify trends and to predict future values based on these trends.

## May 2019

- Updates to the [SQL reference](/docs/services/sql-query?topic=sql-query-sql-reference):
  - With [`JOBPREFIX JOBID/NONE`](/docs/services/sql-query?topic=sql-query-sql-reference#resultClause) you can now specify if you want       the jobid= appended to the target prefix, or not.
  - The [`SORT BY`](/docs/services/sql-query?topic=sql-query-sql-reference#sortClause) clause for SQL targets is new, you can use it to sort SQL result sets in many ways, before writing the results to {{site.data.keyword.cos_full}}. It can be used in combination with [`PARTITIONED BY`](/docs/services/sql-query?topic=sql-query-sql-reference#partitionedClause), [`PARTITIONED INTO`](/docs/services/sql-query?topic=sql-query-sql-reference#partitionedClause) (to cluster the results), or without the PARTITIONED clause.
  - [`PARTITIONED INTO BUCKETS`](/docs/services/sql-query?topic=sql-query-sql-reference#partitionedClause) and [`PARTITIONED INTO OBJECTS`](/docs/services/sql-query?topic=sql-query-sql-reference#partitionedClause) are now both supported, thus can be used synonymously.

## April 2019

- Support for encrypting SQL queries with {{site.data.keyword.keymanagementservicefull}}. {{site.data.keyword.keymanagementservicelong_notm}} is a centralized key management system (KMS) for generating, managing, and destroying encryption keys used by IBM Cloud™ services.
If you are processing particularly sensitive data in your queries, you can now use customer-managed keys to encrypt SQL query texts and error messages that are stored in the job information.
- {{site.data.keyword.sqlquery_notm}} with {{site.data.keyword.keymanagementservicelong_notm}} for managing encryption keys meets the required IBM controls that are commensurate with the Health Insurance Portability and Accountability Act of 1996 (HIPAA) Security and Privacy Rule requirements.

## February 2019

- Beta support for JDBC driver of {{site.data.keyword.sqlquery_short}}. Request to participate by sending an email to cglew@us.ibm.com.
- Beta support for data skipping indexes. You can create custom indexes on any column for minimum and maximum values, list of values,
and geospatial bounding box for any object queried. This significantly reduces I/O and query cost and lower the query execution time.
Request to participate by sending an email to cglew@us.ibm.com.
- Beta support for SQL-native TimeSeries support in {{site.data.keyword.sqlquery_short}}. This includes functions for timeseries segmentation, prediction, alignment,
temporal joins and subsequence mining. Request to participate by sending an email to cglew@us.ibm.com.

## December 2018

Release of a complete SQL Reference Guide, an SQL introduction for Cloud SQL/Spark SQL.
This Guide includes examples that can be copied and directly pasted into the Web UI to be executed.

## November 2018

- Support for controlling the layout of SQL results. Including support for creating hive-style partitioning and paginated result data.
- Support for extensions in [Python SDK](https://pypi.org/project/ibmcloudsql) for result data partitioning, pagination and exporting SQL job history to {{site.data.keyword.cos_full}}.

## October 2018

Support for SELECT INTO to control the format the SQL result is written in.

## August 2018

- {{site.data.keyword.sqlquery_notm}} is now generally available. Its open beta phase has ended.
- Support for [new built-in SQL functions](https://issues.apache.org/jira/browse/SPARK-20746) released with Apache Spark 2.3.
- Set of SQL optimizer and ANSI SQL and Hive SQL compliance enhancements introduced with Apache Spark 2.3.

## June 2018

- Support for ORC data (STORED AS ORC).
- Support for geospatial SQL functions for calculations, aggregations and joins on location data.
- Release of [ibmcloudsql Node.js client SDK](https://www.npmjs.com/package/ibmcloudsql).


## April 2018

[{{site.data.keyword.sqlquery_notm}} release beta](https://www.ibm.com/blogs/bluemix/2018/04/introducing-ibm-cloud-sql-query/).
