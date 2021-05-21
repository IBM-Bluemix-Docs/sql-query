---

copyright:
  years: 2021
lastupdated: "2021-05-21"

keywords: SQL query, event streams, streaming, cloud object storage, Kafka

subcollection: sql-query

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}

# Kafka {{site.data.keyword.messagehub}} Landing
{:event-streams-landing}

With Stream Landing you can now stream your data in real-time from a topic to a bucket of your choice.
{{site.data.keyword.sqlquery_full}} connects to {{site.data.keyword.messagehub_full}} and copies the data to Cloud {{site.data.keyword.cos_full}} in Parquet format. This capability enables efficient analytics on the new objects created.

![Kafka Event Streams landing](streaming_diagram.svg)

You can enable Kafka {{site.data.keyword.messagehub}} landing on the {{site.data.keyword.messagehub}} UI by selecting the required resources, such as Cloud {{site.data.keyword.cos_short}} bucket, {{site.data.keyword.keymanagementservicelong}} instance and the {{site.data.keyword.sqlquery_short}} instance by using a tailored wizard. In the {{site.data.keyword.sqlquery_short}} UI you have an overview of streaming jobs that are running. If you want to stop the streaming job, you need to switch to the {{site.data.keyword.messagehub}} UI. There you get an overview of which Topic is currently consumed by one or more {{site.data.keyword.sqlquery_short}} instances. For more details on streaming in {{site.data.keyword.messagehub}}, see [Streaming to Cloud Object Storage by using SQL Query](/docs/EventStreams?topic=EventStreams-streaming_cos_sql).

## Using Kafka 
{:using-event-streams}

For event consumption, {{site.data.keyword.sqlquery_short}} reads data from a Kafka topic with a simple SQL statement, such as the following example:

```
SELECT * FROM crn:v1:bluemix:public:messagehub:us-south:a/2383fabbd90354d33c1abfdf3a9f35d5:4d03d962-bfa5-4dc6-8148-f2f411cb8987::/jsontopic STORED AS JSON 
EMIT cos://us-south/mybucket/events STORED AS PARQUET 
EXECUTE AS crn:v1:bluemix:public:kms:us-south:a/33e58e0da6e6926e09fd68480e66078e:5195f066-6340-4fa2-b189-6255db72c4f2:key:490c8133-5539-4601-9aa3-1d3a11cb9c44
```

The difference is that the FROM clause now points to a Kafka topic by using the CRN of an {{site.data.keyword.messagehub}} instance. 
The allowed formats of the events are JSON and AVRO. The EMIT specifies the Cloud {{site.data.keyword.cos_short}} bucket 
and the required format is Parquet. The last option to specify is a valid {{site.data.keyword.keymanagementserviceshort}} key that holds an API key with the permissions to read from {{site.data.keyword.messagehub}} and write to Cloud {{site.data.keyword.cos_short}}. 
The API key is needed, as in theory, the job can run forever. 

The optional hint parameter, OPTIMIZED EVENT SIZE n BYES, is available to specify the typical size of an event in {{site.data.keyword.sqlquery_short}}. {{site.data.keyword.sqlquery_short}} then uses an optimized micro batch size for the available memory. If you do not know the typical size of an event, {{site.data.keyword.sqlquery_short}} starts to run with a default setting to consume the Kafka events.

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

## Billing example
{:billing-example}

With a simple capacity metric, you are charged by the hour for each {{site.data.keyword.messagehub}} job enabling you to stream a single topic to a single bucket and then scale up as your workload increases.

For more detailed information on how billing is calculated, see the following examples: 

- 1 topic, 8 partitions, 1 MB ingress costs $0.014 USD per partition hour * 8 * 30 days
- SQL Query landing costs $0.11 USD per capacity unit hour * 1 * 30 days
- Cloud {{site.data.keyword.cos_short}}: 30 days * 24 hours * 60 / 1024 /1024 equals to 2.47 TB per month (Standard Plan, US-South, regional) costs $0.0220 * 2.47 + $1 for Class A and Class B requests

## Permissions
{:permissions-event-streams}

The following permissions are needed for Kafka Landing: 

- Permission to create service-to-service authentication
- Permission to create service IDs and API keys
- Permission to write to {{site.data.keyword.keymanagementservicelong}} (to store the API key)

## Limitations
{:limitations-streams-landing}

With {{site.data.keyword.sqlquery_short}} you can process up to 1 MB event data per second. The final reached data throughput 
depends on parameters, such as topic partitions and size and format of the events. For one {{site.data.keyword.sqlquery_short}} instance 
there is a limit of five concurrent stream landing jobs. The limit can be raised upon request via support ticket. The {{site.data.keyword.messagehub}} feature is currently only available for instances created in the US-South region. 

