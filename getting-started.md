---

copyright:
  year: 2018, 2019
lastupdated: "2019-04-08"

keywords: SQL, query, object storage, bucket, sample, instance, analyze, CSV, JSON, ORC, Parquet

subcollection: sql-query

---

{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:codeblock: .codeblock}
{:pre: .pre}
{:screen: .screen}
{:tip: .tip}


# Getting started
{: #gettingstarted}

In this {{site.data.keyword.sqlquery_full}} tutorial, we'll set you up to begin querying rectangular data in {{site.data.keyword.cos_full}} and storing the results in {{site.data.keyword.cos_short}}.


{: shortdesc}

## Before you begin
{: #prereqs}

Before you can run SQL queries, you need to have one or more Cloud {{site.data.keyword.cos_short}} buckets to hold the data to be analyzed and to hold the query results.
Cloud {{site.data.keyword.cos_short}} offers several plans, including a free "Lite" plan.
To create a Cloud {{site.data.keyword.cos_short}} instance:
1. Go to the [**IBM Cloud Catalog > Storage > Object Storage**](https://console.bluemix.net/catalog/infrastructure/cloud-object-storage) page.
2. Select one of the plans and create the service.  

   If you want to find your Cloud {{site.data.keyword.cos_short}} instance at a later point of time, go to your  
   [**IBM Cloud Dashboard > Services**](https://console.bluemix.net/dashboard/apps). 
(If you do not see it in the list, select the resource group "All Resources".)

You can now manage and browse the buckets and data the instance contains. 
Click [here](https://console.bluemix.net/docs/services/cloud-object-storage/getting-started.html#getting-started-console) 
for more information about how to use Cloud {{site.data.keyword.cos_short}}.

## Step 1: Create your {{site.data.keyword.sqlquery_short}} service instance
{: #sql_query}

1. Go to the [**IBM Cloud Catalog**](https://console.bluemix.net/catalog) and search for **SQL Query**.
2. Click **SQL Query** to open the Catalog details page.
3. Select the Lite plan and Click **Create** to create an instance of the service.
4. Click **Open UI** on the Dashboard page to open the {{site.data.keyword.sqlquery_short}} Console.
When you do this for the first time, the {{site.data.keyword.sqlquery_short}} service automatically creates a bucket for you in your Cloud {{site.data.keyword.cos_short}} instance.
It uses this bucket as the default target for your query results.

## Step 2: Execute one of the samples to see how to use the service
{: #samples}

1. Select a sample query. This loads it into the editor. The input data used by the sample queries is available in a publicly accessible bucket.
2. The Target field is automatically filled in with the unique resource identifier (URI) of your default bucket. You can use this bucket or specify a different URI.
3. Click **Run** to run the query. The query result is displayed below the editor.
4. Each sample data set is available in each of the supported input data formats (CSV, JSON, ORC, and Parquet). 
If you want to experiment with different formats, edit the selected sample query and change the specified file name and format. 
For example, change `orders.parquet STORED AS PARQUET` to `orders.orc STORED AS ORC`.

<!--

BLH; 09 FEB 2018:
I COMMENTED OUT THESE STEPS BECAUSE:
-THEY GO BEYOND JUST GETTING STARTED.
-THEY ARE COVERED WITH THE NECESSARY LEVEL OF DETAIL IN THE OVERVIEW.
-DOING SO SHORTENS THE GETTING STARTED SECTION DOWN TO AN EASILY DIGESTIBLE SIZE AND MAKES IT LESS DAUNTING.

## Step 3: Load your own data into IBM Cloud Object Storage
{: #COS}

Before you can run queries on your own data, you must first upload that data to your Cloud Object Storage instance. 
How to do this is described in the Cloud Object Storage [getting started information](https://console.bluemix.net/docs/services/cloud-object-storage/getting-started.html#getting-started-console)
or [quickstart guide](https://ibm-public-cos.github.io/crs-docs/getting-started-with-cos).

## Step 4: Run an SQL query on your data
{: #query}

1. Specify the [source file](ADD LINK TO NEW SECTION: TABLE URL FORMAT!). 
In your SQL statement, after the FROM keyword, specify the name (including the bucket name) of the file that contains the data you want to query.
If you use a bucket of another user, ensure that you have at least 'Reader' access to that bucket. 
[This article](https://console.bluemix.net/docs/services/cloud-object-storage/iam/buckets.html#bucket-permissions) describes how to manage access to IBM Cloud Object Storage buckets.
2. Specify where to store the query results. If you want to use a bucket other than your default bucket, ensure that you have at least 'Writer' access to that bucket. 
[This article](https://console.bluemix.net/docs/services/cloud-object-storage/iam/buckets.html#bucket-permissions) describes how to manage access to IBM Cloud Object Storage buckets.
In the Target field, specify the name (including the bucket name) of the file that is to contain the result of your query.
If you want to store results directly in the root of the bucket, specify a slash (/) behind the bucket name. 
For example, cos://s3-api.us-geo.objectstorage.softlayer.net/andreabucket/
3. Click **Run** to run your query.

BLH; 09 FEB 2018:
I REMOVED STEP 5 (operationalize) AND ADDED THE VIDEO TO THE VIDEO LIST BECAUSE:
-THE TASK IS MUCH TO ADVANCED FOR GETTING STARTED.
-IT WAS JUST A LINK TO A VIDEO, WITH NO ADDITIONAL INFO, SO DIDN'T DESERVE ITS OWN SECTION.

-->

## Next steps
{: #next_steps}

To analyze your own data (CSV, JSON, ORC, or Parquet), upload it to an Cloud {{site.data.keyword.cos_short}} instance and run SQL queries as described [here](/docs/services/sql-query?topic=sql-query-overview#running).

For more advanced capabilities, check out these video tutorials:
- [How to run basic queries](http://ibm.biz/csq-run-queries)
- [How to connect to IBM Cloud Object Storage Through the Command Line](http://ibm.biz/csq-aws-sli)
- [How to use the SQL Query REST API](http://ibm.biz/csq-use-api)
- [How to operationalize SQL code and call it from an application](http://ibm.biz/csq-e2e)

