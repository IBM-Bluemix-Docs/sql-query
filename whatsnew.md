---

copyright:
  year: 2018, 2021

lastupdated: "2021-06-24"


keywords: what's new, support, release, beta

subcollection: sql-query

---

# What's new

<!--
  When you add new bullets at the top, prefix your commit message with "whatsnew: "
  to trigger an update of the UI build that will make users aware of the new information
-->

## June 2021
<!--- The {{site.data.keyword.sqlquery_full}} UI can now display Parquet results, in addition to CSV and JSON data. --->
- {{site.data.keyword.sqlquery_full}} now supports stream landing that enables you to stream your data in real-time from a topic to a bucket of your choice. This capability enables efficient analytics on the new objects created.
- {{site.data.keyword.cpd_full}} now comes with an integrated connector to {{site.data.keyword.sqlquery_short}} that allows to connect to cloud data lakes and import data assets into projects and catalogs in {{site.data.keyword.cpd_short}}. For more information, see [Connecting to a Cloud Data Lake with IBM Cloud Pak for Data](https://www.ibm.com/cloud/blog/connecting-to-a-cloud-data-lake-with-ibm-cloud-pak-for-data).

## December 2020
- {{site.data.keyword.sqlquery_short}} is available in Chennai, India. When you provision new instances, you can select whether it is being provisioned in Dallas, Frankfurt, or Chennai.
- {{site.data.keyword.cos_full_notm}} web console discovers SQL-queryable objects and folders and directly launches the {{site.data.keyword.sqlquery_short}} web console with a prefilled SQL statement for seamless interactive data exploration.

## November 2020
- The location of Hive partitions can be modified by using the [`ALTER TABLE SET LOCATION`](/docs/sql-query?topic=sql-query-sql-reference#chapterAlterTableSetLocation) feature.

## October 2020
- {{site.data.keyword.sqlquery_short}} index management, also referred to as data skipping is generally available with full production support.
- {{site.data.keyword.sqlquery_full}} comes with open data out of the box, including geolocation, and demographic data that can be used as reference data to combine with your own data sets. It is based on open data from US Census, Eurostat Census, UNdata, OpenStreetMap, and Natural Earth. Explore it by using the new category **Reference data statements** in SAMPLES.
- {{site.data.keyword.sqlquery_short}} time series functions: The anchor functions are deprecated and replaced by the new and more powerful expression creation functions.
- The [ibmcloudsql](https://pypi.org/project/ibmcloudsql) Python SDK significantly expanded in functionality for even more powerful Python analytics with SQL. Take a tour of the functions in the [SQL Query Starter Notebook](https://dataplatform.cloud.ibm.com/exchange/public/entry/view/4a9bb1c816fb1e0f31fec5d580e4e14d). The Python SDK also comes with a dedicated [online documentation](https://ibm-cloud.github.io/sql-query-clients/intro.html#ibmcloudsql).
- The usage of the legacy SoftLayer endpoints of Cloud {{site.data.keyword.cos_full_notm}} is discontinued. Check out the [Cloud Object Storage announcement](https://cloud.ibm.com/status?component=cloud-object-storage&location=global&selected=announcement&query=2020+Actions) for more details.

## September 2020

- You can use our JDBC driver to connect {{site.data.keyword.sqlquery_short}} to business intelligence tools and other applications. To download and configure the driver, see the [JDBC documentation](/docs/sql-query?topic=sql-query-jdbc).
- {{site.data.keyword.sqlquery_short}} supports monitoring metrics for submitted jobs by using
[{{site.data.keyword.mon_full}}](https://cloud.ibm.com/docs/Monitoring-with-Sysdig?topic=Monitoring-with-Sysdig-getting-started).
You can view completed and failed jobs, the number of bytes processed, and the jobs in progress. A default {{site.data.keyword.sqlquery_short}} dashboard exists, and you can define custom dashboards and alerts.

## May 2020

- {{site.data.keyword.sqlquery_short}} supports index management, also referred to as data skipping. Index management can significantly boost performance and reduce cost of your SQL queries by skipping over irrelevant data.![Beta](beta.png)
- Service access role **Manager** is required to run catalog management or index management commands.
- {{site.data.keyword.sqlquery_short}} database catalog support is extended to support [views](/docs/sql-query?topic=sql-query-sql-reference#chapterCreateView).
- {{site.data.keyword.sqlquery_short}} catalog management is out of the Beta stage and can be used with the Standard plan.

## April 2020

- {{site.data.keyword.sqlquery_short}} support for [database catalog](/docs/services/sql-query?topic=sql-query-hivemetastore) is generally available with full production support.

## March 2020

- {{site.data.keyword.sqlquery_short}} has a [database catalog](/docs/services/sql-query?topic=sql-query-hivemetastore). It is based on Hive Metastore and significantly speeds up query execution and decouples data management from SQL users and applications.

## January 2020

- {{site.data.keyword.sqlquery_short}} fully supports all current public and private {{site.data.keyword.cos_full}} endpoints (ending with ".appdomain.cloud", for example, "s3.us.cloud-object-storage.appdomain.cloud") and all new single data center endpoints (for example, sng01).

## December 2019

- You can use [{{site.data.keyword.keymanagementserviceshort}}](https://cloud.ibm.com/catalog/services/key-protect) as a secure credential broker to pass credentials to data resources referenced by your queries, thus ensuring safe handling of your secrets. For more information, see the [authentication documentation](/docs/sql-query?topic=sql-query-authentication).

## November 2019

- You can specify a [`MULTILINE`](/docs/sql-query?topic=sql-query-sql-reference#externalTableSpec) option for JSON input data if individual JSON records are stored across multiple lines.

## October 2019

- The new open source [cos-upload](https://github.com/IBM-Cloud/data-lake/tree/master/upload/cos-upload) script can be used to efficiently upload large volumes of data to {{site.data.keyword.cos_full_notm}} buckets with Aspera by merely providing an IAM API Key.
- {{site.data.keyword.sqlquery_short}} JSON parser is processing and extracting all JSON keys in lowercase, so it can work correctly on LogDNA data.
- Storing new objects in {{site.data.keyword.cos_full_notm}} can trigger {{site.data.keyword.sqlquery_short}} executions. It is enabled by the {{site.data.keyword.cos_full_notm}} [event provider](https://www.ibm.com/cloud/blog/announcements/cloud-object-storage-event-provider-enables-hello-serverless-and-more) for {{site.data.keyword.openwhisk}}. By combining it with the SQL [{{site.data.keyword.openwhisk_short}}](https://hub.docker.com/r/ibmfunctions/sqlquery/), you can automatically run SQL-based post processing for new objects.
- {{site.data.keyword.sqlquery_short}} has query hints for SQL queries that have potential for faster execution by using certain features of {{site.data.keyword.sqlquery_short}}. These hints are flagged with a light bulb icon in the job list and the specific hint is available inside the Details pane.

## September 2019

- Support for ETL to {{site.data.keyword.Db2_on_Cloud_long}}: You can specify [Db2 target tables](/docs/sql-query?topic=sql-query-overview#unique) in your SQL queries to process data from {{site.data.keyword.cos_full_notm}} and save the {{site.data.keyword.sqlquery_short}} result into {{site.data.keyword.Db2_on_Cloud_short}}.

## August 2019

- Support for the [`DESCRIBE`](/docs/sql-query?topic=sql-query-sql-reference#tableTransformer) table transformation function that enables easier exploration of the schema of data on  by returning the schema definition instead of data as the table content. Check out the new Starter Query sample in the UI.

## July 2019

- JSON preview:
  You can directly preview query results in JSON format in the SQL console. Add `INTO <COS URI> STORED AS JSON` to your SQL statement to produce JSON output and preview it in the web console.
- Support for Parquet schema evolution through the [`MERGE SCHEMA`](/docs/sql-query?topic=sql-query-sql-reference#externalTableSpec) sub clause for `STORED AS PARQUET` input data. Check out the new samples in the UI.
- Support for [`CLEANCOLS`](/docs/sql-query?topic=sql-query-sql-reference#tableTransformer) table transformation function that generically cleanses all input column names from characters that are not supported by the Parquet target format. Check out the new Samples.
- Support for [`FLATTEN`](/docs/sql-query?topic=sql-query-sql-reference#tableTransformer) table transformation function that generically flattens all nested input columns into a flat hierarchy, allowing to easily work with, for example, JSON input data and write the results out to flat CSV files. Check out the new samples in the UI.

## June 2019

- {{site.data.keyword.sqlquery_short}} is available in Frankfurt, Germany. When you provision new instances, you can select whether it is being provisioned in Dallas or in Frankfurt.
- Support for [time series SQL functions](/docs/services/sql-query?topic=sql-query-ts_intro) to process time series data, for example,
to identify trends and to predict future values based on these trends.

## May 2019

- Updates to the [SQL reference](/docs/services/sql-query?topic=sql-query-sql-reference):
  - [`JOBPREFIX JOBID/NONE`](/docs/sql-query?topic=sql-query-sql-reference#cosResultClause), you can specify whether you want the job ID to be appended to the target prefix, or not.
  - The [`SORT BY`](/docs/sql-query?topic=sql-query-sql-reference#sortClause) clause for SQL targets is new. You can use it to sort SQL result sets in many ways before writing the results to {{site.data.keyword.cos_full_notm}}. It can be used in combination with [`PARTITIONED BY`](/docs/sql-query?topic=sql-query-sql-reference#partitionedClause), [`PARTITIONED INTO`](/docs/sql-query?topic=sql-query-sql-reference#partitionedClause) (to cluster the results), or without the PARTITIONED clause.
  - [`PARTITIONED INTO BUCKETS`](/docs/sql-query?topic=sql-query-sql-reference#partitionedClause) and [`PARTITIONED INTO OBJECTS`](/docs/sql-query?topic=sql-query-sql-reference#partitionedClause) are both supported, thus you can use them synonymously.

## April 2019

- Support for encrypting SQL queries with {{site.data.keyword.keymanagementservicefull}}. {{site.data.keyword.keymanagementservicelong_notm}} is a centralized key management system (KMS) for generating, managing, and destroying encryption keys used by {{site.data.keyword.Bluemix}} services.
If you are processing sensitive data in your queries, you can use customer-managed keys to encrypt SQL query texts and error messages that are stored in the job information.
- {{site.data.keyword.sqlquery_notm}} with {{site.data.keyword.keymanagementservicelong_notm}} for managing encryption keys meets the required IBM controls that are commensurate with the Health Insurance Portability and Accountability Act of 1996 (HIPAA) Security and Privacy Rule requirements.

## February 2019

- Beta support for JDBC driver of {{site.data.keyword.sqlquery_short}}. Request to participate by sending an email to Joshua.Mintz@ibm.com.
- Beta support for data skipping indexes. You can create custom indexes on any column for minimum and maximum values, list of values,
and geospatial bounding box for any object queried. This significantly reduces I/O and query cost and lower the query execution time.
Request to participate by sending an email to Joshua.Mintz@ibm.com.
- Beta support for SQL-native time series in {{site.data.keyword.sqlquery_short}}. This includes functions for time series segmentation, prediction, alignment, temporal joins, and subsequence mining. Request to participate by sending an email to  Joshua.Mintz@ibm.com.

## December 2018

Release of a complete SQL Reference Guide, an SQL introduction for Cloud SQL/Spark SQL.
This Guide includes examples that can be copied and directly pasted into the web UI to be run.

## November 2018

- Support for controlling the layout of SQL results. Including support for creating hive-style partitioning and paginated result data.
- Support for extensions in [Python SDK](https://pypi.org/project/ibmcloudsql) for result data partitioning, pagination, and exporting SQL job history to {{site.data.keyword.cos_full_notm}}.

## October 2018

Support for SELECT INTO to control the format the SQL result is written in.

## August 2018

- {{site.data.keyword.sqlquery_notm}} is generally available. Its open beta phase ended.
- Support for [new built-in SQL functions](https://issues.apache.org/jira/browse/SPARK-20746) released with Apache Spark 2.3.
- Set of SQL optimizer and ANSI SQL and Hive SQL compliance enhancements that are introduced with Apache Spark 2.3.

## June 2018

- Support for ORC data (STORED AS ORC).
- Support for geospatial SQL functions for calculations, aggregations, and joins on location data.
- Release of [ibmcloudsql Node.js client SDK](https://www.npmjs.com/package/ibmcloudsql).


## April 2018

[{{site.data.keyword.sqlquery_notm}} release beta](https://www.ibm.com/cloud/blog/analyzing-data-with-ibm-cloud-sql-query?mhsrc=ibmsearch_a&mhq=ibm%20sql%20query).
