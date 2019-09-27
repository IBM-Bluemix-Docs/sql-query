---

copyright:
  years: 2018, 2019
lastupdated: "2019-09-25"

keywords: SQL query, analyze, data, CVS, JSON, ORC, Parquet, Avro, object storage, SELECT, cloud instance, URI, endpoint, api, user roles

subcollection: sql-query

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}

# Overview
{:shortdesc}

{{site.data.keyword.sqlquery_full}} is a fully-managed service that lets you run SQL queries (that is, SELECT statements) to analyze, transform, or clean up rectangular data.
{:shortdesc}

**Note:** You can use {{site.data.keyword.sqlquery_short}} to create SELECT statements only; actions such as CREATE, DELETE, INSERT, and UPDATE are not possible.

Input data is read from CSV, JSON, ORC, Parquet, or AVRO files located in one or more {{site.data.keyword.cos_full}} instances.
Each query result is written to a CSV, JSON, ORC, Parquet, or AVRO file in a Cloud {{site.data.keyword.cos_short}} instance of your choice.
Use the {{site.data.keyword.sqlquery_short}} user interface (UI) to develop your queries and the
[SQL Query REST API](#restapi) to automate them.

![SQL Query Overview](overview.png)

## Where your input data and query results are stored
{: #stored}

Before you can use the {{site.data.keyword.sqlquery_short}} service to run SQL queries, the input data must be uploaded to one or more Cloud {{site.data.keyword.cos_short}} instances.
You must also have at least 'Writer' access to at least one Cloud {{site.data.keyword.cos_short}} bucket, so that result files
(that is, the files containing output data) can be written there.
For more information about Cloud {{site.data.keyword.cos_short}}, including how to provision an instance, create buckets, and upload data, refer to the [Cloud Object Storage Getting Started Guide](https://console.bluemix.net/docs/services/cloud-object-storage/getting-started.html#getting-started-console).

## Running an {{site.data.keyword.sqlquery_short}}
{: #running}

In SQL, the term *query* is just another way of saying *SELECT statement*. To run a query:

1. In the SQL editor field of the {{site.data.keyword.sqlquery_short}} UI, enter a SELECT statement. In this statement:
    - After the FROM keyword, specify one or more [unique resource identifiers](#table-unique-resource-identifier) (URIs).
Each URI can be thought of as a table.
Each URI comprises one or more input files; each input file can be thought of as a table partition.
You must have at least 'Reader' access to the buckets that contain the input files.
    - If the format of the input files is CSV, there is no need to specify a STORED AS clause.
However, if the format is JSON, ORC, Parquet, or AVRO, after the FROM clause, specify STORED AS JSON, STORED AS ORC, STORED AS PARQUET, or STORED AS AVRO as appropriate.
    - If the format of the input files is CSV and a delimiter other than the default `,` (comma) is used, you have to specify the delimiter using the [`FIELDS TERMINATED BY`](/docs/services/sql-query?topic=sql-query-sql-reference#externalTableSpec) clause. All one-character Unicode characters are allowed as delimiters.
    - If the format of the input files is CSV and the files don't have a header line (by default a header line is assumed), you have to specify `NOHEADER` after the `FROM` clause or `STORED AS` clause.
    - If required, you can use JOIN constructs to join data from several input files, even if those files are located in different instances.

2. Below the SELECT statement, in the **Target** field, specify the output [URI](#table-unique-resource-identifier),
that is, the URI of the directory to which the result file is to be written. You must have at least 'Writer' access to the corresponding bucket.
3. Click the **Run** button.
The query result is displayed in the result area of the UI. You can run up to 5 queries simultaneously.

## Object result set
{: #result}

Currently three objects are written as a result set per job:

1. `jobid=<job_id>`
2. `jobid=<job_id>/_SUCCESS`
3. `jobid=<job_id>/<part-number>`

Only one object contains the result set (`jobid=<job_id>/<part-number>`), and the other two are empty and don't contain any data.
It is important not to delete any of the files if you want to use the result set.
Each result is stored with an own job ID prefix that allows you to use the result directly in a query.
When you want to specify a result as input in your SQL query, specify the first (`jobid=<job_id>`) or the third one (`jobid=<job_id>/<part-number>`).

## Table unique resource identifier

There are different types of table unique resource identifiers, depending on the type of target service used.
You can either specify Cloud {{site.data.keyword.cos_short}} locations or database locations.
The latter is currently only supported for target locations of an SQL query, and only for {{site.data.keyword.Db2_on_Cloud_long}} database services.

### Cloud Object Storage locations

This identifier points to a location on Cloud {{site.data.keyword.cos_short}} with a URI format. The access to this Cloud {{site.data.keyword.cos_short}} bucket is performed with the {{site.data.keyword.iamlong}} (IAM) identity
of the user that has submitted the SQL statement to {{site.data.keyword.sqlquery_short}}. So, make sure that the user's IAM identity has the necessary access rights granted on the Cloud {{site.data.keyword.cos_short}} bucket.

The identifier has the form:

&nbsp;&nbsp;**`cos://<endpoint>/<bucket>/<object>`**

where:

**`<endpoint>`**
The [endpoint](#endpoints) of your Cloud {{site.data.keyword.cos_short}} instance, or its [alias](#endpoints).

**`<bucket>`**
The bucket name:
- For an input URI, the bucket that contains the input file or files.
- For an output URI, the bucket to which the output file is to be written.

**`<object>`**
A more exact specification of the file or files:
- The specified object is interpreted in a similar way as listing file system contents with `ls`.
When you specify a folder name, it will match all objects in that folder. When you specify a
specific object name, it only matches that single object. When you specify the string with a * wildcard,
it matches all objects accordingly. For example:
  - The pattern `mydir/test1/` matches all files in the specified directory, including files in any subdirectory of that directory.
  - The pattern `mydir/test1/tr*` matches all files in the specified directory whose names begin with `tr`, plus all files in any subdirectory whose name begins with `tr`.

  Because a pattern might match more than one file, ensure that the schema of each matching file is appropriate within the context of the SELECT statement.
- For an output URI, this is the directory to which the result file is to be written.
Each result file is stored in a separate subfolder with a name that indicates the job ID. The name of the result file is automatically generated and also indicates the job ID.
Consequently, each time a query is run, the result is stored in a different subdirectory of the target directory.
For example, if you specify `mydir/out` or `mydir/out/` as the target directory, the result file is written to `mydir/out/jobid=<job_id>`.

  Note that you can use the output data from one query as input data for subsequent queries:
  - A query can reference the output of single query execution by specifying the fully-qualified object path, including the `jobid=<job_id>` phrase.
  - A query can reference the output of multiple executions of the same query by omitting the `jobid=<job_id>` part in the object name.
  Such a query can then treat `jobid` as if it were a column in the input table (for example, in a WHERE clause).
  This is the Hive-style partitioning concept that Hadoop SQL engines employ for data stored in Hadoop Distributed File System (HDFS).

### Database locations

There are two ways to specify database locations, CRN tables, and Db2 table URIs. Which one you choose
depends on the target database plan and the access you have to that database:

1. If the {{site.data.keyword.Db2_on_Cloud_short}} instance is in an {{site.data.keyword.Bluemix_notm}} account that is accessible to the SQL user who has access to the service credentials for that instance, the database location can be specified using its instance CRN. The access to the database is performed with the "username" and "password" found in the service credentials for this Db2 instance.
  Note that newly created Db2 instances don't have any service credentials; to create them, select the instance in the {{site.data.keyword.Bluemix_notm}} console and choose **Service credentials** > **New credential**.

  This option is typcially used with Db2 lite plans, which provide restricted access for a single user in a shared database. It can also be used with standard plans, but the service credentials for standard plans always allow full admin access. If the SQL user should only have restricted access to the target database, use the next option (option 2).

  The CRN table has the form:

  **`<db service crn>/<table name>`**

  You can retrieve the **`<db service crn>`** by opening the resource list in the {{site.data.keyword.Bluemix_notm}} dashboard. Scroll down to the database service instance and click in any of the columns other than the first column. This opens an overlay pane to the right where you find a field labelled `CRN:` with the value and an option to copy it to your clipboard.

  The **`<table name>`** part specifies the table that will be created in your database. It has the format **`<schemaname>.<tablename>`**.
  If you omit the **`<schemaname>`** part, the table is created in the schema of the `"username"` in the credentials of your database service instance &ndash; for a Db2 lite plan, this is the only schema that you have access to.
  The table name is case-preserving, so use upper case to match database defaults.

  An example for a CRN table is: `crn:v1:bluemix:public:dashdb-for-transactions:us-south:s/c3882b7e-00c4-4e7c-a63b-cded1c298f25:23eb50c5-723d-41e0-b7d8-603feaa79ccc:cf-service-instance:/RWS46052.QUERY_RESULT`

2. If the service credentials for the {{site.data.keyword.Db2_on_Cloud_short}} instance are not accessible to the SQL user, but the instance has been enabled for IAM authentication, the database location can be specified using a URI with the Db2 database host name. The access to this database will be performed with the IAM identity of the user that has submitted the query.
  Before using this option, make sure that the IBMid of the SQL user hase been added as a database user. See section "Console User Experience" in the "User management" documentation of the
  [Db2 Knowledge Center](https://www.ibm.com/support/knowledgecenter/SS6NHC/com.ibm.swg.im.dashdb.security.doc/doc/iam.html).

  This option is not available for Db2 lite plans because they don't support IAM authentication in the database.

  The Db2 Table URI has the form:

  **`db2://<db2 host name>/<table name>`**

  The **`<db2 host name>`** is the host name of the Db2 instance that is used to access the Db2 Web console and is also used for Java database connectivity (JDBC).

  The **`<table name>`** part specifies the table that will be created in your database. It has the format **`<schemaname>.<tablename>`**.
  If you omit the **`<schemaname>`** part, the table is created in the schema of database user that has been created for the IBMid of the SQL query user.
  The table name is case-preserving, so use upper case to match database defaults.

  An example for a Db2 table URI is: `db2://db2w-vqplkwx.us-south.db2w.cloud.ibm.com/MYSCHEMA.QUERY_RESULT`


## Endpoints
{: #endpoints}

Your Cloud {{site.data.keyword.cos_short}} instance will have one of the endpoints shown in the following tables.
To save space, you can use the alias shown instead of the full endpoint name.

Internal Cross-Regional Endpoint Name | Alias
--- | ---
s3-api.us-geo.cloud-object-storage.appdomain.cloud     | us-geo
s3-api.dal-us-geo.cloud-object-storage.appdomain.cloud | dal-us-geo
s3-api.wdc-us-geo.cloud-object-storage.appdomain.cloud | wdc-us-geo
s3-api.sjc-us-geo.cloud-object-storage.appdomain.cloud | sjc-us-geo
s3.eu-geo.cloud-object-storage.appdomain.cloud         | eu-geo
s3.ams-eu-geo.cloud-object-storage.appdomain.cloud     | ams-eu-geo
s3.fra-eu-geo.cloud-object-storage.appdomain.cloud     | fra-eu-geo
s3.mil-eu-geo.cloud-object-storage.appdomain.cloud     | mil-eu-geo
s3.ap-geo.cloud-object-storage.appdomain.cloud         | ap-geo
s3.tok-ap-geo.cloud-object-storage.appdomain.cloud     | tok-ap-geo
s3.seo-ap-geo.cloud-object-storage.appdomain.cloud     | seo-ap-geo
s3.hkg-ap-geo.cloud-object-storage.appdomain.cloud     | hkg-ap-geo


External Cross-Regional Endpoint Name | Alias
--- | ---
s3-api.us-geo.cloud-object-storage.appdomain.cloud     | us-geo
s3-api.dal-us-geo.cloud-object-storage.appdomain.cloud | dal-us-geo
s3-api.wdc-us-geo.cloud-object-storage.appdomain.cloud | wdc-us-geo
s3-api.sjc-us-geo.cloud-object-storage.appdomain.cloud | sjc-us-geo
s3.eu-geo.cloud-object-storage.appdomain.cloud         | eu-geo
s3.ams-eu-geo.cloud-object-storage.appdomain.cloud     | ams-eu-geo
s3.fra-eu-geo.cloud-object-storage.appdomain.cloud     | fra-eu-geo
s3.mil-eu-geo.cloud-object-storage.appdomain.cloud     | mil-eu-geo
s3.ap-geo.cloud-object-storage.appdomain.cloud         | ap-geo
s3.tok-ap-geo.cloud-object-storage.appdomain.cloud     | tok-ap-geo
s3.seo-ap-geo.cloud-object-storage.appdomain.cloud     | seo-ap-geo
s3.hkg-ap-geo.cloud-object-storage.appdomain.cloud     | hkg-ap-geo


Internal Regional Endpoint Name | Alias
--- | ---
s3.eu-de.cloud-object-storage.appdomain.cloud    | eu-de
s3.eu-gb.cloud-object-storage.appdomain.cloud    | eu-gb
s3.us-south.cloud-object-storage.appdomain.cloud | us-south
s3.us-east.cloud-object-storage.appdomain.cloud  | us-east
s3.jp-tok.cloud-object-storage.appdomain.cloud   | jp-tok


External Regional Endpoint Name | Alias
--- | ---
s3.eu-de.cloud-object-storage.appdomain.cloud    | eu-de
s3.eu-gb.cloud-object-storage.appdomain.cloud    | eu-gb
s3.us-south.cloud-object-storage.appdomain.cloud | us-south
s3.us-east.cloud-object-storage.appdomain.cloud  | us-east
s3.jp-tok.cloud-object-storage.appdomain.cloud   | jp-tok


Internal Single-Site Endpoint Name | Alias
--- | ---
s3.ams03.cloud-object-storage.appdomain.cloud    | ams03
s3.che01.cloud-object-storage.appdomain.cloud    | che01
s3.tor01.cloud-object-storage.appdomain.cloud    | tor01
s3.osl01.cloud-object-storage.appdomain.cloud    | osl01
s3.mel01.cloud-object-storage.appdomain.cloud    | mel01
s3.sao01.cloud-object-storage.appdomain.cloud    | sao01

External Single-Site Endpoint Name | Alias
--- | ---
s3.ams03.cloud-object-storage.appdomain.cloud   | ams03
s3.che01.cloud-object-storage.appdomain.cloud   | che01
s3.tor01.cloud-object-storage.appdomain.cloud   | tor01
s3.osl01.cloud-object-storage.appdomain.cloud   | osl01
s3.mel01.cloud-object-storage.appdomain.cloud   | mel01
s3.sao01.cloud-object-storage.appdomain.cloud   | sao01

## Programmatic access
{: #access}

### REST API
{: #restapi}

You can use the [SQL Query service REST API](https://console.bluemix.net/apidocs/sql-query?language=curl)
to run queries and retrieve information about their status. This is especially helpful when writing code that automatically queries data.

**Note:** The Cloud Resource Name (CRN) is a mandatory part of an {{site.data.keyword.sqlquery_short}} REST endpoint call. The CRN Copy button copies your CRN to clipboard and you can just paste it into your API call.

### Python applications and notebooks
{: #python}

For a Python application, you can also use the [ibmcloudsql package](https://pypi.python.org/pypi/ibmcloudsql).
This allows you to use IBM Watson Studio to run queries with {{site.data.keyword.sqlquery_short}} and visualize the query results with one of the various widget libraries available in [Watson Studio](https://console.stage1.bluemix.net/catalog/services/data-science-experience).

Using the ibmcloudsql library, you can also interact with {{site.data.keyword.sqlquery_short}} directly from Watson Studio notebooks.
You can start by [Using IBM Cloud SQL Query notebook](https://dataplatform.ibm.com/analytics/notebooks/v2/656c7d43-7ccd-4e50-a3c0-bbc37c001132/view?access_token=baaa77ad715e17a8f823615d45431329fde0fe92fecb85abb9fc55a877939fe8)
in the [Watson Studio community](https://dataplatform.ibm.com/exchange/public/entry/view/4a9bb1c816fb1e0f31fec5d580e4e14d).

### Cloud functions
{: #cloud}

{{site.data.keyword.sqlquery_short}} is a serverless mechanism to submit SQL queries, making it a very natural match for the serverless
[IBM Cloud Functions](https://www.ibm.com/cloud/functions). Accordingly, you can use the generic
[SQL Cloud function](https://hub.docker.com/r/ibmfunctions/sqlquery) to run {{site.data.keyword.sqlquery_short}} as an IBM Cloud function.

### Geospatial functions
{: #geospatial-functions}

The [Geospatial Toolkit](https://www.ibm.com/support/knowledgecenter/SS6NHC/com.ibm.swg.im.dashdb.analytics.doc/doc/geo_intro.html)
provides a set of [geospatial functions](https://www.ibm.com/support/knowledgecenter/SS6NHC/com.ibm.swg.im.dashdb.analytics.doc/doc/geo_functions.html)
that you can use to efficiently process and index spatial data.
These functions are integrated into the {{site.data.keyword.sqlquery_short}} service and ready for immediate use.
The {{site.data.keyword.sqlquery_short}} service also provides several sample queries that illustrate how
to use these functions.

## Required user roles
{: #user-roles}

The following table shows which user roles are required to invoke a particular service action or API endpoint.
Use this information to decide which access rights to grant your users when creating new user IDs.

Description | Service Action | API Endpoint | Required User Roles
--- | --- | --- | ---
Submit an SQL query | sql-query.api.submit | POST/v2/sql_jobs/ | Manager or Writer
Get info for all submitted jobs | sql-query.api.getalljobs | GET/v2/sql_jobs/ | Manager, Writer, or Reader
Get info for a specific submitted job | sql-query.api.getjobinfo | GET/v2/sql_jobs/{job_id} | Manager, Writer, or Reader

## Limitations
{: #limitations}

- If a JSON, ORC, or Parquet object contains a nested or arrayed structure, using a wildcard (for example, `select * from cos://...`) returns an error such as "Invalid CSV data type used: `struct<nested JSON object>`."
Use one of the following workarounds:
  - For a nested structure, specify the fully nested column name instead of the wildcard, for example, `select address.city from cos://...`.
  - For an array, use the Spark SQL explode() function, for example, `select explode(address.city) from cos://...`.
- If you receive a corrupted result, verify that the source file is correct and that the correct input file format is specified using 'STORED AS' in the SQL statement.
- If you receive an error message stating that some columns are not found in the input columns,
but the columns do exist in the input file, check if the input file format being specified using 'STORED AS'
in the SQL statement is the actual file format of your current file.
- In order to further process CSV output with {{site.data.keyword.sqlquery_short}}, all values have to be contained within one line. The multi-line option is not supported and therefore must be manually changed.
To remove new lines from multi-line column values, use the SQL function `regexp_replace`. For example, a CSV object `data` has an attribute `multi_line` containing values spanning multiple lines. To select a subset of rows based on a `condition` and store it on Cloud {{site.data.keyword.cos_short}} for further processing, a skeleton SQL statement looks like the following:

	`SELECT regexp_replace(multi_line, '[\\r\\n]', ' ') as multi_line FROM data STORED AS CSV WHERE condition`

- Ensure that each SQL query updating the composite object uses the same column select list, meaning that names of columns and sequence
of columns have to be identical. Otherwise, composed objects become unreadable due to incompatible headers stored in different parts of the object.
- Ensure that for growing composite objects, all SQL statements that update the object and introduce new columns to a column selection list, add these columns to the end of the column list. If this is not the case, the structure of the object gets corrupted, causing unreadable objects, corrupted data, or unreliable results.
