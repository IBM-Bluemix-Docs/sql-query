---

copyright:
  year: 2018, 2019
lastupdated: "2019-04-04"

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

Use the {{site.data.keyword.cloudaccesstraillong}} service to track how users and applications interact with {{site.data.keyword.sqlquery_full}}.

The {{site.data.keyword.cloudaccesstraillong_notm}} service records user-initiated activities that change the state of a service in {{site.data.keyword.cloud}}. 
For more information, see [Getting started with Activity Tracker](https://console.bluemix.net/docs/services/cloud-activity-tracker/index.html#getting-started-with-cla).

##List of events
{: #events}

The following table lists the actions that generate an event:

Actions  |	Description
--- | ---
sql-query.sql_job.create | 	An SQL query was submitted.
