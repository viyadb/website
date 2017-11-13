Usage
======

The following section describes how to configure and run a ViyaDB instance on a single node.

## General

Interaction with ViyaDB instance is performed using REST API. Sometimes, it doesn't look like REST (See [Data Ingestion](#data-ingestion) or [Querying](#querying) sections below), but it can always be thought as a resource you're sending a request to is an action itself.

## Configuring DB Instance

Store descriptor preresents a configuration file of a single ViyaDB instance. The format is the following:

```json
{
  "query_threads": 1,
  "cpu": [ ... ],
  "workers": 1,
  "http_port": 5000,
  "tables": [ ... ],
  "statsd": {
    "host":  ... ,
    "port": 8125,
    "prefix": "viyadb.%h."
  },
  "supervise": false
}
```

Parameters:

 * query\_threads - Optional number of threads that will serve queries, defaults to 1.
 * cpu\_list - Optional list of zero-based CPU indices that this process will use, defaults to all available CPUs.
 * workers - Optional number of workers to start, defaults to the number of available CPUs.
 * port - All workers will be assigned a different port number, starting from this one (defaults to 5000).
 * tables - List of [table descriptors](#creating-tables).
 * statsd - If specified, some metrics will be reported to the given Statsd host.
 * supervise - Optionally, run a supervisor on top of worker processes. This setting must be `true` if number of workers is greater than 1.

## Creating Tables

Table descriptors can be either a part of [store descriptor](#configuring-db-instance), or they can be created on-demand using REST API.

```json
{
  "name": "<table name>",
  "dimensions": [ ... ],
  "metrics": [ ... ],
  "watch": {
    "directory": "<path to files>",
    "extensions": [".tsv"]
  }
}
```

Parameters:

 * name - Table name
 * dimensions - List of [dimension descriptors](#dimensions)
 * metrics - List of [metric descriptors](#metrics)
 * watch - Optional configuration that enables watching directory for new files, and loading them automatically.
 
To create a table, issue the following command:

```bash
curl --data-binary @table.json http://<viyadb-host>:<viyadb-port>/tables
```

!!! note "Explanation in SQL terms"
    An analogy can be drawn between ViyaDB tables and SQL aggregation queries.
    For example, consider the following SQL statement:

    ```sql
    SELECT app_id, SUM(revenue) FROM events GROUP BY app_id
    ```

    This example translates to ViyaDB table `events` that has a single dimension `app_id` and a single metric `revenue`.

### Dimensions

There are four types of dimensions:

 * String
 * Numeric
 * Boolean
 * Time
 * Microtime

#### String Dimension

String dimension is a basic one, and it's used to describe things like: country, user agent, event name, etc.
Description format is as follows:

```json
{
  "name": "<dimension name>",
  "field": "<input field name>",
  "type": "string",
  "length": ... ,
  "cardinality": ... ,
  "cardinality_guard": {
    "dimensions": ["<other dimension>", ... ],
    "limit":  ... 
  }
}
```

Parameters:

 * name - Column name
 * field - Use alternative input field name (defaults to the column name itself)
 * type - Must be `string` (or can be omitted, since it's default)
 * length - Optionally, specify maximum length for a column value (values exceeding this limit will be stripped).
 * cardinality - Number of distinct values this column holds (optional, but it's recommended to set)

`cardinality_guard` allows defining a rule of how many distinct values is it possible to store per set
of other dimensions. For example, we can decide to store at most 200 distinct event names per event date, per country.
All other events will be accounted still, but they will be marked as `__exceeded`. This is really important option,
especially when incoming events are not controlled by yourself, and you don't want your database memory to explode because
someone decided to sent some random values.

#### Numeric Dimension

This dimension allows to store numbers as a non-metric column. Column description format is
the following:

```json
{
  "name": "<dimension name>",
  "field": "<input field name>",
  "type": "<type>"
}
```

Parameters:

 * name - Column name
 * field - Use alternative input field name (defaults to the column name itself)
 * type - Numeric type (see below)

Supported numeric types are:

 + byte (-128 to 127)
 + ubyte (0 to 255)
 + short (-32768 to 32767)
 + ushort (0 to 65535)
 + int (-2147483648 to 2147483647)
 + uint (0 to 4294967295)
 + long (-9223372036854775808 to 9223372036854775807)
 + ulong (0 to 18446744073709551615)
 + float (floating point 32 bit number)
 + double (floating point 64 bit number)

!!! note "Deprecation note"
    Previously, only `uint` and `ulong` types were supported through `numeric` and `max` specifications.
    You should upgrade to the new numeric types if you're still using the old way.

#### Boolean Dimension

Stores either boolean value of 'true' or 'false'.

```json
{
  "name": "<dimension name>",
  "field": "<input field name>",
  "type": "boolean"
}
```

Parameters:

 * name - Column name
 * field - Use alternative input field name (defaults to the column name itself)
 * type - Must be `boolean`.

#### Time and Microtime Dimensions

This dimension allows to store UTC time. The difference between the two is that `time` precision is up to seconds, while
`microtime` precision is up to microseconds.

```json
{
  "name": "<dimension name>",
  "field": "<input field name>",
  "type": "time|microtime",
  "format":  ... ,
  "granularity": "<time unit>",
  "rollup_rules": [ ... ]
}
```

Parameters:

 * name - Column name
 * field - Use alternative input field name (defaults to the column name itself)
 * type - The type is set according to the required precision
 * format - See below
 * granularity - When specified, this time granularity will be used for rolling up events during data ingestion
 * rollup\_rules - Rules for dynamic period based rollup

Parse format is one of the following:

 * posix - Input time is in POSIX time format (seconds since UNIX epoch)
 * millis - Input time is in milliseconds since UNIX epoch
 * micros - Input time is in microseconds since UNIX epoch
 * Any other string that uses [strptime](http://man7.org/linux/man-pages/man3/strptime.3.html) modifiers for describing format

Supported time units:

 * year
 * month
 * week
 * day
 * hour
 * minute
 * second

Dynamic rollup rules are defined using the following format:

```json
{
  "after": "<num> <time unit>"
  "granularity": "<time unit>"
}
```

For example, if the rules are:

```json
[{
   "after": "3 month",
   "granularity": "week"
 }, {
   "after": "1 year",
   "granularity": "month"
}]
```

Then events time column will change granularity dynamically to `weekly` after 3 months, to `monthly` after 1 year.

### Metrics

There are three supported metric types:

 * Value
 * Count
 * Bitset

#### Value Metric

Value metric is a numeric value combined with an aggregation function.  List of supported numeric types:

 + byte (-128 to 127)
 + ubyte (0 to 255)
 + short (-32768 to 32767)
 + ushort (0 to 65535)
 + int (-2147483648 to 2147483647)
 + uint (0 to 4294967295)
 + long (-9223372036854775808 to 9223372036854775807)
 + ulong (0 to 18446744073709551615)
 + float (floating point 32 bit number)
 + double (floating point 64 bit number)

Supported functions:

 + sum
 + max
 + min
 + avg

The format of defining a value metric is the following:

```json
{
  "name": "<metric name>",
  "field": "<input field name>",
  "type": "<value_type>_<function>"
}
```

#### Count Metric

This type of metric just counts number of incoming rows. To define it, use the following format:

```json
{
  "name": "<metric name>",
  "field": "<input field name>",
  "type": "count"
}
```

#### Bitset Metric

This metric allows storing numeric values in a memory optimized set structure, which supports
operations like `intersect` and `union`. This makes possible to run queries like: "what is a value
cardinality for a given set of dimensions filtered by a perdicate?". For instance: counting unique
mobile app users, which installed an application on last month split by country.

Metric format:


```json
{
  "name": "<metric name>",
  "field": "<input field name>",
  "type": "bitset"
}
```

## Data Ingestion

### Loading from TSV files

Save load descriptor into `load.json` file:

```json
{
  "table": "target_table",
  "format": "tsv",
  "type": "file",
  "columns": [...],
  "file": "/path/to/data.tsv"
}
```

Post this load descriptor to a running ViyaDB instance:

```bash
curl --data-binary @load.json http://<viyadb-host>:<viyadb-port>/load
```

Important notes:

 * The `data.tsv` file must be accessible to the ViyaDB instance you're loading the data into.
 * If `columns` parameter is not provided, order of columns in .tsv file must be as follows: first dimensions, then metrics as they appear in the table descriptor.
 * `columns` parameter must be used whenever one of table column defines `field` attribute.

## Querying

Supported query types are:

 * Aggregate
 * Search

To query, a query request must be submitted using POST method:

```
curl --data-binary @query.json http://<viyadb-host>:<viyadb-port>/query
```

Read further to learn more on different query types.
    
### Aggregate Query

This kind of query aggregates records selected using filter predicate, and returns them to user (optionally, sorted and/or limited). It's important to know that aggregation is done in a memory, therefore all the result set must fit in.

Query format:

```json
{
  "type": "aggregate",
  "table": "<table name>",
  "select": [ ... ],
  "filter":  ... ,
  "having": ...,
  "sort": [ ... ] ,
  "skip": 0,
  "limit": 0
}
```

Parameters:

 * table - Table name
 * select - List of parameters describing how to select a column (see below)
 * filter - Filter description (see below)
 * having - Post-aggregation filter description (similar to SQL HAVING clause). The format is the same as in `filter` attribute.
 * sort - Optional result sorting configuration (see below)
 * skip - Optionally, skip this number of output records
 * limit - Optionally, limit result set size to this number
 
#### Column Selector

Column is either dimension or metric. The format of selecting either of them is the following:

```json
{
  "column": "<column name>",
  "format": "<time format>",
  "granularity": "<time granularity>"
}
```

Parameters:

 * column - Dimension or metric name
 
Time column has two additional optional parameters:

 * format - Time output format (check [strptime](http://man7.org/linux/man-pages/man3/strptime.3.html) documentation for available modifiers). By default, UTC epoch timestamp will be sent
 * granularity - Rollup results by this time unit (see [time dimension](#time-and-microtime-dimensions) configuration for supported time units)

#### Query Filters

Filter is one of mandatory parameters in a query, which allows skipping irrelevant records. There are four different filter types.

##### Value Operator Filter

```json
{
  "op": "<filter operator>",
  "column": "<column name>",
  "value": "<filter value>"
}
```

Parameters:

 * op - Filter kind specified by operator
 * column - Dimension or metric name
 * value - Value that filter operates on
 
Supported operators are:

 * eq - Equals
 * ne - Not equals
 * lt - Less than
 * le - Less or equals to
 * gt - Greater than
 * ge - Greater or equals to

##### Negation Filter

```json
{
  "op": "not",
  "filter":  ... 
}
```

Parameters:

 * op - Must be `not`
 * filter - Other filter descriptor

##### Inclusion Filter

```json
{
  "op": "in",
  "column": "<column name>",
  "values": ["value1", ... ]
}
```

Parameters:

 * op - Must be `in`
 * column - Dimension or metric name
 * values - List of values to filter on

##### Composite Filter

```json
{
  "op": "<composition operator>",
  "filters": [ ... ]
}
```

Parameters:

 * op - One of composition operators: `or`, `and`
 * filters - List of other filters to compose


##### Filter Example

Below is an example of using different filter types:

```json
{
  "type": "aggregate",
  "table": "activity",
  "select": [
    {"column": "install_date"},
    {"column": "ad_network"},
    {"column": "install_country"},
    {"column": "installs_count"},
    {"column": "launches_count"},
    {"column": "inapps_count"}
  ],
  "filter": {"op": "and", "filters": [
    {"op": "eq", "column": "app_id", "value": "com.teslacoilsw.notifier"},
    {"op": "ge", "column": "install_date", "value": "2015-01-01"},
    {"op": "lt", "column": "install_date", "value": "2015-01-30"},
    {"op": "in", "column": "install_country", "values": ["US", "IL", "RU"]}
  ]},
  "having": {"op": "gt", "column": "inapps_count", "value": "10"}
}
```

#### Sorting Results

You can ask to sort output results by set of columns, and specify sort order on each of them. Column sort configuration goes as follows:

```json
{
   "column": "<column name>",
   "ascending": true|false
}
```

`ascending` parameter can be ommitted, in this case the sort order will be descending. 

### Search Query

This query type retrieves dimension values by a given set of filters. This feature can come in handy when implementing a field values type assist, when developing an analytics user interface filter.

The basic format is the following:

```json
{
  "type": "search",
  "table": "<table name>",
  "dimension": "<dimension name>",
  "term": "<search term>",
  "limit": 0,
  "filter":  ...
}
```

### Metadata Queries

There are several endpoints that can be used for retreiving information about database status.

#### Database Metadata Query

The following query returns basic information about the database:

```bash
curl http://<viyadb-host>:<viyadb-port>/database/meta
```

#### Table Metadata Query

To see all available table fields, their types as long as some basic statistics
use the following query:

```bash
curl http://<viyadb-host>:<viyadb-port>/tables/<table name>/meta
```


