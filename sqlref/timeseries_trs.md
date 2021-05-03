---

copyright:
  years: 2019, 2021
lastupdated: "2021-05-03"

keywords: SQL query, time series, SQL function, time reference system, TRS

subcollection: sql-query

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}

# Time reference systems
{: #TRS}

A time reference system (TRS) defines the *granularity* of a time series (that is, the length of each timetick), 
and the date and time from which the timeticks are to be counted (that is, the time series *start time*).

Each TRS comprises two components:  

- A *granularity*, which specifies the length of each timetick. For example, it might specify that each timetick in a time series 
represents 1 second, 5 minutes, 1 hour, 7 days, or any other length of time.
- A zoned datetime that specifies the *start time* of the TRS. For example, a start time of 2019-12-30T14:15:30 specifies that 
all timestamp increments are relative to 30 December 2019 at 14:15:30. A start time can be specified either as a TimestampType or as a string. 

For a time series that uses a TRS, each timetick is relative to the TRS start time. See the following examples:  

- If the TRS granularity is 1 day and the start time is 1990-01-01, a timetick of 10 corresponds to the date 1990-01-11 (that is, 10 days after 1990-01-01).
- If the TRS granularity is 1 millisecond and the start time is 1980-1-1T00:00:00.000, a timetick of 1546300800004 corresponds to the timestamp 2029-1-1T00:00:00.004.

Generally speaking, dates and timestamps are easier for humans to read, and timeticks are better suited for machine processing.
Consider the following simple time series:  

`[(0, 1.0), (1, 2.0), (2, 4.0)]`  

Using a TRS that has a time granularity of 5 minutes and a start time of 50 years past the Unix epoch ("2020-01-01T00:00") results in the following new set of timestamps:  

`[(0=2020-01-01T00:00, 1.0), (1=2020-01-01T00:05, 2.0), (2=2020-01-01T00:10, 4.0)]`  

Note: When you use a TRS to convert a timestamp to a numeric representation, if the time granularity of the TRS is coarser than the granularity of the input timestamp, precision is lost. For example, if the time granularity of a timestamp is 1 second and you use a TRS with a granularity of 1 minute, converting the timestamp 9:05:43 results in a numeric representation that corresponds to 9:05:00 (the seconds are truncated). However, if the time granularity of the TRS is finer than an integral divisor of the granularity of the input time series (for example, 1 second for the TRS and 5 seconds for the input time series), forward projection is lossless.  

## Associating a time series with a TRS
{: #associating}

To associate a time series with a TRS, use one of the creation functions that have WITH_TRS as part of their names. 
A time series can be associated with a TRS only at the time that the time series is created; a TRS cannot be added to a time series retroactively. However, you can use the TS_WITH_TRS function to change the TRS with which a time series is associated.

## Duplicate timestamps
{: #duplicate}

Changing the TRS can result in duplicate timestamps, for example when the time granularity of the new TRS is coarser than the granularity of the old TRS. If necessary, you can use [combiner functions](/docs/services/sql-query?topic=sql-query-artifact#combiner_creation) to combine observations that have duplicate timestamps into a single observation, for example, by calculating the average of their values.
