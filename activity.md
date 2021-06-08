---

copyright:
  year: 2018, 2021
lastupdated: "2021-06-08"

keywords: activity tracker, activity, event

subcollection: sql-query

---

{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:codeblock: .codeblock}
{:pre: .pre}
{:screen: .screen}
{:tip: .tip}



# Activity Tracker events
{: #activitytracker}

Use the {{site.data.keyword.at_full}} service to track how users and applications interact with {{site.data.keyword.sqlquery_full}}.

The {{site.data.keyword.at_full_notm}} service records user-initiated activities that change the state of a service in {{site.data.keyword.cloud}}.
For more information, see [Getting started with {{site.data.keyword.at_short}}](/docs/Activity-Tracker-with-LogDNA?topic=Activity-Tracker-with-LogDNA-getting-started).

## List of events
{: #events}

The following table lists the actions that generate an event:

Actions  |	Description
--- | ---
`sql-query.sql-job.create` |  An SQL query was submitted.
`sql-query.sql-job.list` | 	List of jobs was retrieved.
`sql-query.sql-job.get` |  Details of a job were retrieved.
`sql-query.catalog-table.list` |  List of catalog tables was retrieved.
`sql-query.catalog-table.get` |  Details of a catalog table were retrieved.
