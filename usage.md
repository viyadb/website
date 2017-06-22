Usage
======

The following section describes how to configure and run a ViyaDB instance.

## Configuring DB Instance

Store descriptor preresents a configuration file of a single ViyaDB instance. The format is the following:

```json
{
  "query_threads": 1,
  "cpu": [ ... ],
  "port": 52341,
  "tables": [ ... ]
}
```

Parameters:

 * query\_threads - Number of threads that will serve queries, defaults to 1.
 * cpu - List of zero-based CPU indices that this process will use, defaults to all available CPUs.
 * port - REST service port number, defaults to 52341.
 * tables - List of [table descriptors](#usage-creating-tables).

## Creating Tables

Table descriptors can be either a part of [store descriptor], or they can be created on-demand using REST API.

```json
{
  "name": "<table name>",
  "dimensions": [ ... ],
  "metrics": [ ... ]
}
```

Parameters:

 * name - Table name
 * dimensions - List of [dimension descriptors](#usage-creating-tables-dimensions)
 * metrics - List of [metric descriptors](#usage-creating-tables-metrics)

### Dimensions

There are four types of dimensions:

 * String
 * Numeric
 * Time
 * Microtime

#### String Dimension

String dimension is a basic one, and it's used to describe things like: country, user agent, event name, etc.
Description format is as follows:

```json
{
  "name": "<dimension name>",
  "type": "string",
  "cardinality": ...,
  "cardinality_guard": {
    "dimensions": ["<other dimension>", ...],
    "limit": ...
  }
}
```

Parameters:

 * name - Column name
 * type - Must be `string` (or can be omitted, since it's default)
 * cardinality - Number of distinct values this column holds (optional, but it's recommended to set)

`cardinality_guard` allows defining a rule of how many distinct values is it possible to store per set
of other dimensions. For example, we can decide to store at most 200 distinct event names per event date, per country.
All other events will be accounted still, but they will be marked as `__exceeded`. This is really important option,
especially when incoming events are not controlled by yourself, and you don't want your database memory to explode because
someone decided to sent some random values.

#### Numeric Dimension

This dimension allows to store whole positive numbers as a non-metric column. Description format is:

```json
{
  "name": "<dimension name>",
  "type": "numeric",
  "max": ...
}
```

Parameters:

 * name - Column name
 * type - Must be `numeric`.
 * max - Maximum number this column can be (optional, but it's recommended to set)

#### Time and Microtime Dimensions

This dimension allows to store UTC time. The difference between the two is that `time` precision is up to seconds, while
`microtime` precision is up to microseconds.

```json
{
  "name": "<dimension name>",
  "type": "time|microtime",
  "format": ...,
  "granularity": "<time unit>",
  "rollup_rules": [ ... ]
}
```

Parameters:

 * name - Column name
 * type - The type is set according to the required precision
 * format - Parse format used during data ingestion (check [strptime](http://man7.org/linux/man-pages/man3/strptime.3.html) documentation for available modifiers)
 * granularity - When specified, this time granularity will be used for rolling up events during data ingestion
 * rollup\_rules - Rules for dynamic period based rollup

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

## Data Ingestion

### Loading from TSV files

Save load descriptor into `load.json` file:

```json
{
  "table": "target_table",
  "format": "tsv",
  "type": "file",
  "file": "/path/to/data.tsv"
}
```

Post this load descriptor to running ViyaDB instance:

    ~$ curl --data-binary @load.json http://<viyadb-host>:<viyadb-port>/load

Important notes:

 * The `data.tsv` file must be accessible to the ViyaDB instance you're loading the data into.
 * Order of columns in .tsv file must be as follows: first dimensions, then metrics as they appear in the table descriptor.

## Querying


