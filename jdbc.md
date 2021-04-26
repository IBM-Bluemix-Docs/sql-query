---

copyright:
  years: 2020, 2021
lastupdated: "2021-04-23"

keywords: jdbc

subcollection: sql-query

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}
{:note: .note}
{:deprecated: .deprecated}
{:important: .important}

# JDBC driver
{: #jdbc}

## Driver download
{: #driver_download}

Download the latest version: [`2.5.20`](https://us.sql-query.cloud.ibm.com/download/jdbc/ibmcloudsql-jdbc-2.5.20.jar)

Here you find the previous version for reference: [`2.5.16`](https://us.sql-query.cloud.ibm.com/download/jdbc/ibmcloudsql-jdbc-2.5.16.jar)


## JDBC driver class and URL format
{: #jdbc_class}

The Java™ class name for the JDBC driver is `com.ibm.cloud.sql.jdbc.Driver`.

The JDBC URL needs to include the CRN of an {{site.data.keyword.sqlquery_full}} instance, and must match the following schema:

`jdbc:ibmcloudsql:{instance-crn}[?{key1}={value1}&{key2}={value2}...]`

The CRN can be obtained from the service dashboard in the IBM Cloud console. The following is an example of a JDBC URL:

`jdbc:ibmcloudsql:crn:v1:bluemix:public:sql-query:eu-de:a/290ec9931c0737248f3dc2aa57187d14%3AAf86d20c4-7646-441e-ad4b-182c57008471::?targetcosurl=cos://us-geo/sample-result-bucket/results/`

You can also easily obtain the JDBC URL from the {{site.data.keyword.sqlquery_short}} UI from the **Connect** tab.

Connection properties (except for the CRN) can be specified as part of the URL, separated by `&`, or through the Java connection properties object. The following properties are supported:

- Password (required): IBM Cloud API key for running the queries.
- User (optional): A username is not required and is ignored if given.
- Targetcosurl (optional, but usually needed): Cloud {{site.data.keyword.cos_short}} URI in SQL query style, where the results are stored. If this property is not specified, you cannot run queries that return a JDBC result set. The JDBC connection can still be used to retrieve database metadata and run DDL and [ETL-type statements](#etl-type-statements).
- LoggerFile (optional, default none): file to write driver logs to.
- LoggerLevel (optional, default set by Java SE Development Kit): java.util.logging level for the driver. Java SE Development Kit default is usually `INFO`.
  - DEBUG/FINER or TRACE/FINEST are the most useful values.
- FilterType (optional, default none):
  - Only tables are returned if `filterType` value is set to `table`.
  - Only views are returned if `filterType` value is set to `view`.
- AppendInto (optional, default *true*):
  - If it is set to false, no `INTO` clause is appended, and results are not available through the driver. It is used with [ETL-type statements](#etl-type-statements), where the INTO options are provided as part of the statement.

## Driver functionality
{: #driver_functionality}

This driver is designed as a facade to JVM applications for easy access to {{site.data.keyword.sqlquery_short}} service. The driver uses the REST API of the service to run queries, stores the results in Cloud {{site.data.keyword.cos_short}}, and makes these results accessible through JDBC interfaces.

The driver does not implement full JDBC-compliant functionality, but only the parts of the API that are most commonly used by clients. The following functions are explicitly tested:

- Retrieving query results with the following primitive SQL types: `STRING` or `VARCHAR`, `INTEGER`, `LONG`, `FLOAT`, `DOUBLE`, `DECIMAL`, `DATE`, `TIME`, `TIMESTAMP`. `TIMESTAMP` columns are handled as Coordinated Universal Time (UTC).
- Inspecting result columns and types through the JDBC ResultSetMetaData interface.
- Running DDL statements `CREATE` or `DROP TABLE` to manage objects in the [{{site.data.keyword.sqlquery_short}} catalog](/docs/sql-query?topic=sql-query-hivemetastore).
- Accessing a list of tables and table columns through the JDBC DatabaseMetaData interface. The schema information is retrieved from the {{site.data.keyword.sqlquery_short}} catalog.

The SQL syntax itself is the full syntax that is supported by {{site.data.keyword.sqlquery_short}} except for the `INTO` clause. An `INTO` clause is implicitly added by the driver based on the Cloud {{site.data.keyword.cos_short}} location that is given with the `targetcosurl` connection property.

Results in the `targetcosurl` location are never deleted by the driver. You can use [expiration rules](/docs/cloud-object-storage?topic=cloud-object-storage-expiry) to clean up your results on the IBM Cloud automatically.

## Basic limitations
{: #basic_limitations}

The following limitations are implied by the use of {{site.data.keyword.sqlquery_short}}, which is not a full-featured database:

- The driver is based on an asynchronous REST API and does not maintain persistent "connections" to the backend.
- {{site.data.keyword.sqlquery_short}} is not designed for interactive performance on small data. Even tiny queries usually take several seconds to run.
- Query results are returned through results in Cloud {{site.data.keyword.cos_short}}. A `SELECT *` query creates a full copy of the selected table (or Cloud {{site.data.keyword.cos_short}}) in the `targetcosurl` location.
- Streaming of query results cannot start until the execution fully completed and results were written to Cloud {{site.data.keyword.cos_short}}.
- {{site.data.keyword.sqlquery_short}} works on read-only data in Cloud {{site.data.keyword.cos_short}}, so the following functionality that is related to data updates is not supported:
  - Transactions are not supported. `commit()` and `rollback()` are no-ops.
  - Result sets that can be updated are not supported.
- SQL statements cannot be canceled.

## Driver-specific limitations
{: #driver_limitations}

See the following technical limitations of the driver:

- Only primitive SQL types are supported in the result. Types like `STRUCT`, `ARRAY, `LOB`, or `ROWID` cannot be retrieved from the result.
- When you access the result, use the getter method, matching the result column type, for example, `getInt()` to retrieve an INTEGER column. Some implicit conversions are supported, for example, accessing an INTEGER column with `getLong()`, but not all conversions are implemented.
- All result sets are FORWARD_ONLY and can be iterated with the `ResultSet.next()` method only. Determining the cursor position and explicitly moving the cursor is not supported.
- Prepared statements are not supported.
- Batch execution with the `Statement.addBatch()` and `executeBatch()` methods is not supported.
- Various other uncommon JDBC methods are not implemented.

## ETL-type statements
{: #etl_statements}

You can use the JDBC driver to execute transform-type statements on your data in Cloud {{site.data.keyword.cos_short}}, where you do not want to retrieve the data in the JDBC client itself. See the following example of such a statement:

```
SELECT ...
FROM cos://source1 JOIN cos://source2 ON ... JOIN ...
INTO cos://target STORED AS ... PARTITIONED BY ...
```

You can run this type of statement (with a user-defined INTO clause) by the `Statement.executeUpdate` method. This statement does not give you a result set in the JDBC client (and avoid all overhead for result fetching), but creates a target object with the specific format and partitioning that you need.

You cannot run statements with an `INTO` clause by using the generic `Statement.execute()` method because the driver cannot detect that these statements should be treated as an update, not a query. Therefore, the driver generates a second `INTO` clause, leading to a syntax error.

## JDBC driver logging
{: #jdbc_driver_logging}

JDBC driver logging works similarly to the [postgresql JDBC driver](https://jdbc.postgresql.org/documentation/head/logging.html):

- Logging uses the java.util.logging framework and can be [configured](https://docs.oracle.com/javase/8/docs/api/java/util/logging/LogManager.html) with a configuration file. The file name must be specified as Java™ system property `-D java.util.logging.config.file=<path>`. Java SE Development Kit default is usually to log to the console at `INFO` level.
- The base logger for the JDBC driver is `com.ibm.cloud.sql.query`.
- For convenience, and in cases where JVM system properties are not under your control, two connection properties exist, "loggerFile" and "loggerLevel" that allow to control JDBC driver logging with the JDBC URL. These properties set the log level and install a file handler for the driver base logger. For example, append `&loggerLevel=debug&loggerFile=/tmp/sqlquery.log` to the JDBC URL to create detailed logging output in files `/tmp/sqlquery.log.<n>`.
- If you set loggerLevel higher than `INFO` (for example, `DEBUG`) has no effect if you do not also configure loggerFile or install a log handler because the default console *handler* suppresses all messages with a log level higher than `INFO`.

## Using the driver with Tableau Desktop
{: #using_tableau}

[Tableau Desktop](https://www.tableau.com/products/desktop) is a BI reporting tool that connects to a rich set of data sources. You can connect to any custom JDBC driver by using the generic JDBC connector offered by Tableau. Download Tableau Desktop version 2020.2 or newer.

To make sure that Tableau generates SQL that is supported only by a specific JDBC driver, you must specify the supported and unsupported SQL capabilities of the driver. Tableau generates appropriate SQL statements dependent on this specification.

The following steps describe how to make Tableau Desktop for Windows work with the {{site.data.keyword.sqlquery_short}} JDBC driver:

1. Install Tableau Desktop for Windows.

2. Download the {{site.data.keyword.sqlquery_short}} JDBC driver and copy it to the installation directory of Tableau.

   - For **Windows**: `C:\Program Files\Tableau\Drivers\ibmcloudsql-jdbc-<version>.jar`

   - For **Mac**: `~/Library/Tableau/Drivers/ibmcloudsql-jdbc-<version>.jar`

3. Create a Tableau Datasource Customization file (*.tdc) with the following content:

  ```
  <connection-customization class='genericjdbc' enabled='true' version='2020.2'>
  <vendor name='genericjdbc' />
    <driver name='ibmcloudsql' />
        <customizations>
        <customization name='CAP_CREATE_TEMP_TABLES' value='no'/>
        <customization name='CAP_SELECT_TOP_INTO' value='no'/>
        <customization name='CAP_QUERY_TOP_N' value='yes'/>
        <customization name='CAP_QUERY_TOPSTYLE_TOP' value='no'/>
        <customization name='CAP_QUERY_TOPSTYLE_ROWNUM' value='no'/>
        <customization name='CAP_QUERY_TOPSTYLE_LIMIT' value='yes'/>
        <customization name='CAP_QUERY_SUBQUERIES_WITH_TOP' value='no'/>
        <customization name='CAP_QUERY_GROUP_BY_DEGREE' value='no'/>`
        <customization name='CAP_JDBC_SUPPRESS_ENUMERATE_DATABASES' value='yes'/>
        <customization name='CAP_JDBC_SUPPRESS_ENUMERATE_SCHEMAS' value='yes'/>
       </customizations>
</connection-customization>
```

   - Store the content in a *.tdc file in the following folder:

     **Windows**: `C:\Documents\My Tableau Repository\Datasources\ibmcloudsql-jdbc.tdc`

     **Mac**: `~/My Tableau Repository/Datasources/ibmcloudsql-jdbc.tdc`

     If further customization is needed in future, look [here](https://help.tableau.com/current/pro/desktop/en-us/jdbc_capabilities.htm) for capabilities that can be turned on and off.

4. Start Tableau Desktop. Go to **Connect -> To a Server -> More**.
5. On the next page, you see a list of supported connectors. Select **Other Databases (JDBC)**.
6. On the raised input form enter the following information:
   - URL: `jdbc:ibmcloudsql:{CRN of your {{site.data.keyword.sqlquery_short}} service instance}?targetcosurl={COS location for results}`
   - Dialect: `SQL 92`
   - User: `apikey`
   - Password: `{your IBM Cloud API Key}`
7. Click **Sign in**.


As Tableau does not support complex data types, such as `struct`, if a table contains one or more columns with complex data type, perform the following actions:

- Create a flattened view on the table.
- You can set the `filterType` option to `view`. This option effectively hides all tables and reveals views only to Tableau. To set the option with the JDBC URL, use the following URL:

  `jdbc:ibmcloudsql:{CRN of your {{site.data.keyword.sqlquery_short}} service instance}?targetcosurl={COS location for results}&filterType=view`

## ODBC connectivity using the JDBC driver with Progress Data Direct HDP
{: #odbc_connectivity}

The JDBC driver enables BI Tools to integrate data on Cloud {{site.data.keyword.cos_short}} into business reports.
However, some BI tools exist that only provide ODBC connectors but no JDBC connectors.
If you use such business intelligence tools, you need another way to connect to your data.

An easy way to set up ODBC connectivity to {{site.data.keyword.sqlquery_short}} is to use the [Progress DataDirect Hybrid Data Pipeline](https://www.progress.com/cloud-and-hybrid-data-integration) product.

![Progress DataDirect Hybrid Data Pipeline](ODBCIBMSQLQuery.svg)

There is a [step-by-step tutorial](https://www.ibm.com/cloud/blog/odbc-connectivity-to-ibm-cloud-sql-query) explaining how easily IBM Cloud SQL Query and Hybrid Data Pipeline (HDP) can complement one another to achieve this.
