---

copyright:
  years: 2019
lastupdated: "2019-07-17"

keywords: SQL query, time series, SQL, cleaning, resampling, examples, common functions, converting, timeticks, segmentation, TRS

subcollection: sql-query

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}

# Examples of common timeseries functions
{: #examples_common}

These examples of SQL queries illustrate commonly used time series functions.

## Converting table data to a time series
{: #converting}

Consider the following table, which is stored in a Parquet object with the name cos://us-geo/sql/timeseries.parquet.

| timetick | value |
|-----------|-------|
| 1         | "a"   |
| 3         | "b"   |
| 5         | "c"   |
| 5         | "d"   |
| 5         | "e"   |
| 7         | "f"   |
| 9         | "g"   |

The following SQL statement creates a new table with the name ts_table:

```sql
SELECT TIME_SERIES_S(timetick, value) as ts   
FROM cos://us/sql/timeseries.parquet STORED AS PARQUET   
INTO <your target location> STORED AS PARQUET
```
{: codeblock}

The output object (timeseries.parquet) contains a table with a single column named ts that contains the output value.


The data type of column ts in the following table is DoubleTimeSeries.

| ts                                                                    |
|-----------------------------------------------------------------------|
| [(1,"a"), (3, "b"), (5, "c"), (5, "d"), (5, "e"), (7, "f"), (9, "g")] |

## Combining observations that have identical timeticks
{: #combining}

Consider the following input StringTimeSeries, which is stored in a table column with the name ts1a:

`[(1,"a"), (3, "b"), (5, "c"), (5, "d"), (5, "e"), (7, "f"), (9, "g")]`

The following SELECT statement combines the observations that have identical time-stamps into a single observation whose value is the original values separated by an underscore (_) character:

```sql
SELECT TS_COMBINE_DUPLICATE_TIMETICKS(ts1a, TS_COMBINER_CONCATENATE("_"))    
FROM cos://us/sql/timeseries.parquet STORED AS PARQUET    
INTO <your target location> STORED AS PARQUET
```
{: codeblock}

The result of the example query is shown below: 

`[(1,"a"), (3, "b"), (5, "c_d_e"), (7, "f"), (9, "g")]`  

Consider the following input DoubleTimeSeries, which is stored in a table column with the name ts1b:

`[(1,7.0), (3, 8.5), (5, 9.1), (5, 9.3), (5, 9.8), (7, 10.7), (9, 12.2)]`  

The following SELECT statement combines the observations that have identical time-stamps into a single observation whose value is the average of the original values.  

```sql
SELECT TS_COMBINE_DUPLICATE_TIMETICKS(ts1b, TS_COMBINER_AVERAGE())    
FROM cos://us/sql/timeseries.parquet STORED AS PARQUET    
INTO <your target location> STORED AS PARQUET
```
{: codeblock}
 
The result of the example query is shown below:  

`[(1,7.0), (3, 8.5), (5, 9.8), (7, 10.7), (9, 12.2)]`

## Segmentation
{: #segmentation}

Segmentation functions create, as output, a segmented version of a time series. Consider the following input DoubleTimeSeries, which is stored in a table column with the name ts2:  

`[(1, 1.0), (3, 2.0), (5, 3.0), (7, 4.0), (9, 5.0)]`  

The following SELECT statement uses column ts2 and a window size of 2 and a step size of 1 to generate a new DoubleSegmentTimeSeries:  

```sql
SELECT TS_SEGMENT(ts2, 2, 1)     
FROM cos://us/sql/timeseries.parquet STORED AS PARQUET    
INTO <your target location> STORED AS PARQUET
```
{: codeblock}

The result of the example query is shown below:  

`[(1, [(1, 1.0), (3, 2.0)]), (3, [(3, 2.0), (5, 3.0)]), (5, [(5, 3.0),(7, 4.0)]), (7, [(7, 4.0),(9, 5.0)])]`  

Some statistical functions operate expressly on segmented data. 
For example, the following SELECT statement returns a DoubleTimeSeries that contains, for each segment of a DoubleSegmentTimeSeries, the average of its values.  

```sql
SELECT TS_SEG_AVG(TS_SEGMENT(ts2, 2, 1))    
FROM cos://us/sql/timeseries.parquet STORED AS PARQUET    
INTO <your target location> STORED AS PARQUET
```
{: codeblock}
 
The result of the example query is shown below:  

`[(1, 1.5), (3, 2.5), (5, 3.5), (7, 4.5)]`

## Time reference system (TRS)
{: #trs}

If necessary, use a [TRS](/docs/services/sql-query?topic=sql-query-TRS) to specify the time granularity of and a start time for a time series. Consider the following input DoubleTimeSeries, which is stored in a table column with the name ts3.  

`[(1, 1.0), (3, 2.0), (5, 3.0), (7, 4.0), (9, 5.0)]`  

The following SELECT statement uses column ts3 and a TRS to generate a new DoubleTimeSeries that has a time granularity of 1 day and a start time of 00:00 on 01 January 1990.  

```sql
SELECT TS_WITH_TRS(ts3, TS_TRS('PT24H', '1990-01-01T00:00:00'))    
FROM cos://us/sql/timeseries.parquet STORED AS PARQUET    
INTO <your target location> STORED AS PARQUET
```
{: codeblock}

The result of the example query is shown below:  

`[(1=1990-01-02T00:00:00, 1.0), (3=1990-01-04T00:00:00, 2.0), (5=1990-01-06T00:00:00, 3.0), (7=1990-01-08T00:00:00, 4.0), (9=1990-01-10T00:00:00, 5.0)]`

## Cleaning
{: #cleaning}

Consider the following time series, which is stored in a table column with the name ts5:  

`[(1, NaN), (3, 2.0), (5, NaN), (7, 6.0), (8, 7.1)]`  

You can use the TS_FILLNA function to replace the null values in this time series, for example:  

```sql
SELECT TS_FILLNA(ts5, TS_INTERPOLATOR_PREV(-1.0))    
FROM cos://us/sql/timeseries.parquet STORED AS PARQUET    
INTO <your target location> STORED AS PARQUET
```
{: codeblock}

The result of the example query is shown below:  

`[(1, -1.0), (3, 2.0), (5, 2.0), (7, 6.0), (8, 7.1)]`  

The null value at timestamp 1 is replaced with the fill value (-1.0).
The null value at timestamp 5 is replaced with the value of the previous observation (2.0).

## Resampling
{: #resampling}

Consider the following time series, which is stored in a table column with the name ts6:  

`[(1, NaN), (3, 2.0), (5, 3.5), (7, 6.0), (8, 7.1)]`  

Running the TS_RESAMPLE function with an interpolater of type TS_INTERPOLATOR_PREV, periodicity 1, and fill value -1.0, produces the following output:  

```sql
SELECT TS_RESAMPLE(ts6, 1, TS_INTERPOLATOR_PREV(-1.0))    
FROM cos://us/sql/timeseries.parquet STORED AS PARQUET    
INTO <your target location> STORED AS PARQUET
```
{: codeblock}

The result of the example query is shown below:  

`[(1, NaN), (2, NaN), (3, 2.0), (4, 2.0), (5, 3.5), (6, 3.5), (7, 6.0), (8, 7.1)]`  

For each of the generated observations, the value is set to the value of the previous observation.

Changing the periodicity to 2 produces the following output:  

```sql
SELECT TS_RESAMPLE(ts6, 2, TS_INTERPOLATOR_PREV(-1.0))    
FROM cos://us/sql/timeseries.parquet STORED AS PARQUET    
INTO <your target location> STORED AS PARQUET
```
{: codeblock}

The result of the example query is shown below:  

`[(0, -1.0), (2, NaN), (4, 2.0), (6, 3.5), (8, 7.1)]`  

Note:  

The timestamp of the first generated observation is calculated by the formula TRUNCATE(first_timestamp/period)*x*period, so the first timestamp in the output time series is not necessarily the same as the first timestamp in the input time series. In this example, TRUNCATE(1/2)*x*2=0. Due to the difference in periodicity of the input and output time series, some of the observations in the input time series do not appear in the output. However, the values of any skipped observations are used during interpolation.  

Changing the periodicity to 3 produces the following output:  

```sql
SELECT TS_RESAMPLE(ts6, 3, TS_INTERPOLATOR_PREV(-1.0))    
FROM cos://us/sql/timeseries.parquet STORED AS PARQUET    
INTO <your target location> STORED AS PARQUET
```
{: codeblock}

The result of the example query is shown below:  

`[(0, -1.0), (3, 2.0), (6, 3.5)]`

