---

copyright:
  years: 2021
lastupdated: "2021-06-04"

keywords: SQL query, event streams, streaming, cloud object storage, Kafka

subcollection: sql-query

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}

# Stream landing
{:event-streams-landing}

With stream landing you can now stream your data in real-time from a topic to a bucket of your choice.
{{site.data.keyword.sqlquery_full}} connects to {{site.data.keyword.messagehub_full}} and copies the data to Cloud {{site.data.keyword.cos_full}} in Parquet format. This capability enables efficient analytics on the new objects created.

![Kafka Event Streams landing](streaming_diagram.svg)

You can now enable a stream landing job on the {{site.data.keyword.messagehub}} UI by selecting the required resources, such as Cloud {{site.data.keyword.cos_short}} bucket, {{site.data.keyword.keymanagementservicelong}} instance and the {{site.data.keyword.sqlquery_short}} instance by using a tailored wizard. If you want to stop the streaming job, you need to switch to the {{site.data.keyword.messagehub}} UI. For more details on configuring stream landing in {{site.data.keyword.messagehub}}, see [Streaming to Cloud Object Storage by using SQL Query](/docs/EventStreams?topic=EventStreams-streaming_cos_sql).

## Using {{site.data.keyword.sqlquery_short}}
{:using-event-streams}

You can also configure a stream landing job directly as a SQL Query statement, without using the {{site.data.keyword.messagehub}} UI.

For event consumption, {{site.data.keyword.sqlquery_short}} reads data from an {{site.data.keyword.messagehub}} topic with a simple SQL statement, such as the following example:

```
SELECT * FROM crn:v1:bluemix:public:messagehub:us-south:a/2383fabbd90354d33c1abfdf3a9f35d5:4d03d962-bfa5-4dc6-8148-f2f411cb8987::/jsontopic STORED AS JSON 
EMIT cos://us-south/mybucket/events STORED AS PARQUET 
EXECUTE AS crn:v1:bluemix:public:kms:us-south:a/33e58e0da6e6926e09fd68480e66078e:5195f066-6340-4fa2-b189-6255db72c4f2:key:490c8133-5539-4601-9aa3-1d3a11cb9c44
```

The difference from a batch query is that the FROM clause now points to a {{site.data.keyword.messagehub}} topic by using the CRN of an {{site.data.keyword.messagehub}} instance. 
The allowed formats of the events are JSON and AVRO. The EMIT specifies the Cloud {{site.data.keyword.cos_short}} bucket 
and the required format is Parquet. The last option to specify is a valid {{site.data.keyword.keymanagementserviceshort}} key that holds an API key with the permissions to read from {{site.data.keyword.messagehub}} and write to Cloud {{site.data.keyword.cos_short}}. 
The API key is needed, as in theory, the job can run forever. 

```
SELECT * FROM crn:v1:bluemix:public:messagehub:us-south:a/2383fabbd90354d33c1abfdf3a9f35d5:4d03d962-bfa5-4dc6-8148-f2f411cb8987::/avrotopic STORED AS AVRO 
OPTIMIZED EVENT SIZE 1000 BYTES 
EMIT cos://us-south/mybucket/events STORED AS PARQUET 
EXECUTE AS crn:v1:bluemix:public:kms:us-south:a/33e58e0da6e6926e09fd68480e66078e:5195f066-6340-4fa2-b189-6255db72c4f2:key:490c8133-5539-4601-9aa3-1d3a11cb9c44
```

## Data on Cloud {{site.data.keyword.cos_short}}
{:data-on-cos}

In addition to the objects that are written when you do a [batch query](https://cloud.ibm.com/docs/sql-query?topic=sql-query-overview#result=), the following two objects are also created:

- `_checkpoint`: Do not delete or modify the objects in this structure as it stores the last offsets of the topic.
- `_schema_as_json`: Do not delete or modify the objects in this structure as it stores the schema of the Kafka events.

The objects are written to Cloud {{site.data.keyword.cos_short}} in micro batches. The number of events included in one object 
depends on the specified event size and the provision rate of the topic. The same applies to the times when data is written to Cloud {{site.data.keyword.cos_short}}. If the provision rate is high, you see more objects within the same timeframe. If it is low, you see a new object at least every 5 minutes.

## Streaming job details
{:streaming-job-details}

The details of a streaming job show that the states differ from batch query processing. 
There is an additional state, *stopping*, which tells you that {{site.data.keyword.sqlquery_short}} is stopping a running job. 
And instead of the state *completed*, a streaming job goes into the *stopped* state.
The streaming states change from *queued* to *running* to *stopping* or *failed*.

The job details show the following metrics (instead of *rows_returned*, *rows_read* and *bytes_read*):

- `last_change_time`: Shows when the last object was written to Cloud {{site.data.keyword.cos_short}}.
- `rows_per_second`: Shows the number of records that were processed per second in the last micro batch.
- `last_activity_time`: Shows the time when the streaming job was last active, with or without processing rows.

## Estimating cost of stream landing
{:estimating-cost}

To keep the example simple, assume to persist 1 MB per second of data in Cloud {{site.data.keyword.cos_short}} that originates from {{site.data.keyword.messagehub}}. All pricing in this example is in US currency.

Feature | Price
--- | ---
1 {{site.data.keyword.messagehub}} topic with 1 partition | $0.014 USD per partition hour
{{site.data.keyword.messagehub}} outbound bandwidth charge  | $0.028 for 3.6 GB data transmitted per hour
1 {{site.data.keyword.sqlquery_short}} stream landing job | $0.11 per hour
Cloud {{site.data.keyword.cos_short}} Class A requests for writing data | ~$0.02 per hour
Cloud {{site.data.keyword.cos_short}} storage costs | $0.05 per month for each 3.6 GB using the smart storage tier class

Your total cost per hour, with the data subsequently stored for a month, would be approximately: $0.222.
The above is only an example, and you should evaluate your own planned usage with the IBM Cloud cost calculator.

## Permissions
{:permissions-event-streams}

The following permissions are needed for creating a stream landing job: 

- Permission to create service-to-service authentication
- Permission to create service IDs and API keys
- Permission to write to {{site.data.keyword.keymanagementservicelong}} (to store the API key)

## Limitations
{:limitations-streams-landing}

With a stream landing job you can process up to 1 MB event data per second. The final reached data throughput 
depends on parameters, such as topic partitions and size and format of the events. For one {{site.data.keyword.sqlquery_short}} instance 
there is a limit of five concurrent stream landing jobs. The limit can be raised upon request via support ticket. The {{site.data.keyword.messagehub}} feature is currently only available for instances created in the US-South region. 

