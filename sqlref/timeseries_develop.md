---

copyright:
  years: 2019, 2021
lastupdated: "2021-03-02"

keywords: SQL query, time series, SQL, analyze, string matching, temporal join, align, examples

subcollection: sql-query

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}

# Using SQL statements to process time series data
{: #using_sql}

You can call time series functions from within SQL statements to manipulate and analyze time series data.

- [Examples of common time series queries](/docs/services/sql-query?topic=sql-query-examples_common)  
  These examples of SQL queries illustrate commonly used time series functions.
- [String matching](/docs/services/sql-query?topic=sql-query-string_matching)  
  You can match the values of a string time series against a string sequence (sequence matching) or against a set of strings 
  (set matching).
- [Using a temporal join or align](/docs/services/sql-query?topic=sql-query-temporal_align)  
  A temporal join produces a single array time series based on the observations of the two input time series. 
  A temporal align produces two output time series with identical timeticks. In both cases, an interpolator is 
  used to complete missing values.
